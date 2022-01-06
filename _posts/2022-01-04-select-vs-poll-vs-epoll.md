---
layout: post
title: select v.s. poll v.s. epoll
tags: [Linux, I/O]
---

# Overview
There are a lot of good articles explaining the differences among `select`,
`poll` and `epoll` on the Internet. In my opinion, the best explanation is
always the `man` page itself. And the best way to understand their differences
is to write programs using them because we can usually guess API behaviors
by looking at the usage -- just the parameters required by an API can tell a
lot about the API.

The man pages of the 3 system calls can be found at:
* [man 2 select](https://man7.org/linux/man-pages/man2/select.2.html)
* [man 2 poll](https://man7.org/linux/man-pages/man2/poll.2.html)
* [man 7 epoll](https://man7.org/linux/man-pages/man7/epoll.7.html)

Also, I personally find the best way to explain how things work is to use code
since it's the most unambiguous language. Therefore, in this post, I will use
code to explain the differences among `select`, `poll` and `epoll`. Note that
it's not code that uses these system calls but code that "implements" them.
Because my purpose is to explain "how do they work" instead of "how are they
implemented", the code is not real implementation and I'm not going to analyze
the real implementation itself. If interested, one can look at the Linux code
themselves:
* `poll` and `select`:
  [include/linux/poll.h](https://github.com/torvalds/linux/blob/master/include/linux/poll.h),
  [fs/select.c](https://github.com/torvalds/linux/blob/master/fs/select.c)
* `epoll`:
  [fs/poll.c](https://github.com/torvalds/linux/blob/master/fs/eventpoll.c)

# select

Let's assume we have an API `bool is_read(int fd)` which takes a file
descriptor (fd) and returns true if the fd is ready for I/O. Then the `select`
behavior is the following:

```c++
// Returns true if fd is ready for I/O.
bool is_ready(int fd);

struct fd_info {
  int fd;
  bool ready;
};

int select(set<fd_info> fds, int max_fd) {
  int ready_cnt = 0;
  while (ready_cnt == 0) {
    for (int i = 0; i < max_fd; i++) {
      if (is_ready(i)) {
        auto it = fds.find(i);
        it->ready = true;
        ready_cnt++;
      }
    }
  }
  return ready_cnt;
}
```

**Notes:**
* The caller passes a set of fds they want to monitor and the maximum fd
  (actually max fd + 1) among all interesting fds.
* A caller has to reset the fd set per select call. That means, if `select` is
  called in a loop, then we need to reset the fd set in each iteration.
* The complexity of the inner loop is `O(max_fd + 1)`. When the fds are sparse,
  there will be a lot of waste. For example, if the fds are `{1, 10, 1023}`,
  then the loop size is 1024 instead of 3.


Example usage looks like:
```c++
set<fd_info> fds;
while (1) {
  // Note that we need to re-initialize fds in each loop.
  fds.clear();
  fds.inert({.fd = 1})
  fds.inert({.fd = 100})

  int ready_cnt = select(fds, /*max_fd=*/100 + 1);
  assert(ready_cnt > 0);
  for (int i = 0; i < fds.size(); i++) {
    if (fds[i].ready) {
      // Use fds[i].fd
    }
  }
}
```

# poll

Let's assume we have an API `bool is_read(int fd)` which takes a file
descriptor (fd) and returns true if the fd is ready for I/O. Then the `poll`
behavior is the following:
```c++
// Returns true if fd is ready for I/O.
bool is_ready(int fd);

struct fd_info {
  int fd;
  bool ready;
};

int poll(struct fd_info* fds, int nfds) {
  int ready_cnt = 0;
  while(ready_cnt == 0) {
    for (int i = 0; i < nfds; i++) {
      if (is_ready(fds[i])) {
        fds[i].ready = true;
        ready_cnt++;
      } else {
        fds[i] = false;
      }
    }
  }
  return ready_cnt;
}
```

**Notes:**
* Unlike `select`, a caller no longer needs to reset the fds per call because
  `poll` will reset the `ready` flag of any unready fds.
* The complexity of the inner loop is `O(n)` where n is the number of fds to
  monitor. If the fds are {1, 10, 1023}, then the complexity is O(3).
* In Linux code, both `select` and `poll` implementation are in `fs/select.c`
  file because they both use the same underlying kernel poll functions.

Example usage looks like:
```c++
// Only need to initialize fds once.
fd_info fds[2];
fds[0].fd = 1;
fds[1].fd = 100;

int nfds = 2;

while (1) {
  int ready_cnt = poll(fds, nfds);
  assert(ready_cnt > 0);
  for (int i = 0; i < nfds; i++) {
    if (fds[i].ready) {
      // Use fds[i].fd
    }
  }
}
```
# epoll

Let's assume we have an API `void add_monitor(const vector<int>& all_fds,
vector<int>& ready_fds)` which triggers an external thread to constantly
monitor `all_fds` and add ready fds in it to `ready_fds`. Then the `epoll`
behavior is the following. Note that synchronization is not added for
simplicity purpose.

```c++
// Start monitoring fds in `all_fds` and constantly adds ready ones to
// `ready_fds`.
void add_monitor(const vector<int>& all_fds, vector<int>& ready_fds);

struct fd_info {
  int fd;
  bool ready;
};

struct epoll_info {
  vector<int> all_fds;
  vector<int> ready_fds;
};

map<int, epoll_info> epoll_info_by_epoll_id;

// Create an epoll instance and return its id.
int epoll_create() {
  return epoll_info_by_epoll_fd.size();
}

// Add a fd to monitor to the epoll instance.
void epoll_add(int epoll_id, int fd) {
  epoll_info_by_epoll_id[epoll_id].push_back(fd);
}

// Wait until at least one fd is ready. Return number of ready fds.
// Afte the function returns, the first `ready_cnt` of `ready_fds` contain
// ready fds. The rest can be ignored.
int epoll_wait(int epoll_id, struct fd_info* ready_fds) {
  int ready_cnt = 0;

  struct epoll_info info = epoll_info_by_epoll_id[epoll_id];
  add_monitor(info.allfds, info.ready_fds);
  while (ready_cnt == 0) {
    ready_cnt = ready_fds.size();
    for (int i = 0; i < ready_cnt; i++) {
      ready_fds[i].fd = ready_fds[i];
      ready_fds[i].ready = true;
    }
  }
  return ready_cnt;
}
```

**Notes:**
* Unlike `select` and `poll` both of which only provide one API, `epoll` is not
  a single API but a group of 3 APIs.
* `epoll_create` and `epoll_add` are called to set up the epoll instance while
  `epoll_wait` can be called in a loop to constantly wait on the fds added by
  `epoll_add`.
* The complexity of the inner loop is `O(ready fds)`. The worst case complexity
  is still `O(n)` like `poll`. However, in the case that the ready fds are
  mostly much less than fds to monitor, `epoll` has better performance than
  `poll`. In other words, even when two algorithms both have complexity `O(n)`,
  in reality, n=3 and n=10000 may matter a lot.

Example usage looks like:
```c++
int epoll_id = epoll_create();
epoll_add(epoll_id, 1);
epoll_add(epoll_id, 100);

while (1) {
  struct fd_info fds[2];
  int ready_cnt = epoll_wait(epoll_id, fds);
  assert(ready_cnt > 0);
  for (int i = 0; i < ready_cnt; i++) {
    // Use fds[i].fd
  }
}
```

# A few more words
* All 3 system calls are used for I/O multiplexing rather than non-blocking
  I/O.  This is a misconception I had when I first learned the 3 APIs. I/O
  multiplexing means we can monitor the I/O on several fds at the same time and
  read when any of them is ready whereas non-blocking I/O means when reading a
  fd, if nothing is available, return an error immediately rather than
  waiting/blocking until data is available.
* You may say that I/O multiplexing is one way to implement non-blocking I/O,
  in the sense that it's guaranteed that whenever you read on the fd after the
  I/O muliplexing API (`select`/`poll`/`epoll`) returns, there is always data
  to read (i.e. the `read` call won't block waiting on data). Yet the `read`
  call itself could still be a blocking call.
* With that being said, I/O multiplexing can all be used together with
  non-blocking I/O, especially when using edge-triggered `epoll`. See [man 7
  epoll](https://man7.org/linux/man-pages/man7/epoll.7.html).
* In case of reading from fds, all 3 system calls only make sense for network
  sockets and pipes rather than regular files (or storage I/O). It's because
  you can always say that a regular file fd is ready -- even if you have no
  more data to read, you can still read a EOF.

# Exercise
Like mentioned before, the best way to understand them system calls is to write
programs using them. Write a program that does the following:
* The main process acts as the server. 
* Fork several child processes as clients.
* For each child process/client, create a pipe to communicate with it.
* The client constantly writes to the write side of the pipe.
* The server monitors the read side of each pipe. If there is anything to read,
  print the client id and the message.

My reference solution is
[here](https://github.com/hechaoli/linux_io_multiplexing).
