# NodeJS的事件循环，定时器和`process.nextTick()` #
## 原文：《[The Node.js Event Loop, Timers, and process.nextTick()](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)》 ##
### 什么是事件循环 ###
事件循环就是让nodejs实现非阻塞I/O操作——尽管事实上Javascript是单线程的——尽可能将操作转嫁到系统内核中去。<br />
因为大部分的现代内核都是多线程的，它们能够通过后台运行的方式同时处理多个操作。如果这些操作中的任意一个完成了，内核就会告诉NodeJS，这样的话相关的回调函数就会被添加到**轮询**队列（poll queue），最终会被执行。关于这个话题的细节，我们稍后会聊到。

### 解释事件循环 ###
一旦Node.js启动，它就会启动事件循环，处理提供过来的输入脚本（或者是丢到[REPL](https://nodejs.org/api/repl.html#repl_repl)里面去，但这不在这个文档的讨论范围），可能是要做异步API调用，定时任务或者是调用`process.nextTick()`，然后开始进行事件循环。<br />
下面的示意图是对一个事件循环的操作顺序的简单概述<br />
![the event loop's order of operations](/img/OrderOfOperations.png)<br />
_注：每一个方块都表示事件循环的一个阶段_

每个阶段都有一个回调的先进先出（FIFO）队列会执行。每个阶段都是不一样的，有自己的独特方式。一般来说，当事件循环进入一个给定的阶段，它就会去实现这个阶段指定的任何操作，然后执行这个队列中的回调直到队列已经耗尽或者已经运行了最大数量的回调。当队列已经耗尽或者到达回调数量限制，事件循环就会进入下一个阶段，以此类推。<br />
由于这些操作中的一部分可能会安排更多操作，在轮询阶段处理的新事件将由内核排队，当正在处理轮询中的事件的时候，新进的事件可以排队。因此，耗时较长的回调可能会使阶段轮询运行比预期更长的时间。更多细节请参考[timers](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#timers)和[poll](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#poll)章节。

_注：在Windows和Unix/Linux中的实现有一些小小的差异，但这对于这个话题来说并不重要。最重要的部分在这里，实际上一共有七到八步，但我们关心的，也是Node.js实际上用到的，也就是以上这些。_

### 简述每个阶段 ###
+ **timers**：这个阶段执行`setTimeout()`和`setInterval()`安排的回调。
+ **pending callbacks**：运行被推迟到下次循环迭代的I/O回调
+ **idle, prepare**：只限在内部使用
+ **poll**：接收新的I/O事件；执行I/O相关的回调（差不多是所有那些有异常的回调，那些被定时器安排的和`setImmediate()`）；当时机成熟Node会在这里阻塞。
+ **check**：`setImmediate()`的回调会在这里执行。
+ **close callbacks**：一些关闭的回调，比如`socket.on('close', ...)`。

==未完待续
