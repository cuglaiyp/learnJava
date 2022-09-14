# Future、Callable

## 1. 概述

创建线程的2种方式，一种是直接继承Thread（Thread也继承了Runnable接口），另外一种就是实现Runnable接口。这2种方式都有一个缺陷就是：在执行完任务之后无法获取执行结果。如果需要获取执行结果，就必须通过共享变量或者使用线程通信的方式来达到效果，这样使用起来就比较麻烦。而自从Java 1.5开始，就提供了Callable和Future，通过它们可以在任务执行完毕之后得到任务执行结果。

今天我们就来讨论一下Callable、Future和FutureTask三个类的使用方法。以下是本文的目录大纲：

1. 概述
2. 框架
3. 源码

## 2. 框架

<img src="Future%E3%80%81Callable.assets/FutureTask.png" style="zoom:80%;" />

### 2.1 Future接口

可以看到Future是一个顶层的规范接口，其中规定了5个方法：

- **cancel**：用来取消任务，如果取消任务成功则返回true，如果取消任务失败则返回false。参数mayInterruptIfRunning表示是否允许取消正在执行却没有执行完毕的任务，如果设置true，则表示可以取消正在执行过程中的任务。如果任务已经完成，则无论mayInterruptIfRunning为true还是false，此方法肯定返回false，即如果取消已经完成的任务会返回false；如果任务正在执行，若mayInterruptIfRunning设置为true，则返回true，若mayInterruptIfRunning设置为false，则返回false；如果任务还没有执行，则无论mayInterruptIfRunning为true还是false，肯定返回true。
- **isCancelled**：表示任务是否被取消成功，如果在任务正常完成前被取消成功，则返回 true。
- **isDone**：表示任务是否已经完成，若任务完成，则返回true；
- **get()**：用来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回；
- **get(long timeout, TimeUnit unit)**：用来获取执行结果，如果在指定时间内，还没获取到结果，就直接返回null。

### 2.2 RunnableFuture接口

RunnableFuture继承了Future和Runnable两个接口，也就是说RunnableFuture的子类既可以作为线程的执行任务，又可以拿到这个任务的返回值，从而实现得到返回值的功能。

### 2.3 FutureTask类

FutureTask是Future体系的一个具体实现类，实现了Future接口中的所有方法，也实现了Runnable中的run方法。由这个体系，我们可以知道，

1. 我们可以构造一个FutureTask对象，然后把这个task放到new Thread( task ).start()中，等到线程启动之后，就会执行这个task中的run方法。
2. 既然run方法已经被这个类重写了，那岂不是run方法已经是固定的，那怎么执行我们自己的个性化任务呢？秘密在，这个run方法中，run方法又调用了Callable接口的call方法，所以在构造task的时候，需要传一个Callable实现类对象（其实也可以传递一个Runnable，但是这个Runnable也会被封装成为Callable）
3. 我们知道Callable的中call方法是有返回值的，所以FutureTask封装了一个结果成员变量，在调用call方法的时候，把返回值给哦这个结果成员变量，我们就能用get方法拿到这个结果了。

流程图如下：

<img src="Future%E3%80%81Callable.assets/FutureTask%E6%B5%81%E7%A8%8B.png" style="zoom:40%;" />

#### 2.3.1 FutureTask的生命周期（状态state）

看一下FutureTask的生命周期有哪些：

- **NEW（0）**：新建状态。默认
- **COMPLETING（1）**：完成状态。run方法完成后，在set方法放置结果时，会先用CAS将state改变为此状态，然后再放置结果
- **NORMAL（2）**：正常状态。set放置完结果后，会用CAS将state改变为此状态
- **EXCEPTIONAL（3）**：异常状态。在run方法调用call方法时可能会抛出异常，捕获异常后，现将状态改变为完成状态，然后把异常设置为结果，再将状态改变为此状态
- **CANCELLED（4）**：取消状态。
- **INTERRUPTING（5）**：正在中断状态。
- **INTERRUPTED（6）**：中断状态。

状态的改变的几种可能：

- NEW -> COMPLETING -> NORMAL
- NEW -> COMPLETING -> EXCEPTIONAL
- NEW -> CANCELLED
- NEW -> INTERRUPTING -> INTERRUPTED

