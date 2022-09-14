# HashMap

HashMap使用链表法避免哈希冲突（相同hash值）,在table数组长度大于等于64的时候，当链表长度大于TREEIFY_THRESHOLD（默认为8）时，将链表转换为红黑树，当然小于UNTREEIFY_THRESHOLD（默认为6）时，又会转回链表以达到性能均衡。

<img src="HashMap.assets/HashMap%E7%BB%93%E6%9E%84%E5%9B%BE.png" style="zoom:70%;" />

## 1. tableSizeFor(int cap)方法

```java
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);//此处调用了下面的方法。threshold(扩容阈值)应当等于 capacity * loadFactor(装载因子)，这儿为什么不是呢。下面会有解释。
}

static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;// n = n|( n >>> 1 )
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

调用该方法所传递的参数cap保证了为正值，int类型为32bit，通过先 无符号位移（高位补0） 而 或 操作可以保证为1的最高位往低位都为1，再往高位都为0，见下图。最后返回值+1，即可得到大于等于给定值最接近的2的幂。n = cap-1的原因：假如 cap = 8, 返回“大于等于给定值最接近的2的幂“理应当返回8，但是经过下面的操作之后返回了16，所以要n = cap -1 = 7，使其返回8。

<img src="HashMap.assets/tableSizeFor.png" style="zoom:75%;" />



## 2. HashMap(Map<? extends K, ? extends V> m)

```java
 //实际存储key，value的数组，只不过key，value被封装成Node了
transient Node<K,V>[] table;

//初始容量能充足的容下指定的Map,装载因子为0.75
public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }

