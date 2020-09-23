在了解Fork-Join之前，我们得先了解什么是**并行计算**。

## 并行计算

相对于串行计算，`并行计算`可以划分成`时间并行`和`空间并行`。**时间并行**即`指令流水化`，也就是流水线技术。比如说生产一辆小汽车，有特定的轮子车间/发动机车间，同时进行各自的生产。**空间并行**是指使用`多个处理器执行并发计算`。

以程序和算法设计人员的角度看，`并行计算`又可分为`数据并行`和`任务并行`。**数据并行**把`大的任务化解成若干个相同的子任务`，**任务并行**是指`每一个线程执行一个分配到的任务`，而这些线程则被分配（通常是操作系统内核）到该并行计算体系的各个计算节点中去。

简单来说，并行计算是通过把**大问题划分为小问题，运用计算机资源并行的处理子问题，当需要得到大问题的结果时，将小问题的结果按顺序合并起来得到最终结果**。这种思想就是`分治思想`，小到归并排序，大到大数据计算...

## Fork-Join

Fork-Join框架是Doug Lea 大神在JDK7引入的。`Fork`就是把大问题拆分成小问题，也就是大任务拆成多个子任务，并行执行子任务。`Join`就是把任务的结果按顺序合并起来。

<img src="%E8%B0%88%E8%B0%88ForkJoin%E6%A1%86%E6%9E%B6%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.assets/forkjoin%E6%9E%B6%E6%9E%84.png" style="zoom:50%;" />

假设我们需要求**从 1-1亿之间的数字和**，`按照Fork-Join的思想`，可分为以下三步：

1. 定义拆分子任务和合并子任务的规则

   - 划分子任务的规则

     首先将任务拆为 1-5千万 和 5千万01 - 1亿两个子任务，直到每个子任务计算的数字范围在1万以内的时候，我们才计算这个任务的和。

   - 合并子任务的规则

     同一父任务的所有子任务的结果再相加，就是这一父任务的结果。

2. 充分利用计算机资源，最大并行的执行子任务

3. 充分利用计算机资源，执行合并所有子任务，获得最终的结果

显然一般人做不了后两步，我们只需要把 **怎么拆，怎么和** 告诉Fork-Join框架，**Fork-Join框架**就帮我们做好 `如何最大并行执行子任务` 和 `如何最有效合并子任务`。

### 设计原理

如何充分利用计算机资源,最大并行执行子任务?一般小伙伴应该可以想到**使用多线程**，让线程数等于CPU核数。此时可以充分利用CPU的计算资源。

我们来看一下**JDK普通线程池**是咋玩的。（不要说你不懂为啥池化 :）

<img src="%E8%B0%88%E8%B0%88ForkJoin%E6%A1%86%E6%9E%B6%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.assets/%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%BB%93%E6%9E%84%E5%9B%BE.png" style="zoom:50%;" />

任务都是丢到一个同步队列`BlockingQueue`中的。如果你了解JDK BlockingQueue的实现，就知道有界的同步队列都是用锁阻塞的，有些push/poll操作还共用一把锁。

`问题1:并行的任务有必要共用一个阻塞队列吗？`

`问题2: 如果任务队列中的任务存在依赖，worker线程只能被阻塞着`。啥意思呢？

假设任务队列中存在两个任务task1和task2，**task1的执行结果依赖于task2的结果**。如果worker1先拉取到task1,结果发现此时task2还没有被执行。则worker1只能阻塞等待别的worker拉取到task2,task2执行完了worker1才能继续执行task1。

如果worker1当发现task1无法继续执行下去时，能够先把它放一边，继续拉取任务执行。这样效率是比较高的。

### Work−Stealing

**Fork-Join框架**通过`Work−Stealing`算法解决上面两个问题。

<img src="%E8%B0%88%E8%B0%88ForkJoin%E6%A1%86%E6%9E%B6%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.assets/Work-Stealing.png" style="zoom:50%;" />

