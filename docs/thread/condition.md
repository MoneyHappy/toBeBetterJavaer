---
title: 深入理解Java并发线程协作类Condition
shortTitle: 线程协作类Condition
description: Condition 接口是 Java 并发编程中一个重要的组件，用于线程间的协调和通信。它通常与锁（特别是 ReentrantLock）一起使用，为线程提供了一种等待某个条件成真的机制，并允许其他线程在该条件变化时通知等待线程。这为线程间的协调提供了更灵活、更强大的工具。
category:
  - Java核心
tag:
  - Java并发编程
head:
  - - meta
    - name: keywords
      content: Java,并发编程,多线程,Thread,Condition
---

# 14.17 线程协作类 Condition

关于线程协作类 Condition 我们其实在前面讲 [Lock](https://javabetter.cn/thread/lock.html) 的时候提到过，不知道大家还记得不？

每个对象都可以用继承自 Object 的 wait/notify 方法来实现等待/通知机制。而 Condition 接口也提供了类似 Object 监视器的方法，通过与 Lock 配合来实现等待/通知模式。

两者除了使用方式不同外，**功能特性**也有很多不同：

1. Condition 能够支持不响应中断，而 Object 方式不支持；
2. Condition 能够支持多个等待队列（new 多个 Condition 对象），而 Object 只能支持一个；
3. Condition 能够支持超时时间的设置，而 Object 不支持

Condition 接口一共提供了以下 7 个方法：

![](https://cdn.tobebetterjavaer.com/stutymore/condition-20230812095915.png)

- `await()`：线程等待直到被通知或者中断。类似于 `Object.wait()`。
- `awaitUninterruptibly()`：线程等待直到被通知，即使在等待时被中断也不会返回。没有与之对应的 Object 方法。
- `await(long time, TimeUnit unit)`：线程等待指定的时间，或被通知，或被中断。类似于 `Object.wait(long timeout)`，但提供了更灵活的时间单位。
- `awaitNanos(long nanosTimeout)`：线程等待指定的纳秒时间，或被通知，或被中断。没有与之对应的 Object 方法。
- `awaitUntil(Date deadline)`：线程等待直到指定的截止日期，或被通知，或被中断。没有与之对应的 Object 方法。
- `signal()`：唤醒一个等待的线程。类似于 `Object.notify()`。
- `signalAll()`：唤醒所有等待的线程。类似于 `Object.notifyAll()`。

我们再来回顾一下 Object类：

![](https://cdn.tobebetterjavaer.com/stutymore/condition-20230812100450.png)

- `wait()`：线程等待直到被通知或者中断。
- `wait(long timeout)`：线程等待指定的时间，或被通知，或被中断。
- `wait(long timeout, int nanos)`：线程等待指定的时间，或被通知，或被中断。
- `notify()`：唤醒一个等待的线程。
- `notifyAll()`：唤醒所有等待的线程。


### Condition 实现原理

#### 等待队列

要想深入理解 Condition 的实现原理，就需要挖掘一下 Condiiton 的源码。

创建一个 Condition 对象可以通过`lock.newCondition()`，这个方法实际上会 new 一个**ConditionObject**的对象，ConditionObject 是 [AQS](https://javabetter.cn/thread/aqs.html) 的一个内部类，比如说 [ReentrantLock](https://javabetter.cn/thread/reentrantLock.html)。

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    abstract static class Sync extends AbstractQueuedSynchronizer {
        final ConditionObject newCondition() {
            return new ConditionObject();
        }
    }
    public Condition newCondition() {
        return sync.newCondition();
    }
}
```

AQS 内部维护了一个同步队列，如果是独占式锁的话，所有获取锁失败的线程会尾插入到**同步队列**，同样的，Condition 内部也使用了同样的方式，内部维护了一个 **等待队列**，所有调用 await 方法的线程会加入到等待队列中，并且线程状态转换为等待状态。

注意，ConditionObject 中有两个成员变量：

```java
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```

可以看的出来，ConditionObject 是通过持有等待队列的头尾指针来管理等待队列的。Node 类中有这样一个属性：

```java
//后继节点
Node nextWaiter;
```

进一步说明，**等待队列是一个单向队列**，而在之前说 AQS 时知道同步队列是一个双向队列。接下来我们用一个 demo，通过 debug 进去看是不是符合我们的猜想：

```java
public static void main(String[] args) {
    for (int i = 0; i < 10; i++) {
        Thread thread = new Thread(() -> {
            lock.lock();
            try {
                condition.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally {
                lock.unlock();
            }
        });
        thread.start();
    }
}
```

这段代码没有任何实际意义，甚至很臭，只是想证明下我们刚才所想的。新建了 10 个线程，没有线程先获取锁，然后调用 condition.await 方法释放锁将当前线程加入到等待队列中，通过 debug 走到第 10 个线程的时候，查看`firstWaiter`即等待队列中的头结点：

![debug模式下情景图](https://cdn.tobebetterjavaer.com/tobebetterjavaer/images/thread/condition-01.png)

从这个图我们可以很清楚的看到这样 2 点：

1. 调用 condition.await 方法后线程依次尾插入到等待队列中，依次为 Thread-0,Thread-1,Thread-2....Thread-8；
2. 等待队列是一个单向队列。示意图如下所示：

![等待队列的示意图](https://cdn.tobebetterjavaer.com/tobebetterjavaer/images/thread/condition-02.png)

同时还有一点需要注意：我们可以多次调用`lock.newCondition()`方法创建多个 condition 对象，也就是一个 lock 可以持有多个等待队列。

而如果是 Object 方式的话，**对象 Object 对象监视器上只能拥有一个同步队列和一个等待队列，而并发包中的 Lock 可以拥有一个同步队列和多个等待队列**。示意图如下：

![AQS持有多个Condition](https://cdn.tobebetterjavaer.com/tobebetterjavaer/images/thread/condition-03.png)

ConditionObject 是 AQS 的内部类，因此每个 ConditionObject 都能够访问到 AQS 提供的方法，相当于每个 Condition 都拥有所属同步器的引用。

#### await 实现原理

**当调用`condition.await()`方法后会使得当前获取 lock 的线程进入到等待队列，如果该线程能够从 `await()` 方法返回的话，一定是该线程获取了与 condition 相关联的 lock**。

接下来，我们还是从源码的角度来看，只有熟悉了源码的逻辑，我们的理解才是最深刻的。await 方法源码如下：

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
	// 1. 将当前线程包装成Node，尾插入到等待队列中
    Node node = addConditionWaiter();
	// 2. 释放当前线程所占用的lock，在释放的过程中会唤醒同步队列中的下一个节点
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
		// 3. 当前线程进入到等待状态
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
	// 4. 自旋等待获取到同步状态（即获取到lock）
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
	// 5. 处理被中断的情况
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

代码的主要逻辑**请看注释**，我们都知道，**当前线程调用`condition.await()`方法后，会释放 lock 然后加入到等待队列，直到被 `signal/signalAll`，`signal/signalAll` 会使当前线程从等待队列移动到同步队列中，直到获得 lock 后才会从 await 方法返回，或者在等待时被中断**。

大家可能会有这样几个问题：

1. 怎样将当前线程添加到等待队列中？
2. 释放锁的过程是？
3. 怎样才能从 await 方法中退出？

上面这段代码就是告诉了我们这三个问题的答案。

调用 addConditionWaiter 方法会将当前线程添加到等待队列中，源码如下：

```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
	//将当前线程包装成Node
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
		//尾插入
        t.nextWaiter = node;
	//更新lastWaiter
    lastWaiter = node;
    return node;
}
```

这段代码就很容易理解，将当前节点包装成 Node，如果等待队列的 firstWaiter 为 null 的话（等待队列为空队列），则将 firstWaiter 指向当前的 Node，否则，更新 lastWaiter(尾节点)即可。

就是**通过尾插入的方式将当前线程封装的 Node 插入到等待队列中**，同时可以看出，等待队列是一个**不带头结点的链式队列**，之前我们学习 AQS 时知道同步队列**是一个带头结点的链式队列**，这是两者的一个区别。

将当前节点插入到等待对列之后，会使当前线程释放 lock，由 fullyRelease 方法实现，fullyRelease 源码为：

```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        if (release(savedState)) {
			//成功释放同步状态
            failed = false;
            return savedState;
        } else {
			//不成功释放同步状态抛出异常
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

这段代码也很容易理解，**调用 AQS 的模板方法 release 释放 AQS 的同步状态并且唤醒在同步队列中头结点的后继节点引用的线程**，如果释放成功则正常返回，若失败的话就抛出异常。

以上两段代码已经回答了前面的两个问题，还剩下第三个问题，怎样从 await 方法退出呢？现在回过头再来看 await 方法，其中有这样一段逻辑：

```java
while (!isOnSyncQueue(node)) {
	// 3. 当前线程进入到等待状态
    LockSupport.park(this);
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
        break;
}
```

显然，当线程第一次调用 condition.await 方法时，会进入到这个 while 循环，然后通过 `LockSupport.park(this)` 使当前线程进入等待状态，那么要想退出 await，第一个前提条件就是要先退出这个 while 循环，出口就只两个地方：

1. 逻辑走到 break 退出 while 循环；
2. while 循环中的逻辑判断为 false。

出现第 1 种情况的条件是，当前等待的线程被中断后代码会走到 break 退出，第 2 种情况是当前节点被移动到了同步队列中（即另外一个线程调用了 condition 的 signal 或者 signalAll 方法），while 中逻辑判断为 false 后结束 while 循环。

总结一下，退出 await 方法的前提条件是**当前线程被中断或者调用 condition.signal 或者 condition.signalAll 使当前节点移动到同步队列后**。

当退出 while 循环后会调用`acquireQueued(node, savedState)`，该方法的作用是在**自旋过程中线程不断尝试获取同步状态，直到成功（线程获取到 lock）**。这样也说明了**退出 await 方法必须是已经获得了 condition 引用（关联）的 lock**。

到目前为止，上文提到的三个问题，我们都通过阅读源码的方式找到了答案，也加深了对 await 方法的理解。await 方法示意图如下：

![await方法示意图](https://cdn.tobebetterjavaer.com/tobebetterjavaer/images/thread/condition-04.png)

如图，调用 condition.await 方法的线程必须是已经获得了 lock 的线程，也就是当前线程是同步队列中的头结点。调用该方法后会使得当前线程所封装的 Node 尾插入到等待队列中。

> 超时机制的支持

condition 还额外支持超时机制，使用者可调用 awaitNanos、awaitUtil这两个方法，实现原理基本上与 AQS 中的 tryAcquire 方法如出一辙。

> 不响应中断的支持

要想不响应中断可以调用 `condition.awaitUninterruptibly()` 方法，该方法的源码如下：

```java
public final void awaitUninterruptibly() {
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    boolean interrupted = false;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if (Thread.interrupted())
            interrupted = true;
    }
    if (acquireQueued(node, savedState) || interrupted)
        selfInterrupt();
}
```

这段方法与上面的 await 方法基本一致，只不过减少了对中断的处理。

#### signal/signalAll 实现原理

**调用 condition 的 signal 或者 signalAll 方法可以将等待队列中等待时间最长的节点移动到同步队列中**，使得该节点能够有机会获得 lock。等待队列是先进先出（FIFO）的，所以等待队列的头节点必然会是等待时间最长的节点，也就是每次调用 condition 的 signal 方法都会将头节点移动到同步队列中。

我们来通过看源码的方式验证这个猜想是不是正确的，signal 方法源码如下：

```java
public final void signal() {
    //1. 先检测当前线程是否已经获取lock
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    //2. 获取等待队列中第一个节点，之后的操作都是针对这个节点
	Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

signal 方法首先会检测当前线程是否已经获取了 lock，如果没有获取 lock 会直接抛出异常，如果获取的话，再得到等待队列的头节点，之后的 doSignal 方法也是基于该节点。下面我们来看看 doSignal 方法做了些什么事情，doSignal 方法源码如下：

```java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        //1. 将头结点从等待队列中移除
        first.nextWaiter = null;
        //2. while中transferForSignal方法对头结点做真正的处理
    } while (!transferForSignal(first) &&
                (first = firstWaiter) != null);
}
```

具体逻辑请看注释，真正对头节点做处理的逻辑在**transferForSignal**方法中，该方法源码为：

```java
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
	//1. 更新状态为0
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
	//2.将该节点移入到同步队列中去
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

关键逻辑请看注释，这段代码主要做了两件事情：

- 1.将头结点的状态更改为 CONDITION；
- 2.调用 enq 方法，将该节点尾插入到同步队列中，关于 enq 方法请看 [AQS 的底层实现这篇文章](https://javabetter.cn/thread/aqs.html)。
  
现在我们可以得出如下结论：

**调用 condition.signal 方法的前提条件是当前线程已经获取了 lock，该方法会使等待队列中的头节点即等待时间最长的那个节点移入到同步队列，而移入到同步队列后才有机会被唤醒**，即从 await 方法中的 `LockSupport.park(this)` 方法中返回，才有机会让调用 await 方法的线程成功退出。

signal 执行示意图如下图：

![signal执行示意图](https://cdn.tobebetterjavaer.com/tobebetterjavaer/images/thread/condition-05.png)

> signalAll

sigllAll 与 sigal 方法的区别体现在 doSignalAll 方法上，前面我们已经知道 **doSignal 方法只会对等待队列的头节点进行操作**，doSignalAll 的源码如下：

```java
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```

该方法会将等待队列中的每一个节点都移入到同步队列中，即“通知”当前调用 `condition.await()` 方法的每一个线程。

### await 与 signal/signalAll 的结合思考

文章开篇提到的等待/通知机制，通过 condition 的 await 和 signal/signalAll 方法就可以实现，而这种机制能够解决最经典的问题就是“[生产者与消费者问题](https://javabetter.cn/thread/shengchanzhe-xiaofeizhe.html)”，“生产者消费者问题”也是面试的高频考点。

await、signal 和 signalAll 方法就像一个开关，控制着线程 A（等待方）和线程 B（通知方）。它们之间的关系可以用下面这幅图来说明，会更贴切：

![](https://cdn.tobebetterjavaer.com/stutymore/condition-20230816114036.png)

**线程 awaitThread 先通过 `lock.lock()` 方法获取锁，成功后调用 condition.await 方法进入等待队列，而另一个线程 signalThread 通过 `lock.lock()` 方法获取锁成功后调用了 condition.signal 或者 signalAll 方法，使得线程 awaitThread 能够有机会移入到同步队列中，当其他线程释放 lock 后使得线程 awaitThread 能够有机会获取 lock，从而使得线程 awaitThread 能够从 await 方法中退出并执行后续操作。如果 awaitThread 获取 lock 失败会直接进入到同步队列**。

### 一个例子

我们用一个很简单的例子说说 condition 的用法：

```java
public class AwaitSignal {
    private static ReentrantLock lock = new ReentrantLock();
    private static Condition condition = lock.newCondition();
    private static volatile boolean flag = false;

    public static void main(String[] args) {
        Thread waiter = new Thread(new waiter());
        waiter.start();
        Thread signaler = new Thread(new signaler());
        signaler.start();
    }

    static class waiter implements Runnable {

        @Override
        public void run() {
            lock.lock();
            try {
                while (!flag) {
                    System.out.println(Thread.currentThread().getName() + "当前条件不满足等待");
                    try {
                        condition.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.println(Thread.currentThread().getName() + "接收到通知条件满足");
            } finally {
                lock.unlock();
            }
        }
    }

    static class signaler implements Runnable {

        @Override
        public void run() {
            lock.lock();
            try {
                flag = true;
                condition.signalAll();
            } finally {
                lock.unlock();
            }
        }
    }
}
```

输出结果为：

```
Thread-0当前条件不满足等待
Thread-0接收到通知，条件满足
```

开启了两个线程 waiter 和 signaler，waiter 线程开始执行的时候由于条件不满足，执行 condition.await 方法使该线程进入等待状态，同时释放锁，signaler 线程获取到锁之后更改条件，并通知所有的等待线程，然后释放锁。这时，waiter 线程获取到锁，由于 signaler 线程更改了条件，此时相对于 waiter 来说，条件满足，继续执行。

### 总结

Condition 接口是 Java 并发编程中一个重要的组件，用于线程间的协调和通信。它通常与锁（特别是 ReentrantLock）一起使用，为线程提供了一种等待某个条件成真的机制，并允许其他线程在该条件变化时通知等待线程。这为线程间的协调提供了更灵活、更强大的工具。

>编辑：沉默王二，编辑前的内容主要来自于CL0610的 GitHub 仓库[https://github.com/CL0610/Java-concurrency](https://github.com/CL0610/Java-concurrency/blob/master/12.%E8%AF%A6%E8%A7%A3Condition%E7%9A%84await%E5%92%8Csignal%E7%AD%89%E5%BE%85%E9%80%9A%E7%9F%A5%E6%9C%BA%E5%88%B6/%E8%AF%A6%E8%A7%A3Condition%E7%9A%84await%E5%92%8Csignal%E7%AD%89%E5%BE%85%E9%80%9A%E7%9F%A5%E6%9C%BA%E5%88%B6.md)

---

GitHub 上标星 9000+ 的开源知识库《[二哥的 Java 进阶之路](https://github.com/itwanger/toBeBetterJavaer)》第一版 PDF 终于来了！包括 Java 基础语法、数组&字符串、OOP、集合框架、Java IO、异常处理、Java 新特性、网络编程、NIO、并发编程、JVM 等等，共计 32 万余字，可以说是通俗易懂、风趣幽默……详情戳：[太赞了，GitHub 上标星 9000+ 的 Java 教程](https://javabetter.cn/overview/)

微信搜 **沉默王二** 或扫描下方二维码关注二哥的原创公众号沉默王二，回复 **222** 即可免费领取。

![](https://cdn.tobebetterjavaer.com/tobebetterjavaer/images/gongzhonghao.png)
