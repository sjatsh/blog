---
title: "安装和配置Hexo"
description: ""
tags: "hexo"
date: "2016-07-24"
categories: "vps"
---

#### Hexo

Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

<!--more-->

#### 首先安装node.js和Git
官方下载地址[https://nodejs.org/en/download/](https://nodejs.org/en/download/)，下载和安装好nodes.js之后。  
继续安装Git，[https://git-scm.com/](https://git-scm.com/)(已经有node和Git可以跳过这一步)。  

#### 安装Hexo
```
$ npm install -g hexo-cli
```

#### 使用Hexo构建博客

新建一个文件夹如：hexo，进入hexo目录下运行  

```
$ hexo init
```
 
之后一个博客的基本框架就购建好了，接下来如果不想修改主题的话就可以开始愉快的写博客了。通过：  

```
hexo new [layout] <title>
```

其中（layout）指定文章的布局，默认为post，可以修改站点配置文件`_config.yml`中的default_layout参数来修改指定默认布局。 

接下来通过：  

```
$ heox g
```

将我们写好的markdown文档生成静态的html文档（生成的站点文件默认放在public文件夹下），之后修改站点配置文件`_config.yml`  
  
```
deploy:  
- type: git  
 repo:
``` 

一个正确的部署配置中至少要有`type`参数，repo指定git仓库位置。如果通过git进行部署，我们需要先安装`hexo-deployer-git`  

```
$ npm install hexo-deployer-git --save
```

#### 修改默认主题  

可以现在github上下载好自己想要的主题，本站用的是next主题[在此下载](https://github.com/iissnan/hexo-theme-next)，下载好主题之后我们就可以替换hexo的默认主题了，将下载好的主题放在themes文件夹下修改站点配置文件_config.yml中的theme参数为：next。这样我们就应用了next主题，之后next主题的配置[请参照这里](http://theme-next.iissnan.com/)进行自己想要的配置就可以了。