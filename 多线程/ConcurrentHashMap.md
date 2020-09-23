# ConcurrentHashMap

对比HashMap学习，有很多相似之处

## 几个重要的成员属性、变量

~~~java
// 维护了一个Unsafe对象U，里面的CAS操作都由该对象完成
private static final sun.misc.Unsafe U

// 这种全大写的long型常量为对应小写变量在内存中的地址
private static final long SIZECTL;

// 与HashMap一样，用来存储Node节点的数组
transient volatile Node<K,V>[] table;

// 这是一个连接表，用于哈希表扩容，扩容完成后会被重置为null
private transient volatile Node<K,V>[] nextTable;

// 该属性保存着整个哈希表中存储的所有的结点的个数总和，有点类似于 HashMap 的 size 属性
private transient volatile long baseCount;

// 该属性是 扩容时标签 所占的位数，在这个 ConcurrentHashMap 中占 16 位
private static int RESIZE_STAMP_BITS = 16;

/* 因为 int 类型为 32 位，该值等于 int型位数 减去上面的 扩容标签位数 ，结合这个常量的名字可知
   首先计算出的 扩容标签 占着 int 型的低位 RESIZE_STAMP_BITS 个位数，然后将 这个标签往高位
   位移 RESIZE_STAMP_SHIFT 个位数，使这个标签正好在最高位。（后文详细解释）
*/ 
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

/* sizeCtl一个很重要的成员属性，该属性有以下几种取值：
  		- 0：默认值，表示table数组还没有初始化
  		- -1：代表哈希表正在进行初始化
        - 大于0：没初始化时，代表table数组的长度，初始化后相当于 HashMap 中的 threshold，表示阈值
        - 小于-1：代表有多个线程正在进行扩容(高16位为扩容标记，低16位的int值为扩容线程数量+1)
*/
private transient volatile int sizeCtl;
~~~

首先看一下该类的构造方法，构造方法很多，选取一个

## ConcurrentHashMap(int initialCapacity)

~~~java
public ConcurrentHashMap(int initialCapacity) {
    // 检查调用传值是否合法
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    // 如果 initialCapcity 大于等于 最大容量的一半，那么就直接取最大容量。
    // 否则调用tableSizeFor方法（与HashMap的一样，往上翻）
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}

~~~

在这个构造方法里 tableSizeFor 传递的参数与HashMap构造方法里传递的不一样。为什么会这样呢？

~~~java
tableSizeFor(initialCapacity); // HashMap中的
tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1)); // ConcurrentHashMap中的
~~~

个人猜测原因如下：

假设用户有 12 个键值对需要存放，调用构造方法，传参 12。HashMap通过 tableSizeFor 计算出来的实际容量就是16，又因为默认的 装载因子 为 0.75， 所以扩容阈值计算得12，当用户的12个键值对存储进来后，立马就需要扩容，而扩容很浪费性能。CocurrentHashMap改进了这一点。参数为12时，tableSizeFor的参数为19，计算出来的容量为32，扩容阈值为 24，用户的12个键值对存进来，还有富余空间。这也算空间换时间原则的一种体现吧

构造方法调用完了之后，会调用put方法，添加键值对

## put方法

~~~java
/* put方法只调用了putVal方法 */
public V put(K key, V value) {
    return putVal(key, value, false);
}
~~~

介绍putVal方法前，先介绍一下spread方法，做一下铺垫

## spread方法

~~~java
// 低位16位全1，高16位全0，与 上hash，保证了hash的低16位不变，高16位全为0，从而保证了hash为正数
static final int HASH_BITS = 0x7fffffff;

/* 与HashMap中的hash方法基本一样 */
static final int spread(int h) {
    // 多了一个 与 上HASH_BITS 操作，是为了保证这个计算出来额hash值是正数。
    // 因为有特殊结点，它们hash为负数，区分正负方便判断
    // 例：static final int MOVED     = -1; // ForwardingNode结点的hash
    //	  static final int TREEBIN   = -2; // 红黑树头结点，hash值统一为-2，该头不存储数据
    //    static final int RESERVED  = -3; // hash for transient reservations
    return (h ^ (h >>> 16)) & HASH_BITS;
}
~~~

## putVal方法

