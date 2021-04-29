---
title: 健壮的I/O(RIO)
abbrlink: 8fd6efd8
date: 2019-10-19 11:52:47
tags:
categories:
---

在上篇[Unix系统级I/O](./Unix系统级I-O.md)中，我们介绍了有关在Unix环境下读取和写入文件的函数`read`和`write`，也提到了标准I/O在进行网络I/O时的局限性。但是在某些地方，直接使用`read`和`write`往往会出现不足值，比如在复杂的网络环境中读取socket。如果想让我们的程序更加的可靠，就需要反复的调用`read`和`write`去处理，知道传送完所需要的字节。在csapp一书中，给我们提供了一个健壮可靠的I/O包来自动处理这种不足值的情况，称为RIO（Robust I/O）。本文主要整理RIO提供的函数备忘。

详细代码及用法示例可以在[这里](https://github.com/Varpc/misc/tree/master/rio)找到。

## 无缓冲区的rio

`rio_readn`、`rio_writen`和`read`、`write`用法基本一致，只是`rio_readn`会不断尝试读出，直到读取出n个字节或遇到EOF或出错；`rio_writen`函数绝不会返回一个不足值，它会不断尝试写入直到写入全部字节或者出错。由于没有缓冲区的存在，`rio_readn`和`rio_writen`可以任意交替使用。

```C
#include "rio.h"

ssize_t  rio_readn(int fd, void *buf, size_t n);
/* 返回：若成功则为传送的字节数，若为EOF则为0，若出错则为-1 */

ssize_t  rio_writen(int fd, void *buf, size_t n);
/* 返回：若成功则为传送的字节数，若出错则为-1 */
```

函数定义如下：

```C
ssize_t rio_readn(int fd, void *usrbuf, size_t n)
{
    size_t nleft = n;
    ssize_t nread;
    char *bufp = usrbuf;

    while (nleft > 0) {
        if ((nread = read(fd, bufp, nleft)) < 0) {
            if(errno == EINTR)      /* Interrupted by big handler return */
                nread = 0;          /* and call read() again */
            else
                return -1;          /* errno set by read() */
        }
        else if (nread == 0)
            break;                  /* EOF */
        nleft -= nread;
        bufp += nread;
    }
    return (n - nleft);             /* return >= 0 */
}

ssize_t rio_writen(int fd, void *usrbuf, size_t n)
{
    size_t nleft = n;
    ssize_t nwritten;
    char *bufp = usrbuf;

    while (nleft > 0) {
        if ((nwritten = write(fd, bufp, nleft)) <= 0) {
            if(errno == EINTR)      /* Interrupted by big handler return */
                nwritten = 0;       /* and call write() again */
            else
                return -1;          /* errno set by write() */
        }
        nleft -= nwritten;
        bufp += nwritten;
    }
    return n;
}
```

## 带缓冲区的rio

带缓冲区的rio所包含的函数如下：

```C
#include "rio.h"

void rio_readinitb(rio_t *rp, int fd);
/* 返回：无 */

ssize_t rio_readlineb(rio_t *rp, void *usrbuf, size_t maxlen);
ssize_t rio_readnb(rio_t *rp, void *usrbuf, size_t n);
/* 返回：若成功则为读的字节数，若为EOF则为0，若出错则为-1 */
```

带缓冲区的rio由一个`rio_t`的结构体管理，其形式如下：

```C
#define RIO_BUFSIZE 8192

typedef struct {
    int     rio_fd;                 /* 描述符 */
    int     rio_cnt;                /* 缓冲区中还未读的字节数 */
    char    *rio_bufptr;            /* 缓冲区中下一个未读的字节 */
    char    rio_buf[RIO_BUFSIZE];   /* 缓冲区 */
} rio_t;
```

在使用带缓冲区的rio时，每打开一个描述符，都需要使用`rio_readinitb`来对`rio_t`进行初始化：

```C
void rio_readinitb(rio_t *rp, int fd) {
    rp->rio_fd = fd;
    rp->rio_cnt = 0;
    rp->rio_bufptr = rp->rio_buf;
}
```

对缓冲区的控制主要由`rio_read`来完成：

```C
static ssize_t rio_read(rio_t *rp, char *usrbuf, size_t n) {
    int cnt;

    while (rp->rio_cnt <= 0) {  /* Refill if buf is empty */
        rp->rio_cnt = read(rp->rio_fd, rp->rio_buf, sizeof(rp->rio_buf));
        if (rp->rio_cnt < 0) {
            if (errno != EINTR) /* Interrupted by sig handler return */
                return -1;
        } else if (rp->rio_cnt == 0)  /* EOF */
            return 0;
        else
            rp->rio_bufptr = rp->rio_buf; /* Reset buffer ptr */
    }

    /* Copy min(n, rp->rio_cnt) bytes from internal buf to user buf */
    cnt = n;
    if (rp->rio_cnt < n)
        cnt = rp->rio_cnt;
    memcpy(usrbuf, rp->rio_bufptr, cnt);
    rp->rio_bufptr += cnt;
    rp->rio_cnt -= cnt;
    return cnt;
}
```

> `rio_read`函数由`static`关键字修饰成静态函数，这对与一个函数来说，说明这个函数只对声明它的文件可见，且不同的文件可以声明相同名的静态函数。

`rio_readlineb`和`rio_readnb`的具体实现如下：

```C
ssize_t rio_readnb(rio_t *rp, void *usrbuf, size_t n) {
    size_t nleft = n;
    ssize_t nread;
    char *bufp = usrbuf;

    while (nleft > 0) {
        if ((nread = rio_read(rp, bufp, nleft)) < 0)
            return -1;          /* errno set by read() */
        else if (nread == 0)
            break;              /* EOF */
        nleft -= nread;
        bufp += nread;
    }
    return (n - nleft);         /* return >= 0 */
}

ssize_t rio_readlineb(rio_t *rp, void *usrbuf, size_t maxlen) {
    int n, rc;
    char c, *bufp = usrbuf;

    for (n = 1; n < maxlen; n++) {
        if ((rc = rio_read(rp, &c, 1)) == 1) {
            *bufp++ = c;
            if (c == '\n') {
                n++;
                break;
            }
        } else if (rc == 0) {
            if (n == 1)
                return 0;   /* EOF, no data read */
            else
                break;      /* EOF, some data was read */
        } else
            return -1;      /* Error */
    }
    *bufp = 0;
    return n - 1;
}
```