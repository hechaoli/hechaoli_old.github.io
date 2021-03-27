---
layout: post
title: Always Use always_inline In BPF Programs
tags: [BPF]
---

# TL;DR
Recently, I solved a tricky bug in a BPF program. Because `inline` instead of
`always_inline` was used to declare a function, when the function body grew
big, the compiler decided to not inline it. Then the BPF program was not able
to be loaded due to some verification error. My takeaway is, always use
`always_inline` in BPF programs to avoid surprise unless you know the BPF
verifier well.

# Problem Statement

We have a BPF program like this:

```c
struct cb_space {
  //...
};

static int inline foo_internal(struct __sk_buff* skb, struct cb_space* cb) {
  //... Use skb
  //... Use cb
}

SEC("classifier/foo")
int foo(struct __sk_buff* skb) {
  return foo_internal(skb, (struct cb_space*)&(skb->cb));
}
```

In one recent change, 2 lines were added to `foo_internal`. After that we got
an error from BPF verifier, which caused a BPF program load failure. I spent a
lot time debugging this issue. It is intuitive to suspect the two newly added
lines when everything works perfectly fine without them. The true devil hides
really well because I focused only on the function body but not the function
declaration.

![Hidden Cat](/img/hidden_cat.jpg){: .mx-auto.d-block :}

<div align="center">The root cause is like the hidden cat. Have you found it?</div>

# Debug
The error thrown by BPF verifier is:
```
libbpf: Error loading ELF section .BTF: -22. Ignored and continue.
libbpf: load bpf program failed: Permission denied
libbpf: -- BEGIN DUMP LOG ---
libbpf:
0: (bf) r2 = r1
1: (07) r2 += 48
2: (85) call pc+3
caller:
 R10=fp0,call_-1
callee:
 frame1: R1=ctx(id=0,off=0,imm=0) R2=ctx(id=0,off=48,imm=0) R10=fp0,call_2
6: (bf) r7 = r2
7: (bf) r6 = r1
8: (b7) r1 = 0
9: (63) *(u32 *)(r10 -8) = r1
10: (b7) r1 = 1
11: (63) *(u32 *)(r10 -4) = r1
12: (bf) r2 = r10
13: (07) r2 += -4
14: (18) r1 = 0xffff881c41e62c00
16: (85) call bpf_map_lookup_elem#1
17: (15) if r0 == 0x0 goto pc+3
 frame1: R0=map_value(id=0,off=0,ks=4,vs=8,imm=0) R6=ctx(id=0,off=0,imm=0) R7=ctx(id=0,off=48,imm=0) R10=fp0,call_2 fp-8=mmmm0000
18: (79) r1 = *(u64 *)(r0 +0)
 frame1: R0=map_value(id=0,off=0,ks=4,vs=8,imm=0) R6=ctx(id=0,off=0,imm=0) R7=ctx(id=0,off=48,imm=0) R10=fp0,call_2 fp-8=mmmm0000
19: (07) r1 += 1
20: (7b) *(u64 *)(r0 +0) = r1
 frame1: R0=map_value(id=0,off=0,ks=4,vs=8,imm=0) R1_w=inv(id=0) R6=ctx(id=0,off=0,imm=0) R7=ctx(id=0,off=48,imm=0) R10=fp0,call_2 fp-8=mmmm0000
21: (71) r1 = *(u8 *)(r7 +1)
dereference of modified ctx ptr R7 off=48 disallowed
```

At first, I thought it was due to invalid memory access so I carefully inspect
each and every memory access in both the newly added lines and the rest of the
function body. However, nothing illegal was found.

Finally, the following lines in the log dump caught my attention:

```
caller:
 R10=fp0,call_-1
callee:
 frame1: R1=ctx(id=0,off=0,imm=0) R2=ctx(id=0,off=48,imm=0) R10=fp0,call_2
```

