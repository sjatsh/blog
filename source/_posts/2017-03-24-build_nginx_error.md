---
layout: post
title: "nginx-1.11.11模块编译报错解决"
date: "2017-03-24 00:00:00 +0800"
categories: "C"
tags: "C"
---

####  Nginx最新版1.11.11编译报错

最近nginx官网更新了nginx的最新版nginx-1.11.11，于是下载下来想要搭在新买的vps上。
但是发现在了一个问题，在编译添加openresty外部模块lua-nginx-module和echo-nginx-module时
报错，报错如下：

<!--more--> 

![](https://olef5l6y5.qnssl.com/lua_ngx_module_error.png?imageView2/0/q/75|watermark/2/text/U3VuJ3MgQmxvZw==/font/5a6L5L2T/fontsize/280/fill/I0Y2MEU1Mg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)
![](https://olef5l6y5.qnssl.com/echo_ngx_module_error.png?imageView2/0/q/75|watermark/2/text/U3VuJ3MgQmxvZw==/font/5a6L5L2T/fontsize/280/fill/I0Y2MEU1Mg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

#### 报错分析
由报错信息可以看出lua-nginx-module模块的ngx_http_lua_headers.c文件中的第151和第227行
报类型不匹配的错误。分析lua-nginx-module源码可以看出：
`ngx_http_connection_t`结构体中的`busy`参数,由`ngx_buf_t`的双重指针  
![](https://olef5l6y5.qnssl.com/ngx_http_connection_t_10.png?imageView2/0/q/75|watermark/2/text/U3VuJ3MgQmxvZw==/font/5a6L5L2T/fontsize/280/fill/I0Y2MEU1Mg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)  
变为`ngx_chain_t`的`ngx_buf_t`链表:    
![](https://olef5l6y5.qnssl.com/ngx_http_connection_t_11.png?imageView2/0/q/75|watermark/2/text/U3VuJ3MgQmxvZw==/font/5a6L5L2T/fontsize/280/fill/I0Y2MEU1Mg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)  
下面是报错变量和对应的报错位置：  
![](https://olef5l6y5.qnssl.com/ngx_buf_t.png?imageView2/0/q/75|watermark/2/text/U3VuJ3MgQmxvZw==/font/5a6L5L2T/fontsize/280/fill/I0Y2MEU1Mg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)
![](https://olef5l6y5.qnssl.com/ngx_http_lua_headers.png?imageView2/0/q/75|watermark/2/text/U3VuJ3MgQmxvZw==/font/5a6L5L2T/fontsize/280/fill/I0Y2MEU1Mg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)  
为何函数中`b = hc->busy[i]`改为` b = hc->busy->buf`和结尾的`hc->busy = hc->busy->next`
就可以了呢？看下图:  
![](https://olef5l6y5.qnssl.com/ngx_chain_s.png?imageView2/0/q/75|watermark/2/text/U3VuJ3MgQmxvZw==/font/5a6L5L2T/fontsize/280/fill/I0Y2MEU1Mg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)  
![](https://olef5l6y5.qnssl.com/ngx_chain_s_t.png?imageView2/0/q/75|watermark/2/text/U3VuJ3MgQmxvZw==/font/5a6L5L2T/fontsize/280/fill/I0Y2MEU1Mg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)   
与上面的`ngx_buf_t`的双重指针对比发现了什么呢？   
没错，这里其实就是要取`ngx_buf_t`的值，1.10以及之前的做法是`ngx_buf_t`双重指针的循环遍历。
而1.11开始使用`ngx_chain_s`链表来遍历获取`ngx_buf_t`的值。  
我们要做的只不过是将循环中的`ngx_buf_t`的双重指针的循环修改为链表的方式就好了。  
编译之后没有报错：
![](https://olef5l6y5.qnssl.com/buid_success_11.png?imageView2/0/q/75|watermark/2/text/U3VuJ3MgQmxvZw==/font/5a6L5L2T/fontsize/280/fill/I0Y2MEU1Mg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)
![](https://olef5l6y5.qnssl.com/success_page_11.png?imageView2/0/q/75|watermark/2/text/U3VuJ3MgQmxvZw==/font/5a6L5L2T/fontsize/280/fill/I0Y2MEU1Mg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

到此分析的也就差不多了呢，echo模块的报错原因也是如此，就是写blog的过程中官网又更新了最新版[nginx-1.11.12](http://nginx.org/download/nginx-1.11.12.tar.gz)版本,
1.11.12版本这部分的代码是一样的，其他地方的修改就不知道了，不知道会不会还有别的和第三方模块不兼容的地方。等待后续验证！