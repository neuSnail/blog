---
title: java引用类型与内存泄露
date: 2019-04-13 13:23:03
tags: 
-  java
categories: java
---

### 强引用(StrongReference)
强引用是最常见的引用，下面例子中使用new关键字以及赋值操作创建出来的引用都是强引用
```java
//a 是ObjectA的强引用
ObjectA a = new ObjectA();
//b 是ObjectA的强引用
ObjectA b = a;
```
强引用不会被垃圾回收器回收，当内存空间不足时，jvm宁愿抛出oom使进城终止也不会随意回收具有强引用的对象。把一个对象的引用显示的指向Null或者其他对象可以取消该对象的强引用，帮助虚拟机在下次gc时回收。但是上面的例子即使把a赋值为null`a=null`ObjectA也不会被gc回收，因为b还持有ObjectA的引用，此时我们称ObjectA仍是可达的。这种特性使得我们在开发时可能会造成内存泄露的可能，具体会在后文分析。
### 弱引用(WeakReference)
弱引用的生命周期比较短暂，在垃圾收集器扫描时如果发现了**只有**弱引用的对象就会将其回收。
```java
class ObjectA {
        @Override
        protected void finalize() throws Throwable {
            System.out.println("object destruct");
            super.finalize();
        }
    }
@Test
 public void testReference() {
        Integer a = new ObjectA();
        WeakReference b = new WeakReference<>(a);
        a = null;
        System.gc();
        System.out.println(b.get());
    }
    
```
输出：
```
null
object destruct
```
上面的例子中b是a的一个弱引用，此时ObjectA有两个引用，一个强引用a和一个弱引用b，将a置为null,此时ObjectA只有一个弱引用，之后执行gc会将弱引用回收。

如果这里没有把a置为null，gc是不会把弱引用回收的。
#### referenceQueue

在创建弱引用时，构造函数提供了一个重载可以传入一个ReferenceQueue对象

```java
public WeakReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
```

当一个对象被GC回收掉时，指向他的引用类型会被放入这个referenceQueue中，比如上面的例子如果使用referenceQueue的话：

```java
@Test
    public void testReference() {
        ObjectA a = new ObjectA();
        //创建一个队列
        ReferenceQueue<ObjectA> queue = new ReferenceQueue<>();
        //创建弱引用并传入队列
        WeakReference b = new WeakReference<>(a,queue);
        a = null;
        System.gc();
        System.out.println(b.get());
        try {
        	//阻塞的从队列中获取值
            System.out.println(queue.remove());
        } catch (InterruptedException e) {

        }
    }
```
输出：
```
null
object destruct
java.lang.ref.WeakReference@7b1d7fff
```
可以看到再ObjectA被GC掉之后会将弱引用B放入queue中。不止是弱引用，下面的软引用和虚引用都可以使用referenceQueue.


### 软引用(SoftReference)

软引用的生命力比弱引用强一些，当一个对象只有软引用时，若内存空间足够则不会被回收。如果内存空间不够则对象会被系统回收。

软引用比较适合做缓存，当创建的缓存较多导致内存不够时可以回收这些软引用。

### 虚引用(PhantomReference)

虚引用和前面几种引用不同，它不会决定对象的生命周期。如果一个对象只有虚引用，那它和没有引用一样会被回收。虚引用只能和ReferenceQueue联合使用，用来追踪对象被GC回收的活动


```java
//PhantomReference在任何时候调用get方法都只会返回null
 public T get() {
        return null;
    }
    
//PhantomReference仅有的一个构造函数，必须传入ReferenceQueue
public PhantomReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
```

### 内存泄露
所谓内存泄露就是指某些对象已经不使用了但是还拥有强引用导致垃圾回收器无法回收这些内存造成内存的浪费。
内存泄露最常见的情景就是在使用集合时，比如hashMap，List等。尤其是在静态类和单例模式中使用时

在下面的示例中testMemoryLeak()方法向objectRepo中插入了一条数据，然后使用repoMap执行一些逻辑，最后objectKey置为null。这里其实不用显示把key置为null，因为函数栈执行完了objectKey的生命周期就结束了，会自动清除栈里的局部变量。
但是ObjectKey("keyA")不会被GC回收，因为hashMa的entry里还持有ObjectKey("keyA")的强引用。而这个objectKey在接下来不会再使用但占据的空间又不会被释放，便造成了内存的浪费。

```java
 	private Map<ObjectKey, String> objectRepo = new HashMap<>();

    public void testMemoryLeak() {
        ObjectKey objectKey = new ObjectKey("keyA");
        objectRepo.put(objectKey, "valueA");
        //使用仓储map执行某些逻辑
        useObjectRepoDoSth();
        objectKey = null;
    }
```
#### WeakHashMap

为了解决这个问题,java提供了WeakHashMap。weakHashMap中的Entry继承了WeakReference,可以看到在weakHashMap的构造函数中，创建了一个key的弱引用并将ReferenceQueue传入（该queue被WeakReference对象持有)。当key对象的外部不存在强引用时如果触发GC，该key的弱引用会被添加到referenceQueue中。

```java
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
        V value;
        final int hash;
        Entry<K,V> next;

        /**
         * Creates new entry.
         */
        Entry(Object key, V value,
              ReferenceQueue<Object> queue,
              int hash, Entry<K,V> next) {
            //在这里创建了一个key的弱引用
            super(key, queue);
            this.value = value;
            this.hash  = hash;
            this.next  = next;
        }
}
```

而在getTable()获取hash表的方法中会调用expungeStaleEntries方法，该方法会获取referenceQueue中的所有元素，并对每个key将其value置为null(value是强引用置为null方便jvm回收)并从table中移出对应的entry
```java
private Entry<K,V>[] getTable() {
        expungeStaleEntries();
        return table;
    }
    
    private void expungeStaleEntries() {
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                @SuppressWarnings("unchecked")
                    Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);

                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        if (prev == e)
                            table[i] = next;
                        else
                            prev.next = next;
                        // Must not null out e.next;
                        // stale entries may be in use by a HashIterator
                        e.value = null; // Help GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
    }
```
而getTable()方法在get、put方法中都会调用，所以在每次的get/set时都会执行一次清理工作
举个例子：
```java
@Test
    public void testWeakHashMap() {
        WeakHashMap<ObjectKey, String> weakHashMap = new WeakHashMap<>();
        ObjectKey keyA = new ObjectKey("keyA");
        String valueA = new String("valueA");

        weakHashMap.put(keyA, valueA);
        keyA = null;
        System.gc();
        System.out.println(weakHashMap.get(keyA));
    }

    @Data
    class ObjectKey {
        String name;

        public ObjectKey(String name) {
            this.name = name;
        }
    }
    //input : null
```
ps: 这个例子我在跑的时候开了debug，然后在最后一句system.out.println(weakHashMap.get(keyA))前打的断点，结果发现在调用get方法前weakHashMap中的弱引用已经被清空了。。让我百思不得其解，以为weakHashMap有自动的清空机制？后来才发现调试器会调用一次map的size()方法来展示长度，而weakHashMap中的size()方法也会调用一次`expungeStaleEntries`方法:

```java
 public int size() {
        if (size == 0)
            return 0;
        expungeStaleEntries();
        return size;
    }

```

