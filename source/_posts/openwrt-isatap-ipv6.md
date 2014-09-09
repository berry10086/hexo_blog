title: 在openwrt上配置isatap方式的ipv6
date: 2014-09-08 11:58:41
tags: [openwrt]
categories:
- Tech
- Openwrt
---
学校宿舍的原生ipv6需要认证，并且经常抽风，电脑通过isatap的方式使用ipv6倒是非常稳定，但是只能有一个ipv6地址，因此在路由器下使用ipv6一直是令人头疼的一件事。直到看到赵一开大神的博客，才彻底解决这个问题，在他的方法基础上加入了对动态ip的支持。  
<!--more-->
##原理
路由器使用isatap获取到ipv6地址，然后再设置ipv6 nat使子网的设备也能访问ipv6
##配置
- 禁用openwrt内置的ipv6模块  
`luci > Network > Interfaces > WAN > Advanced Settings`关闭`Use builtin IPv6-management`
同样也要关闭LAN的ipv6
WAN 和 LAN的`General Setup`中的`IPv6 assignment length`设置成`disabled`
如果 `Interfaces`下有wan6接口，则删除之

- 安装软件包
要用到的软件包有`ip` `6rd` `6in4` `6to4` `kmod-ip6tables` `kmod-ipt-nat6` `radvd` `luci-app-radvd`可以通过`opkg`安装  

- 路由器通过isatap接入ipv6  
新建脚本`/etc/init.d/isatap`，脚本内容
```bash
#!/bin/sh /etc/rc.common

START=95
IP=$(/sbin/ifconfig -a|grep inet|grep -v 127.0.0.1 | grep -v 192.168.1.1 |grep -v inet6|awk '{print $2}'|tr -d "addr:")
V4_REMOTE="166.111.21.1"
V6_REMOTE="2402:f000:1:1501:200:5efe"
V6_LOCAL="fe80::200:5efe"

start() {
    ip tunnel add sit1 mode sit remote ${V4_REMOTE} local ${IP}
    ifconfig sit1 up
    ifconfig sit1 add ${V6_LOCAL}:${IP}/64
    ifconfig sit1 add ${V6_REMOTE}:${IP}/64
    ip route add ::/0 via ${V6_REMOTE}:${V4_REMOTE} metric 1
}

stop() {
    ip tunnel del sit1
}
```
修改权限，添加开机启动
```bash
# chmod +x /etc/init.d/isatap
# /etc/init.d/isatap enable
# /etc/init.d/isatap start
```
这时在路由器上就可以ping通ipv6地址了
- 配置NAT
首先指定LAN口的私有ipv6地址
`luci > Network > Interfaces > LAN > IPv6 address`填写`fc00:0101:0101::1/64`
接下来配置radvd分配子网的ipv6地址，可以在luci界面里设置radvd，也可以直接编辑配置文件
`/etc/config/radvd`使用下面的配置
```bash
config interface
	option interface 'lan'
	option AdvSendAdvert '1'
	list client ''
	option ignore '0'
	option IgnoreIfMissing '1'
	option AdvSourceLLAddress '1'
	option AdvDefaultPreference 'high'
	option MinRtrAdvInterval '5'
	option MaxRtrAdvInterval '10'

config prefix
	option interface 'lan'
	option AdvOnLink '1'
	option AdvAutonomous '1'
	option ignore '0'
	list prefix 'fc00:0101:0101::/64'
	option AdvRouterAddr '1'

config route
	option interface 'lan'
	list prefix ''
	option ignore '1'

config rdnss
	option interface 'lan'
	list addr ''
	option ignore '1'

config dnssl
	option interface 'lan'
	list suffix ''
	option ignore '1'
```
设置radvd开机启动
```
# /etc/init.d/radvd enable
```
最后设置NAT路由，在`/etc/firewall.user`里加入`ip6tables -t nat -I POSTROUTING -s fc00:101:101::/64 -j MASQUERADE`
重启防火墙和radvd
```bash
# /etc/init.d/firewall restart
# /etc/init.d/radvd restart
```

这时内网机器就可以获取到ipv6地址并能够ping通ipv6地址了，enjoy！

###参考链接
几乎照搬了赵一开大神的方法，特别感谢  
[https://blog.blahgeek.com/2014/02/22/openwrt-ipv6-nat/](https://blog.blahgeek.com/2014/02/22/openwrt-ipv6-nat/)
	