- 每个`线程`拥有`自己的任务队列`，并且是`双端队列`。
- 线程操作`自己的任务队列`是`LIFO（Last in First out）`模式。
- 线程还可以偷取`别的线程任务队列`中的任务，模式为`FIFO（First in First out）`。

显然 **每个线程拥有自己的任务队列**可以提高获取队列的并行度，**双端任务队列**将所属的自己线程的push/pop操作 和 其他线程的steal操作通过不同的模式区分开。这样`只有当Base==Top-1时，pop操作和steal操作才会有冲突`。

如何才能**准确及时知道Base==Top-1**呢，Fork-Join框架的牛逼之处也在于对**任务的调度是轻量级**的。

> steal操作

考虑steal操作，是**多个其他线程的同步操作**。需要保证：`偷到Base处的任务`和`Base++的原子性`，同时`Base的值`一旦改变，其他线程应该能够`马上可见`。聪明的小伙伴是不是想到 **锁和volatile** 了：）

~~~java
//steal操作 就是 poll()方法
final ForkJoinTask<?> poll() {
    ForkJoinTask<?>[] a; int b; ForkJoinTask<?> t;
    //array就是双端队列，实际用数组实现。
    //base是将要偷的任务下标，base是用volatile修饰的，保证可见性
    //top是将要push进去的任务下标，可参考上面示意图
    while ((b = base) - top < 0 && (a = array) != null) {
        //说明经过while条件初步判断任务队列不为空
        //获取base处的任务在任务队列中的偏移量
        int j = (((a.length - 1) & b) << ASHIFT) + ABASE;
        //用volatile load 语义取出base处的任务t,可以简单理解为一定是最新修改版本的任务
        t = (ForkJoinTask<?>)U.getObjectVolatile(a, j);
        //再次读取base,判断此时t是否被别的线程偷走
        if (base == b) {

            if (t != null) {
                //如果多次读判断都没啥问题，CAS修改base处的任务t为null
                if (U.compareAndSwapObject(a, j, t, null)) {
                    //如果上面修改成功，表示这个任务被该线程偷到了
                    //此时就将base指针向前移一位，注意这一步是原子操作，base++就不是了
                    base = b + 1;
                    return t;
                }
            }
            else if (b + 1 == top) 
                // 如果t==null && b + 1 == top，此时任务队列为空
                break;
        }
    }
    return null;
}
~~~

简单来说，**有任务可偷时**，通过`CAS偷任务`保证`只有一个线程能偷成功`，偷成功的这个线程接着`修改volatile base指针`，使得`马上对其他线程可见`。同时通过前面的`多次读判断减少`后期CAS并发的`冲突`概率。 **没任务可偷时**，通过`CAS偷任务失败`可以判断出来。

请小伙伴一句句看上面的代码，阿姨都注释出来了。虽然上面并没有锁，，但是小伙伴想想`锁`其实是`悲观控制并发`的思想，是不是可以拆成`多次读判断 + CAS原子修改`的`乐观思想`来控制并发。只要`最终保证只有一个能修改成功`就可以了。

> push操作

考虑push操作，是`任务队列所属的线程`才能`操作`，**天生线程安全**： `不需要`通过CAS或锁来保证`同步`，只需要`原子`的`修改top处任务` 和 `top向前移一位` 就可以了。 同理，`top`也`不需要用volatile修饰`。

~~~java
final void push(ForkJoinTask<?> task) {
    ForkJoinTask<?>[] a; ForkJoinPool p;
    int b = base, s = top, n;
    if ((a = array) != null) {    // ignore if queue removed
        int m = a.length - 1;     // fenced write for task visibility
        //更新双端队列array的top处任务为task，直接原子更新，非CAS操作
        //因为这个方法只会被array所属的线程调用，所以这里是线程安全的
        U.putOrderedObject(a, ((m & s) << ASHIFT) + ABASE, task);
        //top指针向前移一位
        U.putOrderedInt(this, QTOP, s + 1);
        if ((n = s - b) <= 1) {
            //说明未push前队列中最多有一个任务
            if ((p = pool) != null)
                //此时唤醒其他等待的线程，表示整体pool中有事情可以做了。。
                p.signalWork(p.workQueues, this);
        }
        else if (n >= m)
            //队列扩容
            growArray();
    }
}
~~~

