---
title: 编程原理
date: 2015-02-13 15:16:24
tags: 编程
---

## [编程原理](http://webpro.github.io/programming-principles/)

每个程序猿如果理解编程原理和模式的话，那将会很受权益。这个概论是我自己的理解。它可能会对你在设计，讨论或者复习会有所帮助。请注意，它还远远未完成，你需要经常质疑这些原理。  

这份列表是受[The Principles of Good Programming](http://www.artima.com/weblogs/viewpost.jsp?thread=331531)启发。我认为这份列表很完全，但无法完全与我个人的想法匹配。另外，我需要更详细的资源。如果你有任何建议或者意见，请联系我。

<!--more-->
### 内容
#### 通用
* [Kiss原则(既简单又傻)](####KISS)
* [YAGNI](####YAGNI)
* [如果能工作的话，就采用最简单的方式]()
* [Keep Things Dry]()
* [Code For The Maintainer]()
* [Avoid Premature Optimization]()
* [Boy-Sout Rule]()

#### 内部模块/类之间关系
* [最小耦合](####最小耦合)
* [Law of Demeter]()
* [Composition Over Inheritance]()
* [Orthogonality]()

#### 模块/类
* [Maximise Cohesion]()
* [Liskov Subtitution Principle]()
* [Open/Closed Principle]()
* [Single REsponsibility Principle]()
* [Hide Implementation Details]()
* [Curly's Law]()
* [Encapsulate What Changes]()
* [Interfae Segregation Principle]()

### KISS
如果保持简单而不是复杂，大多数系统能够运行在最佳状态。

##### 为什么
* 花更少的时间来写代码，产生更少的bug,更容易修改。
* 大道至简
* 只有当什么东西也无法被删除才是完美

###### 资源
* [KISS原则](http://en.wikipedia.org/wiki/KISS_principle)
* [Keep It Simple Stupid(KISS)](http://principles-wiki.net/principles:keep_it_simple_stupid)

### YAGNI
YAGNI表示你不再需要它了:不再增加什么了，除非它真的需要。
##### 为什么
* 如果是为了将来的功能而添加代码，那么会减少当前版本的开发时间
* 这将导致代码臃肿；软件变得庞大而复杂。

##### 怎么做
* 只实现那些你真正需要的东西。

##### 资源
* [You Arent Gonna Need It](http://c2.com/xp/YouArentGonnaNeedIt.html)
* [You're NOT gonna need it !](http://www.xprogramming.com/Practices/PracNotNeed.html)
* [You ain't gonna need it](http://en.wikipedia.org/wiki/You_ain't_gonna_need_it)

### 如果能够工作，那么用最简单的东西
##### 为什么
* 如果我们研究问题究竟是怎样的，那么进度与问题的复杂度成正比

##### 怎么做
* 扪心自问: "能做到这个最简单的方式"

##### 资源
* [Do The Simplest Thing That Could Possibly Work](http://c2.com/xp/DoTheSimplestThingThatCouldPossiblyWork.html)

### 避免重复
在整个软件系统中，每个知识点都必须有单一的，清楚地，权威地呈现。
代码复用，重要的功能仅在一处实现，别处来调用。将相似功能抽出,使用抽象的方式来实现功能的多样性。
##### 为什么
* 重复可能导致维护很艰难，难以重构以及逻辑矛盾
* 修改系统中的任何单一元素，不需要修改其他相关逻辑
* 另外，逻辑上相关的元素都是统一的，可以预测的，保持同步

##### 怎么做
* 将业务规则，长的表达式，例如声明，数学方程式，元数据等仅仅放在一个地方
* 定义单一，明确的代码，使用这些代码来生成程序实例

##### 资源
* [Dont Repeat Yourself](http://c2.com/cgi/wiki?DontRepeatYourself)
* [Don't repeat yourself](http://en.wikipedia.org/wiki/Don't_repeat_yourself)
* [Don't Repeat Yourself](http://programmer.97things.oreilly.com/wiki/index.php/Don't_Repeat_Yourself)

##### 相关
* [Abstraction principle](http://en.wikipedia.org/wiki/Abstraction_principle_(computer_programming))
* [Once And Only Once is a subset of DRY (also referred to as the goal of refactoring)](http://c2.com/cgi/wiki?OnceAndOnlyOnce)
* [Single Source of Truth](http://en.wikipedia.org/wiki/Single_Source_of_Truth)
* [A violation of DRY is WET (Write Everything Twice)](http://thedailywtf.com/articles/The-WET-Cart)

### 代码可维护性
##### 为什么
* 维护是任何项目最昂贵的东西。

##### 怎么做
* 成为维护人
* 写代码时，想象维护的人是一个很暴力的人，并且知道你在哪。
* 多写注释
* 别让维护者去想
* 使用最小惊讶原则

##### 资源
[Code For The Maintainer](http://c2.com/cgi/wiki?CodeForTheMaintainer)
[The Noble Art of Maintenance Programming](http://blog.codinghorror.com/the-noble-art-of-maintenance-programming/)

### 避免过早优化 
引用[ Donald Knuth](http://en.wikiquote.org/wiki/Donald_Knuth)的话:


> 程序员浪费大量的时间来考虑，担心他们程序中不重要部分的速度，当调试和维护被考虑时，这些效率方面的考虑会造成消极的影响.我们应该忘掉所有的细微效率问题，大约97%的时间问题都是过早优化引起的，过早的优化是所有噩梦的根源，然而，我们不应该放过剩下3%的机会

理解什么是和什么不是过早优化，是本节的重点。

##### 为什么
* 不知道瓶颈在哪里。
* 优化后，可能更难阅读和维护

#### 怎么做
* [让它能正确工作](http://c2.com/cgi/wiki?MakeItWorkMakeItRightMakeItFast)
* 只有你真的需要优化，你才去优化,只有当你知道瓶颈在哪里.

##### 资源
* [Program optimization](http://en.wikipedia.org/wiki/Program_optimization)
* [Premature Optimization](http://c2.com/cgi/wiki?PrematureOptimization)

### 解耦


