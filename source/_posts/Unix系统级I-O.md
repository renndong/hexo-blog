---
title: Unix系统级I/O
tags: Linux
categories: Linux
abbrlink: 790db60c
date: 2019-10-18 17:22:59
---

在Unix系统中，一且皆为文件。一个Linux文件就是一个字符序列，并且所有的I/O设备都被模型化成了文件。而所有的输入输出都被当作对对应文件的读和写。Linux提供了一组简单、低级的接口，使得所有的输入输出都可以用一种简单通用的方式来执行。

## Linux文件的分类

每一个文件都有一个类型（type）来表示它在系统中的角色，主要有以下几种：

- 普通文件。普通文件包括文本文件和二进制文件。
- 目录。目录包含一组指向其目录内的连接（link）
- 套接字文件。其主要用来和另外的进程进行跨网络通信。
- 管道。管道包括匿名管道和命名管道。用来进行进程间的通信。
- 符号链接。
- 字符和块设备等。

## 文件的打开与关闭

进程通过`open`函数打开或创建一个新文件：

```C
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int open(char *filename, int flags, mode_t mode);
/* 返回：若成功返回文件的描述符，若出错返回-1 */
```

> 这里的`filename`可能有些误解，不单单是指文件名，它还可以是相对或绝对路径名。

`open`函数将`filename`转换为一个文件描述符，返回的描述符总是在当前进程中**没有打开的最小描述符**。

> 关于文件描述符是如何分配的呢？在Linux中，每一个进程都会维护一个文件描述符表，这个表实现了一个整数到文件的映射关系。在描述符表中，0-2分别对应stdin、stdout和stderr文件。当在进程中要打开一个文件的时候，文件描述符会从3开始分配，然后是4、5...

`flags`指明了进程如何访问这些文件，具体如下：

| 掩码     | 描述                                         |
| -------- | -------------------------------------------- |
| O_RDONLY | 只读                                         |
| O_WRONLY | 只写                                         |
| O_RDWR   | 可读可写                                     |
| O_CREAT  | 如果文件不存在，就创建它的一个截断的空文件   |
| O_TRUNC  | 如果文件已存在，就截断它                     |
| O_APPEND | 在每次读写操作前，设置文件位置到文件的结尾处 |

上面这些掩码还可以叠加，例如：

```C
fd = open("foo.txt", O_WRONLY|O_APPEND, 0);
```

`mode`参数指明了如果要创建新文件时，新文件的访问权限位。如果只是普通的读写文件，`mode`一般为0。具体规则如下：

| 掩码    | 描述                               |
| ------- | ---------------------------------- |
| S_IRUSR | 使用者（拥有者）能够读这个文件     |
| S_IWUSR | 使用者（拥有者）能够写这个文件     |
| S_IXUSR | 使用者（拥有者）能够执行这个文件   |
| S_IRGRP | 拥有者所在组的成员可以读这个文件   |
| S_IWGRP | 拥有者所在组的成员可以写这个文件   |
| S_IXGRP | 拥有者所在组的成员可以执行这个文件 |
| S_IROTH | 其他人（任何人）可以读这个文件     |
| S_IWOTH | 其他人（任何人）可以写这个文件     |
| S_IXOTH | 其他人（任何人）可以执行这个文件   |

进程通过`close`函数来关闭一个已经打开的文件。关闭一个已关闭的描述符会出错。

```C
#include <unistd.h>

int close(int fd);
/* 返回：若成功返回0，出错返回-1 */
```

## 文件的读写

进程通过`read`和`write`函数来对文件进行读写操作:

```C
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t n);
/* 返回：若成功返回读的字节数，若EOF则为0，若出错则为-1 */

ssize_t write(int fd, void *buf, size_t n);
/* 返回：若成功返回写的字节数，若出错则为-1 */
```

例如将从标准输入文件读，写到标准输出文件：

```C
#include <stdio.h>
#include <unistd.h>

void main() {
        char c;
        while(read(STDIN_FILENO, &c, 1) != 0)
		write(STDOUT_FILENO, &c, 1);
}
```

> 需要说明一下的是，上面两个函数中，`ssize_t`的类型为`long`，而`size_t`的类型是`unsigned long`

## 读写文件的元数据

应用程序可以通过`stat`和`fstat`函数读取每个文件的详细信息（也叫做文件的元数据）

```C
#include <unistd.h>
#include <sys/stat.h>

int stat(const char *filename, struct stat *buf);
int fstat(int fd, struct stat *buf);
/* 返回：若成功返回0，出错返回-1 */
```

上面两个函数都通过填写用来描述文件信息的`stat`数据结构来获取文件的详细信息。

