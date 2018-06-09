# Node.js里面及外面的定时器 #

## 原文：《[Timers in Node.js and beyond](https://nodejs.org/en/docs/guides/timers-in-node/ "Timers in Node.js and beyond")》 ##

在Node.js里的定时器模块包含了一段时间后执行代码的功能。定时器不需要通过`require()`来导入，因为所有模拟浏览器javascript API的方法都是全局有效的。为了完全弄懂定时器函数在何时执行，阅读Node.js[事件循环](//NodejsEventLoopTimerAndProcessNextTick_cn.md "NodeJS的事件循环，定时器和process.nextTick()")是个很好的办法。

