---
layout: post
title: "git workflow"
date: "2018-08-21 17:30:00 +0800"
tag: "git"
categories: "vcs"
---

### Git与SVN的比较

#### 原理上
 
- Git直接记录文件快照，SVN每次提交记录哪些文件更新更新了哪些行
- Git有本地仓库，SVN没有本地仓库
- Git大多数是本地操作，SVN大多数操作需要联网

#### 操作上

- Git先提交到本地仓库然后推送到远程仓库，SVN直接推送到远程仓库
- Git有各种"反悔"指令，SVN没有
- Git有真正的branch，而SVN只是工作空间的副本

<!--more-->

### Gitflow工作流

Gitflow为不同的分支分配一个很明确的角色，并定义分支之间如何和什么时候进行交互。

#### 历史分支

`develop`和`master`是两个常驻分支，`master`分支记录了正式发布的历史，而`develop`分支作为功能的集成分支。因此，master分支的每次提交都应分配一个版本号。

#### 功能分支

新的功能分支应该从`develop`分支迁出一个feature分支，新功能开发完成之后再合并回`develop`分支，常用命令：
1. 开发功能a
```
git checkout -b feature-a develop
```
2. 开发完成后合并回develop分支
```
git checkout develop
git merge --no-ff feature-a
git push
git branch -d feature-a
```

#### 发布分支

1. 当开发完成时，直接从`develop`分支`checkout`出`release`分支
```
 git checkout -b release-0.1 develop
```
2. `release`分支用于发布，发布完成后将`release`分支合并到`master`分支并且打标签，方便后续跟踪每次发布。
```
git checkout master
git merge --no-ff release-0.1
git push

git tag -a 0.1 -m "release 0.1 publish" master
git push --tags
```

#### 维护分支/热修复

1. 从`master`分支拉出一个`hotfix`分支维护分支用于bug修复、快速给发布版本打补丁
```
git checkout -b hotfix master
```
2. bug修复完成立即合回`master`分支和`develop`分支，完成维护后删除`hotfix`分支
```
git checkout master
git merge --no-ff hotfix
git push

git checkout develop
git merge --no-ff hotfix
git push 

git branch -d hotfix
```

3. `master`上打新tag
```
git tag -a 0.2 -m "release 0.2 publish" master
git push --tags
```

#### 优点

- 单个功能独立开发，并行开发互不干扰
- master和develop分支分别记录发布和功能开发的历史
- 由于有发布分支，其他暂不发布的功能的开发不受发布的影响，可以继续提交
- 维护分支能快速打补丁，不影响正在开发的功能

#### 缺点

- 复杂，分支繁多
- Git GUI不支持，纯命令行
- 对开发者要求高（理解工作流，熟悉Git命令）
- 所有功能分支基于不稳定的develop
- 需要维护两个长期分支master和develop

### GitHub Flow

- 所有在`master`上的东西都是可发布的（已发布或马上发布）
- 开发新功能时，从`master`拉一个名称清晰的新分支
- 在本地提交到这个分支的同时把它`push`到远程仓库
- 当你需要得到反馈或帮助，或者该分支准备`merge`时，打开一个`pull request`
- 该分支被`review`且同意合并后，合并到`master`
- `push`到`master`后，应该立即发布

#### 优点

- 操作简单
- 主干的代码有质量保证

#### 缺点

- 测试线和正式环境的发布没有区分

### Git-Develop

Git-Develop模式将`develop`分支作为固定的持续集成和发布分支

- 每一个功能都从master拉一个功能分支。
- 在这个功能分支上开发，功能完成到发布时，提交`code review`，通过后自动合并到develop。
- 待所有计划发布的变更分支代码都合并到`develop`后，`rebase master`到`develop`，完成发布。
- 应用发布成功后打一个tag。
- develop分支的发布版本合并回master。

#### 优点

- 操作相对简单
- 流程稍作改动，即可区分测试线和正式环境
- 每次开发都基于正式版本最新的代码（master)，和当前开发的其他分支不产生依赖关系
- master始终是已发布状态

### Pull Requests

`Pull requests`不是一种工作流，而是一个能让开发者更方便地进行协作的功能，可以在提议的修改合并到正式项目之前对修改进行讨论。  

- 开发者在本地仓库新建一个功能分支。
- 功能完成后，开发者push分支修改到远程仓库中。
- 开发者发起Pull requests。
- 团队成员收到通知，进行code review，讨论和修改。
- 项目维护者合并功能到仓库中并关闭Pull Requests。