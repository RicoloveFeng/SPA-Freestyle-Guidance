# Pointer Analysis

## Motivation

先看这个例子：

```
Number n = new One();
int x = n.get();

class Zero implements Number{...}
class One implements Number{...}
```

由于我们在 CHA 中只会分析到 `Number n = ...`，因此我们会认为 x 的 call target 不止一个，因此 x 是 NAC。但这是不准确的，我们可以通过分析 n 所指向的对象来获得确实的常量。



## Introduction to Pointer Analysis

对于一个简单的程序， 我们人脑分析是非常直白的。New 语句不就在那里吗？记下来不就好了。

然而，在实际的工程当中，分析的表项甚至以亿计。在这种量变之中，如何解决引起的质变，就是我们需要考虑的问题。

关于 PTA 的一些介绍，请看讲义本体，这里就不再复制粘贴了。

总之，通过指针分析，我们希望给定一个「指针（Pointer）」，它的指向集（Point-to Set）有可能包含哪些 object。



## Key Factors of Pointer Analysis

指针分析存在几个可以考虑的方面：

<img src="img/8_Pointer Analysis/image-20211021192911882.png" alt="image-20211021192911882" style="zoom:80%;" />

### Heap Abstraction

第一个要回答的问题是，如果我们要知道变量指向哪个 object，那么如何对 object 本身建模？

考虑下面这样一个代码：

```java
1 for (i = 0; i < x; i++){
2     a = new A();
3     //...
4 }
```

在实际运行的过程中，创建对象的数量会因为 x 的不同而不同。在静态分析的过程中，我们不可能去假设 x 每一个可能的值，然后去分析创建出来的对象有什么指向关系。

一种常用的方法是基于 allocation site 来抽象一个对象。即，每一个 New 所创建出来的对象，我们都认为是同一个，然后在这个抽象下去分析。

<img src="img/8_Pointer Analysis/image-20220103143322127.png" alt="image-20220103143322127" style="zoom:67%;" />

例如，我们把第二行的对象抽象为 o_2，而不去构建 o_2_iteration1, o_2_iteration2...

<img src="img/8_Pointer Analysis/image-20220103143423818.png" alt="image-20220103143423818" style="zoom: 50%;" />

由于代码中的 New 语句是有限的，因此我们抽象出来的对象自然也就是有限的了。

### Context Sensitivity

所谓上下文，就是说我可以记住调用是在哪里发生的。例如，同样都是调用 `id(...)`，我可以区分出这是第几行发生的。

<img src="img/8_Pointer Analysis/image-20220103143731966.png" alt="image-20220103143731966" style="zoom: 67%;" />

注意到，如果我们不去考虑上下文，那么由于 `x = id(n1)` 和 `y = id(n2)` 在 ICFG 上无法区分 return 边对应哪个 entry 边，因此数据流只能混杂到一起，我们只能得到 `i = NAC`。

<img src="img/8_Pointer Analysis/image-20220103145319793.png" alt="image-20220103145319793" style="zoom:50%;" />

因此基本上都会考虑上下文，不过在课程中我们会从上下文不敏感开始学习。

### Flow Sensitivity

流的敏感和不敏感主要说的是我们是否考虑一个方法内的前后执行顺序。例如下面这段代码：

<img src="img/8_Pointer Analysis/image-20220103150612877.png" alt="image-20220103150612877" style="zoom: 50%;" />

蓝色的是流敏感分析，橙色的是流不敏感分析。流敏感很好理解，流不敏感则是因为我们认为 `c.f = "y"` 会传播到整个程序上，而非「在 `s = c.f`之后」，因此会出现对 `s` 的可能结果存在误报。

那看起来流敏感是个不错的选择。然而在实现的时候，流敏感会比流不敏感复杂，但是这样的开销，却不会产生远优于流不敏感的效果。因此实践中一般采取流不敏感的策略。

### Analysis scope

最后一个因素是分析什么地方。这里的区别在于，可以分析全程序的指针指向关系，也可以分析影响某个 call site 或者指针的相关程序部分（也就是 demand driven）。

<img src="img/8_Pointer Analysis/image-20220103151302200.png" alt="image-20220103151302200" style="zoom: 67%;" />

全程序分析是蓝色的结果，但是如果我们只需要构建 CG，那么我们其实只需要分析 z 的指向关系，此时我们可以略过 x 和 y 的指向关系分析。本课程选择全程序的分析。

## Concerned Statements

还记得我们提到我们采用的是流不敏感分析。这种分析方法给我们带来的便利就是，控制流语句都可以被省略了。

* if-else
* while-for-do while
* switch-case
* break-continue
* ...

这些语句都可以被忽略。我们的教学环境是 Java，在 Java 中，我们只需要关心：

* 可以用来构建指向关系的东西（即指针）：

    * Local variable: `int x`, `A y`, `String z`

    * Static field: `A.f`，它可以当做一个全局变量来处理，因此和局部变量的处理是差不多的。
    * Instance field: `x.f`
    * Array element: `a[i]`，但是静态分析通常很难算出某个下标指向什么，因此我们会忽略下标，将数组本身视为一个 field（即 `a[*]` 都被视为 `a.arr`）。

* 导致指向关系发生变化的语句：

    * `x = new T()`，这会将一个新的 abstract object 添加到 x 的 point-to set 中
    * `x = y`，这会将 `y` 的 point-to set 传播给 `x`
    * `x.f = y`，这会将 `y` 的 point-to set 传播给 `o.f`，其中 `o` 是 `x` 的 point-to set 中的任一 object
    * `y = x.f`，与上面类似。
    * `r = x.k(a, ...)`，将 `o.k(a, ...)` 的 return 出来的 point-to set 传播给 `r`。注意到 call 也有好几种，我们只关注 virtual call。



## 划重点

* 什么是指针分析（PTA）
* PTA 的关键要素
* PTA 会考虑哪些东西

