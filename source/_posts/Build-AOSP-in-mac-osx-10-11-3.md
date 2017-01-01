---
title: Build AOSP in mac osx 10.11.3
date: 2016-05-01 16:01:03
tags: Android AOSP
---


### 配置OSX系统

1. 安装``Xcode``
2. 找到``Xcode``安装目录，右键打开``show package content``
3. 创建``/Developer/SDK``目录
4. 将``MacOSX10.11.sdk``从``Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/``复制到``/Developer/SDK``
5. 将``Xcode.app``复制到到/Developer目录下

#### 完成``Xcode``的配置

1. 打开``Xcode``
2. 从``Xcode``的菜单打开``Preferences``
3. 选择``Location``标签
4. 在``Command Line Tools``中选择``Xcodd 7.2``(具体版本依个人电脑上的Xcode版本而定)

#### 安装``MacPorts``

1. 获取最新版的[MacPorts](https://www.macports.org/install.php)!
2. 打开控制台，编辑``bash``配置文件

```
$ vim ~/.bash_profile
```

3. 在配置文件中插入

```
$ export PATH=/opt/local/bin:$PATH
```

4. 重新载入配置信息

```
$ . ~/.bash_profile
```

5. 运行``port``命令检测是否成功安装

```
$ port
```

6. 安装需要的依赖包

```
$ POSIXLY_CORRECT=1 sudo port install gmake libsdl git gnupg
```

#### 安装``java``开发包(JDK1.7)

#### 安装``Repo``

``Repo``是基于``git``的一种工具，能很方便的用来管理``Android``源码。更多信息请[查看](http://source.android.com/source/developing.html)

1. 确保根目录下有``/bin``目录

```
$ mkdir ~/bin
$ PATH=~/bin:$PATH
```

2. 下载``Repo``工具，确保它可以运行

```
$ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
$ chmod a+x ~/bin/repo
```

#### 设置文件描述符
``OSX``系统的默认文件描述符的数量太小，高并行构建可能会超出这个限制。为了突破这个限制，添加以下命令到``.bash_profile``中

```
$ vim ~/.bash_profile
~~~~~
# set the number of open files to be 1024
ulimit -S -n 1024

```

#### 创建字符大小写敏感的虚拟驱动

```
$ hdiutil create -type SPARSE -fs 'Case-sensitive Journaled HFS+' -size 100g ~/android.dmg
```
源码很大，因此建议要创建大一点的虚拟光驱

挂载这个驱动:

```
hdiutil attach ~/android.dmg -mountpoint /Volumes/android
```

你也可以用另一种方式，在配置文件中加入如下方法：

```
function mountAndroid { hdiutil attach ~/android.dmg.sparseimage -mountpoint /Volumes/android; }
```

然后运行如下命令:

```
mountAndroid
```

### 下载源文件

首先我们确定我们要下载的是哪个版本的源码，本文以``android-6.0.1_r30``为例

首先在``/Volumes/android``创建一个工作目录

```
$ cd /Volumes/android
$ mkdir WORKING_DIRECTORY
$ cd WORKING_DIRECTORY
```

运行
```
$ repo init -u https://android.googlesource.com/platform/manifest -b android-5.0.0_r7.0.1
```

最后运行
```
repo sync
```
来下载源码

### 编译源码

在源码根目录运行
```
$ source build/envsetup.sh
```

运行
```
$ lunch
```

按"1"选择``aosp_arm-eng``。

运行
```
make -j4
```

开始漫长的编译过程

运行
```
make idegen && development/tools/idegen/idegen.sh
```
生成可被``IDE``识别的工程。

### 碰到的一些问题

* 下载源码时，需要比较长的时间，还好支持断点续传。(被盾的地方，必须使用VPN)
* 源码下载完之后，编译会碰到找不到MacOSsdk。按照文中开头的步骤即可解决
* 编译过程中会出现
```
fatal error: linux/netfilter/xt_DSCP.h: No such file or directory
```

这是由于我的源码没有直接在``/Volumns/android``下下载，是从别的地方下载完之后拷贝过来的。而编译源码会严格区分大小写的。相应目录下有小写的头文件。

解决方案是：

1. 严格按照官方文档描述，在相应目录下下载源码
2. 在``./external/iptables/include/linux/netfilter``中创建头文件``xt_DSCP.h``。在该头文件中加入如下代码:

```
/*
* based on ipt_FTOS.c (C) 2000 by Matthew G. Marsh <mgm@paktronix.com>
* This software is distributed under GNU GPL v2, 1991
*
* See RFC2474 for a description of the DSCP field within the IP Header.
*
* xt_DSCP.h,v 1.7 2002/03/14 12:03:13 laforge Exp
*/
#ifndef _XT_DSCP_TARGET_H
#define _XT_DSCP_TARGET_H
#include <linux/netfilter/xt_dscp.h>
#include <linux/types.h>

/* target info */
struct xt_DSCP_info {
    __u8 dscp;
};

struct xt_tos_target_info {
    __u8 tos_value;
    __u8 tos_mask;
};

#endif /* _XT_DSCP_TARGET_H */
```

* Java文件找不到

```
external/doclava/src/com/google/doclava/ClassInfo.java:20: error: package com.sun.javadoc does not exist import com.sun.javadoc.ClassDoc
```

解决方案是在配置文件中加入

```
$ export PATH=/Library/Java/JavaVirtualMachines/jdk1.7.0_75.jdk/Contents/Home/bin:$PATH
```


