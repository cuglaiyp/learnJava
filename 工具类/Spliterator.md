# Spliterator

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

## ArraySpliterator类

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

