# Assignment 1

## Read The Friendly Manual

Assignment 1 非常容易，主要是为了让大家感受一下 Tai-e 和它的数据流分析框架，同时熟悉一些 Java 编写知识。

在这里，我们简单介绍一下 Tai-e 的实验手册。

```
 ______                                 
/\__  _\          __                    
\/_/\ \/    __   /\_\              __   
   \ \ \  /'__`\ \/\ \  _______  /'__`\ 
    \ \ \/\ \L\.\_\ \ \/\______\/\  __/ 
     \ \_\ \__/.\_\\ \_\/______/\ \____\
      \/_/\/__/\/_/ \/_/         \/____/
```



Tai-e 的实验手册由以下部分构成：

* 算法详解，比如有哪些地方是课堂上没有讲解，需要在本次实验中补充的。
* Tai-e Classes 简介，框架提供了很多需要你利用起来的工具，这部分会简单列举并介绍你需要去了解的 class，并且建议去阅读它们的源码。
* 你的任务，会用红色字体标注你需要实现的 class/method。
* 运行与测试，这部分会介绍本次实验运行的结果，和测试上的细节。
* 提交的文件，Tai-e 只允许你修改指定的方法，对其它文件进行修改是无效的。

此外，每个段落都可能穿插一些 hint，这些 hint 会对实验编写大有帮助。



Tai-e 是由强大的谭添制作的，他惊人的代码能力与详尽的注释能让你仅靠手册和阅读代码就能完成实验。然而，实验手册毕竟只是个手册，当你要学习 CPU 指令知识的时候，看 80386 手册虽然清晰，但是也非常晦涩。很多同学想知道某个 API 在哪，此时仅靠手册无能为力，而翻代码又耗时耗力。本《手册的手册》往往就是为了解决这些问题。



但是别忘了，我们不会重复讲述实验手册。你还是得在阅读本《手册的手册》前，先通读一遍实验手册，试着去实验框架上手一下，有问题了，再来看。否则，你会觉得这边更加的晦涩难懂。



## Where Should I Start

这边我的建议是从 Solver 开始写，再去写 LVA 的逻辑，因为函数的大框架在 Solver 里，具体功能才是在 LVA 里面实现的。



## Tai-e FAQ You Need to Know

很多人可能第一次遇到 Tai-e 的数据流分析框架时，没法直接将框架代码与课上讲过的算法对应起来。这里我们用一张图来解释：

<img src="img/Assignment 1/image-20211101205624301.png" alt="image-20211101205624301" style="zoom:95%;" />

#### `Solver/IterativeSolver`

* `cfg` 是用来管理节点的，`result` 是用来管理数据流分析的结果的。无论是这次实验还是以后的几个实验，都是从 `cfg` 获取节点，再在 `result` 里索取或修改数据流分析结果。

* `Solver` 是一个抽象的分析框架，因此 `Node` 和 `Fact` 都是未确定的，不能做 `new Node()` 这样的操作。

* 为了支持 `meetInto`，你需要同时初始化 `InFact` 和 `OutFact`。手册有说的，但是蛮多人漏了。

* `analysis.transferNode` 是有返回值的，你可以充分利用这一点。

* Java 知识：你可以用 `for(Node node: cfg) {...}` 来遍历 `cfg`，也可以用 `cfg.forEach(lambda)` 来遍历，但是后者作为 lambda 表达式存在着无法重新赋值变量的问题，如果你不了解 lambda 表达式，要谨慎使用。

  * 在这里举一个用 `forEach` 的例子。请看上图中 `meetInto` 这一行在 `InterativeSolver` 里面是怎么写的：

    ```java
    cfg.succsOf(node).forEach(succ ->{
        // do something with succ and out[B], using analysis.meetInto()
    });
    ```

* 另一个 Java 知识：你在寻找 C++ `auto` 在 Java 里的对应体吗？请使用 `var`，然后 IDEA 会自动告诉你推导所得类型。

  <img src="img/Assignment 1/image-20211101210155421.png" alt="image-20211101210155421" style="zoom:95%;" />

#### `LiveVariableAnalysis`

* 你可能注意到了，`newBoundaryFact` 有一个参数 `cfg`。但是 Assignment 1 是用不到这个参数的，因为算法要求初始化为 empty。在 Assignment 2，就会需要 `cfg` 的信息来初始化了。

* 课堂上教学的时候使用的 data fact 存储结构是 bit vector，但是这需要事先收集程序中的所有出现过的变量：这显然是比较麻烦的。因此，本次实验使用的存储结构是 set，即集合。类型的名字就是 `SetFact`，里面很多涉及集合的操作。这就得你自己去看 API 了。

* 有些人并不是太了解 Java 的变量模型的。简单来说，你可以把它理解成 python 那样的模型：

  ```python
  >>> a = []
  >>> b = a
  >>> b.append(1)
  >>> a
  [1]
  ```

  这意味着什么呢？你可能会在 API 看到一个方法叫做 `copy()`。你可能会想，通过这个方法是否可以获得源 `SetFact` 对象的副本呢？像这样：

  ```java
  var tempFact = out.copy();
  // do something to tempFact
  if (tempFact != out){
  	// do something when "out" changes
  }
  ```

  但是其实你在做的是（从 C/C++ 的视角）：

  ```C++
  auto *tempFact = &out;
  if (&tempFact != &out){
  	// ...
  }
  ```

  那这样真是比较了个寂寞，还改了不应该修改的 `out` 本身。所以你应该使用另一个方法 `SetFact.set()` 来创造一个深拷贝。

* `stmt.getUses()` 返回的是一个 `List<RValue>`。在 `RValue` 下，有立即数也有变量。怎么判断一个 `RValue` 变量是不是指向了一个 `Var` 对象呢？Java 里的解决方法就是 `instanceof` 关键字。

* 依然是那个问题，`SetFact<Var>` 储存的是 `Var`，如何把 `RValue use` 内的对象放进去呢？方法就是强制类型转换 `(Var) use;`