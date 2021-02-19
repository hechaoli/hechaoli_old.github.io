---
layout: post
title: Mini Container Series Part 4
subtitle: IPC Isolation
tags: [Container, Linux]
---

[This is the fifth article in this series. The previous one is
[here](http://hechao.li/2020/06/18/Mini-Container-Series-Part-3-Host-and-Domain-Name-Isolation/)].

# Summary
While doing process isolation, we isolated processes running inside the
container from the rest of the host. However, technically speaking, complete
procession isolation has not finished yet. **Though a process inside the
container can't see other processes on the host via `ps` command, it may still
talk to them via IPC (Inter-Process Communication) mechanisms**. To cut off
this channel, we need the help of IPC namespace.

# IPC and IPC namespace
How IPC works or how to use IPC in Linux is out of the scope of this series. So
I won't dive deep into IPC details. If interested in IPC, you may can take at
man [sysvipc(7)](https://man7.org/linux/man-pages/man7/sysvipc.7.html).
(Spoiler alert: it's not fun as it's all about reading API docs.) I also have
some code examples in [this repo](https://github.com/hechaoli/linux_ipc_examples).

There are 3 IPC mechanisms on UNIX systems:

* Message queues
  * Allow processes to exchange data in units called messages.
* Semaphore sets
  * Allow processes to synchronize their actions.
* Shared memory
  * Allow process to share a memory region.

For each mechanism, there are two sets of APIs: System V APIs and POSIX APIs.
Though they are the same functionality wise, they may use different kernel
interfaces under `/proc`. As mentioned in [Part 2: Filesystem
isolation](http://hechao.li/2020/06/10/Mini-Container-Series-Part-2-Process-Isolation/),
**`/proc` can be seen as an interface internal data structures in the kernel**.
And as seen in [Part 3: Host and domain name
isolation](http://hechao.li/2020/06/18/Mini-Container-Series-Part-3-Host-and-Domain-Name-Isolation/),
without proper namespace isolation, the container can have the same view of
files in `/proc` as the host even if they are on different file systems.

Thus, in this part, **our goal is to make sure these IPC-related interface
files are distinct in the container and on the host**. You must have already
guessed - we will use IPC namespace to achieve this. Specifically, the
following `/proc` interfaces will be isolated by IPC namespace:

* `/proc/sys/fs/mqueue`
  * Used by POSIX message queue.
* `/proc/sys/kernel/{msgmax, msgmnb, msgmni}` and `/proc/sysvipc/msg`
  * Used by System V IPC message queue.
* `/proc/sys/kernel/{sem}` and `/proc/sysvipc/sem`
  * Used by System V IPC semaphore sets.
* `/proc/sys/kernel/{shmall, shmmax, shmmni, shm_rmid_forced}` and `/proc/sysvipc/shm`
  * Used by System V IPC shared memory.

Before adding IPC namespace support in our mini container, let's look at an
example in which a process inside the container can talk to a process outside.

# Message Queue Example
This is a simple echo server/client communicating via message queue. The code
used in the example can be found
[here](https://github.com/hechaoli/linux_ipc_examples/tree/main/message_queue/sysv).
Note that the example uses system V APIs.

![Echo server client communicating via message queue](/img/echo_server_client_mq.png)

## Start a container
```bash
$ sudo ./mini_container --rootfs /tmp/mini_container/rootfs --pid "/bin/bash"
[Agent] Container pid: 49562
[Agent] Agent pid: 49561
[Agent] Agent hostname: hechaol-vm
[Agent] Agent NIS domain name: (none)
[Container] Running command: /bin/bash
[Container] Container hostname: hechaol-vm
[Container] Container NIS domain name: (none)
[root@hechaol-vm /]# 
```
## Run echo server on the host
```bash
$ git clone https://github.com/hechaoli/linux_ipc_examples.git
$ cd linux_ipc_examples/message_queue/sysv
$ make echo_server
$ mkdir /tmp/echo_server
$ sudo ./echo_server /tmp/echo_server 1
Server key: 17118584
```

After the server is started, the server's message queue is created. We can view
the queue information using `ipcs -q` or `cat /proc/sysvipc/msg` command:
```bash
$ ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x01053578 14         hechaol    660        0            0

$ cat /proc/sysvipc/msg
       key      msqid perms      cbytes       qnum lspid lrpid   uid   gid  cuid  cgid      stime      rtime      ctime
  17118584         14   660           0          0     0     0  1000  1000  1000  1000          0          0 1613718949
```

Let's see what happens if we run these commands inside the container.

```bash
[root@hechaol-vm /]# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x01053578 14         1000       660        0            0

[root@hechaol-vm /]# cat /proc/sysvipc/msg
       key      msqid perms      cbytes       qnum lspid lrpid   uid   gid  cuid  cgid      stime      rtime      ctime
  17118584         14   660           0          0     0     0  1000  1000  1000  1000          0          0 1613718949

# The container can't see the echo server process
[root@hechaol-vm /]# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0  12028  3344 ?        S    07:12   0:00 /bin/bash
root          12  0.0  0.0  44636  3348 ?        R+   07:18   0:00 ps aux
```

This experiment shows that **though the container doesn't see the echo server
process, it still sees the message queue created by the server**.

## Run echo client in the container
Next we build and copy the echo client program to the container.

```
$ make echo_client
$ sudo cp echo_client /tmp/mini_container/rootfs/
```

In the container, run the echo client and pass the key of the server's queue to
it.
```bash
[root@hechaol-vm /]# ./echo_client 17118584
login
Logged in successfully!
Hello
Hello
World
World
exit
```
Obviously, the client process inside the container can talk to the server
process on the host even without knowing the existence of the server.

# Mini container: IPC Isolation
I bet you already know what I am going to say. Yes, to support IPC isolation,
we only need to pass `CLONE_NEWIPC` to `clone()`. Now the core code skeleton
becomes
```bash
  int cpid = syscall(SYS_clone,
                     SIGCHLD |
                     CLONE_NEWNS |
                     CLONE_NEWPID |
                     CLONE_NEWUTS |
                     CLONE_NEWIPC);

  if (cpid == -1) {
    errExit("fork");
  }
  if (cpid == 0) {
    setupFilesystem(rootfs);
    setHostAndDomainName(hostname, domain);
    execv(argv[1], &argv[1]);
  } else {
    if (waitpid(cpid, NULL, 0) == -1) {
      errExit("waitpid");
    }
  }
  return 0;
```

For complete change, see [this
commit](https://github.com/hechaoli/mini_container/commit/106496966f5f81fd70071b3db6b826b5989511be).

# Test
We only need to rebuild the program and repeat the test above.

## Rebuild and rerun the container
```bash
$ make
$ sudo ./mini_container --rootfs /tmp/mini_container/rootfs --pid --ipc "/bin/bash"
[Agent] Container pid: 50300
[Agent] Agent pid: 50299
[Agent] Agent hostname: hechaol-vm
[Agent] Agent NIS domain name: (none)
[Container] Running command: /bin/bash
[Container] Container hostname: hechaol-vm
[Container] Container NIS domain name: (none)
[root@hechaol-vm /]# 
```
## Get message queues in the container
```bash
# It no longer sees the server's message queue
[root@hechaol-vm /]# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages

[root@hechaol-vm /]# cat /proc/sysvipc/msg
       key      msqid perms      cbytes       qnum lspid lrpid   uid   gid  cuid  cgid      stime      rtime      ctime
```
The container no longer sees message queues on the host. Yay!

## Run the client inside the container
What will happen if the container guesses the server's message queue key
`17118584`? Let's run the echo client to verify.
```
[root@hechaol-vm /]# ./echo_client 17118584
msgget(key, 0): No such file or directory
```
This time `msgget()` returns an error because the server queue key is not
found. Nice! The container can no longer talk to processes outside via IPC.

(You may do some tests with semaphore sets and shared memory using code
[here](https://github.com/hechaoli/linux_ipc_examples/tree/main/) if
interested.)

# Conclusion

Cool! So far we have a container with its own filesystem, process space,
hostname and NIS domain name and IPC objects. We will continue isolating other
resources in next articles.

![IPC isolation](/img/ipc_isolation.png)

# Resources
[1] [man svipc(7)](https://man7.org/linux/man-pages/man7/sysvipc.7.html)  
[2] [man ipc_namespace(7)](https://man7.org/linux/man-pages/man7/ipc_namespaces.7.html)  
