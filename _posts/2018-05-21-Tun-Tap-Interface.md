---
layout: post
title: TUN/TAP Interface
tags: [Linux, Network, TUN, TAP]
---

# Concept
From the Linux kernel
[documentation](https://www.kernel.org/doc/Documentation/networking/tuntap.txt):

> TUN/TAP provides packet reception and transmission for user space programs.
> It can be seen as a simple Point-to-Point or Ethernet device, which,
> instead of receiving packets from physical media, receives them from
> user space program and instead of sending packets via physical media
> writes them to the user space program.

In other words, TUN/TAP interfaces are virtual interfaces that does not have
physical devices associated. A user space program can attach to a TUN/TAP
interface and handle the traffic sent to the interface.

# Difference
A TUN interface is a virtual IP Point-to-Point interface and a TAP interface is
a virtual Ethernet interface. That means the user program can only read/write
IP packets from/to a TUN interface and Ethernet frames from/to a TAP
interface.

# Use Cases
The typical use case of a TUN interface is IP tunneling. For example,
[OpenVPN](https://openvpn.net/) receives packets from a TUN interface such as
`tun0` and encrypts it before sending to the real ethernet interface `eth0`.
Then the OpenVPN client on the peer receives the packet from `eth0` and decrypts
it before sending it to `tun0`. In other words, OpenVPN works as a proxy between
`tun0` and `eth0` and creates a encrypted UDP connection over the internet
between two hosts[5].

![TUN Use Case](/img/tun-use-case.png)
(Image credit [6])

The typical use case of a TAP interface is virtual networking. For example, in
[Linux Bridge Part 1](/2017/12/13/linux-bridge-part1), we've seen that when we
create a VM in the KVM with bridged network, it creates a TAP interface like
`vnet0` and adds it to the Linux bridge. In this case, KVM is the userspace
program which reads from and writes to the TAP interfaces. When VM0 sends a
packet to its `eth0`, KVM sends it to TAP interface `vnet0` so that the bridge
will forward it to `vnet1`. Then KVM receives it and sends it to VM1's `eth0`.

![TAP Use Case](/img/tap-use-case.png)

# Managing TUN/TAP interfaces
`ip tuntap` can be used to manage TUN/TAP interfaces. For example:

```bash
$ ip tuntap help
Usage: ip tuntap { add | del | show | list | lst | help } [ dev PHYS_DEV ]
          [ mode { tun | tap } ] [ user USER ] [ group GROUP ]
          [ one_queue ] [ pi ] [ vnet_hdr ] [ multi_queue ]

Where: USER  := { STRING | NUMBER }
       GROUP := { STRING | NUMBER }
```

# Reference
[1] [Understanding TUN TAP
Interfaces](http://www.naturalborncoder.com/virtualization/2014/10/17/understanding-tun-tap-interfaces/)<br>
[2] [tuntap](https://www.kernel.org/doc/Documentation/networking/tuntap.txt)<br>
[3] [Tun/Tap interface
tutorial](http://backreference.org/2010/03/26/tuntap-interface-tutorial/)<br>
[4] [TUN, TAP and Veth - Virtual Networking Devices
Explained](https://www.fir3net.com/Networking/Terms-and-Concepts/virtual-networking-devices-tun-tap-and-veth-pairs-explained.html)<br>
[5] [What is the principle behind OpenVPN
tunnels?](https://openvpn.net/index.php/open-source/faq/75-general/293-what-is-the-principle-behind-openvpn-tunnels.html)<br>
[6] [OpenVPN: how secure virtual private networks really work](https://cloudacademy.com/blog/openvpn-how-secure-virtual-private-networks-really-work/)
