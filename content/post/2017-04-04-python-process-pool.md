---
title: "Python process pool"
categories: ["tech"]
date: 2017-04-04
tags: ["process", "python"]
---
Trying to understand how python process pool works. 
<!--more-->

# Pool
In the big picture, process pool is class that wrapps a list of process objects. When invoking `apply_async`, the pool just pushes the task
to a queue. Its child process will fetch task from queue, and executes. This sounds easy, but there're lots of details here.

First there're three queues involved, `taskqueue`, `inqueue` and `outqueue`.

- `taskqueue` is an instance of `Queue.Queue`. When we call `pool.apply_async(f, (2, ))`, the task `f` will be pushed taskqueue.
- `inqueue` is a 'pipe'. A pipe is used in unix for process communication. Basically it has two ends, one can write to one end, and read from
another end. `inqueue` is used to pass task to child process. Main process push task to `inqueue`, child process reads task from it.
- `outqueue` is also a 'pipe' It is used by child process to return results to main process. Child process push result to `outqueue`, main
process reads result from it.

Second, The pool needs to do at least three things at the same time: spawning new process when old process finished execution, dispatch task to an idle
process, fetch the result from worker process. The pool achieves this by maintaining three threads. These threads runs `_handle_workers`,
`_handle_tasks` and `_handle_results` respectively.

## _handle_workers
This thread is a while loop which keeps checking the pool. If it find the size of pool is less than disired process numbers, it will spawns new
process.

## _handle_tasks
This thread is another while loop which keeps fetching task from `taskqueue`. When it successfully fetches a task, it will put the task to its
`inqueue`.

## _handle_results
This thread is yet another while loop which keeps fetching results from 'outqueue`. When it got a result, it will call the necessary function of
`AsyncResult`.

# AsyncResult class
We generally use pool as:
```
res = pool.apply_async(f, (args, ))
res.get() # block here until result is ready
```
The `res` here is an instance of `AsyncResult`  class. When `AsyncResult` is initialized, it registers itself to `cache` of the pool. The
`AsyncResult` also holds a `Condition` object. When `get` is called, it will wait on that condition object. The condition object will get
notified when the result is ready, in `_handle_results` of pool which is another thread, when the async function call is success, the `_get`
of `AsyncResult` will be called. It is inside this function, that the condition will be notified and return value is set.

# Process class
`Process` objects are what reside in the process pool. It is used as:
```
p = Process(target=f, args=(a, b,))
p.start()
```
It is the `start` that will actually spawn new process. `Popen` is used to fork a new process: `self._popen = Popen(self)`. At this momnent,
a new process will be spawned which will run code specified by `target` parameter. Parent process holds an instance of Process `p`. With this
instance parent can inspect the current status of child process via `p.is_active()`or wait unit child process terminates via `p.join()`. Most
of these abilities come from its underlying `Popen` instance. Eg. `p.is_active()` actually relies on `self._popen.poll`, `p.join` relies on
`self._popen.wait`.

`Process` has a `run` method which when invoked will run the passed `target` function. This function is called by `_bootstrap` of `Process`. We
will explain this later.

# Popen class
`Popen` is class that tries to make forking and managing a subprocess easy. It is called like this `popen = Popen(process_obj)`. `process_obj`
is a instance of `Process`. When `Popen` is initialized, it will call `os.fork()` to spawn a subprocess, and in this subprocess, `process_obj`'s
`_bootstrap` will be called, and finally `run` method of `Process` will be called. So after `Popen` initialized, the main process will return
with an instance of `Popen` whose `pid` field will be set to pid of child process, and child process will execute its 'target` function.

The main process can do `wait`,`poll` and `terminate` operation through this `Popen` instance.

## poll
`def poll(self, flag=os.WNOHANG)`, this method has two purpose. When invoked with default parameter, it will check if child process has finished,
and will return `None` if it hasn't finished yet or return exit code otherwise. If invoked with parameter 0, it will wait until child process
finished and returns the exit code.
This method achieves this purpose by using `os.waitpid` system function.

## wait
`def wait(self, timeout=None)`. This method keeps using `poll` to wait until child process finished or until timeout. The interesting behevior
of this code is how it determines how long to sleep until next poll. It has a starting delay of 0.0005s and maximum delay of 0.05s. If a `poll`
returns with None (child still executing), it will doubles the delay, and sleep for delay seconds, then do another `poll`. 