小伙伴思考下，这里`Base和Top指针会存在任务冲突`吗？其实不会哦，因为`两个指针都在往前冲，Base永远追赶不上Top`。这个方法**额外需要做的事情** 是 **唤醒空闲线程** 表示有任务进来了， 判断**队列是否需要扩容**就好。

> pop操作

考虑pop操作，虽然`任务队列所属的线程`才能`操作`，但是`当任务队列只有一个任务`时，存在`steal操作`和`pop操作`的**任务竞争**。原理就和`steal操作`一致了，当`CAS修改top-1处任务为空 成功`时，`再更新top值为top-1`。

~~~java
final ForkJoinTask<?> pop() {
    ForkJoinTask<?>[] a; ForkJoinTask<?> t; int m;
    if ((a = array) != null && (m = a.length - 1) >= 0) {
        for (int s; (s = top - 1) - base >= 0;) {
            long j = ((m & s) << ASHIFT) + ABASE;
            if ((t = (ForkJoinTask<?>)U.getObject(a, j)) == null)
                break;
            if (U.compareAndSwapObject(a, j, t, null)) {
                U.putOrderedInt(this, QTOP, s);
                return t;
            }
        }
    }
    return null;
}
~~~

注意这个`pop操作`并没有`steal操作`那么多次预读避免并发竞争，小姐姐yy是因为`pop操作`只有**在任务队列中只有一个任务**时，才会存在和`Steal`操作的竞争问题。而`Steal`操作也时时可能**存在多个其他线程的竞争问题**的。

通过上面三个任务调度方法的分析，你有没有感受到一丝丝FJ的**调度轻量级**呢？

**总结一下**：Fork-Join框架通过将`共享的任务队列`拆分成`线程独有的双端任务队列`，`多线程steal操作`通过`多次读`和`CAS`保证`同步`，`steal操作和pop操作` 通过`CAS` 保证`同步`，push操作线程安全，不需要同步。

### Fork-Join框架使用

要能回答上面的问题，我们先看一下如何使用Fork-Join框架。上面这三个方法并不是我们能直接调用的，这三个方法是Fork-Join自己**在合适的时机**自己调用的。像最开始所说，使用者只需要： `定义好拆分子任务和合并子任务的规则的大任务，并且把任务丢给ForkJoinPool就好`

> 求 1-1亿之间的数字和

`Step1`.**定义一个求和的任务类**

- 继承`RecursiveTask`类,重写其`compute()`方法：

  RecursiveTask如其名，是一个**归并任务**。`compute()`方法是具体如何拆分，如何归并的实现。`fork()`方法就是在确定拆分子任务规则时调用的，该方法会`把子任务push到当前线程自己的任务队列`中；`join()`方法就是在确定合并子任务的规则时调用的，该方法会`等待直到返回子任务的结果`。

  ~~~java
  public class SumTask extends RecursiveTask<Long> {
      private long[] numbers;
      private int from;
      private int to;
  
      public SumTask(long[] numbers, int from, int to) {
          this.numbers = numbers;
          this.from = from;
          this.to = to;
      }
  
      @Override
      protected Long compute() {
          //拆分子任务的规则：
          // 1.当需要计算的数字小于6时，直接计算结果
          if (to - from < 6) {
              long total = 0;
              for (int i = from; i <= to; i++) {
                  total += numbers[i];
              }
              return total;
              // 2.否则，把任务一分为二，递归计算
          } else {
              int middle = (from + to) / 2;
              //构造子任务
              SumTask taskLeft = new SumTask(numbers, from, middle);
              SumTask taskRight = new SumTask(numbers, middle+1, to);
              //将子任务添加到任务队列，这一步我们还是要做了
              taskLeft.fork();
              taskRight.fork();
  
              //合并所有子任务的规则：所有子任务的结果相加
              return taskLeft.join() + taskRight.join();
          }
      }
  }
  ~~~

  **在等待子任务结果的时候，线程被阻塞了吗**？

  （当然没有，这段时间其实就会**偷任务**来做。后面我们再分析：）

