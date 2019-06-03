---
title: CMS和G1的一些细节
date: 2019-04-16 22:04:26
tags: 
- jvm
- gc
categories: java
---

最近看了下关于CMS和G1的一些文章，真的很蛋疼，很多文章你抄我我抄他,抄的到处都是错。或者就扔几个粗粒度的概念在那，真的很浪费查资料的人的时间。

花了一天时间看了下`RednaxelaFX`大神的问答帖和其他资料整理了下CMS和G1的一些实现细节。基本概念以及搜索引擎一搜一大堆的东西就不在这里赘述了。想到哪写到哪 没什么顺序比较乱。

### CMS初始标记（initial marking）阶段的gc root到底都有什么？

> 在CMS initial mark的上下文里，根集合并不包括young gen而是只有stack、register、globals这些常规的。这是因为在接下来的CMS concurrent mark阶段CMS会顺着初始的根集合把young gen里的活对象都遍历了。所以从CMS initial mark + concurrent mark结合在一起的角度看，young gen仍然是根集合的一部分（因为被扫描但不被收集）。

也就是说实际初始标记阶段的gc root set是不包括新生代的 后来在并发标记阶段会去遍历(确切的说是 在预清理阶段）。不过在remark阶段还是会重新遍历新生代的，原因和老年代的dirty card一样，新生代在执行时也会存在这种动态的dirty card.

### G1 region图示

> 摘自美团技术博客https://tech.meituan.com/2016/09/23/g1.html

![G1](https://gitee.com/neusnail/img/raw/master/blogimg/G1.png)

H表示这些Region存储的是巨大对象（humongous object，H-obj），即大小大于等于region一半的对象,H-obj直接分配到了old gen，防止了反复拷贝移动。

### 什么情况下会发生漏标？

在CMS和G1中我们使用黑白灰三种颜色标识扫描的对象：

- 白：对象没有被标记到，标记阶段结束后，会被当做垃圾回收掉。

- 灰：对象被标记了，但是它的field还没有被标记或标记完。

- 黑：对象被标记了，且它的所有field也被标记完了。

而要发生漏标，下面两个条件必须同时满足：

1、应用程序插入了一个从黑色对象到该白色对象的新引用

2、应用程序删除了所有从灰色对象到该白色对象的直接或者间接引用。

插入一个黑色到白色对象的新引用意味着一个原本要当做垃圾被回收的白对象有了新引用，变成可达的了，而因为是黑对象产生的引用，黑对象不会再重新扫描计算所以该白对象会被漏掉，但是如果该白对象还被灰对象引用的话那还可以抢救一下，因为灰对象还没有标记完，接下来还会从灰对象扫描到白对象。

### 使用dirty card解决漏标问题

 JVM将内存分成一个个固定大小的`card`，然后有一个专门的数据结构(即这里的`Card Table`)维护每个`Card`的状态，一个字节对应一个`Card`。当一个`Card`上的对象的引用发生变化的时候，就将这个`Card`对应的`Card Table`上的状态置为dirty(实际上使用的是mod-union table)。之后在预清理和remark阶段会处理这些dirty card

##### CMS和G1标记dirty card区别是什么？

* CMS使用的是**incremental update** write barrier，也叫做insertion barrier,关注的是新引用关系的插入
* G1使用的是**SATB** write barrier（Snapshot-At-The-Beginning）也叫做deletion barrier 关注的是旧引用关系的删除

> Incremental update的做法是：只要在write barrier里发现要有一个白对象的引用被赋值到一个黑对象的字段里，那就把这个白对象变成灰色的（例如说标记并压到marking stack上，或者是记录在类似mod-union table里）。这样就强力杜绝了上述第一种情况的发生。 
>
> SATB的做法是：把marking开始时的逻辑快照里所有的活对象都看作时活的。具体做法是在write barrier里把所有旧的引用所指向的对象都变成非白的（已经黑灰就不用管，还是白的就变成灰的）。 
> 这样做的实际效果是：如果一个灰对象的字段原本指向一个白对象，但在concurrent marker能扫描到这个字段之前，这个字段被赋上了别的值（例如说null），那么这个字段跟白对象之间的关联就被切断了。SATB write barrier保证在这种切断发生之前就把字段原本引用的对象变灰，从而杜绝了上述第二种情况的发生。 

###  CMS和G1的浮动垃圾(Floating Garbage）
CMS只关注新引用关系的插入，不关心删除。所以当一个黑色对象的引用被删除时CMS感知不到，还会把他当初一个已标记对象，导致一个垃圾对象不会在本轮删除。

而G1只关注删除，会把inital marking 之后所有new出来的对象都当做黑对象来处理（原有对象单纯新增引用不会产生浮动垃圾）,而这些对象在concurrent marking时是否会变成垃圾G1是不关心的。所以G1一次GC中产生的浮动垃圾可能比CMS要多。

### CMS的Precleaning（预清理）和AbortablePreclean（可中断的预清理）

有一点要清楚就是预清理和可中断预清理这两个阶段并不是必须的，可以通过参数XX:-CMSPrecleaningEnabled关闭。而AbortablePreclean循环执行的前提是Eden区的内存使用量大于CMSScheduleRemarkEdenSizeThreshold(默认2M）如果新生代的对象太少，就没有必要执行该阶段，直接执行重新标记阶段。预清理阶段主要做两个事1：扫描young gen处理动态产生的新的对old gen的引用 2：处理之前标记的dirty card。而可中断预清理和预清理一样只不过是循环执行的，并具备以下几个中断条件:

* 循环次数超过最多循环的次数 CMSMaxAbortablePrecleanLoops，默认是0，表示没有循环次数的限制。

* 执行循环的时间达到了阈值 CMSMaxAbortablePrecleanTime，默认是5s。

* 新生代Eden区的内存使用率达到了阈值CMSScheduleRemarkEdenPenetration(默认50%)

预清理阶段是并行执行的，目的就是为了减轻remark阶段的压力。循环预清理在5秒内期待一次Young GC，这样在remark阶段就会大幅减少扫描young gen带来的损耗。不过我们也可以通过CMSScavengeBeforeRemark参数指定在AbortablePreclean前执行一次GC。

美团点评有一篇关于CMS优化的文章说使用这个参数使AbortablePreclean提前退出循环，我感觉表述有问题，应该是通过开启这个参数使得Eden区使用量小于CMSScheduleRemarkEdenSizeThreshold导致跳过AbortablePreclean阶段。

### G1的CSet和RSet

##### Collection Set（CSet）

CSet记录了某次GC要收集的Region集合，集合里的Region可以是任意年代的。GC中指向CSet外的引用都是可以忽略的，每次GC只关心本次CSet中的region

##### remembered set

逻辑上说每个Region都有一个RSet，RSet记录了其他Region中的对象引用本Region中对象的关系，属于points-into结构（谁引用了我的对象）Rset是基于card table实现的。比如一个A对象引用了B对象，即A.filed=B。那么B的RSet中就会记录**A对象所在的Card**的地址。


> 图片摘自美团技术博客https://tech.meituan.com/2016/09/23/g1.html

![RsetAndCardTable](https://gitee.com/neusnail/img/raw/master/blogimg/RsetAndCardTable.jpg)

### G1的inital marking阶段和Young gc共用了什么？

一次global concurrent marking的inital marking是和young gc共用了暂停阶段，实际上共用了young gc scan出来的根集合。一个young gc可以顺带做inital marking 也可以不带。

### 为什么G1不关心Young gen的对象引用关系的更新？

分代式G1模式下有两种选定CSet的子模式，分别对应young GC与mixed GC： 
* Young GC：选定所有young gen里的region。通过控制young gen的region个数来控制young GC的开销。 

* Mixed GC：选定所有young gen里的region，外加根据global concurrent marking统计得出收集收益高的若干old gen region。在用户指定的开销目标范围内尽可能选择收益高的old gen region。

   

  可以看到**young gen region总是在CSet内**。而G1 gc时关心的是Cset外对Cset内的引用因此分代式G1不维护从young gen region出发的引用涉及的RSet更新。而CMS是只回收old gen的所以CMS的“CSet”是old gen当然要关心young gen的引用变更

### G1的Evacuation过程

evacuation过程就是清理过程，该过程和global concurrent marking是两个相对独立的过程.Evacuation阶段是全暂停的。它负责把一部分region里的活对象拷贝到空region里去，然后回收原本的region的空间。 evacuation过程可以选择任意多个Cset来进行回收，因为每个region都有对应的Rset ，通过Rset就可以判断那些对象是存活的。

### G1对比CMS的优势在哪？

* G1虽然会mark整个堆，但并不evacuate所有有活对象的region；通过只选择收益高的少量region来evacuate，这种暂停的开销就可以（在一定范围内）可控。但是毕竟要暂停来拷贝对象，这个暂停时间再怎么低也有限。G1的evacuation pause在几十到一百甚至两百毫秒都很正常。所以切记不要把 -XX:MaxGCPauseMillis 设得太低，不然G1跟不上目标就容易导致垃圾堆积，反而更容易引发full GC而降低性能。通常设到100ms、250ms之类的都可能是合理的。
* 而CMS的remark需要重新扫描mod-union table里的dirty card外加整个根集合，而此时整个young gen（不管对象死活）都会被当作根集合的一部分，因而CMS remark有可能会非常慢。如果在CMS的并发标记阶段，mutator仍然在高速分配内存使得young gen里有很多对象的话，那remark阶段就可能会有很长时间的暂停。Young gen越大，CMS remark暂停时间就有可能越长。而G1的finial marking(或者成为remarking)阶段只需要扫描SATB buffer
* CMS使用清除的方式回收，会使内存碎片化。而G1d的evacuation阶段是使用复制算法，不会有碎片