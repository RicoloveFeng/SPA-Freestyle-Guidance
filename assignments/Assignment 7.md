# Assignment 7

>這次作業，InterSolver 給你控制，InterCP 給你控制，API 隨便你加，field 隨便你加，還不夠自由？
>
>我都忍不住想唱歌
>
>正在播放 過火 - 張信哲
>
>▶ ︎ı||ııı||ı|ıı||ı| 9”
>
>​								——譚添，評論于第七次實驗分發之後。



## Disclaimer

本次《手冊的手冊》因為是自己發明算法，所以不保證都是正確的。以下說明的正確性未進行嚴格證明，僅剛剛好能夠通過測試用例。



## Tai-e Classes You Need to Copy and Paste

* From Assignment 4:
  * `pascal/taie/analysis/dataflow/analysis/constprop/ConstantPropagation.java`
  * `pascal/taie/analysis/dataflow/solver/Solver.java`
  * `pascal/taie/analysis/dataflow/solver/WorkListSolver.java`

* From Assignment 6:
  * `pascal/taie/analysis/pta/cs/Solver.java`
  * `pascal/taie/analysis/pta/core/cs/selector/_2ObjSelector.java`
  * 如果你想比對別名分析下的 CS 指針分析，那麼也可以把別的 Selector 複製過來。

* Think twice before copying them:
  * `pascal/taie/analysis/dataflow/inter/InterSolver.java`
    * 事實上抽象算法不需要修改什麼東西，原來的 `initialize` 和 `doSolve` 都是可以直接用的。
  * `pascal/taie/analysis/dataflow/inter/InterConstantPropagation.java`
    * 對於 `transfer*Edge` 和 `transferCallNode`，都可以直接照搬。畢竟我們只是要處理一些新的語句而已。甚至，如果你喜歡在 IntraCP 裡面實現所有語句的功能，那麼整個 InterCP 都可以照搬。



## How to Do

### Basic Idea

讓我們回想一下，做過程間的常量傳播時，我們都幹了什麼：

* 每個節點存儲某個變量是什麼類型的常量
* 對於一個賦值語句，我們直接修改被賦值變量在 out fact 的常量值
* 對於一個計算語句，我們提取操作變量及它們在 fact 中的常量值，進行計算後，修改被賦值變量在 out fact 中的常量值。
* 當我們要更新 in fact 的時候，根據邊的類型，分別獲取源節點的 out fact，然後取 `meetInto` 操作。

在這一過程中，我們的 `transferNonCallNode` 函數採用了過程內的常量傳播的實現。由於我們在寫過程內的常量傳播時並沒有接觸到域和數組的處理，因此在遇到此類語句時，我們一律認爲結果爲 NAC，這是因爲我們沒學指針分析，無法判斷一個域有可能屬於哪一個對象，只能判斷一個變量聲明時的類型。

注意到指針分析是不依賴于常量傳播的，因此可以在過程間常量傳播先去計算指針分析的結果，然後為我們所用。

然而，常量傳播是流敏感的分析。

```
x = 1; // x -> 1
y = x; // x -> 1, y -> 1
x = 2; // x -> 2, y -> 1
```

L3 的賦值不會影響到 L2 的常量傳播。

然而，對域和數組的賦值卻是流不敏感的。

```
x.f = 1;
y = x.f; // y -> NAC (1 meet 2 -> NAC)
x.f = 2;
```

這就要求我們在進行一個 store 操作的時候，需要像 PTA 那樣，以別名為橋樑，把新的賦值傳播到各個相關的 load 中。同時每個被傳播到的 load 都要以其位置開始一次流敏感的常量傳播。



<img src="img/Assignment 7/image-20211211194657811.png" alt="image-20211211194657811" style="zoom:67%;" />

那麼問題來了：如何收集別名信息（圖中由 o 指向 x 的紅色線條）？

其實我們已經完成了指針分析，那麼直接提取指針分析的變量 point-to set 分析結果，建立一個反向的映射就可以了。

需要注意的是，由於 InterCP 不考慮 context，因此需要透过 `pta.vars()` 獲取所有不含 context 的 `Var`。但是，此時不需要考慮 `a.f` 和 `b.f` 存在別名關係而去遍歷  `pta.instanceFields()`，因為 `a.f.g` 和 `b.f.g` 在使用時都會先把 `a.f` 和 `b.f` 用臨時變量存起來，此時兩者的別名關係就體現在臨時變量裡面了。

另外應當指出，在常量傳播中，一個 `x = y` 會 kill 掉 in fact 中原有的 `y` 常量，但是對於 `x.f = y`，內部所記錄的行為應該是 `o.f = meet(o.f, y)`。



### サーキュレーション

#### For Fields...

從上面的代碼繼續延伸，假設我們有這麼一段代碼：

```
x.f = 1;
y = x.f;
// do something...
x.f = z;
```

如果我們一開始就先收集所有的 `x.f = ...` 並傳播到所有 load 中，那麼由於尚未開始常量傳播，我們並不知道 `z` 的值到底是多少。

回想一下指針分析是怎麼處理流不敏感分析的：當你得出一個變量的 point-to set 發生變化，你會掏出這個變量的相關語句，然後把這種變化傳播給它們。

