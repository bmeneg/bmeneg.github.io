# Kernel Crypto API userspace interface (also known as AF_ALG)

OK, lets start with it. Some time ago I was trying to understand how I use and
how actually works the default mechanism to interact an userspace application
with the kernel crypto API. I know there are other ways to do it, for instance,
cryptodev for linux, which creates a /dev/crypto node to handle send/recv calls
to encrypt and decrypt operations or even other userspace only implementations
like OpenSSL and others, but my passion for kernel space make me wonder why and
a default implementation for kernel linux was actually implemented and how does
it acutally works.

The first thing we need to realize is that AF\_ALG is actually a communication
protocol family, the same way AF\_INET is for the sockets in a TCP/IP
communication, so it's possible to deduce that this user kernel interface uses
the usual network socket interface with the usual functions to manipulate it:

```c
/* include <sys/socket.h> */
int socket(int domain, int type, int protocol);
int bind(int sockfd, const struct sockaddr *addr, socklen_t *addrlen);
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

/* include <unistd.h> */
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

and so forth, basically the entire \<sys/socket.h\> header file.

In my opinion it's very important to take some time to read this functions
manpage, directly from Linux manpage chapter 2, before start to dive into
AF\_ALG stuff, i.e.:

```bash
$ man 2 socket
```

## Getting in touch with AF\_ALG 

Everything you're going to need for starting your work over Kernel Crypto API in
userspace applications is to tell to the socket layer what do you really want
from it. When we speak about Cryptographic Algorithms we're going to need 
