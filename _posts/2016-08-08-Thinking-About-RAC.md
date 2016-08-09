---
layout:     post
title:      "对《The future of ReactiveCocoa》的一些思考"
subtitle:   "编程中的数据流"
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

```swift
- (int)fun:(int)num 
{
	return num + 1;
}

int f = [self fun:1];
```
我们很容易就认为 `->` 和Objective-C里面的 `-` 一样嘛，一种修饰符而已，`-> Int` 和 `- (int)`就是一个东西嘛！只不过 swift 在后面（和Golang的函数定义蛮像的，Golang没有这个`->`而已。），OC在前而已，新语言嘛，要和古老的 OC 有所区别的嘛。

### () -> ()  

回想一下C语言的`函数指针`定义，可以定义一个函数指针为：

```c
typedef int (*funPointer)(int); // funPointer = int (*)(int)
```
如果按此举一反三的话，那么 swift 版的 `fun` 就可以看成这样：

```swift
fun = (Int) -> Int
```
把参数什么的都去掉，变成如下：

```swift
fun = () -> ()
```
`() -> ()` 就是我们今天讨论的重点了。`()` 表示数据，`->`表示运算过程，`fun`函数就是从输入一个数据，到得到一个数据的过程, 即值的变化过程。<br>

([sunnyxx的推导过程：《() -> ()》](http://blog.sunnyxx.com/2014/10/14/fp-essential/))

### ()  
`() -> ()` 表示值的变化的过程，是`函数式编程范式`的一种高度抽象，表示了一个函数式编程思想。<br>
swift 本身就支持函数式编程，而在 swift 语言中，`()` 可以表示为一个空的 `元组`,而`元组`可以表示任何的值。<br>

>* 0-Tuple表示空值，也是Void的别名。
* 1-Tuple表示任意类型的实例，如（Int，String，Enum，Struct等等）,也就是 `Int` 可以看作是一个 `(Int)` ，String 是一个`(String)`，以此类推。
* N-Tuple表示多个值的组合，如 `(2, "A")`、`(2, "B", Struct)`等。

Tuple的详细知识请参考 [Tuples(Medium Link)](https://medium.com/swift-programming/facets-of-swift-part-2-tuples-4bfe58d21abf)

在编程中，`数据` 和 `运算` 这两核心就可以在 swift 中抽象成 `() -> ()`了，一切的数据都可以被抽象成 `()`， 一切的运算过程都被抽象成 `->`，那么之前认为`->`只不过是新语言标新立异于古老的 OC 而放在后面的想法就是不成立的了，`->` 表示数据流动的方向，从一个数据到另一个数据。

## 到RAC中去  

熟知RAC的同学都知道，RAC中有三种`Event`，分别为`next`，`error`，`completed`。而RAC中的`Signal`(以`RACSignal`为代表)会发送`事件流(a stream of events)`给`订阅者(Subscribers)`。不熟悉的请先了解[RAC基本用法](https://www.raywenderlich.com/62699/reactivecocoa-tutorial-pt1)。<br>

同样以[RAC基本用法](https://www.raywenderlich.com/62699/reactivecocoa-tutorial-pt1)这篇文章里的**数据流动示例图**作为参考，![RAC事件流](http://www.raywenderlich.com/wp-content/uploads/2014/01/FilterAndMapPipeline.png)<br>
**整个`Signal`的传递过程可以抽象成`(() -> ()) -> (() -> ()) -> ... -> (() -> ())`。**  

而RAC的作者Justin Spahr-Summers 在 GitHub Reactive Cocoa Developer Conference 上的[《The Future Of ReactiveCocoa》](https://github.com/jspahrsummers/the-future-of-reactivecocoa/blob/master/The%20Future%20of%20ReactiveCocoa.md)([YTB视频](https://www.youtube.com/watch?v=ICNjRS2X8WM))参考.NET的四种Interface

>* IEnumerable
* IEnumerator
* IObservable
* IObserver

将函数式编程中的Event分为了：

>* Observer（Push）: Event -> ()
* Obserable（Push）: (Event -> ()) -> ()
* Enumerator（Pull）: () -> Event
* Enumerable（Pull）: () -> (() -> Event)

这四种抽象的范式的表示方式和上文中通过 swift 抽象的 `() -> ()` 有异曲同工之妙。  

将上文中的进行一下操作: <br> 

> **1> `(() -> ()) -> (() -> ()) -> ... -> (() -> ())`   
 2> '1>' 表达式中 `()` 使用 `Event` 替换  
 3> `(Event1 -> ()) -> (Event2 -> ()) -> ... -> (EventN -> ())`，  
 4> '3>'表达式等同于 `Obserable` 的抽象。**

#### Observer: Event -> ()
`Observer` 等同于RAC中的 `subscriber`，当接收到一个`Event`时，就执行某些操作。类似于 swift 中的 `Sink` 协议。

#### Obserable: (Event -> ()) -> ()
在函数式编程中，函数作为一等公民，可以作为值传递，所以`Obserable`（RAC中的 `Signal`） 可以接受 `Observer`，当 `Observer` 在值传递过程中，也会执行某些操作。

#### Enumerator: () -> Event
到目前为止，这些都很好理解，然而硬币也有两面性，正如黑格尔的奥伏赫变一般，`Observer` 翻转 `->` 的方向，得到 `() -> Event`，
即`Enumerator`，当其发生交互时，返回一个 `Event`。很类似于 swift 中的 `Generator` 协议。

#### Enumerable: () -> (() -> Event)
同样翻转`Obserable`的`->`得到 `() -> (() -> Event)`，即`Enumerable`。一个`Enumerable`（类似于`collection`）可以创建一组`Enumerator`，十分接近于 swift 中的 `Sequence` 协议。

#### Push & Pull

* `Observer` 和 `Obserable`属于 Push 类型的，是生产者驱动的。
* `Enumerator` 和 `Enumerable` 属于 Pull 类型的，是消费者驱动的。

### Enumerator 升级  
`() -> Event` 因为调用者必须得到一个`Event`，所以只能通过同步的方式去检索值，就造成阻塞当前线程，这是不好的方式。

#### () -> Promise Event  
`() -> Promise Event` 实现值通过异步的方式获取，封装每个结果到`Promise`中，然后在未来的某个时间点给我们提供`Event`。<br>
同理，`() -> (() -> Event)` 变为 `() -> (() -> Promise Event)`。
（[Promises & Futures](https://en.wikipedia.org/wiki/Futures_and_promises)）<br>

>**Promise：**
>
>* Promise 确保任务能执行
* 提供了无序和异步的功能
* 由调用者完全控制

`Promise` 代表了一种 `延时计算` 的思想，与 swift 和 Golang 中的 `defer` 提供的功能一样。举个例子，消费者可以立刻获取5个 Promises，但是这样就做了无用功，因为只需要从第5个开始，跳过前4个，这样前4个promise就根本不会触发，消费者完全控制。

## 总结  

>**Observables 和 Enumerables 是：**
>
>* 一元运算(Monadic)
* 模块化(Modular)
* 异步的(Asynchronous)

**`Observables` 对任意一个 `observers` 都是相同的。** <br>
**`Enumerables` 对每一次枚举完全独立的。** <br>
**`Observables` 永远都是活跃的（live）。** <br>
**`Enumerables` 每一次的枚举都会开启新的工作（work）。**

**忘记Hot Signal 和 Cold Signal 吧！**

<del>**Hot Signal**</del> **Observables**  
<del>**Cold Signal**</del> **Enumerables**  

本文根据 RAC 的作者在 2014 年的 GitHub Reactive Cocoa Developer Conference 上的[《The Future Of ReactiveCocoa》](https://github.com/jspahrsummers/the-future-of-reactivecocoa/blob/master/The%20Future%20of%20ReactiveCocoa.md)([YTB视频](https://www.youtube.com/watch?v=ICNjRS2X8WM))Talk 及 [sunnyxx的博客文章：《() -> ()》](http://blog.sunnyxx.com/2014/10/14/fp-essential/) ，从 swift 的语法开始，对 RAC 的一些抽象进行了整理，试图去理解函数式编程的思想，如有出入，望不吝斧正（:/手动感谢）。

## 参考  

* [The future of RAC(Github)](https://github.com/jspahrsummers/the-future-of-reactivecocoa/blob/master/The%20Future%20of%20ReactiveCocoa.md)
* [The future of RAC(YouTube)](https://www.youtube.com/watch?v=ICNjRS2X8WM)
* [() -> ()](http://blog.sunnyxx.com/2014/10/14/fp-essential/)
* [Tuples](https://medium.com/swift-programming/facets-of-swift-part-2-tuples-4bfe58d21abf)

