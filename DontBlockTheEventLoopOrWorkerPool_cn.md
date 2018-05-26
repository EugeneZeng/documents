# 不要阻塞事件循环（或工作池） #
## 原文：《[Don't Block the Event Loop (or the Worker Pool)](https://nodejs.org/en/docs/guides/dont-block-the-event-loop/ "Don't Block the Event Loop (or the Worker Pool)")》 ##
### 你该阅读这篇文章吗？（Should you read this guide?） ###
如果你正在写一些比简单的命令行脚本更复杂的东西，阅读这篇文章应该会帮助你写出更加高效，更加安全的应用程序。<br />
这篇文章是用Node服务器的构思写的，但是这个概念也可以应用在复杂的Node应用程序上。考虑到操作系统的差异，这个文档是以Linux为中心的。
### TL; DR ###
Node.js在事件循环里执行Javascript代码（初始化和回调），和提供一个工作池来处理繁重的任务比如文件I/O。Node的可扩展性强，有时候还好于更重量级的实现例如Apache。秘诀就在与Node的可伸缩性是用了很少数量的线程来处理许多任务。如果Node可以使用更少的线程，那它就可以把系统时间和内存用在任务上，而非花更多的时间和空间在系统开销上（内存和上下文切换）。但是，因为Node只有少量的线程，所以你必须明智地使用它们构建你的应用程序。<br />
这里有个很好的经验法则来保持你的Node服务器快速运行：_在给定时间内，与每个客户相关的工作都是小的，Node就会很快。(Node is fast when the work associated with each client at any given time is "small")_<br />
这个应用在事件循环的回调和工作池的任务上。

