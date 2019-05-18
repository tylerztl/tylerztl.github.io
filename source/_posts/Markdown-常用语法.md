---
layout: markdown
title: Markdown 常用语法
categories:
  - 工具
tags:
  - Markdown
translate_title: markdown-common-syntax
date: 2019-05-17 01:21:54
---

Markdown是一种轻量级标记语言，它允许人们“使用易读易写的纯文本格式编写文档，然后转换成有效的XHTML（或者HTML）文档”。由于Markdown的轻量化、易读易写特性，并且对于图片，图表、数学式都有支持，当前许多网站都广泛使用 Markdown 来撰写帮助文档或是用于论坛上发表消息。

本文收集了常用的Markdown语法，以便于快速查找使用。
<!--more-->

## 标题
Markdown 共支持六级标题
```
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题
```

## 锚点
Markdown会自动给每一个h1~h6标题生成一个锚点，其id就是标题内容

### 页内跳转链接
跳转到某个标题的链接，有两种办法： 
1. 使用Markdown的语法来增加跳转链接
```
[跳转到标题](#标题)
```
   [跳转到标题](#标题)
2. 使用HTML语法来增加跳转链接
```
<a href="#标题">跳转到标题
```
   <a href="#标题">跳转到标题
   
### 自定义锚点
跳转到文档中不是标题的位置，则需要在该位置自定义一个锚点
```
<span id="自定义">这是自定义锚点</span>
[跳转](#自定义)
```
<span id="自定义">这是自定义锚点</span>
[跳转](#自定义)


## 引用
Markdown 标记区块引用，只需要在整个段落的第一行最前面加上 `>`
```
> https://github.com/BeDreamCoder
```
> https://github.com/BeDreamCoder

区块引用可以嵌套，只要根据层次加上不同数量的 `>`
```
> 一级引用
> > 二级引用
```
> 一级引用
> > 二级引用

引用的区块内也可以使用其他的 Markdown 语法
```
>  [跳转](#标题)
```
> [跳转](#标题)

## 列表
### 无序列表
使用星号、加号或是减号作为列表标记
```
- Red
* Green
+ Blue
```
- Red
* Green
+ Blue

### 有序列表
使用数字接着一个英文句点
```
1. Red
2. Green
3. Blue
```
1. Red
2. Green
3. Blue

如果要在列表项目内放进引用，那`>`就需要缩进`按Tab键`
```
*  Red
    > Green
```
*  Red
    > Green

### 代办列表
表示列表是否勾选状态（注意：- [ ] 前后都要有空格）
```
- [ ] 不勾选
- [x] 勾选
```
- [ ] 不勾选
- [x] 勾选

## 代码
### 单行代码块
可以使用`Tab`或`四个空格`来插入单行代码块，代码块前需要有至少一个空行

### 行内代码块
可以通过`` ` ` ``(此处就是行内代码块)，插入行内代码
    
### 多行代码块
使用三个`` ` ``来包含多行代码，并且可以指定一个可选的[语言标识符](#语言标识符)，这就为它启用语法着色，代码高亮显示

````
```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello, world")
}
```
````

```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello, world")
}
```

<span id="语言标识符">代码块支持的语言，如下表所示：</span>

| 语言        | 标识符 |
| ----       | -----     |
|Go          |go, golang |
|C           |cpp, c     |
|Java        |java       |
|JavaScript  |js, jscript, javascript|
|C#          |c#, c-sharp, csharp|
|Shell       |bash, shell|
|CSS         |css        |
|Scala       |scala      |
|PHP         |php        |
|Python      |py , python|
|Ruby        |ruby, rails, ror, rb|
|swift       |swift      |
|Objective C |objc, obj-c|
|SQL         |sql        |
|XML         |xml, xhtml, xslt, html|
|text        |text, plain|
|Erlang      |erl, erlang|
|Visual Basic|vb, vbnet  |
|Perl        |perl, pl, Perl|
|matlab      |matlab     |
|ColdFusion  |coldfusion, cf|
|Delphi      |delphi , pascal , pas|
|diff&patch	 |diff patch |
|Groovy      |groovy     |
|JavaFX      |jfx, javafx|
|SASS&SCSS   |sass, scss |
|R           |r, s, splus|
|AppleScript |applescript|
|ActionScript 3.0|actionscript3, as3|

## 强调
使用 `*` 和 `_`  来表示斜体和加粗

### 斜体
```
*hello world*
或者
_hello world_
```
*hello world*
### 加粗
```
**hello world**
或者
__hello world__
```
__hello world__

### 斜体加粗
```
***hello world***
或者
___hello world___
或者
*__hello world__*
或者
**_hello world_**
```
***hello world***

## 转义字符
反斜线（\）用于插入在 Markdown 语法中有特殊意义的字符。如果要在文本中显示成对的 `*` 或 `_`，可以在符号前加入 \
```
\*hello world\*
或者
\_hello world\_
```
\*hello world\*

\_hello world\_

有特殊意义的字符包括：
```
\
`
*
_
{}
[]
()
#
+
-
.
!
```
