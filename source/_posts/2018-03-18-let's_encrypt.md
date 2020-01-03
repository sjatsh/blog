---
layout: post
title: "let's encrypt"
tag: "certificate"
date: "2018-03-18 00:00:00 +0800"
categories: "vps"
---

### 获取let's encrypt免费泛域名证书

首先DNS需要支持CAA，我用的CloudXNS，添加CAA记录记录值`0 issue letsencrypt.org`,vps`curl https://get.acme.sh|sh`,
添加环境变量  

<!--more-->

```
export CX_Key=""
export CX_Secret=""
.acme.sh/acme.sh --issue --dns dns_cx -d *.sjis.me -d sjis.me  --keylength ec-384
```
其中`CX_key`和`CX_Secret`分别是CloudXNS账户中的`API KEY`和`SECRET KEY`
跑完之后会发现CloudXNS多了两条TXT记录。之后等待一两分钟等记录生效。最后直接验证签发证书

``` 
~/.acme.sh/acme.sh --installcert --ecc -d *.sjis.me --key-file /etc/ssl/caddy/*.sjis.me.key --fullchain-file /etc/ssl/caddy/fullchain.cer
```