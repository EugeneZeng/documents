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
2. 处理在**poll**队列中的事件

当事件循环进入**poll**阶段，同时也没有安排定时器，那么以下两个事情当中的一个就会发生：
* _如果轮询队列非空_，事件循环就会遍历它的回调队列，以同步的方式执行它们，直到队列耗尽为止，或者是到达了系统依赖的固定限制。
* _如果轮询队列为空_，则会发生以下两种情况之一：
  * 如果脚本是被`setImmediate()`安排的，事件循环就会结束当前的**poll**阶段，进入**check**阶段来执行那些安排好的脚本。
  * 如果脚本不是被`setImmediate()`安排的，事件循环就会等待回调加入队列，然后立即执行它们。

一旦**poll**队列空了，事件循环就会去检查哪些已经到达阀值的定时器，如果一个或者多个定时器已经准备好了，事件循环就会回到**timers**阶段，执行这些定时器的回调。

检查（check）<br />
这个阶段允许人们在**poll**阶段结束后立即执行回调。如果**poll**阶段变得空闲而脚本已经跟随`setImmediate()`加入队列，事件循环就会继续**check**阶段而不是等待。<br />
`setImmediate()`实际上是一个特殊的定时器，运行在事件循环的特定阶段。它用libuv的API在**poll**阶段结束后来安排回调的执行。<br />
一般来说，随着代码的运行，事件循环最终会进入**poll**阶段，等待输入的连接，请求等等。然而，一个回调已经跟随`setImmediate()`加入队列，**poll**阶段就变得空闲了。他会结束掉然后继续**check**阶段而不是等待**poll**事件。

关闭回调（close callbacks）<br />
如果一个socket或者处理器突然被关闭（例如`socket.destroy()`），一个`close`事件将会在这个阶段触发。否则它就会通过`process.nextTick()`来触发。

## `setImmediate()` VS `setTimeout()` ##
`setImmediate()` 和 `setTimeout()`很相似，但它们按不同的方式行事，视乎它们在何时调用。
* `setImmediate()`是设计用来在一旦当前**poll**阶段完成的时候就执行脚本。
* `setTimeout()`是安排一个脚本在指定阀值毫秒数流逝后尽快执行。

定时器执行的顺序会不一样，视乎它们执行时候的上下文。如果两个都从主模块里调用，那么时间将与处理性能相关（可能会被这个机器上正在运行的其他应用程序影响到）。<br />
例如我们不在I/O环（即主模块）里面执行以下脚本，两个定时器执行的顺序是无法确定的，因为它跟处理性能相关：
```js
// timeout_vs_immediate.js
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});
```
```
$ node timeout_vs_immediate.js
timeout
immediate

$ node timeout_vs_immediate.js
immediate
timeout
```
然而，如果你把两个调用放到一个I/O环里面，`setImmediate()`将永远都是第一个被执行的。

```js
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```
```
$ node timeout_vs_immediate.js
immediate
timeout

$ node timeout_vs_immediate.js
immediate
timeout
```
使用`setImmediate()` 而非 `setTimeout()`主要的优势在于`setImmediate()` 在I/O环里，将永远先于其他定时器执行，不管在场有多少个定时器。

## `process.nextTick()` ##
理解`process.nextTick()`<br />
你可能已经注意到了`process.nextTick()`并没有出现在这个示意图里面，尽管它是异步API的一部分。这是因为`process.nextTick()`不是事件循环技术上的部分。应该说，`nextTickQueue`会在当前操作完成后进行，不管事件循环的当前阶段。<br />
回去看一下我们的示意图，在给定的阶段，任何时候调用`process.nextTick()`，所有传递给`process.nextTick()`的回调都会在事件循环继续之前解决。这样会创建一些非常坏的状况，因为它允许你**通过递归地调用`process.nextTick()`从而使I/O匮乏**，就会阻止事件循环到达**poll**阶段。

为什么会允许这样呢？<br />
为什么这种东西会加到Node.js里面来呢？一部分是因为一种设计哲学：一个API必须一直是异步的，即使有些地方它不一定是。以下列代码片段为例：
```js
function apiCall(arg, callback) {
  if (typeof arg !== 'string')
    return process.nextTick(callback,
                            new TypeError('argument should be string'));
}
```
这个片段是用来做参数检查，如果不对，它将会传一个Error对象给回调函数。最近更新的API允许将参数传递给process.nextTick()，允许它在回调后传递的任何参数作为回调的参数，这样就不必嵌套函数了。<br />
我们正在做的是仅当允许用户的其他代码执行后给用户传回一个Error。通过使用`process.nextTick()`，我们保证了`apiCall()`一定在用户的其他代码之后和在事件循环被允许继续之前会运行他的回调。为了达到这个目的，JS的执行栈被允许解开，然后立即执行提供的回调，这样就允许一个人使用递归调用`process.nextTick()`，而不会得到`RangeError: Maximum call stack size exceeded from v8`。<br />
这种哲学思维可能会造成一些潜在的困境。以下面的代码片段为例：
```js
let bar;

// this has an asynchronous signature, but calls callback synchronously
function someAsyncApiCall(callback) { callback(); }

// the callback is called before `someAsyncApiCall` completes.
someAsyncApiCall(() => {
  // since someAsyncApiCall has completed, bar hasn't been assigned any value
  console.log('bar', bar); // undefined
});

bar = 1;
```
用户定义了`someAsyncApiCall()`去获取一个异步的签名，但实际上它同步地操作了。当它被调用的时候，`someAsyncApiCall()`提供的回调被在事件循环的同一阶段执行了，因为`someAsyncApiCall()`实际上并没有做任何异步的事情。结果就是，回调函数尝试引用`bar`，即使当前作用域没有那个变量，因为脚本无法运行到完成。<br />
通过把回调放在一个`process.nextTick()`里，这个脚本仍然有能力执行到最后，允许所有的变量，函数等等在回调执行之前进行初始化。它还具有不允许事件循环继续的优点。这可能对那些想在事件循环继续之前想对错误发出警报的用户有用。以下是使用了`process.nextTick()`的上面那个例子：
```js
let bar;

function someAsyncApiCall(callback) {
  process.nextTick(callback);
}

someAsyncApiCall(() => {
  console.log('bar', bar); // 1
});

bar = 1;
```
这是另一个真实的例子：
```js
const server = net.createServer(() => {}).listen(8080);

server.on('listening', () => {});
```
当只有一个端口号被传入的时候，这个端口号立即就被绑定了。所以`'listening'`的回调会立即被调用。问题在于`.on('listening')`在这个时候还没有被设置呢。

为了搞定这个，使`'listening'`事件加入到一个`nextTick()`的队列里来使脚本得以运行到最后。这样就允许用户设置任何他们想要的事件处理器了。

`process.nextTick()` VS `setImmediate()`<br />
就用户关心的而言，我们有两个类似的方法，但它们的名字令人困惑。
* `process.nextTick()`在同一个阶段里立即触发。
* `setImmediate()`在接下来的阶段或者是事件循环的某个瞬间触发。

本质上来说，它们俩的名字应该调换过来。`process.nextTick()`比`setImmediate()`触发地更加及时。但这是过去的产物了，不太可能改变。让它俩名字交换可能会破坏NPM上的很大一部分的包。每天都会有很多新的模块加入到NPM，意味着我们等的每一天，都会有更多潜在的问题发生。当它们令人困惑的时候，名字自己并不会改变。<br />
_我们推荐在任何情况下都使用`setImmediate()`，因为它更加容易理解（还有就是它让代码兼容更加广泛的执行环境，例如浏览器的js）_

==未完待续
