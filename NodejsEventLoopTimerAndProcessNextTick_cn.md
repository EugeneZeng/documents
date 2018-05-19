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

在事件循环的每次运行之间，Node.js都会检查它是否在等待异步I/O或者定时器，如果没有的话就关闭它。

### 细说每个阶段 ###
定时器（timers）<br />
定时器指定了一个阈值，在这个阈值之后，提供的回调会被执行，而不是人们希望它被执行的确切时间。定时器的回调会在指定的时间结束后被尽早安排运行。然而，操作系统的调度或者其他回调的运行可能会使它延时。<br />
_注：技术上来说，[轮训阶段（poll phase）](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#poll)控制定时器何时被执行_

例如，你安排了一个定时任务在100ms后运行，然后你的脚本开始异步读取一个文件花了95ms。
```js
const fs = require('fs');

function someAsyncOperation(callback) {
  // Assume this takes 95ms to complete
  fs.readFile('/path/to/file', callback);
}

const timeoutScheduled = Date.now();

setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;

  console.log(`${delay}ms have passed since I was scheduled`);
}, 100);


// do someAsyncOperation which takes 95 ms to complete
someAsyncOperation(() => {
  const startCallback = Date.now();

  // do something that will take 10ms...
  while (Date.now() - startCallback < 10) {
    // do nothing
  }
});
```
当事件循环进入**poll**阶段的时候，它有一个空的队列（`fs.readFile()`还没有完成），所以它会等多几个毫秒，直到最快的那个定时器的阀值为止。当它一直等到95ms过去了以后，`fs.readFile()`完成文件读取，它的回调花了10ms来加入轮询队列和执行。当这个回调完成了以后，队列中就没有其他回调了。这个时候事件循环就会看到最近的那个定时器的阀值已经到达，然后回到**timers**阶段去执行这个定时器的回调。在这个例子里，你会看到从定时器被安排好到它的回调被执行完毕的整个时长将会是105ms。

_注：为了避免**poll**阶段让事件循环处于饥饿状态，[libuv](http://libuv.org/)（一个C的类库实现了Node.js的事件循环和所有这个平台上的异步行为）在它停止更多事件加入轮询之前，会有一个固定的最大限度（视乎系统配置）_

等待回调（pending callback）<br />
这个阶段为一些系统操作执行回调，比如TCP错误的类型。例如一个TCP的socket在尝试连接的时候接收到`ECONNREFUSED`信号，一些类Unix系统就会希望等待报告错误。这就会在等待**pending callback**进行排队执行。

轮询（poll）<br />
**poll**阶段有两个很重要的功能：<br />
1. 计算它会阻塞或者轮询I/O多长时间，然后；
2. 处理在**poll**队列中的事件<br />
当事件循环进入轮询阶段，同时也没有安排定时器，那么以下两个事情当中的一个就会发生：
* _如果轮询队列非空_，事件循环就会遍历它的回调队列，以同步的方式执行它们，直到队列耗尽为止，或者是到达了系统依赖的固定限制。
* _如果轮询队列为空_，则会发生以下两种情况之一：
  * 如果脚本是被`setImmediate()`安排的，事件循环就会结束当前的**poll**阶段，进入**check**阶段来执行那些安排好的脚本。
  * 如果脚本不是被`setImmediate()`安排的，事件循环就会等待回调加入队列，然后立即执行它们。

一旦**轮询**队列空了，事件循环就会去检查哪些已经到达阀值的定时器，如果一个或者多个定时器已经准备好了，事件循环就会回到**timers**阶段，执行这些定时器的回调。

==未完待续
