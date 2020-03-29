---
title: git 基础用法
categories:
  - Doc
top: false
date: 2020-03-28 15:38:58
tags: 
- git
- Doc
---

## 创建本地版本库

### 创建
1. 选择合适的目录执行下面的命令
```shell
mkdir fundodoo //创建fundodoo目录
cd fundodoo  //进入fundodoo目录
pwd      //查看路径
```

2. 通过`git init` 命令将这个目录变成git可以管理的仓库
```shell
git init
```
瞬间Git就把仓库建好了，而且告诉你是一个空的仓库（empty Git repository），细心的读者可以发现当前目录下多了一个`.git`的目录，这个目录是Git来跟踪管理版本库的，没事千万不要手动修改这个目录里面的文件，不然改乱了，就把Git仓库给破坏了。如果你没有看到`.git`目录,使用一下命令查看`ls -ah`

3. 添加文件
在`fundodoo`目录或者子目录下创建test.txt,然后执行下面的命令
 ```shell
git add test.txt //把文件添加到仓库
git commit -m 'add test file' //把文件提交到仓库
 ```
 `git commit` 命令的`-m` 后面输入的是本次提交的说明,语法层面可以是任意内容,业务层面最好是有意义的说明

## 版本管理

- 常用命令
```shell
git status  //查看哪些修改过的文件
git diff <file name> //查看文件哪些地方有修改
git log  //查看历史提交记录
git reflog //记录每一次命令
git reset --hard 版本(可以用HEAD,或者使用commit id)  //回退到某个版本
git checkout -- <file name>  // 丢弃工作区的修改
git remote add origin <远程库地址>  //将本地仓库和远程仓库关联
git push -u origin master  //将本地修改推送到远程库
git clone <远程库地址>   //将远程库克隆到本地
git checkout -b <分支名>  //创建并切换到分支
git branch //列出所有分支,分支名称前面带`*`号的表示当前正在使用的分支
git merge <分支名>  //合并分支到当前分支
git branch -d <分支名>   //删除指定分支
git switch -c <分支名>     //创建并切换分支
```
在`git`中,用`HEAD`表示当前版本,也就是最新的提交,上一个版本就是`HEAD^`,上上一个版本是`HEAD^^`,往上50个版本写50个`^`就比较繁琐了,可以写成`HEAD~50`.
`git chekout -- file` 命令中的`--`很重要,没有`--`就变成切换到另一个分支的命令
`git push -u origin master` 其中`-u`可以将本地的`master`分支和远程的`master`分支关联起来,在以后的推送和拉取就可以简化命令
`git checkout` 命令后面跟上`-b`表示创建并切换,相当于`git branch <分支名>`,`git checkout <分支名>`这两条命令


## Git Submodule
- 背景：某个工作中的项目需要包含并使用另一个项目。 也许是第三方库，或者你独立开发的，用于多个父项目的库。 现在问题来了：你想要把它们当做两个独立的项目，同时又想在一个项目中使用另一个。
- 解决方案： `Git` 通过子模块来解决这个问题。 子模块允许你将一个 `Git 仓库`作为另一个 `Git 仓库的子目录`。 它能让你将另一个仓库克隆到自己的项目中，同时还保持提交的独立。
- [官方文档](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%AD%90%E6%A8%A1%E5%9D%97)

### 添加子项目
```shell
git submodule add <子项目仓库地址> <文件路径>  // 如git submodule add git@github.com:fundodoo/hexo-theme-butterfly.git themes/Butterfly
```
#### 常见错误
1. git submodule: already exists in the index
  - 解决方法： 
    1. git rm -r --cached `xxx/xxx ` // `xxx/xxx`换成你的报错信息里的内容。
    2. 重新添加子模块

## 经验分享

### 案例:某个commit提交到指定的本地分支和远程分支

> `cherry-pick` 就是从不同的分支中捡出一个单独的commit，并把它和你当前的分支合并。如果你以并行方式在处理两个或以上分支，你可能会发现一个在全部分支中都有的bug,如果你在一个分支中解决了它，你可以使用cherry-pick命令把它commit到其它分支上去，而不会弄乱其他的文件或commit.

#### 案例分析
- 背景: 两个分支`test`和`master`,bug 已修改并提交到`test`,commit id  为 `adfas12...`,需要将bug修复同步到`master`

- 操作
    1. `git checkout master` 切换到`master`分支
    2. `git cherry-pick adfas12`，该commit便被提交到了`master`分支。

### 案例:更换git库的远程地址

#### 案例分析
- 背景: github远程仓库名变更，导致本地仓库失效。如何在原有仓库的基础上让本地仓库和新的远程仓库建立关联

- 操作
1. 方法一 直接修改远程地址#
    - git remote 查看所有远程仓库：git remote xxx 查看指定远程仓库地址（ps:此处的xxx为origin）
    - 修改旧仓库地址为：git remote set-url origin <新的远程地址>
    - 查看地址是否已修改：git remote -v
2. 方法二 先删除远程仓库再添加远程仓库#
    - git remote 查看所有远程仓库， git remote xxx 查看指定远程仓库地址
    - git remote rm origin
    - git remote add origin <新的远程地址>

[git官方文档](https://git-scm.com/book/zh/v2)