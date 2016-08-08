---
layout:     post
title:      "从「() -> ()」开始"
subtitle:   "编程中的数据流动"
date:       2016-08-08 20:23:00
author:     "Shin Cheung"
header-img: "img/post-bg-ios9-web.jpg"
header-mask: 0.3
catalog:    true
tags:
    - iOS
    - RAC
---

## 前言
### 我以为
第一次接触 swift 语言时，看到函数的表示形式如下：

```swift
func fun(num: Int) -> Int { 
	 return num + 1 
}

let f = fun(1)
```
和Objective-C对比一下：

```Objective-c
- (int)fun:(int)num 
{
	return num + 1;
}

int f = [self fun:1];
```
我们很容易就认为 `->` 和Objective-C里面的 `-` 一样嘛，一种修饰符而已，`-> Int` 和 `- (int)`就是一个东西嘛！只不过 swift 在后面（和golang的函数定义蛮像的，golang没有这个`->`而已。），OC在前而已，新语言嘛，要和这古老的 OC 有所区别的嘛。

### () -> ()

回想一下C语言的`函数指针`里面，可以定义一个函数指针为：

```C
typedef int (*funPointer)(int); // funPointer = int (*)(int)
```
如果按此举一反三的话，那么 swift 版的 `fun` 不就可以看成这样嘛：

```swift
fun = (Int) -> Int
```
把参数什么的都去掉，变成如下：

```swift
fun = () -> ()
```
`() -> ()` 就是我们今天需要思考的重点了。`()` 表示数据，`->`表示运算过程，`fun`函数就是从输入一个数据，然后得到另一个数据的过程嘛。

### () 与 ->
`() -> ()`代表了从一个数据运算得到另一个数据的过程，这不是和`函数式编程`的思想如出一辙吗？<br>
swift 就是支持函数式编程的啊！而在 swift 语言中，`()` 可以表示为一个空的 `元组`,而`元组`可以表示任何的值。<br>

* 0-Tuple表示空值，也是Void的别名。<br>
* 1-Tuple表示任意类型的实例，如（Int，String，Enum，Struct等等）,也就是 `Int` 可以看作是一个 `(Int)` ，String 是一个`(String)`，以此类推。<br>
* N-Tuple表示多个值的组合，如 `(2,"A")`等。

Tuple的详细知识请参考 [Tuples(Medium Link)](https://medium.com/swift-programming/facets-of-swift-part-2-tuples-4bfe58d21abf)

在编程中，`数据` 和 `运算` 这两核心就可以在 swift 中抽象成 `() -> ()`了，一切的数据都被抽象成 `()`， 一切的运算过程都被抽象成 `->`，那么之前认为`->`只不过是新语言标新立异于古老的 OC 而放在后面的想法就是不成立的了，`->` 表示数据流动的方向，从一个数据到另一个数据。

### 到RAC中去

