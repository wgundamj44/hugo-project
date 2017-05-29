---
title: "note of linux programming(process control)"
categories: ["tech"]
date: 2017-05-15
tags: ["linux"]
---
This chapter focuses on relationship between multiple processes. Processes can form process groups. Process groups can form sessions. Job is
one or more processes that work together.
<!--more-->

# login

## terminal login
Unix system starts with `init` command. `init` command reads `/etc/ttys` file, and forks for each line of that file, then each child process
will exec a `getty`. The `getty` opens terminal device and display "login:" prompt on terminal. When we enter our username, `getty` will further
exec `login`. This `login` will grap our username, and prompt us with password, then do the authentications. If success, the `login` will set
up necessary permissions and finally start up a shell for user.

Note in nowadays, `systemd` will be used instead of `init`.

## network login
When login through network, `init` will brings up a `inetd` process. This process will wait for TCP/IP request to host. For instance, when a
TELNET request comes, `init` will fork a new `telnetd` process which will open a pseudo terminal and further split into two processes, one for
network communication, one for `login`. User will pass their user/pass through TELNET, and `login` will do the authentication.

In my centos enviroment, when doing network login, `sshd` will be used.

# process group
- get process group id of the calling process
```
pid_t getpgrp(void)
pid_t getpgid(pid_t pid)
```

A process with same process id as its process group is call process group leader. A process group will remain as long as there's at least one
process in the group, even when its leader terminates.

- set process group for a process
int setpgid(pid_t pid, pid_t pgid)
This function set a process `pid`'s group id to `pgid`. If pid is 0, the caller's process id will be used. If the pgid is 0, then pid will be 
used. If pid equals to pgid, the process pid will becomes the group leader. One can only call this function on itself or its children. If its 
child has called exec, then its child's group id can't be set by this function.

# session
Several process groups can form a session. A process group can be the result of pipeline, like `proc1 | proc2`.

- create a new session
```
 pid_t setsid(void)
```
This function must be called in a process which is not alread a process group leader. After the call, the calling process will become leader of 
a new process group, and the session leader of new session. The process will have no controlling terminal.

- get session leader's group id (can be considered as the 'id' of the session)
```
pid_t getsid(pid_t pid)
```

# controlling terminal
- Session may have a single controlling terminal.
- Session leader that establishes the connection to the controlling terminal is called controlling process.
- If a session has controlling terminal, then it may have one foreground process groups and multiple background process groups.
- Interrupt signal and quit signal generated from terminal are only sent to foreground process.

- get process group id of the foreground process, `filedes` is the descriptor of the controlling terminal.
```
pid_t tcgetpgrp(int filedes)
```
- set foreground process group id to `pgrpid`
```
int tcsetpgrp(int filedes, pid_t pgrpid)
```
- get process group id of the session leader
```
pid_t tcgetsid(int filedes)
```

# job and controlling terminal
In terminal with job control we only interact with foreground process through terminal. When a background process reads from terminal, a SIGTTIN signal will be raised to stop the process. When a background process tries to write to terminal, we can control whether to allow it with `stty` command. If we disallow it, a SIGTTOU will be raised. In terminal without job control, the background process will automatically
redirects standard input to /dev/null, so the read to standard input to result a EOF immediatly.

In a terminal that supports job control, if we invoke a command from shell normally, it will be in a new process group and connects to the 
controlling terminal. If we invoke a command in background, it will still be in a new process group but controlling terminal will still be held
by its parent process(the shell). In terminal that doesn't have job control, the new process will be in the same procsss group with shell and
controlling terminal will not be taken away.

In a terminal that supports job control, in a pipeline like `ps | cat`, both process will has shell as their parent. Otherwise, `cat`'s parent
will be shell, and `ps`'s parent will be `cat`, that is the last process in the pipeline will be parent of all the process ahead.

# orphaned process group
A process group is orphaned if the parent of every process in group is either in this group or in the other session. That is, if there's a
process in the group whose parent is in another group of the same session, then this process group is not an orphan.

Orphan process group is special in that, if some processes in the group is stopped, there's no parent in other process group that can awake
them. So the system will send every stopped process in an orphaned group a SIGHUP signal then a SIGCONT signal. This will give them a chance to
continue.

