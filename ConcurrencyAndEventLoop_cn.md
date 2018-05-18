# Javascript并发模型和事件循环 #
## 原文：《[Concurrency model and Event Loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop "Concurrency model and Event Loop")》 #

### 运行时概念 ###
接下来的部分解释一种理论模型。现代Javascript引擎实现和极大优化了描述性语义。

示意图（Visual Representation）<br />
![Visual Representation](/img/VisualRepresentation.png "示意图（Visual Representation）")

栈<br />
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

===未完待续
