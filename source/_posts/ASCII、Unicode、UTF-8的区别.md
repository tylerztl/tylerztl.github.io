---
layout: Golang
title: ASCII、Unicode、UTF-8的区别
categories:
  - Golang
tags:
  - Golang
translate_title: the-difference-between-ascii-unicode-utf8
date: 2019-11-23 17:44:14
---
详细梳理了计算机世界字符编码的三种方案的具体细节。
<!--more-->

# ASCII 
> American Standard Code for Information Interchange
>
> 美国信息交换标准代码
  
ASCII第一次以规范标准的类型发表是在1967年，最后一次更新则是在1986年，到目前为止共定义了128个字符。

**ASCII 码使用指定的7 位或8 位二进制数组合来表示128 或256 种可能的字符**。

标准ASCII 码也叫基础ASCII码，使用7 位二进制数（剩下的1位二进制为0）来表示所有的大写和小写字母，数字0 到9、标点符号， 以及在美式英语中使用的特殊控制字符。其中最后一位用于奇偶校验。

问题：随着互联网的发展， 混合多种语言的数据变得很常见。ASCII是单字节编码，无法用来表示中文（中文编码至少需要2个字节） 。世界上有许多不同的语言，所以需要一种统一的编码。 由此诞生了Unicode。

# Unicode
> 统一码、万国码、单一码

Unicode把所有语言都统一到一套编码里，这样就不会再有乱码问题了。

**Unicode最常用的是用两个字节表示一个字符（如果要用到非常偏僻的字符，就需要4个字节）**。现代操作系统和大多数编程语言都直接支持Unicode。

## Unicode和ASCII的区别
**ASCII编码是1个字节，而Unicode编码通常是2个字节。**

如：字母A用ASCII编码是十进制的65，二进制的01000001；而在Unicode中，只需要在前面补0，即为：00000000 01000001。
 
新的问题：如果统一成Unicode编码，乱码问题从此消失了。但是，如果你写的文本基本上全部是英文的话，用Unicode编码比ASCII编码需要多一倍的存储空间，在存储和传输上就十分不划算。由此诞生了UTF-8。

# UTF-8
本着节约的精神，出现了把Unicode编码转化为“可变长编码”的UTF-8编码。

UTF-8编码把一个Unicode字符根据不同的数字大小编码成1-6个字节，常用的英文字母被编码成1个字节，汉字通常是3个字节，只有很生僻的字符才会被编码成4-6个字节。如果你要传输的文本包含大量英文字符，用UTF-8编码就能节省空间。

UTF-8编码有一个额外的好处，就是ASCII编码实际上可以被看成是UTF-8编码的一部分，所以，大量只支持ASCII编码的历史遗留软件可以在UTF-8编码下继续工作。

> UTF-16表示最少用两个字节能表示一个字符的编码实现。同样是对unicode编码进行转换，它的结果是英文占用两个字节，中文占用两个或者四个字节

# Go的编码

**Go语言的源文件采用UTF8编码， 并且Go语言处理UTF8编码的文本也很出色**。

## rune

- byte 等同于uint8 (type byte = uint8)，常用来处理ascii字符
- rune 等同于int32 (type rune = int32),常用来处理 __unicode或utf-8字符__ ，一个rune就代表一个unicode编码
- go对unicode的支持包含三个包:
    1. unicode
    2. unicode/utf8
    3. unicode/utf16
> unicode包包含基本的字符判断函数。utf8包主要负责rune和byte之间的转换。utf16包负责rune和uint16数组之间的转换。

go语言的所有代码都是UTF8的，所以如果我们在程序中的字符串都是utf8编码的，但是我们的**单个字符（单引号扩起来的）却是unicode的**。 

```go
package main

import (
    "fmt"
    "unicode/utf8"
)

func main() {
    var str = "hello 你好"

    //golang中string底层是通过byte数组实现的，直接求len 实际是在按字节长度计算  所以一个汉字占3个字节算了3个长度
    fmt.Println("len(str):", len(str))
    
    //以下两种都可以得到str的字符串长度
    
    //golang中的unicode/utf8包提供了用utf-8获取长度的方法
    fmt.Println("RuneCountInString:", utf8.RuneCountInString(str))

    //通过rune类型处理unicode字符
    fmt.Println("rune:", len([]rune(str)))
}
```
输出：
```
len(str): 12
RuneCountInString: 8
rune: 8
```

## rune、[]byte、string 的相互转换
```
str := "Hello 你好"

//string 转[]byte
b := []byte(str)

//[]byte转string
str = string(b)

//string 转 rune
r := []rune(str)

//rune 转 string
str = string(r)
```