`Step2`.**构造一个Fork-Join线程池，把上面的求和大任务SumTask丢进去**

~~~java
public static void main(String[] args) {
    ForkJoinPool pool = new ForkJoinPool();
    SumTask sumTask =  new SumTask(numbers, 0, numbers.length-1)
        long result = pool.invoke(sumTask);
    System.out.println(result);
}
~~~

从这里，我们可以看到任务丢进线程池是调用的`pool.invoke(sumTask)`

（ 熟悉JDK线程池实现的小伙伴可以结合上面ForkJoin框架的原理想想任务该如何流转。小姐姐开始了：）

### 一个归并任务的流转

`Step1`.**任务提交到任务队列**

包括invoke等所有任务提交方法最终都会调用`ForkJoinPool.externalPush`方法。

**这里面需要考虑将任务提交到哪个队列？**

如果提交到ForkJoinWorkerThread自己的双端任务队列中： 不管提交到头还是尾，都会和我们上面分析的三个操作发生任务冲突。 而且如何选择负载最小的线程来提交也会增加问题复杂性。

ForkJoinPool中双端任务队列是用数组（`volatile WorkQueue[] workQueues`）实现的，其中`奇数下标`存放的是`可激活`的任务队列，`偶数下标`存放的是`不可激活的任务队列`。`激活`指的是`这个队列是否是某个ForkJoin线程的任务队列`。

<img src="%E8%B0%88%E8%B0%88ForkJoin%E6%A1%86%E6%9E%B6%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.assets/%E4%BB%BB%E5%8A%A1%E6%8F%90%E4%BA%A4.png" style="zoom:50%;" />

`ForkJoinPool.externalPush`只能将任务提交到`不可激活任务队列`，该方法的主要逻辑为：

当提交的任务是pool的第一个任务时，会初始化`workQueues`，`ForkJoinWorkerThread`等资源，通过hash算法选择一个偶数下标的`workQueue`，在TOP处放入任务。同时唤醒`ForkJoinWorkerThread`开始拉取任务工作。

当提交的任务不是第一个任务，此时`workQueues`等资源已初始化好。同样需要选择一个偶数下标的`workQueue`存放任务，如果选中的`workQueue`只有这一个任务，说明之前线程资源大概率是闲置的状态，会尝试 **唤醒（`signalWork`方法）** 一个空闲的`ForkJoinWorkerThread`开始拉取任务工作。

`Step2`.**ForkJoinWorkerThread的运行**

我们先看一下**任务的生产和消费**模式：

<img src="%E8%B0%88%E8%B0%88ForkJoin%E6%A1%86%E6%9E%B6%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.assets/%E4%BB%BB%E5%8A%A1%E7%9A%84%E6%89%A7%E8%A1%8C-1.png" style="zoom:50%;" />

- 可激活的workQueue`自己所属`ForkJoinWorkerThread`的任务模式是`LIFO（Last In First Out）
- 不可激活的workQueue`的任务模式是`FIFO（First In First Out）

`orkJoinWorkerThread`刚开始运行时会调用`ForkJoinWorkerThread.scan`方法随机选取一个队列`从Base处`捞取任务。捞取到任务会调用`WorkQueue.runTask`方法执行任务，最终对于`RecursiveTask`任务执行的是`RecursiveTask.exec`方法。

~~~java
protected final boolean exec() {
    //我们一开始定义SumTask的实现方法：compute
    result = compute();
    return true;
}
~~~

里面调用的就是我们一开始定义SumTask的实现方法：`compute`方法。