根據這個思想，我們也可以構造一種差不多的算法：

```
// x.f = y
for obj in pts(x):
	obj.f = meet(obj.f, y)
    if obj.f changes:
    	for z in obj.alias:
    		foreach l: w = z.f:
    			kill (w, _), gen (w, obj.f)
    			start cp from l
```

這個算法基本上和上面的圖片有差不多的意思了。現在需要解決的問題就是：

* 如何存儲 `obj.f` 的值？這個問題就留給你來想了。「field 隨便你加」。
* 怎麼從 `l: w = z.f` 開始常量傳播？講義裡面寫了，InterCP 裡面是有一個 solver 的，只需要稍加改造 solver 就可以把 `l` 加入傳播流程。

#### For Static Fields...

如果你能弄懂 field 怎麼處理，那麼 **static field** 應該也就不是難事了。惟 `JClass` 不像 `Var` 那樣自帶所有的 `RelevantStmt`，所以這似乎就需要花費一次 icfg 遍歷來自行收集了。可能也有更好的辦法，比如一些隱藏的 API，但是，我不清楚。

#### For Arrays...

至於**數組**的話，看講義表格就行了，這裡簡單分析一下：

* 任何 `a[undef] = y` 的操作不會傳播出去；
* 若 `a[const] = y ` 導致 `o[const]` 的 Value 發生變化，那麼需要傳播給 `y = a[const]` 和 `y = a[NAC]`；（`a` 均為 `o` 的任意別名）
* 若 `a[NAC] = y ` 導致 `o[const]` 的 Value 發生變化，那麼需要傳播給 `y = a[*]` 和 `y = a[NAC]`。（`a[*]` 的星號代表任意 const value）



### Array Alias

數組的處理更加困難：因為你不僅需要考慮上面的流不敏感循環，還要考慮 `a[i]` 和 `b[j]` 的別名性質……

換句話說，當你知道 `x.f` 變化了，你知道一定有一個 `v.f` 在等著你去改變。

然而 `a[i]` 變化了，所有的 `b[*]` 都需要被改變嗎？如果需要改變，又要和誰進行 meet 操作呢？

---

首先我們要搞清楚 `a[const]` 和 `a[NAC]` 相互影響的關係。



考慮這樣一段代碼：

```
int[] a = new int[5];
for i in 0...4:
	a[i] = 123;
a[3] = 456;
b = a[3];
c = a[4];
```

那麼 `b` 和 `c` 分別是多少呢？

* 一個顯然的錯誤答案是 `b -> 456`。要記住數組也是流不敏感下的分析，不會因為一條新的 `a[3] = 456` 而 kill 掉什麼，使 `a[3]` 擁有全新的常量 456.

* 根據表格，我們知道 `a[const]` 和 `a[NAC]` 存在別名關係，那麼我們是否可以認為，`a[NAC]` 和 `a[3]` 佔用了同一個「存儲空間」，因此由於 `meet(123, 456) = NAC`，`b -> NAC, c -> NAC`？這也是不對的。

* 事實上，這個問題的答案可以在測試用例 `ArrayLoops/LoopMix` 裡面找到答案：`b -> NAC, c -> 123`。

這意味著，對於 `b = a[const]`，實際獲取的值是`b = meet(o[const], o[NAC])`。

（注意到我們這裡利用了講義中的假設，任何一個 field 或者 array 在被使用前都必須先初始化過，因此這可以避免「只初始化 `a[1]`，讀取未初始化的 `a[4]` 為什麼也可以等於 `a[NAC]`」的問題。）

---

那麼，更進一步地：

```
d = a[NAC]
```

此時 `d` 應該是多少？我們之前已經往 `o[NAC]` 存放了 `123`，那麼答案或許是 `d -> 123`？

但是這是不對的，正是因為 `a[NAC]` 不知道下標是多少，因此我們需要考慮所有情況的總和。既然 `a[*]` 除了 123 之外還可以通過 `a[4]` 得到 456，那麼 `d` 顯然不可能是一個常量。所以正確的答案是 `d -> NAC`。

---

總結一下就是：

* 對於 `a[i] = b`，實際存儲的是 `o[i] = meet(o[i], b)`。其中 `i` 代表 `const` 或 `NAC`。

* 對於 `b = a[NAC]`，實際獲取的值是 `b = meet(o[*])`。其中 `*` 代表所有 `const` 和 `NAC`。
* 對於 `b = a[const]`，實際獲取的值是`b = meet(o[const], o[NAC])`。



很神奇吧？



最後，`a[Undef]` 怎麼說？其實，你只要返回 `Undef` 就可以了。畢竟 `a[Undef] = b` 甚至都不與 `c = a[Undef]` 構成別名關係。



## Tai-e FAQ You Need to Know

* 有很多同學用了 Map 數據結構，但是輕率地使用了 `get` 方法，導致了 `NullPointerException`。請你試試這個樣例，看看自己有沒有中招：

    ```java
    public static void trap(){
        A c1;
        c1.f = 2;
        int r3 = c1.f;
    }
    ```

* 你可以用 `canHoldInt `來判斷一個語句是否要進行常量變化的處理。
