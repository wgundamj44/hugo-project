---
title: "note of linux programming(thread control)"
categories: ["tech"]
date: 2017-05-19
tags: ["linux"]
---

# thread attribute
Thread attribute is used to customize the thread.
- create and destory
```
int pthread_attr_init(pthread_attr_t *attr);
int pthread_attr_destroy(pthread_attr_t *attr);
```
Attribute can customize detachstate, guardsize, statckaddr and stacksize of thread. Special functions should be used to manipulate
`pthread_attr_t`.

## detachstate
```
int pthread_attr_getdetachstate(const pthread_attr_t *restrict attr, int *detachstate);
int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate);
```
`detachstate` can be `PTHREAD_CREATE_DETACHED` or `PTHREAD_CREATE_JOINABLE`. If set to `PTHREAD_CREATE_DETACHED`, the thread will start as
detached mode.

## stack start address and size
```
int pthread_attr_getstack(const pthread_attr_t *restrict attr, void **restrict stackaddr, size_t *restrict stacksize)
int pthread_attr_setstack(const pthread_attr_t *attr, void *stackaddr, size_t *stacksize)
```
`stackaddr` is the start address of stack, `stacksize` is stack size. This function manage both stack address and stack size.
```
int pthread_attr_getstacksize(const pthread_attr_t *restrict attr, size_t *restrict stacksize)
int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize)
```
This function only manage stack size.

## others
There' are other attributes that don't belong to `pthread_attr_t`. 
- concurrency
```
int pthread_getconcurrency(void)
int pthread_setconcurrency(int level)
```
This function controls the number of kernal threads or process on top of which user-level threads are mapped. 
It only gives the system the hint of desired concurrency.

# synchronization attribute

## mutex attributes
- create and destory
```
int pthread_mutexattr_init(pthread_mutexattr_t *attr)
int pthread_mutexattr_destroy(pthread_mutexattr_t *attr)
```
- process level or process private
```
int pthread_mutexattr_getpshared(const pthread_mutexattr_t * restrict attr, int *restrict pshared)
int pthread_mutexattr_setpshared(pthread_mutexattr_t *attr, int pshared)
```
Mutex can be used inside a process or shared between several processes. We control this by give `pshared` value `PTHREAD_PROCESS_PRIVATE` or 
`PTHREAD_PROCESS_SHARED`.
- mutex type
```
int pthread_mutexattr_gettype(const pthread_mutexattr_t * restrict attr, int *restrict type)
int pthread_mutexattr_settype(pthread_mutexattr_t *attr, int type)
```
Mutex has three types: `PTHREAD_MUTEX_NORMAL`, `PTHREAD_MUTEX_ERRORCHECK` and `PTHREAD_MUTEX_RECURSIVE`. In normal type, mutex can't be locked 
if already in locked mode, and no dead-lock is checked. In errorcheck mode, dead-lock will be checked. In recursive mode, the same thread can
lock a mutex multiple times, only when same number of unlock is preformed will the mutex become unlocked.
-
## reader-writer lock attributes
- create and destory
```
int pthread_rwlockattr_init(pthread_rwlockattr_t *attr)
int pthread_rwlockattr_destroy(pthread_rwlockattr_t *attr)
```
- process level or process private
```
int pthread_rwlockattr_getpshared(const pthread_rwlockattr_t * restrict attr, int *restrict pshared)
int pthread_rwlockattr_setpshared(pthread_rwlockattr_t *attr, int pshared)
```
## condition variable attributes
- create and destory
```
int pthread_condattr_init(pthread_condattr_t *attr)
int pthread_condattr_destroy(pthread_condattr_t *attr)
```
- process level or process private
```
int pthread_condattr_getpshared(const pthread_condattr_t * restrict attr, int *restrict pshared)
int pthread_condattr_setpshared(pthread_condattr_t *attr, int pshared)
```

# thread safe
If a function can be safely called by multiple threads at the same time, we call it thread-safe. A thread-safe function doesn't necessarily a
reentrant function for signal handler. For example, if a function use mutex, it may be thread-safe, but when interrupted by a signal while the
thread is holding the lock, then if this function is called in signal handler again, dead-lock will occur if the mutex is not recursive.

