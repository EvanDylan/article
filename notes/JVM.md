# JVM内存管理

## 对象存活判定

### 引用计数算法

通过给对象添加一个引用计数器，每当有一个地方引用它时，计数器值就加1；当引用失效时，计数器值就减1；任何时刻计数器为0的对象就是不可能再被使用的。

优点：简单有效

缺点：无法解决对象直接循环引用的问题。比如A对象引用B对象，B对象又引用了A对象，如果它们都已经不再任何其他地方使用了，实际上已经是可销毁的对象了，但是按照引用计数器算法它们俩都属于存活的范畴。

### 可达性分析算法

通过一系列可达性分析来判定对象是否存活的。这个算法的基本思路就是通过一系列的成为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索走过的路径成为引用链，当一个对象到GC Roots没有任何引用链相连时，则证明此对象是不可用的。

在Java语言中，可作为GC Roots的对象包括下面几种：

- 虚拟机栈（栈帧中的本地变量表）中引用的对象。
- 方法区中类静态属性引用的对象。
- 方法区中常量引用的对象。
- 本地方法栈中JNI引用的对象。

## 垃圾收集算法

### 标记-清除算法（Mark-Sweep）

算法分为“标记”和“清除”两个阶段，首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。

缺点：效率不高，无法解决空间碎片问题。

### 复制算法

通过将内存空间划分成相同的两份，每次只使用其中一块。进行垃圾回收的时候，只要将其中存活的对象直接复制到另外一块预留的内存中即可，然后再把已使用过的内存空间依次清理掉。这样使得每次都是对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，只要一动堆顶指针，按顺序分配内存即可，实现简单，运行高效。

缺点：内存空间浪费了一半。

现在的JVM中新生代使用的就是这种算法，但是划分的不是两个对等的两份，而分出来Eden和两个较小的Survivor（s0和s1）。（基于IBM公司研究表明，新生代中的对象98%是“朝生夕死”的，所以不需要按照1:1的比例来划分空间）他们的比例默认为8:1:1,实际可使用内存空间为90%，每次进行回收时则将存活的对象复制移动到一个较小的Survivor。

缺点：老年代对象存活周期很长，这种复制算法并不适用。

### 标记-整理算法

复制收集算法在对象存活较高时就要进行较多的赋值操作，效率将会变低，更关键的是，如果不想浪费50%的空间，就需要有额外的空间进行分配担保，以应对被使用的内存中所有对象都是100%存活的极端情况，所以老年代一般不能直接选用这种算法。

### 分代收集算法

年轻代使用复制算法发+老年代使用标记整理算法

## 垃圾收集器

### 新生代收集器

#### Serial收集器

单线程的收集器，暂停其他所有线程，使用单线程回收。

#### ParNew收集器

Serial收集器的多线程版本

#### Parallel Scavenge收集器

Parallel Scavenge收集器的目标是达到一个可控制的吞吐量，并提供了两个参数用于精确控制吞吐量，分别是控制最大垃圾收集停顿时间的-XX:MaxGCPauseMillis参数以及直接设置吞吐量大小的-XX:GCTimeRatio参数。GCTimeRatio参数的值应当是一个大于0小于100的整数，是非垃圾回收时间与垃圾回收时间的比值。例如：设置为19，那么允许最大的GC时间久占总时间的5%（即1 / (1 + 19)），默认值为99，就是允许最大1%（即1 / (1 + 99)）。

### 老年代收集器

#### Serial Old收集器（标记整理）

Serial Old是Serial收集器的老年代版本，同样也是一个单线程收集器。另外一个作用当使用并发收集器CMS、G1在并发收集过程中发生内存不足的情况的备选方案。

#### Parallel Old收集器（标记整理）

吞吐量优先收集器的老年代版本，一般搭配新生代的Parallel Scavenge收集器使用。

#### CMS收集器（标记清除）

以最短停顿时间为目标的收集器，比较适合互联网的应用。垃圾回收过程经历以下四个阶段：

- 初始标记（需要STW）
- 并发标记（和应用线程一起运行）
- 重新标记（需要STW）
- 并发清除

可搭配ParNew收集器和Serial收集器使用

#### G1

G1回收器依然属于分代垃圾收集器，它会区分年轻代和老年代，依然有eden区和survivor区，但从堆的结构上看，它并不要求整个eden去、年轻代或者老年代都连续，它使用了分区算法，作为CMS的长期替代方案，G1同时使用了全新的分区算法，其特点如下：

- 并行性：G1在回收期间，可以由多个GC线程同时工作，有效利用多核计算能力。
- 并发性：G1拥有与应用程序交替执行的能力，部分工作可以和应用程序同时执行，因此一般来说，不会在整个回收期间完全阻塞应用程序。
- 分代GC：G1依然是一个分代收集器，但是和之前收集器不同，它同时兼顾年轻代和老年代。
- 空间整理：G1在回收过程中，会进行适当的对象移动，不像CMS，只是简单地标记清理对象，若干次GC后，CMS必须进行一次碎片整理。而GC不同，它每次回收都会有效的赋值对象，减少空间碎片。
- 可预见性：由于分区的原因，G1可以只选取部分区域进行内存回收，这样缩小了回收的范围，因为对于全局停顿也能得到较好的控制。

垃圾回收过程经历以下四个阶段：

- 初始标记（会伴随一次新生代GC，需要STW）

- 并发标记

- 重新标记（需要STW）

- 独占清理（需要STW）

  计算各个区域的存活对象和GC回收比例并进行排序，识别可供混合回收的区域。在这个阶段，还会更新记忆集（Remebered Set）。

- 并发清理阶段


