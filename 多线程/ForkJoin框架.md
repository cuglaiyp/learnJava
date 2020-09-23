# Fork/Join框架

## 1. 概述

最近在看Fork/Join框架，说实话有点难。ForkJoinPool在1.7引入，它只被用来运行ForkJoinTask的子类任务。这个线程池和其他的线程池的不同之处在于它使用分而治之和工作窃取算法去执行任务。有效的去处理大多数任务能衍生出小任务的问题。笔者也是刚接触ForkJoinPool，这个类比较复杂如果有错误还望指正。

## 2. 框架

Fork/Join框架在Java中的具体实现是ForkJoinPool。我们来看一下在ForkJoinPool中一个核心的算法：工作窃取算法

### 2.1 工作窃取算法

看看维基百科对工作窃取算法的描述：在并行计算中，工作窃取是多线程计算机程序的调度策略。它解决了在具有固定数量的处理器（或内核）的静态多线程计算机上执行动态多线程计算的问题，该计算可以“产生”新的执行线程。它在执行时间，内存使用和处理器间通信方面都非常有效。

一般是一个双端队列和一个工作线程绑定，如下图所示。工作线程从绑定的队列的头部取任务执行，从别的队列（一般是随机）的底部偷取任务。

<img src="ForkJoin%E6%A1%86%E6%9E%B6.assets/%E5%B7%A5%E4%BD%9C%E7%AA%83%E5%8F%96%E7%AE%97%E6%B3%95%E5%9B%BE.png" style="zoom:50%;" />

### 2.2 ForkJoinPool类的结构图

<img src="ForkJoin%E6%A1%86%E6%9E%B6.assets/ForkJoinPool%E7%BB%93%E6%9E%84%E5%9B%BE.png" style="zoom:30%;" />

可以看到ForkJoinPool中有很多重要的成员变量，其中有一个WorkQueue数组，数组中存放的是WorkQueue对象，每一个WorkQueue对象也有自己的属性，然后还有一个ForkJoinTask数组array，task都是存在数组中间的。偶数下标的存放的都是外部提交的任务，而奇数下标都是fork出来的子任务。

### 2.3 ForkJoinPool 继承关系

内部类介绍：

1. **ForkJoinWorkerThreadFactory**：内部线程工厂接口，用于创建工作线程ForkJoinWorkerThread
2. **DefaultForkJoinWorkerThreadFactory**：ForkJoinWorkerThreadFactory 的默认实现类
3. **InnocuousForkJoinWorkerThreadFactory**：实现了 ForkJoinWorkerThreadFactory，无许可线程工厂，当系统变量中有系统安全管理相关属性时，默认使用这个工厂创建工作线程。
4. **EmptyTask**：内部占位类，用于替换队列中 join 的任务。
5. **ManagedBlocker**：为 ForkJoinPool 中的任务提供扩展管理并行数的接口，一般用在可能会阻塞的任务（如在 Phaser 中用于等待 phase 到下一个generation）。
6. **WorkQueue**：ForkJoinPool 的核心数据结构，本质上是work-stealing 模式的双端任务队列，内部存放 ForkJoinTask 对象任务，使用 @Contented 注解修饰防止伪共享。具体介绍见上篇。
   - 工作线程在运行中产生新的任务（通常是因为调用了 fork()）时，此时可以把 WorkQueue 的数据结构视为一个栈，新的任务会放入栈顶（top 位）；工作线程在处理自己工作队列的任务时，按照 LIFO 的顺序。
   - 工作线程在处理自己的工作队列同时，会尝试窃取一个任务（可能是来自于刚刚提交到 pool 的任务，或是来自于其他工作线程的队列任务），此时可以把 WorkQueue 的数据结构视为一个 FIFO 的队列，窃取的任务位于其他线程的工作队列的队首（base位）。

>  伪共享状态：缓存系统中是以缓存行（cache line）为单位存储的。缓存行是2的整数幂个连续字节，一般为32-256个字节。最常见的缓存行大小是64个字节。当多线程修改互相独立的变量时，如果这些变量共享同一个缓存行，就会无意中影响彼此的性能，这就是伪共享。

### 2.4 成员变量

1. ForkJoinPool 与 内部类 WorkQueue 共享的一些常量：

   ~~~java
   // Constants shared across ForkJoinPool and WorkQueue
    
   // 限定参数
   static final int SMASK = 0xffff;        //  低位掩码，也是最大索引位（低16位1）
   static final int MAX_CAP = 0x7fff;        //  工作线程最大容量
   static final int EVENMASK = 0xfffe;        //  偶数低位掩码
   static final int SQMASK = 0x007e;        //  workQueues 数组最多64个槽位
    
   // ctl 子域和 WorkQueue.scanState 的掩码和标志位
   static final int SCANNING = 1;             // 标记是否正在运行任务
   static final int INACTIVE = 1 << 31;       // 失活状态(负数)
   static final int SS_SEQ = 1 << 16;         // 版本戳，防止ABA问题
    
   // ForkJoinPool.config 和 WorkQueue.config 的配置信息标记
   static final int MODE_MASK = 0xffff << 16;  // 模式掩码
   static final int LIFO_QUEUE = 0; //LIFO队列
   static final int FIFO_QUEUE = 1 << 16;//FIFO队列
   static final int SHARED_QUEUE = 1 << 31;  // 共享模式队列，负数
   ~~~

2. ForkJoinPool 中的相关常量和实例字段：

   ~~~java
   //  低位和高位掩码
   private static final long SP_MASK = 0xffffffffL;
   private static final long UC_MASK = ~SP_MASK;
    
   // 活跃线程数
   private static final int AC_SHIFT = 48; 
   private static final long AC_UNIT = 0x0001L << AC_SHIFT; //活跃线程数增量
   private static final long AC_MASK = 0xffffL << AC_SHIFT; //活跃线程数掩码
    
   // 工作线程数
   private static final int TC_SHIFT = 32;
   private static final long TC_UNIT = 0x0001L << TC_SHIFT; //工作线程数增量
   private static final long TC_MASK = 0xffffL << TC_SHIFT; //掩码
   private static final long ADD_WORKER = 0x0001L << (TC_SHIFT + 15);  // 创建工作线程标志
    
   // 池状态
   private static final int RSLOCK = 1;
   private static final int RSIGNAL = 1 << 1;
   private static final int STARTED = 1 << 2;
   private static final int STOP = 1 << 29;
   private static final int TERMINATED = 1 << 30;
   private static final int SHUTDOWN = 1 << 31;
    
   // 实例字段
   volatile long ctl;                   // 主控制参数、很重要
   volatile int runState;               // 运行状态锁
   final int config;                    // 并行度|队列模式
   int indexSeed;                       // 用于生成工作线程索引
   volatile WorkQueue[] workQueues;     // 主对象注册信息，workQueue
   final ForkJoinWorkerThreadFactory factory;// 线程工厂
   final UncaughtExceptionHandler ueh;  // 每个工作线程的异常信息
   final String workerNamePrefix;       // 用于创建工作线程的名称
   volatile AtomicLong stealCounter;    // 偷取任务总数，也可作为同步监视器
    
   /** 静态初始化字段 */
   //线程工厂
   public static final ForkJoinWorkerThreadFactory defaultForkJoinWorkerThreadFactory;
   //启动或杀死线程的方法调用者的权限
   private static final RuntimePermission modifyThreadPermission;
   // 公共静态pool
   static final ForkJoinPool common;
   //并行度，对应内部common池
   static final int commonParallelism;
   //备用线程数，在tryCompensate中使用
   private static int commonMaxSpares;
   //创建workerNamePrefix(工作线程名称前缀)时的序号
   private static int poolNumberSequence;
   //线程阻塞等待新的任务的超时值(以纳秒为单位)，默认2秒
   private static final long IDLE_TIMEOUT = 2000L * 1000L * 1000L; // 2sec
   //空闲超时时间，防止timer未命中
   private static final long TIMEOUT_SLOP = 20L * 1000L * 1000L;  // 20ms
   //默认备用线程数
   private static final int DEFAULT_COMMON_MAX_SPARES = 256;
   //阻塞前自旋的次数，用在在awaitRunStateLock和awaitWork中
   private static final int SPINS  = 0;
   //indexSeed的量
   private static final int SEED_INCREMENT = 0x9e3779b9;
   ~~~

3. ForkJoinPool.WorkQueue 中的相关属性：

   ~~~java
   ![ForkJoinPool-ctl](Java%E6%BA%90%E7%A0%81.assets/ForkJoinPool-ctl.png)//初始队列容量，2的幂
   static final int INITIAL_QUEUE_CAPACITY = 1 << 13;
   //最大队列容量
   static final int MAXIMUM_QUEUE_CAPACITY = 1 << 26; // 64M
    
   // 实例字段
   volatile int scanState;    // Woker状态, <0: inactive; odd:scanning
   int stackPred;             // 记录前一个栈顶的ctl
   int nsteals;               // 偷取任务数
   int hint;                  // 记录偷取者索引，初始为随机索引
   int config;                // 池索引和模式
   volatile int qlock;        // 1: locked, < 0: terminate; else 0
   volatile int base;         //下一个poll操作的索引（栈底/队列头）
   int top;                   //  下一个push操作的索引（栈顶/队列尾）
   ForkJoinTask<?>[] array;   // 任务数组
   final ForkJoinPool pool;   // the containing pool (may be null)
   final ForkJoinWorkerThread owner; // 当前工作队列的工作线程，共享模式下为null
   volatile Thread parker;    // 调用park阻塞期间为owner，其他情况为null
   volatile ForkJoinTask<?> currentJoin;  // 记录被join过来的任务
   volatile ForkJoinTask<?> currentSteal; // 记录从其他工作队列偷取过来的任务
   ~~~


看一下比较重要的成员属性：**ctl**。这个成员变量是ForkJoinPool的控制中心，它是一个long型变量，大概分成4份，来记录相关信息。先来看一张图：