## 3. 源码

先来看一个FutureTask的demo：

~~~java
/**
Callable<Integer> callable = new Callable<Integer>() {
    @Override
    public Integer call() throws Exception {
        return 1;
    }
};
*/

FutureTask<Integer> futureTask = new FutureTask<>(() -> 1);
new Thread(futureTask).start();
System.out.println(futureTask.get());
~~~

1. 首先new一个futureTask，因为Callable是函数式接口，这里用了lambda表达式作为参数
2. 将这个task传递给一个线程，并启动
3. 拿到task中的返回值

### 3.1 FutureTask(Callable<T>)

首先来看一下FutureTask的构造方法：

~~~java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}
~~~

可以看到在构造FutureTask的时候，初始化了两个成员变量：

1. callable：这个成员变量在run方法中调用call方法用
2. state：这是个状态成员变量，也可以称之这个FutureTask的生命周期，赋值为NEW

### 3.2 run()

在线程启动后，会调用FutureTask的run方法，看一下run方法的源码：

~~~java
public void run() {
    // 首先判断一下状态，是NEW，再把当前线程设置到成员变量runner中。有一个不对，直接返回
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        // 判断callable是否为空和状态是否为NEW。二者都满足才能继续执行
        if (c != null && state == NEW) {
            V result;
            // 是否run过标记
            boolean ran;
            try {
                // 调用callable的call方法，返回值先用临时变量result接收
                result = c.call();
                // 已经run完，设为true
                ran = true;
            } catch (Throwable ex) {
                // 发生异常
                result = null;
                // run失败
                ran = false;
                // 改变state
                setException(ex);
            }
            // 没有异常，并且run完
            if (ran)
                // 设置结果
                set(result);
        }
    } finally {
        // 成员变量必须不能为null，直到run结束。因为这样可以防止多线程调用
        runner = null;
        // runner置null后，必须重新读取state，防止有其他线程将state改变成了INTERRUPTING状态（清空runner后，
        // 其他线程可以进入了）
        int s = state;
        // 如果state被其他线程改成INTERRUPTING
        if (s >= INTERRUPTING)
            // 那么必须进行处理
            handlePossibleCancellationInterrupt(s);
    }
}
~~~

#### 3.2.1 setException(Throwable)

~~~java
protected void setException(Throwable t) {
    // 切换state为COMPLETING
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        // 把异常封装给结果
        outcome = t;
        // 切换state为EXCEPTIONAL
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
        // 进行扫尾操作
        finishCompletion();
    }
}
~~~

这个方法就是把异常赋值给结果，并切换状态。

##### 3.2.1.1 finishCompletion()

~~~java
private void finishCompletion() {
    // assert state > COMPLETING;
    // 拿到单链表中第一个等待的线程结点
    for (WaitNode q; (q = waiters) != null;) {
        // 拿到结点后，将其置空，不让其他线程拿
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    // 唤醒线程，从哪睡的从哪醒
                    LockSupport.unpark(t);
                }
                // 记录拿到结点的后面结点
                WaitNode next = q.next;
                // 已经到达尾部的话，跳出循环
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                // 循环唤醒单链表中所有线程
                q = next;
            }
            break;
        }
    }

    // 空的钩子方法，留给子类实现
    done();
    callable = null;        // to reduce footprint
}
~~~

#### 3.2.2 set(V)

~~~java
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
    }
}
~~~

看了上面的setException方法，这个方法就很简单了，就是正常的放置结果，改变状态，并调用finishCompletion方法进行扫尾工作

#### 3.2.3 handlePossibleCancellationInterrupt(int)

~~~java
private void handlePossibleCancellationInterrupt(int s) {
	// 我们的run方法没有执行完时候，别的线程正在产生中断
    if (s == INTERRUPTING)
       	// 那么我们就自旋，等待中断产生完全
        while (state == INTERRUPTING)
            Thread.yield(); // wait out pending interrupt

}
~~~

#### 3.2.4 run方法总结

通过以上的分析，我们可以总结出run方法的运行流程

