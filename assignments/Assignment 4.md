# Assignment 4

## Where Should I Start?

按照 Lecture 的讲解顺序，先从 CHABuilder 入手，不然的话连 ICFG 都没法输出。

CHABuilder 的三个待实现算法都在 ppt 上面有，完全可以照着写。

完成了CHABuilder，现在直接运行 Assignment 应该就可以显示正确的 dot 文件了，而且也可以进行 CHA 模块的测试。

然后和 Assignment 1~3 一样，我建议先做抽象的 InterSolver，再做具体的 InterConstProp。

## Tai-e FAQ You Need to Know

请注意：手册的手册不能替代手册，看不懂的话请回去看手册。

### CHABuilder

* `BuildCallGraph`
  * `CallGraph.reachableMethods()` 可以返回一个可达方法的 Stream，而不是 Set，所以不要想着用 `reachableMethods().contains(...)` 这种办法，而是应该利用 `CallGraph.addReachableMethod(...)` 的返回值。当然，自己维护一个 Set 也可以，我这么做了然后也过了。
  * `for call site cs in m` 这个地方可以用 `callSitesIn()` 遍历
  * `add cs->m' to CG` 可以用 `getCallKind()` 来获取边的类型，它的内部实现基本上就是一个个 instanceof 地去判断。

* `resolve`
  * 详解一下 `Invoke`：
    * 对于一个语句 `b = a.m(...)` 来说，`b` 是 `result`，`a.m(...)` 是 `InvokeExp`。
    * `f(...) {...; b = a.m(...); ...;}`，在这里 `f(...)` 是 `b = a.m(...)` 的 `container`。
    * 如果想获得 `a` 的类型，不是到 `InvokeExp` 里面找，而是到 `MethodRef` 里面找。
  * `m` 是一个函数签名，但是 `T` 是方法的集合。那怎么才能通过签名获得方法呢？注意上面说的，不要找到 `container` 去了，答案是通过签名的 `getDeclaringClass()` 获得类信息，再通过类信息获得 `JMethod` 对象。
  * `invokeinterface` 也是一种 `virtual call`
  * 处理 `virtual call` 的时候需要找 subclass，而 `JClass` 只提供了父类的信息。子类要怎么找呢？那就是函数开头已经帮你准备好的 `hierarchy`
  * `getSubclass` 是针对于类的，`getImplementor` 和 `getSubinterface` 是针对于接口的。不要搞错适用对象。
* `dispatch`
  * 注意一下各接口在什么时候会返回 `null` 就行了

### InterSolver

手册没有骗你，说几乎没区别那就是真的没区别。

* `initialize`
  * 手册的介绍可能会有点绕。其实在本次作业的 ICFG 中，entry method 似乎只有 `main` 一个。那么为什么程序会有多个入口方法？请看 Issues #31。
  * 在过程内的分析中，entry 和 exit 是不作为分析结果之一的，所以我当时只初始化了 `out[entry]`，然后把 entry 和 exit 排除在 `workList` 之外。但是在过程间的分析，这个排除过程就没有 `node != cfg.getEntry()` 那么优雅了，所以这次我把 entry 和 exit 也进行了初始化，并加回了 `workList`。应该是只有我自己踩的坑吧？
* `doSolve`
  * 是的，真的只需要把 `workListSolver` 的代码复制过来，然后改一下 IN 的修改方法就可以了。只不过这里从遍历 `cfg.predsOf` 变成了要遍历 `icfg.inEdgesOf` 

### InterConstantPropagation

* `transfer*Node`
  * `callNode` 和 `nonCallNode` 的区别，ppt 上面已经讲得很清楚了。善用自带的 `private final ConstantPropagation cp` 为你服务。
* `transfer*Edge`
  * 手册已经说过，这里再强调一下：不要忘了 Java 的对象管理机制，不要修改 `outFact`，也不要在返回 identity fact 的时候直接返回 `outFact`。

  * `transferCallEdge` 里面，`edge.getCallee().getIR().getParams()` 返回的是 `List<Var>`，所以可以用下标遍历。

    但是，在 `transferReturnEdge` 里面，`edge.returnVars()` 返回的是一个 `Stream<Var>`。

    后者只能通过 `edge.returnVars().forEach(lambda)` 遍历，不能通过下标和 `for (Var v : edge.returnVars())` 遍历。

    自己尝试就知道，lambda 表达式里面如果对外部变量重新赋值，是不会把修改后的变量往外传的。

    所以我建议对 `edge.returnVars().collect(Collectors.toList())` 遍历。当然也许会有更好的遍历方法？我不太了解。

  * 还是关于 `transferReturnEdge`。`edge.returnVars()` 里面有多个值，这是因为整个方法里的 return 语句都会指向 exit，再从 exit 连接单条 return edge 回上层方法。所以你需要把这些值做一个 `meetValue` 操作。



## Tai-e Classes You Need to Copy and Paste

这次作业的过程间处理有另外的处理，因此不需要担心，直接复制粘贴即可。

* pascal/taie/analysis/dataflow/analysis/constprop/ConstantPropagation.java
* pascal/taie/analysis/dataflow/solver/Solver.java
* pascal/taie/analysis/dataflow/solver/WorklistSolver.java

需要注意的是，Assi 2 里面测试通过的常量传播，有可能会在 Assi 4 出错。请务必确保代码正确。

