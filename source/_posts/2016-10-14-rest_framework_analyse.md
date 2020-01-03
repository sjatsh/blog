---
layout: post
title: "公司Rest架构启动过程分析"
tag: "rest"
date: "2016-10-14 00:00:00 +0800"
categories: "技术"
---

#### 什么是REST


具象状态传输（英文：Representational State Transfer，简称REST），是Roy Thomas Fielding博士与2000年在他的博士论文 "Architectural Styles and the Design of Network-based Software Architectures" 中提出来的一种万维网软件架构风格。

REST相对于传统的SOAP到底有哪些不同，或者说两者分别更适合运用于什么样的场景。

<!--more-->


SOAP（简单对象访问协议）：是交换数据的一种协议规范，使用在计算机网络Web服务（web service）中，交换带结构信息。SOAP为了简化网页服务器（Web Server）从XML数据库中提取数据时，节省去格式化页面时间，以及不同应用程序之间按照HTTP通信协议，遵从XML格式执行资料互换，使其抽象于语言实现、平台和硬件。

REST：是一种架构风格，相对SOAP要相对简单，资源是由URI来指定，对资源的操作包括获取、创建、修改和删除资源，这些操作正好对应HTTP协议提供的GET、POST、PUT和DELETE方法，通过操作资源的表现形式来操作资源，资源的表现形式则是XML或者HTML，取决于读者是机器还是人，是消费web服务的客户软件还是web浏览器，当然也可以是任何其他的格式。
REST的几个重要约束：  
 
 * 客户-服务器（Client-Server）
通信只能由客户端单方面发起，表现为请求-响应的形式。
 * 无状态（Stateless）
通信的会话状态（Session State）应该全部由客户端负责维护。
 * 缓存（Cache）
响应内容可以在通信链的某处被缓存，以改善网络效率。
 * 统一接口（Uniform Interface）
通信链的组件之间通过统一的接口相互通信，以 提高交互的可见性。
 * 分层系统（Layered System）
通过限制组件的行为（即，每个组件只能“看到”与其交互的紧邻层），将架构分解为若干等级的层。
 * 按需代码（Code-On-Demand，可选）
支持通过下载并执行一些代码（例如Java Applet、Flash或JavaScript），对客户端的功能进行扩展。

这只是网上搜索出来的解释，更加深入的理解需要具体的项目开发中去慢慢理解。

#### 公司架构源码以及启动过程分析

##### 1.首先加载<context-param>参数
`spring.profiles.active`设置项目环境信息（dev、test、product），配置`contextConfigLocation`配置Spring配置文件路径。紧接着容器会初始化一个`ServletContext`上下文，利用 `ServletContext` 能够获得 WEB 运用的配置信息, 实现在多个 Servlet 之间共享数据等。

##### 2.ContextLoaderListener
根据`contextConfigLocation`指定的Spring配置文件去初始化spring的`WebApplicationContext`上下文，并加入ServletContext中（`ServletContext`是整个WEB应用的上下文，在容器（Tomcat、JBoos等）完全启动WEB应用之前被创建，生命周期伴随真个WEB应用），之后可以通过自定义的框架上下文监听器获取ServletContex去获取`WebApplicationContext`上下文。

#### 上述为什么是WebApplicationContext而不是ApplicationContext呢？
 SpringMVC本身是不具备Web功能的，`WebApplicationContext`通过继承`ApplicationContext`去扩展获得Web功能，Spring MVC是如何在web环境中创建IoC容器呢？web环境中的IoC容器的结构又是什么结构呢？web环境中，spring IoC容器是怎么启动呢？

 在web.xml配置文件中，有两个主要的配置：`ContextLoaderListener`和`DispatcherServlet`。同样的关于spring配置文件的相关配置也有两部分：`context-param`和`DispatcherServlet`中的`init-param`。那么，这两部分的配置有什么区别呢？它们都担任什么样的职责呢？
 
 在Spring MVC中，Spring Context是以父子的继承结构存在的。Web环境中存在一个ROOT Context，这个Context是整个应用的根上下文，是其他context的双亲Context。同时Spring MVC也对应的持有一个独立的Context，它是ROOT Context的子上下文。
 
 对于这样的Context结构在Spring MVC中是如何实现的呢？下面就先从ROOT Context入手，ROOT Context是在`ContextLoaderListener`中配置的，`ContextLoaderListener`读取`context-param`中的`contextConfigLocation`指定的配置文件，创建ROOT Context。
 
##### Spring MVC启动过程大致分为两个过程：
 
* `ContextLoaderListener`初始化，实例化IoC容器，并将此容器实例注册到ServletContext中；
					
* `DispatcherServlet`初始化；
					
 其中`ContextLoaderListener`是在Web容器中初始化Spring的根上下文和实例化IOC容器，`DispatcherServlet`是初始化Spring MVC对应的上下文。具体分析请参考原文：[深入分析Spring与Spring容器](http://mp.weixin.qq.com/s?__biz=MzA3NDcyMTQyNQ==&mid=2649254477&idx=1&sn=7e4fc7094dc967f11a572d161b03c3a2&scene=0#wechat_redirect)

#### 3. ArchListener
 我们可以通过实现`ServletContextListener`去实现我们自己的Servlet监听器，可以在我们的Servlet监听器中的`contextInitialized`方法中通过`WebApplicationContextUtils.getRequiredWebApplicationContext(event.getServletContext());`从应用上下文中去获取`WebApplicationCotext`上下文。基于这一方式，我们可以通过自己创建框架上下文监听类添整个框架所有的上下文实现对整个框架的所有上下文管理。
 
#### 4.RestServiceListener（Rest服务监听）
 从ServletContext中获取Spring的`WebApplicationContext`上下文：
 获取到Spring的上下文后获取上面Spring配置文件  中的`RestServiceConfiguration` Bean Rest基础服务对象：用于保存前置拦截器、自定义前置拦截器、后置拦截器、自定义后置拦截器、CGI服务名称（配置文件指定）、CMD前缀（配置文件指定）、调用业务超时邮件接收者（配置文件指定）、系统异常邮件接收者（配置文件指定），之后通过`RestServiceUtils.registerServices(configuration);`（扫描自定义注解进行依赖注入）`configuration.init();`（初始化导入拦截器）

#### 5.初始化一系列过滤器

#### 6.DispatcherServlet：初始化Spring MVC上下文

#### 7.PropertiesServlet：用于动态修改配置文件

#### 8.紧接着Spring接管`MsgSenderService`时`@PostConstruct`注解的`init`方法会自动执行初始化MQ发送端、初始化`BaseConsumer`