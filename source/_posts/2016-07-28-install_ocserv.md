---
title: "vps安装ocserv"
tag: "vpn"
date: "2016-07-28 00:00:00 +0800"
categories: "net"
---


#### 一、安装必要组件
```
$ yum install epel-release
$ yum install ocserv
```
<!--more--> 

#### 二、生成证书

##### 1、生成ca证书
###### 生成ca私钥
```
$ mkdir -p /etc/ocserv/pki/ca
$ cd /etc/ocserv/pki/ca
$ certtool --generate-privkey --outfile ca-key.pem 
```
###### 编写模板
```
cat << _EOF_ >ca.tmpl
cn = "VPN CA"
organization = "sunsblog"
serial = 1 
expiration_days = 999
ca
signing_key
cert_signing_key
crl_signing_key
_EOF_
```
###### 生成ca证书
```
$ certtool --generate-self-signed --load-privkey ca-key.pem --template ca.tmpl --outfile ca-cert.pem
```
##### 2、生成server端证书
###### 生成server端私钥
```
$ mkdir -p /etc/ocserv/pki/server
$ cd /etc/ocserv/pki/server
$ certtool --generate-privkey --outfile server-key.pem
```
###### 生成server证书模板
```
cat << _EOF_ >server.tmpl
cn = "sunsblog.cn"
organization = "sunsblog.cn"
expiration_days = 999
signing_key
encryption_key
tls_www_server
_EOF_
```
###### 生成server端证书
```
$ certtool --generate-certificate --load-privkey server-key.pem --load-ca-certificate ../ca/ca-cert.pem --load-ca-privkey ../ca/ca-key.pem --template server.tmpl --outfile server-cert.pem
```
##### 3、生成client端证书
###### 生成client端私钥
```
$ certtool --generate-privkey --outfile user-key.pem
```
###### 生成client端证书模板
```
cat << _EOF_ >user.tmpl
cn = "user"
unit = "admins"
expiration_days = 999
signing_key
tls_www_client
_EOF_
```
###### 生成client端证书
```
$ certtool --generate-certificate --load-privkey user-key.pem --load-ca-certificate ../ca/ca-cert.pem --load-ca-privkey ../ca/ca-key.pem --template user.tmpl --outfile user-cert.pem
```
##### 4、使用OpenSSL将客户端证书 user-cert.pem转化成.p12格式
可以按照提示设置证书密码
```
$ openssl pkcs12 -export -inkey user-key.pem -in user-cert.pem -name "client" -certfile ../ca/ca-cert.pem -caname "VPN CA" -out user-cert.p12
```
#### 三、修改配置文件
```
$ vim /etc/ocserv/ocserv.conf
```

```
#设置使用证书登录
auth = "certificate"
#设置端口
tcp-port = 9000
udp-port = 9000
#将以ocserv这个用户和用户组去运行工作进程
run-as-user = ocserv
run-as-group = ocserv
#socket文件
socket-file = ocserv.sock
#服务的默认目录
chroot-dir = /var/lib/ocserv
#时候启用工作进程的命名空间隔离，进程隔离限定特定的libc版本以及系统内核版本，如果启动失败请关闭选项
isolate-workers = true
#客户端最大连接数
max-clients = 16
#同一个账号最多可登陆的终端数
max-same-clients = 16
keepalive = 32400
# Dead Peer检测频率
dpd = 240
# 移动设备用的, 只要发送了 X-AnyConnect-Identifier-DeviceType 头, ocs 就会发送这个 dpd 
mobile-dpd = 1800
try-mtu-discovery = true
# server端证书和私钥
server-cert = /etc/ocserv/pki/server/server-cert.pem
server-key = /etc/ocserv/pki/server/server-key.pem
# ca证书
ca-cert = /etc/ocserv/pki/ca/ca-cert.pem
# 客户端id CN = 2.5.4.3, UID = 0.9.2342.19200300.100.1.1
cert-user-oid = 2.5.4.3
tls-priorities = "NORMAL:%SERVER_PRECEDENCE:%COMPAT:-VERS-SSL3.0"
# 认证超时时间
auth-timeout = 240
# 最短的认证重新连接时间
min-reauth-time = 300
max-ban-score = 50
ban-reset-time = 300
# cookie保存时间，手机关屏在时间内打开会自动重新连接
cookie-timeout = 86400
deny-roaming = false
rekey-time = 172800
rekey-method = ssl
use-occtl = true
# 进程id保存位置
pid-file = /var/run/ocserv.pid
# 设备名称，做策略可能会用到
device = vpns
predictable-ips = true
# 默认绑定域名
default-domain = sunsblog.cn
# 客户端分配的网段
ipv4-network = 192.168.101.0
ipv4-netmask = 255.255.255.0
# dns ip
dns = 172.20.10.60
dns = 8.8.8.8
ping-leases = false
# anyconnect 连接需要设置 true 
cisco-client-compat = true
dtls-legacy = true
user-profile = profile.xml
#route = ip网段/子网掩码
# 全局路由
#route = 0.0.0.0/128.0.0.0
#route = 128.0.0.0/128.0.0.0
route-add-cmd = "ip route add %{R} dev %{D}"
route-del-cmd = "ip route delete %{R} dev %{D}"
#no-route 去掉vps ip地址和国内的网段
custom-header = "X-DTLS-MTU: 1200"
custom-header = "X-CSTP-MTU: 1200"
```
添加开机启动项并且启动
```
$ systemctl enable ocserv
$ systemctl start ocserv
```
允许流量转发,修改/etc/sysctl.conf
```
net.ipv4.ip_forward = 1
```
使配置生效
```
sysctl -p /etc/sysctl.conf
```
在vpn虚拟网卡和vps外网网卡之间做流量转发
```
$ iptables -A FORWARD -i vpns0 -o venet0:0 -j ACCEPT
$ iptables -A FORWARD -i venet0:0 -o vpns0 -m state --state ESTABLISHED,RELATED -j ACCEPT
```
开启NAT并且让iptables来协商MTU值
```
$ iptables -t nat -A POSTROUTING -j MASQUERADE
$ iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```
保存iptables配置，否则重启之后配置会失效
```
$ sysctl iptables save 
```