### 为什么我应该避免阻塞事件循环和工作池? ###
Node用很少数量的线程来处理许多客户。在Node里面，有两种线程：事件循环（又称主池，主线程，事件线程等等），和在工作池里面的`k`工作池（又称线程池）。<br />
如果一个线程用了很长时间去执行一个回调（事件循环）或者一个任务（工作），我们就叫它“阻塞”。当一个线程阻塞了代表一个客户的工作流，它就不能够处理来自于其他任何客户的请求。这就提供了两种阻塞事件循环和工作流的动机：
1. 性能：如果你经常在两种线程中实现重量级的行为，你的服务器的吞吐量（请求数/秒）将受严重影响。
2. 安全：如果你的某个线程会因某种特定的输入而阻塞，那么恶意用户就可能会提交这种“恶意输入”，让你的线程阻塞，而不能服务于其他客户。这就是拒绝服务（[Denial of Service](https://en.wikipedia.org/wiki/Denial-of-service_attack "Denial of Service ")）攻击。

### 快速回顾下Node ###
Node使用事件驱动的架构：用事件循环密谋，用工作池来干重活。

什么样的代码会在事件循环中运行？<br />
当它们开始的时候，Node应用程序首先会完成一个初始化阶段，加载模块和为事件注册回调。Node应用程序然后就会进入事件循环。运行合适的回调来响应客户的请求。这些请求同步地运行，当它完成以后有可能会注册异步的请求来继续处理。这些异步请求的回调也将会在事件循环中运行。<br />
事件循环也会用回调来实现非阻塞的异步请求，比如网络I/O。<br />
总体来说，事件循环执行了为事件注册的回调，它还负责实现像网络I/O那样的非阻塞的异步请求。

什么样的代码会在工作池中运行？<br />
Node的工作池是在libuv（[docs](http://docs.libuv.org/en/v1.x/threadpool.html "libuv documents")）里实现的，它暴露了通用的任务提交API。<br />
Node用工作池来处理“繁重”的任务。这就包括操作系统没提供非阻塞版本的I/O，也就是常见的CPU密集型任务。<br />
以下就是Node内置模块API中用到了工作池的部分：
1. I/O密集型
    1. [DNS](https://nodejs.org/api/dns.html "DNS")：`dns.lookup()`，`dns.lookupService()`。
    2. [File System](https://nodejs.org/api/fs.html#fs_threadpool_usage "File System")：除了`fs.FSWatcher()`和那些明显同步使用了libuv线程池之外的所有文件系统APIs。
2. CPU密集型
    1. [Crypto](https://nodejs.org/api/crypto.html "Crypto")： `crypto.pbkdf2()`，`crypto.randomBytes()`，`crypto.randomFill()`。
    2. [Zlib](https://nodejs.org/api/zlib.html#zlib_threadpool_usage "Zlib")： 除了些明显同步使用了libuv线程池之外的所有zlib APIs。

在许多Node应用程序里，这些API是工作池任务的唯一来源。使用[C++ add-on](https://nodejs.org/api/addons.html "C++ add-on")的那些应用程序可以给工作池提交其他任务。<br />
为了完整性起见，我们注意到，当你从事件循环的回调里面调用这些API的时候，事件循环会花费较小的配置成本，因为它为这些API进入Node的C++绑定，并将任务提交给工作池。这些成本相比与整体的任务成本来说是微不足道的（negligible），这就是为什么在事件循环来搞定它的原因。当提交这些任务中的一个到工作池，Node提供了一个在C++绑定里的指针到对应的C++方法。

Node如何决定下一步运行什么代码？<br />
理论上地说，事件循环和工作池分别维护了正在等待的事件和任务的队列。<br />
事实上，事件循环并没有真正的维护一个队列。而是一个文件描述符的集合，让操作系统去监视，用一种机制类似于[epoll](http://man7.org/linux/man-pages/man7/epoll.7.html "epoll")（Linux），[kqueue](https://developer.apple.com/library/content/documentation/Darwin/Conceptual/FSEvents_ProgGuide/KernelQueues/KernelQueues.html "kqueue")（OSX），event ports（Solaris）或者[IOCP](https://msdn.microsoft.com/en-us/library/windows/desktop/aa365198.aspx "IOCP")（windows）。这些文件描述符对应了网络Sockets和监视中的文件等等。当操作系统说这些文件描述符中的一个已经准备好了，事件循环就会把它翻译成对应的事件，而后执行这个事件附加的回调。关于这个过程的更多情况你可以从[这里](https://www.youtube.com/watch?v=P9csgxBgaZ8 "youtube链接哦，需要翻墙观看")得到。<br />
作为对比，工作池用了一个真正的队列，其中的实体是需要被处理的任务。一个工作器从他的队列里弹出一个任务来处理，当完成以后，工作器就会发出一个“至少有一个任务完成咯”的事件给事件循环。

这对应用程序的设计意味着什么？<br />
在一个单人单线程的系统中，比如Apache，每一个等待中的客户都会被安排一个属于它自己的线程。如果一个正在处理中的客户线程阻塞了，操作系统就会中断它，轮给另一个客户。因此操作系统就会保证那些只需要少量操作的客户不会受到那些需要大量操作的客户的影响。<br />
因为Node用少量的线程来处理大量的客户，如果一个线程因处理一个客户的请求而阻塞了，那么正在等待中的客户请求就会在线程完成回调或者任务之前都轮不到。_因此，对客户的公平对待是你的程序的责任_。这就意味着你不应该为任何客户的任何一个回调或者任务做太多事情。<br />
这就是为什么Node能够很好扩展的部分原因，这也意味着你必须负责保证公平地调度。接下来的部分会讲到如何为事件循环和工作池保证公平的调度。

### 别阻塞事件循环（Don't block the Event Loop） ###
事件循环通知每个新的客户连接，安排响应的生成。所有输入的请求和输出的返回都会在事件循环里头传递。这就意味着如果事件循环在某个点上花了太长时间，那么所有的当前或者新的客户就会没机会轮到。<br />
你应该保证永不阻塞事件循环。换句话说，每个Javascript回调都应该尽快完成。这当然也要应用在你的`await`,`Promise.then`等等的语句中。<br />
保证这一点的好办法就是推测回调的“[计算复杂度（computational complexity）](https://en.wikipedia.org/wiki/Time_complexity "computational complexity")”。如果你的回调使用了固定数量的步骤，不管它的参数是什么，那你将永远给予每个等待中的客户公平的机会。如果你的回调方法根据不同的参数执行了不同数量的步骤，那你就应该考虑一下它可能会有多长的参数。<br />
范例一：固定次数的回调
```js
app.get('/constant-time', (req, res) => {
  res.sendStatus(200);
});
```
范例二：一个`0(n)`的回调。这个回调会因小数值的n而运行得较快，大数值的n而运行得较慢。
```js
app.get('/countToN', (req, res) => {
  let n = req.query.n;

  // n iterations before giving someone else a turn
  for (let i = 0; i < n; i++) {
    console.log(`Iter {$i}`);
  }

  res.sendStatus(200);
});
```
范例三：一个`0(n^2)`的回调。这个回调同样会因小数值的n而运行得较快，但会因大数值的n而运行得比前一个0(n)的例子慢得多。
```js
app.get('/countToN2', (req, res) => {
  let n = req.query.n;

  // n^2 iterations before giving someone else a turn
  for (let i = 0; i < n; i++) {
    for (let j = 0; j < n; j++) {
      console.log(`Iter ${i}.${j}`);
    }
  }

  res.sendStatus(200);
});
```

===未完待续===To be Continue===