~~~java
/** 实际执行put操作的方法 */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 检查 key、value是否合法，与HashMap不一样的是，ConcurrentHashMap不允许null键、null值
    if (key == null || value == null) throw new NullPointerException();
    // spread方法与HashMap中的hash方法类似，得到一个hash值（见上）
    int hash = spread(key.hashCode());
   	// 这个变量保存着正在操作的链条的长度
    int binCount = 0;
    // 起一个循环，只能从内部跳出
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
       	// 1. 判断table(tab 即为 table)是否还没有初始化，是的话就先初始化table
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 2. table已经初始化了，用put方法传来的key的hash值计算该键值对在table中的索引
        // 如果该索引位置为空，那么就new一个结点，用CAS原子操作把该节点放进去，并跳出循环
        // （binCount = 0， f 为该链的头结点，i 为通过hash算出来的 f 在数组中的索引）
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                  
        }
        // 3.  程序走到这，说明table已经初始化了，该hash对应的索引位置上也不为空
        // 就通过这个结点的hash判断一下这个结点是不是MOVED类型的结点
        // 如果是的话，说明当前表正在进行扩容，那么当前线程就放下手中的工作，前去帮忙扩容
        // (判断的过程中顺便把头结点f 的 hash给赋值为了 fh)
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        // 4.  程序走到这，说明前面三个if都没满足：既不是没有初始化、put进来的键值对应该存放的
        // 位置也被占了、table结点数组也没有正在扩容。那么就应该进行正常的put操作：插入或更新
        else {
            // 定义oldVal，做返回值用
            V oldVal = null;
            // 锁住这条链的头结点，保证临界区只能有一个线程操作 
            synchronized (f) {
                // 首先还是判断一下数组table的第i个位置上的结点等不等于f （为了保险）
                if (tabAt(tab, i) == f) {
                    // 判断头结点的hash是不是 >= 0，如果是，说明该结点是普通的链表结点
                    if (fh >= 0) {
                        binCount = 1;
                        // 遍历该条链表，遍历的过程中binCount记录着当前长度
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 如果找到同一个key，更新结点的值
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // 判断是否到达链尾，是的话，插入一个新结点
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 如果该结点不是普通链表结点，而是树头结点
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        // binCount就不代表链表长度，设置成2，标志该位置为红黑树了
                        binCount = 2;
                        // 调用树的插入更新方法，如果返回值为空，则说明插入了一个新的树结点
                        // 返回值不为空，说明找到了，等于传进来key的结点，即为p，把p的值更新即可
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                              value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // binCount == 0,在数组上插入了新的链表结点（binCount == 0，程序早就跳出循环了）
            // binCount > 0,在链上或者树上插入了或更新了结点，需要判断是否需要树化
            if (binCount != 0) {
                // 如果在树上插入或更新，binCount == 2，自然不会超过树化阈值
                // 如果在链上插入，那么binCount就记录着链表的长度，可能超过树化阈值
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                // 如果是添加结点，那么oldVal == null，不会在这返回，还需要去下面的
                // 调用addCount方法，修改baseCount并判断是否扩容
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    //5. 程序到这返回，说明插入了新结点，需要更新baseCount，并判断是否需要扩容
    addCount(1L, binCount);
    return null;
}
~~~

可以看到putVal方法中对很多情况作了判断，比如初始化、扩容等。我们先来看一下1. 中初始化方法

## initTable方法

~~~java
/** 对table结点数组进行初始化 */
private final Node<K,V>[] initTable() {
    // 定义临时table结点数组tab，临时sizeCtl变量sc
    Node<K,V>[] tab; int sc;
    // 起一个循环，只能内部跳出，主要是用于线程 自旋 使用
    // 如果table还没有被初始化，则进入循环
    while ((tab = table) == null || tab.length == 0) {
        // 判断sizeCtl是否小于0， sizeCtl = -1时，表示正有线程在初始化table表
        if ((sc = sizeCtl) < 0)
            // 如果有线程正在初始化，当前线程需要让出CPU执行权，通过自旋不断判断table是否被初始化好
            Thread.yield();
        // 如果没有线程正在执行初始化操作，那么当前线程尝试通过CAS将sizeCtl值设置为-1
        // 表明当前正有线程在执行初始化操作。如果设置失败，说明有线程抢先一步了，那么当前线程继续自旋
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                // 当前线程成功拿到初始化权，先判断一下表没被初始化（保险起见）
                if ((tab = table) == null || tab.length == 0) {
                    // sc还保留着sizeCtl之前的数据
                    // sc=0,说明调用的是无参构造方法，直接赋默认值给n，
                    // sc>0，那么sc的值就是table应该初始化的容量
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    // new一个n长的Node数组赋值给table
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // sc = n - 0.25*n = 0.75*n , 为扩容阈值
                    sc = n - (n >>> 2);
                }
            } finally {
                // 最后赋值给sizeCtl这个常量
                sizeCtl = sc;
            }
            // 初始化完成，退出循环
            break;
        }
    }
    // 返回初始化后的数组
    return tab;
}
~~~

我们再来看一下putVal方法中没有碰撞，插入表结点后需要执行的addCount方法，addCount方法中用到一个静态内部类CounterCell，我们先来看一下。

## CounterCell静态内部类

~~~java
/**
 * 可以看到该类中，只有一个volatile修饰的变量，这个类就是一个计数单元
 * addCount方法是将 baseCount的值 和 计数单元数组中的值 累计起来，返回的就是整个map的size
 * 为什么不直接用size属性表示map中键值对个数呢，还是并发问题。
 * baseCount只用于无争用的情况下，而当出现争用，baseCount临界资源无法访问
 * 那么线程就会把添加的值放在计数单元中，所以计算size的时候需要加上计数单元数组中的每个值
*/
@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
~~~

## addCount

~~~java
/**
 * 该方法是用来修改baseCount，类似HashMap中size的一个属性。并判断是否需要扩容。
 * 参数 x 就是baseCount需要+ 的数据， check是一个标志位，putVal方法传进来的是binCount
*/
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 判断counterCells计数单元成员数组是否为null 或者 修改baseCount是否成功。
    // 如果该成员数组为null，说明还没有线程修改过这个值，就应该尝试去修改baseCount的值
    // 如果修改baseCount失败，那么说明有争用，当前线程放弃修改baseCount，转而把值放到计数单元中
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        // 如果counterCells不为null，或者修改baseCount失败
        CounterCell a; long v; int m;
        // 把 无竞争 标识设为true
        boolean uncontended = true;
        // 判断countCells数组是否为空（为空就是还没有初始化），或者这个数组长度是不是<0
        // ThreadLocalRandom.getProbe()，是给当前线程随机生成一个值，可以简单的理解为hash
        // 这个值 &m 可以得到一个数组下标（可见计数单元数组长度也为2的n次，每个线程有自己特定的格子）
        // 如果这个位置为空的话，那么尝试用CAS操作，把自己这个计数单元+参数x
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            // 如果上面的条件有一个不满足，或者当前有争用，那么就需要调用fullAddCount方法来计数
            fullAddCount(x, uncontended);
            return;
        }
        // 程序能执行到这，说明计数单元数组不为空，且修改baseCount失败了
        // 如果check <= 1，说明是在数组上插入了结点，直接返回，因为baseCount的值没有变化，
        // 不需要判断是否需要扩容
        if (check <= 1)
            return;
        // 统计总size
        s = sumCount();
    }
    // 如果没有争用，并且修改baseCount成功，那么需要判断是否需要扩容
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 没有线程在扩容时，sizeCtl表示扩容阈值
        // 如果size >= 扩容阈值，并且表不空，表长也没超过最大值，那么进入扩容循环
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            // 拿到扩容标签
            int rs = resizeStamp(n);
            // sc < 0，表明有线程在扩容
            if (sc < 0) {
                // 判断sc高16位与扩容标签是否相等、或者其他的条件，来决定要不要前去帮助扩容
                // 与helpTransfer方法中一样（下面）
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 走到这说明，有线程在扩容，且当前线程需要帮助扩容，那么将sizeCtl + 1，
                // 表明扩容线程增加一个。
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 走到这，说明没有线程扩容，而table需要扩容，那么让当前线程前去扩容
            // 把sizeCtl修改为 扩容标识 左移16位 再+2，所以说sizeCtl高16位与扩容标识相等
            // 后面来帮助扩容的线程需要拿到当前数组的扩容标识，并判断与sizeCtl高16位是否想等
            // 不相等说明扩容已经结束。
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
~~~

我们看到addCount方法中，如果当前table需要扩容，要通过resizeStamp方法，通过传递当前表的长度，拿到一个扩容标识，下面我们就来看一下这个resizeStamp方法是怎么拿到扩容标识的

## resizeStamp方法

~~~java
/**
 * 通过当前table长度，拿到一个扩容标识，参数n为当前table长度。
 * 1 << (RESIZE_STAMP_BITS - 1)的意思是将RESIZE_STAMP_BITS位置上置为1，其他全为0
 * 直接看下面的例子解释这个方法：(假设当前 n = 32， 那么Integer.numberOfLeadingZeros(n)就 = 27)
 *   1    =   0000 0000 0000 0000 0000 0000 0000 0001
 * 1<<15  =   0000 0000 0000 0000 1000 0000 0000 0000‬   相
 *   27   =   0000 0000 0000 0000 0000 0000 0001 1011   与
 * ----------------------------------------------------------
 *   rs   =   0000 0000 0000 0000 1000 0000 0001 1011
 * 那么在扩容时 sc = (rs << RESIZE_STAMP_SHIFT) + 2
 *   sc   =   1000 0000 0001 1011 0000 0000 0000 0010
 * 那么扩容时，sizeCtl高16位就等于resizeStamp，低位为2
*/
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
~~~

再来看看 3. 中了helpTransfer方法。介绍helpTransfer方法前先来点铺垫，看看hash为MOVED到底是什么

## ForwardingNode类

~~~java
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        //注意这里
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }
    //省略其 find 方法
}
~~~