final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        if (table == null) { // pre-size
            //table还没有被初始化，用给的map里元素的个数除以装载因子加1计算新的capcity
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
           	//如果需要的容量大于扩容阈值，就重新计算扩容阈值
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        
        else if (s > threshold)
            resize();
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            
      		//put(k,v)也是调用的该方法      
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

## 3.hash(key)

putVal()方法用到了hash(key)，先看一下该方法。

```java
/**
     * key的hash值的计算是通过hashCode()的高16位 异或 低16位实现的:(h = k.hashCode()) ^ (h >>> 16)
     * 主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候,也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销
     原 来 的 hashCode: 1111 1111 1111 1111 0100 1100 0000 1010
     移 位 后 hashCode: 0000 0000 0000 0000 1111 1111 1111 1111
     进行 异或 运算结果 : 1111 1111 1111 1111 1011 0011 1111 0101
  
*/
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

## 4.resize()方法

```java
final Node<K,V>[] resize() {
      	// 保存当前table
        Node<K,V>[] oldTab = table;
    	// 保存当前table的容量
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
    	// 保存当前阈值
        int oldThr = threshold;
    	// 初始化新的table容量和阈值 
        int newCap, newThr = 0;
    /* 1. resize()函数在size > threshold时被调用。oldCap大于 0 代表原来的 table表非空, 		oldCap 为原表的大小,oldThr(threshold) 为 oldCap × load_factor
	*/
        if (oldCap > 0) {
            // 若旧table容量已超过最大容量,更新阈值为Integer.MAX_VALUE(最大整形值),
            //这样以后就不会自动扩容了。
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 容量翻倍，使用左移，效率更高
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
    /*2. resize（）函数在table为空被调用。oldCap小于等于0且oldThr大于0，代表用户创建了一个HashMap，但是使用的构造函数为HashMap(int initialCapacity, float loadFactor) 或 HashMap(int initialCapacity)或 HashMap(Map<? extends K, ? extends V> m)，导致 oldTab为 null，oldCap 为0，oldThr为用户指定的 HashMap的初始容量。
    　　*/
        else if (oldThr > 0) // initial capacity was placed in threshold
            //当table没初始化时，threshold持有初始容量。还记得threshold = tableSizeFor(t)么;
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
   	/*3. resize（）函数在table为空被调用。oldCap 小于等于 0 且 oldThr 等于0，用户调用 HashMap()构造函数创建的　HashMap，所有值均采用默认值，oldTab（Table）表为空，oldCap为0，oldThr等于0，
        */
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
    //在2.中使用的初始化的table还没有给新的threshold赋值，所以满足newThr == 0，
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
     // 初始化table
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            // 把 oldTab 中的节点　reHash 到　newTab 中去
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    // 若节点是单个节点，直接在 newTab　中进行重定位
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    // 若节点是　TreeNode 节点，要进行 红黑树的 rehash　操作
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                     // 若是链表，进行链表的 rehash　操作
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;//低索引处的头尾节点
                        Node<K,V> hiHead = null, hiTail = null;//高索引处的头尾节点
                        Node<K,V> next;
                        // 将同一桶中的元素根据(e.hash & oldCap)是否为0进行分割（代码后有图解，
                        //可以回过头再来看），分成两个不同的链表，完成rehash
                        do {
                            next = e.next;
                       // 根据算法　e.hash & oldCap 判断节点位置rehash　后是否发生改变
                       //假如oldCap = 1 0000（16），hash1 = 0 0101，hash2 = 1 0101 
                       //e.hash1 & oldCap = 0 000 == 0,在新数组中还是原索引下标
                       //e.hash2 & oldCap = 1 000 != 0,在新数组中索引下表 = 原下标 + oldCap
                            //最高位==0，这是索引不变的链表。
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            //最高位==1 （这是索引发生改变的链表）
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;//尾部置空
                            newTab[j] = loHead;//将头结点放到新数组的原下标处
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            // rehash 后节点新的位置一定为原来基础上加上 oldCap,具体解释看下图
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

引自美团点评技术博客。我们使用的是2次幂的扩展(指长度扩为原来2倍)，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。看下图可以明白这句话的意思，n为table的长度，图（a）表示扩容前的key1和key2两种key确定索引位置的示例，图（b）表示扩容后key1和key2两种key确定索引位置的示例，其中hash1是key1对应的哈希与高位运算结果。

![](HashMap.assets/Hash%E5%AE%9A%E4%BD%8D%E5%9B%BE.png)

元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

<img src="HashMap.assets/resize.png" style="zoom:50%;" />

因此，我们在扩充HashMap的时候，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，可以看看下图为16扩充为32的resize示意图 ：

<img src="HashMap.assets/HashMap%E6%89%A9%E5%AE%B9.png" style="zoom:50%;" />

什么时候扩容：

通过HashMap源码可以看到是在put操作时，即向容器中添加元素时，判断当前容器中元素的个数是否达到阈值（当前数组长度乘以加载因子的值）的时候，就要自动扩容了。

扩容(resize)：

其实就是重新计算容量；而这个扩容是计算出所需容器的大小之后重新定义一个新的容器，将原来容器中的元素放入其中。

## 5.putVal()

<img src="HashMap.assets/HashMap%E6%8F%92%E5%85%A5%E6%B5%81%E7%A8%8B%E5%9B%BE.png" style="zoom:50%;" />

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //如果table为空或者长度为0，则resize()
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //确定插入table的位置，算法是(n - 1) & hash，在n为2的幂时，相当于对n的取模(取余)操作。
    //找到key值对应的槽并且是第一个，直接加入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    //在table的i位置发生碰撞，有两种情况，
    //1、key值是一样的，替换value值，
    //2、key值不一样的有两种处理方式：2.1、存储在i位置的链表；2.2、存储在红黑树中
    else {
        Node<K,V> e; K k;
        //1.p为该hash所对应索引下第一个node，它的hash值即为要加入元素的hash相同
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //2.2
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        //2.1
        else {
            //不是TreeNode,即为链表,遍历链表
            for (int binCount = 0; ; ++binCount) {
                ///链表中也没有找到key值相同的节点，则生成一个新的Node,
                //并且判断链表的节点个数是不是到达转换成红黑树的上界达到，则转换成红黑树。
                if ((e = p.next) == null) {
                    //创建新结点，插到尾部
                    p.next = newNode(hash, key, value, null);
                    //超过了链表的设置长度8就转换成红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //如果e不为空就替换旧的oldValue值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

> 注：hash 冲突发生的几种情况：
>
> 1.两节点key 值相同（hash值一定相同），导致冲突；
>
> 2.两节点key 值不同，由于 hash 函数的局限性导致hash 值相同，冲突；
>
> 3.两节点key 值不同，hash 值不同，但 hash 值对数组长度取模后相同，冲突；

这里有个比较重要的点就是，如果我们用引用类型做key为什么**既要重写hashCode()方法又要重写equals()方法**。可以看到上面的代码，用一个引用类型做key。我们看到putVal()方法的17、18行，判断两个key是不是相等，首先是判断传进来的key的hash与通过hash找到那个node的hash是不是相等，如果相等了还要判断key是不是相等（通过 == 或者 equals方法判断），如果又相等了，才能认定传进来了与这个位置的key是一样的，put的操作就更新值，而不是重新插一个；get操作可以取走这个value，而不是找不到。

我们认识到hashCode()和equals()方法的作用后，我们举个例子。

~~~java
HashMap<IdStamp,Student> studentMap = new HashMap<>();
IdStamp id = new IdStamp("1班","1号");
Student s = new Student("张三","男");
studentMap.put(id,s);
//----------------------------
// 取值
Student id1 = new IdStamp("1班","1号");
studentMap.get(id1); //铁定取不到值
~~~

为什么id的信息是一致的，为但是取不到值呢？因为没有重写hashCode和equals方法。在Object类中原生的hashCode方法返回的是该对象的地址。

在上例中，id1与id的地址就不可能一样了。所以在第一关，用hash找node桶的时候，找到的桶就不一样，何谈取到正确的值呢？所以，要重写hashCode的话，就要保证信息一样的两个对象的hash是一样的，简单一点的，比如`return class.hashCode()+number.hashCode()`。

好，重写了hashCode方法之后，我们已经可以顺利的找到对应的node桶了，这时候进入第二关：用 == 或者 equals方法判断两个key是否相等。==肯定是不相等的，因为它还是判断两个对象的地址。那么就要看equals方法。然而，很不幸的是，Object类的equals方法就是用 == 判断两个对象是否相等的。那么就算我们已经找到正确的桶了，就算已经是找到我们需要的那个node了，由于没有重写equals方法，系统始终认为这两key不等。一样的，如果我们想要重写equals方法的话，就需要保证信息一样的对象是相等的，例如：`return class.equals(o) && number.equals(o) `。

**注：**我们平常用作key的String、Integer等，设计者们早就重写好了hashCode和equals方法，所以俺们才能放心大胆的使。



## 6.treeifyBin()

```java
/**
 * tab：元素数组，
 * hash：hash值（要增加的键值对的key的hash值）
 * 把Node对象转换成了TreeNode对象，把单向链表转换成了双向链表
 */
final void treeifyBin(Node<K,V>[] tab, int hash) {
 
    int n, index; Node<K,V> e;
    /*
     * 如果元素数组为空 或者 数组长度小于 树结构化的最小限制
     * MIN_TREEIFY_CAPACITY 默认值64，对于这个值可以理解为：如果元素数组长度小于这个值，没有必要去进行结构转换
     * 当一个数组位置上集中了多个键值对，那是因为这些key的hash值和数组长度取模之后结果相同。（并不是因为这些key的hash值相同）
     * 因为hash值相同的概率不高，所以可以通过扩容的方式，来使得最终这些key的hash值在和新的数组长度取模之后，拆分到多个数组位置上。
     */
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize(); // 扩容，可参见resize方法解析
 
    // 如果元素数组长度已经大于等于了 MIN_TREEIFY_CAPACITY，那么就有必要进行结构转换了
    // 根据hash值和数组长度进行取模运算后，得到链表的首节点
    else if ((e = tab[index = (n - 1) & hash]) != null) { 
        TreeNode<K,V> hd = null, tl = null; // 定义首、尾节点
        do { 
            TreeNode<K,V> p = replacementTreeNode(e, null); //下面解析 将该节点转换为 树节点
            if (tl == null) // 如果尾节点为空，说明还没有根节点
                hd = p; // 首节点（根节点）指向 当前节点
            else { // 尾节点不为空，以下两行是一个双向链表结构
                p.prev = tl; // 当前树节点的 前一个节点指向 尾节点
                tl.next = p; // 尾节点的 后一个节点指向 当前节点
            }
            tl = p; // 把当前节点设为尾节点
        } while ((e = e.next) != null); // 继续遍历链表
 
        // 到目前为止 也只是把Node对象转换成了TreeNode对象，把单向链表转换成了双向链表
 
        // 把转换后的双向链表，替换原来位置上的单向链表
        if ((tab[index] = hd) != null)
            hd.treeify(tab);//此处单独解析
    }
}
```

## 7.replacementTreeNode()、TreeNode类

当我们恰好要根据key寻找一个在链表上的对象的时候，就涉及到遍历链表，逐个调用key对象的equals方法来比对我们要查找的到底是哪个键值对了。可想当链表的长度越长，匹配的时间复杂度就越高，和链表的长度成正比。这也就是HashMap内部实现时会根据链表的长度超过限定的阈值时，会将链表结构转换为红黑树结构，用来提升查询性能。replacementTreeNode方法就是此时被调用。TreeNode类也就是红黑树的节点对象。

~~~java
// 该方法只是调用了TreeNode类的构造方法，依据当前节点信息构造一个树节点对象
TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    return new TreeNode<>(p.hash, p.key, p.value, next);
}

/**
 * 首先该类是HashMap类的一个静态内部类
 * 包内可见、不可被继承
 * 该类继承了LinkedHashMap.Entry，而LinkedHashMap.Entry继承了HashMap.Node
 * PS：要知道LinkedHashMap是HashMap的子类，然而目前的状况是HashMap作为父类，他的一个静态内部类（TreeNode）居然继承了子类LinkedHashMap的一个静态内部类
 *（LinkedHashMap.Entry），这个设计不太理解。
 * 红黑树是一个二叉树，父节点、左节点、右节点、红黑标识都是二叉树中的元素
 */
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next); // 此处调用 LinkedHashMap.Entry的构造方法
    }
 
    // 以下省略了很多方法，后文会逐步解析
}

/**
 * 首先该类是LinkedHashMap的一个静态内部类
 * 包内可见、又因为没有final修饰符（所以HashMap中的TreeNode类才能继承到他）
 * 该类除了增加了before、after两个实例变量之外，没有任何的行为扩展，也就是说他的所有行为都继承自HashMap.Node
 * 该类也只有一个构造方法，且该构造方法就是通过调用HashMap.Node的构造方法构造一个HashMap.Node对象
 * PS：看到这里就更加不理解为何HashMap.TreeNode不直接继承HashMap.Node，而要绕个弯来继承LinkedHashMap.Entry，难道是为了使用before、after？可貌似也没有使用到。
 */
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
~~~

## 8.treeify()

~~~java
/**
 * 参数为HashMap的元素数组
 */
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null; // 定义树的根节点
    for (TreeNode<K,V> x = this, next; x != null; x = next) { // 遍历链表，x指向当前节点、next指向下一个节点
        next = (TreeNode<K,V>)x.next; // 下一个节点
        x.left = x.right = null; // 设置当前节点的左右节点为空
        if (root == null) { // 如果还没有根节点
            x.parent = null; // 当前节点的父节点设为空
            x.red = false; // 当前节点的红色属性设为false（把当前节点设为黑色）
            root = x; // 根节点指向到当前节点
        }
        else { // 如果已经存在根节点了
            K k = x.key; // 取得当前链表节点的key
            int h = x.hash; // 取得当前链表节点的hash值
            Class<?> kc = null; // 定义key所属的Class
            for (TreeNode<K,V> p = root;;) { // 从根节点开始遍历，此遍历没有设置边界，只能从内部跳出
                // GOTO1
                int dir, ph; // dir 标识方向（左右）、ph标识当前树节点的hash值
                K pk = p.key; // 当前树节点的key
                if ((ph = p.hash) > h) // 如果当前树节点hash值 大于 当前链表节点的hash值
                    dir = -1; // 标识当前链表节点会放到当前树节点的左侧
                else if (ph < h)
                    dir = 1; // 右侧
 
                /*
                 * 如果两个节点的key的hash值相等，那么还要通过其他方式再进行比较
                 * 如果当前链表节点的key实现了comparable接口，并且当前树节点和链表节点是相同Class的实例，那么通过comparable的方式再比较两者。
                 * 如果还是相等，最后再通过tieBreakOrder比较一次
                 */
                else if ((kc == null &&
                            (kc = comparableClassFor(k)) == null) ||
                            (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);
 
                TreeNode<K,V> xp = p; // 保存当前树节点
 
                /*
                 * 如果dir 小于等于0 ： 当前链表节点一定放置在当前树节点的左侧，但不一定是该树节点的左孩子，也可能是左孩子的右孩子 或者 更深层次的节点。
                 * 如果dir 大于0 ： 当前链表节点一定放置在当前树节点的右侧，但不一定是该树节点的右孩子，也可能是右孩子的左孩子 或者 更深层次的节点。
                 * 如果当前树节点不是叶子节点，那么最终会以当前树节点的左孩子或者右孩子 为 起始节点  再从GOTO1 处开始 重新寻找自己（当前链表节点）的位置
                 * 如果当前树节点就是叶子节点，那么根据dir的值，就可以把当前链表节点挂载到当前树节点的左或者右侧了。
                 * 挂载之后，还需要重新把树进行平衡。平衡之后，就可以针对下一个链表节点进行处理了。
                 */
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp; // 当前链表节点 作为 当前树节点的子节点
                    if (dir <= 0)
                        xp.left = x; // 作为左孩子
                    else
                        xp.right = x; // 作为右孩子
                    root = balanceInsertion(root, x); // 重新平衡
                    break;
                }
            }
        }
    }
 
    // 把所有的链表节点都遍历完之后，最终构造出来的树可能经历多个平衡操作，根节点目前到底是链表的哪一个节点是不确定的
    // 因为我们要基于树来做查找，所以就应该把 tab[N] 得到的对象一定根节点对象，而目前只是链表的第一个节点对象，所以要做相应的处理。
    moveRootToFront(tab, root); // 单独解析
}
~~~

## 9.balanceInsertion(root, x)

balanceInsertion指的是红黑树的插入平衡算法，当树结构中新插入了一个节点后，要对树进行重新的结构化，以保证该树始终维持红黑树的特性。

关于红黑树的特性：

1. 节点是红色或黑色。
2. 根节点是黑色。
3. 每个叶节点（NIL节点，空节点）是黑色的。
4. 每个红色节点的两个子节点都是黑色。(从每个叶子到根的所有路径上不能有两个连续的红色节点)
5. 从任一节点到其每个叶子的路径上包含的黑色节点数量都相同。

~~~java
/**
 * 红黑树插入节点后，需要重新平衡
 * root 当前根节点
 * x 新插入的节点
 * 返回重新平衡后的根节点
 */
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                    TreeNode<K,V> x) {
    x.red = true; // 新插入的节点标为红色
 
    /*
     * 这一步即定义了变量，又开起了循环，循环没有控制条件，只能从内部跳出
     * xp：当前节点的父节点、xpp：爷爷节点、xppl：左叔叔节点、xppr：右叔叔节点
     */
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) { 
 
        // 如果父节点为空、说明当前节点就是根节点，那么把当前节点标为黑色，返回当前节点
        if ((xp = x.parent) == null) { // L1
            x.red = false;
            return x;
        }
 
        // 父节点不为空
        // 如果父节点为黑色 或者 【（父节点为红色 但是 爷爷节点为空） -> 这种情况何时出现？】
        else if (!xp.red || (xpp = xp.parent) == null) // L2
            return root;
        if (xp == (xppl = xpp.left)) { // 如果父节点是爷爷节点的左孩子  // L3
            if ((xppr = xpp.right) != null && xppr.red) { // 如果右叔叔不为空 并且 为红色  // L3_1
                xppr.red = false; // 右叔叔置为黑色
                xp.red = false; // 父节点置为黑色
                xpp.red = true; // 爷爷节点置为红色
                x = xpp; // 运行到这里之后，就又会进行下一轮的循环了，将爷爷节点当做处理的起始节点 
            }
            else { // 如果右叔叔为空 或者 为黑色 // L3_2
                if (x == xp.right) { // 如果当前节点是父节点的右孩子 // L3_2_1
                    root = rotateLeft(root, x = xp); // 父节点左旋，见下文左旋方法解析
                    xpp = (xp = x.parent) == null ? null : xp.parent; // 获取爷爷节点
                }
                if (xp != null) { // 如果父节点不为空 // L3_2_2
                    xp.red = false; // 父节点 置为黑色
                    if (xpp != null) { // 爷爷节点不为空
                        xpp.red = true; // 爷爷节点置为 红色
                        root = rotateRight(root, xpp);  //爷爷节点右旋，见下文右旋方法解析
                    }
                }
            }
        }
        else { // 如果父节点是爷爷节点的右孩子 // L4
            if (xppl != null && xppl.red) { // 如果左叔叔是红色 // L4_1
                xppl.red = false; // 左叔叔置为 黑色
                xp.red = false; // 父节点置为黑色
                xpp.red = true; // 爷爷置为红色
                x = xpp; // 运行到这里之后，就又会进行下一轮的循环了，将爷爷节点当做处理的起始节点 
            }
            else { // 如果左叔叔为空或者是黑色 // L4_2
                if (x == xp.left) { // 如果当前节点是个左孩子 // L4_2_1
                    root = rotateRight(root, x = xp); // 针对父节点做右旋，见下文右旋方法解析
                    xpp = (xp = x.parent) == null ? null : xp.parent; // 获取爷爷节点
                }
                if (xp != null) { // 如果父节点不为空 // L4_2_4
                    xp.red = false; // 父节点置为黑色
                    if (xpp != null) { //如果爷爷节点不为空
                        xpp.red = true; // 爷爷节点置为红色
                        root = rotateLeft(root, xpp); // 针对爷爷节点做左旋
                    }
                }
            }
        }
    }
}
 
 
/**
 * 节点左旋
 * root 根节点
 * p 要左旋的节点
 */
static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                              TreeNode<K,V> p) {
    TreeNode<K,V> r, pp, rl;
    if (p != null && (r = p.right) != null) { // 要左旋的节点以及要左旋的节点的右孩子不为空
        if ((rl = p.right = r.left) != null) // 要左旋的节点的右孩子的左节点 赋给 要左旋的节点的右孩子 节点为：rl
            rl.parent = p; // 设置rl和要左旋的节点的父子关系【之前只是爹认了孩子，孩子还没有答应，这一步孩子也认了爹】
 
        // 将要左旋的节点的右孩子的父节点  指向 要左旋的节点的父节点，相当于右孩子提升了一层，
        // 此时如果父节点为空， 说明r 已经是顶层节点了，应该作为root 并且标为黑色
        if ((pp = r.parent = p.parent) == null) 
            (root = r).red = false;
        else if (pp.left == p) // 如果父节点不为空 并且 要左旋的节点是个左孩子
            pp.left = r; // 设置r和父节点的父子关系【之前只是孩子认了爹，爹还没有答应，这一步爹也认了孩子】
        else // 要左旋的节点是个右孩子
            pp.right = r; 
        r.left = p; // 要左旋的节点  作为 他的右孩子的左节点
        p.parent = r; // 要左旋的节点的右孩子  作为  他的父节点
    }
    return root; // 返回根节点
}
 
