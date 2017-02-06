# 响应式之路

在现实世界中，事件发生的随机性、应用程序的冲突以及网络错误等情况会导致一切都一团糟，几乎没有应用程序是完全同步的，为了保持程序的高响应异步代码必不可少，程序开发充斥着痛苦，然而我们并不比如此。

现代的应用程序不仅需要超快的响应能力，同时还要具备准确处理不同数据来源中数据流的能力。然而当前的技术没有提供这种能力，因为当我们增加了并发和应用程序各种状态转换再来处理数据流会让代码复杂度指数级增加。如果硬要这样解决会大大增加开发者的精神负担，从而导致代码越来越复杂和层出不穷的bug。

本章会向你介绍一种更自然更简单的异步代码解决方案——响应式编程。我会向你展示事件流（我们称之为被观察者）是怎样优雅的处理异步代码的，然后再创建一个观察者来告诉我们什么事响应式思维。RxJS可以显著的改进现有的技术，你将发现你的开发会更快乐更有效率。

## 什么是响应式？

我们先从一个点击不同按钮处理不同数据源的小程序来看一看用RxJS是如何响应式编程的。这个小程序首先要满足下列的条件：

* 数据必须来自于两个不同的地址，并且拥有不同的JSON结构
* 最终结果不需要关心是否重复
* 为防止过多请求，要求每秒钟只能处理一次用户的点击行为

用RxJS写出的代码就是这样的：

```javascript
var button = document.getElementById('retrieveDataBtn');
var source1 = Rx.DOM.getJSON('/resource1').pluck('name');
var source2 = Rx.DOM.getJSON('/resource2').pluck('props', 'name');
function getResults(amount) { return source1.merge(source2)
.pluck('names')
.flatMap(function(array) { return Rx.Observable.from(array); }) .distinct()
.take(amount);
}
var clicks = Rx.Observable.fromEvent(button, 'click'); clicks.debounce(1000)
  .flatMap(getResults(5))
  .subscribe(
function(value) { console.log('Received value', value); }, function(err) { console.error(err); },
function() { console.log('All values retrieved!'); }
);
```
先不用担心看不懂这些代码，大致看下代码结构，首先你会发现使用了被观察者的模式，我们用寥寥几行代码就实现了所有的处理逻辑。

每个被观察者都代表一个数据源，程序可以很大程度上表示为数据流。在上面的例子中远端的接口，用户鼠标的点击都是被观察者。事实上，在我们的代码中只有点击按钮获取想要结果的事件是必须的被观察者。

响应式编程是十分高效的，就拿我们例子中的限制鼠标点击次数来说，你可以想象如果我们用回调函数或promise来实现是怎么样的：我们需要首先需要一个每秒重置的计时器，另外还要设置一个状态表示过去的一秒钟用户有没有点击按钮。对于这么小的一个功能这么做就显得过于复杂了，而且这些代码甚至不与你程序的实际功能有关联。在更大的应用程序中，这些小的复杂功能叠加起来很快就会把基础代码搞的一团糟。

在响应式编程中，我们使用反射机制来对点击限流，这确保了两次点击之间至少间隔1秒，多余的点击都被忽略。其实我们不需要关心具体如何实现的，只需要知道想要我们的代码做什么就可以了。

更有趣的是我们接下来就看看响应式编程如何帮助我们的程序更加高效易懂。

## 灵活的电子表格

我们先以电子表格这个典型例子开始研究响应式系统。电子表格我们都使用过，但是很少有人停下来想想它们是多么惊人的直观(how shockingly intuitive they are)。例如我们在电子表格的A1格内设定一个值，并将它关联到其他单元格，这样每当我们改动A1的值，其他关联的单元格中的值也都自动更新了。

