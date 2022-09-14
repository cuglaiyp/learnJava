# ThreadLocal

## 源码

ThreadLocal也是开发中比较常用到的一个线程工具类。我们来分析一下它的源码。

首先看一下ThreadLocal类给我们提供的最常用的方法：

> `public T get()`
>
> `public void set(T value)`

get()方法是用来获取ThreadLocal在当前线程中保存的变量引用（值类型是值副本），set()用来设置当前线程中变量引用（值类型是值副本），remove()用来移除当前线程中变量引用（值类型是值副本），initialValue()是一个protected方法，一般是用来在使用时进行重写的，它是一个延迟加载方法，下面会详细说明。

### get()

~~~java
public T get() {
    // 首先拿到当前线程
    Thread t = Thread.currentThread();
    // 拿到当前线程里面的一个map
    ThreadLocalMap map = getMap(t);
    // 判断一下map初始化没有
    if (map != null) {
        // 用当前的ThreadLocal变量为key，从当前线程中取出Entry来
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            // 从entry中把值取出来
            T result = (T)e.value;
            return result;
        }
    }
    // 如果当前线程还没有初始化，那么就自动设置一个初始化值，并且返回这个值
    return setInitialValue();
}
~~~

#### getMap(Thread)

~~~java
// 直接返回Thread的成员变量threadLocals。该类型为ThreadLocal的内部类ThreadLocalMap
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

ThreadLocal.ThreadLocalMap threadLocals = null;
~~~

#### setInitialValue()

~~~java
private T setInitialValue() {
    // 使用这个方法获取一个初始值。这个方法从下面可以看到，是一个钩子方法，留给子类重写，自定义逻辑的
    T value = initialValue();
    // 获取了初始值，那么自然要存进去
    // 拿到当前线程
    Thread t = Thread.currentThread();
    // 拿到线程中的map
    ThreadLocalMap map = getMap(t);
    // 判断一下map初始化没有
    if (map != null)
        // map初始化，刚刚获取的值，用当前的ThreadLocal变量为key，存入当前线程的map中
        map.set(this, value);
    // map没有初始化，那么初始化一个map，并且把数据存入进去
    else
        createMap(t, value);
    // 最后返回这个初始化值
    return value;
}


protected T initialValue() {
    return null;
}
~~~

#### get方法总结

`threadLocal.get()`底层逻辑就是：

获取当前线程的内部map———>用自己（threaLocal变量）为key取出对应的值———>如果这个map没有初始化或者取不到值就用该key设置一个初始值

### set(T value)

~~~java
public void set(T value) {
    // 依旧是先获取当前线程
    Thread t = Thread.currentThread();
    // 获取map
    ThreadLocalMap map = getMap(t);
    // map不为null，就set值
    if (map != null)
        map.set(this, value);
    else
        // 为空就初始化一个map，并set值
        createMap(t, value);
}
~~~

## 原理图

通过上面的分析，我们画一个图直观地展示一下：

<img src="ThreadLocal.assets/ThreadLocal%E7%90%86%E8%A7%A3%E5%9B%BE.png" style="zoom:40%;" />

