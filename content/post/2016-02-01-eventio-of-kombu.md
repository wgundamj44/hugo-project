---
title: "eventio of kombu"
categories: ["tech"]
date: 2016-02-01
tags: ["python"]
---
kombu is python messaging library mainly used by Celery. I'm mainly interested in its ability to communicate with rabbitmq.

## eventio
The most basic part of kombu is eventio with select, epoll or kqueue. It provids a common interface for these three.

Take `select` as an example. It wraps the native select function. It registers `fd` to sets either for read, write or error, and call `_poll` to listen for incoming messages. `fd`s can be sockets. This `_poll` just calls `select`, and populates each fd set with fds that're ready. Each call to `_poll` is blocked for a period of time until timeout, or returns a list of tuple (fd, event).

## hub
Hub represents a event loop that's built upon eventio. It holds a loop object that can be used as an generator. Every call to `next` on the loop object will fire the callbacks which is bound to a specific fd if the fd is ready. The code looks like this:
```python
# we have registered callbacks in readers and writers
while 1:
    events = poll(poll_timeout)
    for fd, event in events:
        if event & READ:
            cb, args = readers[fd]
        elif event & WRITE:
            cb, args = writers[fd]

        # fire callbacks
        cb(*args)
    yield
```