1. 首先判断state是不是NEW，是NEW的话，就设置好runner，就可以执行了
2. 调用Callable的call方法
3. 如果call方法出现异常，那么就把返回结果设置为异常，state设为EXCEPTIONAL；如果正常执行完，那么就把返回结果设置为正常的结果，并改变state到NORMAL
4. 调用扫尾finishCompletion方法，唤醒等待单链上的所有等待线程

**如果下次再调用这个FutureTask的run方法，由于state已经改变，所以并不会真正执行run的方法体，而是直接返回**

### 3.3 get()

这个方法就返回我们的结果

~~~java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}
~~~

可以很清晰看到这个方法的流程：

1. 先拿到state，判断是否run完毕，如果没有，说明结果还没有出来，那么调用awaitDone方法，等待run完毕
2. 如果run完毕了，那么调用report方法，返回结果

#### 3.3.1 awaitDone(boolean, long)

~~~java
/**
 * timed：是否超时等待
 * nanos：超时等待时间，单位是毫微秒
*/
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    // 如果是超时等待的话，就设置超时时间
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        if (Thread.interrupted()) {
            // 如果发生中断，移除q结点
            removeWaiter(q);
            // 抛出异常
            throw new InterruptedException();
        }
		// 拿到state
        int s = state;
        // 1. 如果已经执行结束，说明取结果的线程还没来得及休息，跑结果的线程就执行结束了
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            // 返回当前的state即可
            return s;
        }
        // 2. 如果是COMPLETING，那么就等一小会，因为跑结果线程马上就设置结果了
        else if (s == COMPLETING) 
            // 等待完自旋一下，就进入上面的if，返回啦
            Thread.yield();
        // 3. 如果上面两个if都不满足，说明要等一会了，那么判断一下q是否为空，第一次循环是为空的
        else if (q == null)
            // 就new一个等待结点，会把当前线程封装到等待结点中去，自旋一下
            q = new WaitNode();
        // 4. 第一次过后，因为q已经不为空了，所以要判断一下q是否入了等待队列
        else if (!queued)
            // 第二次自旋的时候，显然没有入队列，所以CAS帮它入队列。注意，这里是头插法。入完队列，自旋一下
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        // 5. 第三次自旋的时候，因为queued也为true了，所以会进行这个判断了，是否超时等待
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        // 6. 不是超时等待的话，就直接park当前线程
        else
            LockSupport.park(this);
    }
}

~~~

这个方法的流程也比较简单：

1. 取结果的线程发现跑结果的线程还没有跑完，那么就自旋。第一次把自己封装进等待结点中
2. 第二次自旋把等待结点用CAS的方式以头插法放进队头
3. 第三次自旋把自己park，休息，等待跑结果的线程调用扫尾的finishCompletion方法，唤醒自己以及这条单链表上的所有线程
4. 这三次自旋，每一次都首先会判断一下state的状态，看结果出来没有。第三次自旋完，就歇着了。等待被唤醒之后，再自旋一次，进入第一个if返回

#### 3.3.2 report(int)

~~~java
private V report(int s) throws ExecutionException {
    Object x = outcome;
    // 正常结束
    if (s == NORMAL)
        // 返回正常结果
        return (V)x;
    // 取消了
    if (s >= CANCELLED)
        // 抛出取消异常
        throw new CancellationException();
    // 上面都不是，说明call方法抛出了异常，在这个接着抛
    throw new ExecutionException((Throwable)x);
}
~~~

#### 3.3.3 get方法总结

这个方法也不难，比较特别的就是：当结果还没跑出来的时候，取结果的线程会阻塞，等待结果出来。而且可以有很多线程都来取结果。



## Completable

~~~java
final boolean tryPushStack(Completion c) {
    Completion h = stack; // 每一个UniAccept中都包装了源任务和当前任务
    lazySetNext(c, h); // c中的next指向h，即将h挂在c后面，c压栈
    return UNSAFE.compareAndSwapObject(this, STACK, h, c); // 设置源任务中的栈头为c
}
~~~

