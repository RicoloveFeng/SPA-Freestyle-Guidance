# Assignment 8

## Tai-e Classes You Need to Copy and Paste 

* 首先是手册中提及的 Solver.java 的几个方法。但是别忘了，不要直接复制粘贴整个文件。
  * 同时你还需要做一个检查：在 PTA 中我们总假设方法调用是有接收变量 `r` 的，即只有 `r = x.m(...)` 这样的语句。然而，在污点分析的作业中，这个假设已经不再成立，所以你要检查处理 `Invoke` 语句的逻辑，预防没有 `lvalue` 的情况。
* Selectors 也是需要复制的，而且因为不同的测试项目用来不同的 selector，所以每个 selector 都要复制过来。



## Where Should I Start?

在刚刚经历过 Assignment 7 的流敏感中的流不敏感中，或许你已经被成功地绕晕了。那么回到污点分析的问题上，或许你就开始想到底这是个流敏感还是流不敏感之类的问题了。别急，让我们梳理一下。

### Taint Object

处理污点分析，我们得先想明白所谓「污点」是什么东西。事实上 Lecture 13 已经用一个样例给我们说清楚了，污点其实就是由 taint source 所产生的一个 taint object。

指针分析和污点分析的相似性就在于，指针分析分析的是 `New` 语句产生的 heap object 在程序中的传播，而污点分析分析的是一系列 `TaintSouce` 语句产生的 taint object 的传播。

理解了这一点，那么我们就能明白，只需要在指针分析的基础上，在处理 `Invoke` 语句的时候加入对 `TaintSource` 语句的检测，那么剩下的整套 PTA 的逻辑就是可以复用的。



### Taint Transfer

问题从污点传播开始就变得 tricky 了起来。因为 PTA 对 heap object 的传播定义为从某个变量的 point-to set 到另一个变量的 point-to set 的赋值，这其中只是发生了集合的并，并没有产生什么新的 heap object。

但是，taint transfer 是会产生新的 taint object 的。例如下面这段经典程序：

```java
StringBuffer sb = new StringBuffer();
sb.append("abc");
sb.append(taint);
sb.append("xyz");
String s = sb.toString();
```

注意到，每个 taint object 都是有类型的（这是为了利用类型系统避免 false positive 的污点流报告），所以从第三行 `taint -> sb` 的过程中，原本的 `String` 类型的 taint object 现在就需要转换为 `StringBuilder` 类型的 taint object。同样的，在第五行，又从 `StringBuilder` 类型转换为了 `String` 类型的 taint object。

这就超出了 PTA 所囊括的范围。虽然我们知道 taint transfer 的规则是什么，但是为了在程序中处理 taint transfer，就需要我们思考好这些问题：

* PTA 对 Invoke 的处理地点这么多，在哪里进行 transfer 的检测？
* 我们是否需要在 PFG 上连接一条从 taint source 到 taint target 的边？（例如，`sb.append(taint);`，`taint` 和 `sb` 需不需要连接一条边？）
* 如何把新生成的 taint object 传播出去？

这些问题先留给你思考，在下一个章节中我们再具体解答。



### Collecting Flows

不同于处理 taint source，处理 taint sink 的时候，不需要参与 PTA 过程。因此可以先走完 PTA 的流程，然后再去检查 point-to set 的情况来收集污点流。

这里唯一的问题就是如何获得所有的 `CSCallSite`。事实上本次作业足够自由，所以你大可以把 `callGraph` 从 `Solver` 传给 `TaintAnalysiss`，然后利用 `CSCallGraph` 的 API 来遍历。

如果你对 `Source` 和 `Sink` 的处理实现正确，那么不需要实现 taint transfer 的逻辑，也应该能够通过 `SimpleTaint` 的测试了。



## Handling Transfer

在 PTA 中，一共有两处处理 `Invoke` 的地方：

* 在 `addReachable` 处理新遇到的 `Invoke` 语句，主要是为 static call 连边；
* 在 `processCall` ，处理因为 `x` 的 point-to set 的改变，而发生的 PFG 变化和 `obj -> m.this` 的传播。

 同时，PTA 的行为是：

* 每当出现新的 PFG 边，我们就把 source Var 的 point-to set 复制到 target Var 上；
* 每当一个 Var 的 point-to set 发生变化，除了要通过 worklist 把变化传播出去，同时为所有流不敏感的相关语句补上 PFG 边。

那么回到污点分析上，我们应该如何污点分析的过程附加到 PTA 上面呢？

---

一个自然的想法是，每当一个 Var 的 point-to set 上的 taint object 子集发生变化，我们就找到所有它作为调用参数或者作为调用 base 的 Invoke 语句，然后根据 transfer 规则一个个检查是否满足，然后把变化通过 transfer 规则传过去。

这个过程很像是 PFG 连边一样，好像是在 `arg` 和 `result` 之间连了一条边，然后复用 PTA 的传播过程。

但是这并不代表我们真的就要根据 transfer 规则在 PFG 上连边，因为只有 taint object 通过特殊边发生了传播，heap object 不应该顺着 transfer 规则转移。

<img src="img/Assignment 8/image-20211223202133148.png" alt="image-20211223202133148" style="zoom:50%;" />

