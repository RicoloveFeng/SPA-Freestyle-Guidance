# Assignment 3

## Tai-e Classes You Need to Copy and Paste

这个实验需要你把:

* pascal/taie/analysis/dataflow/analysis/constprop/ConstantPropagation.java
* pascal/taie/analysis/dataflow/analysis/LiveVariableAnalysis.java
* pascal/taie/analysis/dataflow/solver/Solver.java
* pascal/taie/analysis/dataflow/solver/WorkListSolver.java

复制过来。不需要修改，只需要记得把前后向分析的代码合并一下。

## Tai-e FAQ You Need to Know

### `analyze(IR ir)`

* 死代码的消除过程一次 CFG 遍历就可以完成，所以不需要额外增加什么函数来处理，直接在 `analysis` 内完成所有逻辑就行了。
* `Set<Stmt> deadCode = new TreeSet<>(Comparator.comparing(Stmt::getIndex));` 使用红黑树实现的集合数据类型，并且提供了一个比较器来为 `Stmt` 排序，所以你不用担心 `deadCode` 里面的 `Stmt` 会乱掉。
* 遍历方法由你自己选，DFS BFS 都行。不要忘记处理环路就行了。
* 你可以把 IF 和 SWITCH 语句的 dot 图画出来，观察他们的边情况，然后决定怎么处理。