<img src="ForkJoin%E6%A1%86%E6%9E%B6.assets/ForkJoinPool-ctl.png" style="zoom:50%;" />

可以看到，ctl实际上是分成3份的，由于低32位记录的是一个WorkQueue的scanState，而这个scanState又用低16位记录着自己的WorkQueue在WorkQueue[] 中的下标，所以可以看成是四份。我们来详细说明这四份：

- 高16位：值是 **活跃线程数 - 并行度** ，用以表示活跃线程数（AC）。初始时，没有活跃线程数，所以这个区域的值为 **负并行度**，每增加1个活跃线程，就会在这个区域 +1，直到为0，也就是活跃线程数 = 并行度为止。
- 中高16位：值是 **总线程数 - 并行度** ，用以表示总线程数（TC）。初始时，没有线程，所以这个区域的值为 **负并行度**，每增加1个线程，就会在这个区域 +1，直到为0，也就是总线程数 = 并行度为止。
- 中低16位：因为WorkQueue的scanState在绑定线程的时候，初始化为当前WorkQueue在WorkQueue[] 中的下标，而WorkQueue[]的长度通过MAX_CAP可知，只用低16位就可以表示了。所以scanState的高16位没有信息，所以作者索性就用这16位中最高位为该线程的状态位，剩下的15位用作该线程的版本号。
- 低16位：由上述可知，为最近一次使用的WorkQueue在WorkQueue[] 中的下标。也就是说这16位指着一个空闲的WorkQueue，而这个空闲的WorkQueue中的stackPred又指着前一个空闲的WorkQueue。以此形成了一个栈。

## 3. 源码

以用例驱动源码学习，我们先来看一个ForkJoinPool的简单例子

~~~java
public class CounterTask extends RecursiveTask<Integer> {

    int start;
    int end;

    public CounterTask(int start, int end){
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        int sum  = 0;
        boolean canCompute = (end - start) < 2;
        if(canCompute){
            for (int i = start; i <= end; i++){
                sum += i;
            }
        }else{
            int middle = (start + end)/2;
            CounterTask leftTask = new CounterTask(start, middle);
            CounterTask rightTask = new CounterTask(middle+1, end);
            leftTask.fork();
            rightTask.fork();
            int leftRes = leftTask.join();
            int rightRes = rightTask.join();
            sum = leftRes+ rightRes;
        }
        return sum;
    }


    public static void main(String[] args){
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        CounterTask task = new CounterTask(1,4);
        Future<Integer> future = forkJoinPool.submit(task);    
        System.out.println(future.get());
    }
}

~~~

一个很简单的例子：算出1+2+3+4的结果。由于我们的这个需求的最后结果与计算的过程无关，所以可以用ForkJoinPool来并发执行这个大任务拆散成的小任务。

首先，我们需要把我们的需求编写成ForkJoinPool能够执行的任务，也就是需要继承ForkJoinTask类，这个类有两个子类RecursiveTask（有结果任务）、RecursiveAction（无结果任务）。我们按照只用我们的需求继承这二者之一就行，并且重写compute方法。这样一个任务类就创建好了

然后新建池对象、新建任务对象，把任务用池对象进行提交即可。提交方法又分为：有结果任务提交（submit）和无结果任务提交（execute）。但是本质上无结果任务提交也是调用的有结果任务提交。下面是执行流程图：

<img src="ForkJoin%E6%A1%86%E6%9E%B6.assets/ForkJoinPool%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%E5%9B%BE.png" style="zoom:60%;" />

所以我们主要看一下有结果任务提交：submit方法。

### 3.1 submit(ForkJoinTask)

~~~java
public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task) {
    if (task == null)
        throw new NullPointerException();
    externalPush(task);
    return task;
}
~~~

很简单一方法，就是调用了externalPush方法

#### 3.1.1 externalPush(ForkJoinTask)

~~~java
final void externalPush(ForkJoinTask<?> task) {
    WorkQueue[] ws; WorkQueue q; int m;
    // 拿到线程相关的随机数（称作探针）
    int r = ThreadLocalRandom.getProbe();
    // 拿到池的运行时状态。看看池状态是不是正常、有没有被别的线程上锁什么的
    int rs = runState;
    // 判断WorkQueue[]数组是不是为空，并且通过探针r找到的偶数桶位置上是否为空
    if ((ws = workQueues) != null && (m = (ws.length - 1)) >= 0 &&
        // 这里 && SQMASK是取偶数的意思
        (q = ws[m & r & SQMASK]) != null && r != 0 && rs > 0 &&
        // 前面都满足（WorkQueue[] != null、对应的桶上WorkQueue也不为null），那么就用CAS锁住这个桶
        U.compareAndSwapInt(q, QLOCK, 0, 1)) {
        ForkJoinTask<?>[] a; int am, n, s;
        // 如果WorkQueue[]的数组不为null
        if ((a = q.array) != null &&
            // 并且还有空位
            (am = a.length - 1) > (n = (s = q.top) - q.base)) {
            // 这里是计算这个task在数组里应放位置的地址。
            // s = q.top一般来说是小于 am（数组长度-1），所以 am & s 还是等于 s。
            // s << ASHIFT 就等于 s下标位置距离起始位置的内存大小。（一般一个数组的单元格占4Byte好像）
            // 再加上 数组起始位置的地址ABASE，就等于当前需要放入数组的位置了
            int j = ((am & s) << ASHIFT) + ABASE;
            // 用unsafe的方法把这个task放入数组指定位置（为什么不直接用数组索引放呢？看网上说，这个性能好，速度更快）
            U.putOrderedObject(a, j, task);
            // 把top + 1，以便放下一个task
            U.putOrderedInt(q, QTOP, s + 1);
            // 解锁
            U.putIntVolatile(q, QLOCK, 0);
            // 任务数 <= 1时尝试创建或激活一个工作线程。为什么要有任务数 <= 1这个条件目前还不明朗
            if (n <= 1)
                signalWork(ws, q);
            return;
        }
        // 数组还没有初始化，直接解锁
        U.compareAndSwapInt(q, QLOCK, 1, 0);
    }
    // 简版提交没有成功，因为各项还没有初始化，所以需要完整版提交：先初始化，再进行提交
    externalSubmit(task);
}
~~~

#### 3.1.2 externalSubmit(ForkJoinTask)

完整版的提交方法。会先把相关的进行初始化，然后再进行提交

