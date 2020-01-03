---
layout: post
title: "通过账号密码查看Nginx日志"
tag: "nginx"
date: "2016-08-01 00:00:00 +0800"
categories: "net"
---

#### 修改虚拟主机配置文件

在server中添加如下代码

```
$ vim /usr/local/nginx/nginx.conf

server{

   location /logs {

      alias /usr/local/nginx/logs;
      autoindex on;

      autoindex_exact_size off;

      autoindex_localtime on;
      add_header Cache-Control no-store;

      auth_basic "Restricted";
      auth_basic_user_file /usr/local/nginx/conf/loguser;
   }
}
```

<!--more--> 

其中`auth_basic_user_file /usr/local/nginx/conf/loguser;`指定登陆账号的用户名和密码。

#### 通过htpasswd命令生成用户名及对应密码数据库文件  

```
$ htpasswd -c /usr/local/nginx/conf/loguser admin //admin为认证用户名
$ New password: ********  //输入认证密码 
Re-type new password: ********  //再次输入认证密码
Adding password for user admin

$ chmod 666 /usr/local/nginx/conf  //修改网站认证数据库权限 
$ cat /usr/local/nginx/conf/loguser  //可以看到通过htpasswd生成的密码为加密格式 
```

#### 配置点击日志文件打开文件而不是下载
```
打开文件 
$ vim /usr/local/nginx/mime.types
添加 
text/log         log;
```

#### 重启nginx

```
$ nginx -s reload
```

#### 访问站点查看日志

[https://sunsblog.cn/logs](https://sunsblog.cn/logs)

![](http://7xsbgk.com1.z0.glb.clouddn.com/logsLand.png?imageView2/0/q/75|watermark/2/text/U3VuJ3MgQmxvZw==/font/5a6L5L2T/fontsize/280/fill/I0Y2MEU1Mg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

输入用户名密码登陆

#### htpasswd命令汇总  

##### 语法

htppasswd(选项)（参数）

##### 选项

```
-c：创建一个加密文件； 
-n：不更新加密文件，只将加密后的用户名密码显示在屏幕上； 
-m：默认采用MD5算法对密码进行加密； 
-d：采用CRYPT算法对密码进行加密； 
-p：不对密码进行进行加密，即明文密码； 
-s：采用SHA算法对密码进行加密； 
-b：在命令行中一并输入用户名和密码而不是根据提示输入密码； 
-D：删除指定的用户。
```

##### 参数

用户： 要创建或者更新密码的用户名  
密码： 用户的新密码

##### 实例  

##### 利用htppasswd命令添加用户

```
$ htpasswd -bc .passwd www.linuxde.net php
```

在bin目录下生成一个.passwd文件，用户名www.linuxde.net，密码：php，默认采用MD5加密方式。

##### 在原有密码文件中增加下一个用户

```
$ htpasswd -b .passwd Jack 123456
```

去掉`-c`选项，即可在第一个用户之后添加第二个用户，依此类推。

##### 不更新密码文件，只显示加密后的用户名和密码

```
$ htpasswd -nb Jack 123456
```

不更新.passwd文件，只在屏幕上输出用户名和经过加密后的密码。

##### 利用htpasswd命令删除用户名和密码

```
$ htpasswd -D .passwd Jack
```

##### 利用htpasswd命令修改密码

```
$ htpasswd -D .passwd Jack 
$ htpasswd -b .passwd Jack 123456
```

即先使用htpasswd删除命令删除指定用户，再利用htpasswd添加用户命令创建用户即可实现修改密码的功能。
