---
title: "note of linux programming(process)"
categories: ["tech"]
date: 2017-04-27
tags: ["linux"]
---

This is note of linux programming about process controll. Topic includes creating new process with fork, exec or system, exit from process with
exit or _exit, interact with other process with waitpid and various user ids(effective, real and saved).
<!--more-->

# identification of process
A process can be identified by its process group id and process id. We can obtain these information with following functions:

- get process ID of the calling process
```
pid_t getpid(void)
```
- get parent process ID of the calling process
```
pid_t getppid(void)
```
- get real user id of the calling process
```
pid_t getuid(void)
```
-  get effective user id of the calling process
```
pid_t geteuid(void)
```
- get real group id of the calling process
```
pid_t getgid(void)
```
- get effective group id of the calling process
```
pid_t getegid(void)
```

# fork
```
pid_t fork(void)
```
`fork` is used to create a new process. This new process the child process of the calling process.
`fork` is strange in that, after the call, the following source code will be executed by two processes. This is why we usually needs code like
this:
```
if ((pid = fork()) == 0) {
    // pid == 0 means child    
    // do the child process task
} else {
    // parent
}
```

## file sharing
All the file descriptor opened in the parent will be duplicated by child as if we have called `dup`. 
![file sharing](/2017-04-27-file-sharing.png)
As we can see from the figure, parent and child share the same file table, so the file offset is also shared. That means if one process writes some bytes to a file, the file offset of another process will also change.

Besides file, there're lots of other things will be inheritted by child, like real/effective user id, process group id, enviroment, signal masks, etc.

# exit
```

```
A process can be terminated normally in five ways:
- `return` from main function. This is equivalent to calling `exit`
- `exit`. This function will call exit handlers and close all standard I/O streams.
- `_exit` or `_Exit`. These functions terminate process without calling exit handlers or signal handlers. And on UNIX system, these functions
don't flush standard I/O streams.
- Executing return from start routine of last thread in process. At this condition, the return value won't be used as return value of process,
instead the process will terminate with status 0.
- Calling `pthread_exit` from last thread in process. The exit status of process will always be 0.

A process can be terminated abnormally in three ways:
- `abort`. 
- Receiving some signal
- The last thread responds to a cancellation request.

Parent of a process can examine the termination status with several macros:
- `WIFEXITED` return true if a child is terminated normally. `WEXITSTATUS` can be used to fetch the lower 8 bits of the argument the child has passed to `_exit`.
- `WIFSIGGNALED` return true if a child terminated as a result of an uncaught singal. `WTERMSIG` can be used to fetch the signal number.
- `WIFSTOPPED` return true if a child is currently stopped. `WSTOPSIG` can be used to get the signal number.
- `WIFCONTINUED` return true if child has been continued.

## zombie
If process's parent exited before it, init process will become the new parent of this process. Normally, parent should use `wait` to fetch the 
terminate status of the child, otherwise the child will become a zombie. Init process is special in that, it will automatically call `wait`
for each of its terminated child, so that no zombie will be generated.
If process's child terminates before it can call `wait`, the kernal will keep a small record of that terminated process, so that when parent finally calls `wait`, it will still be able to get the result.

# wait
```
pid_t wait(int *statloc)
pid_t waitpid(pid_t pid, int *statloc, int options)
```
These functions wait for child terminates. `statloc` will hold the process' termination information. It's a integer, some bits represents the 
exit status, some bits represent signal number, etc.
`wait` will block until one of the process's child terminates.
`waitpid` can wait for child process of certain pid, or certain group id, etc. `waitpid` can be makde non-block with `options`.

# exec
exec will replace current process with new program, process id will not be changed, but stack, head, text, data segments are all got replaced.
exec has many forms:
```
int execl(const char* pathname, const char* arg0, ...)
int execv(const char* pathname, char* const argv[])
int execle(const char* pathname, const char* arg0, ..., (char*)0, char* const envp[])
int execve(const char* pathname, char* const argv[], char* const envp[])
int execlp(const char* pathname, const char* arg0, ...)
int execvp(const char* pathname, char* const argv[])
```
Functions end with `l` will pass arguments one by one while functions end with `v` will pass them as array. Functions end with `p` will use
 `PATH` to search for pathname. Functions end with `e` will pass enviroment list instead of `environ` global.

After exec:
 - FD_CLOEXEC with close-on-exec set on file descriptor, process will close all opened file after exec.
 - real user ID will remain the sames.
 - If set-user-ID bit is set for new program, the effective user ID of process will become owner ID of the program file.

 # setuid
 ```
 int setuid(uid_t uid)
 ```
 `setuid`'s effect varies based on the process.
 - IF the process has superuser privilege, `setuid` set real/effective and saved set-user-ID to uid.
 - otherwise, if uid equals to real user ID or saved set-user-ID, `setuid` set effective user ID to `uid`. 

 ## there types of uid
 - real user ID: this can only be set by superuser
 - effective user ID: when program executed by exec has set-user-ID bit set, exec will set effective user ID to owner ID of the program file.
 - saved set-user-ID: copied from effective user ID by exec. Note if the program's set-user-ID bit is set, the copied effective ID is the one 
 that has been updated to owner ID of the program file.

 # interpreter file
 Interpreter file is a file starts with `#!`, eg. bash script often has `#! /bin/sh`.
 The strange part is, when we use exec to execute a interpreter file, the `argv[0]` argument will be replaced by `pathname`.
 eg:
 ```
 //echoarg.c, program to execute
 int main(int argc, char* argv[]) {
  int i = 0;
  for(i = 0; i < argc; i++) {
    printf("argv[%d]: %s\n", i, argv[i]);
  }
  return 0;
}

//inter, interpreter file
# ./echoarg test1

// test.c, exec an interpreter file
execl("./inter", "xx", "yy", (char*)0)
// will get:
// argv[0]: ./echoarg
// argv[1]: test1
// argv[2]: ./inter
// argv[3]: yy
```
We can see that first argv parameter of `execl` `xx` is dispeared, replaced by the pathname `./inter`.

# system
```
int system(const char* cmd)
```
`system` can be used execute a command string, like `system("ls")`.
`system` works as the combination of `fork`, `exec` and `waitpid`.

## never use `system` from a set-user-ID program
Otherwise, when executing this program, the effective user ID of the process generated by `system` will be the owner ID of the set-user-ID 
program. For example, if the program is owned by root, everyone will gain root privilege by using that program. 
The correct way is to `fork` and `exec`, before `exec` restore the user's effective user ID to normal permissions.

