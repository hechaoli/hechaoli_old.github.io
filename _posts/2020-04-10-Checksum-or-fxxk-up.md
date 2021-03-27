---
layout: post
title: Checksum or fxxk-up?
cover-img: /img/checksum.jpg
tags: [Network, BPF, Linux Kernel]
---

# tl;dr
If you don’t want to shoot yourself in the foot, then don’t mess up with
checksum in any way. But if you’d like to see a counterexample, keep reading
this note written by lame me.

# Background
I have a ingress TC BPF program that modify a few bits in IP header and a TCP
flag of certain incoming packets. It worked really well functionally. Until...

# dmesg filed a complaint
While testing the program, the following message was found in `dmesg`:

```
[1216218.835883] <unknown>: hw csum failure
[1216218.843767] CPU: 11 PID: 3627719 Comm: ThriftIO15 Not tainted 4.16.18-210_fbk23_5358_gf2838c074351 #210
[1216218.862917] Hardware name: Quanta Twin Lakes MP/Twin Lakes Passive MP, BIOS F09_3A12 10/08/2018
[1216218.880669] Call Trace:
[1216218.885910]  <IRQ>
[1216218.890287]  dump_stack+0x46/0x68
[1216218.897272]  __skb_checksum_complete+0xb0/0xc0
[1216218.906516]  tcp_v6_do_rcv+0x12b/0x3d0
[1216218.914370]  tcp_v6_rcv+0xad0/0xb00
[1216218.921701]  ip6_input_finish+0xbf/0x460
[1216218.929903]  ip6_input+0x80/0x90
[1216218.936714]  ipv6_rcv+0x358/0x4c0
[1216218.943698]  ? tcf_classify+0x87/0x120
[1216218.951551]  __netif_receive_skb_core+0x541/0xa40
[1216218.961315]  netif_receive_skb_internal+0x24/0xb0
[1216218.971080]  napi_gro_receive+0xba/0xe0
[1216218.979105]  mlx5e_handle_rx_cqe+0x83/0xd0
[1216218.987649]  mlx5e_poll_rx_cq+0xc8/0x920
[1216218.995840]  mlx5e_napi_poll+0xa1/0xc30
[1216219.003866]  net_rx_action+0x128/0x340
[1216219.011718]  __do_softirq+0xd4/0x2ad
[1216219.019219]  irq_exit+0xa5/0xb0
[1216219.025850]  do_IRQ+0x7d/0xc0
[1216219.032132]  common_interrupt+0xf/0xf
[1216219.039798]  </IRQ>
```

That was how I got pushed into the checksum abyss. 

# Root cause and solution
The message indicated a checksum error. I asked *tcpdump* to double check but
he said checksums are fine. Then I realized that the **ingress TC BPF program
runs after tcpdump and before kernel receiving the packets**. That can explain
why *tcpdump* didn't have a problem with checksum but kernel complained.

The root cause is clear: the ingress BPF program meddled with bits in TCP/IP
headers but didn't fix the checksum. The solution also seems to be simple: JUST
FIX THE CHECKSUM.

