# Assignment 5

## Where Should I Start?

本次实验对算法理解和接口阅读的需求非常大，我习惯从最顶层的 `analyze` 开始，然后先实现 `propagate` 和 `addEdge` 这两个过程内分析也需要的函数，然后再是过程间分析的 `processCall` 和 `addReachable`.



## From Algorithm to Implement

本章关注的是如何将抽象的分析算法落实到代码实现，其中的坑会有哪些。至于某个算法的具体实现的坑，请看 FAQ 章节。

### `S, S_m` 在哪？

之所以我想把这个章节特别拿出来记录，主要是 Assignment 1 的恐惧又回来了。

你应该记得，课堂上介绍 data fact 的时候，采用的是 bit vector。然而在实现中，并没有采用 bit vector，而是采用了 set fact。本次实验也有着类似的改动，使得你直接对着代码去寻找，是找不到一些算法里的数据结构的。

在 `initialize` 中，框架已经帮你初始化好了一些实用的工具，其中包括算法的 `WL, PFG, CG, RM`。其中，`RM` 在上一次实验中已经见过，是在 `CG` 中维护的，所以现在的问题就是：我的 `S` 哪去了？



这里就要介绍实验框架的修改。在算法的很多地方，我们需要 `foreach x... in S do`，这时候我们要找出和 `x` 相关的语句，并且根据对应类型来施加不同的操作。

这就带来了问题：我要去遍历 `S` 集合，然后一个个 `instanceof`，这太不优雅了。

实验框架的处理方式是：把各种与 `x` 有关且比较特殊的语句存在其内部，即，在 `Var` 类里面实现了一个 `RelevantStmts` 。

举个例子，如果此时有一个 `y = x.f` 的语句出现，那么在这条 `LoadField` 语句初始化的时候，框架就会往 `x` 添加这条语句。（你可以自己去看看构造函数的实现）

所以，当我们想 `foreach x... in S do` 的时候，只需要 `x.getXXX().forEach(...);` 就可以了。（详情请见 2.3 `pascal.taie.ir.exp.Var` 一节）



诶，等会，`S_m` 去哪里了？`AddReachable` 里面的 `S U= S_m` 怎么办？

是的，在框架里面，因为每个方法都生成了 IR，相应地也会为每个变量添加好 `RelevantStmts`。这会导致一些空间的消耗，但是因为变量都是局部的，当你访问到一个变量时，其相关语句必然也是 `reachable` 的。（死代码在之前已经消除过了）

总之助教说没事就没事，芜湖！



### 新规则的处理

本节主要是介绍新引入的 static/array 部分如何处理。

回想一下，为什么要在 `propagate` 完了之后再去给 `x.f = y` 这些变量做处理呢？因为 `x` 能指向的对象变了，因此对应对象的 `o_i.f` 也要修改。

而如果是 `T.f = y` 这种静态域的修改，没有「变量指向对象」的概念，也就不需要修改什么 `o_i.f`。

所以：

* array 的处理是和 `x.f = y` 并列的；

  <img src="img/Assignment 5/image-20211112135335878.png" alt="image-20211112135335878"  />

* 静态域放在 `AddReachable` 上，和 `x = y` 并列……或者本来你就可以把它当做一种特殊的 `x = y`；

* 静态调用也在 `AddReachable` 上。

  <img src="img/Assignment 5/image-20211112135420868.png" alt="image-20211112135420868"  />

当然，手册也已经给了相应的 hint 了，请 Ctrl + F 搜索 `handle` 一词来查找哪些新规则要引入到哪里。



### Visitor 模式

在手册里介绍了 `addReachable` 可以用 visitor 模式来实现。本节将快速带你了解为什么可以利用这个模式。



假设没有 visitor 模式，我们要实现 `addReachable`，那么我们就要 `getIR().getStmts().forEach(stmt -> ...)`，然后里面用大量的 `stmt instanceof XXX` 判断来做相应的处理。

当然这也是可行的。但是，因为框架提供了对 visitor 模式的支持，那么我们就可以直接让所有的 `stmt` 来 `accept` 我们的 `StmtProcessor`。

使用 visitor 模式的优雅在于，我们对每个可能的 `Stmt` 都做了相应的 `visit` 函数实现，这就使得 `stmt.accept` 会自动解析出应该调用哪个处理方法，不需要我们手动地去 `instanceof`。整体上代码就变得非常简洁。

## Tai-e FAQ You Need to Know

### `analyze`

<img src="img/Assignment 5/image-20211112220003102.png" alt="image-20211112220003102" style="zoom: 67%;" />

* 前面已经提到，`initialize` 已经完成了开头两个语句，所以不需要再在 `analyze` 里面做了。

* 算法中的 `delta = ...` 和 `propagate(n, delta)` 在代码框架中被合并成了一个函数。确实你可以把计算 `delta` 的语句独立到 `propagate` 之外，但是考虑到要实现 `propagate` 要返回 `delta` 的语义，还是把它们合起来做吧。