/**
 * 节点右旋
 * root 根节点
 * p 要右旋的节点
 */
static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                               TreeNode<K,V> p) {
    TreeNode<K,V> l, pp, lr;
    if (p != null && (l = p.left) != null) { // 要右旋的节点不为空以及要右旋的节点的左孩子不为空
        if ((lr = p.left = l.right) != null) // 要右旋的节点的左孩子的右节点 赋给 要右旋节点的左孩子 节点为：lr
            lr.parent = p; // 设置lr和要右旋的节点的父子关系【之前只是爹认了孩子，孩子还没有答应，这一步孩子也认了爹】
 
        // 将要右旋的节点的左孩子的父节点  指向 要右旋的节点的父节点，相当于左孩子提升了一层，
        // 此时如果父节点为空， 说明l 已经是顶层节点了，应该作为root 并且标为黑色
        if ((pp = l.parent = p.parent) == null) 
            (root = l).red = false;
        else if (pp.right == p) // 如果父节点不为空 并且 要右旋的节点是个右孩子
            pp.right = l; // 设置l和父节点的父子关系【之前只是孩子认了爹，爹还没有答应，这一步爹也认了孩子】
        else // 要右旋的节点是个左孩子
            pp.left = l; // 同上
        l.right = p; // 要右旋的节点 作为 他左孩子的右节点
        p.parent = l; // 要右旋的节点的父节点 指向 他的左孩子
    }
    return root;
}
~~~

