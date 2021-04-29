---
title: Unix套接字接口
tags: Linux
categories: Linux
abbrlink: 28c7bd33
date: 2019-11-20 17:40:22
---

## 简介

套接字是操作系统中用于网络通信的重要结构，它是建立在网络体系结构的传输层，用于主机之间数据的发送和接收，像web中使用的http协议便是建立在socket之上的。这一节主要讨论网络套接字。

套接字接口时一组函数，它们和Unix I/O结合起来，用于创建网络应用。许多操作系统都实现了自己的套接字接口。在Unix中，可以将套接字视为一个文件，使用文件I/O函数对套接字进行操作，这也贯彻了Unix中一切皆文件的思想。

既然是网络通信，那么就需要服务端和客户端，一个基本的客户端和服务端的通信模型如下，其中方框里为使用的函数接口：

{% img /images/Unix套接字接口.png %}

由于套接字本质上也是一个文件，所以上图中的`recv`和`send`也可以使用文件I/O函数`read`和`write`替代。

## 套接字地址结构

既然要进行网络通信，那么就要有基本的网络的地址。因特网的套接字地址存放在一个叫`sockaddr_in`的16字节的结构体中，其中的IP地址和端口号都是以网络字节序（大端序）存放的：

```C
struct sockaddr_in {
    uint16_t        sin_family;     /* 协议类型，总是AF_INET */
    uint16_t        sin_port;       /* 端口号 */
    struct in_addr  sin_addr;       /* IP地址 */
    unsigned char   sin_zero[8];    /* sizeof(struct in_addr) */
};

struct sockaddr {
    uint16_t    sin_family;         /* 协议类型 */
    char        sa_data[14];        /* 地址信息 */
}
```

注意到，上面还有一个`sockaddr`结构体类型。由于`connect`、`bind`和`accept`都需要接受一个指向与协议相关的套接字地址结构指针。但是在设计这些接口时，如何定义这些接口，使之能够接受各种类型的套接字地址。为了解决这个问题，设计了`sockaddr`这种通用的结构体，在使用这些接口时，都需要将与协议相关的结构体转化为这个通用的结构体。

## 创建和关闭套接字

使用`socket`函数创建一个套接字，函数原型如下：

```C
#include <sys/types.h>
#include <sys/socket.h>

/* 返回：若成功则为非负描述符，失败返回-1 */
int socket(int domain, int type, int protocol);
```

参数`domain`（域）用于确定通讯的特性，具体如下：

| 域        | 描述         |
| --------- | ------------ |
| AF_INET   | IPv4因特网域 |
| AF_INET6  | IPv6因特网域 |
| AF_UNIX   | UNIX域       |
| AF_UPSPEC | 未指定       |

参数`type`确定套接字的类型，进一步确定通讯特性：

| 类型           | 描述                                            |
| -------------- | ----------------------------------------------- |
| SOCK_DGRAM     | 固定长度的，无连接的，不可靠的报文传递          |
| SOCK_RAW       | IP协议的数据报接口                              |
| SOCK_SEQPACKET | 固定长度的，有序的，可靠的，面向连接的报文协议  |
| SOCK_STREAM    | 有序的，可靠的，双向的，面向连接的字节流（TCP） |

参数`protocol`通常是0，表示使用默认协议，其他选项这里不再赘述。

例如可以像下面这样创建一个套接字：

```C
fd = socket(AF_INET, SOCK_STREAM, 0);
```

其中，`AF_INET`表示我们使用32位IPv4地址，`SOCK_STREAM`表示我们使用TCP协议。但是最好的方法使用`getaddrinfo`函数来自动生成这些函数，这样我们的代码就与具体的协议无关了。我们会在下方进行讲述。

`socket`函数返回的描述符只是部分打开的，还不能直接使用，取决于是作为客户端还是服务端，需要一些初始化工作。

`shutdown`可以用来禁止一个套接字的I/O：

```C
#include <sys/socket.h>

/* 返回：若成功返回1，若出错返回-1 */
int shutdown(int sockfd, int how);
```

参数`how`指明了关闭套接字的方式：

| 标志      | 描述     |
| --------- | -------- |
| SHUT_RD   | 关闭读端 |
| SHUT_WR   | 关闭写端 |
| SHUT_RDWR | 关闭读写 |

尽管使用`close`函数也够关闭套接字，但是和`shutdown`函数却有两点不同：当对一个套接字描述符调用`close`，该套接字的引用计数会减一，只有当引用计数为0时才会真正关闭套接字，而`shutdown`不管引用计数是否为0，都会关闭套接字。另外，`shutdown`可以只关闭套接字读写两个方向中的一个方向，保留另一个方向继续工作。

## 绑定地址

作为服务器，需要给套接字关联一个众所周知的地址，这样别人才能使用这个地址来对服务器进行访问。

使用`bind`函数来关联地址和套接字：

```C
#include <sys/socket.h>

/* 返回：若成功，返回0，出错，返回-1 */
int bind(int sockfd, const struct sockaddr *addr, socklen_t len);
```

`bind`函数将告诉内核将`addr`中的服务器套接字地址和套接字`sockfd`关联起来。其中`len`为`sizeof(sockaddr_in)`。

