---
layout: post
title: "Hexo配置Git一键部署"
date: "2016-07-25 00:00:00 +0800"
categories: "vps"
tag: "vps"
---

#### 走过的坑 

本以为在自己服务器上装一个Git服务，然后Hexo直接配置一下就好了很简单的事，可是实践证明并不是那么简单，虽然走了一点弯路但是最后还是全部配置好了。

<!--more-->

##### 服务器安装Git服务之后 使用XShell用SSH连接服务器失败（暂时没有找出原因）

##### 使用SSH登陆的时候，连接服务器时服务器总是会提示输入密码，这是因为/home/git/.ssh目录没有改变目录所属用户，将.ssh目录所属用户改为git就行了。

##### 使用Git Bash通过SSH连接远程VPS的时候会说连接被决绝，因为默认远程登录使用22端口，在.ssh目录添加config配置文件修改连接服务器的端口。  

```
Host sunsblog.cn
User git
Hostname 23.105.215.43
Port 27765
```

#### 设置Git用户名、生成SSH密钥  

```
git config --global user.email "email@example.com"
git config --global user.name "username"
ssh-keygen -t rsa -C "email@example.com" #一路回车生成公钥和密钥，一会会要用到公钥id_rsa.pub
```

#### 开始安装Git（CentOS） 

```
yum update && apt-get upgrade -y #更新内核
yum install git-core
``` 

#### 新建git用户添加sudo权限  	

```
adduser git
chmod 740 /etc/sudoers
vim /etc/sudoers
```

在编辑器中找到如下内容  

```
 ##Allow root to run any commands anywhere
 root    ALL=(ALL)     ALL
```

添加下面一行  

```
git   ALL=(ALL)     ALL
```  


保存并退出后执行  

```
chmod 440 /etc/sudoers
``` 

#### 创建git仓库，并配置ssh登录  

```
su git
cd /home/git
mkdir .ssh && cd .ssh
touch authorized_keys
vi authorized_keys //在这个文件中粘贴进刚刚申请的key（在id_rsa.pub文件中）
chown git .ssh //注意如果.ssh目录不是git用户创建的一定要修改目录所属用户

修改文件权限
chmod 600 authorized_keys
chmod 700 ~/.ssh
```

#### 设置SSH，打开密钥登陆功能
编辑/etc/ssh/sshd_conf文件，进行如下设置：
```
RSAAuthentication yes
PubkeyAuthentication yes
```

另外，请留意 root 用户能否通过 SSH 登录：
```
PermitRootLogin yes
```

当你完成全部设置，并以密钥方式登录成功后，可禁用密码登录：
```
PasswordAuthentication no
```

#### 初始化Git仓库  

```
cd /home/git
mkdir hexo.git && cd hexo.git
git init --bare
```

#### 创建网站根目录并赋予git用户对网站目录的所有权  

```
cd /www
mkdir blog
chown git:git -R /www/blog
```

#### 配置Git hooks脚本配置项目部署后自动更新网站  

```
su git
cd /home/git/hexo.git/hooks
vim post-receive
```

输入如下内容  

```
 #!/bin/sh
          
 unset GIT_DIR 
        
 NowPath=`pwd`  
 DeployPath="/www/blog"  

 cd $DeployPath  
 git fetch --all  
 git reset --hard origin/master   

 cd $NowPath 
 echo "deploy done"   
 exit 0 
```

改变脚本执行权限  

```
chmod +x post-receive
``` 

#### 部署和更新博客  

```
deploy:
  type: git
  message: update
  repo:git@sunsblog.cn:hexo.git,master
```


