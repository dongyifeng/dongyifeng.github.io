---
title: GC 算法
tags: GC JVM Java
typora-root-url: ../../dongyifeng.github.io
---



[TOC]

垃圾回收主要在：堆区和方法区

<img src="/images/java/WX20221228-144714@2x.png" style="zoom:50%;" />



# 标记阶段：引用计数算法

在堆里存放着几乎所有的 Java 对象实例，在 GC 执行垃圾回收之前，首先<font color=green>需要区分出内存中哪些是存活对象，哪些是已经死亡的对象。</font>只有被标记为已经死亡的对象， GC 才会所占在执行垃圾回收时，释放掉其所占用的内存空间，因此这个过程我们可以称为<font color=green>垃圾标记阶段。</font>

当一个对象已经不再被任何的存活对象继续引用时，就可以宣判为已经死亡。



判断对象存活一般有两种方式：<font color=red>**引用计数算法**</font> 和<font color=red>**可达性分析算法**</font>



引用计数算法（Reference Counting）比较简单：对每个对象保存一个整型的<font color=red>引用计数器属性。用于记录对象被引用的情况</font>

例如：一个对象 A，只要有任何一个对象引用了A，则 A 的引用计数器 += 1。当引用失效时，这个引用计数器 -=1。只要对象 A 的引用计数器的值为 0，就表示对象 A 不再被使用，可以进行垃圾回收。



优点：<font color=green>**实现简单，垃圾对象便于识别；判定效率高，回收没有延迟性。**</font>



缺点：

- 引用计数器有一个严重的问题：<font color=red>无法处理循环引用</font>的情况。 这是一条致命缺陷，导致在 Java 的垃圾回收器中没有使用这类算法。
- 需要单独存储计数器，增加了<font color=green>存储空间开销</font>。增加的不大，可以忽略。
- 每次赋值都需要跟新计数器，增加了<font color=green>时间开销</font>。可以忽略。



如下图：理循环引用

<img src="/images/java/WX20221228-151803@2x.png" style="zoom:50%;" />



验证： Java 的垃圾回收器中没有使用引用计数算法

```java
public class RefCountGC {
    // 5 MB
    private byte[] bigSize = new byte[5 * 1024 * 1024];

    Object reference = null;

    public static void main(String[] args) {
        RefCountGC obj1 = new RefCountGC();
        RefCountGC obj2 = new RefCountGC();

        obj1.reference = obj2;
        obj2.reference = obj1;

        obj1 = null;
        obj2 = null;

//        System.gc();
    }
}
```

没有 GC 时，查看空间占用情况

![](/images/java/screenshot-20221228-164345.png)



GC 后，查看空间占用情况：发现这些循环引用的对象，被垃圾回收了。

![](/images/java/screenshot-20221228-164539.png)



如图 obj1.reference = null 和 obj2.reference = null，Java 堆中虽然有循环依赖，但是在JVM 中依然能被垃圾回收。

<img src="/images/java/screenshot-20221228-165926.png" style="zoom:50%;" />





# 标记阶段：可达性分析算法

可达性分析算法也称：根搜索算法、追踪性垃圾收集。

可达性分析算法有效地<font color=green>解决在引用计数算法中<font color=red>循环引用</font>的问题，防止内存泄漏的发生。</font>

 Java、C# 使用的是：可达性分析算法。

GC Roots 就是一组必须活跃的引用。



基本思路：

- 可达性分析算法是以根对象集合（GC Roots）为起始点，按照从上至下的方式<font color=green>搜索被根对象集合所连接的目标对象是否可达。</font>
- 使用可达性分析算法后，内存中的存活对象都会根对象集合直接或者间接连接着，搜索所走过的路径称为<font color=red>引用链（Reference Chain）</font>
- 如果目标对象没有任何引用链向连，则是不可达的，就意味着该对象已经死亡，可以标记为垃圾对象。

<img src="/images/java/screenshot-20221228-172726.png" style="zoom:50%;" />



## GC Roots 包含以下几类元素

- 虚拟机栈中引用的对象
  - 比如：各个线程被调用的方法中使用到的参数、局部变量等。
