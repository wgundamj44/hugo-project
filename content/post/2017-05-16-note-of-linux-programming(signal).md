---
title: "note of linux programming(signal)"
categories: ["tech"]
date: 2017-05-16
tags: ["linux"]
---

# signal
- install a signal handler, returns previous disposition of signal
```
void (*signal(int signo, void (*func)(int)))(int)
```
This function accepts two parameters, the first is the signal name, eg. SIGINT, SIGTTIN, etc. the second is a function pointer to a function,
which takes int as parameter and returns nothing. This function will also return a function pointer to a function with same signature as its
second parameter. The `func` can also be `SIG_IGN` or `SIG_DFL`

When we exec a new program, the signal will get back to their default status. If the process that's doing exec is ignoring some signals, then
these signals will remain ignored. When we fork a new process, the signal disposition will be inherited by child.

# interrupted system calls
Some system calls may be interrupted by signal. For example, when `read` a terminal, as the terminal may have no input forever, when signal
occurs, the `read` maybe interrupted, return with errno set to EINTR. So we may need such code to resume the system call:
```
again:
    if ((n = read(fd, buf, BUFFSIZE)) < 0) {
        if (errno == EINTR) goto again; // do the system call again
    }
```
The operating system may provide the ability to automatically restart the system call.

# reentrant function
A function may be interrupted by signal during its execution. Then in the signal handler, it's not always safe if we execute the same function
again. Eg. a function may use static data structure, and maybe interrupted in the middle of altering the data structure. Another call to
this function may override or corrupt that data structure, so it can't be used before the previous finishes. Generally, functions that a. use static data structure, b. use malloc or free, c. use standard I/O library can't be used reentrantly.

# kill and raise
- `kill` sends signal to a process, `raise` sends signal to itself
```
int kill(pid_t pid, int signo)
int raise(int signo)
```
When pid > 0, signal is sent to process with pid. When pid == 0, signal is sent to processes in the same group. When pid < 0, signal is sent to
processes in the group with group id -1 * pid. When pid == -1, signal is sent to all processes. To make signal sent to another process, one
needs adequate permissions. The real/effective user id should be the same as the target process's real user id or saved set-user-id id.

# alarm and pause
- timer
```
unsigned int alarm(unsigned int seconds)
```
Set a timer, when expires will send SIGALRM signal. This function returns 0 or seconds until previously set alarm expires.
When `seconds` is 0,  the previous alarm will be canceled and the seconds remained will be returned. When `seconds` is not 0, the previous alarm
is reset to new value and remaining seconds will be returned.

- pause
```
int pause(void)
```
This function stops the process until a signal handler is executed and returns.

# signal set
Signal set is used to hold masks of multiple signals.
```
int sigemptyset(sigset_t *set)
int sigfillset(sigset_t *set)
int sigaddset(sigset_t *set)
int sigdelset(sigset_t *set)
```
As the name implies, these functions will clear all signals, set all signals, turn on some of the signals and turn off some signals respectively.

```
int sigismember(const sigset_t *set, int signo)
```
checks if a signal is in `set`.

# examine and change signal mask
```
int sigprocmask(int how, const sigset_t * restrict set, sigset_t * restrict oset)
```
If `oset` is a non-null pointer, the current signal mask is returned through oset.
If `set` is non-null pointer, then `how` will indicate how the current signal mask is to be handled.
- SIG_BLOCK: signals contained in `set` will be blocked along with current signal masks
- SIG_UNBLOCK: signals contained in `set` will be unblocked.
- SIG_SETMASK: current signal masks will be replaced by `set`
A signal is masked means that signal is blocked.
If `set` is a null pointer, signal mask is not changed, and `how` is ignored.

Note when calling `sigprocmask`, if an ublocked signal is pending, then before function returns, at least one of these signals will be delivered
to process. `deliver of signal` means the action of that signal is taken.

# sigpending
```
int sigpending(sigset_t *set)
```
get the signals that are blocked and pending

# sigaction
```
int sigaction(int signo, const struct sigaction *restrict act, struct sigaction *restrict oact)
```
This function examine and modify the action associated with a signal.
If the `act` pointer is not null, we are modifying  the action. If the `oact` pointer is not null, the previous action will be returned through
`oact`.
`action` is defined as:
```
struct sigaction {
    void (*sa_handler)(int);
    sigset_t sa_mask;
    int sa_flags;
    void (*sa_sigaction)(int, siginfo_t *, void *);
}
```
If `sa_handler` is not null, then signals in `sa_mask` will be added to masks of process before signal handler is invoked. So we can ensure 
certain signals will be blocked during the execution of the signal handler. Also, the signal itself will also be added to mask, so additional
occurrance of the same signal will also be blocked. When signal handler returns, these blocked signals will be unblocked.
`sa_flags` specifies options for handling the signals, and `sa_sigaction` is used instead of `sa_handler` if `sa_flags` is SA_SIGINFO.

Once action is installed for signal, it will remain installed until we explictly change it with another sigaction call.
This function can be used to implement a reliable `signal` function.

# siglongjump
- do the longjump with signal mask restored
```
int sigsetjmp(sigjmp_buf env, int savemask)
void siglongjmp(sigjmp_buf env, int val)
```
When we make longjump within a signal handler, the signal of the handler will remain blocked after the jump. To overcome this problem, this
`siglongjmp` is introduced. When `savemask` is nonzero, the current process mask will be saved in `env`, and be restored after longjump.

# sigsuspend
```
int sigsuspend(const sigset_t *sigmask)
```
This function set the signal mask to `sigmask`, and pause until some signal is caught. After the return of signal handler, this function will
return and restore the signal mask.

# abort
```
void abort(void)
```
This function sends SIGABRT signal to the caller. This will cause abnormal program termination. SIGABRT will ultimately terminate a process,
event if we cought the signal.

# system
`system` function basically equals to a combination of `fork`, `exec` and `waitpid`. But it also needs to consider signal issues. First, it is 
required that SIGINT and SIGQUIT be ignored during the execution of `system`. Because `system` may be used to invoke an interactive program like
ed or vim. SIGINT and SIGQUIT are only intended for the interactive program not the caller of `system`. Second, SIGCHLD should be blocked. As
caller of `system` may also has other children. The SIGCHLD caused by termination of program invoked by `system` will trick the caller of
 `system` into thinking that one of its other child are terminated.

# sleep
```
unsigned int sleep(unsigned int seconds)
```
This function cause the calling process to suspend until `seconds` of time has passed or a signal is delivered. 
Note that `sleep` can be implemented either with `alarm` or without `alarm`. If it is implmeneted with `alarm`, then the interaction between
`sleep` and `alarm` may bring complexity.

