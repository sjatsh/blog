---
title: "Hexo新建Markdown自动打开编辑器"
tag: "hexo"
date: "2016-07-28 00:00:00 +0800"
categories: "技术"
---

#### 添加js脚本
  
在站点目录下新建一个js，内容如下：  

```
var exec = require('child_process').exec;

hexo.on('new', function(data){
  exec('start  "D:\dev\editor\MarkdownPad2\MarkdownPad2.exe" ' + data.path);
});
```

<!--more-->  

应该看的很明显，start后的第一个字符串就是你的编辑器的可执行程序的路径，data.path就是你新生成的Markdown文件的路径。这就话就是用你指定的编辑器打开生成的Markdown文件。  

```
$ hexo new "newMarkdown"
```

新建markdown文件的时候就会自动打开你的编辑器啦！