Well, I will save some time for both of us by not complaining about the pain in
finding the needle in a haystack full of checksum-related API such as
`bpf_l3_csum_replace`, `bpf_l4_csum_replace`, `bpf_csum_diff and
`bpf_csum_update`.

What more frustrating was, in the end, a simple `BPF_F_RECOMPUTE_CSUM` flag
passed to `bpf_skb_store_bytes` (which is used to modify bits) did the trick.

# How does checksum work? 
Before digging into more details, let’s have a look at how checksum works. The
following diagram is a very good illustration.

![checksum](/img/checksum.jpg){: .mx-auto.d-block :}

(Image credit: https://www.geeksforgeeks.org/error-detection-in-computer-networks/)

In this post, to avoid confusion, I will use the term **"csum"** to refer to
the **"sum"** in the diagram above and use the term **"checksum"** to refer to
1's complement of "csum". As you can see, as long as the data is not tampered,
`~(csum + checksum) == 0`.

# One more puzzle to solve
While debugging, I found one thing interesting - though the error was printed,
the packet wasn’t discarded so that the processing continued. Why?

Since some inline functions are not shown in the stack trace above, I will just
"flatten" the logic of the call stack here:

```
tcp_v6_do_rcv() {
  // Checking checksum is unnecessary
  if (skb_csum_unnecessary(skb)) {
     nothing;
  } else {
     // Re-do checksum for whole skb
     csum = skb_checksum(skb, 0, skb->len, 0);
     // Check csum against skb->csum. If skb is intact, sum should be 0
     sum = csum_fold(csum_add(skb->csum, csum));

     if (sum == 0)
       Print "hw csum failure" to dmesg and dump stack trace
     else 
       Checksum fails, throw the skb away
  }
  // Continue processing skb
}
```

When the error is printed, the following conditions must meet:
1. `skb_csum_unnecessary` returns `false`.
2. If we calculate a checksum of the `skb` and check it against `skb->csum`, it
   should pass. That is, `csum_add(skb->csum, skb_checksum(skb))` should return
   zero.

When `skb_csum_unnecessary` returns `false`, it means [`skb->csum_valid` is
false](https://github.com/torvalds/linux/blob/24085f70a6e1b0cb647ec92623284641d8270637/include/linux/skbuff.h#L3900),
which in turn means [`skb->csum_valid` is not set to 1 during
`skb_checksum_init`](https://github.com/torvalds/linux/blob/24085f70a6e1b0cb647ec92623284641d8270637/include/linux/skbuff.h#L3998).
And that’s because [the original `skb->csum` and `psum` don’t add up to
zero](https://github.com/torvalds/linux/blob/24085f70a6e1b0cb647ec92623284641d8270637/include/linux/skbuff.h#L3997).
Also note that [`skb->csum` is set to `psum` after
`skb_checksum_init`](https://github.com/torvalds/linux/blob/24085f70a6e1b0cb647ec92623284641d8270637/include/linux/skbuff.h#L4003).

In human language, this piece of code means,

*"If I am running, that means checking the `skb->csum` provided by hardware
(NIC) against `psum` (I will talk about it later) must have failed. Screw you,
hardware! (error printed) But you know what, I recalculated the skb’s checksum
and it's indeed correct. Let's forget about hardware error and continue!"*

The hardware was undoubtedly innocent, it was my BPF program who commit the
crime. So even if the packets were not discarded and connectivity was not
affected, I still fixed the guilty BPF program to prove hardware’s innocence.

While I was celebrating yet another closed case, I had no idea what the
criminal was up to.

![hidden bear](/img/hidden_bear.jpg){: .mx-auto.d-block :}

# Checksum is fxxked up, again?

While deploying the BPF program, I found that `InCsumErrors` kept increasing
once the program is attached.

```
$ netstat -s | grep InCsumErrors
    InCsumErrors: 101
```

# Debugging
I can't talk about kernel debugging without mentioning my best buddy -
*bpftrace*. This time again, with its help, I was able to narrow the issue down
to the following stack:

```
        __skb_checksum_complete+1
        tcp_v6_do_rcv+293
        tcp_v6_rcv+2762
        ip6_input_finish+186
        ip6_input+128
        ipv6_rcv+853
        __netif_receive_skb_core+1312
        netif_receive_skb_internal+36
        napi_gro_receive+183
        mlx5e_handle_rx_cqe+125
        mlx5e_poll_rx_cq+194
        mlx5e_napi_poll+161
        net_rx_action+290
        __softirqentry_text_start+207
        irq_exit+165
        do_IRQ+119
        ret_from_intr+0
        cpuidle_enter_state+176
        do_idle+260
        cpu_startup_entry+25
        secondary_startup_64+165