If `foo_internal` is an inline function, how come there is a caller and a
callee? Then I realized that the newly added lines might have caused the
function body to be bigger than the inline threshold of the compiler thus the
function is not inlined anymore. At this point, I still wasn't sure if this was
a problem but at least a function call to `foo_internal` was not expected -- it
was supposed to be inlined. Once I changed `inline` to `always_inline`, the
problem was solved.

But why?

![Question Cat](/img/question_cat.jpg){: .mx-auto.d-block :}

# Root Cause

To better understand the root cause, let’s write a minimal BPF program that can’t be loaded.

```c
#include <linux/bpf.h>
#include <bpf/helpers/bpf_helpers.h>

struct cb_space {
  __u32 data;
};

static int __attribute__((noinline)) foo_func_internal(
    struct __sk_buff* skb,
    struct cb_space* cb) {
  if (cb->data != 0) {
    return 1;
  }
  return -1;
}

SEC("classifier/foo_func")
int foo_func(struct __sk_buff* skb) {
  return foo_func_internal(skb, (struct cb_space*)&(skb->cb));
}

unsigned int _version SEC("version") = 1;
char _license[] SEC("license") = "GPL";
```

Note that I am using `__attribute__((noinline))` to force it to be a function
call. When we load above BPF program, we will get the following error:

```
libbpf: load bpf program failed: Permission denied
libbpf: -- BEGIN DUMP LOG ---
libbpf:
0: (07) r1 += 48
1: (85) call pc+1
caller:
 R10=fp0,call_-1
callee:
 frame1: R1=ctx(id=0,off=48,imm=0) R10=fp0,call_1
3: (61) r1 = *(u32 *)(r1 +0)
dereference of modified ctx ptr R1 off=48 disallowed
```

Interestingly, if we change this program to the following semantically
identical one, then it can be loaded successfully.

```c
static int __attribute__((noinline)) foo_func_internal(struct __sk_buff* skb) {
  struct cb_space* cb = (struct cb_space*)&(skb->cb);
  if (cb->data != 0) {
    return 1;
  }
  return -1;
}

SEC("classifier/foo_func")
int foo_func(struct __sk_buff* skb) {
  return foo_func_internal(skb);
}
```

Apparently, the problem is the access (or dereference) of function parameter
`cb`. The last line of the error log has already told us this in its own
language. I found [this error
message](https://elixir.bootlin.com/linux/v4.16.18/source/kernel/bpf/verifier.c#L1655)
in BPF code but I was daunted by the 8000-line verifier code so I didn’t trace
the call stack.

“dereference of modified ctx ptr R1 off=48 disallowed” What does it mean?

My understang is, in the function call to `foo_func_internal`, `skb` is a "ctx
ptr" since it is a parameter which passes context between functions and it is a
pointer. Therefore `cb`(i.e. `skb->cb`) is a "modified ctx ptr" because it is
the context pointer `skb` plus a 48-byte offset (See [definition of
`__sk_buff`](https://elixir.bootlin.com/linux/v4.5/source/include/uapi/linux/bpf.h#L301)).
In `foo_func_internal`, we dereference the value of `cb` (`cb->data` is
equivalent to `*cb`) and that is disallowed.

In the second program, `cb` is a local pointer variable instead of a "modified
ctx ptr" thus it can be dereferenced.  The reason is, to `foo_func_internal` in
the first program, `cb` is `skb` plus a (unknown) non-zero offset. The verifier
doesn’t want to risk dereferencing it in case it points to some invalid memory
address beyond the skb struct boundary. However, to `foo_func_internal` in the
second program, `cb` is `skb` plus a known offset 48 bytes and a read size 4
bytes, which is within `__sk_buff` struct. It knows that this pointer is safe
to dereference as long as skb is valid.


![Hidden Cat Answer](/img/hidden_cat_answer.jpg){: .mx-auto.d-block :}

<div align="center">Here it is!</div>
