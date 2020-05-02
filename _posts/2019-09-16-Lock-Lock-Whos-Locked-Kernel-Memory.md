---
layout: post
title: Lock Lock. Who's Locked? Kernel Memory
image: /img/padlock.jpg
tags: [Linux, Kernel, BPF]
---

# Background
In a BPF program, we use a `BPF_MAP_TYPE_PERF_EVENT_ARRAY` map to communicate
with userspace. Initially, for our BPF program, we set the locked memory limit
to be infinity:

```
struct rlimit lck_mem = {};
lck_mem.rlim_cur = RLIM_INFINITY;
lck_mem.rlim_max = RLIM_INFINITY;
::setrlimit(RLIMIT_MEMLOCK, &lck_mem);
```

However, in a recent change, we updated the limit to 512MB. After that, our
control plane program failed to start.

After some debugging, I found the failure was because the call to `mmap`
returns `EPERM`, which means "Operation not permitted".

According to [capability man
page](http://man7.org/linux/man-pages/man7/capabilities.7.html), mmap needs
`CAP_IPC_LOCK` capability. After talking to security team, we figured that due
to a recent change, even though our container runs a root, we still need to
request `CAP_IPC_LOCK` capability separately. After using the API provided by
security team, the failure was gone.

# But why?

The problem was solved but there remained one unexplained mystery:

According to [this piece of
code](https://github.com/torvalds/linux/blob/f733c6b508bcaa3441ba1eacf16efb9abd47489f/kernel/events/core.c#L5822-L5823),
`CAP_IPC_LOCK` capability was requested because **locked memory size is larger
than the limit**, which implied that the new limit, 512MB was not enough. While
debugging the issue, I tried increasing the limit to 4x, 7x, even 10x of 512MB,
but the problem was still unsolved.  That means, even 512MB * 10 = 5GB memory
was not enough! How could it even be possible?!

# Keep digging
Since the problem was `locked > lock_limit`, I wrote a *bpftrace* program to print
these 2 values when `perf_mmap` is called:

```
#!/usr/bin/env bpftrace

#include <linux/cred.h>
#include <linux/sched.h>
#include <linux/uidgid.h>
#include <linux/mm_types.h>

kprobe:perf_mmap
{
        $PAGE_SIZE = 4096;
        $PAGE_SHIFT = 12;
        $num_online_cpus = 36;
        $RLIMIT_MEMLOCK = 536870912;

        printf("=======pid=%d, comm=%s========\n", pid, comm);
        $vma = (struct vm_area_struct *)arg1;
        $vma_start = $vma->vm_start;
        $vma_end = $vma->vm_end;
        $vma_size = $vma_end - $vma_start;
        printf("vma_start=%lu, vma_end=%lu, vma_size=%lu\n",
                $vma_start, $vma_end, $vma_size);

        $vm_pgoff = $vma->vm_pgoff;
        $nr_pages = ($vma_size / 4096) - 1;
        $user_extra = $nr_pages + 1;

        printf("$user_extra=%lu\n", $user_extra);

        $sysctl_perf_event_mlock = 512 + ($PAGE_SIZE / 1024);
        $user_lock_limit = $sysctl_perf_event_mlock >> ($PAGE_SHIFT - 10);
        $user_lock_limit *= $num_online_cpus;

        printf("$user_lock_limit=%lu\n", $user_lock_limit);


        $task = (struct task_struct *)curtask;
        $cred = (struct cred *)$task->cred;
        $user = (struct user_struct *)$cred->user;
        $user_locked = $user->locked_vm.counter + $user_extra;

        printf("user_locked: %lu\n", $user_locked);
        $extra = 0;
        if ($user_locked > $user_lock_limit) {
                $extra = $user_locked - $user_lock_limit;
        }
        $lock_limit = $RLIMIT_MEMLOCK >> $PAGE_SHIFT;
        $pinned_vm = $vma->vm_mm->pinned_vm;
        $locked = $pinned_vm + $extra;

        printf("locked=%lu, lock_limit=%lu, pinned_vm=%lu, $extra=%lu\n",
                $locked, $lock_limit, $pinned_vm, $extra);

        if ($locked > $lock_limit) {
                printf("!!!!!!!! locked > lock_limit !!!!!!!!\n");
        }
}
```

The following is part of the output:

```
Attaching 1 probe...
=======pid=3983764, comm=ilarouter========
vma_start=140108199350272, vma_end=140108199362560, vma_size=12288
user_extra=3
user_lock_limit=4644
user_locked: 32889
locked=28245, lock_limit=131072, pinned_vm=0, $extra=28245
=======pid=3983764, comm=ilarouter========
vma_start=140108199337984, vma_end=140108199350272, vma_size=12288
user_extra=3
user_lock_limit=4644
user_locked: 32892
locked=56493, lock_limit=131072, pinned_vm=28245, $extra=28248
=======pid=3983764, comm=ilarouter========
vma_start=140108199325696, vma_end=140108199337984, vma_size=12288
user_extra=3
user_lock_limit=4644
user_locked: 32895
locked=84744, lock_limit=131072, pinned_vm=56493, $extra=28251
=======pid=3983764, comm=ilarouter========
vma_start=140108199313408, vma_end=140108199325696, vma_size=12288
user_extra=3
user_lock_limit=4644
user_locked: 32898
locked=112998, lock_limit=131072, pinned_vm=84744, $extra=28254
=======pid=3983764, comm=ilarouter========
vma_start=140108199301120, vma_end=140108199313408, vma_size=12288
user_extra=3
user_lock_limit=4644
user_locked: 32901
locked=141255, lock_limit=131072, pinned_vm=112998, $extra=28257
!!!!!!!! locked > lock_limit !!!!!!!!
=======pid=3983764, comm=ilarouter========
vma_start=140108199288832, vma_end=140108199301120, vma_size=12288
user_extra=3
user_lock_limit=4644
user_locked: 32904
locked=169515, lock_limit=131072, pinned_vm=141255, $extra=28260
!!!!!!!! locked > lock_limit !!!!!!!!
...
```

{: .box-note}
**Note**: the unit of above numbers is *page* and one page is 4KB.

Surprisingly, after only 4 calls, `locked` grew larger than `lock_limit`! 

To debug the issue, let's take a closer look at [this piece of code in
perf_mmap](https://github.com/torvalds/linux/blob/f733c6b508bcaa3441ba1eacf16efb9abd47489f/kernel/events/core.c#L5806-L5826):

```
	user_lock_limit = sysctl_perf_event_mlock >> (PAGE_SHIFT - 10);

	/*
	 * Increase the limit linearly with more CPUs:
	 */
	user_lock_limit *= num_online_cpus();

	user_locked = atomic_long_read(&user->locked_vm) + user_extra;

	if (user_locked > user_lock_limit)
		extra = user_locked - user_lock_limit;

	lock_limit = rlimit(RLIMIT_MEMLOCK);
	lock_limit >>= PAGE_SHIFT;
	locked = atomic64_read(&vma->vm_mm->pinned_vm) + extra;

	if ((locked > lock_limit) && perf_paranoid_tracepoint_raw() &&
		!capable(CAP_IPC_LOCK)) {
		ret = -EPERM;
		goto unlock;
	}
```

This piece of code is a bit involved. Let’s first look at some key variables:

* `user_lock_limit`:

Its value is `(sysctl_perf_event_mlock >> (PAGE_SHIFT - 10)) *
num_online_cpus()` so **it is essentially a constant**.

* `user_extra`:

As per *bpftrace* log above, its value is always 3 (pages) so it can also **be
considered as a constant**.

* `user_locked`:

Its value is `user->locked_vm + user_extra`, which means all locked VM by
current user plus `user_extra`. According to the the log, it is **increased by
3 (which is `user_extra`) per call**. This is because the function has
`user->locked_vm += user_extra` before return.

* `extra`

Its value is `user_locked - user_lock_limit` and is **increased by 3 (which is
`user_extra`) per call** according to the log. This is because `user_locked` is
increased by 3 as mentioned above.

* `lock_limit`

This is the limit we changed from infinity to 512MB. In other words, it’s
**also a constant**.

* `locked`

Its value is vma->vm_mm->pinned_vm + extra, which is **increased by `extra` per
call**.

# Mystery solved (kind of)

Look at the log, I noticed that `extra` is big -- around 28K pages = 28K pages
\* 4KB/page = 112 MB! As mentioned above, `locked` is increased by `extra` per
call, no wonder it grew to more than 512 MB at the 5th calls (5 \* 112 MB =
560MB \> 512MB).

## So why is `extra` so big?

According to the statement, `extra = user_locked - user_lock_limit`. And
according to `bpftrace`, `user_lock_limit` is small compared to `user_locked`
(4K v.s.  32K). We know that `user_lock_limit` is a constant and is determined
by `sysctl_perf_event_mlock`, which is set by *sysctl* and the default value is
516KB:
```
# /sbin/sysctl -a | grep perf_event_mlock
kernel.perf_event_mlock_kb = 516
```
After setting `perf_event_mlock_kb` to 512MB, the issue was solved without
requesting `CAP_IPC_LOCK` capability. Because in this case, extra is always
zero thus `locked` is never increased.

# Mystery solved (fully)

You still smell something strange, don’t you? At least I did. Think about it,
even if `locked` keeps increasing, how come it can grow bigger than a huge
limit such as 5GB?

As mentioned before, `locked` is increased by `extra` per call. And `extra`
becomes non-zero once `user_locked > user_lock_limit`. However, **in each call,
user_extra is just 3 pages (12KB) but why is `locked` increased by 112MB**?
Shouldn’t it be increased by 12 KB per call?

After carefully examining the code again, I found the bug hides in the
following lines:

```
	user_locked = atomic_long_read(&user->locked_vm) + user_extra;

	if (user_locked > user_lock_limit)
		extra = user_locked - user_lock_limit;

  ...
  locked = atomic64_read(&vma->vm_mm->pinned_vm) + extra;
  ...
  vma->vm_mm->pinned_vm += extra;
```

Note that `user->locked_vm` is **total memory locked by the user** and
`vma->vm_mm->pinned_vm` is **total memory locked by the process**. Combine all
statements together, we get:

```
vma->vm_mm->pinned_vm += (user->locked_vm + user_extra - user_lock_limit)

```

Given that `user_lock_limit` and `user_extra` are constant, we have:

```
vma->vm_mm->pinned_vm += (user->locked_vm + K)
```

That is, **for every call, ALL historical user locked memory is added to
`pinned_vm`** instead of only increased memory. Because we call `mmap` (# of
CPUs times), `pinned_vm` and `locked` quickly exploded.

# Fix

## Our fix
There are two solutions to fix the issue 
1. Request `CAP_IPC_LOCK` capability for our container.
2. Set `perf_event_mlock_kb` to 512MB via sysctl.  

We chose solution 1 because sysctl config is a host-level config but we don’t
want to have host-level requirements when running our container.

## Kernel fix

Given that the intention of this code is, **"whatever that can’t charge to the
user (user->locked_vm), charge it to the process (pinned_vm)"**, the logic of
this function should be:

```
if user_locked <= user_limit { // Charge new locked memory to user
  extra = 0; // Charge nothing to the process
}
else if this is the first time user_locked > user_limit {
  user_extra = user_limit - user->locked_vm; // Charge the user until limit
extra = user_locked - user_limit;        // Charge the rest to the process
}
else { // user_limit has already exceeded before
extra = user_extra; // Charge everything to the process
user_extra = 0; // Charge nothing to the user
}

locked = vma->vm_mm->pinned_vm + extra;

...
user->locked_vm += user_extra;
vma->vm_mm->pinned_vm += extra;
```

After I reported bug, Song fixed it in [this
patch](https://github.com/torvalds/linux/commit/d44248a41337731a111374822d7d4451b64e73e4).

## ----------- EDIT (2020-04-01) -------------

After the [initial
fix](https://github.com/torvalds/linux/commit/d44248a41337731a111374822d7d4451b64e73e4),
there came a [fix of the
fix](https://github.com/torvalds/linux/commit/5e6c3c7b1ec217c1c4c95d9148182302b9969b97),
followed by [a fix of the fix of the
fix](https://github.com/torvalds/linux/commit/36b3db03b4741b8935b68fffc7e69951d8d70a89),
followed by [a fix of the fix of the fix of the
fix](https://github.com/torvalds/linux/commit/c4b75479741c9c3a4f0abff5baa5013d27640ac1),
followed by [a fix of the fix of a fix of the fix of the
fix](https://github.com/torvalds/linux/commit/003461559ef7a9bd0239bae35a22ad8924d6e9ad).

![Fix bug](/img/fix_bug.jpg){: .center-block :}

