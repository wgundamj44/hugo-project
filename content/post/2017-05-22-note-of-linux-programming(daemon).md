---
title: "note of linux programming(daemon)"
categories: ["tech"]
date: 2017-05-22
tags: ["linux"]
---

# daemon coding rule
- Use `umask` to set the file creation mask to 0. As the daemon process may inherit mask from parent, which does't have enough permission for
creating files.
- Call `fork` and have the parent `exit`. This makes sure that the child is not a group leader.
- Call `setsid` to create a new session. The child process becomes session leader, leader of a new group, and the session is without controlling
terminal. `fork` again, and have parent `exit` again. This makes sure that the new child is not a sesson leader anymore, which prevents the 
session to regain controlling terminal by opening a terminal device.
- Change the current working directory.
- Close unneeded file descriptors.
- Open file descriptors 0, 1, 2 to /dev/null, so libraries tries to interact with standard streams will have no effect.

# logging error
Deamon can't make error ouput to standard error stream, instead them use syslog. There're tree ways to generate logs:
- kernal routine use `log` to write logs, which can be read by read `/dev/klog` device.
- user routine call `syslog`. `syslog` makes message to be sent to `/dev/log` UNIX domain socket.
- log can also be sent to UDP port 514.
`syslogd` daemon read messages from all above.

```
void openlog(const char *ident, int option, int facility)
void syslog(int priority, const char *format, ...)
void closelog(void)
int setlogmask(int maskpri)
```
`openlog` and `closelog` are optional. `openlog` 's `ident` controls ident that is added to each log message, like name of the program. 
`option` is a bitmask specifying various options, like wether to log process ID to each message. `facility` specifies which facility does the 
message come, eg. `LOG_USER` means the message came from user process.

`syslog` actually generate log message. `priority` is combination of `facility` and level. Values of level could be `LOG_DEBUG`, `LOG_ERR`, etc.
`setlogmask` 

# single instance daemon
To prevent multiple instance of daemon to run at the same time, file lock can be used. Daemon creates a file, and place a write lock on the
file. When another daemon instance starts up, the lock attempt will fail, preventing that instance to execute.

# daemon conventions
- Daemons usually place their lock file in /var/run. File name is usually name.pid. Name is the name of the program.
- Daemons usually place their configuration file under /etc.
- Daemons are often started from init scripts.
- Often use SIGHUP to make daemon reload its configuration file. SIGHUP is used because daemon don't have controlling terminal, so SIGHUP is 
hardly used.

 