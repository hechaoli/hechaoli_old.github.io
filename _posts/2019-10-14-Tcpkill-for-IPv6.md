---
layout: post
title: Tcpkill for IPv6
tags: [Network, TCP]
---

# Tcpkill
*tcpkill* is part of [*dsniff*](https://monkey.org/~dugsong/dsniff/), a
collection of tools for network auditing and penetration testing. It can be
used to kill specified in-progress TCP connections.

# How does it work?
The way *tcpkill* works is essentially a [TCP reset
attack](https://en.wikipedia.org/wiki/TCP_reset_attack). It leverages two
libraries -- [*libpcap*](https://github.com/the-tcpdump-group/libpcap), a
system-independent interface for user-level packet capture, and
[*libnet*](https://github.com/libnet/libnet), an API to help with the
construction and injection of network packets. 

*tcpkill* uses *libpcap* to capture packets on an interface and then for each
received TCP packet, send a TCP packet with `RST` flag back to the sender using
*libnet* to reset the TCP connection. Because it can get the `SEQ` and `ACK`
number in the received TCP headers, it does much better than a [blind reset
attack](https://tools.ietf.org/html/rfc5961#section-3). As we know, **the next
`SEQ` sending to peer should be the last `ACK` received from the peer**. 

However, this is still a guess because if the `RST` packet injected by *libnet*
comes after the real reply, then the RST packet’s `SEQ` number will be outside
the receive window and be discarded. To increase the success rate of the
attack, tcpkill has an option to specify how many `RST` packets to send (3 by
default) for each received packet. And the interval of the `SEQ` numbers
between two consecutive `RST` packets is the window size. 

![TCP kill](/img/tcpkill.jpg)

# IPv6 Support
Because of the age of the tool (19 years old!), it doesn’t support IPv6.
*tcpkill* will print an error message and exit if it doesn’t detect any IPv4
address on the interface. But fortunately, its two dependencies -- *libpcap*
and *libnet* -- both support IPv6. So we only need to adapt the original code
and make it support IPv6. See [the adapted code on
Github](https://github.com/hechaoli/tcpkill6).

# Test
First, let's start an iperf connection between a server and a client. Then, we
start *tcpkill* on the server host with capture filter to be “port 5201”

```
$ make tcpkill6 && sudo ./tcpkill6 -i eth0 port 5201
```

On the client side, we will see something like:
```
$ iperf3 -c <server IPv6>
Connecting to host <server IPv6>, port 5201
[  4] local <client IPv6> port 44204 connected to <server IPv6> port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec   736 KBytes  6.03 Mbits/sec    0    178 KBytes
[  4]   1.00-2.00   sec  25.8 MBytes   216 Mbits/sec    0   8.78 MBytes
[  4]   2.00-3.00   sec  97.5 MBytes   818 Mbits/sec    0   32.2 MBytes
[  4]   3.00-4.00   sec  96.2 MBytes   807 Mbits/sec    0   32.2 MBytes
iperf3: error - unable to write to stream socket: Connection reset by peer
```

Similarly, on the server side:
```
$ iperf3 -s
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
Accepted connection from <client IPv6>, port 44202
[  5] local <server IPv6> port 5201 connected to <client IPv6> port 44204
[ ID] Interval           Transfer     Bandwidth
[  5]   0.00-1.00   sec   165 KBytes  1.35 Mbits/sec
[  5]   1.00-2.00   sec  8.61 MBytes  72.2 Mbits/sec
[  5]   2.00-3.00   sec  72.7 MBytes   610 Mbits/sec
[  5]   3.00-4.00   sec  95.7 MBytes   803 Mbits/sec
iperf3: error - unable to read from stream socket: Connection reset by peer
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```
