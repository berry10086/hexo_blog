title: tips about customize openwrt
tags: []
categories: []
date: 2014-09-10 18:22:00
---
回想最初接触openwrt的时候，喜欢在恩山论坛找各种固件来刷机，按照教程一点一点的配置，心里洋溢着满满的成就感。但网上的固件终究是别人做的，大部分都集成了许多多余的功能，不一定适合自己，也没有提供对应内核版本的软件源，用官方的软件源经常出现内核版本不匹配的问题。  
<!--more-->
既然没有合适的，就自己动手， 丰衣足食吧。自制固件的方法有两种，直接从源代码编译或者使用`image builder`，后面逐个介绍。
##从源码编译
1. 获取源码，先安装`git`或者`svn`，然后按照[官方教程](https://dev.openwrt.org/wiki/GetSource)下载源代码。如果没有特殊要求，直接下载最新的`trunk`版源码。
2. 安装依赖，按照[官方教程](http://wiki.openwrt.org/doc/howto/buildroot.exigence)安装即可。  
3. 更新`feeds`。openwrt通过`feeds`来管理`package`，修改目录下的`feeds.conf.default`配置文件，***取消`oldpackage`这一行的注释***，否则很多软件包找不到。也可以在这里添加额外的软件，例如加上一行  
`src-git ss https://github.com/madeye/shadowsocks-libev.git`  
就可以在`package`列表里加入`shadowsocks`了。修改后更新feeds获取`Makefile`
```bash
$ ./scripts/feeds update -a
$ ./scripts/feeds install -a
```
4. 配置编译选项。`make menuconfig`图形化的界面，选择目标平台，路由器型号，


