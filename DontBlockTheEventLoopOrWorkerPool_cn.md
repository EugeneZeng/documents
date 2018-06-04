# 别阻塞事件循环（或工作池） #
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
那你该多小心呢？<br />
Node用Google的V8引擎来解析Javascript，对普通的操作来说已经是很快的了。不过正则表达式和JSON的操作是例外,以下会讨论到。<br />
然而，对于复杂的任务，你应该考虑限制输入和拒绝太长的输入。那样的话，即使你的回调函数非常复杂，通过限制输入，也可以让你保证回调不会比处理最长输入的最坏情况花的时间多。然后你就可以评估这个回调的最坏情况成本和决定它的运行时间是否在你的接受范围内。

事件循环的阻塞：REDOS<br />
一个常见的方法来阻塞事件循环，是用一种“脆弱（vulnerable）”的正则表达式。

避免脆弱的正则表达式<br />
一个正则表达式（regexp）依据一个模式语句（pattern）来匹配输入的字符串。我们通常认为一个正则表达式的匹配需要执行对输入字符串的一次扫描（pass through）——`0(n)`复杂度，n为字符串的长度。在多数的情况下，单次扫描的确是必须的。<br />
不幸的是，在一些情况下，正则匹配可能会要求指数级`0(2^n)`的扫描测试（trips）次数。一个指数级别的扫描测试意味着如果引擎要求`x`次测试来决定一个匹配，如果输入字符串长度大于一那它就需要`2^x`次的扫描测试。<br />
一个*脆弱的正则表达式（ulnerable regular expression）*就是那个让你的正则表达式引擎运行指数级别的次数，用“恶意输入”使你遭受[正则表达式拒绝服务（REDOS）](https://www.owasp.org/index.php/Regular_expression_Denial_of_Service_-_ReDoS "Regular expression Denial of Service - ReDoS")攻击。你的正则表达式模式是否脆弱（即正则引擎可能会在基于它执行指数次数的扫描测试），实际上是个很难回答的问题，大多数情况下都取决于你是否正在使用Perl，Python，Ruby，Java或者JavaScript等等。但是这里有一些适用于这些语言的一些经验法则：
1. 避免使用嵌套的数量符，例如`(a+)*`。Node的正则引擎虽然可以很快地处理其中的一些，但是另外一些就是脆弱的。
2. 避免使用了重叠字句的OR，例如`(a|a)*`。同样地，这些有时候会很快。
3. 避免使用反向引用，例如`(a.*) \1`。没有任何一个正则引擎能保证在线性次数`0(n)`里对它进行评估。
4. 如果你只需要简单的字符串匹配，用`indexOf`或者是语法等价比较。这样比较简单而且永远不会需要`2^x`次的扫描测试。<br />
如果你不确定你的正则表达式是否脆弱的，记住Node通常不会有问题报告，来报告一个脆弱正则或者长输入字符的_匹配_。当有不匹配但Node不能确定直到它在输入字符串上尝试了很多种路径以后，这样指数行为就被触发了。

一个REDOS的例子<br />
下面是个脆弱正则的例子，使它的服务器暴露在REDOS的威胁中：
```js
app.get('/redos-me', (req, res) => {
  let filePath = req.query.filePath;

  // REDOS
  if (fileName.match(/(\/.+)+$/)) {
    console.log('valid path');
  }
  else {
    console.log('invalid path');
  }

  res.sendStatus(200);
});
```
在这个例子中的脆弱正则是一种在Linux里检查有效路径的（差劲的）方法。它匹配那些用“/”分割的路径名组成的字符串，例如“a/b/c”。这是危险的，因为它违反了规则一：它有双层嵌套的数量符。<br />
如果用户查询一个用`///.../\n`（100个“/”加上一个正则的“.”不会匹配的换行符），那么事件循环将会永远不停，这样就阻塞了事件循环。这个用户的REDOS攻击将导致其他用户在这个正则匹配完成之前无法被轮到。<br />
由于这个原因，你应该在使用复杂的正则匹配去校验用户输入的时候聪明点。

防REDOS攻击的资源<br />
有些工具可以检查你的正则表达式的安全性，比如：
* [safe-regex](https://github.com/substack/safe-regex "safe-regex 正则安全性检查工具")
* [rxxr2](http://www.cs.bham.ac.uk/~hxt/research/rxxr2/ "rxxr2 正则安全性检查工具")

然而，这两个都不能够检查到所有的脆弱的正则表达式。<br />
另一个途径就是使用不同的正则引擎。你可以使用node-re2模块，一个使用了Google的极速RE2的正则引擎。警告，RE2不是100%兼容Node的正则表达式，所以如果你用node-re2模块来处理你的正则表达式，就必须回归检查一下。特别复杂的正则表达式是不被node-re2支持的。<br />
如果你尝试匹配一些“明显”的东西，像URL或者是文件路径之类的，可以从[正则库（regexp library）](http://www.regexlib.com/ "正则库（regexp library）")里找个例子或者使用一个npm模块，例如[ip-regex](https://www.npmjs.com/package/ip-regex "re-regex")。

### 阻塞事件循环：Node的核心模块 ###
一些Node的核心模块有同步的重型API，包括：
* [Encryption](https://nodejs.org/api/crypto.html "Encryption")
* [Compression](https://nodejs.org/api/zlib.html "Compression")
* [File system](https://nodejs.org/api/fs.html "File system")
* [Child process](https://nodejs.org/api/child_process.html "Child process")

这些API是繁重的，因为他们都包含了繁重的计算任务（encryption, compression）、请求I/O（File System）或者可能两者都包含（Child process）。这部分（同步的）APIs是为方便脚本编写而设计的，但并不打算在服务器环境中使用。如果你在事件循环里执行它们，那将花上远比普通JS指令更长的时间来完成。<br />
在服务器里，_你不应该使用这些模块中以下的APIs：_
* Encryption：
  * `crypto.randomBytes` （同步版本）
  * `crypto.randomFillSync`
  * `crypto.pbkdf2Sync`
  * 你也应该小心那些提供大型的输入给加密和解密的日常其他API。
* Compression：
  * `zlib.inflateSync`
  * `zlib.deflateSync`
* File system：
  * 别使用同步的文件系统APIs。例如，如果你正在访问的文件是在像[NFS](https://en.wikipedia.org/wiki/Network_File_System)那样的[分布式文件系统](https://en.wikipedia.org/wiki/Clustered_file_system#Distributed_file_systems "distributed file system 分布式文件系统")里，那么访问时间是无法想象的。
* Child process：
  * `child_process.spawnSync`
  * `child_process.execSync`
  * `child_process.execFileSync`

这个列表截至到Node V9版本还是完整的。

### 阻塞事件循环：JSON DOS（拒绝服务） ###
`JSON.parse` 和 `JSON.stringify`是其他的潜在的重型操作。虽然这些（操作时间）是输入长度的`0(n)`，但是对于大的`n`，操作时间可能是出奇地长。<br />
如果你的服务器操作JSON对象，特别是那些来自于客户的，对于需要你在事件循环里处理的JSON对象或字符串的大小，你就应该特别谨慎了。<br />
例子：JSON阻塞。我们创建一个2^21大小的对象`obj`，然后用`JSON.stringify`它，在字符串上执行`indexOf`，最后用`JSON.parse`一下它。`JSON.stringify`后的字符串有50MB，花了0.7秒来完成，然后花了0.03秒来在这个50MB的字符串上完成`indexOf`，然后花上1.3秒来完成对这个字符串的`JSON.parse`操作。
```js
var obj = { a: 1 };
var niter = 20;

var before, res, took;

for (var i = 0; i < len; i++) {
  obj = { obj1: obj, obj2: obj }; // Doubles in size each iter
}

before = process.hrtime();
res = JSON.stringify(obj);
took = process.hrtime(n);
console.log('JSON.stringify took ' + took);

before = process.hrtime();
res = str.indexOf('nomatch');
took = process.hrtime(n);
console.log('Pure indexof took ' + took);

before = process.hrtime();
res = JSON.parse(str);
took = process.hrtime(n);
console.log('JSON.parse took ' + took);
```
这里有一些npm的模块，提供了异步的JSON API，比如说：
* [JSONStream](https://www.npmjs.com/package/JSONStream)，拥有流式API。
* [Big-Friendly JSON](https://github.com/philbooth/bfj)，拥有流式API，就像是JSON标准APIs的异步版本，使用了下面提到的事件循环上的分区（partitioning-on-the-Event-Loop）模式。

### 无需阻塞事件循环的复杂运算 ###
或许你想用Javascript来作一些复杂的运算而不想因此阻塞事件循环。那你有两种选择：分区（partitioning）和分流（或称负载转移，offloading）。<br />

分区（partitioning）<br />
你可以拆分你的计算，这样的话每一次在事件循环上的运行（后），都会定期地（把资源）分配给其他正在等待的事件（来处理）。在Javascript里，是很容易在一个闭包里保存一个正在运行的任务的状态，就像下面的范例2中的一样。<br />
举个简单的例子，试想你要计算数字`1`到`n`的平均数。<br />
例子1，无分割的平均数计算，代价`0(n)`：
```js
for (let i = 0; i < n; i++)
  sum += i;
let avg = sum / n;
console.log('avg: ' + avg);
```
例子2，分割的平均数计算，每个n的异步步骤代价`0(1)`：
```js
function asyncAvg(n, avgCB) {
  // Save ongoing sum in JS closure.
  var sum = 0;
  function help(i, cb) {
    sum += i;
    if (i == n) {
      cb(sum);
      return;
    }

    // "Asynchronous recursion".
    // Schedule next operation asynchronously.
    setImmediate(help.bind(null, i+1, cb));
  }

  // Start the helper, with CB to call avgCB.
  help(1, function(sum){
      var avg = sum/n;
      avgCB(avg);
  });
}

asyncAvg(n, function(avg){
  console.log('avg of 1-n: ' + avg);
});
```
你可以把这个原则应用在数组的轮询或者之类的场景中。

分流（offloading）<br />
如果你要做一些更加复杂的事情，分区并不是一个很好的选择。这是因为分区只用了事件循环，在你的机器上，几乎可以肯定的说，你将不会从多核cpu中获益。*谨记：事件循环应该调配用户的请求，而不是去实现它们*。对于一个复杂的任务来说，就是把工作从事件循环移交到工作池中。

如何分流<br />
对于需要被分流工作的目标工作池，你有两个选择：<br />
1. 你可以通过开发一个[c++ addon](https://nodejs.org/api/addons.html "c++ addon")来使用Node内置的工作池。在旧版本的Node中，构建你自己的C++ addon用[NAN](https://github.com/nodejs/nan "NAN")，而新版本的用[N-API](https://github.com/nodejs/nan "N-API")。[node-webworker-threads](https://www.npmjs.com/package/webworker-threads "node-webworker-threads")提供了一种Javascript专享的方式来访问Node的工作池。
2. 你可以创建和管理你自己的工作池专注用于计算，而非Node的I/O专用工作池。最直接的方式就是使用[Child Process](https://nodejs.org/api/child_process.html "Child Process")或者[Cluster](https://nodejs.org/api/cluster.html "Cluster")。

你不应该简单地为每个用户都创建一个[Child Process](https://nodejs.org/api/child_process.html "Child Process")。这样的话，你就可以比创建和管理子线程更快地接收用户请求，而你的服务器则可能会成为一个[叉路炸弹](https://en.wikipedia.org/wiki/Fork_bomb "fork bomb 叉路炸弹")。

分流的缺陷<br />
分流方法的缺陷是会带来额外的*通讯开销*。只有事件循环才被允许看到应用的“命名空间”（Javascript状态）。你是不能从作业者里来操纵一个在事件循环命名空间里头的Javascript对象的。因此，你必须序列化或者反序列化任何你想要共享的对象。然后作业者就可以操作它自己的那些对象拷贝和返回修改过的对象(或者一个“补丁”）给事件循环。<br />
关于对序列化的担忧，请回去看一下JSON DOS攻击的部分。

关于分流的一些建议<br />
你可能会希望分辨出CPU密集型和I/O密集型的任务，因为他们有显著不同的特征。<br />
一个CPU密集型的任务，只会在作业者被调度的时候进行，而作业者必须基于你的机器的逻辑内核之上来被调度。如果你有4个逻辑内核和5个作业者，那么这些作业者当中的一个将不能处理。因此，你就会为这个作业者支付开销（内存或者调度成本）,却不能得到返回。<br />
I/O密集型的任务牵涉到外部的服务提供者（DNS，文件系统等等）和等待它们的返回。当一个有I/O密集型任务的作业者正在等待它的返回，它就什么也做不了，会被系统取消调度，给其他作业者提交它们请求的机会。因此，*即使相应线程没有在运行，I/O密集型的任务也会取得进展*。外部服务的提供者，像数据库和文件系统之类的，已经被高度优化来处理许多并行的等待请求。例如，一个文件系统会测试一个大型的正在等待的读写请求集合，合并存在冲突的更新，用一个经过优化的顺序来获取文件（请查看[这些内容](http://researcher.ibm.com/researcher/files/il-AVISHAY/01-block_io-v1.3.pdf)）。<br />
如果你只依赖一个工作池，例如Node的工作池，那么CPU密集型和I/O密集型的任务的不同特性可能会损害应用程序的性能。<br />
基于这个原因，你可能会希望能够维护一个独立的计算工作池。

分流：总结<br />
对于简单的任务，像轮询一个超级长的数组里面的元素，用分区可能是一个好的选择。如果你的计算较复杂，分流就是一个更好的途径：通讯开销，也就是在事件循环和工作池之间传递序列化对象的开销，被多核带来的好处抵消。<br />
然而，如果你的服务重度依赖复杂计算，你应该考虑Node是否真的很适合。Node的优势在于I/O绑定型的工作，但对于重度计算，它可能不是一个最好的选择。

### 别阻塞工作池 ###
Node的工作池由`k`作业者组成。如果你在使用上面讨论到的分流模式，你可能会有一个独立的计算工作池，同样也适用于该池（也是由`k`作业者组成）。无论如何，让我们假设`k`作业者比那些需要你并行处理的用户数小得多。这就是保持了Node的“一个线程服务于多用户”的哲学，它可扩展性强的秘密。<br />
就像我们上面谈到的，在处理工作池队列中的下一个任务之前，作业者都会完成当前的任务。<br />
现在，处理用户请求的任务开销会有所不同。有些任务会很快完成（例如：读取短的或者缓存的文件；产生一个小数量级的随机数），其他会花上更长时间（例如：读取大的或者未缓存的文件；生成很多随机数）。你的目标就应该是*最小化任务时间的变化*，你应该用*任务分区*来达成。

最小化任务时间变化<br />
如果一个作业者的当前任务比其他任务重得多，那它将无法为其他等待中的任务工作。换句话说，*每个相对较长（需要更长时间处理）的任务都会有效减少工作池的大小，直到它被处理完为止*。这是不合需求的，因为，基于这一点，工作池中的作业者越多，工作池的吞吐量（任务数/秒）就越大，因此服务器的吞吐量（请求数/秒）就越大。一个拥有相对重度任务的用户会降低工作池的吞吐量，因而降低服务器的吞吐量。<br />
两个例子应该可以说明任务时间上可能的变化。

变化的例子：长时间运行的文件系统读取<br />
假设你的服务器必须读取一些文件来处理客户请求。在请求完Node的[文件系统](https://nodejs.org/api/fs.html "File System")APIs后，为了简单起见，你选择使用`file.readFile(）`。然而，`file.readFile(）`（[目前](https://github.com/nodejs/node/pull/17054)）并不是已分区的：它提交了一个跨越了整个文件的`fs.read()`任务。如果你为一些用户读取较短的文件，为其他用户读取较长的文件，`file.readFile(）`就可能会引入任务长度上明显的变化，从而来决定工作池的吞吐量。<br />
对于最糟糕的场景，假设一个攻击者会说服你的服务器读取一个任意文件（这就是[目录遍历漏洞](https://www.owasp.org/index.php/Path_Traversal "directory traversal vulnerability")）。如果你的服务器正在运行Linux，攻击者会命名一个特别慢的文件[/dev/random](http://man7.org/linux/man-pages/man4/random.4.html)。在实际应用中， `/dev/random`是无限慢的，每一个作业者请求从`/dev/random`的读操作将无法完成。攻击者然后就会提交`k`请求，每个作业者一个，这样就没有一个使用这个工作池的用户请求能够取得进展。

变化的例子：长时间运行的加密操作<br />
假设你的服务器正在使用[crypto.randomBytes()](https://nodejs.org/api/crypto.html#crypto_crypto_randombytes_size_callback)来生成加密安全的随机字节。`crypto.randomBytes()`不是分区的：它创建了一个`randomBytes()`任务来生成跟你请求的一样长的字节。如果你给一些用户创建较少字节，给其他用户创建较多字节，`crypto.randomBytes()`就成了另一个任务长度的变化。

任务分区<br />
时间成本不定的任务会伤害工作池的吞吐量。为了最小化任务时间的不确定性，尽可能将每个任务*划分（partition）*为相当成本的子任务。当每一个子任务完成的时候它应该提交下一个子任务，当最后一个子任务完成后，它应该通知任务提交者。<br />
继续回到`fs.readFile()`的例子，你应该使用`fs.read()`（手动分区）或者`ReadStream`（自动分区）来代替。<br />
同样的原理应用在重CPU的任务上：`asyncAvg`例子，可能并不太适合事件循环，可是它却非常适合工作池。<br />
当你把任务划分为子任务，较短的任务化为少数的子任务，较长的任务化为较多数量的子任务。在长任务的每个子任务之间，被分配的作业者可以工作在其他更短的任务的子任务上，这样就可以改进工作池的整体任务吞吐量。<br />
注意，子任务完成的数量不能成为工作池吞吐量的有效衡量标准。相反，应该关注完成的任务数量。

任务分区的避免<br />
回忆一下，任务分区的目的是最小化任务时间的不确定性。如果你可以分辨短任务和长任务（例如数组求和跟数组排序）。你就可以为每一类任务创建一个任务池。分流短任务和长任务到不同的工作池，是另一种最小化任务时间不确定性的方法。<br />
支持这种说法的是，任务分区会带来间接开销（创建一个工作池任务的表达和维护工作池队列的成本），而避免分区可以节省达成工作池的额外成本。它也可以让你避免在任务划分里犯错。<br />
这个方法的负面影响是这些工作池的作业者会造成空间和时间的开销，彼此会竞争CPU时间。记住每个重CPU的任务仅当它获得排期的时候才能取得进展。基于此，你应该在详细分析后才可以考虑这种方法。

### NPM模块的风险 ### 
当Node核心模块为广泛的应用程序提供构建块的时候，有时候需要更多的东西。Node开发人员从[npm生态系统](https://www.npmjs.com/)中获益良多，有成千上万的模块提供了加快开发过程的功能。<br />
无论如何，请记住，这些模块中的多数都是由第三方开发者编写的，并且之提供最大努力的保证。一个开发者使用一个NPM模块应该关注两件事情，尽管后者经常被遗忘：
1. 它是否忠于它的APIs？
2. 它的APIs是否会阻塞事件循环或者作业者？许多APIs不会努力地指出它们的成本，从而损害了社区的利益。

对于简单的APIs，你可以估算出它的API成本。字符串操作的成本不会太难被看穿。但是许多情况下，API的成本是多少并不清晰。<br />
*如果你要调用一个可能会有繁重任务的API，请仔细检查它的成本。请开发者编写文档或者自己查看源代码（然后提交PR来记录它的成本）*。<br />
请记住，即使API是异步的，你不知道它会在它的每个分区里的作业者或者事件循环中花多少时间。例如，假如在上面给出的`asyncAvg`例子中，每一次调用`helper`函数，都会对其中一半数字求和，而不是对其中一个数字求和。然后这个函数仍然是异步的，但每一次划分的代价都是`0(n)`而不是`0(1)`，对于任意数值`n`，使它更不安全。

### 结论 ###
Node有两种类型的线程：事件循环和`k`作业者。事件循环负责Javascript回调和非阻塞性I/O，而一个作业者执行与完成异步请求的C++代码对应的任务，包括阻塞I/O和CPU密集型工作。这两种线程的工作一次只有一个是活动的。如果任何回调或者任务花了很长时间，那么线程运行就变成被阻塞了。如果你的应用程序使回调或者任务阻塞，好一点的情况只会降低吞吐量（任务数/秒），最坏的情况就是完全拒绝服务。<br />
为了编写高吞吐量、更能防DOS的web服务，你就必须保证不管在正常或者恶意输入的情况下，你的事件循环或者作业者都不会被阻塞。