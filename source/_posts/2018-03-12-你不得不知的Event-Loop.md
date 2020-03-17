---
title: 你不得不知的Event Loop
date: 2018-03-12 11:03:00
tags: 前端 javascript 基础
---

# 前言

众所周知，JavaScript是一门**单线程**语言，虽然在html5中提出了Web-Worker，但这并未改变JavaScript是单线程这一核心。可看[HTML规范](https://www.w3.org/TR/html5/webappapis.html#event-loops)中的这段话：

> To coordinate events, user interaction, scripts, rendering, networking, and so forth, user agents must use event loops as described in this section. There are two kinds of event loops: those for browsing contexts, and those for workers.

为了协调事件、用户交互、脚本、UI 渲染和网络处理等行为，用户引擎必须使用event loops。Event Loop包含两类：一类是基于Browsing Context，一种是基于Worker，二者是独立运行的。
下面本文用一个例子，着重讲解下基于Browsing Context的事件循环机制。

来看下面这段JavaScript代码：

```javascript
console.log('script start');

setTimeout(function() {
  console.log('setTimeout');
}, 0);

Promise.resolve().then(function() {
  console.log('promise1');
}).then(function() {
  console.log('promise2');
});

console.log('script end');
```

先猜测一下这段代码的输出顺序是什么，再去浏览器控制台输入一下，看看实际输出的顺序和你猜测出的顺序是否一致，如果一致，那就说明，你对JavaScript的事件循环机制还是有一定了解的，继续往下看可以巩固下你的知识；而如果实际输出的顺序和你的猜测不一致，那么本文下面的部分会为你答疑解惑。

# 任务队列

所有的任务可以分为同步任务和异步任务，同步任务，顾名思义，就是立即执行的任务，同步任务一般会直接进入到主线程中执行；而异步任务，就是异步执行的任务，比如ajax网络请求，setTimeout定时函数等都属于异步任务，异步任务会通过任务队列(Event Queue)的机制来进行协调。具体的可以用下面的图来大致说明一下：

![alt Event Queue示意图](/images/EventQueue1.png)

同步和异步任务分别进入不同的执行环境，同步的进入主线程，即主执行栈，异步的进入Event Queue。主线程内的任务执行完毕为空，会去Event Queue读取对应的任务，推入主线程执行。
上述过程的不断重复就是我们说的Event Loop(事件循环)。

在事件循环中，每进行一次循环操作称为tick，通过阅读[规范](https://www.w3.org/TR/html5/webappapis.html#event-loops-processing-model)可知，每一次tick的任务处理模型是比较复杂的，其关键的步骤可以总结如下：

1. 在此次tick中选择最先进入队列的任务(oldest task)，如果有则执行(一次)
2. 检查是否存在Microtasks，如果存在则不停地执行，直至清空Microtask Queue
3. 更新render
4. 主线程重复执行上述步骤

可以用一张图来说明下流程：
![alt Event Queue示意图](/images/eventqueue2.png)

这里相信有人会想问，什么是microtasks?规范中规定，task分为两大类, 分别是Macro Task （宏任务）和Micro Task（微任务）, 并且每个宏任务结束后, 都要清空所有的微任务,这里的Macro Task也是我们常说的task，有些文章并没有对其做区分，后面文章中所提及的task皆看做宏任务(macro task)。

(macro)task主要包含：script(整体代码)、setTimeout、setInterval、I/O、UI交互事件、setImmediate(Node.js 环境)

microtask主要包含：Promise、MutaionObserver、process.nextTick(Node.js 环境)

setTimeout/Promise等API便是任务源，而进入任务队列的是由他们指定的具体执行任务。来自不同任务源的任务会进入到不同的任务队列。其中setTimeout与setInterval是同源的。


# 分析示例代码

千言万语，不如就着例子讲来的清楚。下面我们可以按照规范，一步步执行解析下上面的例子，先贴一下例子代码（免得你往上翻）。

```javascript
console.log('script start');

setTimeout(function() {
  console.log('setTimeout');
}, 0);

Promise.resolve().then(function() {
  console.log('promise1');
}).then(function() {
  console.log('promise2');
});

console.log('script end');
```

1. 整体script作为第一个宏任务进入主线程，遇到console.log，输出`script start`
2. 遇到setTimeout，其回调函数被分发到宏任务Event Queue中
3. 遇到Promise，其then函数被分到到微任务Event Queue中,记为then1，之后又遇到了then函数，将其分到微任务Event Queue中，记为then2
4. 遇到console.log，输出`script end`

至此，Event Queue中存在三个任务，如下表：

宏任务      | 微任务  
---------  | -------- 
setTimeout |  then1  
           |  then2

5. 执行微任务，首先执行then1，输出`promise1`,然后执行then2，输出`promise2`，这样就清空了所有微任务
6. 执行setTimeout任务，输出`setTimeout`
至此，输出的顺序是：`script start, script end, promise1, promise2, setTimeout`

so，你猜对了吗？

# 看看你掌握了没

再来一个题目，来做个练习：

```javascript
console.log('script start');

setTimeout(function() {
  console.log('timeout1');
}, 10);

new Promise(resolve => {
    console.log('promise1');
    resolve();
    setTimeout(() => console.log('timeout2'), 10);
}).then(function() {
    console.log('then1')
})

console.log('script end');
```

这个题目就稍微有点复杂了，我们再分析下：

首先，事件循环从宏任务(macrotask)队列开始，最初始，宏任务队列中，只有一个script(整体代码)任务；当遇到任务源(task source)时，则会先分发任务到对应的任务队列中去。所以，就和上面例子类似，首先遇到了console.log，输出`script start`；
接着往下走，遇到setTimeout任务源，将其分发到任务队列中去，记为timeout1；
接着遇到promise，new promise中的代码立即执行，输出`promise1`,然后执行resolve,遇到setTimeout,将其分发到任务队列中去，记为timemout2,将其then分发到微任务队列中去，记为then1；
接着遇到console.log代码，直接输出`script end`
接着检查微任务队列，发现有个then1微任务，执行，输出`then1`
再检查微任务队列，发现已经清空，则开始检查宏任务队列，执行timeout1,输出`timeout1`；
接着执行timeout2，输出`timeout2`
至此，所有的都队列都已清空，执行完毕。其输出的顺序依次是：`script start`, `promise1`, `script end`, `then1`, `timeout1`, `timeout2`

用流程图看更清晰：

![alt Event Queue示意图](/images/eventqueue3.png)

# 总结

有个小tip：从规范来看，microtask优先于task执行，所以如果有需要优先执行的逻辑，放入microtask队列会比task更早的被执行。

最后的最后，记住，JavaScript是一门单线程语言。

# 参考文献

[这一次，彻底弄懂 JavaScript 执行机制](https://juejin.im/post/59e85eebf265da430d571f89)
[Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
[从一道题浅说 JavaScript 的事件循环](https://github.com/dwqs/blog/issues/61)







