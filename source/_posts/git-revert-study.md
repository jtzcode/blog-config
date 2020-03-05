---
title: 日拱一卒系列（三）—— Git Revert 探究
date: 2020-03-04 20:23:47
categories:
    - 技术
tags:
    - Tech
    - Git
    - 日拱一卒
---
![head](abstract.jpg)
## 引言
今天跟大家分享一个Git Revert的实践。最近在项目中做一些跟持续集成（CI）相关的工作，其中有一块涉及代码的回退操作（Revert），我们需要自动化地实现这些功能，因此需要调用Git的命令来完成。

## 回退代码的方式

在使用Git作为代码管理工具时，常见的回退代码方式有两种，一是 `reset` 命令：
```shell
git reset commit_id
```
这条命令可以让本地的Git目录回退到指定的commit ID，并且**不会保留该commit之后的commit记录**。详细的文档可以参考[这里](https://git-scm.com/docs/git-reset)。这有时不是我们想要的，我们的需求可能是既可以回退代码到某一个版本，又保留这期间的所有commit记录，这时就需要用到 `revert` 命令：
<!--more-->
```shell
git revert commit_id
```
这条命令撤回指定commit的改动，注意是**撤回**某个改动，不是**撤回到**某个改动。另外，这条命令不会删除任何commit记录，而是会新增一条Revert操作的commit记录（弹出commit message的编辑窗口）。举个例子，我在一个空的Git目录里创建了一个 `test.txt` 文件，并添加了一行文本。然后添加一行文本，提交一次，一共添加两行。到这里，一共有三个commit产生：
```shell
commit ab2e366cf06c0bc0591e4abcdb671d6aa22c8ffa "Add 3rd line."
commit 2a980c12d3e8ee49ebb9a3fcffee4e2f2794ec8f "Add 2nd line."
commit 17a141dc77811320d75f258f7d3e53d91633fdfe "Init commit."
```
这时如果想去掉第三行文本的改动，可以执行如下操作：
```shell
git revert ab2e366cf06c0bc0591e4abcdb671d6aa22c8ffa
```
或者直接：
```shell
git revert HEAD
```
这时Git的Log类似这样：
```shell
commit 0d90a9a82353019b14721394567dae174a5d8f35 "Revert 'Add 3rd line.'"
commit ab2e366cf06c0bc0591e4abcdb671d6aa22c8ffa "Add 3rd line."
commit 2a980c12d3e8ee49ebb9a3fcffee4e2f2794ec8f "Add 2nd line."
commit 17a141dc77811320d75f258f7d3e53d91633fdfe "Init commit."
```
我们可以看到原来的commit并没有消失，而是多了一条Revert操作的commit。查看文件，其中最后一行已经不见了。这里我们需要注意的是执行Revert操作的顺序。我们有时需要回退掉多个commit的修改，`revert` 命令也支持这样的参数（参考[这里](https://git-scm.com/docs/git-revert)），但**不按顺序**的Revert可能会导致冲突，只有解决了冲突之后再执行：
```shell
git revert --continue
```
如果不想继续操作，可以放弃：
```shell
git revert --abort
```
如果是从最新的代码开始回退，我们不妨按顺序依次Revert每个commit，以避免冲突。

## 一些实践
有时候我们在做Revert操作需要一些检查，比如查看对应的commit记录是否存在：
```shell
git branch -r --contains COMMIT_ID
```
这条命令可以查看哪些分支（Branch）上有指定的commit记录，由此我们可以判断是不是需要在当前分支上进行Revert操作，这里的commit ID也可以换成tags。我们还可以比较与某个版本的区别：
```shell
git diff --shortstat COMMIT_ID

3 files changed, 481 insertions(+), 139 deletions(-)
```
`shortstat` 选项让结果只显示简短的统计数据。在实现自动化脚本时，这些命令可以用来实现需要的逻辑。

## 参考资料
* https://git-scm.com/docs/git-revert
* https://git-scm.com/docs/git-reset