对于使用的服务器地址有如下限制：
- 指定的地址必须有效，不能是其他机器的地址
- 地址必须和创建套接字时的地址族所支持的格式相匹配
- 地址中的端口号必须不小于1024，除非该进程有相应特权
- 一般只能将一个套接字端点绑定到一个地址。


## 建立监听

作为服务器，为了接受用户的连接，需要监听某个特定的端口。当客户请求连接这个端口时，便创建一个连接。

默认情况下，内核会认为`socket`函数创建的套接字为**主动套接字**，它存在于用于连接的客户端。调用`listen`函数来告诉内核描述符是被服务器所使用的，而不是客户端。

```C
#include <sys/socket.h>

/* 返回：成功返回0，失败返回-1 */
int listen(int sockfd, int backlog);
```

`listen`函数将一个主动套接字转化为一个**监听套接字**，该套接字可以接受来自客户端的连接请求。`backlog`参数指明请求队列中未完成连接的数量。当队列已满，进程将拒绝后续的连接请求。

## 等待连接

一但服务器调用了`listen`，所用的套接字就可以用来接受连接请求，使用`accept`函数来接受客户端的连接请求。

```C
#include <sys/socket.h>

/* 返回：若成功，返回非负连接描述符，失败返回-1 */
int accept(int sockfd, struct sockaddr *restrict addr, socklen_t *restrict len);
```

`accept`返回的描述符是一个新的**已连接的描述符**，这个套接字描述符用来和客户端进行通信。这个新的套接字和原始套接字（`sockfd`）具有相同的套接字类型和地址族，传送给`accept`的原始套接字并没有关联到这个连接，继续可用并接受其他连接。

`accept`函数会将客户端的地址信息填充到参数`addr`指向的缓冲区，参数`len`为缓冲区的大小。如果不需要这些信息。可以将`addr`和`len`设为NULL。

如果没有连接请求，`accept`会阻塞到一个连接请求的到来。如果`sockfd`处于非阻塞模式，则返回-1，并将`errno`设置为`EAGAIN`或`EWOULDBLOCK`。

## 建立连接

对于客户端来说，如果要处理一个面向连接的网络服务（`SOCK_STREAM`或`SOCK_SEQPACKET`）。在交换数据之前，首先要在客户端和服务端之间建立一个连接。

使用`connect`函数来建立一个连接：

```C
#include <sys/socket.h>

/* 返回：成功返回0，出错误返回-1 */
int connect(int sockfd, const struct sockaddr *addr, socklen_t len);
```

`connect`函数会试图与套接字地址为`addr`的服务器建立一个连接，其中`len`为`sizeof(sockaddr_in)`。`connect`函数会阻塞，直到完成连接的建立或出错。随后，描述符`sockfd`便可进行读写了。

## 发送和接收

可以使用`send`函数来发送数据：

```C
#include <sys/socket.h>

/* 返回：若成功，返回发送的字节数，若出错，返回-1 */
ssize_t send(int sockfd, const void *buf, size_t nbytes, int flags);
```

`send`函数与用于文件读写的`write`函数基本一致，只是多了第四个参数`flags`，一般置为0。

使用`recv`来接收数据:

```C
#include <sys/socket.h>

/* 返回：返回数据的字节长度，若无可用数据或对方已经按序结束，返回0，若出错返回-1 */
ssize_t recv(int sockfd, void *buf, size_t nbytes, int flags);
```

`recv`函数与`read`函数基本一致，只是多了第四个参数`flags`，来管理如何接收数据，一般置为0。

## 主机和服务的转换

在前面对套接字的创建中，我们采用手动指明协议的方式，但是这样编写并不恰当。Linux提供了一些函数用于主机名和服务名与地址之间的相互映射，使用这些函数可以使我们的程序独立于任何特定版本的IP协议。下面看一下这些函数。

`getaddrinfo`函数用于将主机名，主机地址，服务名和端口号的字符串表示转化为套接字地址结构。

```C
#include <sys/socket.h>
#include <netdb.h>

/* 返回：若成功返回0，若出错返回非零错误码 */
int getaddrinfo(const char *restrict host,
                const char *restrict service,
                const struct addrinfo *restrict hint,
                strcut addrinfo **restrict res);

/* 返回：无 */
void freeaddrinfo(struct addrinfo *ai);

/* 返回：错误消息字符串 */
const char *gai_strerror(int errcode);
```

`host`参数指定主机地址，可以是域名或者点分十进制的IP地址。`service`参数可以是服务名也可以是十进制的端口号。主机名和服务名可以两者都提供，也可以只提供一个，但另一个必须是一个空指针。

`getaddrinfo`函数返回一个链表结构`addrinfo`，`res`指向该结构的第一个节点。使用`freeaddrinfo`函数可以释放该结构。`addrinfo`结构的定义至少包含以下成员：

