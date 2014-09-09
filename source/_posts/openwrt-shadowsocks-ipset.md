title: openwrt使用ipset和shadowsocks实现自动代理
date: 2014-09-09 10:26:05
tags: [openwrt, ipv6, shadowsocks, 网络]
categories:
- Tech
- Openwrt
---
自从把路由器刷成openwrt后，解决了在学校免流量上网和自由访问Google的问题，觉得整个世界都美好了，把自己的折腾记录整理一下分享出来。   
<!--more-->
##思路  

- 学校ipv4收费，但ipv6免费，那么创建一个走ipv6的代理，让下载等耗流量的任务走这个代理，就可以节省流量，甚至所有流量走代理。这一步通过`shadowsocks`实现。  

- 有了代理实际上已经可以自由访问Google了，但是还需要手动设置代理，我希望路由器下面的设备能够自动实现流量分流。用`ipset`配合`dnsmasq`实现。
- 最后还要解决dns污染的问题，有下面几种方法：
	1. 使用非53端口的DNS服务器，使用方便，但是支持非53端口的DNS不多。 `opendns`
	2. 使用tcp协议查询。 `pdnsd`
	3. 使用隧道把本地53端口的UDP请求转发到远程去解析。 `ss-tunnel`
	4. 设置两个DNS，一个在国内，一个在国外，当国内解析的结果被污染，使用国外DNS服务器的解析结果。`chinadns` 

- 我使用了第2种方案，利用pdnsd得到一个通过tcp向上游dns服务器查询的本地dns服务器，然后利用dnsmasq指定有需要的域名通过pdnsd解析，可以保证获取到正确的ip。

##Shadowsocks
- shadowsocks官网已经上不去了，可以点[这里](http://sourceforge.net/projects/openwrt-dist/files/shadowsocks-libev/)下载编译好的ipk包。1.4.7版本加入了rc4-md5的加密方式，速度比aes-256快很多，加密方式最好用rc4-md5
- 开始之前，需要一台shadowsocks的服务器，修改配置`server_ip:"::"`监听ipv6和ipv4
- 解压后根据自己路由器的cpu型号选择合适的包通过scp上传到/tmp。以wndr3800为例，ar71xx平台，通过下面的命令安装（先安装libpolarssl解决依赖）
```bash
opkg update
opkg install libpolarssl
opkg install /tmp/shadowsocks-libev-polarssl_1.4.6_ar71xx.ipk
```
安装之后还需要做一个libpolarssl.so.7的链接，否则shadowsocks不能启动
```bash
ln -s /usr/lib/libpolarssl.so  /usr/lib/libpolarssl.so.7
```
- shadowsocks安装后有三个命令，`ss-local`启动sock5代理，`ss-redir`启动透明代理,`ss-tunnel`启动隧道。我使用了`ss-local`和`ss-redir`
- 分别建立`ss-local`和`ss-redir`的配置文件
```bash
#/etc/shadowsocks.json
{
    "server":"服务器ipv6地址", 
    "server_port":8888, #服务器端口 
    "local_port":1080, #本地sock5代理端口
    "password":"1111",
    "timeout":300,
    "method":"rc4-md5"
}
```
`ss-redir`的配置除`local_port`以外，其他都和上面的配置相同，例子中使用3333端口。
- 修改配置文件`/etc/init.d/shadowsocks`
```bash
#!/bin/sh /etc/rc.common
# Copyright (C) 2006-2011 OpenWrt.org

START=95

SERVICE_USE_PID=1
SERVICE_WRITE_PID=1
SERVICE_DAEMONIZE=1

start() {
    service_start /usr/bin/ss-local -c /etc/shadowsocks.json
	service_start /usr/bin/ss-redir -c /etc/ss-redir.json
}

stop() {
    service_stop /usr/bin/ss-local
	service_stop /usr/bin/ss-redir
}
```
添加执行权限，设置开机启动
```bash
chmod +x /etc/init.d/shadowsocks
/etc/init.d/shadowsocks enable
```
shadowsocks配置完成！

##pdnsd
安装
```bash
opkg update
opkg insatll ipset iptables-mod-nat-extra
```
配置`/etc/init.d/pdnsd.conf`
```bash
global {
    perm_cache=1024;
    cache_dir="/var/pdnsd";
    run_as="nobody";
    server_port = 1053;   
    server_ip = 127.0.0.1; 
    status_ctl = on;
    query_method=tcp_only;
    min_ttl=15m;
    max_ttl=1w;
    timeout=10;
}
server {
    label= "googledns"; 
    ip = 8.8.8.8;
    root_server = on;
    uptest = none; 
}
```
设置开机启动
```bash
/etc/init.d/pdnsd enable
/etc/init.d/pdnsd restart
```
完成!

##ipset和dnsmasq
openwrt默认安装的dnsmasq不支持ipset，需要先卸载，换成dnsmasq-full
```bash
opkg update
opkg install kmod-ipt-ipset ipset ipset-dns
opkg remove dnsmasq
opkg install dnsmasq-full
```
配置`/etc/dnsmasq.conf`
```bash
server=/google.com/127.0.0.1#1053
server=/googleusercontent.com/127.0.0.1#1053
server=/gstatic.com/127.0.0.1#1053
server=/googlehosted.com/127.0.0.1#1053
server=/golang.org/127.0.0.1#1053
server=/googleapis.com/127.0.0.1#1053
ipset=/google.com/letitgo
ipset=/googleusercontent.com/letitgo
ipset=/gstatic.com/letitgo
ipset=/googlehosted.com/letitgo
ipset=/golang.org/letitgo
ipset=/googleapis.com/letitgo
```
按照这种格式指定特定的域名走代理。
`server=/google.com/127.0.0.1#1053`的含义是google.com通过本地1053端口解析地址
`ipset=/google.com/letitgo`的含义给google.com的数据包打上标记，一会配置`iptables`时会用到
接下来配置`iptables`，在`/etc/firewall.user`里加上两行
```bash
ipset -N letitgo iphash
iptables -t nat -A PREROUTING -p tcp -m set --match-set letitgo dst -j REDIRECT --to-port 3333
```
作用是把打上了标记的数据包重定向到ss-redir的透明代理端口
重启`dnsmasq`和`firewall`就可以实现流量自动分流了
```bash
/etc/init.d/dnsmasq restart
/etc/init.d/firewall restart
```
以后只要修改`dnsmasq`的配置文件就可以指定更多的地址走代理
路由器上还开着 `ss-local`,下载时可以使用`Proxifier`做全局代理节省流量

##参考文章
1. [http://www.shuyz.com/install-shadowsocks-on-hg255d-openwrt-and-config-nat.html](http://www.shuyz.com/install-shadowsocks-on-hg255d-openwrt-and-config-nat.html)
2. [http://hong.im/2014/07/08/use-ipset-with-shadowsocks-on-openwrt/](http://hong.im/2014/07/08/use-ipset-with-shadowsocks-on-openwrt/)

