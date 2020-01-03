---
title: "ubuntu nat"
description: ""
tags: "ubuntu"
date: "2018-02-01"
categories: "net"
---

### 场景  

公司iBox（ubuntu 16.04系统）需要与机床连接实现实现业务，iBox本身只有两个网口，需要一个口连接厂房内网路由，一个口与机床直连，机床本身需要能够连接外网。  
这时候就需要iBox实现nat转发并实现路由功能。

<!--more--> 

### 配置  

router打开路由转发：  
```
# cat /proc/sys/net/ipv4/ip_forward
```  
查看是否打开路由转发，0则没有打开，可以通过如下方式打开： 
```
# echo 1 > /proc/sys/net/ipv4/ip_forward
```  
通过修改`/etc/sysctl.conf`文件，`net.ipv4.ip_forward = 1`，然后执行`sysctl -p`可以永久打开


### SNAT

这里eth0作为内网口，eth1作为外网口
两种实现方式：  
第一种：  
```
# iptables -t nat -A POSTROUTING -s 192.168.6.0/24 -o eth1 -j MASQUERADE  
```
第二种：  
```
iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -m state --state ESTABLISHED,RELATED -j ACCEPT  
```
建议使用第二种方式，第一种存在的弊端是指定了内网网段，当需要修改内网ip时会导致内网机器无法访问外网了，第二种方式则是在两个网卡直接做流量转发，免去修改ip的问题。

#### 保存配置  

```
# iptables-save > /etc/iptables/rules.v4
```
#### 安装iptables-persistent 

```
apt-get install iptables-persistent
```
系统启动时iptables-persistent会以守护进程启动加载`/etc/iptables/rules.v4`的配置

#### 手动恢复配置  

```
# iptables-restore < /etc/iptables/rules.v4	
```

#### 查看nat转发列表

```
# iptables -t nat --list
```

### dhcp安装于配置

#### 安装

```
# sudo apt-get install isc-dhcp-server
```

#### 配置

```
vim /etc/default/isc-dhcp-server
# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
			
#    Separate multiple interfaces with spaces, e.g. "eth0 eth1".
			
INTERFACES="eth0"
```
INTERFACES修改为对应的内网网口  

修改`/etc/dhcp/dhcpd.conf`配置文件
```
# vim /etc/dhcp/dhcpd.conf
```
修改为
```
subnet 192.168.6.0 netmask 255.255.255.0 {
			
        range 192.168.6.100 192.168.6.200;
			
        option routers 192.168.6.1;
			
        option broadcast-address 192.168.6.255;
			
        option domain-name-servers 114.114.114.114;
			
        default-lease-time 600;
			
        max-lease-time 7200;
			
}
```
其中的ip、掩码等修改为对应内网网口的ip与掩码，这里以192.168.6.1为例。配置完之后重启dhcp服务`service isc-dhcp-server restart`




  


