---
layout: post
title: "常见web服务攻击和防御手段"
date: "2019-11-04 16:50:00 +0800"
tag: "web"
categories: "network-security"
---

# 常见web服务攻击和防御手段

## XSS跨站脚本攻击

XSS (Cross-Site Scripting)，跨站脚本攻击，因为缩写和 CSS重叠，所以只能叫 XSS。
XSS 的原理是恶意攻击者往 Web 页面里插入恶意可执行网页脚本代码，当用户浏览该页之时，嵌入其中 Web 里面的脚本代码会被执行，从而可以达到攻击者盗
用户信息或其他侵犯用户安全隐私的目的。

 - 反射型XSS：一般是通过给别人发送带有恶意脚本代码参数的 URL，当URL被打开时特有的恶意代码参数被HTML解析执行。  
 
   防御手段：
    - 前端渲染数据都要从后端来
    - 尽量不要从URL、document.referrer、document.forms等这些api获取数据并直接渲染
    - 尽量不要使用 eval, new Function()，document.write()，document.writeln()等这些可以直接执行字符串的方法
    - 如果必须要用的话也需要对传入的字符串进行转义
    - 前端渲染的时候也需要对字符串做转义编码

 - 持久性XSS：一般是通过留言、评论等这些可以和后端交互的功能将带有脚本的内容存到后端数据库的，当用户查询评论的时候就可能被查询出来渲染并执行
    防御手段：
    - CSP：本质上也就是设置白名单的方式：
       - 设置 HTTP Header 中的 Content-Security-Policy
            - 只允许加载本站资源: Content-Security-Policy: default-src 'self'
            - 只允许加载 HTTPS 协议图片: Content-Security-Policy: img-src https://*
            - 允许加载任何来源框架: Content-Security-Policy: child-src 'none'
       - 设置 meta 标签的方式
           
    - 转义字符：转义用户输入的引号、尖括号、斜杆 
    - HttpOnly Cookie：设置cookie时将属性设置成HttpOnly就可以避免cookie被恶意js获取
    
## CSRF跨站请求伪造

它利用用户已登录的身份，在用户毫不知情的情况下，以用户的名义完成非法操作。用户已登录A网站，此时访问B网站时B网站通过获取A网站的cookie发送请求
给A网站完成非法操作。

防御手段：
    - 设置cookie SameSite属性，cookie不随跨域请求发送
    - Referer Check，但是https跳转到http由于安全考虑浏览器不会携带referer，所以无法完全依赖referer检查来防止CSRF攻击
    - 每次请求都产生一个随机的Token，服务端验证Token
    - 关键操作必须输入验证码
    
## 点击劫持

原理：攻击者将需要攻击的网站通过 iframe 嵌套的方式嵌入自己的网页中，并将 iframe 设置为透明，在页面中透出一个按钮诱导用户点击。
如何防御：
     - 通过HTTP头`X-FRAME-OPTIONS`限制iframe嵌套，
     - js防御，js中判断页面处于iframe中时不显示网页内容, 通过`self==top`判断窗口是否被iframe嵌套
     
## URL跳转漏洞

服务端解析url参数中的网站并跳转，恶意用户可以伪造跳转链接用来攻击安全意识薄弱的用户。

如何防御：
    - refere限制，限制访问来源
    - 通过加入用户不可控的token来进行校验
    
## SQL注入

利用潜在的数据库漏洞访问和修改数据

注入的原理：比如前端需要输入用户名密码以登录系统，后端通过`SELECT * FROM user WHERE username='admin' AND psw='password'`查询用户信息，
这时输入一个username是`admin--`就可以直接以admin账号登录系统

如何防御：
    - 控制web应用访问的操作权限
    - 后端检查输入数据是否符合预期
    - 对于特殊字符进行转义或编码处理
    - 使用数据库提供的参数化查询（预编译)，不进行sql拼接

## OS命令注入攻击

和sql注入原理一样，只不过SQL注入是针对数据库的，而OS命令注入是针对操作系统的。

原理：构造命令提交给web应用程序，web应用程序提取构造的命令，拼接到被执行的命令中，因注入的命令打破了原有命令结构，导致web应用执行了额外的命令。
举例：
```go
exec.Command(fmt.Sprintf("git clone %s",repo))
```
golang中我们让前端输入一个git仓库地址来克隆git仓库代码，这时如果输入一个正常的仓库地址可以正常执行。但是如果我们传入一个这样的参数`repo adress && rm -rf /*`
而正好我们的程序又是以root启动，那么后面的命令就会被执行，根目录会被删除。

如何防御：
    - 后端对前端提前的数据进行规则校验（例如正则校验）
    - 执行命令前对参数进行转义和过滤
    - 不要直接拼接命令通过工具做拼接、转义预处理