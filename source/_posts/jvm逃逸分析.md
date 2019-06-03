---
title: jvm逃逸分析
date: 2019-04-15 12:14:49
tags: 
- jvm
categories: java
---
逃逸分析(Escape Analysis)是JIT中常用的一种技术,其基本行为就是动态分析对象的作用域,之后做出相应的优化
开启逃逸分析的jvm参数：`-XX:+DoEscapeAnalysis`

### 方法逃逸
一个在方法内创建的对象，如果会在方法外被访问，那么就称这个对象存住方法逃逸。
方法逃逸可以以返回值、field属性、对象作为参数传入等形式发生，下面这个例子就是不存在方法逃逸，StringBuffer对象在testEscape方法中创建且对象本身不会被返回。
```java
public String testEscape() {
        StringBuffer stringBuffer = new StringBuffer();
        stringBuffer.append("a");
        return stringBuffer.toString();
    }
```
### 栈上分配
如果一个对象不存在方法逃逸，即说明这个对象不会在该线程栈外被访问。那这个对象就可能会分配在栈上，而不是分配在堆上。随着线程结束栈空间的清除而清除。这样节省了堆上的空间，减少了GC回收的压力。

不过查了下资料很多人说Hotspot现在没有实现栈上分配

### 同步消除

另一个体现了逃逸分析的功能就是同步消除，上面的例子中StringBuffer的append方法是加了锁的。如果jit发现某加锁代码块中的对象不存在方法逃逸，那就可能会消除该锁。启用同步消除需要配置额外参数：
```
 -XX:-EliminateLocks
```
### 标量替换

若一个对象被证明不会被外部访问，并且这个对象可以被拆解成若干个基本类型的形式，那么当程序真正执行的时候可以不创建这个对象，而是采用直接创建它的若干个被这个方法所使用到的成员变量来代替，将对象拆分后，除了可以让对象的成员变量在栈上分配和读写之外，还可以为后续进一步的优化手段创造条件。

开启参数
```
-XX:+EliminateAllocation
```
