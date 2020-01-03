---
layout: post
title: "Go源码阅读-私有库拉取问题"
date: "2019-10-29 14:24:30 +0800"
tag: "golang"
categories: "golang"
---

## go get拉取私有库报错 ##

### http拉取问题(>=1.6) ###

go get github.com/sjatsh/transfo
```bash
# cd .; git clone -- https://github.com/sjatsh/transfo /Users/jane/go/src/github.com/sjatsh/transfo
Cloning into '/Users/jane/go/src/github.com/sjatsh/transfo'...
fatal: could not read Username for 'https://github.com': terminal prompts disabled
package github.com/sjatsh/transfo: exit status 128
```

我们刚开始学习golang的时候我们需要go get拉取某个私有库的时候经常就会遇到类似的问题，这个问题的关键在于 `terminal prompts disabled` 这个错误信息，问题是由于http拉取私有仓库时需要用户名密码，但是git却没有提示输入用户名密码，直接看源码：

参见源码97行：
[go/get.go at release-branch.go1.6 · golang/go · GitHub](https://github.com/golang/go/blob/release-branch.go1.6/src/cmd/go/get.go)
```go
func runGet(cmd *base.Command, args []string) {

 // 这段代码默认设置 GIT_TERMINAL_PROMPT = 0
  if os.Getenv("GIT_TERMINAL_PROMPT") == "" {
		os.Setenv("GIT_TERMINAL_PROMPT", "0")
	}
}
``` 

`os.Setenv("GIT_TERMINAL_PROMPT", "0")` 这行标识关闭git命令行提示 ，也就导致了上面提到的`terminal prompts disabled`错误。


解决办法：`export GIT_TERMINAL_PROMPT=1` 手动设置git开启命令行提示，之后再go get就会提示输入用户名密码，一切正常：
![](https://upload-images.jianshu.io/upload_images/15292980-4359daa479eb71a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个issue有对这个问题的详细讨论：[http://golang.org/issue/9341golang.org/issue/9341](http://golang.org/issue/9341golang.org/issue/9341)


### 替换成ssh方式后存在的问题 ###
替换go get时git将http拉取换成ssh拉取代码
```bash
git config --global url."git@github.com:".insteadOf "https://github.com/"
```

## go get通过ssh方式拉取代码卡死问题(<1.8版本) ##

开启`ssh ControlMaster`配置后：
```bash
Host *
ControlPath ~/.ssh/master-%r@%h:%p
ControlMaster auto
ServerAliveInterval 30
```

由于开启 `ControlMaster` 后ssh会使用守护进程维护一个master链接，之后所有请求都会重用这一个TCP链接，他的stdout和stderr也不会退出。然而go get时使用子进程运行`git clone`并且监听子进程stdout和stderr退出，但是开启`ControlMaster`后ssh的守护进程不会退出，go get就会一直卡死。

1.8之后版本默认设置`GIT_SSH_COMMAND=ssh -o ControlMaster=no`禁用ssh `ControlMaster` 解决。

参见1.8版本源码151行：[go/get.go at release-branch.go1.8 · golang/go · GitHub](https://github.com/golang/go/blob/release-branch.go1.8/src/cmd/go/get.go)
![](https://upload-images.jianshu.io/upload_images/15292980-2d54d2a49a7b6582.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