## 10.TreeNode类的moveRootToFront方法

TreeNode在增加或删除节点后，都需要对整个树重新进行平衡，平衡之后的根节点也许就会发生变化，此时为了保证：如果HashMap元素数组根据下标取得的元素是一个TreeNode类型，那么这个TreeNode节点一定要是这颗树的根节点，同时也要是整个链表的首节点。

~~~java
	/**
 * 把红黑树的根节点设为  其所在的数组槽 的第一个元素
 * 首先明确：TreeNode既是一个红黑树结构，也是一个双链表结构
 * 这个方法里做的事情，就是保证树的根节点一定也要成为链表的首节点
 */
static <K,V> void moveRootToFront(Node<K,V>[] tab, TreeNode<K,V> root) {
    int n;
    if (root != null && tab != null && (n = tab.length) > 0) { // 根节点不为空 并且 HashMap的元素数组不为空
        int index = (n - 1) & root.hash; // 根据根节点的Hash值 和 HashMap的元素数组长度  取得根节点在数组中的位置
        TreeNode<K,V> first = (TreeNode<K,V>)tab[index]; // 首先取得该位置上的第一个节点对象
        if (root != first) { // 如果该节点对象 与 根节点对象 不同
            Node<K,V> rn; // 定义根节点的后一个节点
            tab[index] = root; // 把元素数组index位置的元素替换为根节点对象
            TreeNode<K,V> rp = root.prev; // 获取根节点对象的前一个节点
            if ((rn = root.next) != null) // 如果后节点不为空 
                ((TreeNode<K,V>)rn).prev = rp; // root后节点的前节点  指向到 root的前节点，相当于把root从链表中摘除
            if (rp != null) // 如果root的前节点不为空
                rp.next = rn; // root前节点的后节点 指向到 root的后节点
            if (first != null) // 如果数组该位置上原来的元素不为空
                first.prev = root; // 这个原有的元素的 前节点 指向到 root，相当于root目前位于链表的首位
            root.next = first; // 原来的第一个节点现在作为root的下一个节点，变成了第二个节点
            root.prev = null; // 首节点没有前节点
        }
 
        /*
         * 这一步是防御性的编程
         * 校验TreeNode对象是否满足红黑树和双链表的特性
         * 如果这个方法校验不通过：可能是因为用户编程失误，破坏了结构（例如：并发场景下）；也可能是TreeNode的实现有问题（这个是理论上的以防万一）；
         */ 
        assert checkInvariants(root); 
    }
}
~~~