~~~java
static <U> CompletableFuture<U> asyncSupplyStage(Executor e,
                                                 Supplier<U> f) {
    if (f == null) throw new NullPointerException();
    CompletableFuture<U> d = new CompletableFuture<U>();
    e.execute(new AsyncSupply<U>(d, f));
    return d; // 此时，d又在线程池里面等着执行d.completeValue，又被主线程返回准备执行的d.uniAcceptStage
}

public void run() {
    CompletableFuture<T> d; Supplier<T> f;
    if ((d = dep) != null && (f = fn) != null) {
        dep = null; fn = null;
        if (d.result == null) {
            try {
                d.completeValue(f.get()); // 结果出来后,将结果放入源的结果字段中
            } catch (Throwable ex) {
                d.completeThrowable(ex);
            }
        }
        d.postComplete(); // 调用postComplete
    }
}

private CompletableFuture<Void> uniAcceptStage(Executor e,
                                               Consumer<? super T> f) {
    if (f == null) throw new NullPointerException();
    CompletableFuture<Void> d = new CompletableFuture<Void>(); // 又new了一个新CompletableFuture d。执行这个方法的d（this）称为源，新的成为依赖
    if (e != null || !d.uniAccept(this, f, null)) { // 依赖d执行uniAccept。
        UniAccept<T> c = new UniAccept<T>(e, d, this, f); // 下面返回了false.就进来了. 然后将源和依赖(只关注源和依赖)封装成一个Completion
        push(c); // 将这个Completion压入源的栈顶
        c.tryFire(SYNC); // 用栈顶的Completion以同步的方式执行tryFire
    }
    return d; //返回依赖
}


final <S> boolean uniAccept(CompletableFuture<S> a,
                            Consumer<? super S> f, UniAccept<S> c) {
    Object r; Throwable x;
    // 判断源、源的结果、依赖的步骤是不是为null
    if (a == null || (r = a.result) == null || f == null) // 有一个为null 返回false。当d.completeValue任务耗时很长时，源结果为null，在设计好的场景里，这里会直接返回false
        return false;
    tryComplete: if (result == null) { // 判断结果是否为null,这个结果表示的是谁执行这个方法就是谁的.当源结果已经出来了,那么就可以进到这里来.判断一下依赖结果出来没有
        // 依赖结果没有出来,那么就根据源结果的不同进行处理
        if (r instanceof AltResult) { // 先不管异常结果
            if ((x = ((AltResult)r).ex) != null) {
                completeThrowable(x, r);
                break tryComplete;
            }
            r = null;
        }
        try {
            // 源结果出来,调用栈顶的claim方法,异步唤醒
            if (c != null && !c.claim())
                return false;
            @SuppressWarnings("unchecked") S s = (S) r;
            f.accept(s);
            completeNull();
        } catch (Throwable ex) {
            completeThrowable(ex);
        }
    }
    return true;
}

// 这个方法的意思是尝试去释放,因为可能源的结果已经出来了,所以栈顶的Completion尝试能不能释放一下,执行后续操作
final CompletableFuture<Void> tryFire(int mode) {
    CompletableFuture<Void> d; CompletableFuture<T> a;
    if ((d = dep) == null ||
        !d.uniAccept(a = src, fn, mode > 0 ? null : this)) // 拿到依赖执行的它的uniAccept,因为时SYNC,所以mode=0,如果源的结果为null,还是回返回false
        return null;
    dep = null; src = null; fn = null;
    return d.postFire(a, mode);
}

final void postComplete() {
    /*
         * On each step, variable f holds current dependents to pop
         * and run.  It is extended along only one path at a time,
         * pushing others to avoid unbounded recursion.
         */
    CompletableFuture<?> f = this; Completion h;
    while ((h = f.stack) != null ||
           (f != this && (h = (f = this).stack) != null)) { // 拿到栈顶的Completion
        CompletableFuture<?> d; Completion t;
        if (f.casStack(h, t = h.next)) { // 出栈
            if (t != null) { 
                if (f != this) { // 先不考虑这个if
                    pushStack(h);
                    continue;
                }
                h.next = null;    // detach
            }
            // 调用栈顶Completion tryFire
            f = (d = h.tryFire(NESTED)) == null ? this : d;
        }
    }
}
~~~

![Completable](Future%E3%80%81Callable.assets/Completable.png)