# 多态

~~~java
Person p = new Man();//Person是编译时类型，Man是运行时类型。多态：方法重载（编译时多态）、方法重写（运行时多态）。成员变量没有多态，由编译时类型决定。
~~~

# Java关键字

transient只能修饰成员变量，且作用域为同一包下，对其他包不可见。

# 内部类

将一个类设计成内部类，我觉得有两方面因素：

1. 在一个类内部，需要操作某种属性，而这个属性需要涉及的面又很广，我们可以考虑，将这些属性设计为内部类。

2. 然后就如下面 [铁心男](https://www.zhihu.com/people/tie-xin-nan) 用户回答的那样，好比你设计类 B 的目的只是为了给类 A 使用，那么，我们就可将其设定为内部类，没有必要将类 B 设置成单独的 Java 文件，防止与其他类产生依赖关系。

然后我们再来说说为什么又将内部类设计为静态内部类与内部类：

1. 首先来看一下静态内部类的特点：如 [昭言](https://www.zhihu.com/people/hujf) 用户所述那样，我是静态内部类，只不过是想借你的外壳用一下。本身来说，我和你没有什么“强依赖”上的关系。没有你，我也可以创建实例。那么，在设计内部类的时候我们就可以做出权衡：如果我内部类与你外部类关系不紧密，耦合程度不高，不需要访问外部类的所有属性或方法，那么我就设计成静态内部类。而且，由于静态内部类与外部类并不会保存相互之间的引用，因此在一定程度上，还会节省那么一点内存资源，何乐而不为呢~~

2. 既然上面已经说了什么时候应该用静态内部类，那么如果你的需求不符合静态内部类所提供的一切好处，你就应该考虑使用内部类了。最大的特点就是：你在内部类中需要访问有关外部类的所有属性及方法，我知晓你的一切... ... 

总结：首先需要知道为什么会有内部类，什么时候应该使用内部类，我们再去讨论，为什么 Java 的设计者们又将内部类设计为静态与非静态，这样就很清晰了。

## 内部类加载时机

无论是静态内部类还是普通内部类，都是在第一次用的时候加载。如果直接访问静态内部类，则外部类不会加载。

## 局部内部类

写在方法或者代码段中的内部类（包括匿名内部类）。

局部内部类为什么只能访问final局部变量，对于成员变量却可以随便访问？

局部变量和成员变量对于内部类而言，具有一定的共性，都是该内部类外面的变量。如果要求内部类只能访问final的局部变量是为了确保局部变量不被修改的话，那么内部类访问成员变量应该也有类似的限制才对

我认为是由于他们的存活范围导致了这个区别：

首先内部类的实例可以在方法结束后依然存活，局部变量在方法结束后却无法存活，所以在内部类中无法访问NON-final的局部变量；

而成员变量的存活时间是取决于外部类的实例的，内部类实例中都会引用当前外部类实例，所以他们拥有一致的生命周期，于是可以访问成员变量。 

剩下的问题是，为什么可以访问final的局部变量呢？

如果将一个访问了final的局部变量的内部类进行反编译，可以发现该变量是被作为构造函数的参数传入进去的，与之一起传入的参数还有外部类实例

# 容器（Collection、Map）

<img src="Java源码.assets/Collection.png" alt="Collection" style="zoom:50%;" />



<img src="Java源码.assets/Map.png" alt="Map" style="zoom:50%;" />

## LinkedList

1. LinkedList内部维护了一个Node静态内部类，Node节点有item、prev、next。LinkedList有first、head分别指向头尾节点。

## ArrayList

1. ArrayList底层由Object[] elementData数组实现，其中批量删除、添加多由Arrays.copy()方法实现，而Arrays.copy()又调用的System.arraycopy()，所以自己复制数组的时候可以使用System.arraycopy()方法，速度较快。

## PriorityQueue

优先级队列，底层维护了一个数组，逻辑上是完全二叉树构建的堆。默认是小顶堆，可以通过传递比较器自定义优先级。

## HashMap

HashMap使用链表法避免哈希冲突（相同hash值）,当链表长度大于TREEIFY_THRESHOLD（默认为8）时，将链表转换为红黑树，当然小于UNTREEIFY_THRESHOLD（默认为6）时，又会转回链表以达到性能均衡。

![](Java源码.assets/HashMap结构图.png)

### 1. tableSizeFor(int cap)方法

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

![](Java源码.assets/tableSizeFor.png)



### 2. HashMap(Map<? extends K, ? extends V> m)

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

### 3.hash(key)

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

### 4.resize()方法

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
    /* 1. resize()函数在size > threshold时被调用。oldCap大于 0 代表原来的 table表非空, 			oldCap 为原表的大小,oldThr(threshold) 为 oldCap × load_factor
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

![](Java源码.assets/Hash定位图.png)

元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

<img src="Java源码.assets/resize.png" style="zoom:67%;" />

因此，我们在扩充HashMap的时候，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，可以看看下图为16扩充为32的resize示意图 ：

<img src="Java源码.assets/HashMap扩容.png" style="zoom:67%;" />

什么时候扩容：

通过HashMap源码可以看到是在put操作时，即向容器中添加元素时，判断当前容器中元素的个数是否达到阈值（当前数组长度乘以加载因子的值）的时候，就要自动扩容了。

扩容(resize)：

其实就是重新计算容量；而这个扩容是计算出所需容器的大小之后重新定义一个新的容器，将原来容器中的元素放入其中。

### 5.putVal()

<img src="Java源码.assets/HashMap插入流程图.png" style="zoom:60%;" />

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



### 6.treeifyBin()

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

### 7.replacementTreeNode()、TreeNode类

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

### 8.treeify()

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

### 9.balanceInsertion(root, x)

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

### 10.TreeNode类的moveRootToFront方法

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

### 11.putTreeVal方法

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

### comparableClassFor方法

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

### remove方法

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

### 1.7和1.8的HashMap的不同点

1. JDK1.7用的是头插法，而JDK1.8及之后使用的都是尾插法，那么为什么要这样做呢？因为JDK1.7是用单链表进行的纵向延伸，当采用头插法就是能够提高插入的效率，但是也会容易出现逆序且环形链表死循环问题。但是在JDK1.8之后是因为加入了红黑树使用尾插法，能够避免出现逆序且链表死循环的问题。
2. 扩容后数据存储位置的计算方式也不一样：
   - 在JDK1.7的时候是直接用hash值和需要扩容的二进制数进行&amp;（这里就是为什么扩容的时候为啥一定必须是2的多少次幂的原因所在，因为如果只有2的n次幂的情况时最后一位二进制数才一定是1，这样能最大程度减少hash碰撞）（hash值 &amp; length-1） 。
   - 而在JDK1.8的时候直接用了JDK1.7的时候计算的规律，也就是扩容前的原始位置+扩容的大小值=JDK1.8的计算方式，而不再是JDK1.7的那种异或的方法。但是这种方式就相当于只需要判断Hash值的新增参与运算的位是0还是1就直接迅速计算出了扩容后的储存方式。
   - JDK1.7的时候使用的是数组+ 单链表的数据结构。但是在JDK1.8及之后时，使用的是数组+链表+红黑树的数据结构（当链表的深度达到8的时候，也就是默认阈值，就会自动扩容把链表转成红黑树的数据结构来把时间复杂度从O（N）变成O（logN）提高了效率）。

### HashMap为什么是线程不安全的？

HashMap 在并发时可能出现的问题主要是两方面：

1. put的时候导致的多线程数据不一致

   比如有两个线程A和B，首先A希望插入一个key-value对到HashMap中，首先计算记录所要落到的 hash桶的索引坐标，然后获取到该桶里面的链表头结点，此时线程A的时间片用完了，而此时线程B被调度得以执行，和线程A一样执行，只不过线程B成功将记录插到了桶里面，假设线程A插入的记录计算出来的 hash桶索引和线程B要插入的记录计算出来的 hash桶索引是一样的，那么当线程B成功插入之后，线程A再次被调度运行时，它依然持有过期的链表头但是它对此一无所知，以至于它认为它应该这样做，如此一来就覆盖了线程B插入的记录，这样线程B插入的记录就凭空消失了，造成了数据不一致的行为。

2. resize而引起死循环

   这种情况发生在HashMap自动扩容时，当2个线程同时检测到元素个数超过 数组大小 × 负载因子。此时2个线程会在put()方法中调用了resize()，两个线程同时修改一个链表结构会产生一个循环链表（JDK1.7中，会出现resize前后元素顺序倒置的情况）。接下来再想通过get()获取某一个元素，就会出现死循环。

### HashMap和HashTable的区别

1. HashMap几乎可以等价于Hashtable，除了HashMap是非synchronized的，并可以接受null(HashMap可以接受为null的键值(key)和值(value)，而Hashtable则不行)。
2. HashMap是非synchronized，而Hashtable是synchronized，这意味着Hashtable是线程安全的，多个线程可以共享一个Hashtable；而如果没有正确的同步的话，多个线程是不能共享HashMap的。Java 5提供了ConcurrentHashMap，它是HashTable的替代，比HashTable的扩展性更好。
3. 另一个区别是HashMap的迭代器(Iterator)是fail-fast迭代器，而Hashtable的enumerator迭代器不是fail-fast的。所以当有其它线程改变了HashMap的结构（增加或者移除元素），将会抛出ConcurrentModificationException，但迭代器本身的remove()方法移除元素则不会抛出ConcurrentModificationException异常。但这并不是一个一定发生的行为，要看JVM。这条同样也是Enumeration和Iterator的区别。
4. 由于Hashtable是线程安全的也是synchronized，所以在单线程环境下它比HashMap要慢。如果你不需要同步，只需要单一线程，那么使用HashMap性能要好过Hashtable。
5. HashMap不能保证随着时间的推移Map中的元素次序是不变的。



# 多线程

## 线程状态图

<img src="Java源码.assets/线程状态图.png" style="zoom:75%;" />

## synchornized关键字

synchornized可以修饰代码块、普通方法、静态方法。当synchornize(Object obj)修饰代码块或者普通方法时，每个对象都拥有一把锁，当synchornized(Object.class)或者修饰静态方法时，锁住的不是单个对象，而是这个类的所有对象。

~~~java
public class MyThread {

    public static void main(String[] args) throws InterruptedException {

        Task task1 = new Task();
        Task task2 = new Task();
        Thread t1 = new Thread(task1, "thread1");
        Thread t2 = new Thread(task2, "thread2");
        t1.start();
        t2.start();

    }


}

class Task implements Runnable {

    static int ticketCount = 10;

    public void run() {
        saleTicket();
    }

    private void saleTicket() {
        synchronized (this) {
            for (int i = 5; i > 0; i--) {
                System.out.println(Thread.currentThread().getName() + "卖出：第" + ticketCount--);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }

        }
    }

    private void readTicket() {
        System.out.println(ticketCount);
    }
}
~~~

---



## package java.util.concurrent.atomic包内的类

基本类型：AtomicInteger、AtomicLong等

引用类型：AtomicReference

- 只能保证引用地址的并发安全
- 不能保证内部引用类型的属性的并发安全

内部维护了一个Unsafe类对象，原子操作都是由该类的对象调用本地的CAS方法实现的。

-------

0000 0111







## Unsafe类

Unsafe类中有很多基于CAS操作的native方法。

------



## ConcurrentHashMap

对比HashMap学习，有很多相似之处

### 几个重要的成员属性、变量

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

### ConcurrentHashMap(int initialCapacity)

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

### put方法

~~~java
/* put方法只调用了putVal方法 */
public V put(K key, V value) {
    return putVal(key, value, false);
}
~~~

介绍putVal方法前，先介绍一下spread方法，做一下铺垫

### spread方法

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

### putVal方法

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

### initTable方法

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

### CounterCell静态内部类

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

### addCount

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

### resizeStamp方法

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

### ForwardingNode类

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

### helpTransfer方法

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

### tansfer方法

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



## CopyOnWriteArrayList

### 写入时复制（CopyOnWrite）思想

　　写入时复制（CopyOnWrite，简称COW）思想是计算机程序设计领域中的一种优化策略。其核心思想是，如果有多个调用者（Callers）同时要求相同的资源（如内存或者是磁盘上的数据存储），他们会共同获取相同的指针指向相同的资源，直到某个调用者视图修改资源内容时，系统才会真正复制一份专用副本（private copy）给该调用者，而其他调用者所见到的最初的资源仍然保持不变。这过程对其他的调用者都是透明的（transparently）。此做法主要的优点是如果调用者没有修改资源，就不会有副本（private copy）被创建，因此多个调用者只是读取操作时可以共享同一份资源。

### CopyOnWriteArrayList的实现原理

在使用CopyOnWriteArrayList之前，我们先阅读其源码了解下它是如何实现的。以下代码是向CopyOnWriteArrayList中add方法的实现（向CopyOnWriteArrayList里添加元素），可以发现在添加的时候是需要加锁的，否则多线程写的时候会Copy出N个副本出来。

```java

public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

读的时候不需要加锁，如果读的时候有多个线程正在向CopyOnWriteArrayList添加数据，读还是会读到旧的数据，因为写的时候不会锁住旧的CopyOnWriteArrayList。

```java
public E get(int index) {
    return get(getArray(), index);
}
```

JDK中并没有提供CopyOnWriteMap，我们可以参考CopyOnWriteArrayList来实现一个，基本代码如下：

```java
import java.util.Collection;
import java.util.Map;
import java.util.Set;
 
public class CopyOnWriteMap<K, V> implements Map<K, V>, Cloneable {
    private volatile Map<K, V> internalMap;
 
    public CopyOnWriteMap() {
        internalMap = new HashMap<K, V>();
    }
 
    public V put(K key, V value) {
 
        synchronized (this) {
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            V val = newMap.put(key, value);
            internalMap = newMap;
            return val;
        }
    }
 
    public V get(Object key) {
        return internalMap.get(key);
    }
 
    public void putAll(Map<? extends K, ? extends V> newData) {
        synchronized (this) {
            Map<K, V> newMap = new HashMap<K, V>(internalMap);
            newMap.putAll(newData);
            internalMap = newMap;
        }
    }
}
```

实现很简单，只要了解了CopyOnWrite机制，我们可以实现各种CopyOnWrite容器，并且在不同的应用场景中使用。

### 几个要点

- 内部持有一个ReentrantLock lock = new ReentrantLock();
- 底层是用volatile transient声明的数组 array
- 读写分离，写时复制出一个新的数组，完成插入、修改或者移除操作后将新数组赋值给array

注：

- **volatile** （挥发物、易变的）：变量修饰符，只能用来修饰变量。volatile修饰的成员变量在每次被线程访问时，都强迫从共享内存中重读该成员变量的值。而且，当成员变量发生变 化时，强迫线程将变化值回写到共享内存。这样在任何时刻，两个不同的线程总是看到某个成员变量的同一个值。

- **transient （**暂短的、临时的**）**：修饰符，只能用来修饰字段。在对象序列化的过程中，标记为transient的变量不会被序列化。

### CopyOnWrite的应用场景

CopyOnWrite并发容器用于读多写少的并发场景。比如白名单，黑名单，商品类目的访问和更新场景，假如我们有一个搜索网站，用户在这个网站的搜索框中，输入关键字搜索内容，但是某些关键字不允许被搜索。这些不能被搜索的关键字会被放在一个黑名单当中，黑名单每天晚上更新一次。当用户搜索时，会检查当前关键字在不在黑名单当中，如果在，则提示不能搜索。实现代码如下：

```java
import java.util.Map;
 
import com.ifeve.book.forkjoin.CopyOnWriteMap;
 
/**
 * 黑名单服务
 *
 * @author fangtengfei
 *
 */
public class BlackListServiceImpl {
 
    private static CopyOnWriteMap<String, Boolean> blackListMap = new CopyOnWriteMap<String, Boolean>(
            1000);
 
    public static boolean isBlackList(String id) {
        return blackListMap.get(id) == null ? false : true;
    }
 
    public static void addBlackList(String id) {
        blackListMap.put(id, Boolean.TRUE);
    }
 
    /**
     * 批量添加黑名单
     *
     * @param ids
     */
    public static void addBlackList(Map<String,Boolean> ids) {
        blackListMap.putAll(ids);
    }
 
}[![复制代码](Eeffective%20Java.assets/copycode.gif)](javascript:void(0);
```

代码很简单，但是使用CopyOnWriteMap需要注意两件事情：

1. 减少扩容开销。根据实际需要，初始化CopyOnWriteMap的大小，避免写时CopyOnWriteMap扩容的开销。
2. 使用批量添加。因为每次添加，容器每次都会进行复制，所以减少添加次数，可以减少容器的复制次数。如使用上面代码里的addBlackList方法。

### CopyOnWrite的缺点　

CopyOnWrite容器有很多优点，但是同时也存在两个问题，即内存占用问题和数据一致性问题。所以在开发的时候需要注意一下。

- **内存占用问题**。因为CopyOnWrite的写时复制机制，所以在进行写操作的时候，内存里会同时驻扎两个对象的内存，旧的对象和新写入的对象（注意:在复制的时候只是复制容器里的引用，只是在写的时候会创建新对象添加到新容器里，而旧容器的对象还在使用，所以有两份对象内存）。如果这些对象占用的内存比较大，比如说200M左右，那么再写入100M数据进去，内存就会占用300M，那么这个时候很有可能造成频繁的Yong GC和Full GC。之前我们系统中使用了一个服务由于每晚使用CopyOnWrite机制更新大对象，造成了每晚15秒的Full GC，应用响应时间也随之变长。针对内存占用问题，可以通过压缩容器中的元素的方法来减少大对象的内存消耗，比如，如果元素全是10进制的数字，可以考虑把它压缩成36进制或64进制。或者不使用CopyOnWrite容器，而使用其他的并发容器，如ConcurrentHashMap。
- **数据一致性问题**。CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。所以如果你希望写入的的数据，马上能读到，请不要使用CopyOnWrite容器。

### CopyOnWriteArrayList为什么并发安全且性能比Vector好

我知道Vector是增删改查方法都加了synchronized，保证同步，但是每个方法执行的时候都要去获得锁，性能就会大大下降，而CopyOnWriteArrayList 只是在增删改上加锁，但是读不加锁，在读方面的性能就好于Vector，CopyOnWriteArrayList支持读多写少的并发情况。

---



## AbstractQueuedSynchronizer

### 1. 概述

谈到并发，不得不谈ReentrantLock；而谈到ReentrantLock，不得不谈AbstractQueuedSynchronizer（AQS）！

类如其名，抽象的队列式的同步器，AQS定义了一套多线程访问共享资源的同步器框架，许多同步类实现都依赖于它，如常用的ReentrantLock/Semaphore/CountDownLatch.…..

以下是本文的目录大纲：

1. 概述
2. 框架
3. 源码详解
4. 简单应用

若有不正之处，请谅解和批评指正，不胜感激。

原文链接（原文持续更新，建议阅读原文）：http://www.cnblogs.com/waterystone/p/4920797.html

### 2. 框架

<img src="Java源码.assets/AQS同步队列.png" alt="img" style="zoom:80%;" />

它维护了一个volatile int state（代表共享资源）和一个FIFO线程等待队列（多线程争用资源被阻塞时会进入此队列）。这里volatile是核心关键词，具体volatile的语义，在此不述。state的访问方式有三种:

- getState()
- setState()
- compareAndSetState()

AQS定义两种资源共享方式：Exclusive（独占，只有一个线程能执行，如ReentrantLock）和Share（共享，多个线程可同时执行，如Semaphore/CountDownLatch）。

不同的自定义同步器争用共享资源的方式也不同。**自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可**，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。自定义同步器实现时主要实现以下几种方法：

- isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。
- tryAcquire(int)：独占方式。尝试获取资源，要求立即返回。成功则返回true，失败则返回false。
- tryRelease(int)：独占方式。尝试释放资源，要求立即成功。则返回true，失败则返回false。
- tryAcquireShared(int)：共享方式。尝试获取资源。**尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源**。
- tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire()独占该锁并将state+1。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的（state会累加），这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的。

再以CountDownLatch以例，任务分为N个子线程去执行，state也初始化为N（注意N要与线程个数一致）。这N个子线程是并行执行的，每个子线程执行完后countDown()一次，state会CAS减1。等到所有子线程都执行完后(即state=0)，会unpark()主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

一般来说，自定义同步器要么是独占方法，要么是共享方式，他们也只需实现tryAcquire-tryRelease、tryAcquireShared-tryReleaseShared中的一种即可。但AQS也支持自定义同步器同时实现独占和共享两种方式，如ReentrantReadWriteLock。

### 3. 源码

#### 3.0 结点状态（waitStatus）

这里我们说下Node。Node结点是对每一个等待获取资源的线程的封装，其包含了需要同步的线程本身及其等待状态，如是否被阻塞、是否等待唤醒、是否已经被取消等。变量waitStatus则表示当前Node结点的等待状态，共有5种取值CANCELLED、SIGNAL、CONDITION、PROPAGATE、0。

- **CANCELLED**(1)：表示当前结点已取消调度。当timeout或被中断（响应中断的情况下），会触发变更为此状态，进入该状态后的结点将不会再变化。
- **SIGNAL**(-1)：表示后继结点在等待当前结点唤醒。后继结点入队时，会将前继结点的状态更新为SIGNAL。
- **CONDITION**(-2)：表示结点等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将**从等待队列转移到同步队列中**，等待获取同步锁。
- **PROPAGATE**(-3)：共享模式下，前继结点不仅会唤醒其后继结点，同时也可能会唤醒后继的后继结点。
- **0**：新结点入队时的默认状态。

注意，**负值表示结点处于有效等待状态，而正值表示结点已被取消。所以源码中很多地方用>0、<0来判断结点的状态是否正常**。

---



#### 3.1 acquire (int)

此方法是独占模式下线程获取共享资源的顶层入口。如果获取到资源，线程直接返回，否则进入等待队列，直到获取到资源为止，且整个过程忽略中断的影响。这也正是lock()的语义，当然不仅仅只限于lock()。获取到资源后，线程就可以去执行其临界区代码了。下面是acquire()的源码：

~~~java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		// 补中断
        selfInterrupt();
}
~~~

函数流程如下：

1. tryAcquire()尝试直接去获取资源，如果成功则直接返回（这里体现了非公平锁，每个线程获取锁时会尝试直接抢占加塞一次，而CLH队列中可能还有别的线程在等待）；
2. addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；
3. acquireQueued()使线程阻塞在等待队列中获取资源，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。

这时单凭这4个抽象的函数来看流程还有点朦胧，不要紧，看完接下来的分析后，你就会明白了。



##### 3.1.1 tryAcquire(int)

此方法尝试去获取独占资源。如果获取成功，则直接返回true，否则直接返回false。这也正是tryLock()的语义，还是那句话，当然不仅仅只限于tryLock()。如下是tryAcquire()的源码：

~~~java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
~~~

什么？直接throw异常？说好的功能呢？好吧，**还记得概述里讲的AQS只是一个框架，具体资源的获取/释放方式交由自定义同步器去实现吗？**就是这里了！！！AQS这里只定义了一个接口，具体资源的获取交由自定义同步器去实现了（通过state的get/set/CAS）！！！至于能不能重入，能不能加塞，那就看具体的自定义同步器怎么去设计了！！！当然，自定义同步器在进行资源访问时要考虑线程安全的影响。

这里之所以没有定义成abstract，是因为独占模式下只用实现tryAcquire-tryRelease，而共享模式下只用实现tryAcquireShared-tryReleaseShared。如果都定义成abstract，那么每个模式也要去实现另一模式下的接口。说到底，Doug Lea还是站在咱们开发者的角度，尽量减少不必要的工作量。

这里贴上ReentrantLock中具体实现类（NonfairSync）的tryAcquire方法：

~~~java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}

final boolean nonfairTryAcquire(int acquires) {
    // 拿到当前线程
    final Thread current = Thread.currentThread();
    // 拿到同步器当前的状态
    int c = getState();
    // 为0表示还没有锁定
    if (c == 0) {
        // 那么当前线程尝试加锁。具体操作为：用CAS修改state，从0变为当前线程需要资源。独占锁每次acquires为1
        if (compareAndSetState(0, acquires)) {
            // 设置独占线程为当前线程
            setExclusiveOwnerThread(current);
            // 返回true，表示加锁成功
            return true;
        }
    }
    // 如果当前线程等于独占线程，说明当前线程在进行重入操作，那么只需要把state + acquires即可
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    // 如果前面两个if都不能进入，说明state > 0，独占资源已被其他线程上锁，那么返回false
    return false;
}
~~~



##### 3.1.2 addWaiter(Node)

此方法用于将当前线程加入到等待队列的队尾，并返回当前线程所在的结点。还是上源码吧：

~~~java
// 两个方法在AQS中是具体实现的

private Node addWaiter(Node mode) {
    // 以给定模式构造结点。mode有两种：EXCLUSIVE（独占）和SHARED（共享）
    Node node = new Node(Thread.currentThread(), mode);

    // 尝试快速方式直接放到队尾。
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }

    //上一步失败则通过enq入队。
    enq(node);
    return node;
}


private Node enq(final Node node) {
    // CAS"自旋"，直到成功加入队尾
    for (;;) {
        Node t = tail;
        // 队列为空（没有初始化，该方法还承担着同步队列初始化的任务）
        // 创建一个空的标志结点作为head结点，并将tail也指向它。
        if (t == null) { 
            // 这里有个疑问，为什么要new一个空结点当头结点，而不是直接把当前结点作为头结点呢？
            //     因为：头结点都是持有临界资源的的结点，而当前结点所代表的线程竞争临界资源失败才要插入同步队列的
            //     所以这里不能把当前结点直接作为头结点，而需要构造一个空头出来
            if (compareAndSetHead(new Node()))
                tail = head;
            
        } else {  // 同步队列不为空，那么正常插入队尾即可
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
~~~



##### 3.1.3  acquireQueued(Node, int)

OK，通过tryAcquire()和addWaiter()，该线程获取资源失败，已经被放入等待队列尾部了。聪明的你立刻应该能想到该线程下一部该干什么了吧：**进入等待状态休息，直到其他线程彻底释放资源后唤醒自己，自己再拿到资源，然后就可以去干自己想干的事了**。没错，就是这样！是不是跟医院排队拿号有点相似~~acquireQueued()就是干这件事：**在等待队列中排队拿号（中间没其它事干可以休息），直到拿到号后再返回**。这个函数非常关键，还是上源码吧：

~~~java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;//标记是否成功拿到资源
    try {
        boolean interrupted = false;//标记等待过程中是否被中断过

        //又是一个“自旋”！（这里，线程要么在休息（park），要么在尝试竞争资源，直至竞争成功，并返回）
        for (;;) {
            final Node p = node.predecessor();//拿到前驱
            //如果前驱是head，即该结点已成老二，那么就有资格去尝试获取资源（可能是老大释放完资源唤醒自己的，当然也可能被interrupt了）。
            if (p == head && tryAcquire(arg)) {
                //拿到资源后，将head指向该结点。所以head所指的标杆结点，就是当前获取到资源的那个结点或null。
                setHead(node);
                p.next = null; // setHead中node.prev已置为null，此处再将head.next置为null，就是为了方便GC回收以前的head结点。也就意味着之前拿完资源的结点出队了！
                failed = false; // 成功获取资源
                return interrupted;//返回等待过程中是否被中断过（由于整个是自旋，在下面if等待过程中，如果被中断，interrupted会被设置成true记录下来）
            }

            //如果自己可以休息了，就通过park()进入waiting状态，直到被unpark()。如果不可中断的情况下被中断了，那么会从park()中醒过来，发现拿不到资源，从而继续进入park()等待。
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;//如果等待过程中被中断过，哪怕只有那么一次，就将interrupted标记为true
        }
    } finally {
        if (failed) // 如果等待过程中没有成功获取资源（如timeout，或者可中断的情况下被中断了），那么取消结点在队列中的等待。
            cancelAcquire(node);
    }
}
~~~

到这里了，我们先不急着总结acquireQueued()的函数流程，先看看shouldParkAfterFailedAcquire()和parkAndCheckInterrupt()具体干些什么。

###### 3.1.3.1 shouldParkAfterFailedAcquire(Node, Node)

此方法主要用于检查状态，看看自己是否真的可以去休息了（进入waiting状态，如果线程状态转换不熟，可以参考本人上一篇写的[Thread详解](http://www.cnblogs.com/waterystone/p/4920007.html)），万一队列前边的线程都放弃了只是瞎站着，那也说不定，对吧！

~~~java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;//拿到前驱的状态
    if (ws == Node.SIGNAL)
        //如果已经告诉前驱拿完号后通知自己一下，那就可以安心休息了
        return true;
    if (ws > 0) {
        /*
         * 如果前驱放弃了，那就一直往前找，直到找到最近一个正常等待的状态，并排在它的后边。
         * 注意：那些放弃的结点，由于被自己“加塞”到它们前边，它们相当于形成一个无引用链，稍后就会被保安大叔赶走了(GC回收)！
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
         //如果前驱正常，那就把前驱的状态设置成SIGNAL，告诉它拿完号后通知自己一下。有可能失败，人家说不定刚刚释放完呢！
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
~~~

整个流程中，如果前驱结点的状态不是SIGNAL，那么自己就不能安心去休息，需要去找个安心的休息点，同时可以再尝试下看有没有机会轮到自己拿号。

###### 3.1.3.2 parkAndCheckInterrupt()

如果线程找好安全休息点后，那就可以安心去休息了。此方法就是让线程去休息，真正进入等待状态。

~~~java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);//调用park()使线程进入waiting状态
    return Thread.interrupted();//如果被唤醒，查看自己是不是被中断的。
}
~~~

park()会让当前线程进入waiting状态。在此状态下，有两种途径可以唤醒该线程：1）被unpark()；2）被interrupt()。需要注意的是，Thread.interrupted()会清除当前线程的中断标记位。 

###### 3.1.3.3 acquireQueued(Node, int)方法的小结

OK，看了shouldParkAfterFailedAcquire()和parkAndCheckInterrupt()，现在让我们再回到acquireQueued()，总结下该函数的具体流程：

1. 结点进入队尾后，检查状态，找到安全休息点；
2. 调用park()进入waiting状态，等待unpark()或interrupt()唤醒自己；
3. 被唤醒后，看自己是不是有资格能拿到号。如果拿到，head指向当前结点，并返回从入队到拿到号的整个过程中是否被中断过；如果没拿到，继续流程1。



##### 3.1.4 acquire小结

OKOK，acquireQueued()分析完之后，我们接下来再回到acquire()！再贴上它的源码吧：

~~~java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
~~~

再来总结下它的流程吧：

1. 调用自定义同步器的tryAcquire()尝试直接去获取资源，如果成功则直接返回；
2. 没成功，则addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；
3. acquireQueued()使线程在等待队列中休息，有机会时（轮到自己，会被unpark()）会去尝试获取资源。获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
4. 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。

由于此函数是重中之重，我再用流程图总结一下：

<img src="Java源码.assets/AQS lock图.png" style="zoom:80%;" />

---



#### 3.2 release(int)

上一小节已经把acquire()说完了，这一小节就来讲讲它的反操作release()吧。此方法是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（即state=0）,它会唤醒等待队列里的其他线程来获取资源。这也正是unlock()的语义，当然不仅仅只限于unlock()。下面是release()的源码：

~~~java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;//找到头结点
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);//唤醒等待队列里的下一个线程
        return true;
    }
    return false;
}
~~~

逻辑并不复杂。它调用tryRelease()来释放资源。有一点需要注意的是，**它是根据tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！所以自定义同步器在设计tryRelease()的时候要明确这一点！！**

**注：**根据前面解析可以知道，假设当临界资源已经被A线程lock的时候，这时候如果B线程进来竞争，就会被放到同步队列中等待。如果这个时候队列没有被初始化，会调用enq方法进行初始化，**可以得到这个队列的头结点并不是A线程所代表的结点，而是一个空dummy结点**，这里与 [2. 框架](#2. 框架)区分开来，此时的队列图应该是这样的：

<img src="Java%E6%BA%90%E7%A0%81.assets/AQS-enq%E5%90%8E%E7%9A%84%E9%98%9F%E5%88%97.png" style="zoom:50%;" />

**因为这里始终是用`Node h = head;`所以必定能找到头结点后的第一个结点B进行唤醒操作，当B被唤醒且竞争到临界资源时，B就会把自己设置为头结点，这个时候队列就变成[2. 框架](#2. 框架)中的那样了。**



##### 3.2.1 tryRelease(int)

此方法尝试去释放指定量的资源。下面是tryRelease()的源码：

~~~java
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
~~~

跟tryAcquire()一样，这个方法是需要独占模式的自定义同步器去实现的。正常来说，tryRelease()都会成功的，因为这是独占模式，该线程来释放资源，那么它肯定已经拿到独占资源了，直接减掉相应量的资源即可(state-=arg)，也不需要考虑线程安全的问题。但要注意它的返回值，上面已经提到了，**release()是根据tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！**所以自义定同步器在实现时，如果已经彻底释放资源(state=0)，要返回true，否则返回false。

这里贴上ReentrantLock中具体实现类（Sync）的tryRelease方法：

~~~java
protected final boolean tryRelease(int releases) {
    // c为释放掉资源后，该线程剩余的资源
    int c = getState() - releases;
    // 判断当前线程是不是独占指定的独占线程，不是的话就会出错（因为只有自己是指定的独占线程才有资格释放资源）
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    // 临界资源标志位
    boolean free = false;
    // 如果该线程的剩余的资源为0，说明临界资源自由，可以进行锁竞争了
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    // 设置临界资源的状态（为0，则可以被竞争）
    setState(c);
    return free;
}
~~~



##### 3.2.2 unparkSuccessor(Node)

此方法用于唤醒等待队列中下一个线程。下面是源码：

~~~java
private void unparkSuccessor(Node node) {
    //这里，node一般为当前线程所在的结点。
    int ws = node.waitStatus;
    if (ws < 0)//置零当前线程所在的结点状态，允许失败。
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;//找到下一个需要唤醒的结点s
    if (s == null || s.waitStatus > 0) {//如果为空或已取消
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev) // 从后向前找。
            if (t.waitStatus <= 0)//从这里可以看出，<=0的结点，都是还有效的结点。
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);//唤醒
}
~~~

这个函数并不复杂。一句话概括：**用unpark()唤醒等待队列中最前边的那个未放弃线程**，这里我们也用s来表示吧。此时，再和acquireQueued()联系起来，s被唤醒后，进入if (p == head && tryAcquire(arg))的判断（即使p!=head也没关系，它会再进入shouldParkAfterFailedAcquire()寻找一个安全点。这里既然s已经是等待队列中最前边的那个未放弃线程了，那么通过shouldParkAfterFailedAcquire()的调整，s也必然会跑到head的next结点，下一次自旋p==head就成立啦），然后s把自己设置成head标杆结点，表示自己已经获取到资源了，acquire()也返回了！



##### 3.2.3 release小结

release()是独占模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果彻底释放了（即state=0）,它会唤醒等待队列里的其他线程来获取资源。

74楼的朋友提了一个非常有趣的问题：如果获取锁的线程在release时异常了，没有unpark队列中的其他结点，这时队列中的其他结点会怎么办？是不是没法再被唤醒了？

答案是**YES**（测试程序详见76楼）！！！这时，队列中等待锁的线程将永远处于park状态，无法再被唤醒！！！但是我们再回头想想，获取锁的线程在什么情形下会release抛出异常呢？？

1. 线程突然死掉了？可以通过thread.stop来停止线程的执行，但该函数的执行条件要严苛的多，而且函数注明是非线程安全的，已经标明Deprecated；
2. 线程被interupt了？线程在运行态是不响应中断的，所以也不会抛出异常；
3. release代码有bug，抛出异常了？目前来看，Doug Lea的release方法还是比较健壮的，没有看出能引发异常的情形（如果有，恐怕早被用户吐槽了）。**除非自己写的tryRelease()有bug，那就没啥说的，自己写的bug只能自己含着泪去承受了**。

---



#### 3.3 acquireShared(int)

此方法是共享模式下线程获取共享资源的顶层入口。它会获取指定量的资源，获取成功则直接返回，获取失败则进入等待队列，直到获取到资源为止，整个过程忽略中断。下面是acquireShared()的源码：

~~~java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
~~~

这里tryAcquireShared()依然需要自定义同步器去实现。但是AQS已经把其返回值的语义定义好了：负值代表获取失败；0代表获取成功，但没有剩余资源；正数表示获取成功，还有剩余资源，其他线程还可以去获取。所以这里acquireShared()的流程就是：

1. tryAcquireShared()尝试获取资源，成功则直接返回；
2. 失败则通过doAcquireShared()进入等待队列，直到获取到资源为止才返回。



##### 3.3.1 doAcquireShare(int)

此方法用于将当前线程加入等待队列尾部休息，直到其他线程释放资源唤醒自己，自己成功拿到相应量的资源后才返回。下面是doAcquireShared()的源码：

~~~java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);//加入队列尾部
    boolean failed = true;//是否成功标志
    try {
        boolean interrupted = false;//等待过程中是否被中断过的标志
        for (;;) {
            final Node p = node.predecessor();//前驱
            if (p == head) {//如果到head的下一个，因为head是拿到资源的线程，此时node被唤醒，很可能是head用完资源来唤醒自己的
                int r = tryAcquireShared(arg);//尝试获取资源
                if (r >= 0) {//成功
                    setHeadAndPropagate(node, r);//将head指向自己，还有剩余资源可以再唤醒之后的线程
                    p.next = null; // help GC
                    if (interrupted)//如果等待过程中被打断过，此时将中断补上。
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }

            //判断状态，寻找安全点，进入waiting状态，等着被unpark()或interrupt()
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
~~~

有木有觉得跟acquireQueued()很相似？对，其实流程并没有太大区别。只不过这里将补中断的selfInterrupt()放到doAcquireShared()里了，而独占模式是放到acquireQueued()之外，其实都一样，不知道Doug Lea是怎么想的。

跟独占模式比，还有一点需要注意的是，这里只有线程是head.next时（“老二”），才会去尝试获取资源，有剩余的话还会唤醒之后的队友。那么问题就来了，假如老大用完后释放了5个资源，而老二需要6个，老三需要1个，老四需要2个。老大先唤醒老二，老二一看资源不够，他是把资源让给老三呢，还是不让？答案是否定的！老二会继续park()等待其他线程释放资源，也更不会去唤醒老三和老四了。独占模式，同一时刻只有一个线程去执行，这样做未尝不可；但共享模式下，多个线程是可以同时执行的，现在因为老二的资源需求量大，而把后面量小的老三和老四也都卡住了。当然，这并不是问题，只是AQS保证严格按照入队顺序唤醒罢了（保证公平，但降低了并发）。

###### 3.3.1.1 setHeadAndPropagate(Node, int)

~~~java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    setHead(node);//head指向自己
     //如果还有剩余量，继续唤醒下一个邻居线程
    if (propagate > 0 || h == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
~~~

此方法在setHead()的基础上多了一步，就是自己苏醒的同时，如果条件符合（比如还有剩余资源），还会去唤醒后继结点，毕竟是共享模式！

doReleaseShared()我们留着下一小节的releaseShared()里来讲。



##### 3.3.2 acquireShared小结

至此，acquireShared()也要告一段落了。让我们再梳理一下它的流程：

1. tryAcquireShared()尝试获取资源，成功则直接返回；
2. 失败则通过doAcquireShared()进入等待队列park()，直到被unpark()/interrupt()并成功获取到资源才返回。

整个等待过程也是忽略中断的。其实跟acquire()的流程大同小异，只不过多了个**自己拿到资源后，还会去唤醒后继队友的操作（这才是共享嘛）**。

---



#### 3.4 releaseShared()

上一小节已经把acquireShared()说完了，这一小节就来讲讲它的反操作releaseShared()吧。此方法是共享模式下线程释放共享资源的顶层入口。它会释放指定量的资源，如果成功释放且允许唤醒等待线程，它会唤醒等待队列里的其他线程来获取资源。下面是releaseShared()的源码：

~~~java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
~~~

此方法的流程也比较简单，一句话：释放掉资源后，唤醒后继。跟独占模式下的release()相似，但有一点稍微需要注意：独占模式下的tryRelease()在完全释放掉资源（state=0）后，才会返回true去唤醒其他线程，这主要是基于独占下可重入的考量；而共享模式下的releaseShared()则没有这种要求，共享模式实质就是控制一定量的线程并发执行，那么拥有资源的线程在释放掉部分资源时就可以唤醒后继等待结点。例如，资源总量是13，A（5）和B（7）分别获取到资源并发运行，C（4）来时只剩1个资源就需要等待。A在运行过程中释放掉2个资源量，然后tryReleaseShared(2)返回true唤醒C，C一看只有3个仍不够继续等待；随后B又释放2个，tryReleaseShared(2)返回true唤醒C，C一看有5个够自己用了，然后C就可以跟A和B一起运行。而ReentrantReadWriteLock读锁的tryReleaseShared()只有在完全释放掉资源（state=0）才返回true，所以自定义同步器可以根据需要决定tryReleaseShared()的返回值。

##### 3.4.1 doReleaseShared()

~~~java
private void doReleaseShared() {
    // 自旋
    for (;;) {
        // 拿到头结点，一般情况是自己（初始情况是dumy结点，我们以一般情况考虑）
        Node h = head;
        // 判断队列中还有没有结点
        if (h != null && h != tail) {
            // 拿到头结点的等待状态
            int ws = h.waitStatus;
            // 如果头结点的状态是要通知
            if (ws == Node.SIGNAL) {
                // 先用CAS把自己的等待状态设为0，默认值
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue; // CAS设置失败，那么自旋一次
                // 上面的条件都满足，唤醒后继
                unparkSuccessor(h);//唤醒后继（后继拿到资源后，如果还有资源，会继续唤醒后继）
            }
            // 如果头结点的等待状态为0，不是通知类型，那么要把头结点的等待状态设置为PROPAGATE（这个状态好像没多大用）
            // 不是很懂这个状态
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;
        }
        // head没有发生变化就跳出循环，发生变化（只有有线程被唤醒且抢到了资源，头结点会变化）那么继续资源唤醒后面的结点
        if (h == head)
            break;
    }
}
~~~



#### 3.5 小结

本节我们详解了独占和共享两种模式下获取-释放资源(acquire-release、acquireShared-releaseShared)的源码，相信大家都有一定认识了。值得注意的是，acquire()和acquireShared()两种方法下，线程在等待队列中都是忽略中断的。AQS也支持响应中断的，acquireInterruptibly()/acquireSharedInterruptibly()即是，相应的源码跟acquire()和acquireShared()差不多，这里就不再详解了。

#### 4.应用

下面我们就以AQS源码里的Mutex为例，讲一下AQS的简单应用。还有其他一些应用（Semaphore、CountdownLatch、CyclicBarrier）

##### 4.1 Mutex

Mutex是一个不可重入的互斥锁实现。锁资源（AQS里的state）只有两种状态：0表示未锁定，1表示锁定。下边是Mutex的核心源码：

~~~java
class Mutex implements Lock, java.io.Serializable {
    // 自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {
        // 判断是否锁定状态
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        // 尝试获取资源，立即返回。成功则返回true，否则false。
        public boolean tryAcquire(int acquires) {
            assert acquires == 1; // 这里限定只能为1个量
            if (compareAndSetState(0, 1)) {//state为0才设置为1，不可重入！
                setExclusiveOwnerThread(Thread.currentThread());//设置为当前线程独占资源
                return true;
            }
            return false;
        }

        // 尝试释放资源，立即返回。成功则为true，否则false。
        protected boolean tryRelease(int releases) {
            assert releases == 1; // 限定为1个量
            if (getState() == 0)//既然来释放，那肯定就是已占有状态了。只是为了保险，多层判断！
                throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);//释放资源，放弃占有状态
            return true;
        }
    }

    // 真正同步类的实现都依赖继承于AQS的自定义同步器！
    private final Sync sync = new Sync();

    //lock<-->acquire。两者语义一样：获取资源，即便等待，直到成功才返回。
    public void lock() {
        sync.acquire(1);
    }

    //tryLock<-->tryAcquire。两者语义一样：尝试获取资源，要求立即返回。成功则为true，失败则为false。
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    //unlock<-->release。两者语文一样：释放资源。
    public void unlock() {
        sync.release(1);
    }

    //锁是否占有状态
    public boolean isLocked() {
        return sync.isHeldExclusively();
    }
}
~~~

同步类在实现时一般都将自定义同步器（sync）定义为内部类，供自己使用；而同步类自己（Mutex）则实现某个接口，对外服务。当然，接口的实现要直接依赖sync，它们在语义上也存在某种对应关系！！而sync只用实现资源state的获取-释放方式tryAcquire-tryRelelase，至于线程的排队、等待、唤醒等，上层的AQS都已经实现好了，我们不用关心。

除了Mutex，ReentrantLock/CountDownLatch/Semphore这些同步类的实现方式都差不多，不同的地方就在获取-释放资源的方式tryAcquire-tryRelelase。掌握了这点，AQS的核心便被攻破了！

## Condition

### 1.概述

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

### 2. 框架



<img src="Java%E6%BA%90%E7%A0%81.assets/Condition.png" style="zoom:40%;" />

每把锁可以拥有多个Condition等待队列。

### 3. 源码

因为Condition是个借口，这里我们以AQS中的内部类ConditionObject类来讲解，ConditionObject实现了Condition接口

#### 3.1 await()

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

##### 3.1.1 addConditionWaiter()

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

###### 3.1.1.1 unlinkCancelledWaiters()

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

##### 3.1.2 fullyRelease(Node)

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

##### 3.1.3 isOnSyncQueue(Node)

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

###### 3.1.3.1 findNodeFromTail(Node)

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

###### 3.1.3.2  isOnSyncQueue(Node)总结

node从Condition队列转移到同步队列：

1. 第一步，就是设置waitStatus为其他值，因此是否等于Node.CONDITON可以作为判断标志，如果等于，说明还在Condition队列中，即不再Sync队列里。在node被放入Sync队列时，第一步就是设置node的prev为当前获取到的尾节点，所以如果发现node的prev为null的话，可以确定node尚未被加入Sync队列。

2. 相似的，node被放入Sync队列的最后一步是设置node的next，如果发现node的next不为null，说明已经完成了放入Sync队列的过程，因此可以返回true。

3. 当我们执行完两个if而仍未返回时，node的prev一定不为null，next一定为null，这个时候可以认为node正处于放入Sync队列的执行CAS操作执行过程中（enq方法）。而这个CAS操作有可能失败，因此我们再给node一次机会，调用findNodeFromTail来检测
4. findNodeFromTail方法从尾部遍历Sync队列，如果检查node是否在队列中，如果还不在，此时node也许在CAS自旋中，在不久的将来可能会进到Sync队列里。但我们已经等不了了，直接返回false。那在这个线程就不能跳出while循环，只能继续park，等待同步队列的前置结点唤醒它。

复杂一点的就是这个checkInterruptWhileWaiting等待过程中的中断检测逻辑，我们先来看一下。

##### 3.1.4 checkInterruptWhileWaiting(Node)

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

###### 3.1.4.1 transferAfterCancelledWait(Node)

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

###### 3.1.4.2 checkInterruptWhileWaiting(Node)总结

这个方法检测有无中断发生：

1. 无中断发生最好，返回0
2. 有中断发生，调用transferAfterCancelledWait判断中断在什么时机发生
   - 等待过程中发生：返回抛出标记-1
   - signal过程发生：返回自行处理中断标记1

##### 3.1.5 reportInterruptAfterWait(int)

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

##### 3.1.6 await()总结

1. 判断当前有无中断，因为已经进入await语义，有的话直接抛出
2. 通过当前线程构建一个node，放入等待队列
3. 释放自己所占用的资源
4. 进入等待状态，线程休息
5. 被唤醒时，判断唤醒自己的是signal还是中断，进入同步队列
6. 竞争资源，如果有中断，则进行相应的处理

---



#### 3.2 signal()

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

##### 3.2.1 isHeldExclusively()

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

##### 3.2.2 doSignal(Node)

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

###### 3.2.2.1 transferForSignal(Node)

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

##### 3.2.3 signal总结

这个方法的工作流程如下：

1. 在等待队列中找到第一个有效等待的结点，并把它放入同步队列尾
2. 判断一下这个结点的前置结点是否取消，或者设置前置结点唤醒后面结点是否成功
   - 如果前置结点取消了，那么直接叫醒当前线程；如果设置前置结点唤醒服务失败，也直接唤醒当前线程
   - 如果上面两条都是正常的，那么说明前置结点代表的线程会在释放锁的时候唤醒自己，所以自己就老老实实在同步队列尾等着就好了

#### 3.4 总结

Condition等待队列中的线程与AQS中同步队列中的线程在等待的时候都是被park的，本质是一样的，不同的是Condition等待队列中的线程必须要被signal，才能转移到同步队列中，以获得竞争锁的资格。

---



## Future、Callable

### 1. 概述

创建线程的2种方式，一种是直接继承Thread（Thread也继承了Runnable接口），另外一种就是实现Runnable接口。这2种方式都有一个缺陷就是：在执行完任务之后无法获取执行结果。如果需要获取执行结果，就必须通过共享变量或者使用线程通信的方式来达到效果，这样使用起来就比较麻烦。而自从Java 1.5开始，就提供了Callable和Future，通过它们可以在任务执行完毕之后得到任务执行结果。

今天我们就来讨论一下Callable、Future和FutureTask三个类的使用方法。以下是本文的目录大纲：

1. 概述
2. 框架
3. 源码

### 2. 框架

<img src="Java%E6%BA%90%E7%A0%81.assets/FutureTask.png" style="zoom:70%;" />

#### 2.1 Future接口

可以看到Future是一个顶层的规范接口，其中规定了5个方法：

- **cancel**：用来取消任务，如果取消任务成功则返回true，如果取消任务失败则返回false。参数mayInterruptIfRunning表示是否允许取消正在执行却没有执行完毕的任务，如果设置true，则表示可以取消正在执行过程中的任务。如果任务已经完成，则无论mayInterruptIfRunning为true还是false，此方法肯定返回false，即如果取消已经完成的任务会返回false；如果任务正在执行，若mayInterruptIfRunning设置为true，则返回true，若mayInterruptIfRunning设置为false，则返回false；如果任务还没有执行，则无论mayInterruptIfRunning为true还是false，肯定返回true。
- **isCancelled**：表示任务是否被取消成功，如果在任务正常完成前被取消成功，则返回 true。
- **isDone**：表示任务是否已经完成，若任务完成，则返回true；
- **get()**：用来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回；
- **get(long timeout, TimeUnit unit)**：用来获取执行结果，如果在指定时间内，还没获取到结果，就直接返回null。

#### 2.2 RunnableFuture接口

RunnableFuture继承了Future和Runnable两个接口，也就是说RunnableFuture的子类既可以作为线程的执行任务，又可以拿到这个任务的返回值，从而实现得到返回值。

#### 2.3 FutureTask类

FutureTask是Future体系的一个具体实现类，实现了Future接口中的所有方法，也是实现了Runnable中的run方法。由这个体系，我们可以知道，

1. 我们可以构造一个FutureTask对象，然后把这个task放到new Thread( task ).start()中，等到线程启动之后，就会执行这个task中的run方法。
2. 既然run方法已经被这个类重写了，那岂不是run方法已经是固定的，那怎么执行我们自己的个性化任务呢？秘密在，这个run方法中，run方法又调用了Callable接口的call方法，所以在构造task的时候，需要传一个Callable实现类对象（其实也可以传递一个Runnable，但是这个Runnable也会被封装成为Callable）
3. 我们知道Callable的中call方法是有返回值的，所以FutureTask封装了一个结果成员变量，在调用call方法的时候，把返回值给哦这个结果成员变量，我们就能用get方法拿到这个结果了。

流程图如下：

<img src="Java%E6%BA%90%E7%A0%81.assets/FutureTask%E6%B5%81%E7%A8%8B.png" style="zoom:40%;" />

##### 2.3.1 FutureTask的生命周期（状态state）

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


### 3. 源码

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

#### 3.1 FutureTask(Callable<T>)

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

#### 3.2 run()

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

##### 3.2.1 setException(Throwable)

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

###### 3.2.1.1 finishCompletion()

~~~java
private void finishCompletion() {
    // assert state > COMPLETING;
    // 拿到单链表中第一个等待的线程结点
    for (WaitNode q; (q = waiters) != null;) {
        // 拿到结点中，把其置空，不让其他线程拿
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

##### 3.2.2 set(V)

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

##### 3.2.3 handlePossibleCancellationInterrupt(int)

~~~java
private void handlePossibleCancellationInterrupt(int s) {
	// 我们的run方法没有执行完时候，别的线程正在产生中断
    if (s == INTERRUPTING)
       	// 那么我们就自旋，等待中断产生完全
        while (state == INTERRUPTING)
            Thread.yield(); // wait out pending interrupt

}
~~~

###### 3.2.3 run方法总结

通过以上的分析，我们可以总结出run方法的运行流程

1. 首先判断state是不是NEW，是NEW的话，就设置好runner，就可以执行了
2. 调用Callable的call方法
3. 如果call方法出现异常，那么就把返回结果设置为异常，state设为EXCEPTIONAL；如果正常执行完，那么就把返回结果设置为正常的结果，并改变state到NORMAL
4. 调用扫尾finishCompletion方法，唤醒等待单链上的所有等待线程

**如果下次再调用这个FutureTask的run方法，由于state已经改变，所以并不会真正执行run的方法体，而是直接返回**

#### 3.3 get()

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

##### 3.3.1 awaitDone(boolean, long)

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

##### 3.3.2 report(int)

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

##### 3.3.3 get方法总结

这个方法也不难，比较特别的就是：当结果还没跑出来的时候，取结果的线程会阻塞，等待结果出来。而且可以有很多线程都来取结果。

---



## 线程池

### 1. 概述

在前面的文章中，我们使用线程的时候就去创建一个线程，这样实现起来非常简便，但是就会有一个问题：

如果并发的线程数量很多，并且每个线程都是执行一个时间很短的任务就结束了，这样频繁创建线程就会大大降低系统的效率，因为频繁创建线程和销毁线程需要时间。

那么有没有一种办法使得线程可以复用，就是执行完一个任务，并不被销毁，而是可以继续执行其他的任务？

在Java中可以通过线程池来达到这样的效果。今天我们就来详细讲解一下Java的线程池。

使用线程池有如下几点优势：

1. 降低系统资源消耗，通过重用已存在的线程，降低线程创建和销毁造成的消耗；
2. 提高系统响应速度，当有任务到达时，通过复用已存在的线程，无需等待新线程的创建便能立即执行；
3. 方便线程并发数的管控。因为线程若是无限制的创建，可能会导致内存占用过多而产生OOM，并且会造成cpu过度切换（cpu切换线程是有时间成本的（需要保持当前执行线程的现场，并恢复要执行线程的现场））。
4. 提供更强大的功能，延时定时线程池。

以下是本文大纲

1. 概述
2. 框架
3. 源码

### 2. 框架

线程池类的组织架构如图所示：

<img src="Java%E6%BA%90%E7%A0%81.assets/Executor.png" style="zoom:60%;" />

>箭头含义：
>
>绿色实线：接口间的继承
>绿色虚线：类实现接口
>蓝色实线：类继承

其中ThreadPoolExecutor是最常用、最核心的线程池类，ForkJoinPool线程池在Stream并行操作时候会用到（跳转[Fork/Join框架](#Fork/Join框架)）。在学习ThreadPoolExeutor之前，我们先来介绍一些ThreadPoolExecutor中的核心成员变量：

#### 2.1 成员变量

~~~java
private static final int COUNT_BITS = Integer.SIZE - 3;     // 29
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;// 2^29 - 1 000,11111111111111111111111111111
~~~

通过变量名字，可以很直接看出来：COUNT_BITS：记录线程数量所使用的bit数，29位bit，那么CAPACITY理所应当就是2^29 - 1了。

既然线程数量占29位，那一个int型还有3位是干嘛呢？当然就是用来记录线程的生命周期啦

##### 2.1.1 线程池生命周期状态

就如同线程的生命周期一样，线程池也有属于它自己的生命周期：（这些状态都会左移29位，到最高3位上面去）

```java
private static final int RUNNING    = -1 << COUNT_BITS; // -536870912  111,00000000000000000000000000000
private static final int SHUTDOWN   =  0 << COUNT_BITS; // 0           000,00000000000000000000000000000
private static final int STOP       =  1 << COUNT_BITS; // 536870912   100,00000000000000000000000000000
private static final int TIDYING    =  2 << COUNT_BITS; // 1073741824  010,00000000000000000000000000000
private static final int TERMINATED =  3 << COUNT_BITS; // 1610612736  011,00000000000000000000000000000
```

- **RUNNING**：创建线程后，初始时就处于此状态
- **SHUTDOWN**：如果调用了shutdown()方法，则线程池处于SHUTDOWN状态，此时线程池不能够接受新的任务，它会等待所有任务执行完毕
- **STOP**：如果调用了shutdownNow()方法，则线程池处于STOP状态，此时线程池不能接受新的任务，并且会去尝试终止正在执行的任务
- **TIDYING**：
  - 线程池在 SHUTDOWN 状态，任务队列为空且执行中任务为空，线程池就会由 SHUTDOWN 转变为 TIDYING 状态。并执行钩子方法 terminated() 方法。 terminated() 方法是空实现，可以重写该方法进行相应的处理。
  - 线程池在 STOP 状态，线程池中执行中任务为空时，就会由 STOP 转变为 TIDYING 状态。
- **TERMINATED**：线程池彻底终止，状态就会变为此状态。在TIDYING状态执行完钩子函数terminated()，状态也会变为此状态

可以看到**正常运行状态就只有RUNNING，是个正数，所以下面也有很多是直接用 >0 、<0来判断线程池是不是RUNNING状态**，其中 isRunning方法，就是判断ctl是否小于SHUTDOWN来判断线程池是否正在运行。

既然生命周期和数量，分别占，高3位和低29位，那么也就是一个int型参数就可以将这两个信息记录下来，这也就是下面这个成员变量的作用：

~~~java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
~~~

ctl是一个原子整型变量，所以是线程安全的。可以看到在new对象的时候，这个变量已经初始化为RUNNING和0个线程额组合了。那么我们就得看一下ctlOf等有关这些成员变量设置、取值的方法：

#### 2.2 成员变量的有关方法

~~~java
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
~~~

很简单的三个方法

- **runStateOf**：把ctl拿过来& ~CAPACITY（CAPACITY取反，正好高3位为1，低29位为0，作为生命周期的掩码），得到生命周期的值
- **workerCountOf**：把ctl拿过来&CAPACITY（CAPACITY高3位为0，低29位为1，作为线程数量的掩码），得到线程数
- **ctlOf**：直接把 生命周期状态 | 线程数，就得到了这个值

### 3. 源码

以用例驱动源码的学习，我们首先来看一个线程池的简单demo：

~~~java
ExecutorService service = Executors.newFixedThreadPool(5);
for (int i = 0; i < 5; i++){
    System.out.println(service.submit(()-> 1).get());
}
service.shutdown();
~~~

可以看到，demo非常简单，就是用线程工具类Executors得到到一个线程池，向上转型返回给ExecutorService，并且向线程池提交了5个任务。这个任务可以是Runnable、或者Callable类型。使用完线程池之后，要调用shutdown方法，关闭线程池资源。

首先，我们来看一下，Executors类是怎么构造线程池对象的：

#### 3.1 Executors.newFixedThreadPool(int)

~~~java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
~~~

可以看到，这个工具类的这个方法，就是简单的new了一个ThreadPoolExecutor对象，并封装了一下默认参数，这个方法只是简化了使用者构造线程池过程。阿里巴巴开发开发手册不允许用这种方式创建线程池，因为屏蔽了细节，容易导致OOM，具体参见阿里巴巴开发手册

那么，我们就看一下这个核心的构造方法吧：

#### 3.2 ThreadPoolExecutor构造方法

~~~java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    // 先省略方法体
}
~~~

可以看到这个构造方法有7个参数，我们挨条来说明：

- **corePoolSize**：核心池的数量。
- **maximumPoolSize**：池的最大数量。
- **keepAliveTime**：保活时间。
- **unit**：保活时间的单位。TimeUnit是一个枚举类型，有7个值，对应7个时间单位。
- **workQueue**：任务缓存队列。这个队列是一个阻塞队列类型，用于存放任务。
- **threadFactory**：线程工厂。这个工厂用来创建线程，在Executors类中，有这个工厂的默认实现。
- **handler**：拒绝策略句柄。拒绝策略有四种，任务提交速度过快，任务缓存队列装不下了，那么就需要使用拒绝策略，因为来不及处理这些过多的任务，看需要怎么把这些多的任务处理掉。ThreadPoolExecutor类中已经实现好了有如下4大拒绝策略：
  - AbortPolicy：丢弃任务并抛出RejectedExecutionException异常。 
  - DiscardPolicy：也是丢弃任务，但是不抛出异常。 
  - DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
  - CallerRunsPolicy：由调用线程处理该任务 

我们通过一个具体的例子，来加深对这个几个参数的理解：

1. 假设这里刚成立了一个**国企（线程池）**，国企里面还没有**员工（线程）**，只有一个CEO（调用线程）。国企成立之后，来了一些任务，这个时候CEO就需要招一些员工来处理任务，因为是国企，没有编制谁来啊是吧。所以就需要给这些员工编制，**这些有编制内的员工就是核心线程，而编制的数量，就是上面的corePoolSize**。

   **注意**：这里有个小细节，就是当有新任务来了之后，只要还有编制，就会招新员工来处理这个任务，而不管老员工是不是有空闲的。这是为什么呢？因为企业刚成立嘛，需要快速扩张，所以要尽快把编制名额招满。

2. 慢慢地，企业越做越大，有编制的员工都有任务忙着在，但是还是有任务源源不断地进来，这个时候就需要把任务放到仓库里存着，等这些员工处理完手头任务再接着处理。**仓库就是workQueue**。那么问题又来了，仓库存不下了怎么办？处理方法是这样的：仓库满了，再每来一个任务，就再招一个工人来处理这些任务。这个时候呢，员工人数就超过了编制数，怎么办呢？看谁**休息的时间超过keepAliveTime，就裁掉谁**，直到员工又有任务或者人数回到编制数内。

   **注意**：这里有个小细节，线程与线程之间是没有区别的，只要线程数小于corePoolSize，这些线程就都是核心线程，默认是不会裁掉的。只有当线程数大于corePoolSize了，才会判断每个线程空闲时间是否大于keepAliveTime，是的话才裁掉。

3. 当任务还是太多了，任务缓存队列放不下，招的工人人数已经到了**企业所能容纳的最大人数（maximumPoolSize）**，那再新进来的任务就没法处理，所以企业就会采取**原先规定好的策略（拒绝策略）**处理这些任务——或丢弃、或不管、或由CEO来干、或丢弃取出任务缓存队列头的任务，把新任务入队。

#### 3.3 execute方法

线程池创建完了之后，就需要提交任务给线程池去执行。主要有两种提交任务的方法：

- **execute**：实现自Executor接口，接受Runnable类型的参数，没有返回值
- **submit**：继承自AbstractExecutorService类，有默认实现。可以接受Runnable、Callable类型的参数，有返回值

由上一节的分析可知，接收的Callable类型的参数，要封装成为RunnableFuture，变成Runnable类型。所以submit最后肯定也会调用execute方法。如下：

~~~java
// AbstractExecutorService中的submit方法
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    // 将Callable封装为RunnableFuture任务
    RunnableFuture<T> ftask = newTaskFor(task);
    // 调用execute执行
    execute(ftask);
    return ftask;
}
~~~

所以提交方法的核心就是execute方法。那么我们来分析一下execute方法的源码

~~~java
public void execute(Runnable command) {
    // 检查任务是否为空
    if (command == null)
        throw new NullPointerException();
    // 先拿到ctl
    int c = ctl.get();
    // 判断当前线程数是否小于核心线程数
    if (workerCountOf(c) < corePoolSize) {
        // 小于的话，不管有没有线程空闲，都会新增一个线程处理任务
        if (addWorker(command, true))
            // 新增成功直接返回
            return;
        // 新增失败，那就再拿一下ctl
        c = ctl.get();
    }
    // 线程数大于核心线程数，或者添加核心线程失败
    // 先判断线程池是否还在运行，是的话，那么就需要把任务存任务队列里去
    if (isRunning(c) && workQueue.offer(command)) {
        // 任务存放成功，再拿一下ctl，再检查一遍生命周期状态
        int recheck = ctl.get();
        // 如果线程池不是运行态，那么就需要把存起来的任务拿出来
        if (! isRunning(recheck) && remove(command))
            // 并对这个任务执行拒绝策略
            reject(command);
        // 走到这，说明线程池运行正常，任务也放进队列了，判断一下线程池里有没有线程
        else if (workerCountOf(recheck) == 0)
            // 没有的话就添加一个线程，不给任务，等它自己去队列里取
            addWorker(null, false);
    }
    // 走到这说明线程池没有运行了，或者队列满了，添加任务失败
    // 如果线程池没有运行，添加线程必然失败，会对该任务执行拒绝策略。
    // 如果线程在运行，线程数量还没有大于最大线程数，添加线程会成功；如果大于，也会添加失败，执行拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}
~~~

上面每一个if都在检查线程的状态，并作出相应的处理。我们看完下面的方法之后，再来总结这个方法

##### 3.3.1 addWorker(Runnable, boolean)

添加线程，并启动的方法：

~~~java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:  // go-to语句
    // 自旋
    for (;;) {
        // 拿到ctl
        int c = ctl.get();
        // 拿到线程状态
        int rs = runStateOf(c);
		
       	// 1. 这个if的有点复杂，我们在下面分析。这里先知道，如果线程的状态不正常了，那么就返回false 
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;
        
		// 进入内层自旋
        for (;;) {
            // 拿到线程数
            int wc = workerCountOf(c);
            // 如果线程数线程大于总容量；或者如果添加的是核心/非核心线程，线程数如果大于核心线程数/总线程数，返回false；
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 一切正常，那么就用CAS把线程数+1
            if (compareAndIncrementWorkerCount(c))
                // +1成功，跳出两层自旋
                break retry;
            // 线程数+1失败，说明有其他线程修改了这个数据，那么再拿一次
            c = ctl.get();  
            
            // 检查一下状态
            //    状态改变：进行外层自旋，看是否满足上面的1. if返回；
            //    状态没改变：进行内层自旋，再尝试CAS把线程数+1
            if (runStateOf(c) != rs)
                // 注意喔：因为retry在外层循环外面，这里的continue，会跳到外层循环继续
                continue retry;
        }
    }
	
    // 走到这，说明一切正常。线程数也+1了，就该真正创建一个线程了
    
    // 起两个标志位，线程是否启动；线程是否添加
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 我们说的创建一个线程，其实首先是new一个Worker，在构造Worker时，封装传进来的任务以及用工厂创建一个线程
        // 创建线程的时候，又把这个worker当参数，传递给了这个线程，因为Worker实现了Runnable接口。
        // 有点绕，等会咱在Worker的构造方法中好好掰扯掰扯
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            // 先上锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 持有了锁之后，再检查一下线程池状态
                int rs = runStateOf(ctl.get());
				// 如果运行状态正常，就进入if把worker加入工人队列保存
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // 这个线程是刚创建的，还没有alive
                    if (t.isAlive()) 
                        // 如果是alive，这里要抛出异常
                        throw new IllegalThreadStateException();
                    // 把worker加入工人队列
                    workers.add(w);
                    int s = workers.size();
                    // 记录下线程池最大的线程数量
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    // 线程已添加标志置true
                    workerAdded = true;
                }
            } finally {
                // 添加完线程，释放锁
                mainLock.unlock();
            }
            // 如果添加了线程
            if (workerAdded) {
                // 启动线程
                t.start();
                // 线程启动置true
                workerStarted = true;
            }
        }
    } finally {
        // 如果线程没有启动起来
        if (! workerStarted)
            // 调用相应的处理方法。就是把这个worker从工人队列再拿出来，并且改变线程池状态
            addWorkerFailed(w);
    }
    // 返回线程是否启动
    return workerStarted;
}
~~~

这个方法很长，但是它主要就是做了两件事：

1. CAS改变ctl，具体来说是把线程数+1
2. new 一个新的Worker，把它添加到工人队列，并启动它封装的线程，执行任务

我们来看一下源码里面那个复杂的1. if

~~~java
if (rs >= SHUTDOWN && ! (rs == SHUTDOWN && firstTask == null && ! workQueue.isEmpty()))
    return false;
~~~

可以看到是一个&&条件，想要进入内部，只有当rs >= SHUTDOWN、线程状态不正常了才会判断后面的逻辑：

线程池状态不正常  &&  ：

1. 线程池状态不为SHUTDOWN（假如为STOP）：根据STOP语义：停止接受任务，不创建新的线程，尝试结束正在执行的任务，所以返回false，结合execute方法，就会对这个新来任务执行拒绝策略
2. 线程状态为SHUTDOWN，那么如果firstTask不为null：线程池状态已经为SHUTDOWN，不接受新任务，而新任务又不为空，就返回false，结合execute方法，对这个新来的任务执行拒绝策略
3. 线程状态为SHUTDOWN，firstTask为null，可是任务缓存队列不为null：线程池已经为SHUTDOWN，不接受新任务，新任务为空，但是任务队列不为空，返回false，结合execute，对这个空任务执行拒绝策略，并等待任务队列中的任务执行完毕

##### 3.3.2 execute方法总结

看完了上面的分析，我们基本上可以总结一下execute的流程了

1. 如果线程池中的线程数少于核心线程数，那么就直接new一个worker，让他的线程来跑这个任务，这些线程在run完后，都会在阻塞队列处阻塞
2. 如果线程池中的线程数不小于核心线程数了，那么就把这个任务入队列，等线程们干完手头的活再去任务阻塞队列取
3. 如果任务队列满了，那么当线程数不超过池的最大线程数时，会new新的worker，让他的线程来跑
4. 在1. 2. 3. 三步中，new worker之前和入队列之后都要检查线程池状态是否正常，如果不正常，或者任务队列满了，就会执行拒绝策略，拒绝掉这个任务

#### 3.4 Worker类

Worker是ThreadPoolExecutor的内部类，是持有线程的类，是整个线程池的结构中比较重要的一个类，我们来看一下这个类主要的源码：

~~~java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable{
    
    // 持有的线程成员变量
    final Thread thread;
    
    // 记录线程的第一个任务
    Runnable firstTask;
    
    // 记录这个worker总共完成的任务数
    volatile long completedTasks;

    // 构造方法，封装了第一个任务和线程。因为继承了AQS，把state也初始化了一下
    Worker(Runnable firstTask) {
        setState(-1); 
        this.firstTask = firstTask;
        // 通过线程工厂拿到线程（重点）
        this.thread = getThreadFactory().newThread(this);
    }

    // 重写的Runnable的run方法
    public void run() {
        runWorker(this);
    }
}
~~~

我们可以看到，在通过线程工厂创建线程的时候，需要一个Runnable参数。线程启动后，就去执行Runnable里面的run方法，也就是我们的任务啦？但是这个地方为什么不是直接将我们的任务传递进去，而是把自己这个worker自己本身传进去了呢？为什么要搞得这么绕呢？我们来分析一下这个地方的的逻辑。

1. 首先，我们们来看一下，如果直接把任务传递给线程会出现什么情况。当线程启动的时候，就会调用任务的run方法，执行run方法里面的具体内容。执行完了之后，如果不加处理，这个线程就死掉了，那么就违背了使用线程池的初衷。如果要加处理，那么就只能任务提供者，也就是线程池调用者在run里面写上线程执行完了怎么去取任务、怎么阻塞的逻辑，既然这些任务都由调用者做了，那么线程池还有什么意义呢，是吧？
2. 所以worker不能直接把任务传递给线程。而是要有一个类，继承了Runnable接口，重写run方法，在这个run方法规定好，线程怎么取任务、取完任务再去调用任务的run方法、没有任务怎么阻塞的逻辑，把这个封装好的Runnable传递给线程就OK了。既然需要一个额外的类，那么Java的设计者们，索性就直接用Worker自己去继承了Runnable，重写run方法，然后把自己传递给了线程。
3. 前面分析的挺有道理的，但是我们看到Worker的run方法只是简单的调用了一个外部类的runWorker方法，如果我们前面的分析是正确的的话，那么在2.中提到的逻辑肯定被封装到了这个runWorker方法中。事实上也正是如此。那么又一个问题来了，为什么不直接在Worker的run方法中封装这些逻辑呢？
4. 这就涉及到线程池中的线程是如何阻塞的了。线程池的线程数在少于核心线程数时，默认情况下都不会死掉，而是阻塞任务到来。阻塞等待任务到来，看到这句话会不会想到什么？没错，就是阻塞队列了。每一个线程在被新建的时候，都一个firstTask，当他执行完这个任务之后，就会不断地去阻塞队列中取任务。阻塞队列：没有任务的时候，会阻塞取任务的线程；任务满的时候，会阻塞放任务的线程。根据阻塞队列的这个特性，可想而知，那些核心线程在没有任务的时候，都是被阻塞在取任务的路上了。因为阻塞队列是线程池的东西，虽然内部类可以访问到，但是面向对象是很讲究这个逻辑的，所以斗胆猜测一下，因为这个原因，要把逻辑封装到线程池的方法runWorker中。

##### 3.4.1 runWorker(Runnable)

书接上文，既然我们分析了runWoker方法封装了线程取任务、阻塞的逻辑，那么我们就来看一下，这个runWorker具体是怎么做的

~~~java
final void runWorker(Worker w) {
    // 拿到当前线程
    Thread wt = Thread.currentThread();
    // 拿到任务
    Runnable task = w.firstTask;
    // 将指针置null，方便GC
    w.firstTask = null;
    // 这个地方没细研究
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 先执行firstTask，再循环拿任务
        while (task != null || (task = getTask()) != null) {
            // worker上锁，应该是怕同一个任务，被几个线程执行
            w.lock();
            // 如果线程池正在stop，确保线程被中断
            // 如果线程池正常，确保线程不被中断
            // 需要双重检查
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                // 钩子方法，可以由子类重写
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 执行任务的run方法，也就是我们自己的逻辑
                    // 如果task是RunnableFuture，那么执行完这个，task里面封装的outcome就有结果了
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 钩子方法
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                // 任务完成数+1
                w.completedTasks++;
                w.unlock();
            }
        }// 注意嗷，上面是在循环里面，执行完了firstTask，该线程就会循环去阻塞队列拿任务了
        completedAbruptly = false;
    } finally {
        // 线程跳出了循环，执行这个，线程退出。因为跳出了循环，说明线程池要关闭了，线程就要退出。
        processWorkerExit(w, completedAbruptly);
    }
}
~~~

接下来，我们看一下取任务的方法

###### 3.4.1.1 getTask()

~~~java
// 这方法里面没有上锁，可能会被并发执行
private Runnable getTask() {
    // 上一次取任务是否超时标记（自己或者其他线程），为什么会超时呢，因为线程多了，任务少了，有的线程在规定时间取不到任务
    boolean timedOut = false; // Did the last poll() time out?
    
    // 自旋
    for (;;) {
        int c = ctl.get();
        // 拿到线程池的状态
        int rs = runStateOf(c);

        // 如果线程池状态停止了，或者是shutdown并且任务队列为空
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            // 那么就要减少worker数
            decrementWorkerCount();
            // 并且返回null，让他跳出循环，退出池
            return null;
        }

        // 线程池状态正常，拿到worker数量
        int wc = workerCountOf(c);

        // 线程是否超时等待，如果设置了allowCoreThreadTimeOut或者线程数大于核心线程线程数，那么就要超时等待
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
		// 这两个条件都表示线程过多
        if ((wc > maximumPoolSize || (timed && timedOut))
            // 这两个条件都表示任务过少
            && (wc > 1 || workQueue.isEmpty())) {
            // 线程过多、并且任务过少，就要减少线程计数器数
            if (compareAndDecrementWorkerCount(c))
                // 返回null，让当前线程退出循环，退出池
                return null;
            continue;
        }

        try {
            // 真正取任务，线程会阻塞在这
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
            workQueue.take();
            if (r != null)
                return r;
            // 任务为空，说明取任务超时了，标记置true
            timedOut = true;
        } catch (InterruptedException retry) {
            // 有异常置false，因为不是取任务超时，而是被中断打断了
            timedOut = false;
        }
    }
}
~~~

###### 3.4.1.2 processWorkerExit(Worker， boolean)

~~~java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    // 上锁
    mainLock.lock();
    try {
        // 把当前线程完成的任务数加到总完成任务数上
        completedTaskCount += w.completedTasks;
        // 工人队列移除工人
        workers.remove(w);
    } finally {
        // 释放锁
        mainLock.unlock();
    }
		
   	// 尝试终止线程池，如果池状态正常，直接返回，不做任何事
    tryTerminate();
	
    int c = ctl.get();
    // 如果池状态正常
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            // 看一下线程池允许最小线程数，如果允许核心线程死亡，那么0；如果不允许，那么核心线程数
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            // 如果最小是0， 但是任务队列还有任务，那么最小至少为1，因为要把任务都执行完，才能推迟
            if (min == 0 && ! workQueue.isEmpty())
                // 设置最小为1
                min = 1;
            // 如果当前线程数 >= min
            if (workerCountOf(c) >= min)
                // 直接反回了
                return; // replacement not needed
        }
        // 小于的话要添加1个线程
        addWorker(null, false);
    }
}
~~~

#### 3.5 总结

至此呢，线程池的基本脉络就梳理清楚了，还有一些线程池状态啊、中断啊等等细节的问题就不赘述了，根据这个脉络，再去看源码就很清晰了。下一节，我们接着整Fork/Join框架。

---



## Fork/Join框架

### 1. 概述

最近在看Fork/Join框架，说实话有点难。ForkJoinPool在1.7引入，它只被用来运行ForkJoinTask的子类任务。这个线程池和其他的线程池的不同之处在于它使用分而治之和工作窃取算法去执行任务。有效的去处理大多数任务能衍生出小任务的问题。笔者也是刚接触ForkJoinPool，这个类比较复杂如果有错误还望指正。

### 2. 框架

Fork/Join框架在Java中的具体实现是ForkJoinPool。我们来看一下在ForkJoinPool中一个核心的算法：工作窃取算法

#### 2.1 工作窃取算法

看看维基百科对工作窃取算法的描述：在并行计算中，工作窃取是多线程计算机程序的调度策略。它解决了在具有固定数量的处理器（或内核）的静态多线程计算机上执行动态多线程计算的问题，该计算可以“产生”新的执行线程。它在执行时间，内存使用和处理器间通信方面都非常有效。

一般是一个双端队列和一个工作线程绑定，如下图所示。工作线程从绑定的队列的头部取任务执行，从别的队列（一般是随机）的底部偷取任务。

<img src="Java%E6%BA%90%E7%A0%81.assets/%E5%B7%A5%E4%BD%9C%E7%AA%83%E5%8F%96%E7%AE%97%E6%B3%95%E5%9B%BE.png" style="zoom:70%;" />

#### 2.2 ForkJoinPool类的结构图

![](Java%E6%BA%90%E7%A0%81.assets/ForkJoinPool%E7%BB%93%E6%9E%84%E5%9B%BE.png)

可以看到ForkJoinPool中有很多重要的成员变量，其中有一个WorkQueue数组，数组中存放的是WorkQueue对象，每一个WorkQueue对象也有自己的属性，然后还有一个ForkJoinTask数组array，task都是存在数组中间的。偶数下标的存放的都是外部提交的任务，而奇数下标都是fork出来的子任务。

#### 2.3 ForkJoinPool 继承关系

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

#### 2.4 成员变量

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

<img src="Java%E6%BA%90%E7%A0%81.assets/ForkJoinPool-ctl.png" style="zoom:50%;" />

可以看到，ctl实际上是分成3份的，由于低32位记录的是一个WorkQueue的scanState，而这个scanState又用低16位记录着自己的WorkQueue在WorkQueue[] 中的下标，所以可以看成是四份。我们来详细说明这四份：

- 高16位：值是 **活跃线程数 - 并行度** ，用以表示活跃线程数（AC）。初始时，没有活跃线程数，所以这个区域的值为 **负并行度**，每增加1个活跃线程，就会在这个区域 +1，直到为0，也就是活跃线程数 = 并行度为止。
- 中高16位：值是 **总线程数 - 并行度** ，用以表示总线程数（TC）。初始时，没有线程，所以这个区域的值为 **负并行度**，每增加1个线程，就会在这个区域 +1，直到为0，也就是总线程数 = 并行度为止。
- 中低16位：因为WorkQueue的scanState在绑定线程的时候，初始化为当前WorkQueue在WorkQueue[] 中的下标，而WorkQueue[]的长度通过MAX_CAP可知，只用低16位就可以表示了。所以scanState的高16位没有信息，所以作者索性就用这16位中最高位为该线程的状态位，剩下的15位用作该线程的版本号。
- 低16位：由上述可知，为最近一次使用的WorkQueue在WorkQueue[] 中的下标。也就是说这16位指着一个空闲的WorkQueue，而这个空闲的WorkQueue中的stackPred又指着前一个空闲的WorkQueue。以此形成了一个栈。

### 3. 源码

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

<img src="Java%E6%BA%90%E7%A0%81.assets/ForkJoinPool%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%E5%9B%BE.png" style="zoom: 67%;" />

所以我们主要看一下有结果任务提交：submit方法。

#### 3.1 submit(ForkJoinTask)

~~~java
public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task) {
    if (task == null)
        throw new NullPointerException();
    externalPush(task);
    return task;
}
~~~

很简单一方法，就是调用了externalPush方法

##### 3.1.1 externalPush(ForkJoinTask)

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

##### 3.1.2 externalSubmit(ForkJoinTask)

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

##### 3.1.3 signalWork(WorkQueue, WorkQueue)

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

###### 3.1.3.1 tryAddWorker(int)

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

###### 3.1.3.2 总结signalWork方法

1. 如果有空闲线程，那么就唤醒空闲线程
2. 如果没有空闲线程，如果还能够添加线程：
   - 就使用线程工厂创建一个线程
   - 创建的时候，创建一个workQueue，放置在WorkQueue[]的奇数下标处，与该线程进行双向绑定
   - 最后启动线程

##### 3.1.4 submit方法总结

1. 先用简版externalPush方法进行提交
2. 如果简版方法提交失败，调用完整版externalSubmit方法进行提交
   - 如果WorkQueue[]还没有初始化，先初始化WorkQueue[]
   - 如果对应偶数槽的WorkQueue还没有，接着创建对应槽的WorkQueue
   - 把任务放到WorkQueue的task数组中
3. 用signalWork方法，唤醒一个空闲线程或者创建一个新的线程来执行任务

上面已经分析完了，外部任务提交主线程所做的一些操作。接着，我们看一下，线程被唤醒或者被创建出来、启动之后进行了哪些操作

#### 3.2 ForkJoinWorkerThread的run()

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

##### 3.2.1 runWorker(WorkQueue)

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

###### 3.2.1.1 scan(WorkQueue, int)

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

###### 3.2.1.2 runTask(ForkJoinTask)

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

###### 3.2.1.3 awaitWork(WorkQueue, int)

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

##### 3.2.2 deregisterWorker(ForkJoinWorkerThread, Throwable)

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

#### 3.3 小结

到这，我们就梳理完了：一个任务从外部提交，放置到偶数槽，然后提交任务线程创建一个工作线程并且绑定一个奇数槽位，工作线程，循环探测偷取任务，并执行的全过程。接下来，我们就来看一下，一个任务如果粒度太大，需要fork划分成小任务的时候，fork方法到底做了什么。

#### 3.4 fork()

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

##### 3.4.1 push(ForkJoinTask)

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

#### 3.5 join()

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

<img src="Java%E6%BA%90%E7%A0%81.assets/ForkJoinTask-join().png" style="zoom:80%;" />

##### 3.5.1 doJoin()

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

###### 3.5.1.1 doExec()

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

###### 3.5.1.2 awaitJoin(WorkQueue, ForkJoinTask, long)

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

#### 3.6 总结 

ForkJoin框架内部设计相当复杂，只需要理解大致思路即可。

**在网上发现一篇能帮助理解ForkJoin框架原理的博文，一定要看：[指北 | 谈谈ForkJoin框架的设计与实现](D:\TyporaDocument\Java\谈谈ForkJoin框架的设计与实现.md)**

参考：

[JUC源码分析-线程池篇（五）：ForkJoinPool - 2](https://www.jianshu.com/p/6a14d0b54b8d)

[Executor（五）：ForkJoinPool详解 jdk1.8](https://blog.csdn.net/LCBUSHIHAHA/article/details/104449454)























------



# 包装类

## Integer

### getChars方法

~~~java
//将给定的int i原样转换成字符串
static void getChars(int i, int index, char[] buf) {
    int q, r;
    int charPos = index;
    char sign = 0;
    
    if (i < 0) {
        sign = '-';
        i = -i;
    }

    // 如果i >= 65536， 就一次性取末尾两位转换
    while (i >= 65536) { //假设此时i = 65536
        q = i / 100; 	// q = 655	
        //(q << 6) + (q << 5) + (q << 2) = q *(2的6次+2的5次+2的2次) = q*100 = 65500
        r = i - ((q << 6) + (q << 5) + (q << 2)); // r = 65536 - 65500 = 36
        i = q;
        buf [--charPos] = DigitOnes[r];
        buf [--charPos] = DigitTens[r];
    }

    // 如果i < 65536，就每次取末尾一位转换
    for (;;) {//假设此时 i = 65535
        // 向右无符号右移19位，即除以2的19次方（524288），所以前面乘以524288/10 + 1 = 52429
        //（加1是为了避免向下取整而产生误差）,即得到q ≈ i/10 = 6553
       	// 那为什么是无符号右移19位不是18位、20位呢？ 
        
        // 2^10=1024, 103/1024=0.1005859375
		// 2^11=2048, 205/2048=0.10009765625
		// 2^15=32768, 3277/32768=0.100006103515625
		// 2^16=65536, 6554/65536=0.100006103515625
		// 2^19=524288, 52429/524288=0.10000038146972656
		// 2^20=1048576, 104858/1048576=0.1000003815
        
        // 这里有两个前提——效率：移位 > 乘法 > 除法
        
        // 可以看到无符号右移位数越多，精度越高，所以右移位数尽量多，但是因为int型只有32个二进制位
        // 右移位数越多，i乘以的数就越大，为了不溢出，i就只能尽量小。i小了之后，更多的数就落到上面
        // 的while循环里面去了（while循环里面用的：除法+移位，此处for循环用的：乘法+移位。
        // for循环效率更高），两者就产生了矛盾。所以当精度够高的时候，我们就不移那么多位，保证效率。		 // 例如:
        // 这个里面的i就必须小于: 2的32次方(4294967296)/52429 = 81919。所以就让 i < 2的16次方
        q = (i * 52429) >>> (16+3);
        r = i - ((q << 3) + (q << 1));  // r = i-(q*10)
        buf [--charPos] = digits [r];
        i = q;
        if (i == 0) break;
    }
    if (sign != 0) {
        buf [--charPos] = sign;
    }
}



//这两个数组很巧妙
//十位为0，得到0，十位为1，得到1，无视个位，不用做运算，提高效率
final static char [] DigitTens = {
        '0', '0', '0', '0', '0', '0', '0', '0', '0', '0',
        '1', '1', '1', '1', '1', '1', '1', '1', '1', '1',
        '2', '2', '2', '2', '2', '2', '2', '2', '2', '2',
        '3', '3', '3', '3', '3', '3', '3', '3', '3', '3',
        '4', '4', '4', '4', '4', '4', '4', '4', '4', '4',
        '5', '5', '5', '5', '5', '5', '5', '5', '5', '5',
        '6', '6', '6', '6', '6', '6', '6', '6', '6', '6',
        '7', '7', '7', '7', '7', '7', '7', '7', '7', '7',
        '8', '8', '8', '8', '8', '8', '8', '8', '8', '8',
        '9', '9', '9', '9', '9', '9', '9', '9', '9', '9',
} ;
//个位为0，得到0，个位为1，得到1，无视十位
final static char [] DigitOnes = {
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
} ;
~~~

### toUnsignedString0方法（转二进制、八进制、16进制的核心方法）

~~~java
private static String toUnsignedString0(int val, int shift) {
    // mag为该数二进制串，从第一个为1的位置，到最低位总共的位数
    int mag = Integer.SIZE - Integer.numberOfLeadingZeros(val);
    // chars为存储mag位数所需要的数组长度，shift为进制数以二为底的对数（2-1,8-3,16-4），
    // 也即2进制准换成2进制位，1位用1位存储，转换成8进制，3位用1位存储，16进制4位用1位存储
	// 因为最少都需要1位，所以当前面小于1时，就取1
    // 假设转16进制，mag = 5~8位，都需要两位存储，因为java除法是向下取整的，
    // 所以除了后要加1，但是直接在 mag/shift 后加1,8个2进制位chas = 3出错
    // 所以用 mag 先加 shift-1再除以shift就没问题了 
    int chars = Math.max(((mag + (shift - 1)) / shift), 1);
    char[] buf = new char[chars];

    formatUnsignedInt(val, shift, buf, 0, chars);

    // Use special constructor which takes over "buf".
    return new String(buf, true);
}

	/**
	 * toUnsignedString0核心操作
     */
static int formatUnsignedInt(int val, int shift, char[] buf, int offset, int len) {
    int charPos = len;
    //通过shift计算进制的基
    int radix = 1 << shift;
    //掩码，比进制基少1，即低位全位1。基为8的话，mask = 00...0111(binary)
    int mask = radix - 1;
    do {
        // val & mask 很巧妙，val 与上 mask（假设为111）之后，就得到val的最低3位
        // 再以这个值为索引，取得字符数组里对应的字符
        buf[offset + --charPos] = Integer.digits[val & mask];
        // 上述操作完后，就把val无符号右移3位，继续操作
        val >>>= shift;
    } while (val != 0 && charPos > 0);

    return charPos;
}

final static char[] digits = {
        '0' , '1' , '2' , '3' , '4' , '5' ,
        '6' , '7' , '8' , '9' , 'a' , 'b' ,
        'c' , 'd' , 'e' , 'f' , 'g' , 'h' ,
        'i' , 'j' , 'k' , 'l' , 'm' , 'n' ,
        'o' , 'p' , 'q' , 'r' , 's' , 't' ,
        'u' , 'v' , 'w' , 'x' , 'y' , 'z'
};

~~~

### IntegerCache内部类

~~~java
// 该静态内部类在第一被使用到的时候加载
// 缓存有-128-127(最大值可以设置)，所以用valueOf会首先到这个缓存中寻找
private static class IntegerCache {
    
    //缓存中最小值
    static final int low = -128;
    //缓存中最大值
    static final int high;
    //缓存数组
    static final Integer cache[];

    static {
        // high value may be configured by property
        int h = 127;
        // 从配置中读取high的值
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        // 如果读出的high值不为null
        if (integerCacheHighPropValue != null) {
            try {
                // 把该值解析成为int型
                int i = parseInt(integerCacheHighPropValue);
                // 将较大者复制给i
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                // 数组最大长度不能超过下面，所以对h做出约束
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        // 将最终的h赋值给high
        high = h;
		// 新建缓存数组
        cache = new Integer[(high - low) + 1];
        int j = low;
        // 初始化缓存数组
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);

        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }

    private IntegerCache() {}
}
~~~



# 工具类

## java.util.stream

### Stream的常见用法：

~~~java
Stream.of(1, 5, 1, 4, 2, 4)
                .map(e -> e + 1)
                .sorted((e1,e2) -> e1 - e2)
                .filter(t -> t > 3)
                .forEach(e -> System.out.println(e));
~~~

从上面的代码可以看出，首先是用of方法先建一个Stream，然后链式调用。这里面由Stream.of方法产生的Stream对象是头结点，map、sorted、filter是中间操作，每进行一次中间操作就会产生一个流对象结点(stage:Stream)，forEach是结束操作。操作总结如下：

<table>
    <tbody>
        <tr>
            <td colspan="3" align="center" border="0">Stream操作分类</td>
        </tr>
        <tr>
            <td rowspan="2" border="1">中间操作(Intermediate operations)</td>
            <td>无状态(Stateless)</td>
            <td>unordered() filter() map() mapToInt() mapToLong() mapToDouble() flatMap() flatMapToInt() flatMapToLong() flatMapToDouble() peek()</td>
        </tr>
        <tr>
            <td>有状态(Stateful)</td>
            <td>distinct() sorted() sorted() limit() skip() </td>
        </tr>
        <tr>
            <td rowspan="2" border="1">结束操作(Terminal operations)</td>
            <td>非短路操作</td>
            <td>forEach() forEachOrdered() toArray() reduce() colect() max() min() count()
            </td>
        </tr>
        <tr>
            <td>短路操作(short-circuiting)</td>
            <td>anyMatch() allMatch() noneMatch() findFirst() findAny()</td>
        </tr>
    </tbody>
</table>
看完了操作，我们来看一下Stream体系的组织架构。

### Stream架构

<img src="Java源码.assets/Stream架构.png" style="zoom:75%;" />

上面表格里面的方法，基本上都是在ReferencePipeline这个类里面实现的

接着，我们根据上面的流应用的例子结合源码来看一下每一步干了什么事，生成了什么对象（注意结合体系架构图看，更清晰）。

首先是：

### 源码

#### Stream.of(T... values)

~~~java
// 1. Stream接口中的一个of方法，用来生成一个流
public static<T> Stream<T> of(T... values) {
    return Arrays.stream(values);
}

// 2. Arrays类中的方法，调用同类中的stream方法继续执行
public static <T> Stream<T> stream(T[] array) {
    return stream(array, 0, array.length);
}

// 3. Arrays类中的方法，调用StreamSupport中的stream方法继续执行
// 在调用stream方法之前，先用传进来的参数调用spliterator方法生成数据源
public static <T> Stream<T> stream(T[] array, int startInclusive, int endExclusive) {
    return StreamSupport.stream(spliterator(array, startInclusive, endExclusive), false);
}

// 4. Arrays类中的方法，调用Spliterators类中的spliterator方法继续执行
// 因为是Arrays类中的方法，所以生成的这个数据源需要带着符合数组的一些特征值（特征值之间用或操作）
// 这里就传递了两个特征值：Spliterator.ORDERED（有序）和Spliterator.IMMUTABLE（不可变）
public static <T> Spliterator<T> spliterator(T[] array, int startInclusive, int endExclusive) {
    return Spliterators.spliterator(array, startInclusive, endExclusive,
                                    Spliterator.ORDERED | Spliterator.IMMUTABLE);
}

// 5. Spliterators类中的方法，真正用来生成spliterator的方法，这里生成了一个ArraySpliterator
// 根据这个类的名字我们就能知道，这个类是各种spliterator类和生成各种spliterator的集合。所以各种
// 生成spliterator的操作最终都是由这个类的各种方法完成的
public static <T> Spliterator<T> spliterator(Object[] array, int fromIndex, int toIndex,int additionalCharacteristics) {
    checkFromToBounds(Objects.requireNonNull(array).length, fromIndex, toIndex);
    return new ArraySpliterator<>(array, fromIndex, toIndex, additionalCharacteristics);
}

// 回到3. 我们看一下StreamSupport中的stream方法。通过该类名可得知，该类是为流提供支持的，比如：
// 生成流管道结点
// 该方法接受两个参数：数据源和是否并行操作，显然调用的时候没有要求并行操作，所以parallel是false
// 该方法new了一个Head的流结点。
public static <T> Stream<T> stream(Spliterator<T> spliterator, boolean parallel) {
    Objects.requireNonNull(spliterator);
    return new ReferencePipeline.Head<>(spliterator,
                       StreamOpFlag.fromCharacteristics(spliterator), parallel);
}
~~~

在上面的操作中出现了数据源，我们来看看这个spliterator到底是个什么东西，[Spliterator](#Spliterator)。

通过上面一系列的操作，我们得到了一个Head对象，下面我们来看一下Head类

#### Head类

~~~java
static class Head<E_IN, E_OUT> extends ReferencePipeline<E_IN, E_OUT> {
    /**
         * Constructor for the source stage of a Stream.
         *
         * @param source {@code Supplier<Spliterator>} describing the stream
         *               source
         * @param sourceFlags the source flags for the stream source, described
         *                    in {@link StreamOpFlag}
         */
    Head(Supplier<? extends Spliterator<?>> source,
         int sourceFlags, boolean parallel) {
        super(source, sourceFlags, parallel);
    }

    /**
         * Constructor for the source stage of a Stream.
         *
         * @param source {@code Spliterator} describing the stream source
         * @param sourceFlags the source flags for the stream source, described
         *                    in {@link StreamOpFlag}
         */
    Head(Spliterator<?> source,
         int sourceFlags, boolean parallel) {
        super(source, sourceFlags, parallel);
    }

    @Override
    final boolean opIsStateful() {
        throw new UnsupportedOperationException();
    }

    @Override
    final Sink<E_IN> opWrapSink(int flags, Sink<E_OUT> sink) {
        throw new UnsupportedOperationException();
    }

    // Optimized sequential terminal operations for the head of the pipeline

    @Override
    public void forEach(Consumer<? super E_OUT> action) {
        if (!isParallel()) {
            sourceStageSpliterator().forEachRemaining(action);
        }
        else {
            super.forEach(action);
        }
    }

    @Override
    public void forEachOrdered(Consumer<? super E_OUT> action) {
        if (!isParallel()) {
            sourceStageSpliterator().forEachRemaining(action);
        }
        else {
            super.forEachOrdered(action);
        }
    }
}
~~~

根据以上代码可以看到，Head类啥也没干，就算重写了父类方法，也只是用super调用父类的方法。而根据代码中的英文注释可以得出，Head类对象就是一个管道流的起点，本身不包含对数据源的操作。下面我们通过他的构造方法看看他是怎么成为流的头结点的。

~~~java
// 直接super父类构造方法
Head(Spliterator<?> source, int sourceFlags, boolean parallel) {
    super(source, sourceFlags, parallel);
}

// 父类直接再父类
ReferencePipeline(Spliterator<?> source, int sourceFlags, boolean parallel) {
    super(source, sourceFlags, parallel);
}

// 可以看到new Head的时候最终起效果的是爷爷类的构造方法
AbstractPipeline(Spliterator<?> source, int sourceFlags, boolean parallel) {
    this.previousStage = null;        // 直接设置前指针为null（自己就是源）
    this.sourceSpliterator = source;  // 设置数据源
    this.sourceStage = this;          // 设置源stage为当前stage（因为自己就是源）
    this.sourceOrOpFlags = sourceFlags & StreamOpFlag.STREAM_MASK;
    this.combinedFlags = (~(sourceOrOpFlags << 1)) & StreamOpFlag.INITIAL_OPS_VALUE;
    this.depth = 0;					  // 源stage深度为0	
    this.parallel = parallel;         // 设置并行标志
}
~~~

通过上面的构造方法很清楚地看到，当调用生成流方法的时候，会返回一个Head stage，而在new这个stage的时候通过调用它爷爷的一个构造方法，使这个stage成为了流的头。

构造出一个流管道对象之后，就可以进行流相应的操作了。我们接着代码往下看，流对象调用了一个map方法（映射的意思），传进了一个lambda表达式，也即一个匿名内部类对象。我们来看一下，调用map的背后到底做了哪些操作

#### 流stage的map方法

~~~java
/**
 * ReferencePipeline类
 * map方法为一个泛型方法。mapper接受P_OUT类型，返回R类型
*/
public final <R> Stream<R> map(Function<? super P_OUT, ? extends R> mapper) {
    Objects.requireNonNull(mapper);
    // 返回一个StatelessOp的匿名子类对象，因为map方法是每个元素是独立处理的，没有相互的关系
    // 这里的this表示调用这个方法的对象，也即上流结点
    return new StatelessOp<P_OUT, R>(this, StreamShape.REFERENCE,
                                     StreamOpFlag.NOT_SORTED | StreamOpFlag.NOT_DISTINCT) {
        // 重写了父类的的opWrapSink方法，这个方法定义在AbstractPipeline中（该类的曾祖父类）
        // 方法用来生成一个Sink类（子类）对象
        Sink<P_OUT> opWrapSink(int flags, Sink<R> sink) {
            // 这个方法根据参数sink，构造一个Sink.ChainedReference的匿名内部类对象并返回
            // 原理：ChainedReference中维护了一个Sink对象downstream，在构造方法中将sink赋值给它
            // 即当前返回的Sink对象包含了另一个Sink对象，这个对象即是管道的下游Sink
            return new Sink.ChainedReference<P_OUT, R>(sink) {
              	// 这个匿名内部类重写了accept方法，先用当前Sink的mapper对数据进行处理，然后交给下游Sink继续处理
                public void accept(P_OUT u) {
                    downstream.accept(mapper.apply(u));
                }
            };
        }
    };
}
~~~

**注意**：mapper只是当前map方法的一个形参，理应当随着这个方法的结束而死亡，为什么在后面Sink对象能使用该mapper呢，参见[局部内部类](#局部内部类)。

这个方法其实挺简单的，就是new了一个StatelessOp的匿名内部类并返回，至于里面两层的重写方法，因为还没有调用到，所以不急着讨论。前面说到，每调用一个中间操作就会相应地生成一个流结点，那我们就来看看这新生成的结点是怎么与前面的结点产生联系。下面来看一下StatelessOp类的构造方法

#### StatelessOp类的构造方法

~~~java
/**
 * 这个构造方法有三个参数
 * upstream: 上流结点（stage）
 * inputShape：这个流结点处理的数据的类型（StreamShape是enum，其中REFERENCE表示引用类型）
 * opFlags：操作标识符
*/
StatelessOp(AbstractPipeline<?, E_IN, ?> upstream,
            StreamShape inputShape,
            int opFlags) {
    // 直接调用父类的构造方法
    super(upstream, opFlags);
    assert upstream.getOutputShape() == inputShape;
}

// 父类构造方法
ReferencePipeline(AbstractPipeline<?, P_IN, ?> upstream, int opFlags) {
    // 又直接交给父类
    super(upstream, opFlags);
}

// 最终进行实际操作的构造方法
AbstractPipeline(AbstractPipeline<?, E_IN, ?> previousStage, int opFlags) {
    // 检查一下传进来的stage是否已连接或者已被消费
    if (previousStage.linkedOrConsumed)
        throw new IllegalStateException(MSG_STREAM_LINKED);
  	// 把上流stage的linkedOrConsumed标识符设置为true
    previousStage.linkedOrConsumed = true;
    // 上流stage的next指针设置成当前正在new的stage，
    previousStage.nextStage = this;
	// 把当前stage的previous指针设置成上流stage
    this.previousStage = previousStage;
    // 设置一些属性（还没琢磨明白这些属性具体的内容，暂且不表）
    this.sourceOrOpFlags = opFlags & StreamOpFlag.OP_MASK;
    this.combinedFlags = StreamOpFlag.combineOpFlags(opFlags,previousStage.combinedFlags);
    this.sourceStage = previousStage.sourceStage;
    if (opIsStateful())
        sourceStage.sourceAnyStateful = true;
    // 把depth + 1，头结点的depth为0
    this.depth = previousStage.depth + 1;
}
~~~

可以看到，在new出这个新的stage的时候，就已经把这个stage挂到了流管道的末尾了，并且可以看到，**这个流管道是以双向链表组织起来的**。从这个我们就可以类比推出，每一个中间操作都只会直接返回一个StatelessOp或者StatefulOp匿名内部类（匿名子类）对象，并且在new的过程中把这个对象挂到流管道的末尾。从这，可以看出中间操作并没有执行具体的操作，是一种**懒加载的模式**。

那么问题来了，是什么时机触发整个流开始执行呢，执行过程又是怎样的呢？第一个问题，我们基本上可以推断出在调用结束操作的时候，触发了整个流的执行。我们来看看源码中是怎触发的，怎么执行的。下面以forEach结束操作为例

#### 流stage的forEach方法

~~~java
// ReferencePipeline类中的方法
public void forEach(Consumer<? super P_OUT> action) {
    // 首先调用了ForEachOps中的makeRef的方法，根据上述内容推断，这个方法应该是创建了一个流尾stage
    // evaluate方法应该就是触发执行的方法
    evaluate(ForEachOps.makeRef(action, false));
}

// ForEachOps中的makeRef方法，该方法返回一个TerminalOp对象
public static <T> TerminalOp<T, Void> makeRef(Consumer<? super T> action,
                                              boolean ordered) {
    Objects.requireNonNull(action);
    // 直接new一个ForEachOp.OfRef类对象返回，似乎初步验证了上面的猜想
    // 可以看出ForEachOp是Terminal类的子类，OfRef是ForEachOp的内部类也是子类
    return new ForEachOp.OfRef<>(action, ordered);
}

// OfRef构造方法
OfRef(Consumer<? super T> consumer, boolean ordered) {
    super(ordered);
    // 只是把具体操作consumer封装进了成员变量中，没有其他过多的操作
    this.consumer = consumer;
}
~~~

从OfRef的构造方法可以看出，它只是封装了具体操作，也没有把stage挂到管道流的末尾，并且从继承关系上可以看出OfRef类是一个Sink类，跟管道流的stage不在一个体系内，压根也不能挂到管道流上。那么前面的管道流与这个Sink有什么关系呢？

注意到管道流stage都重写了父类的opWrapSink方法，通过传进来的Sink返回一个新的Sink，结合管道流是一个双向链表，似乎可以发现些什么：结束操作直接创建了一个Sink对象，并触发管道流的执行操作，执行过程：

1. 首先用管道流尾stage（**因为结束操作是管道流尾stage调用的**），以结束操作返回的Sink对象为参数，返回一个新的Sink，这个新的Sink封装了结束操作返回的Sink。从链尾遍历回链头，不断返回的Sink做参数返回新Sink
2. Sink对象封装了每一个stage具体的操作
3. 最后的执行过程就是调用Sink对象中的方法

来看一下我们上述的分析是否正确

#### 流stage的evaluate方法

~~~java
/**
 * AbstractPipeline中的方法，并且为final，所有管道流stage都有、都是这个方法，不可被重写
 * 本例中terminalOp为ForEachOp.OfRef对象
*/
final <R> R evaluate(TerminalOp<E_OUT, R> terminalOp) {
    assert getOutputShape() == terminalOp.inputShape();
    if (linkedOrConsumed)
        throw new IllegalStateException(MSG_STREAM_LINKED);
    linkedOrConsumed = true;
    // 判断是否是并行流
    return isParallel()
        // 并行流执行并行方法
        ? terminalOp.evaluateParallel(this, sourceSpliterator(terminalOp.getOpFlags()))
        // 串行流执行串行方法（这里执行的串行方法）
        // 注意：这里的this是调用evaluate的对象，这里为流管道尾stage
        : terminalOp.evaluateSequential(this, sourceSpliterator(terminalOp.getOpFlags()));
}

// ForEachOp中的方法，重写自TerminalOp类，也即每个TerminalOp子类都有这个执行方法
public <S> Void evaluateSequential(PipelineHelper<T> helper,
                                   Spliterator<S> spliterator) {
    // 调用管道尾stage的wrapAndCopyInto方法。这个this是调用evaluateSequential的对象，即为terminalOp
    return helper.wrapAndCopyInto(this, spliterator).get();
}

/**
 * AbstractPipeline中的方法，并且为final，所有管道流stage都有、都是这个方法，不可被重写
 * 该方法调用了两个方法，首先是wrapSink，通过wrapSink返回的sink调用copyInto执行
*/
final <P_IN, S extends Sink<E_OUT>> S wrapAndCopyInto(S sink, Spliterator<P_IN> spliterator) {
    copyInto(wrapSink(Objects.requireNonNull(sink)), spliterator);
    return sink;
}
~~~

通过上面的分析，我们能看到，wrapSink就是我们上面分析的 1. 中包装Sink的方法，而copyInto就是执行最后返回Sink

#### 流stage的wrapSink方法

~~~java
/**
 * AbstractPipeline中的方法，并且为final，所有管道流stage都有、都是这个方法，不可被重写
*/
final <P_IN> Sink<P_IN> wrapSink(Sink<E_OUT> sink) {
    Objects.requireNonNull(sink);
    // 链尾向向链头迭代，用深度depth进行控制
    for ( @SuppressWarnings("rawtypes") AbstractPipeline p=AbstractPipeline.this; p.depth > 0; p=p.previousStage) {
        // 调用循环中当前stage的opWrapSink方法把管道流下游封装成一个新的sink，并用这个sink做参数继续封装（套娃）
        sink = p.opWrapSink(p.previousStage.combinedFlags, sink);
    }
    return (Sink<P_IN>) sink;
}
~~~

我们以开头调用map方法返回的stage为例，看一下这个stage中opWrapSink方法干了些啥。

#### map方法返回的stage的opWrapSink方法

~~~java
/**
 * 具体Stage中的方法，重写自AbstractPipeline类
*/
Sink<P_OUT> opWrapSink(int flags, Sink<R> sink) {
    // 这个方法根据参数sink，构造一个Sink.ChainedReference的匿名内部类对象并返回
    // 原理：ChainedReference中维护了一个Sink对象downstream，在构造方法中将sink赋值给它
    // 即当前返回的Sink对象包含了另一个Sink对象，这个对象即是管道的下游Sink
    return new Sink.ChainedReference<P_OUT, R>(sink) {
        // 这个匿名内部类重写了accept方法，先用当前Sink的mapper对数据进行处理，然后交给下游Sink继续处理
        public void accept(P_OUT u) {
            downstream.accept(mapper.apply(u));
        }
    };
}
~~~

封装完成后，返回最终的sink给copyInto方法使用

#### copyInto方法

~~~java
/**
 * AbstractPipeline中的方法，并且为final，所有管道流stage都有、都是这个方法，不可被重写
*/
final <P_IN> void copyInto(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator) {
    Objects.requireNonNull(wrappedSink);
    if (!StreamOpFlag.SHORT_CIRCUIT.isKnown(getStreamAndOpFlags())) {
        // 调用sink的begin方法
        wrappedSink.begin(spliterator.getExactSizeIfKnown());
        // 调用sink的accept方法（由于sink本身是cosumer类型的，而forEachRemaining方法会为每个元素调用一次accpet方法）
        spliterator.forEachRemaining(wrappedSink);
        // 调用sink的end方法
        wrappedSink.end();
    }
    else {
        copyIntoWithCancel(wrappedSink, spliterator);
    }
}
~~~

可以看到，上面方法中，使用到了sink的是三个方法：begin、accept、end，这个接口采取了模版方法的设计模式，这三个方法要按顺序调用。

#### Sink接口

~~~java
// 主要有三类方法
interface Sink<T> extends Consumer<T> {
    
    // 主要进行一些初始化作用，比如排序操作里面的new数组
    default void begin(long size) {}

    // 主要进行一些结束操作，比如排序里面的排序，并调用下游sink begin方法
    default void end() {}

    // 主要进行我们传进去的lambda表达式的操作
    default void accept(int value) {
        throw new IllegalStateException("called wrong accept method");
    }  
}

~~~

这三个方法都是default修饰，并不需要全部重写。一般简单的操作，比如map只需重写accept方法，另外两个直接使用接口里面的默认空方法；复杂点的操作比如sort，需要重写三个方法

上游sink在调用这三个方法时，会在合适的时机调用下游sink的方法，这样就形成了链式调用，例如map方法返回的sink和sort方法中返回的sink：

~~~java
// map方法返回的sink只重写了accept方法。会在调用accept方法处理完数据后，调用下游sink的accept方法
public void accept(P_OUT u) {
    downstream.accept(mapper.apply(u));
}

// sort方法返回额sink，重写了模版方法的三个方法。
private static final class SizedRefSortingSink<T> extends AbstractRefSortingSink<T> {
    private T[] array;
    private int offset;

    SizedRefSortingSink(Sink<? super T> sink, Comparator<? super T> comparator) {
        super(sink, comparator);
    }

	// begin方法接受上游sink传来的数据源的数组长度，并new出一个数组来，为接受数据做准备
    public void begin(long size) {
        if (size >= Nodes.MAX_ARRAY_SIZE)
            throw new IllegalArgumentException(Nodes.BAD_SIZE);
        array = (T[]) new Object[(int) size];
    }


    // 实际上的排序是由end方法来完成的，并且触发下游的操作
    public void end() {
        Arrays.sort(array, 0, offset, comparator);
        /** 
         * 排完序后，调用下游的begin方法，初始化下游的数据结合。例如
         * 下游如果是：
       	 *     sort的sink：那么下游就会new一个数组出来，为接收数据做准备；
         *     map的sink：那么这个方法将什么也不做，也不需要做。因为map的sink没有重写begin方法。
         */
        downstream.begin(offset);
        if (!cancellationWasRequested) {
        /**
         * 对每一个元素调用下游的accept方法。
         * 下游如果是：
       	 *     sort的sink：那么下游就会把当前元素放入数组；
         *     map的sink：将会调用封装的lambda表达式对数据进行处理，再调用下下游的accept方法
        */   
            for (int i = 0; i < offset; i++)
                downstream.accept(array[i]);
        }
        else {
            for (int i = 0; i < offset && !downstream.cancellationRequested(); i++)
                downstream.accept(array[i]);
        }
        // 最后调用下游的end方法
        downstream.end();
        // 当前数组置空，好被gc
        array = null;
    }


    // accept方法在这个sink中，只是接受了上游sink传递来的数据并存储在begin方法new出的数组中
    public void accept(T t) {
        array[offset++] = t;
    }
}

~~~

到这里，其实有一个疑问：为什么每一个stage已经包含了操作，而且stage也是双向链表连接起来了，不直接使用stage处理完，调用下一个stage的处理呢？这是因为每一个stage的操作都是不一样的，比如map方法返回的stage，就只用lambda遍历每一个元素进行相应的处理就好，而sort方法返回的stage需要先new一个数组，然后把每一个元素填入到数组中，然后再调用数组排序。这样每一个stage的涉及到的操作和方法并不一样，调用起来很复杂。为了能很好的完成上下游之间联动，就抽象出了三个模版方法。上游只需要在合适的时机按顺序调用下游的三个模版方法就好了（因为每一个sink都有这三个方法，上游知道下游必定有这三个方法，放心调用就完事了）。

上面说了这么多，流操作的基本流程已经理清楚了，下面一图以蔽之：

<img src="Java%E6%BA%90%E7%A0%81.assets/stream%E6%B5%81%E7%A8%8B%E5%9B%BE.png" style="zoom:40%;" />

**补**：补一个流的代码

~~~fJava
// 好好研究这个代码，很有价值
Arrays.stream(nums)
    .boxed()
    .map(e -> e.toString())
    .sorted((e1, e2) -> {
        String s1 = e1 + e2;
        String s2 = e2 + e1;
        for(int i = 0; i < s1.length(); i++){
            if(s1.charAt(i) - s2.charAt(i) > 0) return -1;
            else if(s1.charAt(i) - s2.charAt(i) < 0) return 1;
        }
        return 0;
    })
    .reduce(String::concat) //这里流已经消费完了，下面的filter方法是这个方法返回的Optional类的方法，不要混淆
    .filter(e -> !e.startsWith("0"))
    .orElse("0");
~~~

**注：**并行流使用了[ForkJoin](#Fork/Join框架)框架

## Spliterator

Spliterator从字面意思看就是可开拆分的迭代器，官方定义：是一个能够对源数据进行分区和遍历操作的对象。我们来看一下这个接口的源码

~~~java
/**
 * Spliterator接口中还有一些内部接口，为了简洁，此处我已经全部删去
*/
public interface Spliterator<T> {

    // 单个对元素执行给定的动作，如果有剩下元素未处理返回true，否则返回false
    boolean tryAdvance(Consumer<? super T> action);

	// 对每个剩余元素执行给定的动作，依次处理，直到所有元素已被处理或被异常终止。
   	// 默认方法调用tryAdvance方法
    default void forEachRemaining(Consumer<? super T> action) {
        do { } while (tryAdvance(action));
    }

    // 对任务分割，返回一个新的Spliterator迭代器
    Spliterator<T> trySplit();
	
   	// 用于估算还剩下多少个元素需要遍历
    long estimateSize();

    // 当迭代器拥有SIZED特征时，返回剩余元素个数；否则返回-1
    default long getExactSizeIfKnown() {
        return (characteristics() & SIZED) == 0 ? -1L : estimateSize();
    }

    // 返回当前对象有哪些特征值
    int characteristics();

    // 是否具有当前特征值
    default boolean hasCharacteristics(int characteristics) {
        return (characteristics() & characteristics) == characteristics;
    }

    // 如果Spliterator的list是通过Comparator排序的，则返回Comparator
    // 如果Spliterator的list是自然排序的 ，则返回null
    // 其他情况下抛错
    default Comparator<? super T> getComparator() {
        throw new IllegalStateException();
    }

   	// 一些特征值
    // 表示为元素定义遇到顺序
    public static final int ORDERED    = 0x00000010;
    
    // 表示对于每对遇到的元素x, y，!x.equals(y)
    public static final int DISTINCT   = 0x00000001;
    
    // 表示遇到的顺序遵循定义的排序顺序
    public static final int SORTED     = 0x00000004;
    
    // 表示从estimateSize()遍历或分割之前返回的值表示有限大小，在没有结构源修改的情况下，表示完全遍历将遇到的元素数量的精确计数
    public static final int SIZED      = 0x00000040;
    
    // 表示源保证遇到的元素不会nul。
    public static final int NONNULL    = 0x00000100;
    
    // 表示元素源不能在结构上进行修改; 即不能添加，替换或删除元素，因此在遍历过程中不会发生这种更改。
    public static final int IMMUTABLE  = 0x00000400;  
    
    // 表示可以通过多个线程安全同时修改元素源（允许添加，替换和/或删除），而无需外部同步
    public static final int CONCURRENT = 0x00001000;
    
    // 表示由所产生的所有Spliterator trySplit()都将SIZED和SUBSIZED
    public static final int SUBSIZED = 0x00004000;
    
}

~~~

可以看到Spliterator接口规定了一些方法和一些特征值。trySplit方法将数据源分割，每个迭代器访问一部分数据，那么几个迭代器之间可以并行操作，提高迭代效率。特征值其实就是为表示该Spliterator有哪些特性，用于可以更好控制和优化Spliterator的使用。关于获取比较器getComparator这一个方法，目前我还没看到具体使用的地方，所以可能理解有些误差。下面看一下spliterator的一个具体实现类

### ArraySpliterator类

~~~java
static final class ArraySpliterator<T> implements Spliterator<T> {
	
    private final Object[] array;        // 维护的数据源
    private int index;                   // 起始索引（包含），会被trySplit修改    
    private final int fence;             // 结束索引（不包含）
    private final int characteristics;   // 该类的特征值集合

    public ArraySpliterator(Object[] array, int additionalCharacteristics) {
        this(array, 0, array.length, additionalCharacteristics);
    }

    public ArraySpliterator(Object[] array, int origin, int fence, int additionalCharacteristics) {
        this.array = array;
        this.index = origin;
        this.fence = fence;
        // 该类肯定有SIZED、SUBSIZED（特征值之间用或操作），再或上传进来的特征值
        this.characteristics = additionalCharacteristics | Spliterator.SIZED | Spliterator.SUBSIZED;
    }
    
    
    @Override
    // 重写的trySplit方法
    public Spliterator<T> trySplit() {
        // 根据index和fence求一个mid，数组中间索引
        int lo = index, mid = (lo + fence) >>> 1;
        // 判断起始索引是否大于中间索引，是的话就返回null，说明已不可再分割
        // 否的话，就新建一个迭代器，起始索引为原起始索引，结束索引为mid，并且把mid赋值给index
        // 也就是新的迭代器迭代数组低位的一半，原先的迭代器迭代数组高位的一半
        return (lo >= mid)
            ? null
            : new ArraySpliterator<>(array, lo, index = mid, characteristics);
    }

    @SuppressWarnings("unchecked")
    @Override
    // 遍历方法
    public void forEachRemaining(Consumer<? super T> action) {
        Object[] a; int i, hi; // hoist accesses and checks from loop
        if (action == null)
            throw new NullPointerException();
        // 条件判断，数组长度要 >= fence，并且 起始索引 >= 0，并且索引 < fence
        // 注意在最后一个条件中，把起始索引index赋值为了fence，所以这个方法体只会执行一次
        if ((a = array).length >= (hi = fence) &&
            (i = index) >= 0 && i < (index = hi)) {
            do { action.accept((T)a[i]); } while (++i < hi);
        }
    }

    @Override
    // 方法名字，尝试前进，也就是尝试用action处理下一个元素，处理成功返回true，下标越界则返回false
    public boolean tryAdvance(Consumer<? super T> action) {
        if (action == null)
            throw new NullPointerException();
        // 由于成员变量index一直在增加，所以这个方法与forEachRemaining一样，只会执行一次
        if (index >= 0 && index < fence) {
            @SuppressWarnings("unchecked") T e = (T) array[index++];
            action.accept(e);
            return true;
        }
        return false;
    }

    @Override
    public long estimateSize() { return (long)(fence - index); }

    @Override
    public int characteristics() {
        return characteristics;
    }

    @Override
    // 判断当前迭代器特征值是否有自然排序特征值，是的话返回null（目前还没搞懂这个方法的实际价值）
    public Comparator<? super T> getComparator() {
        if (hasCharacteristics(Spliterator.SORTED))
            return null;
        throw new IllegalStateException();
    }
}

~~~

[回到Stream.Of](#Head类)