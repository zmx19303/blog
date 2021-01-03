# 序

​		在工作的过程中，HashMap是我们使用最多的集合之一。但是HashMap的数据结构是什么？HashMap的初始化过程是怎么样的？HashMap的数据的存取是怎么一个过程？HashMap的扩容原理？在这儿，带着这些问题，我们也研究一下源码，对这个过程进行一个详细的了解。

> 以下内容基于jdk1.8

# HashMap的初始化过程

​		要了解HashMap的初始化过程，首先要了解的是HashMap的构造函数。HashMap的构造函数总共有以下4个：

- HashMap()；
- HashMap(int initialCapacity)；
- HashMap(int initialCapacity, float loadFactor)；
- HashMap(Map<? extends K, ? extends V> m)。

  首先是无参的构造函数，源码如下：

```java
/**
 * Constructs an empty <tt>HashMap</tt> with the default initial capacity
 * (16) and the default load factor (0.75).
 */
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

​		先看注释：构造（创建）一个空HashMap，具有默认的初始容量(16)和默认的负载因子0.75)。那么，再具体看看在源码做了什么？整个构造函数只做了一件事，`this.loadFactor = DEFAULT_LOAD_FACTOR;`，在Java开发中，我们习惯将常量名全大写，`this.`都是引用当前类中的变量，所有`DEFAULT_LOAD_FACTOR`应该是一个常量，`loadFactor`是一个变量，这句话的意思就是把一个常量`DEFAULT_LOAD_FACTOR`赋值给了变量`loadFactor`。另外还有一句注释，意思是：其他所有的字段都取默认值。那么，这个常量`DEFAULT_LOAD_FACTOR`是多少，变量`loadFactor`又是什么意思呢？

​		源码如下：

```java
/**
 * The load factor used when none specified in constructor.
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

```java
/**
 * The load factor for the hash table.
 *
 * @serial
 */
final float loadFactor;
```

​		首先是常量`DEFAULT_LOAD_FACTOR`，先看注释：在构造函数中没有指定时使用的负载因子。没有指定啥，那肯定是没有指定负载因子。默认的负载因子的值是0.75。

​		然后是变量`loadFactor`，这个变量注释的意思是：哈希表的负载因子。哈希表是一种数据结构，这个我们应该知道，但是啥是负载因子？不知道，根据现有线索，也没有答案，先暂放。另外有个点需要注意一下，这个负载因子`loadFactor`的修饰词是`final` ，那就意味着这个变量在HashMap的生命周期内，只能赋值一次。

​		无参的构造函数了解完了，接下来，我们看下一个构造函数，上源码：

```java
/**
 * Constructs an empty <tt>HashMap</tt> with the specified initial
 * capacity and the default load factor (0.75).
 *
 * @param  initialCapacity the initial capacity.
 * @throws IllegalArgumentException if the initial capacity is negative.
 */
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```

​		先看注释：构造（创建）一个空HashMap，具有指定的初始容量和默认负载因子(0.75)。再来看源码，传入了一个参数，参数名称叫初始容量，也就是注释中的指定的初始容量；构造函数中就一句代码，调用另外的构造函数，传入当前构造函数的参数和`DEFAULT_LOAD_FACTOR`，这个常量我们熟啊，就是上面说的默认负载因子，值是0.75。那就和注释对应上了，具有指定的初始容量，也就是我们构造函数的参数；默认负载因子，就是常量`DEFAULT_LOAD_FACTOR`。那这个构造函数就完全明白了。

​		接下来，我们看看当前构造函数调用的另外一个构造函数，老规矩，源码如下：

```java
/**
 * Constructs an empty <tt>HashMap</tt> with the specified initial
 * capacity and load factor.
 *
 * @param  initialCapacity the initial capacity
 * @param  loadFactor      the load factor
 * @throws IllegalArgumentException if the initial capacity is negative
 *         or the load factor is nonpositive
 */
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
    this.threshold = tableSizeFor(initialCapacity);
}
```

​		先看注释：构造（创建）具有指定初始容量和加载因子的空HashMap。

​		源码第一步先对这个初始容量进行了校验，如果初始容量小于0，直接抛出`IllegalArgumentException`异常。如果初始容量大于常量`MAXIMUM_CAPACITY`，把初始容量设置为常量`MAXIMUM_CAPACITY`。这个常量是啥意思呢？值又是多少呢？

```java
/**
 * The maximum capacity, used if a higher value is implicitly specified
 * by either of the constructors with arguments.
 * MUST be a power of two <= 1<<30.
 */
