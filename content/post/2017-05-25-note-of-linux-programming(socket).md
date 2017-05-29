---
title: "note of linux programming(socket)"
categories: ["tech"]
date: 2017-05-25
tags: ["linux"]
---

# UNIX domain socket
```
int socketpair(int domain, int type, int protocal, int sockfd[2])
```
This function creates a socket pair for communication. The created socket pair doesn't have name, so that can be used between unrelated
processes. So named UNIX domain socket should be used for this case.

## named UNIX domain socket
The "name" used by UNIX domain socket is a file of special type. The socket's address structure is:
```
struct sockaddr_un {
    sa_family_t sun_family; /* AF_UNIX */
    char sun_path[108]; /* pathname */
}
```
or 
```
struct sockaddr_un {
    unsigned char sun_len; /* length including null */
    sa_family_t sun_family; /* AF_UNIX */
    char sun_path[104]; /* pathname */
};
```
The `sun_path` field contains the pathname of the file. When we call `bind`, the a file with name `sun_path` will be created. If the file
already exists, the `bind` will fail. And we should unlink the file manually when process terminates.
In order to calculate the actual size of `sockaddr_un` in a portable way(the presence of `sun_len`), we use "offset of sun_path" + "strlen(sun_path)". 
The way to calculate the offset of `sun_path` is a tricky one: `offset(struct sockaddr_un, sun_path)`.
```
#define offsetof(TYPE, MEMBER) ((int)&((TYPE *)0)->MEMBER)
```
The meaning of this macro is: we cast the memory start from address 0 to `TYPE`(`sockaddr_un` in this case), and get its `MEMBER` field
(`sun_path` in this case). So the address of `MEMBER` will be the offset since the `TYPE` starts from 0.

For server, when connects come, it must first check if the file contained in the address is really a socket type, and the permission for the
file is rwx only for user, and checks the file is no older than 30s. These checks are to be sure the client is really the owner of the file.
For client, when it binds an address, a socket fill will be created. The client must make the file's permission to rwx only for user. The client
only needs to make sure that the socket file has a unique name, this can be achieved by appending its pid to the filename and `unlink` file of 
that name before `bind` incase of duplication.