```

The stack is same as previous issue. However, this time, **there is no error
message in dmesg and the packet IS discarded**, causing a retransmission. In
*tcpdump*, what I saw was the following sequence:

1. A TCP SYN packet arrives but it doesn’t get a SYN-ACK.
2. After one second, a retransmitted SYN packet arrives and got a SYN-ACK.
3. The connection is successfully established.

# Guessing & Praying
After hours of browsing kernel code without any major findings, I decided to
use the not-so-scientific yet effective way of debugging - try out different
things hoping to find a working solution and pray to god to increase the
success rate.

![Sheldon Pray](/img/sheldon_pray.jpg){: .mx-auto.d-block :}

<div align="center">(An example of scientist praying to god when things not working)</div>

Then I noticed that the issue only happened when the TCP header is modified.
Even though `skb->csum` is fixed accordingly using `BPF_F_RECOMPUTE_CSUM` after
the modification. I decided to see what would happen if the `skb->csum` is not
fixed after TCP header is modified. And, to my surprise, it works! 

The following table shows things I tried and their results:

![Things I tried](/img/checksum_things_I_tried.png){: .mx-auto.d-block :}

<div align="center">Can you find some patterns?</div>

At least two questions came out of this table:
* Why "hw csum failure" if NOT recomputing csum for IP header change?
* Why "TcpInCsumErrors" if recomputing csum for TCP header change?

The next goal is to answer these two questions and make sure I discovered a
solution instead of a coincidence.

# Chasing the answers
`TcpInCsumErrors` error happens because [`skb_checksum_init` returns non-zero
value](https://github.com/torvalds/linux/blob/24085f70a6e1b0cb647ec92623284641d8270637/net/ipv6/tcp_ipv6.c#L1569).
The call stack is a bit involved. So let’s flatten the logic again (Irrelevant
code has been omitted). 

![skb_checksum_init](/img/skb_checksum_init.png){: .mx-auto.d-block :}

<div align="center">One of the most involved pieces of code I've seen</div>

I marked the two error cases in above code. We need to be very careful with the
logic so that we can avoid both errors while messing up with checksums.

## What is ip6_compute_pseudo?

It calculates the csum of [TCP pseudo
header](http://www.tcpipguide.com/free/t_TCPChecksumCalculationandtheTCPPseudoHeader-2.htm).
The result is the psum we’ve seen above.

![IP pseudo header](/img/ip_pseudo_header.png){: .mx-auto.d-block :}

<div align="center">Image credit:
http://www.tcpipguide.com/free/t_TCPChecksumCalculationandtheTCPPseudoHeader-2.htm</div>

## What triggers "hw csum failure"?
The following conditions must all be met:
1. `~(psum + skb->csum) != 0` That is, checking of the `skb->csum` against `psum` fails.
2. `~(skb_checksum(skb, 0, skb->len, 0) + psum) == 0` That is, checking of the
   newly calculated csum of `skb` against `psum` passes.

## What triggers "TcpInCsumErrors"?
The following conditions must all be met:
1. `~(psum + skb->csum) != 0` That is, checking of the `skb`'s csum against `psum` fails.
2. `~((skb_checksum(skb, 0, skb->len, 0) + psum) != 0` That is, checking of the
   newly calculated csum of `skb` against `psum` also fails.

## Something is strange
You may have noticed something strange. The `skb->csum` is checked against
`psum`, the checksum of the pseudo header. Why is that? Shouldn't it be checked
against the checksum of the whole skb using something like `~(skb->csum +
checksum(skb)) == 0`?

I will try my best to explain it. The key is, like the diagram above shows, TCP
header also has a checksum.

![TCP header format](/img/tcp_header_format.jpg){: .mx-auto.d-block :}

And the formula of calculating TCP header checksum is

```
tcp_hdr_checksum = ~(pseudo_hdr + tcp_hdr_with_checksum_0)
```

Note:
* I will use "+" to denote "csum add", the algorithm shown in the diagram in
  *"How does checksum work?"* section.
* Payload is intentionally omitted because it doesn’t matter.

`skb->csum` is a [16-bit csum calculated by
NIC](https://github.com/torvalds/linux/blob/24085f70a6e1b0cb647ec92623284641d8270637/drivers/net/ethernet/mellanox/mlx5/core/en_rx.c#L950)
using the whole packet. That is,

```
skb->csum  = ip_hdr + tcp_hdr
```

Note:
* When doing "csum add", both IP and TCP headers are "sliced" into 16-bit
  chunks which are added up / folded together.
* Because IP header, pseudo header and TCP header are all 16-bit aligned, the
  two formulas can be rewritten to 
  
```
tcp_hdr_checksum = ~(pseudo_hdr_0_15 + pseudo_hdr_16_31 + ... +
                   tcp_hdr_0_15 + tcp_hdr_16_31 + ... + 0x0000 (checksum) + ...)
```

If we do "~" operation on both sides, we get

```
~tcp_hdr_checksum = pseudo_hdr_0_15 + pseudo_hdr_16_31 + ... +
                    tcp_hdr_0_15 + tcp_hdr_16_31 + ... + 0x0000 (checksum) + ...
```

Then  we add `tcp_hdr_checksum` on both sides:

```
~tcp_hdr_checksum + tcp_hdr_checksum
= pseudo_hdr_0_15 + pseudo_hdr_16_31 + ... +
  tcp_hdr_0_15 + tcp_hdr_16_31 + ... + tcp_hdr_checksum + ...
```

Because we know that `~tcp_hdr_checksum + tcp_hdr_checksum = 0xFFFF`. So

```
pseudo_hdr_0_15 + pseudo_hdr_16_31 + ... +
tcp_hdr_0_15 + tcp_hdr_16_31 + ... + tcp_hdr_checksum + ... = 0xFFFF
```

Thus,

```
tcp_hdr_0_15 + tcp_hdr_16_31 + ... + tcp_hdr_checksum + ...
= 0xFFFF - (pseudo_hdr_0_15 + pseudo_hdr_16_31 + ... )
= 0xFFFF - psum
```

We can plug this into `skb->csum`

```
skb->csum  = ip_hdr_0_15 + ip_hdr_16_31 + ... +
             tcp_hdr_0_15 + tcp_hdr_16_31 + ... + tcp_hdr_checksum + ...

           = ip_hdr_0_15 + ip_hdr_16_31 + ... + 0xFFFF - psum
```

Keep this formula in mind and I will answer the two questions above.

## Why "hw csum failure" if NOT recomputing csum for IP header change?

(For simplicity, let's assume TCP header is not changed.)

In the kernel network stack, after IP header is processed, [its csum is pulled
out of or subtracted from
`skb->csum`](https://github.com/torvalds/linux/blob/24085f70a6e1b0cb647ec92623284641d8270637/net/ipv6/ip6_input.c#L410).
Then `skb->csum` becomes

```
skb->csum = (ip_hdr_0_15 + ip_hdr_16_31 + ... + 0xFFFF - psum) -
            (ip_hdr_0_15 + ip_hdr_16_31 + ...)

          = 0xFFFF - psum
```

Now we know why in `skb_checksum_init`, `skb->csum` is checked against `psum` --
because **it's the only variable left in the formula**.

However, if the IP header is changed (say, a few bits in `ip_hdr_16_31` are
flipped) but `skb->csum` is not changed accordingly, then it becomes

```
skb->csum = skb->csum - (ip_hdr_0_15 + ip_hdr_16_31' + ...)

          = (ip_hdr_16_31 - ip_hdr_16_31') + 0xFFFF - psum
```

* The first condition for "hw csum failure" is `~(psum + skb->csum) != 0` or
  `(psum + skb->csum) != 0xFFFF`. And the math is

  ```
  psum + skb->csum = psum + (ip_hdr_16_31 - ip_hdr_16_31') + 0xFFFF - psum
                   = (ip_hdr_16_31 - ip_hdr_16_31') + 0xFFFF
  ```

  The result is apparently not `0xFFFF`. So the first condition is met.

* The second condition for "hw csum failure" is `~(skb_checksum(skb, 0,
  skb->len, 0) + psum) == 0` or `skb_checksum(skb, 0, skb->len, 0) + psum ==
  0xFFFF`.

  `skb_checksum(skb, 0, skb->len, 0)` is to re-calculate csum of skb, which now
  only has TCP header. It will be

  ```
    skb_checksum(skb, 0, skb->len, 0)
  = tcp_hdr_0_15 + tcp_hdr_16_31 + ... + tcp_hdr_checksum + ...
  = 0xFFFF - psum
  ```
  
  Therefore,

  ```
    skb_checksum(skb, 0, skb->len, 0) + psum
  = 0xFFFF - psum +  psum
  = 0xFFFF
  ```

  The second condition is also met. The question is answered.

## Why “TcpInCsumErrors” if recomputing csum for TCP header change?
(For simplicity, let’s assume IP header is not changed.)

Note that when we use `BPF_F_RECOMPUTE_CSUM`, it only updates `skb->csum` but
NOT `tcp_hdr_checksum`. Suppose a few bits in `tcp_hdr_0_15` are flipped. When
`skb_checksum_init` is called, `skb->csum` is in fact `skb->csum'`. 

```
skb->csum’ = tcp_hdr_0_15' + tcp_hdr_16_31 + ... + tcp_hdr_checksum + ...

           = tcp_hdr_0_15' - tcp_hdr_0_15 +
             tcp_hdr_0_15 + tcp_hdr_16_31 + ... + tcp_hdr_checksum + ...

           = tcp_hdr_0_15' - tcp_hdr_0_15 + 0xFFFF - psum
```

* The first condition for "TcpInCsumErrors" is `~(psum + skb->csum') != 0` or
  `(psum + skb->csum') != 0xFFFF`. So the math is

  ```
  psum + skb->csum' = psum + tcp_hdr_0_15' - tcp_hdr_0_15 + 0xFFFF - psum
                    = tcp_hdr_0_15' - tcp_hdr_0_15 + 0xFFFF
  ```
  The result is apparently not `0xFFFF`. So the first condition is met.

* The second condition for "TcpInCsumErrors" is `~(skb_checksum(skb, 0,
  skb->len, 0) + psum) != 0` or `skb_checksum(skb, 0, skb->len, 0) + psum !=
  0xFFFF`.

  `skb_checksum(skb, 0, skb->len, 0)` is to re-calculate csum of skb. It will be
  same as `skb->csum'` so the math is the same as the first condition. In other
  words, the second condition is also met.

As you can see, if we don't fix `skb->csum` when updating `tcp_hdr_0_15`, the
first condition won’t be met and the code will return from the happy ending
path directly.

# Wait, isn’t this a hack?

Yes, I agree. Updating the TCP header but not updating its checksum and
`skb->csum` sounds like a hack. The alternative is:

1. Update bits in TCP header.
2. Update `skb->csum` to reflect the bits change.
3. Update TCP checksum field to reflect the bits change.
4. Update `skb->csum` to reflect the TCP checksum change.

In this way, the diff in the TCP header and the diff in TCP checksum cancel out
each other. The math will be

```
skb->csum’ = tcp_hdr_0_15' + tcp_hdr_16_31 + ... + tcp_hdr_checksum' + ...

           = tcp_hdr_0_15' - tcp_hdr_0_15 +
             tcp_hdr_0_15 + tcp_hdr_16_31 + ... +
             tcp_hdr_checksum - tcp_hdr_checksum + tcp_hdr_checksum' + ...

           = (tcp_hdr_0_15' - tcp_hdr_0_15) +
             (tcp_hdr_checksum' - tcp_hdr_checksum) +
             0xFFFF - psum

           = 0xFFFF - psum
```

Since the result is same as not touching `skb->csum` at all, why not just using
the hack?

# Takeaways
* Do NOT mess up with checksum if you could.
* The person who invented checksum is a genius.
* I am a fool.