- 本地方法栈内 JNI （通常说的本地方法）引用的对象。
- 方法区中类静态属性引用的对象
  - 比如：Java 类的引用类型静态变量
- 方法区中常量引用的对象
  - 比如：字符串常量池（String Table）里的引用
- 所有被同步锁 synchronized 持有的对象
- Java 虚拟机内部的引用。
  - 基本数据类型对应的 Class 对象，一些常驻的异常对象（如：NullPointerException、OutOfMemoryError），系统类加载器。
- 反映 java 虚拟机内部情况的 JMXBean、JVMTI 中注册的回调、本地代码缓存等。



**小技巧**

由于 Root 采用栈方式存放变量和指针，所以如果一个指针，它保存了堆内存里面的对象，但是自己又不存放在堆内存里面，那它就是一个 Root（堆周边的指针）。

<img src="/images/java/screenshot-20221228-193314.png" style="zoom:38%;" />



除了上述固定的 GC Roots 集合以外，根据用户所选用的垃圾收集器以及当前回收的内存区域不同，还可以有其他对象“临时性”地加入，共同构成完整 GC Roots 集合。比如：分代收集和局部回收（Partial GC）

比如：在 YGC 时只对新生代进行 GC，那么老年代就成了 GC 边界外。

<img src="/images/java/screenshot-20221228-194623.png" style="zoom:50%;" />

**注意：**

如果要使用可达性分析算法来判断内存是否可回收，那么分析工作必须在一个能保障一致性的<font color=red>快照</font>中进行。这点不满足的话分析结果的准确性就无法保证。

因此 GC 进行时必须 “ Stop The World ”。即使号称（几乎）不会发生停顿的 CMS 收集器中，<font color=red>枚举根节点时也是必须要停顿的。</font>



# 对象的 finalization 机制

Java 语言提供了对象终止（finalization）机制来允许开发人员提供<font color=green>对象被销毁之前的自定义处理逻辑。</font>

触发时机：垃圾回收此对象之前，总会先调用这个对象的 finalize() 方法。

重写  finalize()  方法一般是<font color=green>用于在对象被回收时进行资源释放。</font>比如：关闭文件、套接字和数据库连接等。

注意：<font color=red>永远不要主动调用某个对象的 finalize() 方法</font>

- 在 finalize() 时可能会导致对象复活。
- 一个糟糕的 finalize() 会严重影响 GC 的性能。



由于 finalize() 方法的存在，<font color=green>虚拟机中的对象一般处于三种可能的状态。</font>

<font color=red>一个无法触及的对象（垃圾）有可能在某些条件下 “复活” 自己。</font>

- <font color=red>可触及的：</font>从根节点开始，可以到达这个对象
- <font color=red>可复活的：</font>对象的所有引用都被释放，但是对象有可能在 finalize() 中复活。
- <font color=red>不可触及的：</font>对像的 finalize() 被调用，并且没有复活，那么就会进入不可触及状态。不可触及的对象不可能被复活，因为<font color=green> finalize() 只会被调用一次。</font>

 

标记过程：

判定一个对象 obj 是否可回收，至少要经历两次标记过程：

1. 如果对象 obj 到 GC Roots 没有引用链，则进行一次标记。
2. 进行筛选，判断此对象是否有必要执行 finalize() 方法
   1. 如果对象 obj 没有重写 finalize() 方法，或者 finalize() 方法已经被虚拟机调用过，则虚拟机视为“没有必要执行”，obj 被判定为<font color=green>不可触及的</font>。
   2. 如果对象 obj 重写了 finalize() 方法，且还未执行过，那么 obj 会被插入到 F-Queue 队列中，由一个虚拟机自动创建的、低优先级的 Finalizer 线程触发其 finalize() 方法执行。
   3. <font color=red>finalize() 方法是对象逃脱死亡的最后机会，</font>稍后 GC 会对 F-Queue 队列中的对象进行二次标记。如果 <font color=green>obj 在 finalize() 方法中与引用链上的任何一个对象建立了联系（此时状态为可复活）</font>，那么在第二次标记时，obj 对象会被移除 “即将回收” 集合。之后如果对象会再次没有引用的存在的情况。在这种情况下， finalize() 方法不会再次调用，对象会直接变为不可触及的状态，也就是说，一个对象的 finalize 方法只会被调用一次。

