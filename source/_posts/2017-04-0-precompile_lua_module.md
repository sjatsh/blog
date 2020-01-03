---
layout: post
title: "nginx lua预编译"
description: ""
tag: "nginx"
date: "2017-04-05 00:00:00 +0800"
categories: "blog"
---

#### 前景提要  

前段时间nginx新版1.11.11更新之后`nginx_http_connection_t`结构体中的`ngx_buf_t **busy`从二级指针改为了`ngx_chain_t *busy`链表。
如何修改见[之前的blog](https://sunsblog.cn/2017/03/24/buildNewNginxError/)。     

<!--more-->

#### 问题

由于是直接将源码修改为支持nginx-1.11.11及之后版本，存在的问题是：如果我想编译nginx-1.11.11之前的版本怎么办呢？这样代码就不兼容之前版本的nginx
了，之前只考虑到解决不兼容新版本的问题。晚上下班和大神聊到前端时间nginx-1.11.11新版本的第三方lua模块的不兼容问题，受大神的提示应该兼容不同版本的
nginx包括新版和之前的老版本。

#### 解决

在lua模块中通过预编译判断nginx的版本来决定使用哪段代码，废话不多说上代码：
```
if (hc->nbusy) {
        b = NULL;
        for (i = 0; i < hc->nbusy; i++) {

            /*get first buf*/
            /* b = hc->busy[i]; */
#if nginx_version >= 1011011
            b = hc->busy->buf;
#else
            b = hc->busy[i];
#endif

            dd("busy buf: %d: [%.*s]", (int) i, (int) (b->pos - b->start),
               b->start);

            if (first == NULL) {
                if (mr->request_line.data >= b->pos
                    || mr->request_line.data
                       + mr->request_line.len + line_break_len
                       <= b->start)
                {
                    continue;
                }

                dd("found first at %d", (int) i);
                first = b;
            }

            dd("adding size %d", (int) (b->pos - b->start));
            size += b->pos - b->start;
            /*buf next*/
#if nginx_version >= 1011011
            hc->busy = hc->busy->next;
#endif
        }
    }
```  
可以看到这里通过`#if nginx_version >= 1011011`预编译指令判断nginx版本来决定使用`b = hc->busy->buf;`来兼容高版本还是使用` b = hc->busy[i];`
兼容老版本，至于`nginx_version`是啥，见nginx源码中的`nginx.h`头文件：
```
#ifndef _NGINX_H_INCLUDED_
#define _NGINX_H_INCLUDED_


#define nginx_version      1011013
#define NGINX_VERSION      "1.11.13"
#define NGINX_VER          "nginx/" NGINX_VERSION

#ifdef NGX_BUILD
#define NGINX_VER_BUILD    NGINX_VER " (" NGX_BUILD ")"
#else
#define NGINX_VER_BUILD    NGINX_VER
#endif

#define NGINX_VAR          "NGINX"
#define NGX_OLDPID_EXT     ".oldbin"


#endif /* _NGINX_H_INCLUDED_ */
```  
为了便于判断nginx版本，nginx中定义了`nginx_version`的规则，1.11.13就是1011013,1.11.11就是1011011，由此解决第三方lua模块的兼容性问题。