---
layout: post
title: 在Archlinux环境下配置TinyOS开发环境
tags:
- Embedded
- Operating system
categories: Linux
---

最近拿到了Dr. Ke Shi发的DEVISER TelosB节点，由于Dr.给的文档是基于Ubuntu14.04的，而我使用的是Archlinux。并且发现在Arch环境下配置还有些不同的地方，于是收集资料整理了这一篇在Archlinux环境下配置TinyOS的教程。详细说明了在Arch环境下需要安装的软件包，配置的环境变量和py版本的处理等问题。最后还附加了一个烧写亮灯程序的示例。

## DEVISER TelosB节点介绍

TelosB节点(TPR2420)最初由Berkeley大学研发、发布，具有低功耗和快速苏醒功能，可以保证更长的电池寿命，与开源TinyOS兼容。TPR2420把所有要素都集中在一个独立的平台上：USB编程能力，IEE802.15.4 射频器和天线，低功耗扩大内存得MCU和可选的传感器套件。

下图即为TelosB节点：

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Arch-TinyOS-01.jpg)

## 必要的软件包

Arch下很多和tinyos相关的包都已经在AUR里了，我们可以方便的下载如下几个包：

```
yaourt -S tinyos
yaourt -S tinyos-tools
yaourt -S nesc
yaourt -S gcc-avr-tinyos
yaourt -S binutils-avr-tinyos
```

>安装过程中你可能会看到AUR里还有一个avr-libc-tinyos包，我有尝试装过，但是始终安装不上，后来在没有安装这个包的情况下成功了，推测可能是功能重复或者暂时没有用上的包

系统默认的tinyos应该被安装到了`/opt`目录下，这里对其进行一系列的修改需要root权限，为了方便自己使用，我把它移动到了自己的home目录下。

## 环境变量配置

接下来设置环境变量，我这里建议将需要配置的环境变量写到一个脚本里，在你的终端的配置文件的包含这个脚本。

例如，在tinyos目录下创建一个`tinyos.sh`，写入以下内容：

```sh
#! /usr/bin/env bash
# To setup the environment variables needed by the TinyOS
TOSROOT="$HOME/tinyos-2.1.2"
TOSDIR="$TOSROOT/tos"
CLASSPATH="$TOSROOT/support/sdk/java/tinyos.jar:."
MAKERULES="$TOSROOT/support/make/Makerules"

export TOSROOT TOSDIR CLASSPATH MAKERULES
```

然后在你的shell配置文件(例如我的是.zshrc)里写入如下两行

```sh
#Sourcing the TinyOS environment variables
source $HOME/tinyos-2.1.2/tinyos.sh
```

## 交叉编译器安装

这里安装`msp430`，AUR中有如下的包：

```
yaourt -S msp430mcu
yaourt -S mspgcc-ti
yaourt -S mspdebug
yaourt -S msp430-libc
yaourt -S gcc-msp430
yaourt -S binutils-msp430
```

这里不会发生什么问题，需要注意的就是和`msp430`相关的很多AUR中的包已经过时了，安装时注意一下AUR中的Out of Date标识。

## 环境检查

执行命令`tos-check-env`，应该看到类似如下的显示。

```
linux> tos-check-env
Path:
	/usr/local/bin
	/usr/local/sbin
	/usr/bin
	/opt/cuda/bin
	/usr/lib/jvm/default/bin
	/opt/ti/mspgcc/bin
	/usr/bin/site_perl
	/usr/bin/vendor_perl
	/usr/bin/core_perl

Classpath:
	/home/(your user name)/tinyos-2.1.2/support/sdk/java/tinyos.jar
	.



rpms:


nesc:
	/usr/bin/nescc
	Version: nescc: 1.3.6


perl:
	/usr/bin/perl
	Version: v5.26.1) built for x86_64-linux-thread-multi

flex:
	/usr/bin/flex

bison:
	/usr/bin/bison

java:
	/usr/bin/java

--> WARNING: The JAVA version found first by tos-check-env may not be version 1.4 or version 1.5one of which is required by TOS. Please ensure that the located Java version is 1.4 or 1.5

graphviz:
	/usr/bin/dot
	dot - graphviz version 2.40.1 (20161225.0304)

--> WARNING: The graphviz (dot) version found by tos-check-env is not 1.10. Please update your graphviz version if you'd like to use the nescdoc documentation generator.


tos-check-env completed with errors:

--> WARNING: The JAVA version found first by tos-check-env may not be version 1.4 or version 1.5one of which is required by TOS. Please ensure that the located Java version is 1.4 or 1.5
--> WARNING: The graphviz (dot) version found by tos-check-env is not 1.10. Please update your graphviz version if you'd like to use the nescdoc documentation generator.
```

其中，关于java和graphviz的警告是由于java版本较新造成的，不需要关心。

执行命令`printenv MAKERULES`，应该看到如下的显示。

```
linux> printenv MAKERULES
/home/(your user name)/tinyos-2.1.2/support/make/Makerules
```

如果执行命令得到的结果基本一致，则说明配置完好了。

## 烧写第一个程序

将节点电源断开[注意：非常重要，这里一定不要打开电源，否则会导致节点烧坏]，用USB连接到电脑，使用`motelist`命令查看输出。

```
linux > motelist
Reference  Device           Description
---------- ---------------- ---------------------------------------------
FTOJLCD    /dev/ttyUSB0     FTDI USB <-> Serial Converter
```

这里`/dev/ttyUSB0`是该节点在Linux下对应的文件，在之后的操作中会遇到。

这里参考Dr. Ke Shi给的文档，`motelist`命令应该只适用于telos B节点，其他的节点应用`dmesg`命令，应该可以看到如下的输出。

```
[  709.846133] usb 1-3: FTDI USB Serial Device converter now attached to ttyUSB0
```

接下来执行编译程序，进入LED程序目录，进行编译。

```sh
cd $TOSROOT/apps/Blink
sudo make telosb install /dev/ttyUSB0
```

但是，我们却遇到了如下的问题：

```
ModuleNotFoundError: No module named 'serial'
```

原来是没有安装Python的模块？那么我们pacman安装试试？

```
sudo pacman -S python-pyserial
```

结果仍然无法编译，思考之后发现Dr.给的文档是基Ubuntu的，而Ubuntu下默认的Python是py2版本，于是我更换了py2的串口通信包。

```
sudo pacman -S python2-pyserial
```

奇怪的是，我们遇到了如下的输出：

```
AttributeError: 'Serial' object has no attribute 'setBaudrate'
```

还是不行？在Google查找相关资料后，在github issue上找到了同样的问题，原来是这个对象out of date了，在新版的包中，setBaudrate已经被替换掉了。于是我在网上找到了老版本的包，链接如下，手动编译安装老版本。

[pyserial2.7](https://pypi.python.org/pypi/pyserial/2.7)

注意，在手动安装了这个包之后，pacman不会记录下它，如果你尝试使用pacman安装新版本的pyserial，则会发生冲突。当你需要的时候再删掉它并安装新的包就好了。

最后，执行编译命令，搞定！指示灯马上亮起来！

```
sudo make telosb install /dev/ttyUSB0
```