`fork`所做的事情就是将我们切分的子任务添加到当前ForkJoinWorkerThread自己的workQueue中

~~~java
if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
    ((ForkJoinWorkerThread)t).workQueue.push(this);
~~~

`join`所做的事情就是等待子任务的返回结果

~~~java
public final V join() {
    int s;
    //doJoin会返回执行结果

    if ((s = doJoin() & DONE_MASK) != NORMAL)
        //结果有异常，抛出异常信息
        reportException(s);
    //结果无异常，返回正常结果
    return getRawResult();
}
~~~

讲原理的时候我们提到了当调用`join`获取任务结果时，`ForkJoinWorkerThread`会根据当前任务的情况，做出最正确的执行判断，而不是单纯的阻塞等待结果。

`Step3`.**`join`时执行任务的判断**

结合上面求和的例子，我们来看一下**求1-10之间的数字和**的求和任务的**可能join过程**：

<img src="%E8%B0%88%E8%B0%88ForkJoin%E6%A1%86%E6%9E%B6%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.assets/%E4%BB%BB%E5%8A%A1%E7%9A%84%E6%89%A7%E8%A1%8C-2.png" style="zoom:50%;" />

**`case1:任务未被偷`** 

假设求和 `1-10任务`被Thread1执行，fork出两个子任务：`1-5` 和 `6-10`，只要Thread1能`判断`出来`要join的任务在自己的任务队列`中，那当前join哪个子任务`就把它取出来执行`就可以。

**`case2:任务被偷，此时自己的任务队列为空，可以帮助小偷执行它未完成的任务`**

<img src="%E8%B0%88%E8%B0%88ForkJoin%E6%A1%86%E6%9E%B6%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.assets/%E4%BB%BB%E5%8A%A1%E7%9A%84%E6%89%A7%E8%A1%8C-3.png" style="zoom:50%;" />

假设求和 `1-10`任务被`Thread1`执行，fork出两个子任务：`1-5` 和 `6-10`。`6-10`已成功执行完成，join返回了结果。但此时发现`1-5`被`Thread2`偷走了，自己的任务队列中已经没有任务可以执行了。此时`Thread1`可以找到小偷`Thread2`，并偷取`Thread2`的`10-20`任务来帮助它执行。

**`case3:任务被偷，此时自己的任务队列不为空`**

<img src="%E8%B0%88%E8%B0%88ForkJoin%E6%A1%86%E6%9E%B6%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.assets/%E4%BB%BB%E5%8A%A1%E7%9A%84%E6%89%A7%E8%A1%8C-4.png" style="zoom:50%;" />

假设求和 `1-10`任务被`Thread1`执行，fork出两个子任务：`1-5` 和 `6-10`，要join `1-5`时发现已经被`Thread2`偷走了，而自己队列中还有`6-10`等待join执行。不好意思帮不了小偷了。

只好尝试挂起自己等待`1-5`的执行结果通知，并尝试`唤醒空闲线程`或者`创建新的线程`替代自己执行任务队列中的`6-10`任务。

上述三种情况代码均在`ForkJoinPool.awaitJoin`方法中。整体思路是：

**当任务还在自己的队列：**

- 自己执行，获取结果。

**当被别人偷走阻塞了：**

- 自己又没任务执行，就帮助小偷执行任务。
- 自己有任务要执行，就尝试挂起自己等待小偷的反馈结果，同时找队友帮助自己执行。

这里任务模式有意思的是：

`scan/steal`操作都是从**Base处**获取任务，那么**更容易获取到大的任务**执行，从而使得整体线程的资源分配更加均衡。

`任务队列所属的线程`是`LIFO`的任务生产消费模式，刚好`符合递归任务的执行顺序`。

至此你有没有对ForkJoin框架的**轻量级调度**和**Work−Stealing算法**有一些了解呀：）

### 文章来源：https://blog.csdn.net/Monica2333/article/details/106172279