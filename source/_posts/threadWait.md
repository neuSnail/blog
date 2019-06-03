---
title: 关于线程wait/notify
date: 2019-04-05 22:09:12
tags: 
- java 
- 并发
categories: java
---

### 使用场景

wait/notify的使用场景很简单，就是一个线程A需要某个特定的状态或数据(我们在这里称之为required data)，在获得这个required data之前线程A处于wait状态。线程B在提供required data之后notify线程A(或线程A们)唤醒线程A。

这个场景最常见的一个模型就是生产/消费模型。consumer等待publisher推送消息,没有消息的时候wait。publisher推送消息之后通知消费者。如果有多个consumer 会将所有consumer加入同步队列依次唤醒执行。
```java
@Test
    public void testWait() throws Exception {
        MessageChannel messageChannel = new MessageChannel();

        Thread consumer = new Thread(() -> {
            synchronized (messageChannel) {
            //想想这里为什么要用循环
                while (!messageChannel.hasMessage) {
                    try {
                        System.out.println("consumer wait for message");
                        messageChannel.wait();
                        System.out.println("continue run ");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println("get message : " + messageChannel.getMessage().remove());
            }
        });

        Thread publisher = new Thread(() -> {
            synchronized (messageChannel) {
                messageChannel.getMessage().add("new message");
                messageChannel.setHasMessage(true);
                //如果不只一个consumer，publisher会通知所有consumer
                messageChannel.notifyAll();
            }
        });

        consumer.start();
        Thread.sleep(1000);
        publisher.start();

        while (consumer.isAlive()) {
            //wait consumer done
        }
    }
```
输出结果:
```
consumer wait for message
continue run 
get message : new message
```
上面的例子中consumer是一个消费者线程，publisher是个发布者线程。required data是消息管道里的message数据。consumer在获取不到消息时会进入wait状态，等待publisher发布消息后再唤醒.
### 为什么使用synchronized?
#### reason1:
想象一下，如果这里没有加锁(假设能编译通过)，consumer在执行到`!messageChannel.hasMessage` 时判断管道内没有消息。此时在执行到wait()之前，publisher发布了一条消息，因为没有同步机制 consumer并不能感知到这次发布，仍然会执行wait()，导致错过这一次push。在一些系统模型中，这可能是致命的。
#### reason 2:
 《java并发编程的艺术》中有一句话：
 >等待/通知机制依托于同步机制，其目的就是确保等待线程从wait()方法返回时能够感知到通知线程对变量做出的修改。

wait和notify行为发生在不同的线程中，如果没有同步机制的保障可能因为指令重排或缓存可见性问题导致wait线程被唤醒时拿不到正确的required data

#### reason 3：

wait功能就是依赖监视器monitor实现的，拿不到锁无权操作monitor,只有持有锁的线程才能操作_WaiSet也就是wait()和notify()操作


### 为什么要使用循环

在该例子中,publisher发布一条消息之后使用了notifyAll()来通知所有消费者，假设消费者有多个。consumer1消费了消息之后consumer2开始消费，此时consumer2发现channel中的消息是空的，于是再次进入wait状态，循环此过程。

如果没有这个二次检查（循环）的过程，consumer2可能会被错误的唤醒。所以一般情况下被notify唤醒的线程还要对required data做一次检查

### join()的实现
线程的join也是使用wait机制实现的
```java
public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                wait(0);
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }

```

在join中required data就是被join线程的运行状态。当被join线程是alive时一直等待。直到被join线程结束时，会调用notifyAll方法通知所有等待线程。

所以join()的注释中也会提示我们用户不要在线程实例上直接使用wait/notify方法

> As a thread terminates the this.notifyAll method is invoked. It is recommended that applications not use wait, notify, or notifyAll on Thread instances.

### 在Lock中实现
基于AbstractQueuedSynchronizer 实现的ReentrantLock等锁拥有比synchronized更灵活的特性和更好的性能，那在ReentrantLock中怎么实现类似的等待/通知功能呢？
AQS提供了一个叫Condition的功能，Condition接口提供了await/signal 方法来实现wait/notify。该功能使用起来和wait/notify几乎一样。把上面的例子改一下如下：
```java
@Test
    public void testWaitInLock() throws Exception {
        MessageChannel messageChannel = new MessageChannel();
        ReentrantLock lock = new ReentrantLock();
        Condition channelEmpty = lock.newCondition();

        Thread consumer = new Thread(() -> {
        	//获取锁
            lock.lock();
            try {
                while (!messageChannel.hasMessage) {
                    System.out.println("consumer wait for message");
                    //等待
                    channelEmpty.await();
                    System.out.println("continue run ");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
            	//释放锁
                lock.unlock();
            }
            System.out.println("get message : " + messageChannel.getMessage().remove());
        });

        Thread publisher = new Thread(() -> {
        	//获取锁
            lock.lock();
            try {
                messageChannel.getMessage().add("new message");
                messageChannel.setHasMessage(true);
                //通知
                channelEmpty.signalAll();
            } finally {
            	//释放锁
                lock.unlock();
            }

        });

        consumer.start();
        Thread.sleep(1000);
        publisher.start();

        while (consumer.isAlive()) {
            //wait consumer done
        }
    }
```
可以看到就是用lock()代替synchronized,记得一定要加锁和释放锁，这里忘加锁不会在编译阶段发现，运行时会抛出异常。忘记释放锁会导致await的线程永远阻塞因为await恢复执行的前提是获取到锁。
