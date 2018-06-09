# 弄懂Javascipt的延展操作符（Spread Operator） #

## 原文：（[Understanding the Spread Operator in JavaScript](https://zendev.com/2018/05/09/understanding-spread-operator-in-javascript.html "Understanding the Spread Operator in JavaScript")） ##

较新版本的Javascript在术语表达和开发便利性方面已经为语言带来了大量的改进，但快速的变化让许多开发人员正在苦于跟上节奏。

随着Wordpress在新版的Gutenberg编辑器中拥抱了React和现代Javascript，Wordpress开发者的庞大受众被带到了这个世界，不管你喜欢与否，他们都在迅速赶上。

在这个文章中，我们将详解Javascript语言新功能当中最流行的一个——延展操作符（也就是 `...` 操作符）。

有个朋友最近向我求助，想弄懂一些来自Gutenberg块类库中的代码实例，特别是gallary组件。直到写这篇文章的时候，代码还可以在[这里](https://github.com/WordPress/gutenberg/blob/823dcf2ac446d21d76bf43a797347ce5a33ece3d/core-blocks/gallery/edit.js)看到，但它已经被多次移动过了，所以我把代码重现到这里：

```js
setImageAttributes( index, attributes ) {
  const { attributes: { images }, setAttributes } = this.props;
  if ( ! images[ index ] ) {
    return;
  }
  setAttributes( {
    images: [
      ...images.slice( 0, index ),
      {
        ...images[ index ],
        ...attributes,
      },
      ...images.slice( index + 1 ),
    ],
  } );
}
```

尤其令人疑惑的部分在于：

```js
images: [
    ...images.slice( 0, index ),
    {
        ...images[ index ],
        ...attributes,
    },
    ...images.slice( index + 1 ),
],
```

这当然看上去有一点点吓人，特别是你最近没有花很多时间在现代Javascript的编程上。让我们详细看看代码里面发生了什么。

### Array的延展操作符 ###

要知道的核心语法是`...`语法。这就是延展操作符，它本质上是取一个数组或对象，将其展开为它的一个项的集合。这样就可以让你做一些很奇特的事情，举个例子，如果你有以下代码：

```js
const array = [1, 2];
const array2 = [...array, 3, 4];
```

array2的值最终变成了`[1, 2, 3, 4]`。

延展操作符让你将一个数组放入，并获得它的值的集合。

回到我们例子中的代码，在最外层我们有：

```js
images = [...images.slice(0, index), {some stuff}, ...images.slice(index+1)]
```

这就是在说：把images数组设置成为旧images数组的0到index的元素，然后是我们稍后要介绍的新东西，接着就是旧images数组index+1到结尾的元素。

换句话说，我们是要替换掉位于index的数组项。

### 对象的延展操作符 ###

接下来，关于对象，延展语法让你做一些与Object.assign同样的事情，把一个对象的值拷贝进一个新的对象里。来看个简单的代码示例：

```js
const obj1 = {a: 'a', b: 'b'};
const obj2 = {c: 'c', ...obj1};
```

`obj2`的结果就成了`{a: 'a', b: 'b', c: 'c'}`。

我们回到Gutenberg的代码例子，里面一层（在数组的例子里标记为`{some stuff}`的部分），我们有：

```js
{
  ...images[ index ],
  ...attributes,
}
```

翻译过来：创建一个对象，首先用来自`images[index]`的值来填充，然后用来自`attributes`的值填充。任何重复的值将会被后面的一个覆盖。

这就是在说：从`index`的位置拿走我的旧图片项，然后用我在`attributes`里的任意值来替换，`attributes`里的值优先。

回到我们整个的代码示例：

```js
images: [
    ...images.slice( 0, index ),
    {
        ...images[ index ],
        ...attributes,
    },
    ...images.slice( index + 1 ),
],
```

整个奇特的事情是在说：我有一个images数组，一个index，和一个需要被应用上的attributes的集合。返回一个把index位置的数组项改为我那的attributes的新images数组。

### 延展语法使简洁和表述性的代码成为可能 ###

让我们看看我们实现了什么。简短来说，希望现在这种可读性的语句，我们已经创建了一个数组的新副本，该数组在一个特定的index上，有一个更新的，复杂的对象。我们没有修改原始的数组，意味着我们代码中的其他部分还可以调用它，无需考虑边界效应。完美！