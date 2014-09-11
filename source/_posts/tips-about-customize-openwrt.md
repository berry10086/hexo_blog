title: 定制openwrt固件心得
tags: [openwrt]
categories: 
- Tech
- Openwrt
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
4. 配置编译选项。`make menuconfig`图形化的界面，选择目标平台，路由器型号以及软件包，选中软件包，`y`集成进，`n`取消集成,`m`编译成ipk包但不集成到固件里。按下`/`可以搜索软件包，快速定位。配置好可以导出，留作以后备用。默认使用`.config`配置文件。
5. 开始编译。`make`即可，很多地方写到可以加入`-j`参数加速编译，但是我的经验是，只要加上`-j`开启多线程，编译过程就会报错，所以***千万不要加`-j`参数***。这个过程中还会下载软件包，部分软件包可能被墙，最好通过代理下载。
6. 报错的处理。出错后再次执行`make V=s`看报错信息，根据报错信息处理，例如，有次编译到`pdnsd`时报错，大概是说，git无法从远程仓库获取内容，查看`package/feeds/oldpackages/pdnsd/Makefile`，修改一行
`PKG_SOURCE_URL:=git://gitorious.org/pdnsd/pdnsd.git`
改成
`PKG_SOURCE_URL:=https://gitorious.org/pdnsd/pdnsd.git`
默认的ssh登陆方式可能出问题了，改成https就可以了。
7. 自定义固件。`package`文件夹里的软件包的配置文件就可以更改固件的默认配置。
	- 默认开启无线的方法  
    	`package/mac80211/files/lib/wifi/mac80211.sh`修改一行
        `option disabled 0`
    - 常用的目录  
    	`package/base-files/files/etc`   
        `package/base-files/files/etc/confit/network`  
        
	但是每次openwrt源码更新都会覆盖配置文件，所以最佳实践是，***先刷好基础的固件，把配置文件拷出来，放到openwrt根目录下的files文件夹里，编译时会自动打包到固件里。***隔一段时间`git pull`获取最新的源码，直接`make`编译，可以一直用上最新的固件，减少折腾。

##使用`image builder`快速定制
编译一次固件至少需要1个小时，再次编译就快了，但是编译的过程实际上是挺折腾的。openwrt[官网](https://downloads.openwrt.org/snapshots/trunk/)提供了`image builder`可以无需编译制作自己想要的固件，但前提是你的路由器被openwrt官方支持。根据自己路由器的平台选择`image builder`，下载后解压，在其根目录执行执行下面的命令可以制作固件。  
`make image PROFILE=WNDR3800 PACKAGES="shadowsocks-libev" FILES=files`  
解释下参数，`PROFILE` 是自己路由器对应的配置名称，通过`make info`查看可用的`PROFILE`。`PACKAGES`是希望添加到固件里的软件包，注意包依赖关系。`FILES`是打包到固件里的配置文件的目录。可以把命令写到脚本里，定期下载最新的`image builder`，运行脚本生成固件。生成的固件在`bin`目录。
***想加入新的软件包，可以把包直接放到`package`文件夹里，在`PACKAGES`里加入包名***


##总结
使用`image builder`简单实用，是首选方法，但是如果自带的`package`里面没有需要的包，或者路由器不被openwrt官方支持，就只能通过源代码编译了。但是不论使用哪种方法，***最重要的是把自己的配置文件备份出来直接集成到固件里，刷机即用，养成经常备份的好习惯！***

##参考链接
[编译个性化的openwrt固件](http://www.zoublog.com/compile-customized-openwrt/)  
[官方文档](http://wiki.openwrt.org/zh-cn/doc/howto/obtain.firmware.generate)
    
