---
layout: post
title: "hsts preload"
tag: "https"
date: "2018-04-07 00:00:00 +0800"
categories: "net"
---

### 服务器https跳转的问题


一般我们的站点强制跳转https都是通过nginx、caddy进行http的强制https跳转。这个过程就可能存在问题，我们在浏览器直接输入域名example.com
的时候发起的是一个明文的http请求很容易被攻击者拦截并定向到钓鱼网站
<!--more-->
，并且如果使用的是国内的vps时很可能要求域名必须备案，没有备案直接进行
http访问的时候，会直接跳转到一个提示页面很是恶心~

![](https://olef5l6y5.qnssl.com/20180407202800.png)

### hsts

既然建立HTTPS连接之前的这一次HTTP明文请求和重定向有可能被攻击者劫持，那么解决这一问题的思路自然就变成了如何避免出现这样的HTTP请求。
我们期望的浏览器行为是，当用户让浏览器发起HTTP请求的时候，浏览器将其转换为HTTPS请求，直接略过上述的HTTP请求和重定向，从而使得中间人攻
击失效，规避风险。
HSTS最为核心的是一个HTTP响应头（HTTP Response Header）。正是它可以让浏览器得知，在接下来的一段时间内，当前域名只能通过HTTPS进行访问，
并且在浏览器发现当前连接不安全的情况下，强制拒绝用户的后续访问要求。
HSTS Header的语法:
`Strict-Transport-Security: <max-age=>[; includeSubDomains][; preload]`

- `max-age`是必选参数，是一个以秒为单位的数值，它代表着HSTS Header的过期时间，通常设置为1年，即31536000秒。
- `includeSubDomains`是可选参数，如果包含它，则意味着当前域名及其子域名均开启HSTS保护。
- `preload`是可选参数，只有当你申请将自己的域名加入到浏览器内置列表的时候才需要使用到它。关于浏览器内置列表，下文有详细介绍。

### hsts preload list

HSTS存在一个比较薄弱的环节，那就是浏览器没有当前网站的HSTS信息的时候，或者第一次访问网站的时候，依然需要一次明文的HTTP请求和重定向才能
切换到HTTPS，以及刷新HSTS信息。而就是这么一瞬间却给攻击者留下了可乘之机，使得他们可以把这一次的HTTP请求劫持下来，继续中间人攻击。

#### 加入hsts preload list

[https://hstspreload.org/](https://hstspreload.org/)设置好hsts响应头，并且证书有效可以在这里提交加入hsts preload list，这样就
可以让浏览直接强制访问https而不需要在这之前发起一次http请求。同样也避免了国内vps要求域名备案的问题（当然能备案还是备案一下，就目前知道.me
已经不能备案了）。
