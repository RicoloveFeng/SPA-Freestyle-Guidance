# Assignment 6

## Tai-e Classes You Need to Copy and Paste

本次作业的 `Solver.java` 有很大一部分可以直接用 Assignment 5 的代码，只需要改成 CS 的版本即可。

### 几乎可以完全复制粘贴的

以下两个方法在 CI 和 CS 的版本是完全一致的，因为它们要处理的对象都已经在调用时配置好了 `context`。See Lecture 12 page 94.

* `addPFGEdge`: 一个字符都不用改。
* `propagate`: 现在 `PointToSet` 是一个接口而不是一个实在类了，所以需要用最新的工厂来为你生成新的 `PointToSet` 对象。

### 只需要把 CI 方法换成 CS 方法的

之前很多 API 是由 `PointFlowGraph` 负责的，例如 `getVar`，`getStoreField`。现在，这些 API 转移到了 `CSManager` 上，所以部分函数基本上只要提取对应的 `context`，再更换 API，就能接着用了。

* `analyze`。
* `Stmt.Processor` 里面除了对 `Invoke`, `New` 以外的其它处理方法。
* `processCall` 里面 `if c:l -> c_t:m not in  CG then` 的部分

### 需要一些小小的改动

#### `processCall`

原本只需要 `m = resolveCallee(...)` 就可以确定 `m_this`，但是现在 `m` 需要一个新的上下文 `c_t`，因此需要调用 `selectContext`。

在 lecture notes 中，`selectContext` 的抽象原型是 `Select(c, l, c':o_i)`，但是在实现中，`c` 和 `l` 被整合成了一个 `CSCallSite`，并且额外添加了一个参数 `JMethod Callee`。😨啊，这个参数有什么用？！这个额外的参数在本次实验中是没有使用的，所以请坐和放宽，不要惊慌。😆

#### `addReachable, StmtProcessor`

现在为了利用 `context` 信息，每个 `stmtProcess` 需要在每次遇到新方法的时候自行新建。

* 对于 `visit(Invoke)`，其改动和 `processCall` 相同，只不过现在不需要 `c':o_i` 这个参数了而已。

* 对于 `visit(New)`，请注意，虽然算法上写的是 `add <c:x, {c:o_i}> to WL`，然而，两个 `c` 是不一样的，对于 k-limit 的 CS 场景，`o_i` 的 `c` 是 k-1 的。举个例子：
  * 现在是 2-Call-site Sensitive，当 `c:m` 的 `c` 是 `[la, lb]` 的时候…
    * `c:x` 的 `c` 是 `[la, lb]`
    * `c:o_i` 的 `c` 是 `[lb]`

​		这也就是为什么我们需要在 `ContextSelector` 上实现一个 `selectHeapContext`。



## `ContextSelector` FAQ You Need to Know

* 直接运行程序的话，默认是以 `CISelector` 运行的，所以你会发现输出里面 `context` 都是空的。为了设置恰当的 `ContextSelector`，你需要修改 `plan.yml`。
* 如何处理静态调用的 `selectContext` 已经在手册里提过了，请仔细阅读。
* `ObjSelector` 所使用的 `context` 是 `Obj` 而不是 `CSObj`，不要弄混了。
