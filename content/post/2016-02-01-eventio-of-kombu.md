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

## Timer

Hub is powerful in that it supports ETA tasks. (The ETA (estimated time of arrival) lets you set a specific date and time that is the earliest time at which your task will be executed.)

In Kombu's Timer.py, there's a `Timer` class. This class supports `call_at`, `call_after`, `call_repeatly` function. They are used to call a
function at specific time, after several seconds and every few seconds. They way to support this is that, the `Timer` holds a queue inside,
each entry of the queue is a tuple of `(eta, priority, entry)`. `Timer` also has `__iter__` method which makes it can be used as a generator.
Inside `__iter__`, it's a while loop which checks the top of the queue, and see if eta is smaller than current time. If so, yield the entry,
or returns the time to be wait otherwise. The caller side will keep querying the timer object use for loop. If the loop yield an entry, it 
means the entry is due to be called, if the loop yileds a time, it means the next entry is due in the returned time, so the caller just sleep
for that long to give cpu to others.

## Hub with eta
When Hub and time are combined together, the Hub not only can react to read/write events of event loop, but also react to timer events.
The Hub object holds a timer registered with timed events, everytime Hub's loop is going to wait for new event, it will pass a `poll_time` to
`poll` method, this specifies the timeout for `poll`, so the process won't be stuck in `poll` forever. Instead, after `poll_time`, it will have
a chance to respond to timer event. The `poll_time` is the return value of `Timer` generator. As we mentioned before, it indicates how long will
next entry be due.