## 11.putTreeVal方法

~~~java
/**
 * 当存在hash碰撞的时候，且元素数量大于8个时候，就会以红黑树的方式将这些元素组织起来
 * map 当前节点所在的HashMap对象
 * tab 当前HashMap对象的元素数组
 * h   指定key的hash值
 * k   指定key
 * v   指定key上要写入的值
 * 返回：指定key所匹配到的节点对象，针对这个对象去修改V（返回空说明创建了一个新节点）
 */
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                int h, K k, V v) {
    Class<?> kc = null; // 定义k的Class对象
    boolean searched = false; // 标识是否已经遍历过一次树，未必是从根节点遍历的，但是遍历路径上一定已经包含了后续需要比对的所有节点。
    TreeNode<K,V> root = (parent != null) ? root() : this; // 父节点不为空那么查找根节点，为空那么自身就是根节点
    for (TreeNode<K,V> p = root;;) { // 从根节点开始遍历，没有终止条件，只能从内部退出
        int dir, ph; K pk; // 声明方向、当前节点hash值、当前节点的键对象
        if ((ph = p.hash) > h) // 如果当前节点hash 大于 指定key的hash值
            dir = -1; // 要添加的元素应该放置在当前节点的左侧
        else if (ph < h) // 如果当前节点hash 小于 指定key的hash值
            dir = 1; // 要添加的元素应该放置在当前节点的右侧
        else if ((pk = p.key) == k || (k != null && k.equals(pk))) // 如果当前节点的键对象 和 指定key对象相同
            return p; // 那么就返回当前节点对象，在外层方法会对v进行写入
 
        // 走到这一步说明 hash值 相等的，但是equals不等，就还需要通过其他的方法判断。||（或）前面
        // 的方法用来判断 插入的key有没有实现comparable接口，有实现的话就通过后面的方法比较
        else if ((kc == null &&
                    (kc = comparableClassFor(k)) == null) ||
                    (dir = compareComparables(kc, k, pk)) == 0) {
 
            // 走到这里说明：指定key没有实现comparable接口   或者   实现了comparable接口并且和当前节点的键对象比较之后相等（那么下面做的就是，从当前节点递归查找有没有与当前key是同一个的节点，找到了就返回，没找到就还是继续循环，找到能插入的空）
        
 
            /*
             * searched 标识是否已经对比过当前节点的左右子节点了
             * 如果还没有遍历过，那么就递归遍历对比，看是否能够得到那个键对象equals相等的的节点
             * 如果得到了键的equals相等的的节点就返回
             * 如果还是没有键的equals相等的节点，那说明应该创建一个新节点了。
             */
            if (!searched) { // 如果还没有比对过当前节点的所有子节点
                TreeNode<K,V> q, ch; // 定义要返回的节点、和子节点
                searched = true; // 标识已经遍历过一次了
                /*
                 * 红黑树也是二叉树，所以只要沿着左右两侧遍历寻找就可以了
                 * 这是个短路运算，如果先从左侧就已经找到了，右侧就不需要遍历了
                 * find 方法内部还会有递归调用。参见：find方法解析
                 */
                if (((ch = p.left) != null &&
                        (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                        (q = ch.find(h, k, kc)) != null))
                    return q; // 找到了指定key键对应的
            }
 
            // 走到这里就说明，遍历了所有子节点也没有找到和当前键equals相等的节点
            dir = tieBreakOrder(k, pk); // 再比较一下当前节点键和指定key键的大小
        }
 
        TreeNode<K,V> xp = p; // 定义xp指向当前节点
        /*
        * 如果dir小于等于0，那么看当前节点的左节点是否为空，如果为空，就可以把要添加的元素作为当前节点的左节点，如果不为空，还需要下一轮继续比较
        * 如果dir大于等于0，那么看当前节点的右节点是否为空，如果为空，就可以把要添加的元素作为当前节点的右节点，如果不为空，还需要下一轮继续比较
        * 如果以上两条当中有一个子节点不为空，这个if中还做了一件事，那就是把p已经指向了对应的不为空的子节点，开始下一轮的比较
        */
        if ((p = (dir <= 0) ? p.left : p.right) == null) {  
            // 如果恰好要添加的方向上的子节点为空，此时节点p已经指向了这个空的子节点
            Node<K,V> xpn = xp.next; // 获取当前节点的next节点
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn); // 创建一个新的树节点
            if (dir <= 0)
                xp.left = x;  // 左孩子指向到这个新的树节点
            else
                xp.right = x; // 右孩子指向到这个新的树节点
            xp.next = x; // 链表中的next节点指向到这个新的树节点
            x.parent = x.prev = xp; // 这个新的树节点的父节点、前节点均设置为 当前的树节点
            if (xpn != null) // 如果原来的next节点不为空
                ((TreeNode<K,V>)xpn).prev = x; // 那么原来的next节点的前节点指向到新的树节点
            moveRootToFront(tab, balanceInsertion(root, x));// 重新平衡，以及新的根节点置顶
            return null; // 返回空，意味着产生了一个新节点
        }
    }
}
 
 
~~~