```C
struct stat { 
     dev_t              st_dev;             // 文件所在设备ID 
     ino_t              st_ino;             // inode编号 
     mode_t             st_mode;            // 保护模式和文件类型
     nlink_t            st_nlink;           // 硬链接个数  
     uid_t              st_uid;             // 所有者用户ID  
     gid_t              st_gid;             // 所有者组ID  
     dev_t              st_rdev;            // 设备ID(如果是特殊文件) 
     off_t              st_size;            // 总体尺寸，以字节为单位 
     unsigned long      st_blksize;         // 文件系统 I/O 块大小
     unsigned long      st_blocks;          // 已分配块个数
     time_t             st_atime;           // 上次访问时间 
     time_t             st_mtime;           // 上次更新时间 
     time_t             st_ctime;           // 上次状态更改时间 
};
```

Linux建议我们使用在`sys/stat.h`中定义的宏谓词来确定`st_mode`成员的文件类型，例如：

- `S_ISREG(m)`：这是一个普通文件吗？
- `S_ISDIR(m)`：这是一个目录文件吗?
- `S_ISSOCK(m)`：这是一个网络套接字文件吗？

其他宏谓词就不再赘述。

## 读取目录内容

应用程序可以通过`readdir`系列函数来读取文件的内容：

```C
#include <sys/types.h>
#include <dirent.h>

DIR *opendir(const char *name);
/* 返回：若成功，返回处理的指针，否则，返回NULL */
```

`opendir`返回指向目录流的指针。流是对条目有序列表的抽象，这里是指目录项的列表。

可以使用`readdir`函数来读取目录流中的内容。

```C
#include <dirent.h>

struct dirent *readdir(DIR *dirp);
/* 返回：若成功，则返回下一个目录项的指针；若没有更多目录或出错，者为NULL */
```

每个目录项都是一个结构，其形式如下：

```C
struct dirent {
        ino_t  d_ino;           /* inode */
        char   d_name[256];     /* 文件名 */
        /* 虽然在某些Linux系统中还包括其他成员，但上面两个成员对所有Unix系统都是通用的 */
};
```

最后，使用`closedir`来关闭流并释放所有资源：

```C
#include <dirent.h>

int closedir(DIR *dirp);
/* 返回：若成功为0，错误为-1 */
```

下面给出一个读取指定目录文件的例子：

```C
#include <stdio.h>
#include <sys/types.h>
#include <dirent.h>

void main(int argc, char **argv) {
	DIR *streamp;
	struct dirent *dep;
	
	streamp = opendir(argv[1]);
	while((dep = readdir(streamp)) != NULL) {
		printf("%s\n", dep->d_name);
	}
	closedir(streamp);
}
```

## I/O重定向

这里稍稍提及一下I/O重定向。上文在介绍文件的打开与关闭的时候提到，每一个进程都会维护一个文件描述符表，这个描述符表的表项由进程打开的文件描述符来索引，描述符表项实现了描述符到真实文件的映射关系。但是，如果我们改变这种映射关系，这便是文件重定向。

文件重定向可以使用`dup2`函数：

```C
#include <unistd.h>

int dup2(int oldfd, int newfd);
/* 返回：若成功返回非负的描述符，若出错返回-1 */
```

`dup2`实现了将`newfd`重定向到`oldfd`，这其中实际上是使用`oldfd`的表项去覆盖`newfd`表项以前的内容，如果`newfd`已经打开，则先关闭`newfd`再进行覆盖。并且，如果`newfd`所指向的文件的引用计数变为0，则会释放相应的资源（包括打开文件表、v-node表中的表项）。例如，可以将标准输出重定向到一个文件：

```C
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>

void main() {
	int fd;
	char c;

	fd = open("foo.txt", O_WRONLY, 0);
	dup2(fd, 1);
	while(scanf("%c", &c) != 0) {
		write(1, &c, 1);
	}
	close(fd);
	exit(0);
}
```

## 标准I/O

C语言中提供了标准的I/O库，里面定义了一组高级的输入输出函数。例如：

- 打开和关闭文件的`fopen`和`fclose`
- 读和写字节的`fread`和`fwrite`
- 读和写字符串的`fgets`和`fputs`
- 从流中格式化读取和写入的`fscanf`和`fprintf`
- 以及`printf`、`scanf`、`fprintf`等等。

标准I/O库将打开的文件模型化成一个流，一个流是一个指向`FILE`结构体的指针。另外，每一个ASCI C程序在开始时都打开三个流`stdin`、`stdout`和`stderr`。

`FILE`流是对文件描述符和流缓冲区的抽象，因为缓冲区的存在，提高了读取文件的性能。但是标准I/O也不是完美的，也正是由于某些缓冲区的存在，使得标准I/O在网络I/O方面出现短板。

## 小结

以上，主要讨论了如下几个I/O相关的接口：

- 用于打开和关闭文件的`open`和`close`
- 用于读取和写入文件的`read`和`write`
- 用于读取文件元数据的`stat`和`fstat`，以及判断文件类型的宏谓词
- 用于读取目录的`opendir`、`readdir`和`closedir`
- C语言提供给我们的更加高级别的I/O接口

---