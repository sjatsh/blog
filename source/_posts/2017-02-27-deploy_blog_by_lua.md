---
layout: post
title: "lua脚本解析webhooks实现自动部署"
date: "2017-02-27 00:00:00 +0800"
tag: "vps"
categories: "技术"
---

### 码云配置webhooks

实现自动部署的方式有很多，可以把项目放在github用github的webhooks，但是github只能创建公开项目私有项目是要钱的。也可以使用一种相对麻烦的方式，可以在vps上建一个git仓库每次提交到vps上之后触发hook去更新blog，这种方法相对麻烦而且vps
上的项目容易丢失。所以最后选择了码云，在码云上初始化一个blog项目，之后配置一个webhooks并且跑配置密码，防止url被恶意调用。

<!--more-->  


如下图：![](https://olef5l6y5.qnssl.com/mayunwebhooks.png?imageView2/0/q/75|watermark/2/text/U3VuJ3MgQmxvZw==/font/5a6L5L2T/fontsize/280/fill/I0Y2MEU1Mg==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

### nginx配置
前提是安装了lua或者Luajit以及nginx编译了lua-nginx-module模块
```
  location /webhooks {
    content_by_lua_file blog_hook.lua;
  }
```
现在开始编写lua脚本
```
ngx.req.read_body()
local data = ngx.req.get_body_data()
local args = ngx.req.get_uri_args()
local cjson = require "cjson"

value = cjson.decode(data)

if value["password"]=="your_password" then

   os.execute("sh /www/blog_update.sh");
   ngx.say("OK")
   ngx.exit(200)

else

    ngx.exit(404)

end
```
这里应该注意，在获取body内容之前必须`ngx.req.read_body()`，通过`local data = ngx.req.get_body_data()`获取body内容，通过`local args = ngx.req.get_uri_args()`获取url后面携带的参数。

### "cjson"
使用[cjson](https://github.com/openresty/lua-cjson)模块解析json字符串

```
$ git clone https://github.com/openresty/lua-cjson.git
$ cd ./lua-cjson
```

在编译之前需要修改一下Makefile

```
21 LUA_INCLUDE_DIR ?=   $(PREFIX)/include
```

打开Makefile修改第21行，修改为

```
21 LUA_INCLUDE_DIR ?=   $(PREFIX)/include/luajit-2.0
```

因为luajit默认将lua头文件放在 /usr/local/include/luajit-2.0 中，修改之后执行

```
$ make all
$ make install
```

之后就可以通过cjson解析body中的json串了，密码验证通过才可以继续调用自动更新脚本否则返回404，更新的脚本这里也列出来。
### 编写shell脚本

```
#!/bin/sh
export PATH=$PATH:/usr/local/nodejs/bin
export LANG="en_US.UTF-8"
unset GIT_DIR 

NowPath=`pwd`
WebSite="/www"
BlogPath="/www/blog"

if [ ! -d "$BlogPath" ]; then
  cd $WebSite
  git clone git@git.oschina.net:sunblog/blog.git
  cd $BlogPath
else
  cd $BlogPath
  git fetch --all
  git reset --hard origin/master
fi

hexo clean
hexo g

#awk '{message="亲爱的"$1"，您所关注的博客Sun‘s Blog已更新，快去围观吧！https://www.sunsblog.cn";tittle="Sun’s Blog更新";system("echo \""message"\" | mail -s \""tittle"\" "$2);}' /www/mail.txt

echo 您的博客已成功更新，https://www.sunsblog.cn | mail -s 博客更新成功 admin@sunsblog.cn

cd $NowPath
echo "deploy done"
exit 0
```

脚本会去判断blog文件夹是否存在不存在就clone否则强制更新，每次更新成功之后都会向你自己或者其他人发送邮件，我这里使用awk读取mail
.txt中的qq好友的邮箱账号去发送邮件，由于同一时间发送大量邮件可能会存在vps被封的风险。所以最好还是不要大量发送，只是每次给自己或者几个重要的人发送就好了。感兴趣的话可以自己研究awk以及sendmail。