ForwardingNode 继承自 Node 结点，并且它唯一的构造函数将构建一个键，值，next 都为 null 的结点，反正它就是个标识，无需那些属性。但是 hash 值却为 MOVED。

这个节点内部保存了一 nextTable 引用，它指向一张 hash 表。在扩容操作中，我们需要对原表中每一个桶中的结点进行分离和转移到新的这个nextTable中（与HashMap类似，不过HashMap是在原表上直接扩容，ConcurrentHashMap是新建了一张表），如果某个桶结点中所有节点都已经迁移完成了（已经被转移到新表 nextTable 中了），那么会在原 table 表的该位置挂上一个 ForwardingNode 结点，说明此桶已经完成迁移。

所以，我们在 putVal 方法中遍历整个 hash 表的桶结点，如果遇到 hash 值等于 MOVED，说明已经有线程正在扩容 rehash 操作，整体上还未完成，不过我们要插入的桶的位置已经完成了所有节点的迁移。

那么既然有线程正在进行扩容操作，所以牛逼的大神就觉得既然有线程正在扩容，那当前线程也不能闲着，就让当前线程去帮助扩容（给Doug Lea这牛逼的操作跪了），而helpTransfer方法的作用就是在帮助扩容前做一些判断和初始化。

## helpTransfer方法

