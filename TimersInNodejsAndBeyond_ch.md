# Node.js里面及外面的定时器 #

## 原文：《[Timers in Node.js and beyond](https://nodejs.org/en/docs/guides/timers-in-node/ "Timers in Node.js and beyond")》 ##

在Node.js里的定时器模块包含了一段时间后执行代码的功能。定时器不需要通过`require()`来导入，因为所有模拟浏览器javascript API的方法都是全局有效的。为了完全弄懂定时器函数在何时执行，阅读Node.js[事件循环](//NodejsEventLoopTimerAndProcessNextTick_cn.md "NodeJS的事件循环，定时器和process.nextTick()")是个很好的办法。

### 用Node.js控制计时的连续性 ###

Node.js的API提供了多种在当前时刻之后执行代码的方法。以下的这些函数可能会看上去很相似，因为它们存在于大部分浏览器里，但Node.js实际上提供了它自己的一套对这些方法的实现。定时器与系统整合得非常紧密，尽管这些API模拟了浏览器的API，在实现上有一些不同。

### “当我说这样的时候”运行 ～ _`setTimeout()`_ ###

`setTimeout()`可以被用来预约在指定数量的毫秒数后的代码执行。这个方法与来自于浏览器Javascript API的`window.setTimeout()`类似，虽然它不能执行字符串形式传入的代码片段。

`setTimeout()`接收一个需要被执行的函数作为第一个参数，定义成数字的毫秒延时作为第二个参数。其他参数也可以传入，那就会被传给这个函数。以下是一个例子：

```js
function myFunc(arg) {
  console.log(`arg was => ${arg}`);
}

setTimeout(myFunc, 1500, 'funky');
```