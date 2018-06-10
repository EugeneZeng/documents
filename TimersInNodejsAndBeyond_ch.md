# Node.js里面及外面的定时器 #

## 原文：《[Timers in Node.js and beyond](https://nodejs.org/en/docs/guides/timers-in-node/ "Timers in Node.js and beyond")》 ##

在Node.js里的定时器模块包含了一段时间后执行代码的功能。定时器不需要通过`require()`来导入，因为所有模拟浏览器javascript API的方法都是全局有效的。为了完全弄懂定时器函数在何时执行，阅读Node.js[事件循环](/NodejsEventLoopTimerAndProcessNextTick_cn.md "NodeJS的事件循环，定时器和process.nextTick()")是个很好的办法。

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

以上的函数`myFunc()`会尽可能接近1500毫秒（1.5秒）的时候执行，因为它是通过`setTimeout()`来调用的。

延时的间隔不可信赖，只因它不会在_精确_的毫秒数后马上执行指定的方法。这是因为阻塞或者暂停了事件循环的其他代码运行会使现在的这个延时的执行推后。_唯一_的保证是函数不会执行得比指定的延时间隔更快。

`setTimeout()`返回了一个`Timeout`对象，可以用来指向这个已经设置好的延时。这个返回的对象可以用来取消延时（看看下面的`clearTimeout()`），也可以用来改变运行行为（看看下面的`unref()`）。

### "就在这之后"执行 ～ `setImmediate()` ###

`setImmediate()`将在当前事件循环的末端执行代码。这些代码将会在当前事件循环的任何I/O操作*之后*和下一个事件循环的任何定时器被预约*之前*执行。这些代码的执行会被认为是发生在“就在这之后”，意思就是任何跟随在`setImmediate()`函数调用后面的代码都将先于`setImmediate()`里的函数参数执行。

`setImmediate()`的第一个参数就是即将被执行的函数。随后的所有参数都会在这个函数被执行的时候被传入到这个函数里。这是个例子：

```js
console.log('before immediate');

setImmediate((arg) => {
  console.log(`executing immediate: ${arg}`);
}, 'so immediate');

console.log('after immediate');
```

在上面例子中，传给`setImmediate()`的函数将在其他可运行的代码都执行完成后执行，console将会打印出以下内容：

```text
before immediate
after immediate
executing immediate: so immediate
```

`setImmediate()`函数返回了一个`Immediate`对象，可以用来取消已经预订的immediate（看下面的`clearImmediate()`）。

请注意：别把`setImmediate()`跟`process.nextTick()`混淆了。它们有一些主要的不同之处。第一点就是`process.nextTick()`将会运行在任何设置好的`Immediate`之前，也会在任何安排好的I/O之前。第二点是`process.nextTick()`不能被清除，意思就是一旦代码被预订了用`process.nextTick()`来执行，那么这个执行就不能被终止，就像一个普通的方法运行。参考[这个文档](//NodejsEventLoopTimerAndProcessNextTick_cn.md "NodeJS的事件循环，定时器和process.nextTick()")可以更好地理解`process.nextTick()`的操作。

### “无限循环”执行 ～ `setInterval()` ###

