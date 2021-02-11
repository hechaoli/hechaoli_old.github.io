---
layout: post
title: Mini Container Series Part 0
subtitle: Not a Real Container
tags: [Container, Linux]
---

# Summary
Recently, I learned some container technologies in Linux such as namespace and
cgroup. However, after reading several articles and man pages, there was still
one question lingering in my head - "How are the technologies put together?" In
other words, how are they used to implement containers?

To me, the fastest way to learn a technology is to get my hands dirty. So, I
did some experiments with namespaces, mount, chroot, cgroups by writing small
programs. Then I thought, why don't I weave my code snippets into a mini
container? It may also help others in the future. That's why I decided to start
this mini container series. In this series, I will talk about some basics of
container technologies and use minimal code snippets to demonstrate how to
use them in practice.

Keep in mind that these notes are written by a noob who only has a few weeks of
experience in container space. Errors are unavoidable. To avoid misleading
others, please point out my errors if you see any. Any feedback is appreciated!

# Code

All the code used in this series can be found [on
Github](https://github.com/hechaoli/mini_container). I will also point you to
each commit log in order to show incremental changes along the way. 

{: .box-warning}
**Disclaimer:** Error handling in the code is usually ignored for simplicity
purpose. Thus, the code is not production ready. 

# Let's get started!

I will start the mini container project with the following simple code
skeleton.

```c++
int main(int argc, char* argv[]) {
  int cpid = fork();

  if (cpid == -1) {
    errExit("fork");
  }
  if (cpid == 0) {
    // Child
    execv(argv[1], &argv[1]);
    errExit("execv");
  } else {
    // Parent
    if (waitpid(cpid, NULL, 0) == -1) {
      errExit("waitpid");
    }
  }
  return 0;
}
```

See [this
commit](https://github.com/hechaoli/mini_container/commit/2e09d967ad21ad6309249f3b7b955296fae94f81)
for complete source code.

This is definitely not a container implementation but more like a shell, which
takes a command and then execute it in a child process. None of the global
resources are isolated. Therefore, both the parent and child processes share
the same view of all system resources such as file system and network. Going
forward, I will call the parent process "container agent" and call the child
process "container".

![Not a real container](/img/not_a_real_container.png)

We will convert it to a real container step by step starting from the next
article.
