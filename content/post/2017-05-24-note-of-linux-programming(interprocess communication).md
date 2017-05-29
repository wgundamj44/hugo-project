---
title: "note of linux programming(Interprocess communication)"
categories: ["tech"]
date: 2017-05-24
tags: ["linux"]
---

# pipe
Pipe is pair of file descriptor through which process can communicate. Pipe should be used a half duplex and can be only used between processes
with same ancestor.

```
int pipe(int filedes[2])
```
`pipe` function will open a pair of file descriptors `filedes`. `filedes[0]` is for reading and `filedes[1]` is used for writing.
A common use of pipe for sending data to child process is: The parent opens a pairs of descriptors, then it forks a child, the child will 
inherit the copy of the same descriptors. The parent closes the descriptor for reading and the child closes the descriptor for writing.

## popen and pclose
`popen` combines `pipe`, `fork` and `system` together. 
```
#include <stdio.h>
FILE *popen(const char *cmdstring, const char *type)
```
It executes `cmdstring` just as `system` command and return a `FILE*`. The returned `FILE*` is readable if `type` is "r", writable if `types` 
is "w".

# coprocess
If we open two pipes in a process A, and connect another process B's standard input and standard ouput to these two pipes, then B is called A's
coprocess. Coprocesses are often used as filters.

# FIFOs
FIFO is file like, and can be used for communiate between unrelated processes.
```
#include <sys/stat.h>
int mkfifo(const char *pathname, mode_t mode)
```
`mkfifo` creates a FIFO named `pathname`. `mode` is the same as used in `open`.
FIFO has two uses.
- It can be used by shell commands to pass data to from one shell pipeline to another without creating intermediate temporary files.
eg.
```
mkfifo fifo1
prog3 < fifo1 &
prog1 < infile | tee fifo1 | prog2
```
In this example, `prog3` reads from fifo1, and `tee` command reads from its standard input are output to file filo1 and to its standard output
which will further be piped to `prog2`. This effectively send infile to both prog3 and prog2 without using intermediate file.

- It can be used in server-client structure where a server creates and reads from a well-known FIFO, and clients all writes to that FIFO.

# XSI IPC
XSI IPC includes message queue, semaphore and shared memory. XSI IPC objects are identified by kernal with an identifier, but for processes to
get access to these objects to commnunicate with each other, an external key with which they agreed upon should be used.
```
key_t ftok(const char* path, int id)
```
This functon can be used to generate a key for IPC object. Every cooperating process will agree on some `path` and `id`, and use this function
to generate key that they will use. `path` should be an existing file, and only `id`'s lower 8 bits will be used.

XSI IPC object also has a `ipc_perm` structure which controls their permissions. 
```
struct ipc_perm {
    uid_t uid; /* owner's effective user id */
    gid_t gid; /* owner's effective group id */
    uid_t cuid; /* creator's effective user id */
    gid_t cgid; /* creator's effective group id */
    mode_t mode; /* access modes */
.
.
.
}
```
`mode` is similar to that of file, except execution bit is not needed.

## message queue
A message queue is a linked list stored in kernel and identified by message queue identifier.
Message queue is get or created by `msgget`, a message is sent with `msgsnd` and received with `msgrcv`.
A message queue has a `msqid_ds` structure.
```
struct msqid_ds {
struct ipc_perm msg_perm; /* see Section 15.6.2 */
msgqnum_t msg_qnum; /* # of messages on queue */
msglen_t msg_qbytes; /* max # of bytes on queue */
pid_t msg_lspid; /* pid of last msgsnd() */
pid_t msg_lrpid; /* pid of last msgrcv() */
time_t msg_stime; /* last-msgsnd() time */
time_t msg_rtime; /* last-msgrcv() time */
time_t msg_ctime; /* last-change time */
.
.
.
};
```
This structure is initialized on queue creation, and can altered with `msgctl`.

```
#include <sys/msg.h>
int msgget(key_t key, int flag)
```
This function get or create a message queue and returns a message ID, which can be used by other functions.

```
int msgctl(int msqid, int cmd, struct msqid_ds *buf )
```
`cmd` can be `IPC_GET`, `IPC_SET` and `IPC_RMID`, they mean get, modify and remove a message queue respectively. The latter two can only be used
by process whose owner user ID equals to `msg_perm.uid` or `msg_perm.cuid`, or process is owned by superuser.

```
int msgsnd(int msqid, const void *ptr, size_t nbytes, int flag)
```
`ptr` points the message to be sent, and `nbytes` is the length of the message payload. A message consists of a positive integer message type
followed by a message payload. Eg. if a message payload is 512 bytes, then the `ptr` will point to a structure:
```
struct mymesg {
long mtype; /* positive message type */
char mtext[512]; /* message data, of length nbytes */
};
```  

```
ssize_t msgrcv(int msqid, void *ptr, size_t nbytes, long type, int flag)
```
`type` can be used to specify how we want to receive the data.
- type == 0: the first message on the queue will be received
- type > 0: the first message on the queue with `mtype` equals to `type` will be received
- type < 0: the message with lowest `mtype` which less than abs(type) will be received.

## semaphores
A semaphore is a counter used by multiple processes to provide access to a shared data object. Theoretically, a semaphore is a non-negative 
integer sepcifying number of availble resources. A process can obtain some of resources by decreasing the value of the integer, or release 
resources by increasing the value. A process will be blocked if the desired value is larger than current value.
Unix's semaphore is not a simple non-negative integer, but a set of semaphore values. Each semaphore value in the set has the structure:
```
struct {
unsigned short semval; /* semaphore value, always >= 0 */
pid_t sempid; /* pid for last operation */
unsigned short semncnt; /* # processes awaiting semval>curval */
unsigned short semzcnt; /* # processes awaiting semval==0 */
.
.
.
}
```

```
#include <sys/sem.h>
int semget(key_t key, int nsems, int flag)
```
This function get or create a semaphore with `key` and return the semaphore's ID.

```
int semctl(int semid, int semnum, int cmd, ... /* union semun arg */)
```
This function do various operations to a semaphore in the set, like set value, get value, get information, etc.

```
int semop(int semid, struct sembuf semoparray[], size_t nops)
```
This function do obtain or release semaphore values operations to a set. `semoparray` is array contains the separate operation to each value in
the set.

## shared memory
Shared memory allows multiple processes to share a given region of memory.
Each shared memory region has the structure:
```
struct shmid_ds {
struct ipc_perm shm_perm; /* see Section 15.6.2 */
size_t shm_segsz; /* size of segment in bytes */
pid_t shm_lpid; /* pid of last shmop() */
pid_t shm_cpid; /* pid of creator */
shmatt_t shm_nattch; /* number of current attaches */
time_t shm_atime; /* last-attach time */
time_t shm_dtime; /* last-detach time */
time_t shm_ctime; /* last-change time */
.
.
.
};
```
Similarly to other XSI IPC, a shared memory region can be created or get by:
```
#include <sys/shm.h>
int shmget(key_t key, size_t size, int flag)
```

```
int shmctl(int shmid, int cmd, struct shmid_ds *buf)
```
This function is used to get or set `shimid_ds` associated with memory region.

When a shared memory is get, a process can attach to that region with:
```
void *shmat(int shmid, const void *addr, int flag)
```
`addr` and `flag` determines how the semgent will be attached. This function returns a pointer to the shared memory.

Finally, process can detach from shared memory with:
```
int shmdt(void *addr)
```
This does not remove the shared memory. The shared memory can only be removed with `shmctl`.