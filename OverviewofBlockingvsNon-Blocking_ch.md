# 概述阻塞和非阻塞 #
## 原文：《[Overview of Blocking vs Non-Blocking](https://nodejs.org/en/docs/guides/blocking-vs-non-blocking/)》 ##
本文概述了Node.js的阻塞和非阻塞调用的区别。本文会提到事件循环和libuv，但你不需要事先就对它们有认识。我们假定读者都对Javascript语言和Node.js的回调模式有基本的了解。

    “I/O”主要是指在libuv的支持下，与系统磁盘和网络进行的交互。

### 阻塞（Blocking） ###
**阻塞**是指当Node.js进程里的Javascript执行必须等待非Javascript操作完成的时候。这是因为事件循环在**阻塞**操作正在发生的时候是不能够继续运行Javascript的。<br />
在Node.js里，Javascript呈现低下的性能是因为它是cpu密集型的，而非正在进行非Javascript操作。就像是I/O，通常不会认为是**阻塞**。在Node.js标准库里使用libuv的同步方法大部分都是用了**阻塞**操作。本地模块也有**阻塞**的方法。<br />
在Node.js标准库里的所有I/O方法都提供了异步版本，也就是非阻塞的，接受回调函数。一些方法也有阻塞的副本，方法名以`Sync`结尾。

### 代码比对 ###
**阻塞**方法同步地执行，而**非阻塞**方法异步地执行。<br />
用文件系统模块作为例子，下面是同步的文件读取：
```js
const fs = require('fs');
const data = fs.readFileSync('/file.md'); // 阻塞在这里，直到文件读取完毕。
```
下面是等价的异步例子：
```js
const fs = require('fs');
fs.readFile('/file.md', (err, data) => {
  if (err) throw err;
});
```
第一个例子看起来比第二个要简单，但是会有个劣势，那就是第二行会**阻塞**其他Javascript代码的执行，直到整个文件读取完毕。需要注意的是，在同步的版本中，如果抛出了错误，那就必须要被捕捉，否则整个进程就会崩溃。在异步的版本中，这就要由作者来决定这个错误是否被抛出来显示。<br />
让我们稍微扩展以下我们的例子：
```js
const fs = require('fs');
const data = fs.readFileSync('/file.md'); // 阻塞在这里，直到文件读取完毕。
console.log(data);
// moreWork(); 将在 console.log 之后运行
```
而以下是一个类似但不等价的异步例子：
```js
const fs = require('fs');
fs.readFile('/file.md', (err, data) => {
  if (err) throw err;
  console.log(data);
});
// moreWork(); 将在 console.log 之后运行
```
在上面的第一个例子里，`console.log`会在`moreWork()`之前执行。在第二个例子里，`fs.readFile()`是非阻塞的，所以Javascript得以继续，`moreWork()`会先被调用。这种运行`moreWork()`而不需要等待文件读取完毕的能力就是一个关键的设计取向，实现了更高的吞吐量。

### 并发和吞吐量(Concurrency and Throughput) ###
Javascript在Node.js里面的运行是单线程的,所以并发就指事件循环在完成其他工作之后执行Javascript回调函数的能力。


==未完待续==