举个例子，如图，蓝色边代表 PFG 边，红色边代表 transfer 规则。当前 `a1: {o3, o4, A t1}` 有一条边通过 PFG 传播到了 `r` 上，同时又有一个 `arg-to-result` 规则把污点传播到 `r`。

此时，`a1` 的 point-to set 通过蓝色边传到 `r`，因此 `r` 的 point-to set 会有 `A t1`。而 `a1` 又通过红色边将 `t1` 传播给 `r`，但是规则上使得 `t1` 的类型被修改了。最后，我们得到`r: {o3, o4, A t1, B t1}`。

---

最后，我们得到处理 transfer 的结论：

* 当一个 Var 的 taint object 子集发生变化，检查其相关的语句，并看是否有符合条件的 transfer，然后将变化通过 worklist 算法传播出去。
* 由于 Var 不会保存作为调用参数时的那些相关语句，所以需要在初始化的时候自行保存。



现在你可以选择两种处理方法来应对 taint transfer。一种是仿造 PFG 建一个 taint flow graph, 一个是实时从 config 里查 taint transfer。

对于前者，就有了利用 PTA 算法本身的遍历，只需要自定义只传污点的特殊边，就不需要另外构建 TFG 了。

对于后者，则需要挨个查询 transfer 规则，但是考虑到实验体量较小，这个方法也是可以而棘手的。



## Dispatching

到目前为止我们解决了以下问题：

* 如何针对 Source 添加 taint object？在 `Invoke` 语句插桩，符合规则的就产生一个 taint object。
* 如何收集 taint flow？在 PTA 完成之后。
* 在什么时机，需要用什么方式利用 taint transfer 规则？当 Var 的 taint object 子集发生变化，除了要通过 PFG 传播变化 point-to set 的变化，还需要通过 transfer 规则进行传播。

但是有一个问题没有解决，就是：

* virtual invoke 怎么办？



刚刚我们的分析，事实上仅仅是考虑了静态的 `Invoke`，只不过因为本次实验的测试代码只有静态调用作为 Source，所以看不出来。

事实上在手册里面已经提到了 `c:l -> ct:m` 这一前提，因此对于 virtual call，应该先试图解析出方法及其上下文，再去找 Source 和 transfer rules.

Source 部分其实很好说，在 `processCall` 中判断一下就可以了。

但是对于 `arg-to` 类型的 transfer，需要把 base 指向的对象找出来，再一个个地进行 dispatch。当然，如果你采用的方法是「加 TFG 边」，应该就不会那么麻烦了。



在本文末尾会附加两个测试用例，测试你有没有进行 dispatch 的处理。



## Tai-e FAQ You Need to Know

本次实验实际上还算比较简单的，只要想清楚怎么做，那么很快也能做完。只有一个错误是比较多人出现的：

* 为什么正确答案是两条 taint flow，而我却有四条？A 方法的 Source 竟然流动到了 B 方法的 Sink 上！

鉴于我的经验，有可能是收集 taint flow 的时候没有弄对。当然，似乎还有别的原因，这就有待于你 debug 了！



## Appendix

以下是一些增强的测试用例。请选课的同学注意，这部分测试仅仅是为了提高程序的正确性，在实际的作业评测中，似乎不需要专门 dispatch 也可以得到满分。这就看你的选择了。

### Source

请测试以下代码：

```java
// Dispatch.java
class Dispatch{
    public static void main(String[] args) {
        A b = new B();
        A c = new C();
        SourceSink.sink(b.source()); // taint
        SourceSink.sink(c.source()); // no taint
    }
}

interface A{
    public String source();
}

class B implements A{
    public String source() {
        return new String();
    }
}

class C implements A{
    public String source() {
        return new String();
    }
}

```

并添加这条 Source：

`- { method: "<B: java.lang.String source()>", type: "java.lang.String" }`

### Transfer

请测试以下代码：

```java
class TransferDispatch{
    public static void main(String[] args) {
        argToResult();
        baseToResult();
    }

    static void argToResult(){
        String taint = SourceSink.source();
        A b = new B();
        A c = new C();
        String s1 = b.a2r(taint);
        String s2 = c.a2r(taint);
        SourceSink.sink(s1); // taint
        SourceSink.sink(s2); // no taint
    }

    static void baseToResult(){
        String taint = SourceSink.source();
        A b = new B();
        A c = new C();
        b.getTaint(taint);
        c.getTaint(taint);
        SourceSink.sink(b.b2r()); // taint
        SourceSink.sink(c.b2r()); // no taint
    }
}

interface A{
    public String a2r(String a);
    public void getTaint(String a);
    public String b2r();
}

class B implements A{
    public String a2r(String a){
        return new String();
    }
    public void getTaint(String a){

    }
    public String b2r(){
        return new String();
    }
}

class C implements A{
    public String a2r(String a){
        return new String();
    }
    public void getTaint(String a){

    }
    public String b2r(){
        return new String();
    }
}
```

并添加以下 transfer:

```
  - { method: "<B: java.lang.String a2r(java.lang.String)>", from: 0, to: result, type: "java.lang.String" }
  - { method: "<B: void getTaint(java.lang.String)>", from: 0, to: base, type: "B" }
  - { method: "<B: java.lang.String b2r()>", from: base, to: result, type: "java.lang.String" }
```

