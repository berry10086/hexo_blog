title: ipset的一个小问题
date: 2014-09-12 00:05:53
tags: [openwrt]
categories: 
- Tech
- Openwrt
---
最近教育网ipv6被中间人攻击，访问google会提示证书错误，挺烦人的。  
但是路由器已经通过ipset和dnsmasq走了代理，难道设置没有成功？
<!--more-->
`nslookup google.com`同时解析到ipv4和ipv6的地址，系统默认ipv6，也就是说，以前设置的ipset和iptables分流都没有生效，只不过是解决了dns污染的问题罢了。

- 为什么ipv6地址没有被ipset标记呢？  
	最初我以为是因为ipset不知道ipv6，后来找到了[这篇资料](http://unix.stackexchange.com/questions/134578/is-there-a-way-to-match-an-inet-and-inet6-ip-set-in-a-single-rule)  
	ipset 默认只针对ipv4地址，可以通过`family`参数指定ipv6
	`ipset -N gov6 iphash family inet6`
	好了，获得了ipv6的set  


- 怎样让解析到的IP地址自动加到set里呢？
	1. `ipset-dns letitgo gov6 5353 8.8.8.8`  
		ipset-dns是一个自动把解析的ip添加到set里的工具，这句命令的意识是创建一个本地的dns服务器，监听5353端口，上游dns服务器使用8.8.8.8，解析的ipv4地址添加到letitgo的set里，ipv6地址添加到gov6里。非常cool，但是不能指定上游服务器的端口，解析的地址会被污染，并不实用。
	2. 回到dnsmasq继续研究，看到一份[配置文件](http://thekelleys.org.uk/gitweb/?p=dnsmasq.git;a=blob_plain;f=dnsmasq.conf.example)示例里有这样一行  `ipset=/yahoo.com/google.com/vpn,search`  
	也就是说，其实dnsmasq本身是支持对应多个set的，问题迎刃而解，把gov6这个set也加在配置文件里，解析到的所有地址都会被打上标记。  
- 设置ipv6转发  
	很简单，把打上标记的ipv6包都转发到透明代理端口  
	`ip6tables -t nat -A PREROUTING -p tcp -m set --match-set gov6 dst -j REDIRECT --to-port 1111`


小小的问题，折腾了一整天，期间试过很多方法，还把pdnsd换成了chinadns，印证了***认真看官方文档和说明时多么重要***，对iptables还是不太明白，以后有时间研究下，先挖个坑。


	