```C
struct addrinfo {
    int             ai_flags;       /* customize behavior */
    int             ai_family;      /* address family，first arg to socket function */
    int             ai_socktype;    /* socket type，second arg to socket function */
    int             ai_protocol;    /* protocol，third arg to socket function */
    char            *ai_canonname;  /* canonical name for hostname */
    socklen_t       ai_addrlen;     /* length in byte of address */
    struct sockaddr *ai_addr;       /* address */
    struct addrinfo *ai_next;       /* next in list */
    ...
};
```

`getaddrinfo`可以指定一个可选的`hint`来过滤符合条件的地址，包括`ai_family`、`ai_flags`、`ai_protocol`、`ai_socktype`字段。剩余的字段必须置为0或空指针。

下表总结了`ai_flags`字段的标志，这些标志可以通过or运算指定多个：

| 标志           | 描述                                                                                                                  |
| -------------- | --------------------------------------------------------------------------------------------------------------------- |
| AI_ADDRCONFIG  | 查询配置的地址类型（IPv4或IPv6）。使用连接时推荐。只有当本地主机被配置为IPv4时，`getaddrinfo`返回IPv4地址。对IPv6同样 |
| AI_ALL         | 查找IPv4和IPv6地址（仅用于AI_V4MAPPED）                                                                               |
| AI_CANONNAME   | 需要一个规范的名字（与别名相对）                                                                                      |
| AI_NUMERICHOST | 以数字格式指定主机地址                                                                                                |
| AI_NUMERICSERV | 将服务指定为数字端口号                                                                                                |
| AI_PASSIVE     | 套接字地址用于监听绑定                                                                                                |
| AI_V4MAPPED    | 如果没有找到IPv6地址，返回映射到IPv6格式的IPv4地址                                                                    |

如果`getaddrinfo`出错，可以将返回的错误码传入`gai_strerror`函数获取具体的错误信息。


接下来的`getnameinfo`函数与`getaddrinfo`函数功能相反，用于将一个地址转化为一个主机名和一个服务名：

```C
#include <sys/socket.h>
#include <netdb.h>

/* 返回：若成功返回0，出错返回非0值 */
int getnameinfo(const struct sockaddr *restrict addr, socklen_t alen,
                char *restrict host, socklen_t hostlen,
                char *restrict service, socklen_t servlen,
                int flags);
```

`getnameinfo`将字节长度为`alen`的套接字地址`addr`翻译成一个主机名和一个服务名，如果`host`非空，则指向一个字节长度为`hostlen`的缓冲区用于存放主机名。同样的，如果`service`非空，则指向一个长度为`servlen`的缓冲区来存放服务名。`flags`参数指定了函数工作的方式，如下：

| 标志            | 描述                                                                                                                        |
| --------------- | --------------------------------------------------------------------------------------------------------------------------- |
| NI_DGRAM        | 服务基于数据报而非基于流                                                                                                    |
| NI_NAMEREQD     | 如果找不到主机名，将其作为一个错误对待                                                                                      |
| NI_NOFQDN       | 对于本地主机，仅返回全限定域名的节点名部分                                                                                  |
| NI_NUMERICHOST  | 函数默认返回host中的域名，该标志指定返回主机的数字形式，而非主机名                                                          |
| NI_NUMERICSCOPE | 对于IPv6，返回范围ID的数字形式，而非名字                                                                                    |
| NI_NUMERICSERV  | 函数默认检查/etc/services，如果可能会返回服务名而不是端口号。该标志指定跳过检查，返回服务地址的数字形式（端口号），而非名字 |


下面一个来自书中的例子展示域名到它IP地址的映射：

```C
/* main.c */
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MAXLEN 8192

int main(int argc, char **argv) {
    struct addrinfo *p, *listp, hints;
    char buf[MAXLEN], buf1[MAXLEN];
    int rc, flags;

    if(argc != 2) {
        fprintf(stderr, "argv error\n");
        exit(-1);
    }

    memset(&hints, 0, sizeof(struct addrinfo));
    hints.ai_family = AF_INET;
    hints.ai_socktype = SOCK_STREAM;
    if((rc = getaddrinfo(argv[1], NULL, &hints, &listp)) != 0) {
        fprintf(stderr, "getaddrinfo error: %s\n", gai_strerror(rc));
        exit(-1);
    }

    flags = NI_NUMERICHOST;
    for(p = listp; p; p = p->ai_next) {
        getnameinfo(p->ai_addr, p->ai_addrlen, buf, MAXLEN, buf1, MAXLEN, flags);
        printf("%s %s\n", buf, buf1);
    }

    freeaddrinfo(listp);
    exit(0);
}
```

使用如下：

```C
$ gcc main.c -o main
$ ./main baidu.com
39.156.69.79 0
220.181.38.148 0
```

## 小结

unix提供了一组函数用于创建套接字并完成数据传输：
- 使用`getaddrinfo`函数和`socket`函数来创建套接字，可以使程序更具有通用性。
- 服务端使用`bind`给套接字绑定地址，使用`listen`指定套接字用于监听，并使用`accept`接受连接
- 客户端使用`connect`来连接到服务器
- 客户端和服务端都可以使用`send`和`recv`来传递数据
- 使用`close`或`shutdown`函数来关闭套接字

---

总结自《深入理解计算机系统》《Unix环境高级编程》