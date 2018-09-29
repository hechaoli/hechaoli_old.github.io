---
layout: post
title: How Is Ping Deduplexed?
tags: [Network, Linux Code Walkthrough, Lab]
---

# Preface
A few days back, one of my friends asked an interesting question - How is ping
deduplexed by Linux kernel network stack? In other words, when a Linux machine
receives the ping reply, how does it know which socket to send to?

Compared to TCP and UDP, which are uniquely identified by a port number, ICMP
seems to be stateless. After a short discussion, we believe that it is the `id`
field in ICMP header that acts as the identifier. But what if on the same
machine, at the same time, two processes ping with the same id, what will Linux
do when receiving the replies? To answer this, I read the Linux kernel code as
well as `ping` code in `iputils`. I also did some interesting experiments.

# ICMP
Internet Control Message Protocol (ICMP) is used by the famous `ping` tool to
detect the reachability of the other host. When we say "Can you ping google?",
we actually mean, "When you send an ICMP echo request to google, can you receive
a ICMP echo reply back?".

ICMP message can have many types, for the purpose of this article, let's only
focus on the echo and echo reply message, which both have the following format:

![ICMP Echo or Echo Reply Message](/img/icmp_echo_message.png)

From [RFC 792](https://tools.ietf.org/html/rfc792)
```
   IP Fields:

   Addresses

      The address of the source in an echo message will be the
      destination of the echo reply message.  To form an echo reply
      message, the source and destination addresses are simply reversed,
      the type code changed to 0, and the checksum recomputed.

   ICMP Fields:

   Type

      8 for echo message;

      0 for echo reply message.

   Code

      0

   Checksum

      The checksum is the 16-bit ones's complement of the one's
      complement sum of the ICMP message starting with the ICMP Type.
      For computing the checksum , the checksum field should be zero.
      If the total length is odd, the received data is padded with one
      octet of zeros for computing the checksum.  This checksum may be
      replaced in the future.

   Identifier

      If code = 0, an identifier to aid in matching echos and replies,
      may be zero.

   Sequence Number

      If code = 0, a sequence number to aid in matching echos and
      replies, may be zero.
```

{: .box-warning}
**Note**: Though ICMP header is inside IP header, ICMP is considered as a Layer
3 protocol.

As we can see from above description, identifier and sequence number together
can be used to match request and reply. But what if two packets come with the
same identifier and sequence number? RFC doesn't say anything about this
situation, which means we have to look at the implementation.

# Ping
The source code of `ping` in `iputils` can be found
[here](https://github.com/iputils/iputils/blob/master/ping.c).

Part of the question asked by my friend is, what if you have two ping process
sending two ICMP echo requests with the same identifier at the same time? By
reading `ping` source code, we will know that the short answer is, that's
impossible.

`ping4_send_probe` is called to send ICMP echo request messages:

```c
int ping4_send_probe(socket_st *sock, void *packet, unsigned packet_size)
{
    struct icmphdr *icp;
    int cc;
    int i;

    icp = (struct icmphdr *)packet;
    icp->type = ICMP_ECHO;
    icp->code = 0;
    icp->checksum = 0;
    icp->un.echo.sequence = htons(ntransmitted+1);
    icp->un.echo.id = ident;			/* ID */
    ...
}
```

This function set the id field of the ICMP header to `ident`. This variable is
set to the `ping` process's id when it starts. This can be found in
`setup()` function in
[ping_common.c](https://github.com/iputils/iputils/blob/master/ping_common.c).

```c
void setup(socket_st *sock)
{
    ...
    if (sock->socktype == SOCK_RAW)
    	ident = htons(getpid() & 0xFFFF);
    ...
}
```

Now we know that there is no way to send two ICMP echo request with the same id
by using `ping`. In this case, the unique identifier can be used as the "port"
of ICMP. The sequence number is increased by 1 each time an ICMP echo request is
sent. When you run `ping -c 10 google.com`, all 10 ICMP messages have the same
id but increasing sequence number.

You may still ask, though two `ping` processes always have different ids, what if
I write my own `ping` program which can use identical ids? In this case, can the
kernel delivers the message to the right socket? To answer that, let's dive into
Linux kernel code.

# Linux Kernel Ping
Function `icmp_rcv()` in file
[/net/ipv4/icmp.c](https://elixir.bootlin.com/linux/v4.18.10/source/net/ipv4/icmp.c)
is called to handle incoming ICMP packets.
```c
int icmp_rcv(struct sk_buff *skb)
{
    ...
    success = icmp_pointers[icmph->type].handler(skb);
    ...
}
```

`icmp_recv` depends on a list of handlers to handle ICMP packets of different
types. And it's definition is shown below:

```c
static const struct icmp_control icmp_pointers[NR_ICMP_TYPES + 1] = {
    [ICMP_ECHOREPLY] = {
    	.handler = ping_rcv,
    },
    ...
}
```

The handler used to handle `ICMP_ECHOREPLY` is `ping_recv` in file
[/net/ipv4/ping.c](https://elixir.bootlin.com/linux/v4.18.10/source/net/ipv4/ping.c#L973).

```c
bool ping_rcv(struct sk_buff *skb)
{
    struct sock *sk;
    struct net *net = dev_net(skb->dev);
    struct icmphdr *icmph = icmp_hdr(skb);

    /* We assume the packet has already been checked by icmp_rcv */

    pr_debug("ping_rcv(skb=%p,id=%04x,seq=%04x)\n",
    	 skb, ntohs(icmph->un.echo.id), ntohs(icmph->un.echo.sequence));

    /* Push ICMP header back */
    skb_push(skb, skb->data - (u8 *)icmph);

    sk = ping_lookup(net, skb, ntohs(icmph->un.echo.id));
    if (sk) {
    	struct sk_buff *skb2 = skb_clone(skb, GFP_ATOMIC);

    	pr_debug("rcv on socket %p\n", sk);
    	if (skb2)
    		ping_queue_rcv_skb(sk, skb2);
    	sock_put(sk);
    	return true;
    }
    pr_debug("no socket, dropping\n");

    return false;
}
```

`ping_rcv` calls the function `ping_lookup`, which finds the socket by ICMP echo
identifier. If a proper socket is found, then the echo reply will be sent to it.

Next, let's explore the magic `ping_lookup`. (Debug log and code related to IPv6
has been ignored.)

```c
static struct sock *ping_lookup(struct net *net, struct sk_buff *skb, u16 ident)
{
    struct hlist_nulls_head *hslot = ping_hashslot(&ping_table, net, ident);
    struct sock *sk = NULL;
    struct inet_sock *isk;
    struct hlist_nulls_node *hnode;
    int dif = skb->dev->ifindex;

    read_lock_bh(&ping_table.lock);

    ping_portaddr_for_each_entry(sk, hnode, hslot) {
    	isk = inet_sk(sk);

    	if (isk->inet_num != ident)
    		continue;

    	if (skb->protocol == htons(ETH_P_IP) &&
    	    sk->sk_family == AF_INET) {

    		if (isk->inet_rcv_saddr &&
    		    isk->inet_rcv_saddr != ip_hdr(skb)->daddr)
    			continue;
    	} else {
    		continue;
    	}

    	if (sk->sk_bound_dev_if && sk->sk_bound_dev_if != dif)
    		continue;

    	sock_hold(sk);
    	goto exit;
    }

    sk = NULL;
exit:
    read_unlock_bh(&ping_table.lock);

    return sk;
}
```

Obviously, an important structure used is a hash table called `ping_table`.

```c
struct ping_table {
	struct hlist_nulls_head	hash[PING_HTABLE_SIZE];
	rwlock_t		lock;
};

static struct ping_table ping_table;
```

The key is calculated by the following function. Basically it is a hash value of
`(net, ICMP echo id, a 32-bit mask)`.

```c
static inline u32 ping_hashfn(const struct net *net, u32 num, u32 mask)
{
	u32 res = (num + net_hash_mix(net)) & mask;

	pr_debug("hash(%u) = %u\n", num, res);
	return res;
}
```

The value is a list of `socket` and can be gotten by `ping_hashslot`.

```c
static inline struct hlist_nulls_head *ping_hashslot(struct ping_table *table,
                                                     struct net *net, unsigned int num)
{
	return &table->hash[ping_hashfn(net, num, PING_HTABLE_MASK)];
}
```

After `ping_lookup` gets the list of socket by calling `ping_hashslot`, it
iterates through the list and find the right socket by checking 3 conditions:

* Is the received identifier same as the one sent out?
* Is the destination address of reply same as the source address of the request?
* Is the device where the reply is received same as the one where the request
  is sent out?

Basically, the first socket that matches all 3 conditions will be returned by
`ip_lookup`. Note that sequence number is not in the picture. In other words,
Linux kernel doesn't match the sequence number when receiving an ICMP packet. It
is up to the user space program (`ping`) to match the sequence number.

# Lab Time!

{: .box-note}
Code used in the labs can be found [here](https://github.com/hechaoli/nf_icmp).

## Part 1
At this point, we know the answer of previous question. If two ICMP echo
requests with the same id are sent at the same time, then when the reply is
received, it will be delivered to the first socket in the list that matches all
3 conditions. That means it can be delivered to a wrong socket when the replies
are received out of order because the kernel doesn't match the sequence number.
Consider the following case:

* At time 1, program 1 sends an ICMP echo request with id = 1, seq = 1 through
  interface 0. The socket is appended to the list in `ping_table` with key `k`.
* At time 2, program 2 sends an ICMP echo request with id = 1, seq = 2 through
  interface 0. The socket is also appended to the same list in `ping_table`.
* At time 3, ICMP echo reply with id = 1, seq = 2 is received. Since program
  1's socket appears first in the list, the reply is delivered to program 1.
* At time 4, ICMP echo reply with id = 1, seq = 1 is received and is delivered
  to program 2.

Apparently, in this case, the user program has to match the sequence number.

Let's do an experiment to see the actual behavior of the `ping` in `iputils`. We
will write a kernel module which modifies the echo reply before it is sent to
peer with the help of [netfilter](https://www.netfilter.org/). In the fist part
of this experiment, we only modify the sequence number.

### Environment
```bash
$ uname -a
Linux hechaol-ubuntu 4.15.0-34-generic #37~16.04.1-Ubuntu SMP Tue Aug 28 10:44:06 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

### Code

```c
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/netfilter.h>
#include <linux/netfilter_ipv4.h>
#include <linux/icmp.h>
#include <linux/ip.h>

unsigned int hook_func(void *priv, struct sk_buff *skb,
                       const struct nf_hook_state *state)
{
  struct iphdr *ip_header = (struct iphdr *)skb_network_header(skb);
  if (ip_header->protocol != IPPROTO_ICMP) {
    return NF_ACCEPT;
  }
  struct icmphdr *icmp_header = (struct icmphdr *)(ip_header + 1);
  if (!icmp_header || icmp_header->type != ICMP_ECHOREPLY) {
    return NF_ACCEPT;
  }
  unsigned int data_size = skb->len - sizeof(struct iphdr) - sizeof(struct icmphdr);
  if (data_size == 0) {
    return NF_ACCEPT;
  }
  uint8_t* data = (uint8_t *)(icmp_header + 1);
  printk(KERN_INFO "Received ICMP packet: id = %d, seq = %d, data_size = %d\n",
         icmp_header->un.echo.id, icmp_header->un.echo.sequence, data_size);
  icmp_header->un.echo.sequence = htons(123);
  return NF_ACCEPT;
}

//Called when module loaded using 'insmod'
int init_module()
{
  printk(KERN_INFO "Loading ICMP hook module\n");
  nfho.hook = hook_func;           //Function to call when conditions below met
  nfho.hooknum = 4;                //NF_IP_POST_ROUTING (For some reason the macro is not found)
  nfho.pf = PF_INET;               //IPV4 packets
  nfho.priority = NF_IP_PRI_FIRST; //set to highest priority over all other hook functions
  nf_register_net_hook(&init_net, &nfho);

  printk(KERN_INFO "Loaded ICMP hook module\n");
  return 0;                             //return 0 for success
}

//Called when module unloaded using 'rmmod'
void cleanup_module()
{
  printk(KERN_INFO "Removing ICMP hook module\n");
  nf_unregister_net_hook(&init_net, &nfho);
  printk(KERN_INFO "Removed ICMP hook module\n");
}
```

The purpose of the module is to change the sequence number of all ICMP echo
replies to `123`.

Makefile used to compile the module:
```
obj-m += nf_icmp.o

all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

To compile and install this module:

```bash
$ make
$ sudo insmod nf_icmp.ko
```

To confirm that the module is loaded successfully:

```bash
$ dmesg | tail -2
[48760.508495] Loading ICMP hook module
[48760.518615] Loaded ICMP hook module
```

Next we ping this machine from another machine:

![Ping Lab Part 1 Linux](/img/ping_lab_part_1_linux.png)

From this result, we see that the `ping` program of the version shown above can
detect duplicate ICMP echo replies, but it can't figure out that the sequence of
the reply doesn't match the request.

Echo replies captured by Wireshark:

![Ping Lab Part 1 Wireshark 1](/img/ping_lab_part_1_wireshark_1.png)

Notice that when the sequence number is changed, the old checksum becomes
incorrect.

Interestingly, `ping` provided by MacOS won't accept this change. Result
from my laptop:

![Ping Lab Part 1 Mac](/img/ping_lab_part_1_mac.png)

However, in this case we don't know whether MacOS `ping` dropped the replies due
to incorrect checksum or sequence number. To verify it, we have to fix the
checksum after modifying the sequence number.

Add a function to calculate checksum:
```c
uint16_t cal_checksum(const uint8_t *buf, uint32_t len) {
    const uint16_t *w = (uint16_t *)buf;
    uint16_t answer;
    int sum = 0;
    int nleft = len;

    while (nleft > 1)  {
        sum += *w++;
        nleft -= 2;
    }

    /* mop up an odd byte, if necessary */
    if (nleft == 1)
        sum += htons(((*w) & 0xFF) << 8);

    /*
     * add back carry outs from top 16 bits to low 16 bits
     */
    sum = (sum >> 16) + (sum & 0xffff); /* add hi 16 to low 16 */
    sum += (sum >> 16);         /* add carry */
    answer = ~sum;              /* truncate to 16 bits */
    return (answer);
}
```

Call it after modifying sequence number:

```c
  icmp_header->un.echo.sequence = htons(123);
  icmp_header->checksum = 0;
  icmp_header->checksum = cal_checksum((uint8_t *)icmp_header, skb->len - sizeof(struct iphdr));
```

Then compile the module and reinstall it:
```bash
$ make
$ sudo rmmod nf_icmp
$ sudo insmod nf_icmp.ko
```

Ping again from Mac:

![Ping Lab Part 1 Mac](/img/ping_lab_part_1_mac_1.png)

This time the result is similar to the `ping` in `iputils`, which means though
MacOS `ping` rejects replies with incorrect checksum, it doesn't check the
sequence number either.

## Part 2
Now you may ask, what if they have the same id and same sequence number? In this
case, we still have one field that can distinguish them - **data (payload).**

An ICMP echo request can have payload. And the reply must also contain the same
payload. For example, by default `ping` put current timestamp as the payload so
that when it can figure out RTT (Round Trip Time) from the reply.

At this point, it seems that if two ICMP replies are identical, which means they
have same id, same sequence number and same data, it doesn't matter which reply
is sent to which request socket because there is no difference!

But does `ping` really match data field? Let's do an experiment.

We will reuse code in part 1 but the `hook_func` is slightly different - in
stead of modifying the sequence number, data is modified this time. Checksum is
also fixed.

```c
    (*data)++;
    icmp_header->checksum = 0;
    icmp_header->checksum = cal_checksum((uint8_t *)icmp_header, skb->len - sizeof(struct iphdr));
```

Only the first byte of the data is modified.

Reinstall the module:

```bash
$ make
$ sudo rmmod nf_icmp
$ sudo insmod nf_icmp.ko
```

Ping from Linux machine with 1-bytes payload:
![Ping Lab Part 2 Linux](/img/ping_lab_part_2_linux.png)

![Ping Lab Part 2 Wireshark](/img/ping_lab_part_2_wireshark.png)

Obviously, `ping` in `iputils` doesn't realize that the data is tampered!

Ping from MacOS with 1-byte payload:

![Ping Lab Part 2 Mac](/img/ping_lab_part_2_mac.png)

MacOS `ping` seems to be more intelligent because it can detect the wrong data
in reply.

# Summary
In this article, we have answered the following questions:

* Can two `ping` processes send two ICMP echo requests with same id at the same
  time?

{: .box-note}
**Answer:** No. When using `ping` provided by `iputils`, the id field in ICMP
echo request is always the process id.

* If I write my own `ping` which sends two ICMP echo requests with same id at
  the same time, how does Linux kernel delivers them to the right sockets?

{: .box-note}
**Answer:** Linux kernel identifies an ICMP echo message only by id and
interface where it is sent/received. If two echo requests on the wire have the
same id and is sent through the same interface, then the reply received first
will be sent to the first sender, despite that it might be the reply to the
second sender. It is the responsibility of user program to match other fields in
the reply such as sequence number and payload.

* Can iputils `ping` detect an incorrect checksum?

{: .box-note}
**Answer:** No. But MacOS `ping` can.

* Can iputils `ping` detect an incorrect sequence number?

{: .box-note}
**Answer:** No. If `ping` sends an ICMP echo request with id = 1, seq = 1 but
receives an echo reply with id = 1, seq = 2, it still counts it as the reply of
the request. MacOS `ping` has the same behavior as long as the checksum is
correct.

* Can iputils `ping` detect an incorrect payload?

{: .box-note}
**Answer:** No. But MacOS `ping` can.

# Reference
[1] [RFC 792 INTERNET CONTROL MESSAGE PROTOCOL](https://tools.ietf.org/html/rfc792)<br>
[2] [Linux Source Code](https://elixir.bootlin.com/linux/v4.18.10/source/)
