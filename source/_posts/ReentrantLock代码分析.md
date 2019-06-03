---
title: ReentrantLock代码分析
date: 2019-04-21 14:17:33
tags: 
- java
- 并发
categories: java
---

### Structure

ReentrantLock的结构如下图所示，其中Sync是ReentrantLock的抽象内部静态类,继承自AQS。FairSync和NonfairSync继承Sync.

![image-20190421141918076](https://gitee.com/neusnail/img/raw/master/blogimg/struct.png)
AQS使用了模板方法的模式，一个模板方法是定义在抽象类中的，把基本操作方法组合在一起形成一个总算法或一个总行为的方法。一个抽象类可以有任意多个模板方法，而不限于一个。每一个模板方法都可以调用任意多个具体方法。
AQS中的模板方法有 acquire,acquireShared,release,releaseShared等,而模板方法通过调用子类实现的具体方法(钩子)来实现。
AQS中定义的钩子方法：

* tryAcquire
* tryRelease
* tryAcquireShared
* tryReleaseShared
* isHeldExclusively

钩子方法不需要全部重写，没重写的钩子方法默认会抛出UnsupportedOperationException。

![image-20190421145309101](https://gitee.com/neusnail/img/raw/master/blogimg/structure2.png)

### Lock

ReentrantLok构造函数时候可以指定使用公平性，不指定的话默认使用非公平锁。所谓公平就是按锁等待时间排序，非公平就是lock时候立即尝试获取一次能抢到就用,抢不到再去acquire排队。非公平效率更高一些所以默认使用非公平。但是非公平可能导致线程饥饿，即一个线程一直被其他线程抢占锁导致长时间获取不到锁。

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
}
```
```java
//ReentrantLock的lock方法
public void lock() {
        sync.lock();
		}
//NoneFairSycn的lock实现
 final void lock() {
   			// 先立即尝试一下加锁(非公平)
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
          //获取不到再去acquire
            acquire(1)
    }
//compareAndSetState通过unsafe的CAS实现(jdk9之后用的是VarHandler)
protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
    
```

ReentrantLock中的lock方法最后会调用AQS中的模板方法acquire，acquire方法会调用tryAcquire尝试获取锁，获取不到（返回false）的话再使用acquireQueue方法排队等待。注意这里selfInterrupt()不会中断,只会设置下中断标记,只有使用acquireInterruptibly时才会响应中断,不过那个方法也不会调到这里来而是调用doAcquireInterruptibly()

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
      //acquireQueued这个方法会block住，如果block的过程中被中断了会返回true，此时if成立 执行selfInterrupt(Thread.interrupt,不会抛异常)
        selfInterrupt();
}
```
非公平锁的tryAcquire实现:

```java
//NonfairSync 中的tryAcquire实现
protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
  //state为0说明没有线程占用锁
    if (c == 0) {
      //直接cas尝试占用
        if (compareAndSetState(0, acquires)) {
          //exclusiveOwnerThread是非volatile的,其getter和setter也是未加锁的
          //所以使用时要确认不会有其他线程尝试set
          //这里(compareAndSetState(0, acquires)保证只有当前线程操作
            setExclusiveOwnerThread(current);
            return true;
        }
    }
  //如果锁的当前持有者就是当前线程,那么就直接获取(state+1),实现了ReentrantLock的可重入
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
  //都不满足抢占锁失败,返回false
    return false;
}
```

addWaiter将当前线程是封装为Node对象,加入同步队列(实际上就是个Node的链表)

```java
private Node addWaiter(Node mode) {
				//mode用来标识是独占还是共享 
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
           	//先快速尝试一遍把节点插到尾部,cas失败的话再使用enq方法插入
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
  			//enq就是在自旋锁里使用cas插到队列尾部
        enq(node);
        return node;
    }
```
Node的结构以及waitstatus:
```java
 static final class Node {
        /** Marker to indicate a node is waiting in shared mode */
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        static final Node EXCLUSIVE = null;
				//在同步队列中等待的线程等待超时或者被中断，需要从同步队列中取消等待。
        static final int CANCELLED =  1;
				//后继节点处于等待状态，当前节点如果释放了同步状态或者被取消，
				//将会通知后继节点，使后继节点得以运行.
        static final int SIGNAL    = -1;
        //节点在等待队列中,等其他线程调用了condition的signal之后
        //该节点将会从等待队列中转移到同步队列中
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        static final int PROPAGATE = -3;
				//初始状态0
        volatile int waitStatus;
				//前节点
        volatile Node prev;
				//后节点
        volatile Node next;
      	//当前节点装载的线程
        volatile Thread thread;
				//等待队列中的后继节点,因为只有独占锁才有等待队列所以如果当前节点是共享的
				//复用该字段作为节点类型标记(常量SHARED). (我个人很讨厌这种骚操作)
        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * Returns previous node, or throws NullPointerException if null.
         * Use when predecessor cannot be null.  The null check could
         * be elided, but is present to help the VM.
         *
         * @return the predecessor of this node
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

acquireQueued是AQS中的方法,通过自旋锁+park(停滞)的方式将一个node插入同步队列中

这里在tryAcquire之前会判断前驱节点是否是头结点,这保证了同步队列的FIFO,如果当前节点被提前唤醒(中断或者前驱节点cancelAcquire导致parkSuccessor),只有前驱节点是head的时候才会尝试去获取锁

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        //这里用一个死循环自旋
        for (;;) {
        		//p是当前节点的前驱节点
            final Node p = node.predecessor();
            //如果前置节点是头结点(即当前节点是下一个要获取锁的节点),再去尝试获取锁
            if (p == head && tryAcquire(arg)) {
            //将当前节点置为头结点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
          //短路操作,尝试获取锁失败之后判断是否可以park
          //可以park,调用parkAndCheckInterrupt方法park当前线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
              //骚写法,parkAndCheckInterrupt()返回true时才能执行到这里,说明有中断
                interrupted = true;
        }
    } finally {
      //如果在tryAcquire成功之前异常了,取消acquire,删除当前节点
        if (failed)
            cancelAcquire(node);
    }
}
```
shouldParkAfterFailedAcquire判断是否可以park当前进程,代码如下:

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
  	//如果前驱节点waitStatus是signal,前驱节点释放了后会通知当前node,可以直接park
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
  //唯一大于0的状态就是cancel,前驱节点状态为cancel说明前驱节点被中断或超时了,那么向前继续寻找
  //知道找到状态>=0的节点,插在该节点后面
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
      //第一次调用这个方法时都会进到这个分支,因为前驱节点的状态为0.然后回到自旋流程
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
      //cas将前驱接节点状态设置为signal
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
  //前驱节点不为signal,继续自旋
    return false;
}
```

park阻塞当前线程,调用unpark或者中断时候才会从park方法返回,返回之后通过Thread.interrupted()确认线程是否是被中断.

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

### Unlock(release)

通过调用ReentrantLock的unlock方法来释放一个锁

```java
public void unlock() {
    sync.release(1);
}
```

release是AQS里的模板方法 先调用tryRelease把state减到0 再调用unparkSuccessor unpark后继节点.

这里没有对同步队列的链表做操作,链表的头结点替换是acquireQueue里做的

```java
public final boolean release(int arg) {
  //调用reentrantLock里的tryRelease具体实现
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
          //unpark后继节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

java.util.concurrent.locks.ReentrantLock.Sync#tryRelease:

```java
protected final boolean tryRelease(int releases) {
		//state release一次减1(可能重入过多次)
    int c = getState() - releases;
  	//未拿到锁的线程不能释放其他线程的锁
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
  	//state减到0释放锁
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

释放锁之后unpark后继节点java.util.concurrent.locks.AbstractQueuedSynchronizer#unparkSuccessor:

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
  //如果后继节点为null或者被cancel了 从尾部往前找,node后面的第一个ws小于0的节点
  //然后unpark这个节点
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

### Condition

condition是为了提供类似Object监视器中的wait/notify方法

ConditionObject是AQS的内部类,实现Conditon接口. 每个Condition对象都包含一个等待队列,和同步队列复用了Node对象.当前线程调用Conditon#awai方法会以当前线程构造节点,并将节点插入等待队列尾部.

一个同步器拥有一个同步队列和多个等待队列,等待队列持有同步队列的引用.

![image-20190423155406586](https://gitee.com/neusnail/img/raw/master/blogimg/condition.png)

通过ReentrantLock的newCondition方法创建一个condition
```java
//java.util.concurrent.locks.ReentrantLock#newCondition
public Condition newCondition() {
    return sync.newCondition();
}
//java.util.concurrent.locks.ReentrantLock.Sync#newCondition
final ConditionObject newCondition() {
            return new ConditionObject();
        }
```

看一个使用示例:

```java
@Test
public void testAQSCondition() {
    ReentrantLock lock = new ReentrantLock();
    Condition condition = lock.newCondition();
    Thread threadA = new Thread(() -> {
      //先获取锁
        lock.lock();
        condition.signalAll();
        System.out.println("threadA signal all");
        try {
            Thread.sleep(5 * 1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            System.out.println("threadA unlock");
          //释放锁
            lock.unlock();
        }
    });

    lock.lock();
    threadA.start();
    try {
      //等待并释放锁
        condition.await();
        System.out.println("wake from await");
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        lock.unlock();
    }
}

input:
threadA signal all
threadA unlock
wake from await
```

在上面的例子中主线程先获取锁,然后再调用await方法阻塞当前线程,该方法会释放锁.之后线程A才会获得锁.threadA调用signalAll之后还要释放锁主线程才会从await处返回,可以看到和监视器锁的wait/notify十分类似.

```java
public final void await() throws InterruptedException {
  //如果当前线程被中断了抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
  //新建一个Node,添加到等待队列
    Node node = addConditionWaiter();
  //释放当前锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
  //自旋,确定node转移到同步队列退出
    while (!isOnSyncQueue(node)) {
      //park阻塞
        LockSupport.park(this);
      //唤醒之后如果有中断退出循环
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
  //先插入同步队列 获得锁之后再判断中断类型
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
  //根据中断模式处理中断(抛异常或者Thread.interrupt)
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

### SharedLock
ReentrantReadWriteLock 通过writeLock和readLock分别获取读写锁

```java
public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }
```

这玩意就是内部包了两个静态内部类,一个WriteLock一个ReadLock,两个都实现了Lock接口,拥有一个实现了AQS的sync field. witeLock是独占锁,readlock是共享锁.

读写锁和reentrantLock一样,提供公平性选择,也是可重入的.

读写锁也是使用一个volatile的state来表示锁状态. 做了按位分割.高16位用来表示读状态,低16位用来表示写状态.

**如果存在读锁 不能获取写锁** 因为写操作应该对读操作可见,只有所有读锁都是放了才能获取写锁

**如果其他线程获取了写锁 当前线程不能获得读锁** 只有当前线程获得了写锁或者写锁没有被任何线程获取 才能获得读锁

不想写了 烦 代码先不贴了 有时间再说吧

