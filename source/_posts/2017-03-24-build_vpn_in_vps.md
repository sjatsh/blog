---
layout: post
title: "在搬瓦工VPS上搭建VPN服务"
date: "2017-03-24 00:00:00 +0800"
categories: "技术"
tag: "VPS"
---

### 搭建Shadowsocks服务

#### 安装组件

```
$ yum install m2crypto python-setuptools
$ easy_install pip
$ pip install shadowsocks
```

<!--more-->

### 安装完成以后配置服务端参数

```
$ vi  /etc/shadowsocks.json
```  

写入如下配置 

```
{
    "server":"0.0.0.0",
    "server_port":443,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"mypassword",
    "timeout":300,
    "method":"rc4-md5",
    "fast_open": false,
    "workers": 1
}
```

将上面的mypassword替换成你自己的密码server_port也可以修改，默认是443端口  

#### 运行启动Shadowsocks服务  

运行下面的命令
```
$ ssserver -c /etc/shadowsocks.json
```  

如果想要后台运行，运行如下命令
```
 ssserver -c /etc/shadowsocks.json -d start
```

#### 添加开机启动项  

命令行输入如下命令
```
$ cd /etc/rc.d/rc.local
```

添加
```
ssserver -c /etc/shadowsocks.json -d start
```

#### 配置多客户端登陆
修改shadowsocks.json，修改配置文件内容为下面的配置

```
{
    "server":"23.105.215.43",   #你服务器的ip
    "local_address":"127.0.0.1",
    "local_port":1080,
    "port_password":{           #端口以及对应的密码
         "9000":"password",
         "9001":"password",
         "9002":"password"
    },
    "timeout":300,
    "method":"rc4-md5",         #选择的加密方式
    "fast_open": false
}
```

#### Shadowsocks客户端下载

在这里下载安装[请点击下载](https://shadowsocks.org/en/download/clients.html)，选择自己想要的版本，这里提供的iphone版本需要购买貌似要40RMB，还是挺贵的，我选择的是appstore上的其他的版本的。直接在appstore上搜索<font color="#0000CD">shadowrocket</font>，这个很好用而且只要6RMB。下载好客户端只要配置好ip、port、method（你的服务端配置的加密方式）、以及密码之后你就可以愉快的翻墙了！下面贴出了app上的连接，windows按照上面的地址下载好按照相应的配置好就行了。
![](https://olef5l6y5.qnssl.com/shadowrocket2.png?imageView2/0/q/75|watermark/2/text/U3VuJ3MgQmxvZw==/font/5a6L5L2T/fontsize/280/fill/I0Y2MEU1Mg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)
![](https://olef5l6y5.qnssl.com/shadowrocket1.png?imageView2/0/q/75|watermark/2/text/U3VuJ3MgQmxvZw==/font/5a6L5L2T/fontsize/280/fill/I0Y2MEU1Mg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)