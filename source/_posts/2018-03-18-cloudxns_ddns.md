---
layout: post
title: "CloudXNS DDNS"
date: "2018-03-18 00:00:00 +0800"
categories: "dns"
tag: "dns"
---

### DDNS

Dynamic Domain Name Server(动态域名服务)，DDNS是将用户的动态IP地址映射到一个固定的域名解析服务上，用户每次连接网络的时候客户端程序
就会通过信息传递把该主机的动态IP地址传送给位于服务商主机上的服务器程序，服务器程序负责提供DNS服务并实现动态域名解析。

<!--more-->

### 场景
电信光纤虽然会给一个公网ip，但是这个公网ip经常会变，这就给我们部署些自己的博客等小服务带来很多麻烦，我们不可能总是在公网ip变了后先去看路
由的公网ip然后再重新设置dns解析ip。这就给我们带来很多困扰，ddns就非常适合这种场景。

### CloudXNS DDNS
我这里用的一个github一个开源项目[cloudxns-ddns](https://github.com/zwh8800/cloudxns-ddns)，go语言封装的CloudXNS的api通过定时
请求CloudXNS的api接口来动态设置对应域名的解析ip地址。