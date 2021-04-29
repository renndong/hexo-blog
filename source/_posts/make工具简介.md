---
title: make工具简介
tags: Linux
categories: Linux
abbrlink: 6aacce01
date: 2019-10-14 21:58:35
---

在Linux C/C++的开发过程中，当源代码文件较少时，我们可以手动使用gcc或g++进行编译链接，但是当源代码文件较多且依赖变得复杂时，我们就需要一种简单好用的工具来帮助我们管理。于是，make应运而生。

make主要用来管理C/C++项目，通过Makefile书写的规则来对项目中的源代码文件进行编译，生成可执行的程序。

## make流程

make执行的主要过程如下：当在shell中使用make命令时，make会寻找当前目录下的Makefile文件，根据该文件中的规则来确定依赖关系，如果一个文件所依赖的文件比这个文件要新，或者说修改时间更晚，那么make会根据Makefile中指明的命令来重新编译生成该文件。

另外，make除了自动寻找定义了编译规则的Makefile文件外，还可以手动指明定义了规则的文件。比如：
```
$ make -f rule.txt  # rule.txt中为make规则
```

## Makefile的写法

### 规则

Makefile由一系列的规则构成，一条规则的基本格式如下：

```
目标 : 条件 
[tab]  命令
```

其中，需要在命令之前加一个Tab制表符，并且条件和命令都是可以省略的，但是只能省略其一，条件省略时一般做一些编译以外的其他工作，当命令省略时其实也可以对目标进行编译生成，这涉及到了Makefile中的隐式规则，这里不过多赘述，我们只讨论显式规则。

如make流程所述，当条件中的文件比目标要新时，会执行tab后的命令。

Makefile中有很多规则时，当在shell中执行`make`命令，默认会将第一条规则的目标作为最终生成的目标。

比如下面这个Makefile例子：

```
main: main.o sub.o
	gcc -o main main.o sub.o

sub.o: sub.c sub.h
	gcc -c -o sub.o sub.c

main.o: main.c sub.h
	gcc -c -o main.o main.c

clean:
	rm sub.o main.o

.PHONY: clean
```

当我们在shell中执行make时，会最终生成`main`这个最终目标。

但是如果我们只想生成某个中间的目标也是可以的，比如只生成`sub.o`，只需要采用`make 最终目标`的形式就可以了，即`make sub.o`

注意到示例中省略了条件的那条规则（目标为clean的那条规则），正规上把它叫做伪目标，用来执行一些其他的任务，如本例中清除编译中生成的.o文件。当然，伪目标下的命令可以是多种多样的，比如将`clean`下的命令改为`ls`，当执行`make clean`时，会列出当前目录下的所有文件。但是有一点需要注意的是，如果我们的目录下已经有了一个叫做clean的文件，当我们执行`make clean`时，make就分不清这个clean到底是那个了，为了避免这种情况，需要用`.PHONY: 伪目标1，伪目标2..`的方式来显式的声明伪目标。

### 变量

当我们Makefile中的规则变得非常多时，为了方便，也为了可维护性，我们一般使用变量来代替某些信息。

Makefile中定义变量的格式如下，并用`$(变量名)`的形式来使用变量：

```
变量名 <赋值符> 变量值
```

其中赋值符可以是`=`、`:=`、`?=`、`+=`。它们的区别如下表：

| 符号 | 作用                                                                                                                     |
| ---- | ------------------------------------------------------------------------------------------------------------------------ |
| `=`  | 基本的赋值                                                                                                               |
| `:=` | 使用的是当前的值。例如：`y := $(x)`这句中`y`会被赋值到这句话时`x`的值，而`y = $(x)`会被赋值为`x`在当前Makefile中最终的值 |
| `?=` | 如果变量未被赋值，那么便被赋予`?=`后的值                                                                                 |
| `+=` | 变量添加`+=`后面的值                                                                                                     |


先前示例中便可精简如下：

```
CC = gcc
LD = gcc
CFLAGS = -c
OBJS = main.o \
	sub.o

main: $(OBJS)
	$(LD) -o  main $(OBJS)

sub.o: sub.c sub.h
	$(CC) $(CFLAGS) -o sub.o sub.c

main.o: main.c sub.h
	$(CC) $(CFLAGS) -o main.o main.c

clean:
	rm $(OBJS)
.PHONY: clean
```

> Tip: 有时候我们的规则可能太长，写在一行又不好看，可以使用`\`来进行换行。

### 内置变量

Makefile中还提供了一些内置变量，比如`$(CC)`代表默认的C编译器，`$(CXX)`代表默认的C++编译器。更多内置变量请参考[这里](https://www.gnu.org/software/make/manual/html_node/Implicit-Variables.html)

### 自动变量

Makefile中还提供了一些特殊的变量，不用定义且会根据所在的规则而改变，减少一些目标文件名和条件文件名的输入。以下是六个常用的自动变量：

| 变量名 | 作用                                               |
| ------ | -------------------------------------------------- |
| `$@`   | 目标的文件名                                       |
| `$<`   | 第一个条件的文件名                                 |
| `$?`   | 时间戳在目标之后的所有条件，并以空格隔开这些条件   |
| `$^`   | 所有条件的文件名，并以空格隔开，且排除了重复的条件 |
| `$+`   | 与`$^`类似，只是没有排除重复条件                   |
| `$*`   | 目标的主文件名，不包含扩展名                       |

根据以上自动变量，我们可以将上面的示例改成更简便的形式：

```
CC = gcc
LD = gcc
CFLAGS = -c
OBJS = main.o \
	sub.o

main: $(OBJS)
	$(LD) -o $@ $^

sub.o: sub.c sub.h
	$(CC) $(CFLAGS) -o $@ $<

main.o: main.c sub.h
	$(CC) $(CFLAGS) -o $@ $<

clean:
	rm $(OBJS)
.PHONY: clean
```

## 后记

另外，尽管make工具常常用来管理C/C++项目，但是用来管理其他项目也是可以的，比如汇编项目，Pascal项目，甚至是node.js的项目，make就是一个工具，来帮我们管理一些构建的规则，只要规则写的得当，怎么用就随你了。


最后，make虽然可以很好来管理项目了，但是还是不够方便。试想一下，当Makefile中的规则越来越多，又臭又长的时候，make就又显得很难用了，这也就是为什么cmake诞生的原因。通过编写Cmakelist，来指导cmake生成各种Makefile文件和project文件，从而减轻管理Makefile的负担。


---

参考：
- [GNU make](https://www.gnu.org/software/make/manual/make.html#Reading)