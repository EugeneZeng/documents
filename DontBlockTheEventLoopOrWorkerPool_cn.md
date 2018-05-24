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