static final int MAXIMUM_CAPACITY = 1 << 30;
```

​		先看注释：最大容量，如果两个带参数的构造函数中的任何一个隐式指定了更高的值，则使用该值。一定是2的幂。由注释可知，这个常量`MAXIMUM_CAPACITY`是HashMap容器的最大值，最大值是1<<30，等于2的30次方。那这个容器是什么呢？从我们以前看到过得一些资料可知，HashMap是由数组+链表+红黑树组成的，那这个容器是不是就是数组呢？先保留这个疑问，接着向下读源码。

​		让我们返回构造函数代码，下个阶段是对变量`loadFactor`进行了校验，如果小于0或者不是float，就抛出`IllegalArgumentException`异常。然后把参数中的负载因子赋值给了`loadFactor`，而参数中的初始化容量，调用方法`tableSizeFor(int cap)`后赋值给了变量`threshold`。这个变量`threshold`是啥作用呢？源码如下：

```java
/**
 * The next size value at which to resize (capacity * load factor).
 *
 * @serial
 */
// (The javadoc description is true upon serialization.
// Additionally, if the table array has not been allocated, this
// field holds the initial array capacity, or zero signifying
// DEFAULT_INITIAL_CAPACITY.)
int threshold;
```

​		翻译注释：要调整大小的下一个大小值(容量*负载因子)。哦，那就是下一个了临界值，到了这个临界值，就需要进行容器大小的调整。那么，什么东西需要调整容器的大小呢？应该是数组，因为数组需要指定初始容量，超过容量就不能再只能替换了。那现在我们就有一个初始的概念了，上面所说的所有容量都应该是数组的初始容量。接下来，注释还透露了一个信息，扩容的临界值是`容量*负载因子`。一般我们使用HashMap，用无参和指定初始化数组长度的比较多，所以负载因子一般就是0.75。

​		接下来，看下`tableSizeFor(int cap)`这个方法到底做了什么东西？源码如下：

```java
/**
 * Returns a power of two size for the given target capacity.
 */
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

​		先看注释：对于给定的目标容量，返回大小为2的幂。如果大小不是2的幂，那应该怎么返回？再看源码，全是位运算，解释起来就很麻烦，直接写测试方法看结果总结规律，测试代码如下：

```java
@Test
public void tableSizeForTest() {
    int n = 1 << 30;
    System.out.println(n);
    Scanner sc = new Scanner(System.in);
    while (true) {
        int i = sc.nextInt();
        int result = tableSizeFor(i);
        System.out.println("result = " + result);
    }
}
private int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= 1 << 30) ? 1 << 30 : n + 1;
}
```

​		经过测试之后，如果指定容量是0和1，那么结果就是1；如果指定容量大于2的30次方，结果就是2的30次方；其余的遵循规则：cap∈(2^n,2^n+1]，结果就是2^n+1。例如，2∈（2^0,2^1]，结果是2；3属于(2^1,2^2]，结果就是4；依次类推。

​		那么问题来了，`tableSizeFor(int cap)`函数返回的是扩容的临界值`threshold`，但是这个临界值很可能就出现比数组容量大的情况，但是上面的注释又说临界值=容量*负载因子，有点自相矛盾，所以，为啥要这么设计呢？这么设计有啥好处呢？或者是为了防止什么情况呢？先留下这个问题。

​		最后一个构造函数先不看，根据已经看完源码的三个构造函数我们来总结下：

1. 在HashMap初始化的过程中，只创建了一个类，没有创建数组、链表和红黑树；
2. 在HashMap初始化的过程中，最多只有两个变量发生了变化：负载因子`loadFactor`和临界值`threshold`;
3. 在HashMap初始化的过程中，初始容量`initialCapacity`不能小于0，否则会抛出异常`IllegalArgumentException`；初始容量`initialCapacity`大于2的30次方的时候，这个参数会被设置为最大值2的30次方；
4. 在HashMap初始化的过程中，负载因子`loadFactor`不能小于0，且不能是非数字；否则，会抛出异常`IllegalArgumentException`。

​    当然，还有一些疑问如下：

1. 负载因子是什么？
2. HashMap中的容量指的是不是数组的长度？
3. HashMap到底是什么时候创建的数组、链表和红黑树这些我们所熟知的数据结构？
4. 为啥注释说临界值=容量*负载因子，但是初始化的时候，却又不一样了？

​	带着这些疑问，我们进入下个模块，HashMap的存值。

# HashMap存值

​		HashMap初始化之后，我们一般做什么？没错，就是把我们需要的数据放入HashMap中。同理，我们先看HashMap的存值方法有哪些？

- `V put(K key, V value)`；
- `void putAll(Map<? extends K, ? extends V> m)`；
- `V putIfAbsent(K key, V value)`；

其中，putAll不做研究，我们需要看的就是`V put(K key, V value)`和`V putIfAbsent(K key, V value)`这两个方法。先看源码：

```java
/**
 * Associates the specified value with the specified key in this map.
 * If the map previously contained a mapping for the key, the old
 * value is replaced.
 *
 * @param key key with which the specified value is to be associated
 * @param value value to be associated with the specified key
 * @return the previous value associated with <tt>key</tt>, or
 *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
 *         (A <tt>null</tt> return can also indicate that the map
 *         previously associated <tt>null</tt> with <tt>key</tt>.)
 */
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

先看注释：将指定的值与此映射中的指定键关联。如果该映射先前包含了该键的映射，则旧值将被替换。

关于返回值的注释：





