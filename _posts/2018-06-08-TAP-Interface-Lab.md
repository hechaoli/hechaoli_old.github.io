---
layout: post
title: TAP Interface Lab
subtitle: A Hypervisor Emulator
tags: [Linux, Network, TAP, Lab]
---

# Overview
We talked about TUN and TAP interface in [previous
post](/2018/05/21/Tun-Tap-Interface/). In the article [Tun/Tap interface
Tutorial](http://backreference.org/2010/03/26/tuntap-interface-tutorial), the
author wrote a program to demonstrate the usage of a TUN interface. In this
post, we will write a program that uses TAP interfaces. The code for this lab
can be found at [Github](https://github.com/hechaoli/tap-lab).

# What Will We Do?

We are going to write a hypervisor. Don't worry. We are not going to write a
real hypervisor with all functionalities. We will only simulate the network
manager part. When a "virtual machine" starts, the "hypervisor" creates a TAP
interface and associates it with the VM. Then the hypervisor forwards traffic
between VMs and TAP interfaces. Following is a diagram from [previous
post](/2018/05/21/Tun-Tap-Interface/).

![TAP Use Case](/img/tap-use-case.png)

# What Will We Learn?

Through this lab, we will:

* Learn how a VM gets network access with the help of the hypervisor and TAP
  interface.
* Review the usage of [Linux bridge](/2017/12/13/linux-bridge-part1).
* Practice Object-Oriented-Programming (OOP) in C++.

# How Do We Implement it?
To better demonstrate the concept and make our life easy, in this lab I will
omit error handling and simplify a lot of things.

## Design
The two most important classes are `Hypervisor` class and `VirtualMachine`
class. Other utility classes are used to help `VirtualMachine` to handle network
packets.

![TAP Lab Design](/img/tap-lab-design.png)

## Hypervisor
A hypervisor provides a public method to create a virtual machine. As we know,
each VM has a TAP interface associated with it. These TAP interfaces are also
added to a Linux bridge. To make it easy, the step of creating TAP interfaces
and adding them to the bridge is done by the driver script.

### VM Creation
When a VM is created, a TAP interface is also created and is added to the
bridge (This step is not in the code but in the script in this lab). Creation of
a VM may update the `max_fd` that is used in `select()` call we will see soon.

```c++
VirtualMachine *Hypervisor::createVM(const string &mac, const string &ip) {
    int vm_id = next_vm_id++;
    string tap_name = "tap" + to_string(vm_id);
    int tap_fd = GetTapFd(tap_name);

    VirtualMachine *vm = new VirtualMachine(mac, ip, tap_fd);
    lock_guard<std::mutex> lock(vm_map_mutex);
    vm_map[tap_fd] = vm;
    max_fd = max(max_fd.load(), tap_fd);
    return vm;
}
```

`GetTapFd` is used to get the file descriptor of the TAP interface. The method
will create a new TAP interface if it does not exist already. Again, in our lab,
it won't create one because the driver script has created all TAP interfaces.
```c++
static int GetTapFd(const string &name) {
    int fd, err;

    if ((fd = open("/dev/net/tun", O_RDWR)) < 0) {
        perror("Failed to open /dev/net/tun");
        return fd;
    }

    struct ifreq ifr;
    memset(&ifr, 0, sizeof(ifr));

    // Don't provide packet information
    ifr.ifr_flags = IFF_TAP | IFF_NO_PI;
    if (!name.empty()) {
        strncpy(ifr.ifr_name, name.c_str(), IFNAMSIZ);
    }

    if ( (err = ioctl(fd, TUNSETIFF, (void *) &ifr)) < 0 ) {
        close(fd);
        perror("Failed to create TAP interface");
        return err;
    }
    return fd;
}
```

### Initialization
When a hypervisor starts, it starts a thread to monitor all file descriptors of
the TAP interfaces that are associated with VMs.

```c++
void Hypervisor::Init() {
    max_fd = -1;
    next_vm_id = 0;
    auto loop = [&]() {
        while (true) {
            fd_set fds;
            BuildFdSet(&fds);
            struct timeval timeout;
            timeout.tv_sec = 1;
            timeout.tv_usec = 0;
            int ret = select(max_fd + 1, &fds, NULL, NULL, &timeout);
            if (ret < 0) {
                if (errno == EINTR) {
                    continue;
                }
                perror("select()");
                return;
            } else if (ret == 0) {
                continue;
            }
            HandleRead(&fds);
        }
    };

    select_thread = thread(loop);
}
```

Initially the `max_fd` is -1, it is increased as we add more VMs. `BuildFdSet`
will set all existing VM's TAP file descriptors.
```c++
void Hypervisor::BuildFdSet(fd_set *fds) {
    lock_guard<std::mutex> lock(vm_map_mutex);
    FD_ZERO(fds);
    for (auto &kv : vm_map) {
        int fd = kv.first;
        FD_SET(fd, fds);
    }
}
```
`HandleRead` will read from all TAP file descriptors that are ready and forward
the data to corresponding VM.
```c++
void Hypervisor::HandleRead(fd_set *fds) {
    lock_guard<std::mutex> lock(vm_map_mutex);
    for (auto &kv : vm_map) {
        int fd = kv.first;
        if (FD_ISSET(fd, fds)) {
            VirtualMachine *vm = kv.second;
            uint8_t buf[BUF_SIZE];
            int len = read(fd, buf, sizeof(buf));
            vm->SendToVm(buf, len);
        }
    }
}
```

## Virtual Machine

### Initialization
When the virtual machine starts, it starts a thread receiving packets from the
network. It will handle the packets based on its type. Currently it only handles
4 types of packets:

* **ARP Request** - If the target IP address matches its own IP address, it sends an
  ARP reply packet to the sender.
* **ARP Reply** - If it receives an ARP reply packet, it means that there is a
  blocking outstanding ARP request. Thus it will unblock that request.
* **ICMP Echo Request** - Sends a ICMP echo reply packet to the sender.
* **ICMP Echo Reply** - If it receives an ICMP echo reply packet, it means that
  there is a blocking outstanding ICMP echo request. Thus it will unblock that request.

```c++
void VirtualMachine::Init() {
    icmp_id = 1;
    icmp_seq = 1;
    // Right now we only support ingress ARP and ICMP packets
    auto loop = [&]() {
        cout << "VM [" << ip << ", " << mac << " starts running." << endl;
        while (true) {
            struct eth_hdr eth_hdr;
            RecvFromNetwork((uint8_t *)&eth_hdr, sizeof(eth_hdr));
            string src_mac = EthUtil::MacBytesToString(eth_hdr.h_source);
            cout << "[" << ip << "] Received ethernet frame from " << src_mac << endl;
            eth_hdr.h_proto = ntohs(eth_hdr.h_proto);
            if (ETH_P_ARP == eth_hdr.h_proto) {
                HandleIngressArp();
            } else if (ETH_P_IP == eth_hdr.h_proto) {
                HandleIngressIcmp(src_mac);
            } else {
                cout << "[" << ip << "] Received unsupported ethernet type "
                     << eth_hdr.h_proto << endl;
            }
        }
    };
    ingress_proc_thread = thread(loop);
}
```

### Ping
Ping sends an ICMP echo request to destination IP. Before that, if the
destination MAC address is not in the ARP table, it will first send a ARP
request packet. To make it simple, after sending the ARP request and ICMP
request, the thread will be blocked until the VM receives the reply.

```c++
void VirtualMachine::Ping(const string &dst_ip) {
    unique_lock<mutex> arp_lock(arp_table_mutex);
    if (arp_table.find(dst_ip) == arp_table.end()) {
        cout << "[" << ip << "] Sending ARP request to "
             << dst_ip << "..." << endl;
        SendArp(dst_ip, kEthBroadcastAddr, ARP_OP_REQUEST);
        cout << "[" << ip << "] Waiting for ARP reply from "
             << dst_ip << "..." << endl;
        arp_cv.wait(arp_lock, [&]{ return arp_table.find(dst_ip) != arp_table.end(); });
    }

    string dst_mac = arp_table[dst_ip];
    uint16_t id = icmp_id++, seq_num = icmp_seq++;
    pair<uint16_t, uint16_t> key = make_pair(id, seq_num);

    cout << "[" << ip << "] Ping " << dst_ip << " ..." << endl;
    auto start = chrono::steady_clock::now();

    SendIcmp(dst_ip, dst_mac, ICMP_ECHO_REQUEST, id, seq_num);
    unique_lock<mutex> icmp_lock(icmp_reply_mutex);
    icmp_cv.wait(icmp_lock, [&]{ return icmp_replies.find(key) != icmp_replies.end();});

    auto end = chrono::steady_clock::now();
    auto diff = end - start;
    cout << "[" << ip << "] Ping response from " << ip << ": icmp_seq=" << seq_num;
    cout << " time=" << chrono::duration <double, milli> (diff).count() << " ms" << endl;

    icmp_replies.erase(key);
}
```


## Utility Classes
The utility classes provide helper functions to construct network packets. Since
they are not the core of this lab, I won't bother describing them one by one.
You can checkout the [code](https://github.com/hechaoli/tap-lab/tree/master/src)
if interested.

# How Do We Run It?
In the main function, we create a hypervisor and launch two VMs using it. Then
we ping from VM1 to VM2:
```c++
int main() {
    Hypervisor hypervisor;

    string mac1 = "00:00:00:00:00:01";
    string ip1 = "192.168.1.1";
    VirtualMachine *vm1 = hypervisor.createVM(mac1, ip1);

    string mac2 = "00:00:00:00:00:02";
    string ip2 = "192.168.1.2";
    VirtualMachine *vm2 = hypervisor.createVM(mac2, ip2);

    // Wait for the VM to start
    this_thread::sleep_for (std::chrono::seconds(1));

    // Ping VM2 from VM1
    vm1->Ping(ip2);
    //vm2->Ping(ip1);

    this_thread::sleep_for (std::chrono::seconds(300));
    return 0;
}
```
Before running it, we need to do some preparation.
```bash
$ sudo brctl addbr br0 # Add bridge br0
$ sudo ip tuntap add dev tap0 mode tap # Add TAP interface tap0
$ sudo ip tuntap add dev tap1 mode tap # Add TAP interface tap1
$ sudo brctl addif br0 tap0 # Add tap0 to br0
$ sudo brctl addif br0 tap1 # Add tap1 to br0
$ sudo ifconfig tap0 up
$ sudo ifconfig tap1 up
$ sudo ifconfig br0 up
```
And we can start wireshark to capture the packets on `tap0`.
Now compile the code and run.
```bash
$ make
$ ./tap-lab
VM [192.168.1.1, 00:00:00:00:00:01] starts running.
VM [192.168.1.2, 00:00:00:00:00:02] starts running.
[192.168.1.1] Sending ARP request to 192.168.1.2...
[192.168.1.1] Waiting for ARP reply from 192.168.1.2...
[192.168.1.2] Received ethernet frame from 00:00:00:00:00:01
[192.168.1.2] Received ARP request: [Who has 192.168.1.2? Tell 192.168.1.1]
[192.168.1.2] Sending ARP reply: [192.168.1.2 is at 00:00:00:00:00:02]
[192.168.1.1] Received ethernet frame from 00:00:00:00:00:02
[192.168.1.1] Ping 192.168.1.2 ...
[192.168.1.2] Received ethernet frame from 00:00:00:00:00:01
[192.168.1.2] Received ICMP request id = 1, seq_num = 1
[192.168.1.2] Sending ICMP reply id = 1, seq_num = 1
[192.168.1.1] Received ethernet frame from 00:00:00:00:00:02
[192.168.1.1] Received ICMP reply id = 1, seq_num = 1
[192.168.1.1] Ping response from 192.168.1.1: icmp_seq=1 time=1.78907 ms
```

In wireshark, we get
![TAP Lab Ping](/img/tap-lab-ping.png)

This is a very typical ping traffic. And this traffic shows that we successfully
simulate the behavior of hypervisor and VMs with the help of TAP interface and
Linux bridge.

# Next Steps
Readers are encouraged to extend this lab to support more packet type and make
the hypervisor and VM more robust. The code can be found at
[Github](https://github.com/hechaoli/tap-lab)

# Reference
[1] [tuntap](https://www.kernel.org/doc/Documentation/networking/tuntap.txt)<br>
[2] [Tun/Tap interface
tutorial](http://backreference.org/2010/03/26/tuntap-interface-tutorial/)<br>
