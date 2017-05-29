---
title: "note of linux programming(Advanced I/O)"
categories: ["tech"]
date: 2017-05-23
tags: ["linux"]
---

# Nonblocking I/O
Nonblocking I/O makes I/O operations to return an error immediately if the operation cannot be completed.
There're two ways to specify nonblocking for a given descriptor:
- call `open` with O_NONBLOCK flag
- call `fnctl` to turn on the O_NONBLOCK flag for an existing file descriptor

# record locking
Recording locking is used to lock a portion of a file. Lock can be read lock or write lock. Write lock is exclusive, meaning if one process
holds a write lock, all the other process won't be able to hold read or write lock on the overlapping portion.

Note that record locking can be advisory or mandatory. If it is advisory, then event if we put a write lock on a portion of a file, another
process can still be able to write to that region. The advisory lock is meaningful only when the processes doing I/O are 'cooperative'.
Processes are cooperative if they will try to obtain the write lock before it writes, tries to obtain the read lock before reads. Mandatory lock
on the other hand, will enforce check on every open, read and write, to see if the operation is violating the lock policy. Advisory lock is the
default kind of lock. If we want to turn on mandatory lock, we should make the file's set-group ID bit to 1, and group execute bit to 0.

## fcntl record locking
```
int fnctl(int filedes, int cmd, struct flock *flockptr)
```
To record lock a file with `fnctl`, the `cmd` should be one of `F_GETLK`, `F_SETLK` or `F_SETLKW`, and `flockptr` is a pointer to a `flock`
structure.
```
struct flock {
short l_type; /* F_RDLCK, F_WRLCK, or F_UNLCK */
off_t l_start; /* offset in bytes, relative to l_whence */
short l_whence; /* SEEK_SET, SEEK_CUR, or SEEK_END */
off_t l_len; /* length, in bytes; 0 means lock to EOF */
pid_t l_pid; /* returned with F_GETLK */
};
```
- `l_type`: `F_RDLCK` means read lock, `F_WRLCK` means write lock and `F_UNLCK` means unlock.
- `l_pid`: contains the pid of the process holding the lock that will block the current process.

If a process is holding a lock on some range of a file, a subsequent attempt a hold a lock on the same range will success and replace the
existing lock with the new one.

To obtain a write lock, the descriptor must be opened for writing, to obtain a read lock, the descriptor must be opened for reading.

Record lock DOES NOT prevents `unlink`.

About the `cmd` of `fnctl`:
- F_GETLK is used to check wether lock described in `flockptr` will be blocked by other processes. If it will, `l_pid` field of `flockptr` will
contain the process id that will block this lock. If it won't be blocked, then `l_type` will be set to `F_UNLCK`.
- F_SETLK set the lock described by `flockptr`. If the lock attempt is blocked by others, it will return immediatly with errno set to EACESS or 
EAGAIN.
- F_SETLKW is the blocking version of `F_SETLK`.

## lock inheritance
- locks are associated with process and a file. This means if the process terminates, the lock owned by the process will be released. Also, if
the descriptor associated with the file closed, the lock owned by the process will also be released.
- locks are NOT inherited throught fork.
- locks are inherited through exec, if close-on-exec flag is OFF.


# memory mapped I/O
Memory mapped I/O maps a file on disk to portion of memory, so that I/O operations to buffer will reflect to file.

## map a file to memory
```
#include <sys/mman.h>
void *mmap(void *addr, size_t len, int prot, int flag, int filedes, off_t off )
```
`addr` is the starting address of memory, usually set to 0. `len` is number of bytes to map, `filedes` is the descriptor of the file we want to
map, `off` is the offset in the file. `propt` is the protection flag of the mapped region. `flag` can be `MAP_FIXED`, `MAP_SHARED` or 
`MAP_PRIVATE`. This function returns the starting address of the memory mapped.

`flag` effects the behavior of the function:
- MAP_FIXED: makes the return value equals to `addr`.
- MAP_SHARED: makes the write operation modify the mapped file.
- MAP_PRIVATE: makes the first write operation generate a seprate copy of the original file. Succesive writes to happen to that new file.

## other operations
```
int mprotect(void *addr, size_t len, int prot)
``` 
This function changes the protection flag of an existing region.

```
int msync(void *addr, size_t len, int flags)
```
This function flushes the changes of the memory to the file.

```
int munmap(caddr_t addr, size_t len)
```
This function unmap the region. This happens automatically when process terminates.

# I/O multiplexing

## select and pselect
```
#include <sys/select.h>
int select(int maxfdp1, fd_set *restrict readfds, fd_set *restrict writefds, fd_set *restrict exceptfds, struct timeval *restrict tvptr)
```
`select` listens to descriptors in `readfds`, `writefds` and `exceptfds`. If one of the descriptors become readable, this function returns.
On return, we can check wether a fd is ready for read by `FD_ISSET(fd, &fds)`.
`tvptr` is struct:
```
struct timeval {
long tv_sec; /* seconds */
long tv_usec; /* and microseconds */
};
```
When `tvptr` is NULL, the `select` will wait forever or be interrupted. When tvptr->tv_sec == 0 and tvptr->tv_usec == 0, `select` will not block
at all. Otherwise, `select` will block for specified time.

```
int pselect(int maxfdp1, fd_set *restrict readfds, fd_set *restrict writefds, fd_set *restrict exceptfds, const struct timespec *restrict tsptr,
const sigset_t *restrict sigmask);
```
`pselect` is similar, expect `tsptr` is constant, and `sigmask` can be used to block certain signals.


## poll
```
#include <poll.h>
int poll(struct pollfd fdarray[], nfds_t nfds, int timeout)
```
`poll` does similar things as `select`. `fdarray` is an array of struct `pollfd`. This function returns number of fds that are ready.
```
struct pollfd {
int fd; /* file descriptor to check, or <0 to ignore */
short events; /* events of interest on fd */
short revents; /* events that occurred on fd */
};
```
In each struct, `events` contain the events we're intrested, `revents` will contain the events actually happend on return.

# STREAMS 
Don't understand...