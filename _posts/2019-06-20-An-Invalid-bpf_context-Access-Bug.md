---
layout: post
title: An Invalid bpf_context Access Bug
tags: [Linux, BPF]
---

I got an interesting issue while debugging a BPF program. The BPF program
couldn't be loaded because of an "invalid memory access" error. Though the
access was completely within the valid memory boundary, the verifier was still
not happy about it.

# Problem
The following BPF program does 3 things:

1. Get the first element from a BPF array. The value type is `in6_addr_u64`, a
   struct consisting of 2 64-bit (8-byte) integer.
2. Cast `skops->remote_ip6` to the same struct and do an `&` operation with the
   value from step 1.
3. Read the first 4 bytes of the value from step 2.

```
struct in6_addr_u64 {
  __u64 addr_hi;
  __u64 addr_lo;
};

struct bpf_map_def SEC("maps") ip_masks = {
    .type = BPF_MAP_TYPE_ARRAY,
    .key_size = sizeof(__u32),
    .value_size = sizeof(struct in6_addr_u64),
    .max_entries = 1,
};

SEC("sockops/hechaol_func")
int func(struct bpf_sock_ops* skops) {
  struct in6_addr remote_ip = *(struct in6_addr*)skops->remote_ip6;
  __u32 i = 0;
  struct in6_addr_u64* mask = bpf_map_lookup_elem(&ip_masks, &i);
  if (!mask) {
    return 0;
  }

  struct in6_addr masked_ip = {0};
  struct in6_addr_u64* masked_ip64 = (struct in6_addr_u64*)&masked_ip;
  struct in6_addr_u64* ip64 = (struct in6_addr_u64*)&remote_ip;

  masked_ip64->addr_hi = ip64->addr_hi & mask->addr_hi;
  masked_ip64->addr_lo = ip64->addr_lo & mask->addr_lo;

  if (masked_ip.s6_addr32[0]) {
    return 1;
  }
  return 0;
}
```

`libbpf` returns an error when trying to load this program:
```
libbpf: load bpf program failed: Permission denied
libbpf: -- BEGIN DUMP LOG ---
libbpf:
0: (79) r7 = *(u64 *)(r1 +32)
invalid bpf_context access off=32 size=8
processed 1 insns (limit 1000000) max_states_per_insn 0 total_states 0 peak_states 0 mark_read 0

libbpf: -- END LOG --
libbpf: failed to load program 'sockops/hechaol_func'
libbpf: failed to load object 'hechaol_bpf'
Failed to load bpf object! errno = 13
```

According to the error log, **the 8-byte access to bpf_context at offset 32 is
invalid**. We know that the context is just the function’s parameter - `struct
bpf_sock_ops* skops`. Let’s see what is at its offset 32 (bytes).

```
struct bpf_sock_ops {
	__u32 op;
	union {
		__u32 args[4];		/* Optionally passed to bpf program */
		__u32 reply;		/* Returned by bpf program	    */
		__u32 replylong[4];	/* Optionally returned by bpf prog  */
	};
	__u32 family;
	__u32 remote_ip4;	/* Stored in network byte order */
	__u32 local_ip4;	/* Stored in network byte order */
	__u32 remote_ip6[4];	/* Stored in network byte order */
        ...
}
```

Not surprisingly, offset 32 is `remote_ip6`, the only field we access. Since it
is a 4-size element array of 32-bit (4-byte) integers, the field’s total size
is 4*4 = 16 bytes. **Then why the 8-byte access is invalid given that it is
within the field’s boundary**?

More interestingly, if I add the following lines to the BPF program:
```
  if (remote_ip.s6_addr32[0]) {
    return 1;
  }
```

then the program can be loaded successfully. This line does nothing but one
meaningless 4-byte access to `remote_ip` offset zero.

