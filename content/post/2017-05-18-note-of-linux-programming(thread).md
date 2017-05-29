---
title: "note of linux programming(thread)"
categories: ["tech"]
date: 2017-05-18
tags: ["linux"]
---

# thread id
pthread_t is used to identify a thread, but pthread_t is not guaranteed to be an unsigned integer, some in implementation it may be a pointer to
a structure.
- compare pthread_t
```
int pthread_equal(pthread_t tid1, pthread_t tid2)
```
- obtain pthread_t of itself
```
pthread_t pthread_self(void)
```

# thread creation
```
int pthread_create(pthread_t* restrict tidp, const pthread_attr_t *restrict attr, void* (*start_rtn)(void), void *restrict arg)
```
`tidp` is to used to store the pthread_t of the newly created thread. `attr` is used to customize the behavior of the thread.
`start_rtn` is the where the thread will start. It accept one argument `arg`.

# thread termination
thread can terminate in three ways:
- Thread can return from its start routine. The return value is the exit code of thread.
- Thread can be canceled by another thread in the same process.
- Call `pthread_exit`.

```
void pthread_exit(void* rval_ptr)
```
`rval_ptr` is a pointer which will be available to other threads if they called `pthread_join`.
```
int pthread_join(pthread_t thread, void **rval_ptr)
```
This function will block unitl thread `thread` terminated. If the thread terminated by returning from its start routine, the return value will 
be contain the return code. Otherwise rval_ptr will point to `PTHREAD_CANCELED`. If `rval_ptr` is NULL, then no return value will be retrieved.
It will only wait until the `thread` terminates.
Calling `pthread_join` will put the calling thread into a detached state so its resource can be recovered. Detached means the thread's storage
can be immediately reclaimed when terminates, as generally, when thread terminates, its storage will retained until `pthread_join` is called.
When thread is detached, `pthread_join` can't used to wait for its termination status. Also we detach a thread with:
```
int pthread_detach(pthread_t tid)
```

```
int pthread_cancel(pthread_t tid)
```
This function will request thread `tid` to terminate. However, target thread has control over wether it will terminate and how it terminates.

```
void pthread_cleanup_push(void (*rtn)(void*), void *arg)
void pthread_cleanup_pop(int execute)
```
Functions pushed by `pthread_cleanup_push` will executed when:
- pthread_exit
- thread respondes to pthread_cancel
- `pthread_cleanup_pop` is called with non-zero argument

`pthread_cleanup_pop` will remove a function pushed by `pthread_cleanup_push`. When execute is non zero, the function pushed will be executed.

# thread synchronization

## mutex
- create and destroy mutex
```
int pthread_mutex_init(pthread_mutex_t *restrict mutex, const pthread_mutexattr_t *restrict attr)
int pthread_mutext_destroy(pthread_mutext_t *mutex)
```

- lock and unlock
```
int pthread_mutex_lock(pthread_mutex_t *mutex)
int pthread_mutex_trylock(pthread_mutex_t *mutex)
int pthread_mutex_unlock(pthread_mutex_t *mutex)
```

# reader-writer lock
read-writer lock can gain better grain of control than mutex. Read-writer lock has tree states: read-locked, write-locked and unlocked.
When it's write-locked, all threads that try to lock it will be blocked. When it's read-locked, threads that want to read-lock it will sucess,
but threads that want to write-lock it will be blocked until all the read locks are unblocked. To avoid starve a write lock, when an write lock
attempt is blocking, reader-writer lock will block additional read locks.

- create and destory
```
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock, const pthread_rwlockattr_t *restrict attr)
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock)
```

- read and write lock
```
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock)
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock)
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock)
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock)
int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock)
```

# condition variables
Condition variable supports for wait and notify operation. Multiple threads can wait a condition, and one thread can wake one of the threads
with notify.

- initialize and destory
```
int pthread_cond_init(pthread_cond_t *restrict cond, pthread_condattr_t *restrict attr)
int pthread_cond_destory(pthread_cond_t* cond) 
```

- wait for condition
```
int pthread_cond_wait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex)
int pthread_cond_timedwait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex, const struct timespec *restrict timeout)
```
Condition should be used with mutex. Mutex should first be locked, then passed to `pthread_cond_wait`. Then the thread will be blocked, and
mutex unlocked. When thread return from `pthread_cond_wait`, the mutex will be locked again. So commonly it will be used like:
```
pthread_mutex_lock(&mutex);           // mutex locked
pthread_cond_wait(&cond, &mutex);     // mutex unlocked during wait, and locked after return
pthread_mutex_unlock(&mutex);         // finally unlock mutex 
```

- wake up one or all threads
```
int pthread_cond_signal(pthread_cond_t *cond)
int pthread_cond_broadcast(pthread_cond_t *cond)
```

