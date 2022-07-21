---
title: JVM 垃圾回收笔记
date: 2022-07-20 13:43:37
tags:
- JVM
---
# JVM 垃圾回收笔记
# 参考文章
[JVM 规范](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)

https://blogs.oracle.com/jonthecollector/our-collectors

关于 minor gc 和 full gc 的名词，从该文章可以获取

[Java Hotspot 内存管理白皮书](https://www.oracle.com/technetwork/java/javase/tech/memorymanagement-whitepaper-1-150020.pdf)


## GC 算法
### 标记-清除算法(Mark Sweep)
Mark-Sweep 算法，在 1960 年由 Lisp 之父 John McCarthy 所提出。分为 "标记" 和 "清除" 两个阶段：

- 标记：标记出所有需要回收的对象，或者标记存活的对象
- 清除：标记完成后统一回收需要回收的对象

不足之处主要有两点：

- 效率问题：执行效率不稳定，如果堆中包含大量对象，且其中大部分对象都需要被回收，这时必须进行大量标记和清除动作。
- 空间问题：内存碎片


### 标记-复制算法(Copying)
常被简称为 "复制算法"。为了解决标记-清除算法面对大量可回收对象时执行效率低的问题。1969 年 Fenichel 提出一种称为 "半区复制"（Semispace Copying）垃圾回收算法，将可用内存按容量划分为大小相等的两块，每次只使用其中的一块，当这一块内存用完了，就将还存活着的对象复制到另外一块上面，再把已使用过的内存空间一次清理掉。

> 如果内存中的多数对象都是存活的，这种算法会产生大量的复制开销。但对于多数对象都是可回收的情况，算法需要复制的就是占少数的存活对象，而且每次都是有序复制到另一块内存，分配内存时也就不用考虑空间碎片的情况。

经过研究，大部分新生代的对象是 "朝生夕死" 的，也就是只有极少的的对象会存活下来。在 1989 年，Andrew Appel 针对这个特点，提出了一种更优化的半区复制分代策略，现在称为 "Appel 式回收"。HotSpot VM 的 Serial、ParNew 等新生代回收器均采用了这种策略来设计新生代的内存布局。Appel 式回收的具体做法时把新生代分为一块较大的 Eden 空间和两块较小的 Survivor 空间，每次分配内存只使用 Eden 和其中一块 Servivor。发生 GC 时，将 Eden 和 Servivor 中仍然存活的对象一次性复制到另外一块 Servivor 空间上，然后清理掉 Eden 和已经使用过的那块 Survivor 空间。


HotSpot VM 将新生代划分为 Eden 和 2 个 Survivor Space，Eden 和 Survivor 的比例为 8:1，也就是新生代中可用内存为总容量的 90% （80% + 10%），只有 10% 会被浪费。

> 考虑到最极端的情况，90% 的对象都是存活的，显然 10% 的 Survivor Space 不够用，需要依赖其他内存（老年代）进行分配担保，多出来的对象直接通过分配担保机制进入老年代。


### 标记-压缩算法(Mark Compact)
有些地方也叫标记-整理算法。

复制算法在对象存活率较高时，就要进行较多的复制，效率就会变低。所以，老年代一般不能直接选用这种算法。

根据老年代的特点，1974 年 Edward Lueders 提出了另外一种有针对性的 "标记-整理"（Mark-Compact）算法。标记过程与 "标记-清除" 算法一样，但后续不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。



## 对象存活的判定
### 引用计数法(Reference Count)
对象持有一个引用计数器，每当有一个地方引用它时，值加1；引用失效时，值减 1。当计数器值为 0 时，对象就是不可能再被使用。

主流 JVM 都没有选用引用计数法，最主要原因是难以解决对象之间循环引用的问题。


### 可达性分析算法(Root Searching)
基本思想：通过一系列称为 ”GC Roots“ 的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用恋，当一个对象到 GC Roots 没有任何引用链相连，则此对象时不可用的。

在 Java 中，可以当做 GC roots 的对象有以下几种：

- 栈帧中的本地变量表中引用的对象
- 方法区中的静态属性引用的对象
- 方法区中的常量引用的对象
- 本地方法栈中 JNI（一般说的Native方法）引用的对象

#### finalize()

即使在可达性分析算法中不可达的对象，也不是非死不可。认为一个对象死亡，至少要经历两次标记过程：如果在可达性分析后发现没有与 GC Roots 相连的引用链，那么会第一次标记并筛选，筛选的条件时此对象是否有必要执行 finalize() 方法。当对象没有覆盖 finalize() 方法，或者finalize() 方法已经被虚拟机调用过，虚拟机将这两种情况都视为 "没有必要执行"。

如果对象被判定有必要执行 finalize() 方法，那么这个对象将会放置在一个叫 F-Queue 的队列之中，并在稍后由 Finalizer 线程去执行。但是这里，虚拟机只是会触发这个方法并不承诺会等待它运行结束，这样做的原因是，如果一个对象在 finalize() 方法中执行缓慢，或者发生死循环，可能导致 F-Queue 队列中其他对象永久等待。

> 任何一个对象的 finalize() 方法都只会被系统自动调用一次

建议：尽量避免使用 finalize()，因为它不是 C/C++ 中的析构函数，而是 Java 刚诞生时为了使 C/C++ 程序员更容易接受它所做出的妥协。有些教材中描述它适合做 "关闭外部资源" 之类的工作，完全是对该方法用途的自我安慰。finalize() 能做的所有工作，使用 try-finally 或者其他方式都可以做的更好、更及时。所以建议忘掉该方法的存在。


#### 三色标记
将遍历对象图的过程中，对象分为三种颜色：

- 白色：表示对象尚未被垃圾回收器访问过。显然，在可达性分析刚开始的时候，所有对象都是白色的，若在分析结束的阶段，对象仍然是白色，表示不可达（垃圾）。
- 灰色：表示对象已经被垃圾回收器访问过，但这个对象上至少存在一个引用还没有被扫描过。灰色可能直接引用着白色对象。
- 黑色：表示对象以及它的所有引用都已经扫描过。黑色的对象代表存活的。如果有其他对象引用了黑色对象，也无需重新扫描一遍。黑色对象不可能 "直接" 指向某个白色对象。

#### 可达性分析的并发
GC Roots 相比起整个 Java 堆中的全部对象还算极少数，且在各种优化技巧（如 OopMap）的加持下，它带来的停顿已经是非常短暂且相对固定。但是，从 GC Roots 再继续往下遍历对象图，这个过程的停顿时间必定与 Java 堆容量成正比例关系。如果能够削减这部分停顿时间，收益将会是系统性的。

如果用户线程和回收器并发工作，用户线程也在修改引用关系，可能出现两种后果：

- 把原本死亡的对象错误标记为存活。这不是好事，但可以容忍，不过是产生了浮动垃圾。
- 把原本存活的对象标记为死亡。这样后果非常致命，出现 "对象消失" 问题，程序肯定会出错。

<span id="对象消失">对象消失</span>：1994 年，Wilson 在理论上证明了，当且仅当下面两个条件同时满足，才会产生 "对象消失" 的问题：

- 用户线程插入了一条或多条从黑色对象到白色对象的新引用
- 用户线程删除了全部从灰色对象到该白色对象的直接或间接引用

为了避免对象消失的现象，只需要破坏这两个条件任意一个即可。由此产生了两种解决方案：增量更新（Incremental Update）和原始快照（Snapshot At The Beginning，STAB）。

增量更新是破坏第一个条件，当黑色对象插入新的指向白色对象的引用关系时，就将这个新插入的引用记录下来，等并发扫描结束之后，再将这些记录过的引用关系中的黑色对象以跟，重新扫描一次。可以理解为，黑色对象一旦新插入了指向白色对象的引用关系之后，它就变成灰色对象了。

原始快照要破坏的是第二个条件，当灰色对象要删除指向白色对象的引用关系时，就将这个要删除的引用记录下来，在并发扫描结束之后，再以这些记录过的引用关系中的灰色对象为根，重新扫描一次。

> 虚拟机针对以上的记录操作都是通过写屏障实现的。在 HotSpot VM 中，CMS 时基于增量更新来做并发标记的，G1 是用原始快照来实现的。

## 垃圾收集器（G1）
### 堆布局
G1 将堆划分为一组大小相等的堆 Region（区域），每个区域都是连续的虚拟机内存范围，如下图所示。区域是内存分配和回收的单位。

<img src="https://img-blog.csdnimg.cn/img_convert/e9d5353d9062d194fc1edcf97a6179f8.png">

> - 用户可以随意设置 Region 的大小，但是内部会将用户的值向上调整为 2 的指数幂（$2^{n}$）；设置 G1 堆区域大小：`-XX:G1HeapRegionSize=n`，默认 n = 2M，可选值 1M、2M、4M、8M、16M、32M
> - 空闲区域是通过链表管理的

显然，在任何给定时间，每个区域可以是空的（灰色），或者年轻代，或者老年代。

年轻代区域可以再细分为两类：创建区域、存活区域。
- 创建区域：用来存放刚刚生成、一次也没有被转移过的对象。可以认为是 Eden。
- 存活区域：用来存放被转移过至少一次的对象。可以认为是 Survivor Space。

老年代区域可以再细分为两类：普通老年代、Humongous 区域
- 普通老年代：用来存放年轻达到阈值的对象
- Humongous 区域：用来存放巨大对象

当内存请求到来时，内存管理器会拿出空闲区域，将它们分配给某个代。

> 未必就直接分配给年轻代（创建区域），如果对象过大，则直接分配给老年代（Humongous 区域）。


在年轻代没有达到最大空间前，G1 GC 会根据 Java 应用对象分配速率，从空闲空间里面挑选出区域加入年轻代。当 Eden 区间分配内存失败，一次年轻代 GC 就被触发，它的工作是释放一些内存，属于一次轻量级操作。



随着年轻代回收，大对象分配等操作方式，越来越多的对象从年轻代进入到了老年代，年轻区域和老年区域可以同时被垃圾回收。这称为 *mixed collection*（混合回收）。

垃圾回收是一种压缩回收，它将存活对象拷贝到选定的、最初为空的区域。

> 跟 CMS 回收器不同，G1 对老年代的回收是压缩回收。

根据幸存对象的年龄，可以将对象复制到幸存区域（标记为 "S"）或者老年区域（图上暂未具体显示）。标有 "H" 的区域包含大于半个区域并经过特殊处理的巨大对象。

> - 与传统的连续堆内存不同，G1 的内存布局以 Region 进行划分。
> - "H" 区域存储那些大小 >= 0.5 倍 Region 的大型对象。



#### 标记位图
每个区域都带有两个标记位图：next 和 prev。"位" 即 bit。
- next：本次标记的标记位图
- prev：上次标记的标记位图，保存了上次标记的结果。


<img src="https://img-blog.csdnimg.cn/6ba5828839774d319ffd7b0612a679b4.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70#pic_center">


标记位图中每个 bit 都对应着关联区域内的对象的开头部分。

> - 之所以是开头部分，是因为一个对象如果占用多个 bit 位置，那么只标记它的起始 bit。
> - 黑色的地方表示比特值为 1（存活对象），白色的地方表示比特值是 0（带 "X" 的死亡对象）。

标记位图中 4 个指针的含义：
- bottom：众多对象的末尾。
- top：众多对象的开头。
- nextTAMS：next Top At Marking Start，标记开始时的 top，每次并发标记开始时会移动到与 top 相同的位置，下面会细说。
- prevTAMS：prev Top At Marking Start，上次标记开始时 top。

> 每 8 个字节对应标记位图中的 1 个比特。所以，上图中对象 C 虽然占用多个位置，但是 next 位图只记录了对象头所在的位置。


### 垃圾回收周期
G1 回收器在两个阶段之间交替。在 young-only 阶段包含垃圾回收，即逐渐用老年代中的对象填充当前可用内存。在 space-reclamation 阶段，G1 除了处理年轻代之外，还逐步回收老年代的空间。然后，周期从 young-only 重新开始。

- Young-only 阶段：这个阶段从一些 young-only 集合开始，将对象晋升到老年代。当老年代占用达到某个阈值（即初始堆占用阈值）时，将会从 young-only 阶段过渡到 space-reclamation 阶段。此时，G1 调度 initial mark（初始标记） young-only 集合，而不是一个常规的 young-only 收集了。
	- Initial Mark：除了执行一个常规的 young-only 回收之外，这种回收还开始标记过程。并发标记确定所有当前老年代区域可达的存活对象，以便保留给接下来的 space-reclamation 阶段。当标记还没完全结束的时候，可能发生常规的年轻回收。标记结束之后，随之而来的是两个 stop-the-world 阶段：Remark 和 Cleanup。
	- Remark：此暂停完成标记本身，并执行全局引用处理和类卸载。在 Remark 和 Cleanup 之间 G1 会并发地计算一个存活信息的摘要，该摘要确定之后会在 Cleanup 暂停中使用来更新内部数据结构。
	- Cleanup：此暂停还会回收完全空的区域，并确定是否 space-reclamation 阶段会真的到来。如果接下来是 space-reclamation 阶段，young-only 阶段以单个 young-only 回收完成。

- Space-reclamation 阶段：这个阶段由多个 mixed collections 组成，除了年轻代区域外，还转移老年代区域的存活对象。当 G1 确定回收更多的老年代区域不会产生足够的可用空间时，space-reclamation 阶段结束。

在 space-reclamation 之后，回收周期从另一个 young-only 阶段重新开始。作为后备，如果应用程序在收集存活信息时内存不足，G1 会像其他回收器一样执行就地的 STW 全堆压缩（Full GC）。

### G1 总体执行过程
G1GC 主要有以下两个功能：

- 并发标记（concurrent marking）
- 转移（evacuation），也有人叫疏散

并发标记：基本能和 mutator 并发执行，会针对区域内所有存活的对象进行标记。

转移：选择区域，如果该区域有存活对象，则将它们复制到其他空闲区域中。对象复制完成之后，只剩下死亡对象的区域会被重置（回收）为空闲区域以便复用。

> 并发标记和转移在处理上是相互独立的。并发标记的结果信息对转移来说并不是必须的。因此，转移处理可能发生在并发标记开始之前，也可能发生在并发标记的过程中。


### 并发标记
并发标记包括以下 5 个步骤：

① 初始标记阶段
② 并发标记阶段
③ 最终标记阶段
④ 存活对象计数
⑤ 收尾工作

##### 步骤 ① 初始标记阶段
该阶段有两个过程：

- 创建标记位图：与 mutator 并发执行
- 根扫描：标记可由根 "直接引用" 的对象。

<img src="https://img-blog.csdnimg.cn/a41c527ed0d047c990003c0668db53f7.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70">

在初始标记阶段，GC 线程首先会创建标记位图 next。指针 nextTAMS 指向标记开始时 top 所在的位置。在创建位图时，其大小也和 top 对齐，为 (top - bottom) / 8 字节。

> 上述处理都是和 mutator 并发进行的。

等所有区域的标记位图都创建完成之后，就可以开始进行根扫描了。

> 根扫描，指可由根 "直接引用" 的对象进行标记的过程。

为了防止在根扫描的过程中根被修改，在这个过程中 mutator 是暂停执行的。

> 虽然 G1GC 采用的写屏障可以获知对象的修改，但是大多数根不是对象。而且，根需要频繁的修改，与其频繁地通过写屏障去获取修改，不如直接暂停 mutator 进行根扫描性能更好。

如果一个对象本身被标记了，但其子对象并没有被扫描，称它为未扫描对象（灰色）。上图中，对象 C 已经在根扫描中被标记，但由于根扫描不会扫描子对象，对象 C 又持有对象 A 和 E 的引用，所以对象 C 被认为是未扫描对象，表示为灰色。

完成根扫描后，mutator 会再次开启执行。


##### 步骤 ② 并发标记阶段
在并发标记阶段，GC 线程继续扫描在初始标记阶段被标记过的对象（根直接引用对象），完成堆大部分存活对象的标记。

下图表示并发标记阶段结束后的区域状态。对象 C 和子对象 A 和 E 都被标记了。

<img src="https://img-blog.csdnimg.cn/d5287491ab114c0299b26f75041a9f4c.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70">


并发标记阶段地一个重要特点是 GC 线程和 mutator 是并发执行的。因此，mutator 在执行的过程中可会会改变对象之间的引用关系，所以，采用一般的标记方法，可能会发生 "标记遗漏"。因此，必须使用写屏障技术来记录对象间引用关系的变化。

###### SATB
SATB（Snapshot At The Beginning，初始快照）是一种将并发标记阶段开始时对象间的引用关系，以逻辑快照的形式进行保存的手段。

在 SATB 中，标记过程中新生成的对象会被看作 "已完成扫描和标记"，因此其子对象不会被标记。

> - 因为 SATB 记录的是并发标记阶段开始时的对象间引用关系，而标记过程中新生成的对象与其他对象间的引用关系在标记开始时并不存在，因此其子对象不会被标记。
> - 如上图所示，nextTAMS 和 top 之间的对象 J 和 K 就是在标记过程中新生成的对象。因为它们的引用关系在标记开始时并不存在，所以它们都会被当成存活对象。
> - 此外，标记位图也不会记录对象 J 和 K

如果在并发标记的过程中对象的域发生了写操作（被修改），就必须以某种方式记录下被改写之前的引用关系。

<img src="https://img-blog.csdnimg.cn/832f67c0e19a42d396171a3d75112cf6.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70">

参数 field 表示被写入对象的域（类属性），参数 newobj 表示被写入域的值。

第 2 行的 GC_CONCURRENT_MARK 用来表示并发标记阶段的标志位（flag）。

第 4 行会检查当前处于并发标记且被写入之前 field 的域的值是不是 Null，因为如果为 Null，代表之前的引用关系并不存在，无需记录。

第 5 行将 oldobj 添加到 $current_thread.stab_local_queue 中。

第 7 行进行实际的写入操作。

在并发标记阶段，GC 线程会定期检查 SATB 队列集合大小。如果发生其中由队列，则会对队列中的全部对象进行标记和扫描。

SATB 本地队列在装满（默认大小为 1 KB）之后，会被添加到全局的 SATB 队列集合中。这些被添加的 SATB 本地队列，都是并发标记阶段的待标记对象。


##### 步骤 ③ 最终标记阶段
最终标记阶段的处理是暂停处理，需要暂停 mutator 的运行。

> 因为 SATB 本地队列中的数据会被 mutator 操作，所以不能和 mutator 并发执行。

因为未装满的 SATB 本地队列不会被添加到 SATB 队列集合中，所以在并发标记阶段结束后，各个线程 SATB 本地队列中可能仍然存在待扫描的对象。而最终标记阶段就会扫描这些 "残留的 SATB 本地队列"。

队列中保存了对象 G 和 H 的引用。因此在扫描 SATB 本地队列之后，对象 G 和 H，以及对象 H 的子对象都会被标记。
<img src="https://img-blog.csdnimg.cn/fc57958090ae4219aef11f68b652956c.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70">
本步骤结束后，所有的存活对象都已被标记。因此，此时所有不带标记的对象都可以判定为死亡对象。


##### 步骤 ④ 存活对象计数
计数处理和 mutator 是并发执行的。

> 不过，计数过程中操作的对象可能会被转移的记忆集合线程使用，因此需要先停掉记忆集合线程。

这个步骤会扫描各个区域的标记位图 next，统计区域内存活对象的字节数，然后将其存入 Region 内的 next_marked_bytes 中。
![在这里插入图片描述](https://img-blog.csdnimg.cn/53c8eba0bcf64133aad67ab245d28b27.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70)
> - 在计数过程中新创建了对象 L 和 M。由于 nextTAMS 和 top 之间的对象都会被当做存活对象来处理，所以不会特意计数。
> - prev_marked_bytes 存放了上次标记结束时存活对象的字节数。因为该 Region 在此之前未曾进行过标记，因此值为 0。

##### 步骤 ⑤ 收尾工作
收尾工作所操作的数据有些是和 mutator 共享的，因此需要暂停 mutator 的运行。

在此期间，GC 线程会逐个扫描每个区域，将标记位图 next 中的并发标记结果移动到标记位图 prev 中，再对并发标记中使用过的标记值进行重置，为下次并发标记做好准备。

此外，对没有存活对象的区域进行回收的工作。可以理解为以区域为单位的清除。

在扫描过程中还会计算每个区域的转移效率，并按照该效率对区域进行降序排序。

![在这里插入图片描述](https://img-blog.csdnimg.cn/00fa127e22dc49daab598e43cb302225.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70)
收尾工作结束后的几个状态变化：

- next.next_marked_bytes 中的值被转移到 prev.prev_marked_bytes
- prevTAMS 被移动到 nextTAMS 之前的位置
- next.next_marked_bytes 被重置
- nextTAMS 移动到 bottom 的位置，nextTAMS 会在下次并发标记开始时，移动到 top 的最新位置

收尾工作结束后，整个并发标记结束。并发标记线程一直处于等待状态，直到下次并发标记开始。


##### 总结
并发标记结束后，转移处理可以得到以下信息

① 并发标记完成时存活对象和死亡对象的区分（标记位图 prev）

② 存活对象的字节数（prev_marked_bytes）

这些信息在并发标记阶段不会被改变，因此，即使并发标记阶段开始了，进行转移处理也没有问题。

### 转移
通过转移，所选区域内的所有存活对象都会被转移到空闲区域。这样一来，被转移区域就只剩下死亡对象。重置之后，该区域就会称为空闲区域，能够再次利用。

下图表示了转移开始前后的状态。转移结束后，可从 GC Root 到达的存活对象 a、b、c 会被转移到空闲区域 C，而死亡对象 d、e 不会被转移，整个区域 A、B 会被重置以再次利用。


<img src="https://img-blog.csdnimg.cn/683d2548395e4e6f8319036f7bc699da.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70">

#### 转移专用记忆集合
除了可以从根和并发标记的结果发现存活对象之外，转移功能还可以通过转移专用记忆集合来发现对象。转移专用记忆集合用来记录区域之间的引用关系。

> 对比：SATB 队列集合主要用来标记过程中对象之间引用关系的变化

为了简化表达，下面可能会使用 Remember Set（或者 RSet）表示转移专用记忆集合
 
<img src="https://img-blog.csdnimg.cn/1979c2328b354ada953f7a27a12ef4d5.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70">

> - 每个区域都有自己的 RSet，RSet 中记录了来自其他区域的引用，无需扫描所有区域内的对象，就可以确定待转移对象所在区域内的对象被其他区域引用的对象，从而简化单个区域的转移处理。
> - G1GC 是通过卡表（Card Table）来实现转移专用记忆集合的。


##### 卡表

卡表是一个字节数组，如下图。卡表里的元素称为卡片。堆中一定大小的存储空间（卡页）会对应卡表中的一个元素（字节 / 卡片）。

堆中的对象所对应的卡片在卡表中的索引值可以通过以下公式快速计算出：
<center><code>（对象的地址 - 堆的头部地址）/ 512</code></center>

<img src="https://img-blog.csdnimg.cn/5a626af8bee0454b9b759a93413393e2.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70">

> 一般来说，卡页的大小都是 2 的 N 次幂。HotSpot 的卡页大小为 2 的 9 次幂，即 512 字节。因此，如果堆大小为 1 GB 时，那么被分为 2 M 个卡页，卡表的数组长度为 2 M，大小为 2 MB。




##### 转移专用记忆集合的构造

<img src="https://img-blog.csdnimg.cn/cd194a4f9f1c491ca4d66d51461d5333.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70">

> RSet 是通过散列表实现的，可以理解为 Java 的 Map，key 是区域的地址，值是卡片索引的数组。图中对象 b 引用了对象 a，因此对象 b 对应卡片的索引（2048）就被记录在区域 A 的转移专用记忆集合中。


##### 转移专用写屏障
当对象的域被修改时，被修改对象所对应的卡片会被转移专用写屏障记录到 RSet 中。


每个 mutator 线程都持有一个名为转移专用记忆集合日志的缓冲区，其中存放的是卡片索引的数组。当对象 b 的域被修改时，写屏障就会感知，并会将对象 b 所对应的卡片索引添加到转移专用记忆集合日志中。
![在这里插入图片描述](https://img-blog.csdnimg.cn/6d98bdcc19a14ca6a130537c98232093.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70)
当转移专用记忆集合日志写满之后，它会被添加到转移专用集合日志的集合（可以认为是 Set<转移专用集合日志>）中。

#### 转移总体步骤
① 选择回收集合：参考并发标记提供的信息来选择被转移的区域。被选中的区域称为回收集合（Collection Set）。

② 根转移：将回收集合内由根直接引用的对象，以及被其他区域引用的对象转移到空闲区域。

③ 转移：以 ② 中转移的对象为起点，扫描其子孙对象，将所有存活对象一并转移。

当 ③ 结束之后，回收集合内的所有存活对象就转移完成了。

这 3 个步骤都是暂停处理的。在转移开始后，即使并发标记正在进行也会先中断，而有限进行转义处理。

另外，② 和 ③ 都是可以多线程并行执行的。

#### 步骤 ① 选择回收集合
选择回收集合的标准简单来说有 2 个：

- 转移效率高
- 转移的预测暂停时间在用户的容忍范围内

在选择回收集合时，堆中的区域按照转移效率降序排列。接下来，按照排好的顺序依次计算各个区域的预测暂停时间，并选择回收集合。当所有已选区域预测暂停时间的总和快要超过用户的容忍范围时，后续选择就会停止。

#### 步骤 ② 根转移
根转移的转移对象包含以下 3 类：

- GC Root 直接引用对象
- 并发标记处理中的对象
- 由其他区域对象直接引用的回收集合内的对象

根转移伪代码：

<img src="https://img-blog.csdnimg.cn/f8a858d64c9b43779c98c585cd055499.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70">


代码清单第 2 ~ 4 行先是把被根引用的位于回收集合内的对象转移到其他的空闲区域。

> 被根引用却不在回收集合内的对象会被直接忽略。
> $roots 中还包含 SATB 本地队列和 SATB 队列集合中的引用

第 4 行的 evacuate_obj() 是用于转移对象的函数，它的返回值是转移后对象的地址。

第 6 行中的 force_update_rs() 的作用是将未被转移专用记忆集合维护线程扫描的脏卡片更新到转移专用记忆集合中。

##### 对象转移
<img src="https://img-blog.csdnimg.cn/3271341455034b00a534ea6ce3eabfd4.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70">

① 将对象 a 转移到空闲区域
② 将对象 a 在空闲区域中的新地址写入到转移前所在区域中的旧地址。
③ 将对象 a 引用的所有位于回收集合内的对象都添加到转移队列中。转移队列是用来临时保存待转移对象的引用放。图中 a'.field1 引用了对象 b，而且 b 所在的区域在回收集合中。因为 a' 是存活对象，所以 a' 引用的对象 b 也是存活对象。
④ 针对对象 a 引用的位于回收集合外的对象，更新转移专用记忆集合。图中 a'.field2 引用了对象 c，而 c 所在的区域不在回收集合中。c 所在的转移专用记忆集合中虽然记录了 a.field2 对应的卡片，但是 a 被转移到了 a'，所以有必要更新转移专用记忆集合。
⑤ 针对对象 a 的引用放，更新转移专用记忆集合。对象转移时只有 1 个引用放能够以参数的形式进行传递。图中 a 的引用方是 d.field1。d.field1 指向的是 a 的地址，但是 a 被转移到 a'，所以有必要让 d.field1 指向 a 的新地址 a'。如图中所示，d.field1 对应的卡片被添加到了 a' 所在区域的转移专用记忆集合中。
⑥ 这一步并非转移的处理内容，只是补充说明。对象转移最终返回额是转移后的地址。在调用转移的地方，返回的地址会被赋值给引用放。图中 d.field1 的地址被替换成了对象 a' 的地址

#### 步骤 ③ 转移
完成根转移之后，那些被转移队列引用的对象会依次转移。直到转移队列被清空，转移就完成。至此，回收集合内所有的存活对象都被成功转移。


#### Card Tables and Concurrent Phases
如果垃圾回收器没有回收整个堆（增量回收），则垃圾回收器需要知道从堆的未回收的部分到正在回收的堆的部分的指针在哪。这通常用于分代垃圾回收器，其中，堆的未回收部分通常是老年代，而堆的回收部分是年轻代。保存这些信息的数据结构（指向年轻代对象的老年代指针）是一个 Remembered Set（记忆集）。Card Table（卡表）是一种特殊类型的记忆集。Java HotSpot VM 使用字节数组作为卡表。每个字节称为一张卡。一张卡对应于堆中的一系列地址。Dirtying a card（脏卡）意思是将字节的值更改为一个 dirty value（脏值）。一个脏值可能包含一个从老年代到年轻代，卡所覆盖地址的指针。

Processing a card 意味着检查卡片，看看是否有一个老年代到年轻代的指针，并且可能对这些信息做一些事情，比如将其转移到另一个数据结构。




##### 热卡片
频繁发生修改的存储空间所对应的卡片称为热卡片（hot card）。热卡片可能会多次被转移专用记忆集合线程处理成脏卡片，从而加重转移专用记忆集合线程的负担，因此需要特别处理。

要想发现热卡片，需要用到卡片计数器，它记录了卡片变成脏卡片的次数。卡片计数器记录了自上次转移以来哪些卡片变成了脏卡片，以及变成脏卡片的次数，其内容会在下次转移时被清空。

变成脏卡片的次数超过阈值（默认是 4）的卡片会被当成热卡片，在被处理为脏卡片后添加到热队列尾部。热队列的大小是固定的（默认是 1 KB）。如果队列满了，则从队列头部取出老卡片，给新的卡片腾出位置。取出的卡片由转移专用记忆集合维护线程当成普通卡片处理。

热队列的卡片不会被转移专用记忆集合维护线程处理，因为即使处理了，它也有可能马上又变成脏卡片。因此，热队列中的卡片会被留到转移的时候再处理。




#### 分代 G1GC 模式
G1GC 中存在 "纯 G1GC 模式"（pure garbage-first mode）和 "分代 G1GC 模式"（generational garbage-first mode） 两种模式。实际上，OpenJDK 虽然实现了纯 G1GC 模式，但是并没有将这种模式开放给用户。用户们使用的都是分代 G1GC 模式。

分代 G1GC 模式也分为新生代 GC 和老年代 GC。G1GC 中的新生代 GC 称为完全新生代 GC（fully-young collection），老年代 GC 称为部分新生代 GC（partially-young collection）。

完全新生代 GC 和部分新生代 GC 的主要区别在于回收集合的选择：

- 完全新生代 GC 将所有新生代区域选入回收集合
- 部分新生代 GC 将所有新生代区域，以及一部分老年代区域选入回收集合

> 注意：所有新生代区域都会被选入回收集合


##### 完全新生代 GC 的执行过程
完全新生代 GC 不会选择老年代区域，而是将所有新生代区域（创建区域、存活区域）都选入回收集合，然后转移回收集合内的存活对象。晋升的对象会被转移到老年代，其余对象则被转移到存活区域。

<img src="https://img-blog.csdnimg.cn/cd986d759bca4581819bf99db59e87ed.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70">

<center>完全新生代 GC 的执行过程</center>

部分新生代 GC 则是除了所有新生代区域外，还会选择一部分老年代区域进入回收集合。除了回收集合的选择方式，部分新生代 GC 和完全新生代 GC 的执行过程是一样的。

<img src="https://img-blog.csdnimg.cn/8f49e4c49f164e299135fb912a6fc10d.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70">

<center>部分新生代 GC 的执行过程</center>


### [Allocation Evacuation Failure](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html#allocation_evacuation_failure)
与 CMS 一样，G1 与应用程序并发执行，并且存在应用程序分配对象的速度比垃圾回收器恢复可用空间速度快的风险。

> 这种会导致回收过程中反而内存扩大

在 G1 中，Java 堆耗尽的问题可能发生在 G1 将存活数据从一个区域复制到另一个区域的过程中。如果在垃圾回收区域的疏散期间无法找到闲置的区域，则会发生分配失败，并且将会执行一个 stop-the-world 的 full collection。

### 浮动垃圾
对象可能会在 G1 回收期间死亡，且不会被回收。G1 使用一种称为 snapshot-at-the-beginning (SATB) 的技术来保证垃圾回收器找到所有存活对象。SATB 声明任何在并发标记（整个堆上的标记）开始的时候处于存活状态的对象都被认为是存活的，以便回收。SATB 允许浮动垃圾，以类似于 CMS 增量更新的方式。

### Pauses
G1 暂停应用程序以将活动对象复制到新区域。这些暂停可以是仅回收年轻区域的年轻回收暂停，也可以是疏散年轻和老年区域的混合回收暂停。与 CMS 一样，当应用程序停止时，有一个最终标记或注释暂停来完成标记。CMS 也有初始标记暂停，而 G1 将初始标记工作作为疏散暂停的一部分。G1 在回收结束时有一个清理阶段，它是部分 STW，部分并发的。清理阶段的 STW 部分识别空区域，并确定作为下一次回收候选的老年区域。