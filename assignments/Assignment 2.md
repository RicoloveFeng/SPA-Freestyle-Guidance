# Assignment 2

## Tai-e FAQ You Need to Know

#### `Solver/WorkListSolver`

* 前向算法和后向算法几乎是一样的，直接复制粘贴 Assignment 1 然后改改就行了。
* 然而，`workList` 算法是有一个队列的，而不是在有改变的时候再把整个 CFG 跑一次。因此你需要考虑怎么维护队列。在这里我推荐使用 `Queue`，但是 Java 的 `Queue` 只是一个接口类型，内部实现是自己挑的，我这里使用的是 `LinkedList`，具体的使用方法可以去搜搜。
* IDEA 教了我一个很有意思的写法：`cfg.succsof(node).forEach(workList::offer)`，这比 `forEach(node -> {workList.offer(node)})` 更加简洁。

#### `ConstantPropagation`

* `o.f = v`，在这个式子的处理上，我们提供 "identity function"。这个函数的意思就是，直接把 `in` 拷贝到 `out` 上，不做任何操作，就当这条指令是 `nop`。

* 你可能想知道 `getUses()` 除了 `x = y` 这条语句之外还返回些什么东西。我们举点例子吧：

  * `x = y op z `: `[y, z, y op z]`
  * `x = m(n)`: `[mClass, n, invokevirtual mClass.m(n)]`

* 手册里也有说 `DefinitionStmt` 的使用，于是你可能会想把 `Stmt` 类型转换到 `DefinitionStmt` 上。但是作为最顶层的方法，`getUses()` 其实也是够用的，从数组里面确定的位置挑出你想使用的东西就可以了。但是，如果你依然想做类型转换，那么可能会遇到模板参数不知道怎么填的问题。解决方法就是使用通配符：

  `DefinitionStmt<?, ?> definitionStmt = (DefinitionStmt<?, ?>) stmt;`

* API 里面的 `restrictTo`是用来做 `meetInto` 反向操作的（即，`restrictTo(const, Undef) == Undef`），请不要在意它。

* 在 `meetInto` 里面，如何优雅地遍历两个 map 并且将其合并呢？

  ```java
  target.update(key, meetValue(fact.get(key), target.get(key)));
  ```

* 不要忘了除零操作返回 `Undef`。

