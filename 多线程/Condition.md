# Condition

## 1.概述

任意一个Java对象，都拥有一组监视器方法（定义在java.lang.Object上），主要包括wait()、 wait(long timeout)、notify()以及notifyAll()方法，这些方法与synchronized同步关键字配合，可以 实现等待/通知模式。Condition接口也提供了类似Object的监视器方法，与Lock配合可以实现等 待/通知模式，但是这两者在使用方式以及功能特性上还是有差别的。
通过对比Object的监视器方法和Condition接口，可以更详细地了解Condition的特性，对比项与结果如下表所示。

| 对比项                                       | Monitor Methods             | Condition                                                    |
| -------------------------------------------- | --------------------------- | ------------------------------------------------------------ |
| 前置条件                                     | 获取对象的锁                | 调用lock.lock()获取锁<br />调用lock.newCondition()获取Condition对象 |
| 调用方式                                     | 直接调用。如：object.wait() | 直接调用。如：condition.await()                              |
| 等待队列的个数                               | 1                           | 多个                                                         |
| 当前线程释放锁，并进入等待状态               | √                           | √                                                            |
| 当前线程释放锁，进入等待状态，不响应中断     | ×                           | √                                                            |
| 当前线程释放锁，并进入超时等待状态           | √                           | √                                                            |
| 当前线程释放锁，并进入等待状态到将来某个时间 | ×                           | √                                                            |
| 唤醒等待队列中的一个线程                     | √                           | √                                                            |
| 唤醒等待队列中的所有线程                     | √                           | √                                                            |

## 2. 框架



<img src="Condition.assets/Condition.png" style="zoom:40%;" />

每把锁可以拥有多个Condition等待队列。

## 3. 源码

因为Condition是个借口，这里我们以AQS中的内部类ConditionObject类来讲解，ConditionObject实现了Condition接口

### 3.1 await()