<img src="/images/java/screenshot-20221228-221227.png" style="zoom:38%;" />



代码测试

```java
public class CanReliveObj {
    // 类变量，属于 GC Roots
    public static CanReliveObj obj;

   // 此方法只能被调用一次
    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        System.out.println("CanReliveObj.finalize()");
        // 复活 obj
        obj = this;
    }

    public static void main(String[] args) {
        try {
            obj = new CanReliveObj();
            // 对象第一次拯救自己
            obj = null;
            System.gc();
            System.out.println("第一次 GC");

            // 由于 Finalizer 线程优先级低，暂停 2 秒，等等它。
            Thread.sleep(2000);
            if (obj == null) {
                System.out.println("obj is dead");
            } else {
                System.out.println("obj is still alive");
            }

            obj = null;
            System.gc();
            System.out.println("第二次 GC");
            Thread.sleep(2000);
            if (obj == null) {
                System.out.println("obj is dead");
            } else {
                System.out.println("obj is still alive");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

执行结果：

```apl
第一次 GC
CanReliveObj.finalize()
obj is still alive
第二次 GC
obj is dead
```



如果注释掉 重写的方法 finalize()，执行结果为：

```apl
第一次 GC
obj is dead
第二次 GC
obj is dead
```



# MAT 与 JProfiler 的 GC Roots 溯源

MAT 是 Memory Analyzer 的简称，它是一款功能强大的 Java 堆内存分析器。用于查找内存泄漏以及查看内存消耗情况。

 

**获取 dump 文件**

方式一：命令行使用 jmap

![](/images/java/WX20221229-144927@2x.png)



方式二：使用JVisualVM 导出

<img src="/images/java/WX20221229-145244@2x.png" style="zoom:30%;" />



<img src="/images/java/WX20221229-145359@2x.png" style="zoom:50%;" />







# 清除阶段

当成功区分出内存中存活对象和死亡对象后，GC 接下来的任务就是<font color=red>执行垃圾回收</font>，释放掉无用对象所占用的内存空间，以便有足够的可用内存空间为新对象分配内存。



三种垃圾收集算法：

- 标记-清除算法（Mark-Sweep）
- 复制算法（Copying）
- 标记-压缩算法（Mark-Compact）



## 标记-清除算法

标记-清除算法（Mark-Sweep）是一种非常基础和常见的垃圾收集算法。

**执行过程**

当堆中的有效内存空间（available memory）被耗尽的时候，就会停止整个程序（也被称为 stop the world）,然后进行两项工作：标记和清除。

- <font color=green>**标记：**</font> Collector 从引用根节点开始遍历，<font color=red>标记所有被引用的对象</font>。一般是在对象的 Header 中记录为可达对象。
- <font color=green>**清除：**</font>Collector 对堆内存从头到尾进行线性的遍历，如果发现某个对象在其 Header 中没有标记为可达对象，则将其回收。

<img src="/images/java/WX20221229-153121@2x.png" style="zoom:30%;" />

**缺点**

- 这种方式清理出来的空闲内存是不连续的，产生内存碎片。需要维护一个空闲列表
- 效率不高
- 在进行 GC 的时候，需要停止整个应用程序，导致用户体验差



**注意：**这里所谓的清除并不是真的置空，而是把需要清除的对象地址保存在<font color=red>空闲的地址列表</font>里。下次有新对象需要加载时，判断垃圾的位置是否足够，如果够，就存放。



## 清除阶段：复制算法

**背景：**为了解决标记-清除算法在垃圾收集效率方面的缺陷。

**核心思想**

将活着的<font color=red>内存空间分为两份</font>，每次只使用其中一块，在垃圾回收时将正在使用的内存中的存活对象复制到未被使用的内存块中，之后清除正在使用的内存块中的所有对象，交换两个内存的角色，最后完成垃圾回收。



如下图：内存块 A 区和 B 区交替使用。GC 后内存是连续的，适合指针碰撞。

<img src="/images/java/WX20221229-155014@2x.png" style="zoom:33%;" />



**优点**

- 没有标记和清除过程，实现简单，运行高效
- 复制过去以后保证空间的连续性，不会出现“碎片”问题。



**缺点**

- 需要两倍的内存空间。
- 对于 G1 这种分拆成为大量 region 的 GC，复制而不是移动，意味着 GC 需要维护 region 之间的对象引用关系，不管是内存占用或者时间开销也是不小



由于对象地址变更了，那么在 Java 栈中引用地址也需要变更（所有存放地址的地方都）。

<img src="/images/java/WX20221209-171620@2x.png" style="zoom:33%;" />





**注意：**

如果系统中的<font color=green>垃圾对象很多，复制算法需要复制的对象数量不是太大</font>，这种场景适合复制算法，比如 YGC 的 s0 和 s1

<img src="/images/java/WX20221204-130444@2x.png" style="zoom:33%;" />

<img src="/images/java/WX20221204-184740@2x.png" style="zoom:33%;" />



而老年代中大部分对象都是长期存活对象，并且数量非常多，如果要使用复制算法，那么需要复制的对象就非常多了。这个场景就不合适复制算法。

 

## 清除阶段：标记-压缩算法

**背景**

复制算法的高效性是建立在<font color=green>存活对象少，垃圾对象多</font>的前提下的。这种情况再新生代经常发生，但是老年代，更常见的情况是大部分对象都是存活对象。<font color=red>基于老年代垃圾回收的特性，需要使用其他算法。</font>

为了解决标记-清除算法产生的内存碎片的问题，设计出了标记-压缩（Mark-Compact）算法。



**执行过程**

- 第一阶段和标记清楚算法一样，从根节点开始标记所有被引用的对象。
- 第二阶段将所有的<font color=green>存活对象压缩到内存的一端</font>，按顺序排放。之后，清理边界外所有的空间。

<img src="/images/java/WX20221229-173945@2x.png" style="zoom:33%;" />



  标记-压缩算法的最终效果等同于：标记-清除算法执行完毕后，再进行一次内存碎片整理。因此，也可以把它称为：<font color=red>标记-清除-压缩（Mark-Sweep-Compact）</font>

二者的本质差异在于标记-清除算法是一种<font color=green>非移动式的回收算法</font>。标记-压缩是<font color=green>移动式的</font>。是否移动回收后的存活对象是一项优缺点并存的风险决策。



**优点**

- 消除了标记-清除算法当中，内存区域分散的缺点，我们需要给新对象分配内存时，JVM 只需要持有一个内存的起始地址即可。
- 消除了复制算法当中，内存减半的高额代价



**缺点**

- 从效率上来说，标记-整理算法要低于复制算法
- 移动对象的同时，如果<font color=red>对象被其他对象引用，则还需调整引用的地址。</font>
- 移动过程中，需要全程暂停用户应用程序。即：STW。

 

## 小节

|          | Mark-Sweep       | Mark-Compact     | Copying                                 |
| -------- | ---------------- | ---------------- | --------------------------------------- |
| 速度     | 中等             | 最慢             | 最快                                    |
| 空间开销 | 少（会堆积碎片） | 少（不堆积碎片） | 通常需要活对象的 2 倍大小（不堆积碎片） |
| 移动对象 | 否               | 是               | 是                                      |

从效率上来说，复制算法是当之无愧的老大，但是却浪费了太多内存。



## 分代收集算法

前面所有算法中，并没有一种算法可以完全替代其他算法，它们都具有自己独特的优势和特点。因此分代收集算法应运而生。

分代收集算法将<font color=green>不同生命周期的对象采用不同的收集方式，以便提高回收效率。</font>

在 Java 程序运行过程中，会产生大量的对象，其中有些对象是与业务息息相关，比如：<font color=green>Http 请求中的 Session 对象、线程、Socket 连接</font>，这类对象生命周期比较长。但是有一些对象是程序运行过程中生成的临时变量，这些对象生命周期比较短，比如：<font color=green>String 对象</font>，由于其不变类的特性，系统会产生大量的这些对象，有些对象甚至只用一次即可回收。



<font color=green>目前几乎所有的 GC 都是采用分代收集（Generational Collecting）算法执行垃圾回收的</font>

- 年轻代（Young Gen）
  - 年轻代特点：<font color=orange>区域相对老年代较小，对象生命周期短、存活率低，回收频繁。</font>
  - 年轻代使用<font color=red>复制算法</font>回收整理，速度最快。复制算法的效率只和当前存活对象大小有关，因此很适合用于年轻代的回收。而复制算法内存利用率不高的问题，通过 hotspot 中的两个 survivor 的设计得到缓解。
- 老年代（Tenured Gen）
  - 老年代特点：<font color=orange>区域较大，对生生命周期长、存活率高，回收不及年轻代频繁。</font>
  - 这种情况存在大量存活率高的对象，复制算法明显变得不合适。一般由<font color=red>标记-清除算法或者标记-整理算法</font>混合实现
    - Mark 阶段的开销与存活对象的数量成正比
    - Sweep 阶段的开销与所管理区域的大小成正相关
    - Compact 阶段的开销与存活对象的数量成正比



以 HotSpot 中的 CMS 回收器为例，CMS 是基于 Mark-Sweep 实现的，对于对象的回收效率很高。而对于碎片问题，CMS 采用基于 Mark-Compact 算法的 Serial Old 回收器作为补偿措施：当内存回收不佳（碎片导致的 Concurrent Mode Failure 时），将采用 Serial Old 执行 Full GC 以达到老年代内存的整理。



分代的思想被现有的虚拟机广泛使用。几乎所有的垃圾回收器都区分新生代和老年代。



## 增量收集算法

**背景**

上述现有算法，在垃圾回收过程中，应用软件将处于一种 <font color=green>Stop the world</font> 的状态。在 Stop the world 状态下，应用程序所有线程都会挂起，暂停一切正常的工作，等待垃圾回收的完成。如果垃圾回收时间过长，应用程序会被挂起很久，<font color=green>将严重影响用户体验或者系统稳定性</font>。为了解决这个问题，即对实时垃圾收集算法的研究导致了增量收集（Incremental Collecting）算法的诞生。



**基本思想**

如果一次性将所有的垃圾进行处理，需要造成系统长时间的停顿，那么就可以让垃圾收集线程和应用程序线程交替执行。<font color=green>每次垃圾收集线程只收集一小片区域的内存空间，接着切换到应用程序线程。依次反复，直到垃圾收集完成。</font>



总的来说，增量收集算法的基础仍是传统的：标记-清除和复制算法。增量收集算法通过对<font color=red>**对线程间冲突的妥善处理，允许垃圾线程以分阶段的方式完成标记-清理或复制工作**</font>



**缺点**

使用这种方式，由于在垃圾回收过程中，间断性地还执行了应用程序代码，所以能减少系统的停顿时间。但是，因为线程切换和上下文转换的消耗，会使得垃圾回收的总体成本上升，<font color=green>造成系统吞吐量的下降</font>。



## 分区算法

一般来说，在相同条件下，堆空间越大，一次 GC 所需时间就越长，有关 GC  产生的停顿越长。为了更好地控制 GC 产生的停顿时间，<font color=green>将一块大的内存区域分割成多个小块，根据目标的停顿时间，每次合理地回收若干个小空间</font>，而不是整个堆空间，从而减少一次 GC 所产生的停顿。



分代算法按照对象的生命周期长短划分成两个部分，分区算法将这个堆空间划分成连续的不同小区间（region）。每个小区间都是独立使用，独立回收。这种算法的好处是<font color=green>可以控制一次回收多少个小区间。</font>

<img src="/images/java/WX20230101-130019@2x.png" style="zoom:33%;" />



注意：这些只是基本的算法思路，实际 GC 实现过程要复杂的多，目前还在发展中的前沿 GC 都是复合算法，并且并行和并发兼备。