---
title: Markdown 语法
tags: 
- Markdown
- Doc
categories: 
- Doc
top: false
date: 2020-03-27 15:48:35
keywords:
description:
top_img:
comments:
cover:
toc:
toc_number:
copyright:
mathjax:
katex:
hide:
---

# 语法指南

---

## Headers 标签
***n*个\#号就是h*n***(1 &lt;= ***n*** &lt;= 6);**\#** 号和**标题内容**之间一定要`有`一个`空格`

```Markdown
# h1
## h2
### h3
#### h4
##### h5
###### h6
```

#### `效果`
# h1
## h2
### h3
#### h4
##### h5
###### h6

---

## 字体加粗,斜体,删除线

```Markdown
**加粗** 或 __加粗__
*斜体* 或 _斜体_
~~删除线~~
***加粗且斜体*** 或 **_这样也可以_** 或 _**这样还是可以的**_
***~~加粗斜体且删除~~*** 或 ~~**_这样也可以_**~~ 或 _~~**这样还是可以的**~~_
```
#### `效果`

**加粗** 或 __加粗__
*斜体* 或 _斜体_
~~删除线~~
***加粗且斜体*** 或 **_这样也可以_** 或 _**这样还是可以的**_
***~~加粗且斜体且删除~~*** 或 ~~**_这样也可以_**~~ 或 _~~**这样还是可以的**~~_

---

## 列表

### 无序列表
```Markdown
- 第一项
- 第二项
```
#### `效果`
- 第一项
- 第二项
* 第三项
* 第四项

### 有序列表
```Markdown
1. 第一项
2. 第二项
3. 第三项
```
#### `效果`
1. 第一项
2. 第二项
3. 第三项

### 任务列表
```Markdown
- [ ] 未完成任务
- [x] 已完成任务
1. [x] 已完成任务
2. [ ] 未完成任务
```
#### `效果`
- [ ] 未完成任务
- [x] 已完成任务
1. [x] 已完成任务
2. [ ] 未完成任务

---

## 图片
```Markdown
悬停以查看标题文本:

方式一(内联):

![alt text](https://hexo.io/icon/favicon-196x196.png "我是标题1")

方式二(引用):
![alt text][logo]

[logo]: https://hexo.io/icon/favicon-196x196.png "我是标题2"

```
#### `效果`
悬停以查看标题文本:

方式一(内联):

![alt text](https://hexo.io/icon/favicon-196x196.png "我是标题1")

方式二(引用):
![alt text][logo]

[logo]: https://hexo.io/icon/favicon-196x196.png "我是标题2"

---

## 链接
```Markdown
[普通方式](https://www.fundodoo.com)

[普通方式带鼠标hover显示'Fundodoo'](https://www.fundodoo.com "Fundodoo")

[引用链接][任意不区分大小写的参考文字]

[相对路径](../blob/master/LICENSE)

[使用数字作为引用][1]

留空 [link text itself]

[任意不区分大小写的参考文字]: https://fundodoo.io
[1]: https://hexo.io/docs/
[link text itself]: https://hexo.io/api/

```
#### `效果`
[普通方式](https://www.fundodoo.com)

[普通方式带鼠标hover显示'Fundodoo'](https://www.fundodoo.com "Fundodoo")

[引用链接][任意不区分大小写的参考文字]

[相对路径](../blob/master/LICENSE)

[使用数字作为引用][1]

留空 [link text itself]

[任意不区分大小写的参考文字]: https://fundodoo.io
[1]: https://hexo.io/docs/
[link text itself]: https://hexo.io/api/

---

## 块引用
```Markdown
> 我们要对未来抱有希望
> 不要沉迷在过去
```
#### `效果`
> 我们要对未来抱有希望
> 不要沉迷在过去

---

## 内联代码
```Markdown
我认为你可以`这样<addr>`使用
```
#### `效果`
我认为你可以`这样<addr>`使用

---

## 使用Html标签
```Markdown
<p>请按 <kbd>ctrl</kbd>+<kbd>alt</kbd>+<kbd>del</kbd>重启电脑.</p>
```
#### `效果`
<p>请按 <kbd>ctrl</kbd>+<kbd>alt</kbd>+<kbd>del</kbd>重启电脑.</p>

```Markdown
<dl>
    <dt>定义清单</dt>
    <dd>有时候可以这样使用.</dd>

    <dt>但是在html里使用Markdown兼容性不好</dt>
    <dd>建议使用如下html标签 <em>tags</em>.</dd>
</dl>
```
#### `效果`
<dl>
    <dt>定义清单</dt>
    <dd>有时候可以这样使用.</dd>

    <dt>但是在html里使用Markdown兼容性不好</dt>
    <dd>建议使用如下html标签 <em>tags</em>.</dd>
</dl>

---

## 编程语言语法高亮
```Java
public class Test {
    System.out.println("123");
}
```

```Javascript
var test = 1;
alert(test);
```

```Python
s = "Python syntax highlighting"
print s
```

```Code
没有选择语言,就没有高亮效果
```

---

## 表格
```Markdown
|              |Center            |Right             |      Left         |
|--------------|:----------------:|-----------------:|:------------------|
| A            |        *C*       |        `R`       |        **L**      |
| B            |       C          | R                | ~~Left~~             |
```
#### `效果`
|              |Center            |Right             |      Left         |
|--------------|:----------------:|-----------------:|:------------------|
| A            |        *C*       |        `R`       |        **L**      |
| B            |       C          | R                |~~Left~~              |
