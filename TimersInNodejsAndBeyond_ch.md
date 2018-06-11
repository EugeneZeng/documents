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

如果有一个代码块需要被多次执行，`setInterval()`就可以用来执行这个代码。`setInterval()`接收一个方法作为参数，将用作为第二参数的毫秒数延时间隔执行无限次数。像`setTimeout()`一样，随后的其他参数可以添加在延时值的后面，将被传入给第一个方法运行时候作为参数使用。同样地也像`setTimeout()`一样，这些延时不能被保证，因为操作有可能会中止事件循环，因此它应该被视为是一个大约的延时。请看下面的例子：

```js
function intervalFunc() {
  console.log('Cant stop me now!');
}

setInterval(intervalFunc, 1500);
```

在上面的例子中，`intervalFunc()`将会在每大约1500毫秒（1.5秒）后执行一次，直到它被终止（下面会提到）。

就像`setTimeout()`一样，`setInterval()`也返回一个`Timeout`对象，可以用来指向和修改这个已经被设置好的Interval。

### 未来清除 ###

如果一个`Timeout`或者`Immediate`对象需要被取消，需要做什么呢？`setTimeout()`，`setImmediate()`和 `setInterval()`返回了一个定时器对象，用来指向设置好的`Timeout`或者`Immediate`对象。通过传递这个对象给相应的`clear`函数，这个对象的执行将会被完全停止。这些相应的方法是`clearTimeout()`，`clearImmediate()`和`clearInterval()`。请看下面的例子：

```js
const timeoutObj = setTimeout(() => {
  console.log('timeout beyond time');
}, 1500);

const immediateObj = setImmediate(() => {
  console.log('immediately executing immediate');
});

const intervalObj = setInterval(() => {
  console.log('interviewing the interval');
}, 500);

clearTimeout(timeoutObj);
clearImmediate(immediateObj);
clearInterval(intervalObj);
```

### 留下超时 ###

请记住`Timeout`对象是由`setTimeout`和`setInterval`方法返回的。`Timeout`对象提供了两个方法，想要用`unref()`和 `ref()`方法来增强`Timeout`的行为。如果有一个`Timeout`对象用一个`set`方法来预订，`unref()`就可以被这个对象调用。这将会略微改变这个行为，而不会调用这个`Timeout`对象，_如果它是需要运行的最终代码_。`Timeout`对象将不会保持运行，等待运行。

类似的方式，一个在上面执行了`unref()`方法的`Timeout`对象可以通过在同一个对象上调用`ref()`方法来移除那个行为，这样就会保证它的运行。然而，由于性能原因，这并不能完全恢复初始行为。请看以下两个例子：

```js
const timerObj = setTimeout(() => {
  console.log('will i run?');
});

// if left alone, this statement will keep the above
// timeout from running, since the timeout will be the only
// thing keeping the program from exiting
timerObj.unref();

// we can bring it back to life by calling ref() inside
// an immediate
setImmediate(() => {
  timerObj.ref();
});
```

### 事件循环之后 ###

事件循环和定时器比本文档提到的内容要多得多。要了解更多关于Node.js的事件循环内幕以及定时器在运行的时候怎么操作，请阅读Node.js的这个指南：[NodeJS的事件循环，定时器和process.nextTick()](/NodejsEventLoopTimerAndProcessNextTick_cn.md "NodeJS的事件循环，定时器和process.nextTick()")