## comparableClassFor方法

~~~java
/**
* 如果对象x的类是C，如果C实现了Comparable<C>接口，那么返回C，否则返回null
*/
static Class<?> comparableClassFor(Object x) {
    if (x instanceof Comparable) {
        Class<?> c; Type[] ts, as; Type t; ParameterizedType p;
        //getClass()方法返回一个该对象运行时类型对象（无论该对如何转型，返回new后面的类型）
        if ((c = x.getClass()) == String.class) // 如果x是个字符串对象
            return c; // 返回String.class
        /*
         * 为什么如果x是个字符串就直接返回c了呢 ? 因为String  实现了 Comparable 接口，可参考如下String类的定义
         * public final class String implements java.io.Serializable, Comparable<String>, CharSequence
         */ 
 
        // 如果 c 不是字符串类，获取c运行时类型直接（从父类继承不行）实现的接口（如果是泛型接口则附带泛型信息） 
        if ((ts = c.getGenericInterfaces()) != null) {
            for (int i = 0; i < ts.length; ++i) { // 遍历接口数组
                // 如果当前接口t是个泛型接口 
                // 如果该泛型接口t的原始类型p 是 Comparable 接口
                // 如果该Comparable接口p只定义了一个泛型参数
                // 如果这一个泛型参数的类型就是c，那么返回c
                if (((t = ts[i]) instanceof ParameterizedType) &&
                    ((p = (ParameterizedType)t).getRawType() ==
                        Comparable.class) &&
                    (as = p.getActualTypeArguments()) != null &&
                    as.length == 1 && as[0] == c) // type arg is c
                    return c;
            }
            // 上面for循环的目的就是为了看看x的class是否 implements  Comparable<x的class>
        }
    }
    return null; // 如果c并没有实现 Comparable<c> 那么返回空
}


