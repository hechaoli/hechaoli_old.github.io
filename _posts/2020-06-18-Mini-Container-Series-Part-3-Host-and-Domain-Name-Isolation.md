---
layout: post
title: Mini Container Series Part 3
subtitle: Hostname and Domain Name Isolation
tags: [Container, Linux]
---

[This is the forth article in this series. The previous one is
[here](http://hechao.li/2020/06/10/Mini-Container-Series-Part-2-Process-Isolation/)].

# Summary
After the last two long articles, let's take a break and discuss something
simple. In this article, I will talk about how to make container think it has
its own hostname and domain name independent of the actual host.

Though simple, there are still some concepts to clarify before jumping into
implementation.

# Basics
## Hostname

A hostname is ... well, a host's name. There is really nothing to say about it.
By default, it is a free-form string with up to 64 characters in length. You
can get its length limit by the following command.
```
$ getconf HOST_NAME_MAX
64
```

Hostname is simple yet confusing sometimes. Because **it's stored in several
locations and there are several ways to get and set it**.

At least two files are used to store the hostname - `/etc/hostname` and
`/proc/sys/kernel/hostname`.
* The former is used to store the static hostname that is set during boot using
  the `sethostname()` system call [2]. You can use `hostnamectl` to change its
  value during runtime. 
* The latter is used to store the transient hostname that can be changed using
  `hostname NAME` or `hostnamectl --transient set-hostname NAME`. **It's
  "transient" as it doesn't survive a reboot**.

The following commands can be used to get/set transient hostname.
```bash
# Get transient hostname
$ hostname
hechaol-vm
$ uname -n
hechaol-vm
$ sysctl kernel.hostname
kernel.hostname = hechaol-vm
$ cat /proc/sys/kernel/hostname
hechaol-vm
$ hostnamectl --transient
hechaol-vm

# Set transient hostname (need to be root)
$ sudo hostname new-name
$ sudo hostnamectl --transient set-hostname new-name
$ echo "newname" | sudo tee /proc/sys/kernel/hostname
```

You may also use
[gethostname() and sethostname()
syscalls](https://www.man7.org/linux/man-pages/man2/gethostname.2.html) to
get/set transient hostname programmatically.

And the following commands can be used to get/set static hostname.
```bash
# Get static hostname
$ cat /etc/hostname
hechaol-vm
$ cat /etc/hostname
hechaol-vm

# Set static hostname
$ sudo hostnamectl --static set-hostname new-name
$ echo "newname" | sudo tee /etc/hostname
```
You can imagine that it will be confusing if one uses `hostname NAME` to change
the hostname but find it unchanged in `/etc/hostname`.

## NIS domain name

Network Information Service (NIS), is a client–server directory service
protocol for distributing system configuration data such as user and host names
between computers on a computer network [4]. It was developed by Sun
Microsystems.

Honestly, you don't need to know what it is. My layman understanding is, it is
similar to DNS (Domain Name System) but is simpler and designed for LAN (local
area network).

The only thing I'd like to clarify is, **NIS domain name is different from DNS
domain name**. Command `hostname --domain` displays the DNS domain name instead
of the NIS domain name. Instead, command `hostname --nis` displays the NIS
domain name.

The following commands can be used to get/set the NIS domain name.
```bash
# Get the NIS domain name
$ hostname --nis
hostname: Local domain name not set
$ cat /proc/sys/kernel/domainname
(none)
$ sysctl kernel.domainname
kernel.domainname = (none)

# Set the NIS domain name
$ sudo hostname --nis newname
$ echo "newname" | sudo tee /proc/sys/kernel/domainname
$ sudo sysctl kernel.domainname=newname
```

You may also use [getdomainame and setdomainname
syscalls](https://www.man7.org/linux/man-pages/man2/getdomainname.2.html) to
get/set the NIS domain name programmatically.

Note that **the change to NIS domain name is transient (does not survive a
reboot)**. And I am not aware of any static NIS domain file.

## UTS namespace
UTS namespaces provide isolation of two system identifiers we just talked
about: **the hostname and the NIS domain name** [6]. UTS stands for Unix Time
Sharing and I think UTS namespace is named after `utsname()`, which is [used
when getting the
hostname](https://elixir.bootlin.com/linux/latest/source/kernel/sys.c#L1358).

# Mini container: Host and domain name isolation
As you may have already guessed, to create a UTS namespace, we need to pass
`CLONE_NEWUTS` flag to `unshare()` or `clone()`. Similar to mount namespace,
during creation, **the hostname and the NIS domain name of the new UTS namespace
are copied from the calling process's namespace**.

We only need to chang one line in the code. The code skeleton now becomes:
```cpp
  int cpid = syscall(SYS_clone,
                     SIGCHLD |
                     CLONE_NEWNS |
                     CLONE_NEWPID |
                     CLONE_NEWUTS);

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

See [this
commit](https://github.com/hechaoli/mini_container/pull/4/commits/fdc453227a69f821aedbf659abbaced2ae582222)
for complete source code.

# Test

## Build
```bash
$ mkdir build
$ cd build
$ cmake ..
$ make
```

## Default hostname and domain name
```bash
$ sudo ./mini_container --rootfs /tmp/mini_container/rootfs "/bin/bash" --pid
[Agent] Container pid: 27717
[Agent] Agent pid: 27716
[Agent] Agent hostname: hechaol-vm
[Agent] Agent NIS domain name: (none)
[Container] Running command: /bin/bash
[Container] Container hostname: hechaol-vm
[Container] Container NIS domain name: (none)
[root@hechaol-vm /]# hostname
hechaol-vm
```

This test shows that when the new UTS namespace is created, both hostname and
domain name are copied from the host.

## Set hostname and domain name for the container
```bash
$ sudo ./mini_container --rootfs /tmp/mini_container/rootfs --pid --hostname foo --domain bar "/bin/bash"
[Agent] Container pid: 27977
[Agent] Agent pid: 27976
[Container] Running command: /bin/bash
[Agent] Agent hostname: hechaol-vm
[Container] Container hostname: foo
[Container] Container NIS domain name: bar
[Agent] Agent NIS domain name: (none)
[root@foo /]# hostname
foo
```

This test shows that the agent on the host and the container get different
results from `gethostname()` and `getdomainname()` syscalls.

## Something interesting
The two tests above are very straightforward and kind of boring. Let's test
something interesting.

One question I had was, since we can change the hostname by updating
`/proc/sys/kernel/hostname` file and the container has an isolated filesystem,
if we don't create a UTS namespace and change the hostname by updating that
file, will the host be affected? Let's see.

```bash
$ sudo ./mini_container --rootfs /tmp/mini_container/rootfs --pid "/bin/bash"                            
[Agent] Container pid: 28009
[Agent] Agent pid: 28008
[Container] Running command: /bin/bash
[Container] Container hostname: hechaol-vm
[Container] Container NIS domain name: (none)
[Agent] Agent hostname: hechaol-vm
[Agent] Agent NIS domain name: (none)
[root@hechaol-vm /]# echo "newname" > /proc/sys/kernel/hostname
[root@hechaol-vm /]# hostname
newname
[root@hechaol-vm /]# 
```

On the host
```bash
$ hostname
newname
$ cat /proc/sys/kernel/hostname
newname
```

Apparently, without a UTS namespace, even with an isolated filesystem, the
host's hostname is still affected when the container updates
`/proc/sys/kernel/hostname` file. The reason is that the host and the container
shares one *procfs* (which is a virtual filesystem) though it is mounted in the
host and the container separately.

Alright. Now we have a container with its own filesystem, process space,
hostname and NIS domain name. We will continue isolating other resources in
next articles.

![Host and domain name isolation](/img/host_domain_name_isolation.png)


# Resources
[1] [man hostname(1)](https://www.man7.org/linux/man-pages/man1/hostname.1.html)  
[2] [man hostname(5)](https://www.man7.org/linux/man-pages/man5/hostname.5.html)  
[3] [man hostnamectl(1)](https://man7.org/linux/man-pages/man1/hostnamectl.1.html)  
[4] [Wikipedia: Network Information Service](https://en.wikipedia.org/wiki/Network_Information_Service)  
[5] [man setdomainame(2)](https://www.man7.org/linux/man-pages/man2/setdomainname.2.html)  
[6] [man uts_namespaces(7)](https://www.man7.org/linux/man-pages/man7/uts_namespaces.7.html)  
[7] [man uname(2)](https://man7.org/linux/man-pages/man2/uname.2.html)  