![img](https://images0.cnblogs.com/blog/458716/201401/172259164557.jpg)

通过代码和上面的图也可以看出，每个线程的map里面存的并不是value的一个副本那么简单：对于值类型来说，存的是value的副本，而对于引用类型来说，存的是value的一个引用。所以**在value为共享变量的时候，并不能通过ThreadLocal来解决冲突，因为每个线程里面存的都是同一个对象引用，依旧会造成冲突**。

## 作用

所以ThreadLocal不是用来解决对象共享访问问题的，而主要是提供了保持对象的方法和避免参数传递的一种方便的对象访问方式。归纳了两点：

1. 每个线程中都有一个自己的ThreadLocalMap类对象，可以将线程自己的对象保持到其中，各管各的，线程可以正确的访问到自己的对象。
2. 将一个共用的ThreadLocal静态实例作为key，将不同对象的引用保存到不同线程的ThreadLocalMap中，然后在线程执行的各处通过这个静态ThreadLocal实例的get()方法取得自己线程保存的那个对象，避免了将这个对象作为参数传递的麻烦。

## 疑惑

1. 为什么要用ThreadLocal绕一下，最后把数据存在线程自己的map中，而不是直接在ThreadLocal中弄一个map结构，存进去？

   - **最终存在线程中，可以保证在线程死去的时候，线程共享变量ThreadLocalMap则销毁。**
   - **如果ThreadLoad直接使用Map<Thread, Object>为底层数据结构，当有大量的线程使用ThreadLocal时，首先Map访问的性能会下降，伴随着线程生命周期，底层的Map还需要频繁的添加删除entity，这就很容易造成性能瓶颈。而且这种模式违反了单一职责原则**
   - 理论上是可以，但没那么优雅。你提出的做法实际上就是所有的线程都访问ThreadLocal的Map，而key是当前线程。但这有点小问题，一个线程是可以拥有多个私有变量的嘛，那key如果是当前线程的话，意味着还点做点「手脚」来唯一标识set进去的value。假设上一步解决了，还有个问题就是；并发量足够大时，意味着所有的线程都去操作同一个Map，Map体积有可能会膨胀，导致访问性能的下降。这个Map维护着所有的线程的私有变量，意味着你不知道什么时候可以「销毁」。现在JDK实现的结构就不一样了。线程需要多个私有变量，那有多个ThreadLocal对象足以，对应的Map体积不会太大。只要线程销毁了，ThreadLocalMap也会被销毁。

   

2. 关于ThreadLocalMap<ThreadLocal, Object>内存泄露问题：

   问题：

   由于ThreadLocalMap是以弱引用的方式引用着ThreadLocal，换句话说，就是**ThreadLocal是被ThreadLocalMap中的key以弱引用的方式关联着，因此如果ThreadLocal没有被ThreadLocalMap以外的对象引用，则在下一次GC的时候，ThreadLocal实例就会被回收，那么此时ThreadLocalMap里的一组KV的K就是null**了，因此在没有额外操作的情况下，此处的V便不会被外部访问到，而且**只要Thread实例一直存在，Thread实例就强引用着ThreadLocalMap，因此ThreadLocalMap就不会被回收，那么这里K为null的V就一直占用着内存**。

   **当线程没有结束，但是ThreadLocal已经被回收，则可能导致线程中存在ThreadLocalMap<null, Object>的键值对，造成内存泄露。（ThreadLocal被回收，ThreadLocal关联的线程共享变量还存在）。**

   综上，发生内存泄露的条件是

   - ThreadLocal实例没有被外部强引用，比如我们假设在提交到线程池的task中实例化的ThreadLocal对象，当task结束时，ThreadLocal的强引用也就结束了
   - ThreadLocal实例被回收，但是在ThreadLocalMap中的V没有被任何清理机制有效清理
   - 当前Thread实例一直存在，则会一直强引用着ThreadLocalMap，也就是说ThreadLocalMap也不会被GC

   

   解决方案：

   **虽然ThreadLocalMap的get，set方法可以清除ThreadLocalMap中key为null的value，但是get，set方法在内存泄露后并不会必然调用，所以为了防止此类情况的出现，我们有两种手段。**

   1. 使用完线程共享变量后，显式调用ThreadLocalMap.remove方法清除线程共享变量；
   2. JDK建议ThreadLocal定义为private static，这样ThreadLocal的弱引用问题则不存在了。

   

3. ThreadLocalMap中的key为什么要以弱引用的方式引用着threadLocal实例？

   正如上面所说，使用弱引用时，如果我们不需要threadLocal，把它的引用置为null，释放它和它在map里面的键值对的内存。线程不死的时候（比如使用了线程池），并且我们没有手动清理ThreadLocalMap，这个是时候就会出现ThreadLocalMap<null, Object>的键值对，造成内存泄露。

   **那么试想一下，如果key不用弱引用，而是强引用着threadLocal实例。这个时候，我们把外部的threadLocal引用置null，想释放threadLocal内存时，由于它又被map内的key强引用着，所以这个threadLocal所代表的键值对无法被回收。并且我们再也无法访问到这个键值对了，因为这个key的外部引用已经被置为null了。我们使用ThreadLocalMap.remove方法也不好使，因为这个key的threadLocal实例并不为null，唯一能做的大概就是盼望着这个线程早点挂掉把！**