# Root Cause
One of my coworkers pointed out that **all fields in `struct bpf_sock_ops`
except `bytes_received` and `bytes_acked` must be accessed as `u32` instead of
`u64`.** See [this piece of
code](https://elixir.bootlin.com/linux/v4.16.18/source/net/core/filter.c#L3950)
for more details. Knowing the root cause, the fix is simple - don’t cast the
IP address to `u64` but convert it to `u64` using two u32.
```
__u64 byte0 = skops->remote_ip6[0];
__u64 byte1 = skops->remote_ip6[1];
__u64 byte2 = skops->remote_ip6[2];
__u64 byte3 = skops->remote_ip6[3];

__u64 addr_hi = (byte1 << 32) | byte0;
__u64 addr_lo = (byte3 << 32) | byte2;
```

However, it still doesn’t explain one thing - why can the code pass the
verifier when adding one more 4-byte access to `remote_ip` offset zero?

This puzzle was solved by disassembling the generated BPF object file.

Let’s first disassemble the original BPF program:
```
$ llvm-objdump -S bpf_func.o
Disassembly of section sockops/func:
bpf_func:
       0:       79 17 20 00 00 00 00 00         r7 = *(u64 *)(r1 + 32)
       1:       b7 06 00 00 00 00 00 00         r6 = 0
       2:       63 6a fc ff 00 00 00 00         *(u32 *)(r10 - 4) = r6
       3:       bf a2 00 00 00 00 00 00         r2 = r10
       4:       07 02 00 00 fc ff ff ff         r2 += -4
       5:       18 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00         r1 = 0 ll
       7:       85 00 00 00 01 00 00 00         call 1
       8:       15 00 05 00 00 00 00 00         if r0 == 0 goto +5 <LBB0_3>
       9:       61 01 00 00 00 00 00 00         r1 = *(u32 *)(r0 + 0)
      10:       5f 71 00 00 00 00 00 00         r1 &= r7
      11:       b7 06 00 00 01 00 00 00         r6 = 1
      12:       55 01 01 00 00 00 00 00         if r1 != 0 goto +1 <LBB0_3>
      13:       b7 06 00 00 00 00 00 00         r6 = 0

LBB0_3:
      14:       bf 60 00 00 00 00 00 00         r0 = r6
      15:       95 00 00 00 00 00 00 00         exit
```

The first instruction `r7 = *(u64 *)(r1 + 32)` is **an 8-byte access at offset
32 of r1**, which supposedly is the context (`sock_op`). This is **an invalid
access** according to the verifier and no wonder the load failed.

Next let’s disassemble the the BPF program which has that additional `if`
statement:
```
$ llvm-objdump -S bpf_func.o

Disassembly of section sockops/func:
hechaol_func:
       0:       61 11 20 00 00 00 00 00         r1 = *(u32 *)(r1 + 32)
       1:       b7 02 00 00 00 00 00 00         r2 = 0
       2:       63 2a fc ff 00 00 00 00         *(u32 *)(r10 - 4) = r2
       3:       b7 07 00 00 01 00 00 00         r7 = 1
       4:       b7 06 00 00 01 00 00 00         r6 = 1
       5:       55 01 01 00 00 00 00 00         if r1 != 0 goto +1 <LBB0_2>
       6:       b7 06 00 00 00 00 00 00         r6 = 0

LBB0_2:
       7:       bf a2 00 00 00 00 00 00         r2 = r10
       8:       07 02 00 00 fc ff ff ff         r2 += -4
       9:       18 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00         r1 = 0 ll
      11:       85 00 00 00 01 00 00 00         call 1
      12:       55 00 01 00 00 00 00 00         if r0 != 0 goto +1 <LBB0_4>
      13:       b7 07 00 00 00 00 00 00         r7 = 0

LBB0_4:
      14:       5f 76 00 00 00 00 00 00         r6 &= r7
      15:       bf 60 00 00 00 00 00 00         r0 = r6
      16:       95 00 00 00 00 00 00 00         exit
```

Look at the first instruction, **the 8-byte access becomes a 4-byte access**:
`r1 = *(u32 *)(r1 + 32)`. My guess is, after adding this additional statement,
*LLVM* decides to do some optimization. And it notices that, even though we do
an `&` operation on an 8-byte variable, we only access the lower 4 byte of the
result (`masked_ip.s6_addr32[0]`). So it casts the operand to 4 bytes instead
of 8. If we also access the higher 4 bytes of `masked_ip`
(`masked_ip.s6_addr32[0] && masked_ip.s6_addr32[1]`), then the verifier will
complain again.