* `foreach x.f = y in S do ...` 这几个算法里，如何获取符合 `x.f = y` 的各条相应语句，在上一节已经提到过了，这里就不再赘述。这里的问题是，如何把 `o.f` 提取出来。

  我们首先需要获取 `f` 这个域是什么，这个问题可以看手册 2.4 addReachable 2) 来解决。然后，我们该怎么把 `o_i` 和 `f` 拼起来得到 `o_i.f` 呢？相应的 API 在 `pointerFlowGraph` 里面，请记住 `o_i.f` 这种东西叫做：

  <img src="img/Assignment 5/image-20211112220846683.png" alt="image-20211112220846683" style="zoom:150%;" />

* 为了不混淆 `addPFGEdge` 的 source 和 target，你可以令自己把 o_f, y 这些变量显式地写出来，然后再简洁地写上 `addPFGEdge(y, o_f)`，这虽然没有全写到一行干净利落，但是便于你 debug。
* 不要忘了处理 array.
* 由于 pointer 有 4 种类型，你可能会想要不要 `if n represents an instanceField o_f then` 这样子，给算法加加码。强调一下，不用！

### `addPFGEdge`

* 这个没什么好说的，就是调用 PFG 提供的 API 就好了。

### `propagate`

<img src="img/Assignment 5/image-20211112221737810.png" alt="image-20211112221737810" style="zoom: 80%;" />

* 关于 `delta` 的计算，你可以遍历 `pts`，跳过其中 `pt(n)` 有的元素，逐个构成 `delta`。不过，考虑到 `PointToSet.objects()` 返回的是一个 `stream`，你也可以用 `stream` 的方法来生成一个 `delta`。酷炫就完事了。（我试了，能行，但是我没用这个代码，所以把它贴出来，哈哈哈）

  ```java
  pointsToSet.objects().filter(ptsObj -> pointer.getPointsToSet().objects().noneMatch(ptnObj -> ptnObj.equals(ptsObj))).forEach(delta::addObject);
  ```

### `processCall`

<img src="img/Assignment 5/image-20211112222247197.png" alt="image-20211112222247197" style="zoom: 80%;" />

* 首先，static call 也是同样的处理逻辑，但是不需要 dispatch o_i，也不需要把 o_i 加到 m_this 里面（静态的哪来 object），但是 `processCall` 在语义上要求处理 instance call，所以还是不要把它改成能处理 static call 的样子，并把 `foreach l: r = x.k(...) in S do` 分拆到函数外。推荐把后面相同的处理逻辑提取一个新的函数出来，然后在 `processCall` 和 `addReachable` 都去调用。
* `Solver.resolveCallee()` 能处理所有种类的 invoke，所以不需要专门为 virtual call 从 hierarchy 里面拿 dispatch 函数出来了。事实上，如果你去看它的内部实现，在遇到 virtual call 时确实也去调了 `dispatch`。有更加全能，更兼容的 API，当然要用啦！
* `Solver.resolveCallee()` 返回的是 `JMethod`，但是如何根据 `JMethod` 拿到对应的 `this` 变量呢？答案是 `getIR`，IR 可以提供 `this` 变量。同样的，后面的 `return` 变量也是在 IR 里获取。
* 连接 CG 的边，逻辑是和 Assignment 4 一样的，但是 `getCallKind` 所属的 `CallGraphs.java` 在本次作业中放到了 `lib` 里，所以 IDEA 可能不会立刻给你补全。

### `addReachable` / `StmtProcessor`

<img src="img/Assignment 5/image-20211112223932920.png" alt="image-20211112223932920" style="zoom:80%;" />

* 前面已经提到过 `S` 在代码实现中的修改，这里不再赘述。总之，你在 `addReachable` 里面要做的事情就是遍历新 reach 到的方法的每一个 stmt，并根据其是否属于 New, Copy 等，分别作出处理。这个遍历的方法既可以是一个个 `instanceof`，也可以用 `stmt.accept(stmtProcessor)` 来充分利用框架提供的 visitor 模式。
* 你可能注意到了，`int a = 1`, `String b = "word"` 这样的语句在本次实验中并不会出现，所以不需要针对它们做支持。
* 在 `addReachable` 只需要处理 `New`, `Copy`, 以及静态的 `Inoke`, `StoreField`, `LoadField`，不需要多处理别的。
* 对于静态的 `Invoke`，其处理逻辑与 `processCall` 基本相同。需要注意的是，手册中 `1) do not need to dispatch on the receiver object` 说的是不需要**对 `recv` 做 dispatch**（因为 `recv == null`），但是不代表你不需要做 dispatch，这里依旧是调用 `resolveCallee` 就好了。
* 如果你是和我一样先写的 `analyze` ，那么你应该知道 `T.f` 应该去什么地方找了。