~~~java
private void externalSubmit(ForkJoinTask<?> task) {
    int r;          
    // 先拿探针
    // 如果 == 0，说明还没有初始化过，需要进行初始化后再拿
    if ((r = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();
        r = ThreadLocalRandom.getProbe();
    }
    // 起一个自旋
    for (;;) {
        WorkQueue[] ws; WorkQueue q; int rs, m, k;
        // move设为false。这个move的意思是：如果通过探针找到的桶已经被上锁或者已经有WorkQueue，即发生了碰撞，就要move，重新找桶
        boolean move = false;
        // runState < 0 说明线程池不正常，需要尝试终止并抛出异常
        if ((rs = runState) < 0) {
            tryTerminate(false, false);    
            throw new RejectedExecutionException();
        }
        // 如果runState & STARTED == 0 说明线程池还没有启动 或者 WorkQueues为空 或者 WorkQueues长度 <= 0
        // 都说明 线程池相应的还没初始化，需要先进行初始化操作
        else if ((rs & STARTED) == 0 ||     
                 ((ws = workQueues) == null || (m = ws.length - 1) < 0)) {
            int ns = 0;
            // 首先用CAS把整个线程池锁起来
            rs = lockRunState();
            try {
                // 再判断一遍线程池是否启动
                if ((rs & STARTED) == 0) {
                    // 没有启动的话，先初始化STEALCOUNTER
                    U.compareAndSwapObject(this, STEALCOUNTER, null,
                                           new AtomicLong());
                    // 取config的低16位。因为config = 并行度|模式，而模式要么为0，要么在第17位，所以这里&操作就是
                    // 取的并行度
                    int p = config & SMASK; 
                    // p - 1是因为要留一个给主线程
                    int n = (p > 1) ? p - 1 : 1;
                    // 这里参考HashMap初始化，就是找到一个比并行度大于等于的最小的（2的幂 - 1）
                    n |= n >>> 1; n |= n >>> 2;  n |= n >>> 4;
                    n |= n >>> 8; n |= n >>> 16; 
                    // 再+1，并扩大两倍。
                    n = (n + 1) << 1;
                    // 初始化WorkQueues
                    workQueues = new WorkQueue[n];
                    ns = STARTED;
                }
            } finally {
                // 解锁，并且把runState | 上STARTED，标志线程池已启动
                unlockRunState(rs, (rs & ~RSLOCK) | ns);
            }
            // 上面执行完之后，就自旋一下
        }
        // 3. 在第1、2次自旋的时候，对workQueues和workQueue进行了初始化，第3次自旋就到这了
        // 这里与上面的简版提交方法类似，是正式提交的地方
        // 先用探针、数组长度-1、偶数掩码，随便找到一个偶数下标k的桶
        else if ((q = ws[k = r & m & SQMASK]) != null) {
            // 如果桶中workQueue不为null
            // 那么判断该桶有没有上锁，没有上锁，则当前线程给它上锁，并进行相应操作
            if (q.qlock == 0 && U.compareAndSwapInt(q, QLOCK, 0, 1)) {
                // 拿到该桶里面的task数组
                ForkJoinTask<?>[] a = q.array;
                // 拿到数组应该放task的下标top
                int s = q.top;
                // 提交完成标记
                boolean submitted = false; 
                try {
                    // 先判断如果桶为null或者大小不足
                    if ((a != null && a.length > s + 1 - q.base) ||
                        // 就growArray方法扩容
                        (a = q.growArray()) != null) {
                        // 计算应填放的下标的地址
                        int j = (((a.length - 1) & s) << ASHIFT) + ABASE;
                        // 填放
                        U.putOrderedObject(a, j, task);
                        // top+1
                        U.putOrderedInt(q, QTOP, s + 1);
                        // 提交完成
                        submitted = true;
                    }
                } finally {
                    // 解锁
                    U.compareAndSwapInt(q, QLOCK, 1, 0);
                }
                // 提交成功
                if (submitted) {
                    // 唤醒worker
                    signalWork(ws, q);
                    return;
                }
            }
            // 如果该桶已经被其他线程上锁了，那么需要move
            move = true;                   
        }
        // 2. 因为第2次自旋，上面的if找到桶里的workQueue为null，所以会跳到这来，初始化workQueue
        // 先判断线程池有没有被锁
        else if (((rs = runState) & RSLOCK) == 0) { // create new queue
            // new一个workQueue出来，会持有当前池对象，并且会把base和top指向array数组的中间
            q = new WorkQueue(this, null);
            // 把这个workQueue的hint设置为自己的探针r
            q.hint = r;
            // 设置它的config为：它在WorkQueues中下标 | 共享模式 （与池的config区分开喔）
            q.config = k | SHARED_QUEUE;
            // 设置它的scanState为INACTIVE，未激活状态
            q.scanState = INACTIVE;
            // 用runState加锁，同时赋值给rs
            rs = lockRunState();        
            // 加锁成功，并且workQueues已经初始化好了，并且下标k也是正常范围
            if (rs > 0 &&  (ws = workQueues) != null &&
                k < ws.length && ws[k] == null)
                // 那么就把这个workQueue放进对应的桶里面
                ws[k] = q;                 
            // 解锁
            unlockRunState(rs, rs & ~RSLOCK);
        }
        // 走到这，说明线程池被其他线程上了锁（上面的move是因为对应的桶被其他线程上锁，注意区分）
        else
            // 那么需要move
            move = true;                  
        // 如果需要move的话，就重新随机一个探针
        if (move)
            r = ThreadLocalRandom.advanceProbe(r);
    }
}
~~~

总结一下这个方法：

1. 就是用自旋，如果WorkQueue[]还没有初始化，先初始化WorkQueue[]，长度一定为2的幂
2. 再初始化workQueue，并把这个workQueue放到WorkQueue[]对应的槽里，因为是外部提交，这个槽的下标一定是偶数
3. 再把task放到workQueue中task数组中top下标处，并且激活一个空闲线程或者创建一个新线程，用来处理任务

我们再来看一下激活或者创建线程的方法：signalWork方法

#### 3.1.3 signalWork(WorkQueue, WorkQueue)

~~~java
final void signalWork(WorkQueue[] ws, WorkQueue q) {
    long c; int sp, i; WorkQueue v; Thread p;
   	// 因为ctl最高16位为：活跃线程数 - parallelism。所以当活跃线程数不足时，ctl就 < 0
    while ((c = ctl) < 0L) {
        // 取ctl低32进行判断（初始的时候，ctl低32位全为0），因为低32位存储的是空闲队列的信息，所以：
        // 		!= 0，那说明有空闲队列，执行唤醒操作
        // 		== 0，说明没有空闲队列，执行添加操作
        if ((sp = (int)c) == 0) {    
            // == 0，准备执行添加操作
            // 先判断总线程数是否达到并行度。这个ADD_WORKER就是第48位为1，与ctl相与，可以判断出ctl第48位是否为1：
            // 		!= 0：为1，也就是说总线程数还没有达到并行度，可以添加
            // 		== 0：不为1，也就说总线程数已经达到了并行度，不能再添加线程了
            if ((c & ADD_WORKER) != 0L)            
                // != 0，尝试添加worker
                tryAddWorker(c);
            break;
        }
        // 走到这，说明ctl记录了一个空闲WorkQueue
        // 判断一下WorkQueue[]是否为null，这个为null，肯定不能继续，对吧
        if (ws == null)                            
            break;
        // 继续判断一下，空闲队列所在WorkQueue[]的下标是否越界，越界了肯定不行，对吧
        if (ws.length <= (i = sp & SMASK))         
            break;
        // 继续判断一下，这个空闲队列是否为null，为null，也没有必要继续，对吧
        if ((v = ws[i]) == null)                   
            break;
        // 前面的条件都满足了，就可以激活这个空闲队列了
        // 首先用这个空闲队列的版本号+1，防止ABA问题，然后再把激活位置0，用临时变量vs存起来。
        int vs = (sp + SS_SEQ) & ~INACTIVE;        
        // ctl中低32位就是空闲队列的scanState，又通过这个scanState中的下标找到了这个空闲队列v，
        // 所以这里的d == 0
        int d = sp - v.scanState;
        // ctl的活跃线程数 +1（因为是激活，总线程数不变），低32位，变成刚刚那个空闲队列所指向的下一个空闲队列
        long nc = (UC_MASK & (c + AC_UNIT)) | (SP_MASK & v.stackPred);
        // d == 0没什么问题，就用CAS把ctl更新一下
        if (d == 0 && U.compareAndSwapLong(this, CTL, c, nc)) {
            // ctl已经更新了，把临时变量vs赋值给v的scanState，把它的scanState也更新一下
            v.scanState = vs;                      
            // 标记都做好了，接下来就始是把线程真正激活
            // 拿到v上等待的线程，激活（这个parker就是与这个workQueue绑定的线程）
            if ((p = v.parker) != null)
                U.unpark(p);
            // 跳出循环
            break;
        }
        // 如果添加线程失败、或者唤醒空闲线程失败
        // 那么判断一下传进来的workQueue里面有没有task
        if (q != null && q.base == q.top)         
            // 没有的话，也直接跳出了，不需要添加或者唤醒了
            break;
    }
}
~~~

总结一下这个方法：

活跃线程数不足并行度的时候：

1. 如果有空闲worker（WorkQueue），就唤醒相应的线程
2. 没有的话，如果总线程数还没有达到并行度的话，那么就进行添加操作

##### 3.1.3.1 tryAddWorker(int)

我们来看看这个方法是怎么添加worker的

~~~java
private void tryAddWorker(long c) {
    // 添加完成标记位
    boolean add = false;
    do {
        // newCtl，给ctl的活跃线程数+1，总线程数+1，低位全为0，不变
        long nc = ((AC_MASK & (c + AC_UNIT)) |
                   (TC_MASK & (c + TC_UNIT)));
        // 判断一下，确保ctl没有被别的线程修改
        if (ctl == c) {
            int rs, stop;                 
            // 给整个线程池上锁，并且判断线程池运行状态不是stop的话
            if ((stop = (rs = lockRunState()) & STOP) == 0)
                // 用刚刚的nc更新ctl，并返回成功信号给add
                add = U.compareAndSwapLong(this, CTL, c, nc);
            // 解锁
            unlockRunState(rs, rs & ~RSLOCK);
            // stop != 0的话，说明线程池已经停了，直接break
            if (stop != 0)
                break;
            // add为true，意味着ctl更新完成，可以真正创建一个worker出来
            if (add) {
              	// 创建worker
                createWorker();
               	// 创建完了break
                break;
            }
        }
        // 这个循环，要么只有池状态异常，或者成功添加worker，从内部break
        //         要么线程线程总数已经达到最大，或者有空闲WorkQueue的时候才能退出
    } while (((c = ctl) & ADD_WORKER) != 0L && (int)c == 0);
}


private boolean createWorker() {
    ForkJoinWorkerThreadFactory fac = factory;
    Throwable ex = null;
    ForkJoinWorkerThread wt = null;
    try {
        // 很简单的逻辑，就是用线程工厂，创建一个线程出来
        if (fac != null && (wt = fac.newThread(this)) != null) {
            // 创建成功，就启动这个线程
            wt.start();
            return true;
        }
    } catch (Throwable rex) {
        ex = rex;
    }
    deregisterWorker(wt, ex);
    return false;
}


public final ForkJoinWorkerThread newThread(ForkJoinPool pool) {
    // 就很简单的new了一个线程出来，封装了当前线程池参数
    return new ForkJoinWorkerThread(pool);
}


// 我们继续往下看，看ForkJoinWorkerThread类的构造方法
protected ForkJoinWorkerThread(ForkJoinPool pool) {
    // 交给父类初始化名称
    super("aForkJoinWorkerThread");
    // 封装池参数
    this.pool = pool;
   	// 这个线程里面维护了一个workQueue对象，这个对象是池对象调用registerWorker，把当前线程当参数传进去，所返回的
    // 这个方法有点绕。大概意思就是：
    // 		WorkQueue[]里有个WorkQueue，WorkQueue里持有了当前线程对象，当前线程有维护了这个WorkQueue，绑定在一起了
    this.workQueue = pool.registerWorker(this);
}


//接着看是不是这样的
final WorkQueue registerWorker(ForkJoinWorkerThread wt) {
    UncaughtExceptionHandler handler;
    // 把当前线程设置为守护线程，主线程结束，它就挂掉
    wt.setDaemon(true);                         
    if ((handler = ueh) != null)
        wt.setUncaughtExceptionHandler(handler);
    // new一个WorkQueue，封装了当前池对象和线程对象！！ 你看我们上面分析得没错
    WorkQueue w = new WorkQueue(this, wt);
    int i = 0;               
    // 通过池的config（并行度|队列进出模式），拿到池中队列的进出模式
    int mode = config & MODE_MASK;
    // 对整个池上锁
    int rs = lockRunState();
    try {
        WorkQueue[] ws; int n; 
        if ((ws = workQueues) != null && (n = ws.length) > 0) {
            // 如果池已经初始化好
            // indexSeed在刚开始的时候没有初始化，为0，给它加一个很怪的数，得到s，用来计算下标。不太可能碰撞了
            int s = indexSeed += SEED_INCREMENT;  // unlikely to collide
            // 数组长度 - 1。就低位全是1，做掩码
            int m = n - 1;
            // 把s扩大一倍。再或上1，保证为奇数。与上m，相当于对 数组长度 取余操作，因为数组长度为偶数，所以余数必为奇数
            i = ((s << 1) | 1) & m;               
            // 判断找到的下标位置是否为空（是否碰撞）
            if (ws[i] != null) {              
                // 发生碰撞
             	// 定义一个探测（查找）次数   
                int probes = 0;
                // 定义步长，大约取数组长度的一半的偶数。这么定义步长是为了，让我们的所有探查均匀地分布在数组的奇数下标处
                int step = (n <= 4) ? 2 : ((n >>> 1) & EVENMASK) + 2;
                // 通过步长循环查找下一个位置
                while (ws[i = (i + step) & m] != null) {
                    // 又发生了碰撞
                    // 看一下查找次数是否超过了数组长度
                    if (++probes >= n) {
                        // 如果超过了，那么说明要对WorkQueue[]进行扩容
                        workQueues = ws = Arrays.copyOf(ws, n <<= 1);
                        // 重新复制掩码
                        m = n - 1;
                        // 令探测次数为0
                        probes = 0;
                    }
                }
                // 走到这说明，已经找到一个没有发生碰撞的奇数下标位置了
            }
            // 初始化workQueue其他的一些属性
            w.hint = s;  
            // config记录下标和队列共享模式
            w.config = i | mode;
            // scanState记录下标
            w.scanState = i;        
            // workQueue填入WorkQueue[]
            ws[i] = w;
        }
    } finally {
        // workQueue绑定好线程、初始化完成，解锁
        unlockRunState(rs, rs & ~RSLOCK);
    }
    // 给线程设置一个名称
    wt.setName(workerNamePrefix.concat(Integer.toString(i >>> 1)));
    // 返回workQueue给线程的构造方法，让线程也维护这个workQueue，完成双向绑定（注册）
    return w;
}
~~~

##### 3.1.3.2 总结signalWork方法

1. 如果有空闲线程，那么就唤醒空闲线程
2. 如果没有空闲线程，如果还能够添加线程：
   - 就使用线程工厂创建一个线程
   - 创建的时候，创建一个workQueue，放置在WorkQueue[]的奇数下标处，与该线程进行双向绑定
   - 最后启动线程

#### 3.1.4 submit方法总结

1. 先用简版externalPush方法进行提交
2. 如果简版方法提交失败，调用完整版externalSubmit方法进行提交
   - 如果WorkQueue[]还没有初始化，先初始化WorkQueue[]
   - 如果对应偶数槽的WorkQueue还没有，接着创建对应槽的WorkQueue
   - 把任务放到WorkQueue的task数组中
3. 用signalWork方法，唤醒一个空闲线程或者创建一个新的线程来执行任务

上面已经分析完了，外部任务提交主线程所做的一些操作。接着，我们看一下，线程被唤醒或者被创建出来、启动之后进行了哪些操作

### 3.2 ForkJoinWorkerThread的run()

线程启动后，就会执行它的run方法。我们来看一下run方法里面的逻辑

~~~java
public void run() {
    // 由于没有自旋，这里只会执行一次
    // 先判断自己维护的workQueue是否为空
    if (workQueue.array == null) { 
        Throwable exception = null;
        try {
            // 空方法，钩子方法，留给子类重写
            onStart();
            // 用池的runWorker方法来执行自己维护的workQueue
            // 这里与上面线程池的逻辑是类似的，也是不能直接调用任务的run方法，而是要在执行任务的run方法前后添加一些线程池相关的逻辑
            pool.runWorker(workQueue);
        } catch (Throwable ex) {
            exception = ex;
        } finally {
            try {// 处理异常
                onTermination(exception);
            } catch (Throwable ex) {
                if (exception == null)
                    exception = ex;
            } finally {
                // 走到这，说明该销毁线程了
                pool.deregisterWorker(this, exception);
            }
        }
    }
}
~~~

#### 3.2.1 runWorker(WorkQueue)

这是线程池的方法

~~~java
final void runWorker(WorkQueue w) {
    // 因为每个线程的runWorker方法只执行一次。所以这里就是给workQueue的task数组分配控件
    w.growArray();                  
    // 拿到seed
    int seed = w.hint;
    // 确保seed不为0
    int r = (seed == 0) ? 1 : seed; 
    // 起一个自旋
    for (ForkJoinTask<?> t;;) {
        // 通过scan方法窃取task
        if ((t = scan(w, r)) != null)
            // 用workQueue的runTask来执行这个任务
            w.runTask(t);
        // 如果没有窃取到task，那么就让线程阻塞，等待唤醒
        else if (!awaitWork(w, r))
            // 如果阻塞线程失败跳出循环
            break;
        // 线程阻塞被唤醒后，更新一下seed，重新尝试窃取任务
        r ^= r << 13; r ^= r >>> 17; r ^= r << 5; // xorshift
    }
}
~~~

这个代码很简单，就是线程启动后，就不断的用scan方法去窃取任务，然后执行这个任务。如果没有窃取到，就阻塞，等待唤醒。

那我们就接着看一下窃取任务的核心方法

##### 3.2.1.1 scan(WorkQueue, int)

~~~java
/**
 * 此方法是任务窃取的核心方法
 * w：是当前线程所绑定的WorkQueue
 * r：是WorkQueue的hint，hint又是indexSeed += SEED_INCREMENT。总而言之，就是一个很奇怪的数字
*/
private ForkJoinTask<?> scan(WorkQueue w, int r) {
    WorkQueue[] ws; int m;
    // 先检查WorkQueue[]有没有初始化
    if ((ws = workQueues) != null && (m = ws.length - 1) > 0 && w != null) {
        // 拿到WorkQueue的scanState。初始化的时候，这个scanState是这个WorkQueue在WorkQueue[]的下标，是个正奇数。
        // scanState只有在这个WorkQueue（线程）失活的时候为负
        int ss = w.scanState;             
        // 起一个自旋。
      	// 解释一下这几个变量：
        // 		origin：  用r这个很怪的数字，随机找的起始下标
        //		k：       工作索引，从origin下标开始
        // 		oldSum：  记住上一次扫描探测标记的和，用于和checkSum比较
        // 		checkSum：记录每一遍探测标记的和 （这个标记就是WorkQueue中base的值）
        for (int origin = r & m, k = origin, oldSum = 0, checkSum = 0;;) {
            WorkQueue q; ForkJoinTask<?>[] a; ForkJoinTask<?> t;
            int b, n; long c;
            // 1. 取到k下标的WorkQueue为q。尝试偷取它的任务
            if ((q = ws[k]) != null) {
                // q不为null，说明这个WorkQueue可能存在任务，能够偷取
                // 2. 判断一下WorkQueue的task数组是否已经有任务
                if ((n = (b = q.base) - q.top) < 0 &&
                    (a = q.array) != null) { 
                    // 数组已有任务，进入if体，进行偷取
                    // 计算一下base下标位置的地址
                    long i = (((a.length - 1) & b) << ASHIFT) + ABASE;
                    // 3. 拿出base下标处的task
                    if ((t = ((ForkJoinTask<?>)
                              U.getObjectVolatile(a, i))) != null &&
                        q.base == b) {
                        // task不为null，进入if体
                        // 4. 有任务能偷取，那么先判断一下自己有没有失活
                        if (ss >= 0) {
                            // 如果ss >= 0，说明该WorkQueue（线程）没有失活，进入if体
                            // 5. 因为base处的task已被变量t保存，以便偷取，所以这里置为null，gc
                            if (U.compareAndSwapObject(a, i, t, null)) {
                                // base+1
                                q.base = b + 1;
                                // 6. 如果task数组里面还有任务，那么需要signal另外的线程来进行处理
                                if (n < -1)       
                                    signalWork(ws, q);
                                // 返回偷到的任务
                                return t;
                            }
                        }
                        // 走到这，说明虽然能偷的任务不为null，但是ss < 0，当前WorkQueue（线程）已经失活
                        // 7. 那么判断一下 oldSum是不是等于0，同时再检测一下自己现在失活没有
                        else if (oldSum == 0 &&   
                                 w.scanState < 0)
                            // 都满足的话，尝试激活ctl记录的栈顶空闲队列（线程）去偷这个任务（自己偷不了，喊别人偷）。 
                            tryRelease(c = ctl, ws[m & (int)c], AC_UNIT);// 这个方法下面讲。
                    }
                    // 走到这，说明：
                    // 		1.要么是自己偷到的任务是null。（可能被其他线程先偷一步，导致自己偷了空的）
                    // 		2.要么之前自己失活了，有任务偷不了
                    // 8. 那么这里就判断一下，看看之前自己是不是失活了
                    if (ss < 0)                  
                        //如果之前自己是失活状态，那么更新一下ss。（说不定自己又被其他线程唤醒，scanState这时候为正）
                        ss = w.scanState;
                    // 走到这，说明：不管是探测的任务为null也好，自己失活也罢，反正任务是偷失败了
                    // 那么更新一下相关参数，为重新扫描探测做准备
                    // 更新一下自己的奇怪随机数
                    r ^= r << 1; r ^= r >>> 3; r ^= r << 10;
                    // 重新计算origin
                    origin = k = r & m;           
                    // 重置oldSum和checkSum
                    oldSum = checkSum = 0;
                    continue;
                }
                // 走到这，说明扫描探测的WorkQueue q为null，把checkSum加上一个b（base）的值
                // 这里因为探测到的q为null，所以b为默认值0
                checkSum += b;
            }
          	// 任务没偷成功，进行下一个if的判断
            // 9. k+1，判断是否已经探测一圈回到起点
            if ((k = (k + 1) & m) == origin) {    
                // 探测了一圈，回到了起点
                // 10. 判断一下自己的存活状态和探测状态
                if ((ss >= 0 || (ss == (ss = w.scanState))) &&
                    oldSum == (oldSum = checkSum)) {
                    // 如果自己还存活或者自己的状态没有改变，并且自己已经探测了两圈了，而且两圈都没有任务
                    // 11. 那么判断一下自己是否失活，或者当前WorkQueue是否已经终止
                    if (ss < 0 || w.qlock < 0)  
                        // 自己已经失活或者已经终止，那么跳出自旋即可，需要其他操作
                        break;
                    // 走到这，说明自己还活着，并且自己扫描探测WorkQueues两遍都没有发现任务
                    // 那么自己也不能老在那空跑浪费精力，所以把自己灭活，去休息（这里只是从控制信息上面灭活，线程真正的阻塞要等到runWorker方法中，调用的awaitWork方法）
                    // newScanState，置scanState最高位为1
                    int ns = ss | INACTIVE;       
                    // newCtl，保存newScanState和活跃线程数-1
                    long nc = ((SP_MASK & ns) |
                               (UC_MASK & ((c = ctl) - AC_UNIT)));
                    // 然后当前WorkQueue指着之前的栈顶空闲WorkQueue，自己就成了新的栈顶空闲WorkQueue
                    w.stackPred = (int)c;         
                    // 更新自己的scanState为newScanState
                    U.putInt(w, QSCANSTATE, ns);
                    // 更新ctl
                    if (U.compareAndSwapLong(this, CTL, c, nc))
                        // 更新成功的话，把局部ss也更新一下
                        ss = ns;
                    else
                        // 更新失败，要把w的scanState恢复原状
                        w.scanState = ss;         
                }
                // 最后把checkSum重置，再自旋，重新探测一圈，没有任务才能退出自旋，返回null到runWork方法中进行阻塞
                checkSum = 0;
            }
            // 没有的话，因为k已经加1，继续探测
        }
    }
    return null;
}
~~~

这个方法很长，也很核心，我们下面来总结一下这个方法的脉络：

这个方法主要做的就是：**随机找一个下标为起点，遍历探测WorkQueue[]，如果有WorkQueue，它的任务数组里面有任务，并且自身状态正常，那么就直接偷取，返回**。正常逻辑就是这么简单，但是，还有其他很多情况，比如：**没有任务、刚探测过的槽又被别的线程添加了任务、自身状态不正常**等等，所以需要很多额外的逻辑去处理这些情况。

1. **没有任务**：通过上面的流程，我们可以看到，oldSum = 0的时候为第一遍循环探测，探测完会将checkSum的值给oldSum；那么第二遍的探测，标志是checkSum != 0。如果没有任务，第二遍的checkSum就会与第一遍的值相同，既然没任务，就会进入9. 对自己进行灭活（标记灭活，线程还没有阻塞）。灭活之后，会再循环探测一遍，所以第三遍探测的标志是oldSum  != 0 && 灭活。没有任务的话，checkSum还是等于oldSum，线程进入11. break自旋，最后返回null。

2. **刚探测过的槽又被别的线程添加了任务**：第一遍刚探测完的槽被添加了任务，第二遍探测的时候发现；第二遍刚探测完的槽被添加了任务，等第三遍灭活后探测发现，进行相应的逻辑处理。可以看到，只要添加了任务，在2. if处通过base - top就能发现，那么checkSum这个值到底有什么用呢？

   可以看到每次探测，checkSum += base，说明只有base变了，后面的checkSum才不会等于前面的checkSum。那么我们看一下什么情况下base会变：

   - 另一个线程提交了任务，新建了一个WorkQueue，该WorkQueue的base会初始化，导致当前线程下一圈探测的checkSum与前一圈的不同
   - 另一个线程提交了任务，创建了一个线程，奇数位新建了一个WorkQueue，导致checkSum变化
   - 另一个线程提交了任务，但这个任务又被窃取了，这个时候base - top检测不出来，但是base发生变化，checkSum能检测出来

   可以看到上面三种情况，都与探测过程中任务的提交有关，所以当checkSum发生变化的时候，就需要重新再探测一遍，看能不能获取到任务：第二遍checkSum发生了变化，就再进行一次第二遍的探测；第三遍失活后的探测checkSum发生了变化，就再进行一下第三次失活后的探测。

3. **自身状态不正常**：线程灭活后，发现任务会唤醒栈顶线程前来执行任务，自己则通过两遍循环探测，返回null，并阻塞

这个整个逻辑比较复杂，这里只是简单的做一些梳理。

偷取了任务，我们再来看一下，怎么执行的

##### 3.2.1.2 runTask(ForkJoinTask)

~~~java
// 此方法为WorkQueue的方法
final void runTask(ForkJoinTask<?> task) {
    if (task != null) {
        // 先设置线程状态（线程绑定一个WorkQueue，所以可以不区分），为繁忙
        scanState &= ~SCANNING; // mark as busy
        // 执行偷取来任务，并封装结果
        (currentSteal = task).doExec();
        U.putOrderedObject(this, QCURRENTSTEAL, null); // release for GC
        // 执行自己任务队列的任务
        execLocalTasks();
        // 拿到自己WorkQueue所属的线程（如果为shared模式，则该值为null）
        ForkJoinWorkerThread thread = owner;
        // 记录偷取数
        if (++nsteals < 0)      // collect on overflow
            // 数太大越界，就转移到pool的计数器上
            transferStealCount(pool);
        // 复位scanState
        scanState |= SCANNING;
        if (thread != null)
            // 钩子方法，一般感觉用不着
            thread.afterTopLevelExec();
    }
}
~~~

这个方法逻辑比较简单，里面几个简单的方法也就不展开讲了，很容易懂。

##### 3.2.1.3 awaitWork(WorkQueue, int)

~~~java
/** 
 * 线程实际上的阻塞就是在这个方法里面进行的。
 * w：该线程绑定的WorkQueue
 * r：随机探测种子
 */
private boolean awaitWork(WorkQueue w, int r) {
    // 先判断一下状态
    if (w == null || w.qlock < 0)                 // w is terminating
        return false;
    // 起一个自旋，拿到当前WorkQueue所记录的空闲线程，拿到阻塞前自旋的次数（SPINS一般为0,）
    for (int pred = w.stackPred, spins = SPINS, ss;;) {
        // 判断一下该线程在标志位上是否已经失活
        if ((ss = w.scanState) >= 0)
            // 没有失活那么不够条件阻塞，直接跳出自旋，返回true
            break;
        // 如果在标志位上已经失活，再判断一下，阻塞前需要自旋的次数。
        else if (spins > 0) {
            // 如果需要自旋
            // 更新一下探测种子
            r ^= r << 6; r ^= r >>> 21; r ^= r << 7;
            // 对spin--。等自己自旋完（spin == 0）判断一下，是否还要更新一下spins，继续一轮的自旋
            if (r >= 0 && --spins == 0) {         // randomize spins
                WorkQueue v; WorkQueue[] ws; int s, j; AtomicLong sc;
                // 检查当前WorkQueue所记录的空闲线程（标志位已经失活），是否真正阻塞了。
                // 		是：那么不用更新spins，继续一轮自旋
                //      否：那么需要更新spins，继续自旋，等待自己前面的线程，先阻塞。
                if (pred != 0 && (ws = workQueues) != null &&
                    (j = pred & SMASK) < ws.length &&
                    (v = ws[j]) != null &&        // see if pred parking
                    (v.parker == null || v.scanState >= 0))
                    spins = SPINS;                // continue spinning
            }
        }
        // 不需要自旋或者自旋完毕，会判断这个if，检查自身状态是否正常
        else if (w.qlock < 0)                     // recheck after spins
            return false;
        // 状态正常，也不需要空转自旋了，那么就再判断一下当前线程是否有被中断
        else if (!Thread.interrupted()) {
            // 一切条件正常，那么进入这个if进行阻塞操作
            long c, prevctl, parkTime, deadline;
            // 计算出活跃线程数
            int ac = (int)((c = ctl) >> AC_SHIFT) + (config & SMASK);
            // 	如果线程池状态不正常，返回false
            if ((ac <= 0 && tryTerminate(false, false)) ||
                (runState & STOP) != 0)           // pool terminating
                return false;
            // 活跃线程没有了且如果自己是栈顶空闲线程
            if (ac <= 0 && ss == (int)c) {        // is last waiter
                // 构造出前一个ctl：活跃线程数+1 | 栈顶第二个scanState
                prevctl = (UC_MASK & (c + AC_UNIT)) | (SP_MASK & pred);
                // 拿到 并行度 - 总线程数
                int t = (short)(c >>> TC_SHIFT);  // shrink excess spares
                // 如果 并行度 - 总线数 > 2，并且更换ctl成功
                // 		这里解释一下：增加线程的时候会有判断，当总线程数 == 并行度，就不再增减线程了。所以一般不会出现
                //      t > 0的情况。但是如果出现了并发问题（？）几个线程同时创建了几个线程，那么线程数就可能超出并行度
                //		度，所以就需要把ctl设置为前一个ctl，并且返回false，让这个线程不要自旋、阻塞，而是销毁掉
                if (t > 2 && U.compareAndSwapLong(this, CTL, c, prevctl))
                    return false;                 // else use timed wait
             	// 设置超时等待时间。为什么要设置这个超时等待时间呢？因为这两句话在活跃线程 <= 0的if里。因为活跃线程没有
                // 所以自己不能睡太久
                parkTime = IDLE_TIMEOUT * ((t >= 0) ? 1 : 1 - t);
                deadline = System.nanoTime() + parkTime - TIMEOUT_SLOP;
            }
            // 如果还有活跃线程
            else
                // 那么不需要超时等待了
                prevctl = parkTime = deadline = 0L;
            // 拿到当前线程
            Thread wt = Thread.currentThread();
            // 放置BLOCKER，blocker意思是：线程阻塞在谁身上
            U.putObject(wt, PARKBLOCKER, this);   // emulate LockSupport
            // 设置WorkQueue的parker，标志着线程也真正阻塞了（标志位灭活 -> 线程阻塞）
            w.parker = wt;
            // 重新检查一遍
            if (w.scanState < 0 && ctl == c)      // recheck before park
                // park线程。
                U.park(false, parkTime);
            // 线程被唤醒之后，从这开始执行。置parker为null
            U.putOrderedObject(w, QPARKER, null);
            // 置parkBlock为null
            U.putObject(wt, PARKBLOCKER, null);
            // 如果自己的状态已经正常，就break
            if (w.scanState >= 0)
                break;
            // 走到这说明当前线程不是被唤醒的，而是，线程数溢出，自己阻塞一定时间自己醒来的
            // parkTime != 0也说明线程有多余，再判断一下自己醒来的时间是否超过自己所需要阻塞的时间
            if (parkTime != 0L && ctl == c &&
                deadline - System.nanoTime() <= 0L &&
                // CAS设置ctl为前一个ctl
                U.compareAndSwapLong(this, CTL, c, prevctl))
                // 返回false，让当前线程销毁
                return false;                     // shrink pool
        }
    }
    // 线程正常被唤醒后，返回true
    return true;
}

~~~

看完了awaitWork方法后，我们再回到runWorker方法中，看看返回的true和false究竟是怎么控制线程是继续执行还是销毁

~~~java
// runworker方法的部分：
for (ForkJoinTask<?> t;;) {
    // 通过scan方法窃取task
    if ((t = scan(w, r)) != null)
        // 用workQueue的runTask来执行这个任务
        w.runTask(t);
    // 如果没有窃取到task，那么就让线程阻塞，等待唤醒
    else if (!awaitWork(w, r))
        // 如果阻塞线程失败跳出循环
        break;
    // 线程阻塞被唤醒后，更新一下seed，重新尝试窃取任务
    r ^= r << 13; r ^= r >>> 17; r ^= r << 5; // xorshift
}

// run方法的部分：
	pool.runWorker(workQueue);
	} catch (Throwable ex) {
    	exception = ex;
	} finally {
    	try {// 处理异常
        	onTermination(exception);
    	} catch (Throwable ex) {
        	if (exception == null)
            	exception = ex;
    	} finally {
        	// 走到这，说明该销毁线程了
        	pool.deregisterWorker(this, exception);
    	}
~~~

可以看到

- 返回true的话，线程就继续循环，窃取任务执行
- 返回false的话，线程会跳出循环，然后返回到run方法，最后执行deregisterWorker方法

#### 3.2.2 deregisterWorker(ForkJoinWorkerThread, Throwable)

~~~java
final void deregisterWorker(ForkJoinWorkerThread wt, Throwable ex) {
    WorkQueue w = null;
    // 删除自己所绑定的WorkQueue
    if (wt != null && (w = wt.workQueue) != null) {
        WorkQueue[] ws;                           // remove index from array
        int idx = w.config & SMASK;
        int rs = lockRunState();
        if ((ws = workQueues) != null && ws.length > idx && ws[idx] == w)
            ws[idx] = null;
        unlockRunState(rs, rs & ~RSLOCK);
    }
    long c;                                       // decrement counts
	// CAS修改ctl
    do {} while (!U.compareAndSwapLong
                 (this, CTL, c = ctl, ((AC_MASK & (c - AC_UNIT)) |
                                       (TC_MASK & (c - TC_UNIT)) |
                                       (SP_MASK & c))));
    // 删掉自己WorkQueue里面的任务，置任务状态为cancel
    if (w != null) {
        w.qlock = -1;                             // ensure set
        w.transferStealCount(this);
        w.cancelAll();                            // cancel remaining tasks
    }
    // 自旋，做扫尾工作
    for (;;) {                                    // possibly replace
        WorkQueue[] ws; int m, sp;
        // 如果线程池terminating了，直接break
        if (tryTerminate(false, false) || w == null || w.array == null ||
            (runState & STOP) != 0 || (ws = workQueues) == null ||
            (m = ws.length - 1) < 0)              // already terminating
            break;
        // 如果还有空闲线程，帮忙唤醒一下
        if ((sp = (int)(c = ctl)) != 0) {         // wake up replacement
            if (tryRelease(c, ws[sp & m], AC_UNIT))
                break;
        }
        // 没有空闲线程，但是需要添加线程的话，帮忙添加一下
        else if (ex != null && (c & ADD_WORKER) != 0L) {
            tryAddWorker(c);                      // create replacement
            break;
        }
        // 啥也不用，直接break
        else                                      // don't need replacement
            break;
    }
    if (ex == null)                               // help clean on way out
        ForkJoinTask.helpExpungeStaleExceptions();
    else                                          // rethrow
        ForkJoinTask.rethrow(ex);
}
~~~

### 3.3 小结

到这，我们就梳理完了：一个任务从外部提交，放置到偶数槽，然后提交任务线程创建一个工作线程并且绑定一个奇数槽位，工作线程，循环探测偷取任务，并执行的全过程。接下来，我们就来看一下，一个任务如果粒度太大，需要fork划分成小任务的时候，fork方法到底做了什么。

### 3.4 fork()

我们再来看看前面我们的小例子，看看task粒度过大时，是怎么fork的

~~~java
@Override
protected Integer compute() {
    int sum  = 0;
    boolean canCompute = (end - start) < 2;
    if(canCompute){
        for (int i = start; i <= end; i++){
            sum += i;
        }
    }else{
        int middle = (start + end)/2;
        CounterTask leftTask = new CounterTask(start, middle);
        CounterTask rightTask = new CounterTask(middle+1, end);
        leftTask.fork();
        rightTask.fork();
        int leftRes = leftTask.join();
        int rightRes = rightTask.join();
        sum = leftRes+ rightRes;
    }
    return sum;
}
~~~

可以看到，我们的任务逻辑是重写在compute方法里面的，当任务粒度过大时，就会new两个小的task，并用这两个小task调用一下fork方法。由此，我们大概可以猜出来，这个fork方法应该是类似于ForkJoinPool的添加方法，把任务提交到线程池里面。下面我们就来看看fork方法里面具体的逻辑

~~~java
/**
 * 这个为ForkJoinTask里面的方法
*/
public final ForkJoinTask<V> fork() {
    Thread t;
    // 先判断一下当前线程是否是ForkJoinWorkerThread类。如果是这个类，说明是内部提交
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        // 拿到当前线程绑定WorkQueue，push这个任务到这个WorkQueue的array中
        ((ForkJoinWorkerThread)t).workQueue.push(this);
    else
        // 如果不是ForkJoinWorkerThread类型，那么进行外部提交
        ForkJoinPool.common.externalPush(this);
    return this;
}
~~~

#### 3.4.1 push(ForkJoinTask)

~~~java
/**
 * 这个为WorkQueue类的方法
 * push这个任务到WorkQueue中
*/
final void push(ForkJoinTask<?> task) {
    ForkJoinTask<?>[] a; ForkJoinPool p;
    int b = base, s = top, n;
    // 判断一下当前array是否已经被初始化了
    if ((a = array) != null) {    // ignore if queue removed
        // 拿到掩码
        int m = a.length - 1;     // fenced write for task visibility
        // 放置task
        U.putOrderedObject(a, ((m & s) << ASHIFT) + ABASE, task);
        // top+1
        U.putOrderedInt(this, QTOP, s + 1);
        // 判断一下，如果n <= 1，说明当前array里面的任务数 1 <= x <= 2
        if ((n = s - b) <= 1) {
            if ((p = pool) != null)
                // 所以需要signal一个work
                p.signalWork(p.workQueues, this);
        }
        // 如果 n过大
        else if (n >= m)
            // 对数组进行扩容
            growArray();
    }
}
~~~

可以看到内部提交，也就是把任务放到调用fork方法线程所绑定的WorkQueue中，后面的signalWork就与上面我们所讲的一致了。所以fork方法也挺简单的。

### 3.5 join()

fork完了之后再次回到我们之前的小例子

~~~java
CounterTask leftTask = new CounterTask(start, middle);
CounterTask rightTask = new CounterTask(middle+1, end);
leftTask.fork();
rightTask.fork();
int leftRes = leftTask.join();
int rightRes = rightTask.join();
~~~

可以看到fork完了之后，接着就是join了，并且有返回值。看到这里我们其实就可以猜测，task调用join，会等待自己的状态变为完成状态，然后获取返回值。那我们就看看join方法具体的代码

~~~java
public final V join() {
    int s;
    // 先调用doJoin()方法，返回值 & DONE_MASK，看看执行状态是不是NORMAL正常的
    if ((s = doJoin() & DONE_MASK) != NORMAL)
        // 不正常就报告异常
        reportException(s);
    // 正常就调用getRawResult()方法，返回原始结果
    return getRawResult();
}
~~~

<img src="ForkJoin%E6%A1%86%E6%9E%B6.assets/ForkJoinTask-join().png" style="zoom:80%;" />

#### 3.5.1 doJoin()

~~~java
private int doJoin() {
    int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
    // 这个三元运算符写得比较复杂。我们在下面解析。
    return (s = status) < 0 ? s :
    ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
        (w = (wt = (ForkJoinWorkerThread)t).workQueue).
        tryUnpush(this) && (s = doExec()) < 0 ? s :
    wt.pool.awaitJoin(w, this, 0L) :
    externalAwaitDone();
}
~~~

这里有必要先看一下ForkJoinTask的任务状态。在ForkJoinTask类中，任务状态是由status标志的，主要有以下几类

- **初始状态**：status == 0
- **完成状态**：status < 0。完成状态又有如下几种细分：
  - NORMAL（0xf0000000）：正常完成状态，< 0
  - CANCELLED（0xc0000000）：取消状态，< NORMAL
  - EXCEPTIONAL（0x80000000）：异常状态，< CANCELLED
- **通知状态**：status == SIGNAL（0x00010000） > 0

---

我们再来看一下上面的三元运算：

**status < 0 ?**

- **是**：任务已经是完成状态，所以直接返回status就行，不用再做处理

- **否**：任务还没有被处理过，当前线程就不等其他线程而是自己来处理。判断一下当前线程是否是ForkJoinWorkerThread类型

  - **是**：当前线程是ForkJoinPool的内部线程，可以处理任务

    先调用tryUnpush方法把这个任务从任务队列移除，然后调用doExec方法执行这任务。执行之后，返回任务状态。

    - **任务状态为完成态**：返回任务状态s
    - **任务状态为初始态**：tryUnpush移除任务失败，或者执任务失败，任务就还是初始态。调用池的awaitJoin方法。

  - **否**：当前线程是外部线程。例在main方法中仅有`new CounterTask(1,2).join()`，没有用到池，那么这个线程就是普通线程

    那么就调用Task的externalAwaitDone方法，意为外部提交等待完成方法。

tryUnpush方法很简单，就是把当前线程所绑定的这个WorkQueue中的这个task用CAS移除，但是需要注意的是，为什么会移除失败？因为**当当前任务不在top-1位的时候，就会移除失败。所以在官方文档中建议了我们对ForkJoinTask任务的调用顺序，一对 fork-join操作一般按照如下顺序调用：`a.fork(); b.fork(); b.join(); a.join();`。因为任务 b 是后面进入队列，也就是说它是在栈顶的（top-1 位），在它`fork()`之后直接调用`join()`就可以直接执行而不会调用`ForkJoinPool.awaitJoin`方法去等待。**）。下面看看doExec方法

##### 3.5.1.1 doExec()

~~~java
/**
 * ForkJoinTask类中的方法
 * 只要执行了这个方法，任务的状态就一定会变成完成态
*/
final int doExec() {
    int s; boolean completed;
    // 判断一下任务状态。< 0，直接返回
    if ((s = status) >= 0) {
        // >= 0就执行
        try {
            // 完成标记
            completed = exec();
        } catch (Throwable rex) {
            // 出现异常的话，设置任务状态为EXCEPTIONAL，并返回
            return setExceptionalCompletion(rex);
        }
        // 已完成，则设置任务状态为NORMAL，正常完成标记，最后返回	
        if (completed)
            s = setCompletion(NORMAL);
    }
    return s;
}
~~~

##### 3.5.1.2 awaitJoin(WorkQueue, ForkJoinTask, long)

~~~java
/**
 * ForkJoinPool中的方法
 * w：当前线程绑定的WorkQueue
 * task：tryUnpush方法没有移除掉的任务
 * deadline：超时等待的时间
*/
final int awaitJoin(WorkQueue w, ForkJoinTask<?> task, long deadline) {
    int s = 0;
    if (task != null && w != null) {
        // 因为有新任务join了，所以需要将currentJoin赋值给prevJoin
        ForkJoinTask<?> prevJoin = w.currentJoin;
        // 把currentJoin置为当前task
        U.putOrderedObject(w, QCURRENTJOIN, task);
        // 判断任务类型是不是CountedCompleter类型，是的话就强转一下，赋值给cc
        CountedCompleter<?> cc = (task instanceof CountedCompleter) ?
            (CountedCompleter<?>)task : null;
        // 起一个自旋。直到任务完成退出。
        for (;;) {
            // 如果任务状态已完成，返回
            if ((s = task.status) < 0)
                break;
            // 如果任务类型是CountedCompleter类型
            if (cc != null)
                // 则调用这个方法，帮助完成这个任务
                helpComplete(w, cc, 0);
            // 如果不是那个类型
 			// 尝试执行
            else if (w.base == w.top || w.tryRemoveAndExec(task))
                helpStealer(w, task); // 队列为空或执行失败，任务可能被偷，帮助偷取者执行该任务
            //已经完成|取消|异常，跳出循环
            if ((s = task.status) < 0)	
                break;
            // 计算任务等待时间
            long ms, ns;
            // 如果deadline == 0，那么不需要等待
            if (deadline == 0L)
                ms = 0L;
            // 如果deadline != 0，计算一下需要等待多久
            else if ((ns = deadline - System.nanoTime()) <= 0L)
                // 需要等待时间 <= 0，直接break，不用等了
                break;
            // 需要等待的话，把纳秒值转成毫秒
            else if ((ms = TimeUnit.NANOSECONDS.toMillis(ns)) <= 0L)
                ms = 1L;
            // 执行补偿操作（补偿操作什么意思：因为经过前面我们可以知道，任务没有执行完毕，所以这个地方需要挂起自己，等待被偷取任务的结果。自己被挂起了，那么算力损失了，所以需要补偿。例如：创建一个线程、唤醒一个线程等）
            // tryCompensate方法做线程等待的一些前置辅助工作，返回值为：线程是否能够阻塞
            if (tryCompensate(w)) {
                // 线程阻塞在task上面ms毫秒
                task.internalWait(ms);
                // 阻塞完，再把活跃线程数+1
                U.getAndAddLong(this, CTL, AC_UNIT);
            }
        }
        // 把currentJoin再换回前面的task
        U.putOrderedObject(w, QCURRENTJOIN, prevJoin);
    }
    // 返回任务状态
    return s;
}
~~~

**总结：** 如果当前 join 任务不在Worker等待队列的`top-1`位，或者任务执行失败，调用此方法来帮助执行或阻塞当前 join 的任务。函数执行流程如下：

- 由于每次调用`awaitJoin`都会优先执行当前join的任务，所以首先会更新`currentJoin`为当前join任务；
- 进入自旋：
  1. 首先检查任务是否已经完成（通过`task.status < 0`判断），如果给定任务执行完毕|取消|异常 则跳出循环返回执行状态`s`；
  2. 如果是 CountedCompleter 任务类型，调用`helpComplete`方法来完成join操作（后面笔者会开新篇来专门讲解CountedCompleter，本篇暂时不做详细解析）；
  3. 非 CountedCompleter 任务类型调用`WorkQueue.tryRemoveAndExec`尝试执行任务；
  4. 如果给定 WorkQueue 的等待队列为空或任务执行失败，说明任务可能被偷，调用`helpStealer`帮助偷取者执行任务（也就是说，偷取者帮我执行任务，我去帮偷取者执行它的任务）；
  5. 再次判断任务是否执行完毕（`task.status < 0`），如果任务执行失败，计算一个等待时间准备进行补偿操作；
  6. 调用`tryCompensate`方法为给定 WorkQueue 尝试执行补偿操作。在执行补偿期间，如果发现 资源争用|池处于unstable状态|当前Worker已终止，则调用`ForkJoinTask.internalWait()`方法等待指定的时间，任务唤醒之后继续自旋。

3.5.1.2.1 tryRemoveAndExec(ForkJoinTask)

~~~java
/**
 * 为WorkQueue类中的方法
*/
final boolean tryRemoveAndExec(ForkJoinTask<?> task) {
    ForkJoinTask<?>[] a; int m, s, b, n;
    if ((a = array) != null && (m = a.length - 1) >= 0 &&
        task != null) {
        while ((n = (s = top) - (b = base)) > 0) {
            // 自旋，从top位往base位处查找
            for (ForkJoinTask<?> t;;) {      // traverse from s to b
                long j = ((--s & m) << ASHIFT) + ABASE;  // 计算地址
                if ((t = (ForkJoinTask<?>)U.getObject(a, j)) == null)  // 拿到指定位置task（从top递减查找）
                    return s + 1 == top;     // shorter than expected
                // 如果探测到的任务等于传进来的任务
                else if (t == task) {
                    // 设一个标志
                    boolean removed = false;
                    // 检查一下当前任务是不是在栈顶
                    if (s + 1 == top) {      // pop
                        // 是栈顶
                        // 那么pop任务
                        if (U.compareAndSwapObject(a, j, task, null)) {
                            // top-1
                            U.putOrderedInt(this, QTOP, s);
                            // 移除标志置true
                            removed = true;
                        }
                    }
                    // 如果传进来的任务不是在栈顶
                    // 判断一下base有没有改变
                    else if (base == b)      // replace with proxy
                        // 没有改变的话，就把这个任务所在的位置放一个空任务，做占位符
                        removed = U.compareAndSwapObject(
                        a, j, task, new EmptyTask());
                    // 成功移除掉任务后
                    if (removed)
                        // 执行这个任务
                        task.doExec();
                    // 退出自旋
                    break;
                }
                // 如果探测的任务!=传进来的任务
                // 那么判断一下探测到的任务是否完成、且在栈顶
                else if (t.status < 0 && s + 1 == top) {
                    // 满足的话，顺手把这个任务移除掉
                    if (U.compareAndSwapObject(a, j, t, null))
                        U.putOrderedInt(this, QTOP, s);
                   	// 移除成功，跳出for循环
                    break;                  // was cancelled
                }
                //如果任务队列遍历完，则返回
                if (--n == 0)
                    return false;
            }
            // 任务已经执行
            if (task.status < 0)
                return false;
        }
    }
    return true;
}
~~~

tryRemoveAndExec如果找到这个任务会直接执行，然后用一个空任务放入原来的位置。如果没有找到这个任务说明任务被某个队列线程偷取了，会调用helpStealer方法去寻找这个小偷。在helpStealer中只会遍历奇数槽位的队列，因为也只有奇数槽位的队列才会有线程去偷取任务。如果小偷没有执行到自己队列的任务，会帮小偷执行任务。如果自己队列有任务没有执行完会退出方法，然后会进行一次补偿后阻塞线程等待任务完成唤醒。

**注意**：注意返回的参数：如果任务队列为空或者任务**未**执行完毕返回`true`；任务执行完毕返回`false`。因为未执行完毕，说明没有找到任务，可能是被偷了，所以需要返回到awaitJoin方法中，helpStealer。如果自己已经执行完毕了，返回false，说明不需要helpStealer

3.5.1.2.2 helpStealer(WorkQueue, ForkJoinTask)

~~~java
/**
 * ForkJoinPool中的方法
 * w：当前线程所绑定的WorkQueue
 * task：执行join的task
*/
private void helpStealer(WorkQueue w, ForkJoinTask<?> task) {
    WorkQueue[] ws = workQueues;
    // checkSum、oldSum参考scan方法里面的作用
    int oldSum = 0, checkSum, m;
    // 先做一下条件判断
    if (ws != null && (m = ws.length - 1) >= 0 && w != null &&
        task != null) {
        // 自旋
        do {                                       // restart point
            checkSum = 0;                          // for stability check
            ForkJoinTask<?> subtask;
            WorkQueue j = w, v;                    // v is subtask stealer
            // 这里是一个go-to语句
            descent: for (subtask = task; subtask.status >= 0; ) {
                //确保h为奇数，k每次增加2，h+k则是每次都为奇数
                for (int h = j.hint | 1, k = 0, i; ; k += 2) {
                   	// 扫描索引超出数组长度，找不到，跳出go-to
                    if (k > m)                     // can't find stealer
                        break descent;
                    // 取出h+k位置的WorkQueue
                    if ((v = ws[i = (h + k) & m]) != null) {
                        // 如果取出的这个WorkQueue的当前偷取的任务，就等于我们这个任务，那么小偷就是它
                        if (v.currentSteal == subtask) {
                            // 自己用hint记住这个小偷下标，跳出循环
                            j.hint = i;
                            break;
                        }
                        // 每探测一次，没找到的话，就更新一下chekcSum
                        checkSum += v.base;
                    }
                }
                // 到这，就开始帮偷取者执行任务了
                for (;;) {                         // help v or descend
                    ForkJoinTask<?>[] a; int b;
                    // 先用之前的checkSum加上偷取者的base。因为上面找到偷取者就break了，没加上
                    checkSum += (b = v.base);
                    // 拿到小偷当前join的task
                    ForkJoinTask<?> next = v.currentJoin;
                    // 如果这个任务已经完成                    或者
                    // 自己当前join的任务 != 这个任务了  或者
                    // 偷取者当前偷取的任务 != 这个任务了 
                    // 那么就跳出go-to重新找
                    if (subtask.status < 0 || j.currentJoin != subtask ||
                        v.currentSteal != subtask) // stale
                        break descent;
                    // 如果小偷没有任务了
                    if (b - v.top >= 0 || (a = v.array) == null) {
                        // 判断一下小偷有没有正在join的任务
                        if ((subtask = next) == null)
                            // 如果没有，说明小偷没有任务了，那么当前任务就又被别的偷了。需要跳出go-to重新查找新的小偷
                            break descent;
                        // 如果小偷有正在join的任务。currentJoin是在awaitJoin方法中设置的，currentJoin的任务理论上，没被偷的话，还存在于自己的数组中。
                        // 而这里，偷取者没有任务了，但是还有currentJoin，所以，偷取者的任务又被别人偷了。
                        // 那么这里，就需要再去找偷取者的偷取者
                        j = v;
                        break;
                    }
                    // 走到这，说明偷取者还有任务，或者偷取者的偷取者还有任务
                    // 计算base的地址
                    int i = (((a.length - 1) & b) << ASHIFT) + ABASE;
                    // 拿到base处的任务
                    ForkJoinTask<?> t = ((ForkJoinTask<?>)
                                         U.getObjectVolatile(a, i));
                    if (v.base == b) {
                        // base处的任务为null
                        if (t == null)             // stale
                            break descent;
                        // 置base处的任务为null
                        if (U.compareAndSwapObject(a, i, t, null)) {
                            // base + 1
                            v.base = b + 1;
                            // 获取调用者偷来的任务
                            ForkJoinTask<?> ps = w.currentSteal;
                            int top = w.top;
                            //首先更新自己workQueue的currentSteal为偷取者的base任务，然后执行该任务
                            //然后通过检查top来判断给定workQueue是否有自己的任务，如果有，
                            // 则依次弹出任务(LIFO)->更新currentSteal->执行该任务（注意这里是自己偷自己的任务执行）
                            do {
                                U.putOrderedObject(w, QCURRENTSTEAL, t);
                                t.doExec();        // clear local tasks too
                            } while (task.status >= 0 &&
                                     w.top != top &&    // 内部有自己的任务，依次弹出执行
                                     (t = w.pop()) != null);
                            // 还原给定workQueue的currentSteal
                            U.putOrderedObject(w, QCURRENTSTEAL, ps);
                            // 给定workQueue有自己的任务了，帮助结束，返回
                            if (w.base != w.top)
                                return;            // can't further help
                        }
                    }
                }
            }
        } while (task.status >= 0 && oldSum != (oldSum = checkSum));
    }
}
~~~

**总结：** 如果队列为空或任务执行失败，说明任务可能被偷，调用此方法来帮助偷取者执行任务。基本思想是：偷取者帮助我执行任务，我去帮助偷取者执行它的任务。
 函数执行流程如下：

1. 循环定位偷取者，由于Worker是在奇数索引位，所以每次会跳两个索引位。定位到偷取者之后，更新调用者 WorkQueue 的`hint`为偷取者的索引，方便下次定位；
2. 定位到偷取者后，开始帮助偷取者执行任务。从偷取者的`base`索引开始，每次偷取一个任务执行。在帮助偷取者执行任务后，如果调用者发现本身已经有任务（`w.top != top`），则依次弹出自己的任务(LIFO顺序)并执行（也就是说自己偷自己的任务执行）。

3.5.1.2.3 tryCompensate(WorkQueue)

~~~java
private boolean tryCompensate(WorkQueue w) {
    boolean canBlock;
    WorkQueue[] ws; long c; int m, pc, sp;
    // 判断池状态。异常的话，返回false
    if (w == null || w.qlock < 0 ||           // caller terminating
        (ws = workQueues) == null || (m = ws.length - 1) <= 0 ||
        (pc = config & SMASK) == 0)           // parallelism disabled
        canBlock = false;
    // 状态正常。如果有空闲线程，那么返回true，让当前线程挂起。并且唤醒空闲线程，以作补偿算力损失。
    else if ((sp = (int)(c = ctl)) != 0)      // release idle worker
        canBlock = tryRelease(c, ws[sp & m], 0L);
	// 没有空闲线程
    else {
        int ac = (int)(c >> AC_SHIFT) + pc;   // 活跃线程数  
        int tc = (short)(c >> TC_SHIFT) + pc; // 总线程数
        int nbusy = 0;                        // validate saturation
        for (int i = 0; i <= m; ++i) {        // two passes of odd indices
            WorkQueue v;
            if ((v = ws[((i << 1) | 1) & m]) != null) {  // 取奇数位索引
                if ((v.scanState & SCANNING) != 0)       // 找到线程空闲，则跳出
                    break;
                ++nbusy;    // 线程忙碌，则记录一下
            }
        }
        // 查找到的忙碌线程数 != 总线程数的两倍（这逻辑硬是没懂）
        // 或者ctl改变，那么当前线程不用阻塞
        if (nbusy != (tc << 1) || ctl != c)
            canBlock = false;                 // unstable or stale
        // 如果
        //    总线线程数 >= 并行度 && 活跃线程数 > 1 && 自己任务没有任务了
        // 那么 不需要补偿算力（线程过多，无法添加）。自己放心挂起等待自己被偷的join任务的结果就好
        else if (tc >= pc && ac > 1 && w.isEmpty()) {
            long nc = ((AC_MASK & (c - AC_UNIT)) |
                       (~AC_MASK & c));       // uncompensated
            canBlock = U.compareAndSwapLong(this, CTL, c, nc);
        }
        // 如果总线程数过大，抛出异常
        else if (tc >= MAX_CAP ||
                 (this == common && tc >= pc + commonMaxSpares))
            throw new RejectedExecutionException(
            "Thread limit exceeded replacing blocked worker");
        // 到这说明 自己休息了，算力会损失。所以需要补偿。有如下几种情况
        // 	   - 总线程数 > 并行度，但是没有活跃线程数、或者自己还有任务：需要创建新的线程，来执行自己的任务（这个地方就会导致线程总数超过并行度）
        //     - 总线程数正常，有活跃线程数，自己没有任务：也需要创建（一赠一减嘛）
        // 创建一个新的线程来补偿
        else {                                // similar to tryAddWorker
            boolean add = false; int rs;      // CAS within lock
            long nc = ((AC_MASK & c) |
                       (TC_MASK & (c + TC_UNIT)));
            if (((rs = lockRunState()) & STOP) == 0)
                add = U.compareAndSwapLong(this, CTL, c, nc);
            unlockRunState(rs, rs & ~RSLOCK);
            canBlock = add && createWorker(); // throws on exception
        }
    }
    return canBlock;
}
~~~

**总结：** 具体的执行看源码及注释，这里我们简单总结一下需要和不需要补偿的几种情况：

- 需要补偿：
  - 调用者队列不为空，并且有空闲工作线程，这种情况会唤醒空闲线程（调用`tryRelease`方法）
  - 池尚未停止，活跃线程数不足，这时会新建一个工作线程（调用`createWorker`方法）
- 不需要补偿：
  - 调用者已终止或池处于不稳定状态
  - 总线程数大于并行度 && 活动线程数大于1 && 调用者任务队列为空

到这，我们的join方法就说完了。

### 3.6 总结 

ForkJoin框架内部设计相当复杂，只需要理解大致思路即可。

**在网上发现一篇能帮助理解ForkJoin框架原理的博文，一定要看：[指北 | 谈谈ForkJoin框架的设计与实现](D:\TyporaDocument\Java\多线程\谈谈ForkJoin框架的设计与实现.md)**

参考：

[JUC源码分析-线程池篇（五）：ForkJoinPool - 2](https://www.jianshu.com/p/6a14d0b54b8d)

[Executor（五）：ForkJoinPool详解 jdk1.8](https://blog.csdn.net/LCBUSHIHAHA/article/details/104449454)