~~~java
/** 
 * 帮助扩容方法
 * tab为当前table，f为根据传进来key的hash找到的链头
*/
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 首先判断一下当前表是否为空，并且f要为ForwardingNode结点，并且f这个结点维护的nextTable不为空
    // 只有满足了这三点，才表明有线程正在扩容（判断的过程也是赋值的过程）
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        // 首先通过原数组（或者说当前数组）的长度拿到扩容标记。resizeStamp方法上面详述。
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            // 如果sizeCtl无符号右移shift为不等于扩容标记 或者 其他不满足的条件，就不用帮助扩容
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            // 走到这，说明需要帮助扩容，那么当前线程通过CAS把sizeCtl的值+1（又多了一个线程扩容）
            // 修改成功后就去帮助扩容
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                // 扩容方法
                transfer(tab, nextTab);
                break;
            }
        }
        // 帮助扩容完成之后，返回扩容之后的表
        return nextTab;
    }
    // 没有帮助扩容，那么就直接返回当前表
    return table;
}
~~~

addCount和helpTransfer方法都调用了tansfer这个实际扩容的方法，我们来看一下

## tansfer方法

~~~java
/**
 * 真正的扩容方法
 * tab ：原表
 * nextTab ：新表
 * nextTab为空的话，说明当前线程是第一个进来扩容的
*/
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    // n为原表长度， stride为扩容线程所需处理桶的个数
    int n = tab.length, stride;
    // 这个地方 n >>> 3不太懂，如果NCPU（虚拟机可以用的cpu核心数）为 1 的话，stride就直接
    // 为原表的长度，再判断这个值是否大于设置的最小值16，不是就赋值最小值
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    // 第一个进来扩容的线程需要对nextTable进行初始化工作
    if (nextTab == null) {            
        try {
            @SuppressWarnings("unchecked")
            // 长度为原表的两倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        // tansferIndex成员变量，意思是：转移过程中，新来的线程转移区间的最大下标加1
        transferIndex = n;
    }
    // 新表长度
    int nextn = nextTab.length;
    // 该结点索引新表，原表每一个桶转移完毕，就在该桶头结点上插入这个结点
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // 继续前进标识符
    boolean advance = true;
    // 结束标识符
    boolean finishing = false; // to ensure sweep before committing nextTab
    // 起一个死循环，开始进行扩容转移操作
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 每当一个线程第一次进来的时候advance总是true，所以可以执行这个循环
        while (advance) {
            // nextIndex，为当前线程处理区间的最大下标，nextBound为当前线程处理区间的最小下标
            int nextIndex, nextBound;
            // 第一次判断这个，总是不成立的
            if (--i >= bound || finishing)
                advance = false;
            // 第一次判断，这个也不成立，transferIndex为原表长度 (赋值了nextIndex)
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 所以线程进入循环后，第一次是执行的这个判断
            // CAS修改transferIndex的值
            // 梳理一下这里第一个扩容线程执行的逻辑：
            //    stride：为一个线程需要处理的桶数
            //    nextIndex：被赋值了transferIndex，transferIndex又是原表的长度
            // 所以这里是判断原表长度是否大于需要处理的桶的个数，如果大于，用原表长度减去需处理区间
            // 就是最小下标了，如果不大于，那么最小下标就直接为0了。再把transferIndex修改为最小
            // 下标的值
            // 第二个线程进来后，transferIndex就是前面线程转移区间的最小下标
            // 所以这个判断就是每个新进来的线程领取自己的任务区间（下标由高到低），或者
            // 原有扩容线程，执行完任务后继续领取新任务
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                // CAS成功后，把成员变量赋值给局部变量
                bound = nextBound;
                // i为处理区间的最大下标，也就是说转移操作是从最大下标开始往低位处理的
                i = nextIndex - 1;
                // 把继续前进标识符设置为false，那么就会跳出while循环
                advance = false;
            }
        }
        // 跳出while循环后，继续往下走
        // 判断当前线程的下标指针是否越界（<0, >=原表长度，+原表长度>=新表长度）
        if (i < 0 || i >= n || i + n >= nextn) {
            // 如果越界，说明当前线程的扩容任务已经完成
            int sc;
            // 判断整个扩容操作是否全部完成
            if (finishing) {
                // 如果都完成，释放空间
                nextTable = null;
                // 把原表索引指向新表
                table = nextTab;
                // 扩容完毕，恢复sizeCtl为新的扩容阈值 = 0.75*新表长度 = 0.75*n*2
                //                                = n*2 - n/2 = 1.5*n
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 走到这说明，当前线程的扩容任务完成，但是整体扩容任务没有完成
            // sizeCtl - 1，当前线程准备退出
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 退出前，判断一下自己是不是最后一个扩容线程
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                // 如果自己最后一个扩容线程，那么说明扩容任务已经结束，置标识符为true
                finishing = advance = true;
              	// 令i = n，重新再检查一遍整个表后退出
                i = n; // recheck before commit
            }
        }
        // 走到这，说明当前线程任务还没完成，i没有越界
        // 取原表i下标处的结点赋值给f
        else if ((f = tabAt(tab, i)) == null)
            // 如果该结点为空，那么这个桶也相当于转移完成，用CAS把ForwardingNode结点放入
            advance = casTabAt(tab, i, null, fwd);
        // 如果该结点不为空，且hash等于ForwardingNode的hash，说明该桶已经完成转移
        else if ((fh = f.hash) == MOVED)
            // 那么advance设置为true，继续下一个桶的转移
            advance = true; // already processed
        // 如果上面两种情况都不是，说明这个桶需要进行转移操作
        else {
            // 锁住该桶的头结点
            synchronized (f) {
                // 冗余判断一下
                if (tabAt(tab, i) == f) {
                    // lowNode, highNode
                    Node<K,V> ln, hn;
                    // 判断结点类型 fh >= 0,说明该结点是链表结点，需要转移
                    if (fh >= 0) {
                        // 可以参考HashMap rehash。
                        // runbit == 0：该结点在新表的原索引处
                        // runbit != 0：该结点在新表的（原表长度+原索引）索引处
                        int runBit = fh & n;
                        // lastRun记录
                        Node<K,V> lastRun = f;
                        // 循环判断一下，这一桶后面到桶尾有没有p.hash & n相同的结点
                        // 有的话，就用lastRun记录相同结点的第一个结点，用runBit记录p.hash&n
                        // 因为p.hash & n就两个结果（0或!0），所以大概率有相同结点连在一起
                        // 用lastRun记录下来后，后面就不用判断，直接根据runBit的值，挂到相应
                        // 位置，以节省时间
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        // 如果runBit == 0，那么说明相同的这些元素，是在新表的原索引（低索引）
                        if (runBit == 0) {
                            // 用ln记录
                            ln = lastRun;
                            // hn置null
                            hn = null;
                        }
                        // 反之则相反
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        // 开始分桶
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                // 等于0，就往ln指向的链上挂，头插法
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                // 反之亦然
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 最后把ln挂到新表原索引处
                        setTabAt(nextTab, i, ln);
                        // hn挂到新表（原表长度+原索引）索引处
                        setTabAt(nextTab, i + n, hn);
                        // 转移完毕后，把原表该索引处放置一个转移结点
                        setTabAt(tab, i, fwd);
                        // 继续执行
                        advance = true;
                    }
                    // 走到这说明，结点不为链表（桶）结点，判断一下是否为红黑树结点
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        // 循环处理该桶的每一个树节点
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            // 以当前结点的数据new一个新结点出来
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            // 分类判断该结点是属于新表的高位还是低位
                            if ((h & n) == 0) {
                                // 如果属于低位，该结点prev指针指向低位链的尾结点
                                // 如果低位链尾结点为null，说明当前结点是该链的第一个结点
                                if ((p.prev = loTail) == null)
                                    // 那么低位链的头指针指向当前结点
                                    lo = p;
                                // 如果低位链尾结点不为空，且在if判断中，当前结点的前指针已
                                // 指向尾结点
                                else
                                    // 尾结点后指针指向p
                                    loTail.next = p;
                               	// 最后当前结点变成尾结点
                                loTail = p;
                                // 低位链个数+1
                                ++lc;
                            }
                            // 如果为高位，同理
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        // 上面的代码只是维护了树结点的链式结构，树化在下面。
                        
                        // 判断低位链结点个数是否超过树化阈值，没超过，就把结点转换成链表
                        // 超过阈值，再判断一下高位链的结点个数是否为0
                        // 如果高位链结点个数不为0，说明该桶树结点分为了两条链
                        // 那么new一个TreeBin结点赋值给ln（new操作同时也将lo为首的树链
                        // 树化成了红黑树）
                        // 如果高位链结点个数为0，说明该桶树结点全在低位链上，并没有分裂
                        // 所以直接将原桶头赋值给ln，上面新建的树结点就全部没用了
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                        (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        // 与上同理
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                        (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        // 最后把ln挂到新表原索引处
                        setTabAt(nextTab, i, ln);
                        // hn挂到新表（原表长度+原索引）索引处
                        setTabAt(nextTab, i + n, hn);
                        // 转移完毕后，把原表该索引处放置一个转移结点
                        setTabAt(tab, i, fwd);
                        // 继续执行
                        advance = true;
                    }
                }
            }
        }
    }
}

~~~

整理一下transfer方法，transfer方法总体上就做了两件事：

1. 领取任务（拿到自己的扩容区间的上下界索引，索引由高到低领取）
2. 完成区间扩容任务（分为链表结点、树结点两种），执行完当前任务后，判断总体扩容任务是否结束，否则领取新任务

---