/**
* 如果x所属的类是kc，返回k.compareTo(x)的比较结果
* 如果x为空，或者其所属的类不是kc，返回0
*/
@SuppressWarnings({"rawtypes","unchecked"}) // for cast to Comparable
static int compareComparables(Class<?> kc, Object k, Object x) {
    return (x == null || x.getClass() != kc ? 0 :
            ((Comparable)k).compareTo(x));
}



/**
* 如果两者不具有compare的资格，或者compare之后仍然没有比较出大小。那么就要通过一个决胜局再比一次，这
* 个决胜局就是tieBreakOrder方法。
* 用这个方法来比较两个对象，返回值要么大于0，要么小于0，不会为0
* 也就是说这一步一定能确定要插入的节点要么是树的左节点，要么是右节点，不然就无法继续满足二叉树结构了
* 
* 先比较两个对象的类名，类名是字符串对象，就按字符串的比较规则
* 如果两个对象是同一个类型，那么调用本地方法为两个对象生成hashCode值，再进行比较，hashCode相等的话返回-1
*/
static int tieBreakOrder(Object a, Object b) {
    int d;
    if (a == null || b == null ||
        (d = a.getClass().getName().
            compareTo(b.getClass().getName())) == 0)
        d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
                -1 : 1);
    return d;
}
~~~

## remove方法

~~~java
/**
* 从HashMap中删除掉指定key对应的键值对，并返回被删除的键值对的值
* 如果返回空，说明key可能不存在，也可能key对应的值就是null
* 如果想确定到底key是否存在可以使用containsKey方法
*/
public V remove(Object key) {
    Node<K,V> e; // 定义一个节点变量，用来存储要被删除的节点（键值对）
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value; // 调用removeNode方法
}



/**
* 方法为final，不可被覆写，子类可以通过实现afterNodeRemoval方法来增加自己的处理逻辑（解析中有描述）
*
* @param hash key的hash值，该值是通过hash(key)获取到的
* @param key 要删除的键值对的key
* @param value 要删除的键值对的value，该值是否作为删除的条件取决于matchValue是否为true
* @param matchValue 如果为true，则当key对应的键值对的值equals(value)为true时才删除；否则不关心value的值
* @param movable 删除后是否移动节点，如果为false，则不移动
* @return 返回被删除的节点对象，如果没有删除任何节点则返回null
*/
final Node<K,V> removeNode(int hash, Object key, Object value,
                            boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index; // 声明节点数组、当前节点、数组长度、索引值
    /*
     * 如果 节点数组tab不为空、数组长度n大于0、根据hash定位到的节点对象p（该节点为 树的根节点 或 链表的首节点）不为空
     * 需要从该节点p向下遍历，找到那个和key匹配的节点对象
     */
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v; // 定义要返回的节点对象，声明一个临时节点变量、键变量、值变量
 
        // 如果当前节点的键和key相等，那么当前节点就是要删除的节点，赋值给node
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
 
        /*
         * 到这一步说明首节点没有匹配上，那么检查下是否有next节点
         * 如果没有next节点，就说明该节点所在位置上没有发生hash碰撞, 就一个节点并且还没匹配上，也就没得删了，最终也就返回null了
         * 如果存在next节点，就说明该数组位置上发生了hash碰撞，此时可能存在一个链表，也可能是一颗红黑树
         */
        else if ((e = p.next) != null) {
            // 如果当前节点是TreeNode类型，说明已经是一个红黑树，那么调用getTreeNode方法从树结构中查找满足条件的节点
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            // 如果不是树节点，那么就是一个链表，只需要从头到尾逐个节点比对即可    
            else {
                do {
                    // 如果e节点的键是否和key相等，e节点就是要删除的节点，赋值给node变量，调出循环
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                            (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
 
                    // 走到这里，说明e也没有匹配上
                    p = e; // 把当前节点p指向e，这一步是让p存储的永远下一次循环里e的父节点，如果下一次e匹配上了，那么p就是node的父节点
                } while ((e = e.next) != null); // 如果e存在下一个节点，那么继续去匹配下一个节点。直到匹配到某个节点跳出 或者 遍历完链表所有节点
            }
        }
 
        /*
         * 如果node不为空，说明根据key匹配到了要删除的节点
         * 如果不需要对比value值  或者  需要对比value值但是value值也相等
         * 那么就可以删除该node节点了
         */
        if (node != null && (!matchValue || (v = node.value) == value ||
                                (value != null && value.equals(v)))) {
            if (node instanceof TreeNode) // 如果该节点是个TreeNode对象，说明此节点存在于红黑树结构中，调用removeTreeNode方法（该方法单独解析）移除该节点（红黑树删除比较难，插入较简单）
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p) // 如果该节点不是TreeNode对象，node == p 的意思是该node节点就是首节点
                tab[index] = node.next; // 由于删除的是首节点，那么直接将节点数组对应位置指向到第二个节点即可
            else // 如果node节点不是首节点，此时p是node的父节点，由于要删除node，所有只需要把p的下一个节点指向到node的下一个节点即可把node从链表中删除了
                p.next = node.next;
            ++modCount; // HashMap的修改次数递增
            --size; // HashMap的元素个数递减
            afterNodeRemoval(node); // 调用afterNodeRemoval方法，该方法HashMap没有任何实现逻辑，目的是为了让子类根据需要自行覆写
            return node;
        }
    }
    return null
