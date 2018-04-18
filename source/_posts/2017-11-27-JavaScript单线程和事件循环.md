---
title: JavaScript单线程和事件循环
date: 2017-11-27 17:49:45
tags: [JavaScript]
category: JavaScript
description: JavaScript, 单线程，事件循环，任务队列
---

JavaScript的一大特点就是单线程，在JS引擎里，同一时间只能做一件事，不存在并发。

<!-- more -->

## 进程和线程

进程有系统分配的内存空间，相互之间独立。一个进程中有一个或多个线程，协作完成任务。

Chrome一个Tab可以看成一个浏览器渲染进程，包含常用线程：

1. JS引擎：执行js脚本

1. GUI渲染：解析HTML、CSS，布局，绘制。当界面需要repaint或reflow，线程会执行。当JS引擎执行时，GUI线程会挂起，GUI更新会保存在一个队列里等空闲执行。

1. 事件触发：事件触发时将任务加到队列末尾

1. 定时触发器

1. 异步http请求

## Event Loop

- 执行时的任务分为同步和异步。

- 同步任务在主线程依次执行，异步任务进入Event Table并注册回调函数callback，如`setTimeout` `on`。

- 当指定的事情完成后，如`on`事件触发，`setTimeout`时间到了，Event Table会将这个函数移入Event Queue。

- 主线程内的任务执行完毕之后，会往Event Queue读取对应的函数，进入主线程执行。

- 上述过程不断重复，也就是Event Loop(事件循环)。

![](http://p02ojz1i2.bkt.clouddn.com/20171127-%E4%BB%BB%E5%8A%A1%E9%98%9F%E5%88%97.png)

## Macro Task/Micro Task

前面的事件循环模型中，只有一个事件队列，而实际上由于任务的优先级不同，可以看做有不同优先级的事件队列。

macro: 整体代码script, setTimeout, setInterval, on

micro: Promise, process.nextTick

执行过程

- 执行一个宏任务

- 执行过程中如果遇到微任务，就将它添加到微任务的任务队列中

- 宏任务执行完毕后，立即依次执行当前微任务队列中的所有微任务

- 检查GUI更新，有的话开始渲染更新，包括scroll，resize，fullscreen，requestAnimationFrame，GUI渲染

- 渲染完毕后，JS线程继续接管，从事件队列开始下一个宏任务

![](http://p02ojz1i2.bkt.clouddn.com/20171127-%E5%AE%8F%E4%BB%BB%E5%8A%A1%EF%BC%8C%E5%BE%AE%E4%BB%BB%E5%8A%A1.png)

## 练习

```js
console.log('1');

setTimeout(function() {
    console.log('2');
    process.nextTick(function() {
        console.log('3');
    })
    new Promise(function(resolve) {
        console.log('4');
        resolve();
    }).then(function() {
        console.log('5')
    })
})
process.nextTick(function() {
    console.log('6');
})
new Promise(function(resolve) {
    console.log('7');
    resolve();
}).then(function() {
    console.log('8')
})

setTimeout(function() {
    console.log('9');
    process.nextTick(function() {
        console.log('10');
    })
    new Promise(function(resolve) {
        console.log('11');
        resolve();
    }).then(function() {
        console.log('12')
    })
})
```

去掉nextTick浏览器： 1 7 8 2 4 5 9 11 12

理论输出：1 7 **6** 8 2 4 **3** 5 9 11 **10** 12

Node环境还可能有这种情况：1 7 **6** 8 2 4 9 11 **3** **10** 5 12

原因在于Node事件循环与浏览器环境不同。

## Node事件循环

简单来说，在从宏任务队列取任务的时候，浏览器是只取一个，执行完后再下一轮循环；而Node是全拿出来执行，执行完才下一个Tick。

```js
setTimeout(function(){
  console.log(1);
  process.nextTick(() => {
    console.log(2);
  })
})
setTimeout(() => {
  console.log(3)
})
```

结果可能是：1 2 3 或 1 3 2，区别就在于在取宏任务队列的时候，是否两个回调都已超时。

## 结语

浏览器中的事件循环理解到此已经足够，但Node还可以继续深究，以后有时间研究Node的时候再回来讨论。

## 参考

1. [W3C规范Event Loop](https://www.w3.org/TR/html5/webappapis.html#event-loops)

1. [这一次，彻底弄懂 JavaScript 执行机制](https://juejin.im/post/59e85eebf265da430d571f89)

1. [事件循环总体概览 —— Node 事件循环 Part 1](http://www.zcfy.cc/article/event-loop-and-the-big-picture-nodejs-event-loop-part-1-3566.html?t=new)
