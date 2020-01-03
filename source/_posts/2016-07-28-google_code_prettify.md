---
layout: post
title: "替换Hexo代码段样式为google code prettify"
tag: "hexo"
date: "2016-07-28 00:00:00 +0800"
categories: "技术"
---

#### 下载google code prettify  


node下载：  
```
npm install google-code-prettify --save
```

<!--more-->

GitHub下载：
[https://github.com/google/code-prettify](https://github.com//googlecode-prettify)  

#### 替换hexo模板  
将下载好的`prettify.js`和`themes`文件夹放在主题下的`source/js/google-code-prettify`目录下。  
现在就开始修改主题的模板文件，修改主题目录下的`layout/_layout.swig`文件，在`</head>`前添加  

```
<link href="/js/google-code-prettify/themes/tomorrow-night.min.css" type="text/css" rel="stylesheet" />
```

其中`tomorrow-night.min.css`可以换成你想要的样式，[参考这里](https://jmblog.github.io/color-themes-for-google-code-prettify/)选择一个自己喜欢的样式。  
在`</body>`前添加：

```
<script type="text/javascript" src="/js/google-code-prettify/prettify.js"></script>
<script type="text/javascript">
   $("pre").addClass("prettyprint");
   $("pre").addClass("linenums");
   $("pre").addClass("lang-html");
   $("pre").addClass("prettyprinted");
   prettyPrint();	
</script>
```

现在你只要在你只需要用`<pre></pre>`将你需要高亮的代码包裹起来就可以了。