# thread specific data
Thread can have thread specific data. This data must be associated with a key.
- create key
```
int pthread_key_create(pthread_key_t *keyp, void (*destructor)(void *))
```
`keyp` is the address the key created, `destructor` is used for cleaning up the data associated with the key, like `free`. 
When key is created, all the threads will see the key and the initial data will be null. `destructor` is called when thread calls
`pthread_exit`. It won't be called when thread calls `exit`, `_exit` or `abort`. As new thread specific data can be created in  `destructor`, 
when thread exists, the cleanup will be iteractive until max interation limit is reached.

- delete key
```
int pthread_key_delete(pthread_key_t *key)
```
This function breaks the association between key and data. But destructor of data won't be called.

- pthread_once
```
pthread_once_t initflag = PTHREAD_ONCE_INIT;
int pthread_once(pthread_once_t *initflag, void (*initfn)(void))
```
Sometimes we want certain operation to be carried out only once among all threads, this is when `pthread_once` can be used. `initflag` must
first be set to `PTHREAD_ONCE_INIT`, and `pthread_once` will ensure that `initfn` will get executed only once.

- association key with data
```
void *pthread_getspecific(pthread_key_t key)
int pthread_setspecific(pthread_key_t key, const void *value)
```
When key is not associated with any data, `pthread_getspecific` will return null.

# cancel operation

## enable and disable cancellation
When we call `pthread_cancel`, a thread can choose to block the cancellation request.
```
int pthread_setcancelstate(int state, int *oldstate)
```
When `state` is `PTHREAD_CANCEL_ENABLE`, the thread will be canceled at next cancel point. If `state` is `PTRHEAD_CANCEL_DISABLE`, the cancel
request will be blocked unitl `state` is enabled again and next cancel point is reached. Cancel point is the moment where thread will check if 
itself is cancelled. There're lots of unix functions contain cancellation point.

```
int pthread_setcancelstate(int state, int *oldstate)
```
We can use this function to forcely check cancellation.

## deffered and asynchronous cancellation
Thread cancel type can be `PTHREAD_CANCEL_DEFFERED` or `PTHREAD_CANCEL_ASYNCHRONOUS`.
```
int pthread_setcanceltype(int type, int *oldtype);
```
This function is used to set the cancel type. If `type` is `PTHREAD_CANCEL_DEFFERED` worked as stated before. If `type` is `PTHREAD_CANCEL_ASYNCHRONOUS`, the cancel point won't be needed to make a thread cancel.

# thread and signal
Each thread has its own signal masks but they all share the same signal handler. So if one thread choose to ignore a signal, another thread may
change this disposition with another handler.
If the signal is the result of hardware fault or alarm, then the thread that caused the event gets the signal. Otherwise, the signal is sent to
one of the threads.

```
int pthread_sigmask(int how, const sigset_t *restrict set, sigset_t *restrict oset)
```
This function is the thread version of `sigprocmask`.

```
int sigwait(const sigset_t *restrict set, int *restrict signop)
```
This function is used to wait for some signals. The thread will be blocked in this function, and when a signal in `set` is pending, the signal
will be removed from pending list, and function returns. Upon returns, the signal mask will be restored to before the function call, and 
`signop` contains the signal no.
If multiple thread is waiting for the same signal, then only one of the thread will return. If a signal is caught by process and waited by a
thread, then which one takes effect is up to the implementation.

```
int pthread_kill(pthread_t thread, int signo)
```
This function sends signal to a thread. When `signo` is 0, it can be used to test if a thread exists.

# thread and fork
In a multi-thread process, when one of the thread calls fork, then the child process will have copy of only one thread, the one that called 
fork. If at the time of the fork, other threads are holding mutex, this mutex will be inherited by child process. However, as the thread that
holding the mutex is gone in child process, there's no way for the thread in the child to unlock the mutex. 
So `pthread_atfork` is introduced to solve this.
```
int pthread_atfork(void (*prepare)(void), void (*parent)(void), void (*child)(void))
```
This function makes three hooks. `prepare` will be called before the `fork` has created the child, this can be used to hold all the mutexes.
`parent` will be called in the parent context, at the time when child is created but before fork returns. It is used to unlock the mutexes 
collected in `prepare`. `child` will be called at child context, before return of fork. It is used to unlock the mutexes collected in `prepare`.
Note that the mutex should be unlocked twice, as the child will get a copy of all mutex of parent process.