![spreadsheet](http://ojo0vdkcl.bkt.clouddn.com/spreadsheet.jpeg)

这样的行为让人感觉非常自然，我们不用告诉电脑去更新A1关联的其他单元格的值，也不用告诉电脑怎样做，这些单元格跟A1的变化是关联的。在电子表格中我们只需定义要处理的问题就行了，完全不用关心电脑是如何计算这些结果的。

这就是响应式编程想要实现的，我们定义成员之间的关联，程序随着这些成员的改变去计算更新后的值。

## 将鼠标操作看成数据流

回顾一下本章开始的例子，我们将事件理解了成数据流。在例子中用户实时点击操作我们用一组循环的事件序列去处理，这种思路是RxJS开创者Erik Meijer在《Your Mouse Is a Database》一文中提出的。

在响应式编程中我们将鼠标点击看作是一系列连续的事件流，而这个事件流是可以查询喝操作的。用事件流代替孤立的值给我们编程打开了一条全新的道路，这种方式中我们可以操作整个数值序列，但实际上它并没有被创建。

静下心来想一想，以往的做法中，我们要将这些值先存到一个地方（数据库或数组），需要的时候再从里面取。但是如果它们不能马上获取到，比如要用过网络请求获取的时候，那么我们只能等待获取到它们之后再继续了。

![](http://ojo0vdkcl.bkt.clouddn.com/WechatIMG1.jpeg)

我们将这一序列想像成了一个数组，只不过这个数组中的值是用时间维度划分而不是内存维度。无论按时间划分还是按内存划分，我们都将得到一序列的元素如下：

![](http://ojo0vdkcl.bkt.clouddn.com/WechatIMG2.jpeg)

将程序看作这么一序列的数据是理解RxJS编程的关键，这需要一些练习但并不困难。事实上我们应用程序处理的绝大多数数据都可以用序列来表达，在本书17页，第二章“深入序列”中我们会学习更多关于序列的概念。

## 查询序列

举一个用javascript传统事件监听处理鼠标事件流的简单例子：每当鼠标点击的时候打印出鼠标的x和y坐标。通常我们会这么写：

```
document.body.addEventListener('mousemove', function(e) { console.log(e.clientX, e.clientY);
});
```
这段代码在每次鼠标点击的时候会打印出x、y坐标如下：
```
252 183 
211 232 
153 323
...
```
这看起来不就是一个序列吗？唯一的问题就是操作事件比操作数组麻烦多了，例如：将上面的例子改成只打印屏幕右侧的前10次点击（请原谅有点儿随意…），那么代码会改成下面这样。
```
var clicks = 0;
document.addEventListener('click', function registerClicks(e) {
	if (clicks < 10) {
		if (e.clientX > window.innerWidth / 2) {
			  console.log(e.clientX, e.clientY);
				clicks += 1; 
		}
	} else {
		document.removeEventListener('click', registerClicks);
	}
});
```
为了满足功能，我们不得不定义一个全局变量click来记录一直以来的点击次数；为了判断两种不同的条件，我们又不得不用双重条件判断。而且当打印结束的时候我们还要释放变量和事件注销以防止内存泄漏。

### 副作用和外部状态

当一个功能触发的时候会对作用域之外的部分造成影响，我们称之为副作用。改变函数外的变量、console函数中打印或者更新数据库中的一个值都是副作用的典型例子。

举个例子，改变我们函数内的变量是安全的，但是如果这个变量在我们的函数中，别的函数却可以修改它，那么这就意味着我们失去了对这个变量的控制，甚至我们都没有办法保证它的值是否还是我们期望的。因此我们必须增加与函数完全不相关的代码逻辑，平白增加了代码的复杂度和出错率。

在构建任何有趣的程序时，有时候不得不加入这种副作用，但是我们应该努力减少这种代码，这在响应式编程中非常重要，因为响应式编程中有很多随时间变化的部分。在本书中，我们将寻求一种避免外部状态和副作用的方法，事实上，在第三章“构建并发程序”中我们将构建一个完全没有副作用的视频游戏。

为了实现一个简单的功能，我们最终写了一段复杂的代码，这对第一次接触它的开发人员来说是很难维护的。更重要的是因为我们需要保持状态，所以未来开发的时候很容易产生隐藏的bug。

在这种情况下我们想要的是查询点击的“数据库”。如果使用关系型数据库，就需要使用声明式语言SQL：
```
SELECT x, y FROM clicks LIMIT 10
```
如果我们将该点击事件流视为可以查询和转换的数据源，该怎么办呢？其实它与数据库的唯一区别就是它是实时发送的，其他并没有什么不同。我们所需要的也就是一个抽象概念的数据类型。

使用RxJS和它的被观察者数据类型如下：
```

Rx.Observable.fromEvent(document, 'click')
	.filter(function(c) { return c.clientX > window.innerWidth / 2; })
	.take(10)
	.subscribe(function(c) { console.log(c.clientX, c.clientY) })
```
这段代码完成的功能跟第四页的代码一样，可以这样理解：

	创建点击事件的被观察者，并过滤出屏幕左侧发生的点击次数，然后只有前10次点击发生的时候在控制台打印出坐标。	
就算你对RxJS不熟悉也能轻松读懂这段代码，而且你再也不用创建独立于业务代码之外的变量来保持状态，这也就更难引入bug。释放变量的问题也不复存在，不用担心忘记注销事件带来的内存泄漏，是不是很完美？

在上面的代码中我们用一个DOM事件创建了一个被观察者，这个被观察者给我们提供了一个可以作用整体操作的事件序列或流，而不是往常的每次一个单独的事件。处理序列给我们带来巨大的力量；我们可以轻松的合并，转换或传递被观察者。我们将无法处理的事件转变为有型的数据结构，就像操作数组一样简单，但更灵活。

在下一节中，我哦们将看到使Observables成为一个伟大工具的原理。

## 观察者和迭代器

要理解被观察者怎么来的，我们先要学点基础的东西：观察者模式和迭代器模式。在本节中我们快速了解一下这两种设计模式，然后再看看Observables如何以简单而强大的方式组合两者的概念。

### 观察者模式

### 迭代器模式

## Rx模式和被观察者

## 创建被观察者

### First Contact with Observers

### 将Ajax请求封装成被观察者

### 总有一个控制器

### 用数组创建被观察者

### 用javascript事件创建被观察者

### 用回调函数创建被观察者

## 总结
在本章中，我们探讨了响应式编程的方法，并了解了RxJS如何通过被观察者来解决其他方案的问题，例如回调函数或者promise。现在你明白被观察者这种模式为什么如此强大，并且知道如何创建它们。有了这个基础，我们现在可以继续创建更有趣的响应式程序了。下一章将向您展示如何在web开发中一些常见场景应用被观察者这种模式，创建和编写基于序列的程序。