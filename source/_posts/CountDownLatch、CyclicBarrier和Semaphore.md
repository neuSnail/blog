---
title: CountDownLatch、CyclicBarrier和Semaphore
date: 2019-04-06 21:55:50
tags: 
- java 
- 并发
categories: java
---

CountDownLatch、CyclicBarrier和Semaphore是java.util.concurrent包下的三个并发工具类。由于三者使用上有些相似的地方，这里在一起对比说一下。

### CountDownLatch
CountDownLatch应该是三者中使用频率最高的一个，直译的话叫做“倒数闩锁“ 或者“倒数闭锁”。它允许一个或多个线程等待其他指定数量的线程执行完毕再执行
被等待线程通过调用countDown()方法减少计数,当计数减为0时，等待的线程从await处继续执行。举例如下
```java
    class LatchTask implements Runnable {
        private String name;

        private CountDownLatch latch;

        LatchTask(String name, CountDownLatch latch) {
            this.name = name;
            this.latch = latch;
        }

        @Override
        public void run() {
            System.out.println("task " + name + " start");
            try {
                Thread.sleep(2 * 1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("task " + name + " finish");
            latch.countDown();
        }
    }
    
     @Test
    public void testCountDownLatch() throws Exception {
        CountDownLatch latch = new CountDownLatch(2);
        Thread threadA = new Thread(new LatchTask("TASK_A", latch));
        Thread threadB = new Thread(new LatchTask("TASK_B", latch));

        threadA.start();
        threadB.start();
        latch.await();
        System.out.println("main thread finish");
    }
```
输出：
```
task TASK_A start
task TASK_B start
task TASK_B finish
task TASK_A finish
main thread finish
```
上例中主线程中创建了一个计数为2的latch.在taskA和taskB执行完后各会倒数一次。主线程等待所有倒数都执行完毕后从await处继续执行。
latch可以在多个线程被使用，比如我们在上面的例子中再添加一个C线程，c线程也等待该latch倒数完毕再继续执行。
```java
@Test
    public void testCountDownLatch() throws Exception {
        CountDownLatch latch = new CountDownLatch(2);
        Thread threadA = new Thread(new LatchTask("TASK_A", latch));
        Thread threadB = new Thread(new LatchTask("TASK_B", latch));

        Thread threadC = new Thread(() -> {
            System.out.println("task c start");
            try {
                latch.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("task c finish");
        });
        threadA.start();
        threadB.start();
        threadC.start();
        latch.await();
        System.out.println("main thread finish");
    }

```
输出：
```
task c start
task TASK_B start
task TASK_A start
task TASK_B finish
task TASK_A finish
task c finish
main thread finish
```
可以看到线程C是等待A、B的倒数完成后再继续执行的。

#### 原理

CountDownLatch通过AQS的共享锁实现的

```java
//CountDownLatch中的Sync
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;
		
  //初始化的时候设置count就是state的值
    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
         //countDown一次减1
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}

public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
}
//await可以指定超时时间
public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
public void countDown() {
        sync.releaseShared(1);
}
```

### CyclicBarrier

CyclicBarrier直译为“循环栅栏”或“循环屏障” ，CyclicBarrier可以使一组线程阻塞在某一处（屏障），当处于屏障状态的线程达到指定数量时同时继续执行。 CountDownLatch的计数器无法被重置；CyclicBarrier的计数器可以被重置后使用，因此它被称为是循环的barrier。



```java
 class BarrierTask implements Runnable {
        private String name;

        private CyclicBarrier barrier;

        BarrierTask(String name, CyclicBarrier barrier) {
            this.name = name;
            this.barrier = barrier;
        }

        @Override
        public void run() {
            System.out.println("task " + name + " start");
            try {
                //在此等待
                barrier.await();
            } catch (Exception e) {
                e.printStackTrace();
            }

            System.out.println("task " + name + " finish");
        }
    }

    @Test
    public void CyclicBarrierTest() {
    	//创建barrier并指定parties为3
        CyclicBarrier barrier = new CyclicBarrier(3);
        List<Thread> taskList = Arrays.asList(
                new Thread(new BarrierTask("taskA", barrier)),
                new Thread(new BarrierTask("taskB", barrier)),
                new Thread(new BarrierTask("taskC", barrier))
        );

        taskList.forEach(thread -> {
            thread.start();
            try {
                Thread.sleep(2 * 1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
    }
```
输出：
```
task taskA start
task taskB start
task taskC start
task taskB finish
task taskA finish
task taskC finish
```
上例中创建了三个线程，线程在执行时都会阻塞在 barrier.await()处。因为这里在创建barrier时指定的parties就是3，所以当三个线程都阻塞在 barrier.await()时，三个线程会继续执行。
### Semaphore
Semaphore译为“信号量”,可以用来控制同时访问特定资源的线程数量.
```java
public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }
```
在构造Semaphore对象时可以指定许可数量,可以看到Semaphore是有公平机制的，如果没指定默认使用的是非公平模式，在公平模式下等待时间久的线程会有限获得许可。

下面的例子创建了三个线程，以及一个拥有2个许可的信号量 A和B先获得许可，在执行完之后释放。线程C在尝试获取许可的时候由于许可只有两个且被AB占用只能等待A或B释放许可在恢复执行。
```java
class SemaphoreTask implements Runnable {
        private String name;
        private Semaphore semaphore;

        public SemaphoreTask(String name, Semaphore semaphore) {
            this.name = name;
            this.semaphore = semaphore;
        }

        @Override
        public void run() {
            System.out.println("task " + name + " start");
            try {
                //获取许可
                semaphore.acquire();
                System.out.println("task " + name + " get acquire");
                Thread.sleep(2 * 1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally{
                //释放许可
            	semaphore.release();
            }
            
            System.out.println("task " + name + " finish");
        }
    }

    @Test
    public void SemaphoreTest() throws Exception{
        Semaphore semaphore = new Semaphore(2);
        List<Thread> taskList = Arrays.asList(
                new Thread(new SemaphoreTask("taskA", semaphore)),
                new Thread(new SemaphoreTask("taskB", semaphore)),
                new Thread(new SemaphoreTask("taskC", semaphore))
        );

        taskList.forEach(thread -> {
            thread.start();
            try {
                Thread.sleep( 100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        taskList.get(2).join();
    }
```
semaphore的这种特性使得其适用于做流量控制或某些资源池的资源控制。