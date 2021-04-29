---
title: 'AlphaZ OS: 开发环境配置'
tags:
  - 操作系统
  - AlphaZ
categories: AlphaZ
abbrlink: 8801d300
date: 2019-12-12 22:03:28
---

{% note success %}
该篇是有关操作系统自制的系列文章，请参考[目录](/AlphaZ/)
{% endnote %}

开发环境配置起来很简单，只要五步就可以

开始之前，请先更新一下包列表和已安装的包

```bash
$ sudo apt update
$ sudo apt upgrade
```

## 安装构建环境

打开终端，执行下面的命令，安装需的编译器和链接器等软件，包括gcc、g++、gdb、make等，并且下面安装bochs时也需要这些构建软件

```bash
$ sudo apt install build-essential
```

## 安装nasm

nasm是一个开源的汇编语言编译器，其安装方式如下：

```bash
$ sudo apt install nasm
$ nasm --version
NASM version 2.13.02
```

## 安装bochs

bochs是一个开源的并支持调试功能的虚拟机，它是我们开发操作系统时用于调试的主要工具。

你可以在[这里](https://sourceforge.net/projects/bochs/files/bochs/2.6.10/bochs-2.6.10.tar.gz/download)下载bochs

现在完成后，进入下载好的压缩包所在的目录，执行：

```bash
$ tar xzvf bochs-2.6.10.tar.gz
$ cd bochs-2.6.10/

# 由于bochs依赖gtk，首先安装
$ sudo apg-get install libgtk2.0-dev

# 一定要加上 --enable-debugger --enable-disasm 打开bochs的调试功能
# 由于bochs的中并为指明X11所在的目录，需要手动指定，避免报错
$ ./configure --enable-debugger --enable-disasm --x-include=/usr/include/X11 --x-lib=/usr/lib/x11
$ make
$ make install
```

至此，我们要使用的工具就安装完毕了


## 创建虚拟软盘

现在我们的虚拟机还没有启动盘，还没有办法工作。bochs中有一个bximage程序可以供我们创建一个虚拟软盘。我们在此后的开发中，便可以将我们写的程序烧录到该软盘中，然后供bochs启动。


在shell中输入`bximage`命令，便可以按下面的步骤创建一个虚拟软盘了。
```
$ bximage 
========================================================================
                                bximage
  Disk Image Creation / Conversion / Resize and Commit Tool for Bochs
         $Id: bximage.cc 13481 2018-03-30 21:04:04Z vruppert $
========================================================================

1. Create new floppy or hard disk image
2. Convert hard disk image to other format (mode)
3. Resize hard disk image
4. Commit 'undoable' redolog to base image
5. Disk image info

0. Quit

Please choose one [0] 1

Create image

Do you want to create a floppy disk image or a hard disk image?
Please type hd or fd. [hd] fd

Choose the size of floppy disk image to create.
Please type 160k, 180k, 320k, 360k, 720k, 1.2M, 1.44M, 1.68M, 1.72M, or 2.88M.
 [1.44M] 

What should be the name of the image?
[a.img] 

Creating floppy image 'a.img' with 2880 sectors

The following line should appear in your bochsrc:
  floppya: image="a.img", status=inserted
```

## bochs虚拟机配置文件

现在有了虚拟机又有了虚拟软盘，离我们的目标又进了一步。但是有了这些后我们的虚拟机该如何工作呢，这就需要我们给bochs一个配置文件。告诉我们虚拟机启动时加载哪个虚拟盘、给虚拟机多少内存等等。你可以直接在工作目录下创建一个文本文件，写入以下内容：

```
# 内存32M
megs: 32

romimage: file=/usr/local/share/bochs/BIOS-bochs-latest

vgaromimage: file=/usr/local/share/bochs/VGABIOS-lgpl-latest

# 虚拟机的盘
floppya: 1_44=a.img, status=inserted

# 启动盘
boot: floppy

log: bochsout.txt

mouse: enabled=0
# 键盘映射
keyboard: keymap=/usr/local/share/bochs/keymaps/x11-pc-us.map
```

将文件保存后，在shell中输入`bochs -f <配置文件>`即可启动我们的虚拟机了。需要说明的一点是，如果你将配置文件命名为`bochsrc`，那么直接输入`bochs`就可启动我们的虚拟就了，bochs会在当前目录下自动寻找名叫`bochsrc`的文件作为自己的配置文件。


至此，我们的环境配置完成了。但是如果你启动bochs虚拟机的除了报错外并无其他，那是因为我们的启动盘只是一个空盘，并没有程序供bochs启动。下面，我们来写一个简单的程序烧录进软盘，用bochs虚拟机运行，来验证一下我们的环境。

## 验证

进入你的工作目录，创建一个`test.asm`文件，并在其中写入以下内容并保存：

```
org         7c00h

    mov	ax, 0xb800
	mov	gs, ax
	mov	ah, 0x02
	mov	al, 'S'
	mov	[gs:((80*10+40)*2)], ax

	hlt

times	510-($-$$) db 0
	dw	0xaa55
```

该程序的作用是在屏幕中央显示一个绿色的`S`

按照下面的命令来对程序进行编译并烧录进虚拟软盘的第一个扇区：

```
$ nasm -o test.bin test.asm 
$ dd if=test.bin of=a.img bs=512 count=1 conv=notrunc 
记录了1+0 的读入
记录了1+0 的写出
512 bytes copied, 0.000354724 s, 1.4 MB/s
```

现在输入`bochs`，即可启动bochs虚拟机了。按照虚拟机的提示，第一步直接回车，然后再键入`c`，回车，如果你在虚拟机的屏幕中央看到一个绿色的`S`（如下），那么恭喜你，环境配置成功。

{% img /images/AlphaZOS开发环境配置.png %}

---