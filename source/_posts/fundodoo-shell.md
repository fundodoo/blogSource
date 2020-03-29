---
title: fundodoo 常用命令
categories: 
  - Doc
top: false
date: 2020-03-28 16:40:25
tags: 
- Doc
- Shell
---

## fundodoo 网站常用操作
> 进入`fundodoo.github.io`目录
### 常规命令
- 启动服务： `npx hexo server` 或`npx hexo s`
- 生成文件： `npx hexo g`
- 清空缓存： `npx hexo clean`
- 发布内容： `npx hexo deploy`
- 新建新文章： `npx hexo new post -p ./fileName  xxx` // `./fileName`:_post文件夹生成`fileName.md`文件； `xxx`：`fileName.md`文件头`Front-matter`里`title`的值

### 组合命令
- 组合清除-生成-发布： `npx hexo clean && npx hexo g && npx hexo deploy`
- 组合清除-生成-启动服务： `npx hexo clean && npx hexo g && npx hexo server`

## fundodoo 网站源码提交到github
> 进入`fundodoo.github.io`目录
- 提交命令： git push github master