~~~

## 1.7和1.8的HashMap的不同点

1. JDK1.7用的是头插法，而JDK1.8及之后使用的都是尾插法，那么为什么要这样做呢？因为JDK1.7是用单链表进行的纵向延伸，当采用头插法就是能够提高插入的效率，但是也会容易出现逆序且环形链表死循环问题。但是在JDK1.8之后是因为加入了红黑树使用尾插法，能够避免出现逆序且链表死循环的问题。
2. 扩容后数据存储位置的计算方式也不一样：
   - 在JDK1.7的时候是直接用hash值和需要扩容的二进制数进行&amp;（这里就是为什么扩容的时候为啥一定必须是2的多少次幂的原因所在，因为如果只有2的n次幂的情况时最后一位二进制数才一定是1，这样能最大程度减少hash碰撞）（hash值 &amp; length-1） 。
   - 而在JDK1.8的时候直接用了JDK1.7的时候计算的规律，也就是扩容前的原始位置+扩容的大小值=JDK1.8的计算方式，而不再是JDK1.7的那种异或的方法。但是这种方式就相当于只需要判断Hash值的新增参与运算的位是0还是1就直接迅速计算出了扩容后的储存方式。
   - JDK1.7的时候使用的是数组+ 单链表的数据结构。但是在JDK1.8及之后时，使用的是数组+链表+红黑树的数据结构（当链表的深度达到8（也就是默认阈值）且桶数大于等于64的时候，就会自动扩容把链表转成红黑树的数据结构来把时间复杂度从O（N）变成O（logN）提高了效率）。

## HashMap为什么是线程不安全的？

HashMap 在并发时可能出现的问题主要是两方面：

1. put的时候导致的多线程数据不一致

   比如有两个线程A和B，首先A希望插入一个key-value对到HashMap中，首先计算记录所要落到的 hash桶的索引坐标，然后获取到该桶里面的链表头结点，此时线程A的时间片用完了，而此时线程B被调度得以执行，和线程A一样执行，只不过线程B成功将记录插到了桶里面，假设线程A插入的记录计算出来的 hash桶索引和线程B要插入的记录计算出来的 hash桶索引是一样的，那么当线程B成功插入之后，线程A再次被调度运行时，它依然持有过期的链表头但是它对此一无所知，以至于它认为它应该这样做，如此一来就覆盖了线程B插入的记录，这样线程B插入的记录就凭空消失了，造成了数据不一致的行为。

2. resize而引起死循环

   这种情况发生在HashMap自动扩容时，当2个线程同时检测到元素个数超过 数组大小 × 负载因子。此时2个线程会在put()方法中调用了resize()，两个线程同时修改一个链表结构会产生一个循环链表（JDK1.7中，会出现resize前后元素顺序倒置的情况）。接下来再想通过get()获取某一个元素，就会出现死循环。

## HashMap和HashTable的区别

1. HashMap几乎可以等价于Hashtable，除了HashMap是非synchronized的，并可以接受null(HashMap可以接受为null的键值(key)和值(value)，而Hashtable则不行)。
2. HashMap是非synchronized，而Hashtable是synchronized，这意味着Hashtable是线程安全的，多个线程可以共享一个Hashtable；而如果没有正确的同步的话，多个线程是不能共享HashMap的。Java 5提供了ConcurrentHashMap，它是HashTable的替代，比HashTable的扩展性更好。
3. 另一个区别是HashMap的迭代器(Iterator)是fail-fast（你不能用迭代器遍历，而使用容器的移除方法）迭代器，而Hashtable的enumerator迭代器不是fail-fast的。所以当有其它线程改变了HashMap的结构（增加或者移除元素），将会抛出ConcurrentModificationException，但迭代器本身的remove()方法移除元素则不会抛出ConcurrentModificationException异常。但这并不是一个一定发生的行为，要看JVM。这条同样也是Enumeration和Iterator的区别。
4. 由于Hashtable是线程安全的也是synchronized，所以在单线程环境下它比HashMap要慢。如果你不需要同步，只需要单一线程，那么使用HashMap性能要好过Hashtable。
5. HashMap不能保证随着时间的推移Map中的元素次序是不变的。