~~~java
public final void await() throws InterruptedException {
    // 检测一下当前线程是否被中断，是的话抛出异常。在语义上，这个线程已经进入了等待状态，只是还没有完成操作
    // 这个时候发生了中断，根据Java的逻辑需要抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 用当前线程构建一个等待结点并放到condition的队尾，返回构建的结点
    Node node = addConditionWaiter();
    
    // 释方当前线程所占用全部锁资源，返回所占用锁（state）的次数
    int savedState = fullyRelease(node);
    
    /** 
     * 中断类型
     * 		0：默认初始类型（没有中断）
     * 		1：REINTERRUPT（自己按需处理中断）
     *	   -1：THROW_IE（抛出中断）
     */
    int interruptMode = 0;
    
    // 当前结点不在AQS同步队列中，则进入循环。（因为在前面已经把当前结点放到了等待队列，所以这里会进入循环,“自旋”）
    while (!isOnSyncQueue(node)) {
        
        // park当前线程（当前线程走到这就休息了，不继续执行。等待condition的signal或者中断唤醒）
        LockSupport.park(this);
        
        // 走到这，说明当前线程被signal或者中断唤醒了。判断一下是不是被中断唤醒的
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            // 是被中断唤醒的话，那就跳出循环
            // 是被signal唤醒的话，那么就继续循环判断一下自己的node是不是已经放入同步队列了，是的话就跳出循环
            break;
    }
    /** 
     * 1.如果是被signal唤醒，已经进入了同步队列，那么会进入acquireQueued方法，自旋，尝试获取自己之前释放的锁资源
     * 2.如果是被中断唤醒的，在上面检查是否被中断的方法中，也会将其放入同步队列。都会竞争锁
     */
    // 这里的逻辑是：等待过程没有发生中断或者是不需要抛出的中断，但是在竞争锁的过程中发生了中断，则不需要抛出中断
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    // 清理一下后面的无效结点
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    // 根据中断类型不同处理中断
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

~~~

这个逻辑还是比较简单的，就是当前线程释放资源，并等待。被唤醒之后，竞争锁，并根据有无中断、中断类型进行相应的处理。我们来看一下处理这些逻辑相应的方法

#### 3.1.1 addConditionWaiter()

~~~java
private Node addConditionWaiter() {
    // 拿到队尾结点
    Node t = lastWaiter;
    // 如果队尾结点不为null，但是它的状态已经不为CONDITION了，那么就清除一遍链表里面的废弃结点
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 用当前线程构建一个node，等待状态为CONDITION
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    // 放入队列
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    // 返回当前结点
    return node;
}
~~~

##### 3.1.1.1 unlinkCancelledWaiters()

~~~java
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    // 记录前置结点
    Node trail = null;
    // 从头开始
    while (t != null) {
        // 记录next结点
        Node next = t.nextWaiter;
        // 当前结点的等待状态不等于CONDITION
        if (t.waitStatus != Node.CONDITION) {
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}
~~~

#### 3.1.2 fullyRelease(Node)

~~~java
final int fullyRelease(Node node) {
    // 释放资源失败标记
    boolean failed = true;
    try {
        // 拿到当前线程的state
        int savedState = getState();
        // 调用release全部释放掉
        if (release(savedState)) {
            // 释放成功，置失败标记为false
            failed = false;
            // 返回释放资源的数量
            return savedState;
        } else { // 释放失败抛出异常（因为release会检查当前线程是不是获取锁的线程，释放失败一般都是没有获取锁）
            // 所以抛出这个异常
            throw new IllegalMonitorStateException();
        }
    } finally {
        // 释放失败，把当前结点等待状态置为取消
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
~~~

#### 3.1.3 isOnSyncQueue(Node)

检查当前结点是否在同步队列中，看一下源码：

~~~java
final boolean isOnSyncQueue(Node node) {
    // 如果当前结点等待状态为CONDITION或者prev为null，那么肯定不在同步队列中
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    // 如果next不等于null，那么肯定在同步队列中，并且后继
    if (node.next != null) // If has successor, it must be on queue
        return true;
    // 走到这，说明结点既不在同步队列中，又不在等待队列中，那么这个结点可能在哪？在等待队列向同步队列转移过程中
    // 那么调用findNodeFromTail方法，再在同步队列中从尾到头找一次
    return findNodeFromTail(node);
}
~~~

##### 3.1.3.1 findNodeFromTail(Node)

```java
// 从头到尾查找
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```

##### 3.1.3.2  isOnSyncQueue(Node)总结

node从Condition队列转移到同步队列：

1. 第一步，就是设置waitStatus为其他值，因此是否等于Node.CONDITON可以作为判断标志，如果等于，说明还在Condition队列中，即不再Sync队列里。在node被放入Sync队列时，第一步就是设置node的prev为当前获取到的尾节点，所以如果发现node的prev为null的话，可以确定node尚未被加入Sync队列。

2. 相似的，node被放入Sync队列的最后一步是设置node的next，如果发现node的next不为null，说明已经完成了放入Sync队列的过程，因此可以返回true。

3. 当我们执行完两个if而仍未返回时，node的prev一定不为null，next一定为null，这个时候可以认为node正处于放入Sync队列的执行CAS操作执行过程中（enq方法）。而这个CAS操作有可能失败，因此我们再给node一次机会，调用findNodeFromTail来检测
4. findNodeFromTail方法从尾部遍历Sync队列，如果检查node是否在队列中，如果还不在，此时node也许在CAS自旋中，在不久的将来可能会进到Sync队列里。但我们已经等不了了，直接返回false。那在这个线程就不能跳出while循环，只能继续park，等待同步队列的前置结点唤醒它。

复杂一点的就是这个checkInterruptWhileWaiting等待过程中的中断检测逻辑，我们先来看一下。

#### 3.1.4 checkInterruptWhileWaiting(Node)

该方法判断在线程在等待过程中有无中断发生，并检测中断是在等待过程中、还是在被signal唤醒过程中发生的

~~~java
private int checkInterruptWhileWaiting(Node node) {
    // 该方法会清除中断标记位
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
    0;
}
~~~

这个方法的逻辑是这样的：

Thread.interrupted()用这个方法检测当前线程是否发生中断

- 否：返回0给上层的interruptMode

- 是：调用transferAfterCancelledWait方法判断这个中断是在等待过程中，还是被signal唤醒过程中发生的

  - 在等待过程中发生，按照Java中断的逻辑，需要抛出中断，那么返回THROW_IE

  - 在被signal唤醒过程中发生，那么需要自己之后再按需处理中断，返回REINTERRUPT

**signal并不是原子操作，在signal方法唤醒的过程中也有可能发生中断。这个中断在语义上，是在signal唤醒之后发生的，因为我已经进行了signal操作，只是还没完成而已，需要按照在signal之后的中断进行处理；而在程序看来，signal操作并没有完成，这个线程并没有被signal叫醒，而是被中断叫醒的，这里的逻辑就不符合，所需要做出判断。**

##### 3.1.4.1 transferAfterCancelledWait(Node)

直接上源码：

~~~java
final boolean transferAfterCancelledWait(Node node) {
    // 这里直接用CAS设置结点能否成功，状态来判断中断何时发生
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        // 设置结点状态成功，把当前结点放入AQS同步队列（被中断唤醒的线程也要去争用锁）
        enq(node);
        // 返回true，说明中断在被signal唤醒之前发生
        return true;
    }
	
    // 走到这说明设置结点状态失败，那么进入循环，间歇性的判断自己是否已经被加入同步队列
    while (!isOnSyncQueue(node))
        Thread.yield();
    // 最后返回false，说明中断在signal过程中发生
    return false;
}
~~~

这里的逻辑有点绕，我们来梳理一下，为什么通过设置结点的等待状态能够判断中断是在signal之前之中发生的呢？

这里就涉及到signal方法了，它的源码在后面讲，这里只需要知道signal方法在执行的第一步，便是**拿到有效（也就是waitStatus == CONDITION）的第一个等待结点，并将其waitStatus设置为SIGNAL**。所以：

- 如果中断在该线程被signal唤醒之前发生，那么该结点的waitStatus就没有改变，CAS设置结点状态就能够成功，所以返回true
- 如果中断在该线程被signal唤醒中间发生，那么该结点的waitStatus已经发生了改变，CAS设置结点状态会失败，那么当前线程只需等待signal把自己放入同步队列，再放回false就好了。

##### 3.1.4.2 checkInterruptWhileWaiting(Node)总结

这个方法检测有无中断发生：

1. 无中断发生最好，返回0
2. 有中断发生，调用transferAfterCancelledWait判断中断在什么时机发生
   - 等待过程中发生：返回抛出标记-1
   - signal过程发生：返回自行处理中断标记1

#### 3.1.5 reportInterruptAfterWait(int)

~~~java
private void reportInterruptAfterWait(int interruptMode) throws InterruptedException {
    // 如果终端类型是THROW_IE，抛出
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    // 如果是REINTERRUPT，补上一次中断
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}
~~~

#### 3.1.6 await()总结

1. 判断当前有无中断，因为已经进入await语义，有的话直接抛出
2. 通过当前线程构建一个node，放入等待队列
3. 释放自己所占用的资源
4. 进入等待状态，线程休息
5. 被唤醒时，判断唤醒自己的是signal还是中断，进入同步队列
6. 竞争资源，如果有中断，则进行相应的处理

---



### 3.2 signal()

~~~java
public final void signal() {
    // 先判断当前线程是否获取独占锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 拿到第一个等待结点
    Node first = firstWaiter;
    if (first != null)
        // 唤醒
        doSignal(first);
}
~~~

#### 3.2.1 isHeldExclusively()

~~~java
protected boolean isHeldExclusively() {
    throw new UnsupportedOperationException();
}
~~~

这个AQS中的方法抛出了一个异常，说明需要具体的子类根据不同的功能去实现。从这个方法的名字可以看出，独占锁才需要实现这个方法。ReentrantLock类中Sync中的实现如下：

~~~java
protected final boolean isHeldExclusively() {
    // 就是判断当前线程是不是独占了锁
    return getExclusiveOwnerThread() == Thread.currentThread();
}
~~~

#### 3.2.2 doSignal(Node)

~~~java
private void doSignal(Node first) {
    // 从第一个结点开始唤醒，唤醒成功跳出循环
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
~~~

##### 3.2.2.1 transferForSignal(Node)

~~~java
final boolean transferForSignal(Node node) {
    // 用CAS设置结点状态，为0，如果失败返回false
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
	// 状态改变成功，将这个结点放入同步队列中，返回该结点的前面那个结点
    Node p = enq(node);
    // 拿到前面结点的状态
    int ws = p.waitStatus;
    // 如果前面结点已经取消，或者修改前置结点状态失败，那么就唤醒当前线程。如果前面的结点有效，并且也会通知当前结点
    // 那么就不需要在这个时候唤醒，等前置结点的线程获取锁、释放锁后再唤醒。
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    // 最后返回true
    return true;
}
~~~

#### 3.2.3 signal总结

这个方法的工作流程如下：

1. 在等待队列中找到第一个有效等待的结点，并把它放入同步队列尾
2. 判断一下这个结点的前置结点是否取消，或者设置前置结点唤醒后面结点是否成功
   - 如果前置结点取消了，那么直接叫醒当前线程；如果设置前置结点唤醒服务失败，也直接唤醒当前线程
   - 如果上面两条都是正常的，那么说明前置结点代表的线程会在释放锁的时候唤醒自己，所以自己就老老实实在同步队列尾等着就好了

### 3.4 总结

Condition等待队列中的线程与AQS中同步队列中的线程在等待的时候都是被park的，本质是一样的，不同的是Condition等待队列中的线程必须要被signal，才能转移到同步队列中，以获得竞争锁的资格。