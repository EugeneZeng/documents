# Javascript并发模型和事件循环 #
## 原文：《[Concurrency model and Event Loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop "Concurrency model and Event Loop")》 #

### 运行时概念 ###
接下来的部分解释一种理论模型。现代Javascript引擎实现和极大优化了描述性语义。

示意图（Visual Representation）<br />
![Visual Representation](/img/VisualRepresentation.png "示意图（Visual Representation）")

栈（Stack）<br />
从帧的栈里调用函数

```js
function foo(b) {
  var a = 10;
  return a + b + 11;
}

function bar(x) {
  var y = 3;
  return foo(x * y);
}

console.log(bar(7)); //returns 42
```
当调用`bar`的时候，包含`bar`的参数和本地变量的第一个帧就被创建了。
当`bar`调用`foo`的时候，第二个帧就被创建了并且被放到了第一个帧的上方，同样包含了`foo`的参数和本地变量。
当`foo`return的时候，顶端的帧元素就从栈中被取出（这时只剩下`bar`在栈里了）。
当`bar`return的时候，整个栈就空了。

堆（Heap）<br />
对象被放到一个堆里面，也就是一个用名字来表示的一大块多数非结构化的内存区域（a large mostly unstructured region of memory）。

队列（Queue）<br />
一个Javascript运行时使用消息队列，也就是一个就要被处理的消息列表。每个消息都有一个对应的函数被调用，以便处理消息。<br />
某些时候，在事件循环中，运行时在队列中开始处理消息，从最旧的那个开始。为此，消息被从队列中移除，而对应的函数被以消息作为输入参数的方式调用。
同样地，调用一个函数就会创建一个新的栈帧给函数使用。<br />
函数的处理会一直持续到栈又一次清空。然后事件循环就会继续处理下一个队列中的消息（如果有的话）。

### 事件循环 - Event Loop ###

事件循环因它的通常实现而得名，通常长这个样子：

```js
while (queue.waitForMessage()) {
  queue.processNextMessage();
}
```

`queue.waitForMessage()`同步等待一个消息的到来，如果目前没有的话。

执行到完为止（Run-to-Completion）<br />
在处理其他消息之前，每一个消息都会被完全处理。这就在解析（reasoning）你的程序的时候，提供了一些有趣的特性，包括，无论函数在何时运行，它都不能被捷足先登，而将会在其他代码运行之前运行（还能修改它自己维护的一套数据）。这点不同于C，举个例子来说，一个函数在一个线程里运行，它可以在任何点上被其他线程的其他代码中止。<br />
这个模型的负面影响在于，如果完成一个消息花费了太长时间，应用程序就无法处理其他的用户交互，例如点击或者滚动。浏览器就会弹出一个写着“a script is taking too long to run”的对话框。比较好的做法是使一个消息尽可能地短或者如果可能的话，就把一个消息分割成几个消息来处理。

增加消息（Adding Messages）<br />
在浏览器里，消息会被添加一旦事件发生了，同时还会附带一个事件监听器。如果没有监听器，事件就会被丢弃。所以，点击在一个绑定了点击事件处理器的元素上，将会增加一个消息。同样地，其他事件也是如此。<br />
调用函数setTimeout需要两个参数：加到队列里的消息和一个时间值（可选，默认为0）。这个时间值表示一个（最小）延时后，消息才会被加入到队列里面。
如果队列中没有其他消息，这个消息就会在延时后被立即处理。然而，如果队列中有消息，setTimeout的消息将需要等待处理完队列中所有消息。由于这个原因，第二个参数只能表示最短时间，而并不是一个确切时间。<br />
下面这个例子将演示这个概念（setTimeout不会在时间到的时候马上执行）
```js
const s = new Date().getSeconds();

setTimeout(function() {
  // prints out "2", meaning that the callback is not called immediately after 500 milliseconds.
  console.log("Ran after " + (new Date().getSeconds() - s) + " seconds");
}, 500);

while(true) {
  if(new Date().getSeconds() - s >= 2) {
    console.log("Good, looped for 2 seconds");
    break;
  }
}
```

零延时（Zero Delay）<br />
零延时实际上并不是指回调会在0毫秒后被执行。执行0毫秒延时的`setTimeout`并不会在指定时间（0毫秒）后调用它的回调函数。<br />
这种情况要看队列中等待执行的任务数量。在下面的例子中，回调中的消息被处理之前，消息“this is just a message”将会先显示在console中，因为延时只是一个最小时间值来要求运行时处理这个请求，而并不是一个确切的时间。<br />
基本上，`setTimeout`必须等待所有的代码都完成，尽管你为你的`setTimeout`指定了一个特定的时间限制。
```js
(function() {

  console.log('this is the start');

  setTimeout(function cb() {
    console.log('this is a msg from call back');
  });

  console.log('this is just a message');

  setTimeout(function cb1() {
    console.log('this is a msg from call back1');
  }, 0);

  console.log('this is the end');

})();

// "this is the start"
// "this is just a message"
// "this is the end"
// note that function return, which is undefined, happens here 
// "this is a msg from call back"
// "this is a msg from call back1"
```

多个运行时一起通讯（Several runtimes communicating together）<br />
一个web worker或者是一个跨源（cross-origin）的`iframe`拥有它们自己的栈，堆和消息队列。两个不同的运行时只能通过`postMessage`方法发送消息来进行通讯。这个方法会添加一个消息到其他运行时，如果后者监听了`message`事件。

### 永不阻塞 ###
事件循环模型一个非常有趣的特性就是，Javascript不像许多其他语言一样，它永不阻塞。处理IO是典型的基于事件和回调来做的。所以当应用程序正在等待一个`indexDB`的查询返回或者是一个XHR请求的返回，它仍然可以处理其他事情比如用户输入。<br />
遗留问题也存在，例如`alert`或者是同步的XHR。但是避免使用它们已经被认为是一种好的实践。需要注意的是，[凡事总有例外（exceptions to the exception do exist）](http://stackoverflow.com/questions/2734025/is-javascript-guaranteed-to-be-single-threaded/2734311#2734311)（但通常是由于引入了Bug，而非其他问题）。

===未完待续
