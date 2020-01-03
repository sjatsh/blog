---
Layout: post
title: "nginx request copy"
tag: "nginx"
date: "2018-01-29 00:00:00 +0800"
categories: "技术"
---

### 问题

目前公司前端项目的图片文件全部是通过请求后台获取token接口获取七牛上传token，然后前端直接上传七牛。这个机制本身不存在问题，问题在于
公司使用了一个第三方授权服务器，这个服务器需要下载拿到用户上传的图片。问题来了，这个服务器当初部署在内网，如果需要它访问到上传图片，需要
将文件在用户上传后再备份到一个我们自己的服务器。  

<!--more-->

### 实现  

首先就问题本身，我们能够给出以下几种方案：  

- 上传时同时上传到我们自己的服务器   
- 上传七牛后七牛自己回调我们的文件备份接口   
- 上传后在七牛回调我们的接口时下载后再备份到服务器     

 首先看第一种，这种就是上传了两次文件，太浪费客户端流量和带宽，而且也不优雅。第二种想法如果可行的话当然是最好的方式，但是就目前来看七牛
 貌似没有类似的接口。那么就只有第三种方式。

### 具体实现  

通过七牛回调我们自己的接口再去备份，这里同样也有两种实现方式：  

- 在回调接口中通过mq异步下载和备份文件  
- nginx中通过请求的拷贝|镜像请求实现将请求同时转给另一个专门用于处理文件备份的接口 
 从程序的解耦和整体架构上看可能nginx流量的拷贝|镜像更为理想，同时实现稍显复杂，同时需要与运维的同事进行沟通交流并进行nginx的配置调试。
 直接通过原有接口mq异步备份文件也是一种实现方式，两者个人认为都是可行方案，如果条件允许的情况下建议第二种，毕竟程序员就是喜欢折腾-_-。
 通过mq异步进行文件备份的方式没什么好说的，就是发mq消息，下载上传。本文重点，如何通过nginx进行请求的copy|镜像转发。  

#### post_action方式  

```
location / {
    uwsgi_pass      unix:app.sock;
    post_action     @post_action; 
}

location @post_action {
    proxy_pass      http://dst_host:dst_port; 
}
```
这种方式请求首先传递给unix：app.sock，并在完成时，post_action指令将请求传递给指定位置的@post_action。但是这种方式@post_action中的
proxy_pass不能是uri，比如：http://ip:port/test;这种是不可以的。很可惜...

#### mirror_module  

nginx1.13.4之后开始支持ngx_http_mirror_module模块，这是一个请求镜像转发模块，相信听名字就知道是干嘛的了，请求镜像拷贝转发呗。  

```
# original配置
location / {
    mirror /mirror;
    mirror_request_body off;
    proxy_pass http://127.0.0.1:9502;
}

# mirror配置
location /mirror {
    internal;
    proxy_pass http://127.0.0.1:8081$request_uri;
    proxy_set_header X-Original-URI $request_uri;
}
```

参考[https://www.centos.bz/2017/08/nginx-request-copy-ngx_http_mirror_module/](https://www.centos.bz/2017/08/nginx-request-copy-ngx_http_mirror_module/)