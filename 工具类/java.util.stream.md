# java.util.stream

## Stream的常见用法：

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

## Stream架构

<img src="java.util.stream.assets/Stream%E6%9E%B6%E6%9E%84.png" style="zoom:60%;" />

上面表格里面的方法，基本上都是在ReferencePipeline这个类里面实现的

接着，我们根据上面的流应用的例子结合源码来看一下每一步干了什么事，生成了什么对象（注意结合体系架构图看，更清晰）。

首先是：

## 源码

### Stream.of(T... values)

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

### Head类

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

### 流stage的map方法

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

**注意**：mapper只是当前map方法的一个形参，理应当随着这个方法的结束而死亡，为什么在后面Sink对象能使用该mapper呢，参见[局部内部类](D:\TyporaDocument\Java\内部类\内部类.md)。

这个方法其实挺简单的，就是new了一个StatelessOp的匿名内部类并返回，至于里面两层的重写方法，因为还没有调用到，所以不急着讨论。前面说到，每调用一个中间操作就会相应地生成一个流结点，那我们就来看看这新生成的结点是怎么与前面的结点产生联系。下面来看一下StatelessOp类的构造方法

### StatelessOp类的构造方法

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

### 流stage的forEach方法

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

### 流stage的evaluate方法

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

### 流stage的wrapSink方法

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

### map方法返回的stage的opWrapSink方法

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

### copyInto方法

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

### Sink接口

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

<img src="java.util.stream.assets/stream%E6%B5%81%E7%A8%8B%E5%9B%BE.png" style="zoom:50%;" />

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

**注：**并行流使用了[ForkJoin](D:\TyporaDocument\Java\多线程\ForkJoin框架.md)框架

### 并行流

当一个流调用parallel()方法时就会变成并行流

#### parallel()

~~~java
public final S parallel() {
    sourceStage.parallel = true;
    return (S) this;
}
~~~

可以看到，这个方法很简单地把流头stage的parallel标记置为true。然后在执行的时候再来做判断

~~~java
final <R> R evaluate(TerminalOp<E_OUT, R> terminalOp) {
    assert getOutputShape() == terminalOp.inputShape();
    if (linkedOrConsumed)
        throw new IllegalStateException(MSG_STREAM_LINKED);
    linkedOrConsumed = true;

    return isParallel()
        ? terminalOp.evaluateParallel(this, sourceSpliterator(terminalOp.getOpFlags()))
        : terminalOp.evaluateSequential(this, sourceSpliterator(terminalOp.getOpFlags()));
}
~~~

就是判断流头结点的parallel标记，如果为true，就走并行执行的逻辑

~~~java
public <S> Void evaluateParallel(PipelineHelper<T> helper,
                                 Spliterator<S> spliterator) {
    if (ordered)
        new ForEachOrderedTask<>(helper, spliterator, this).invoke();
    else
        new ForEachTask<>(helper, spliterator, helper.wrapSink(this)).invoke();
    return null;
}
~~~

并行执行

- 如果不需要保持顺序，那么就直接把相关数据包装成一个ForkJoinTask，包装时执行了包装Sink的操作。

  ForkJoinTask里面同时封装好了compute方法：任务分割的逻辑。使用Fork/Join框架来执行这个任务。**这也是为什么需要可分割迭代器（Spliterator）作为数据源的原因**。因为ForkJoinTask在执行的过程中会分割，正好使用Spliterator的特性。

- 需要保持顺序的话，也是封装成一个ForkJoinTask。（这个逻辑里，包装Sink是在任务执行的时候进行的）。但是在任务fork的过程中，**会使用一个ConcurrentHashMap来记录任务的顺序**







