---
title: JavaScript事件循环
date: 2018-02-08 01:33:17
tags:
    - JavaScript
    - 事件循环
categories:
    - JavaScript
---


今天斗胆来谈谈JavaScript的事件循环，只讲最粗浅的理解

<!--more-->

关于JavaScript事件循环的机制，网上有很多博客写过这个话题，我看了很多别人写的内容。老实说越看越懵逼。有意思的是阮一峰老师曾经写过两遍这个话题，第一遍写完了之后，过了一阵子说是看到另一篇文章发现自己的理解是错误的，然后又写了一遍。尴尬的是朴灵大大毫不留情地指出了他第二遍文章的很多错误。除了阮一峰老师和朴灵大大的批注，我还看了好多人写的关于事件循环的总结。说实话有好多人的博客，两个人讲却是不同观点。有一些观点，也不那么容易验证谁对谁错，甚至我认为有时候他们两个人说的都是对的，只是讲的角度不同。总之到最后我也没有很明白JavaScript的事件机制具体到底是什么样。但是总归心里要有点底。所以今天我斗胆来讲讲JavaScript的事件循环，只讲到最粗浅的度。

## 执行栈

首先是执行栈，这一部分在[冴羽的博客](https://github.com/mqyqingfeng/Blog/issues/17)有非常精彩的论述。我就不重复了。简单讲就是每次调用一个函数就会把这个函数的执行上下文压入执行上下文栈里，函数运行完的话就从栈中pop出来。
## 异步操作

如果所以的操作都是同步操作，那JavaScript运行时中事件队列机制的实现就毫无意义，那什么是同步操作什么是异步操作?

我自己的理解是这样的: 所有的操作其实都是需要耗费时间的，但是耗费时间的多少有很大的区别。比如说像var a = 1 + 2你可以很快的得到结果，又或者说var a = fetch(“cai.com”)你或许需要一点时间才能得到结果。在前端操作中，一般IO读写、网络请求、DOM操作、和setTimeout、等操作都需要较长的时间才能得到结果。对于很快得到结果的操作，我们一般只需要同步调用他们就行了，而需要耗费时间较长的操作，我们一般用异步调用他们。(实际上并没有任何人规定耗时操作一定得异步调用，你也可以同步调用耗时操作，只是你后面的代码会被阻塞(block)而已)

## JavaScript是单线程的
类似于Java这样的语言是支持多线程的，所以一般遇到耗时操作，如果你用Java来处理的话，会选择开一条新线程，然后把耗时操作在这一条新线程中处理。等到处理完毕再到主线程中来更新UI啥的。。
可惜的是，JavaScript是单线程的，没有办法在执行JavaScript代码的时候再多开一个新线程处理耗时操作。但是JavaScript又不甘心每次遇到耗时操作的时候，后面的代码就被阻塞。所以在JavaScript Runtime里实现里事件循环机制。

## JavaScript runtime
我们先来分清两个概念，一个是JavaScript engine，一个是JavaScript runtime。我们所熟知的Chrome V8是属于JavaScript engine的，而像Nodejs，和 chrome浏览器 则属于JavaScript runtime的范畴。我们知道chrome浏览器以及Nodejs都是用的V8这个JavaScript engine。然而Nodejs和chrome浏览器却分别各自有一套
EventLoop机制。这一点就告诉了我们所谓JavaScript Eventloop机制的概念，应该是在JavaScript runtime中实现的。而JavaScript runtime是一个相对广泛的概念。
1. 首先讲讲Nodejs这个JavaScript runtime…额说实话我讲不出什么我对此并不了解，也不是我们今天的主角，但是据我所知是和Nodejs底层的libuv库是有一定关系的。

2. 而关于chrome浏览器这个JavaScript runtime。据我所知的是chrome浏览器是采用webkit内核的，并且用V8引擎替换了webkit的js引擎部分。浏览器的内核是多线程的，在webkit内核下面会有多个常驻线程组成。

- GUI 渲染线程
- JavaScript引擎线程(也就是我们的V8)
- 定时触发器线程
- 事件触发线程
- 异步http请求线程
- EventLoop轮询的处理线程
- …还有些不常驻线程

所以我们看到，虽然JavaScript引擎是单线程的，但是我们在浏览器的内核里面还有许多额外的线程，他们可以用来为我们实现异步调用所服务。

## JavaScript Eventloop(Chrome Browser)

在Chrome Browser这个JavaScript runtime里，它包含了一个待处理的消息队列(也有叫任务队列的)。

现在假设我们的JavaScript引擎线程(V8)开始执行一份js代码，因为执行代码的时候会不断调用函数，所以执行上下文栈就会有不断的push和pop操作等等。如果在遇到DOM操作，Ajax请求、以及setTimeout等WebAPI调用的耗时操作，JavaScript引擎线程会把这些操作交给浏览器内核(webkit)的其他模块去处理，webkit内核在Javasctipt执行引擎之外，有一个重要的模块是webcore模块。对于一些WebAPIs的耗时操作，webcore分别提供了DOM Binding、network、timer模块来处理底层实现。webcore处理完这些耗时操作(比如如果是setTimeout的话，timer模块的处理方法就是等待一定秒数(第二个参数)之后，将回调函数(第一个参数)放进消息队列里)的时候会把一个消息添加到消息队列里，每一个消息都与一个函数(就直接当是callback吧)相关联。

由于JavaScript引擎线程在执行js代码的时候非常迅速，因为所有的耗时操作都直接交给webcore了，交给webcore后，JavaScript引擎线程就接着执行后面的代码了，不去等待耗时操作返回结果。因为自己只要执行那些非耗时操作就行了。所以很快执行上下文栈就空了，因为所有函数都执行完了，所以他们的执行上下文都pop出来了。这也是为什么说JavaScript永不阻塞，以及Nodejs在服务器端的杀手锏(能高效处理IO操作)。

现在我们的消息队列里存在着许多消息(回调函数)，然而这些消息也是JavaScript代码，所以势必是要给JavaScript引擎线程去执行的。所以现在轮到EventLoop轮询的处理线程登场了。他就是事件循环的主角，他以类似下面的方式运行

```js
while (queue.waitForMessage()) {
queue.processNextMessage();
}
```
他会一直去查看消息队列里面有没有消息，有的话他就把消息队列的第一个消息交给JavaScript引擎线程去执行。如果没有的话，那他就会等消息到达。每一个消息完整的执行后，其它消息才会被执行。而webcore模块的一些模块会一直往这个消息队列添加消息，比如说像事件触发线程，我们点击了一个按钮，那这个按钮上如果有onclick函数啥的，那这个函数又会被添加到消息队列里，等待事件循环把它交给JavaScript引擎执行。所以假设我们在本机疯狂点击某个按钮，一下点击1000次，那就有1000个消息被放进消息队列,这时对第1000个消息的执行来说我们可能会明显感到延迟，因为得等前面999个都执行完了才会轮到它。这也是setTimeout里面的那个第二个参数只是处理请求所需的最小时间，但不是有所保证的时间，你传500ms进去，有可能得隔502ms才能得到处理。

*关于消息队列，有一种说法是不只有一个消息队列，分为macro-task和micro-task，每边又有好多个队列，我赞同这一说法，但是觉得在大多数情况下，可以当作只有一个消息队列来看，至少MDN是说只有一个，所以这应该可以当成一种抽象说法*

以下面一张被引用很多次的图来结束本篇

![](https://raw.githubusercontent.com/caistrong/Blog/master/_posts/js-eventloop/eventloop.png)