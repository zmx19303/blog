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

​		然后看最后一个构造函数，源码如下：

```java
/**
 * Constructs a new <tt>HashMap</tt> with the same mappings as the
 * specified <tt>Map</tt>.  The <tt>HashMap</tt> is created with
 * default load factor (0.75) and an initial capacity sufficient to
 * hold the mappings in the specified <tt>Map</tt>.
 *
 * @param   m the map whose mappings are to be placed in this map
 * @throws  NullPointerException if the specified map is null
 */
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

先看注释，构造一个新的<tt>HashMap</tt>，与指定的`Map`有相同的映射。<tt>HashMap</tt>使用默认负载系数(0.75)和初始容量足以容纳指定的<tt>Map</tt>的映射。从源码分析，这个构造函数和无参构造函数具有相同的结构，之后进行了一个put操作。所以，我们可以认为这个构造函数就是和无参构造函数相同。

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

​		先看注释：将指定的值与此映射中的指定键关联。如果该映射先前包含了该键的映射，则旧值将被替换。返回值的注释：前一个值与key相关联，如果没有key的映射，则为null。(返回null也可以表明当前map以前关联这个key的就是null)。综合注释，关于当前函数，可以表明的是，如果key重复，返回的就是以前存放在map中的值；如果不重复，返回的就是null；而在返回null的时候，也可能是key在map中存放的就是null。也就是说，HashMap的value可以为null。

​		再看函数`hash(key)`，源码如下：

```java
/**
 * Computes key.hashCode() and spreads (XORs) higher bits of hash
 * to lower.  Because the table uses power-of-two masking, sets of
 * hashes that vary only in bits above the current mask will
 * always collide. (Among known examples are sets of Float keys
 * holding consecutive whole numbers in small tables.)  So we
 * apply a transform that spreads the impact of higher bits
 * downward. There is a tradeoff between speed, utility, and
 * quality of bit-spreading. Because many common sets of hashes
 * are already reasonably distributed (so don't benefit from
 * spreading), and because we use trees to handle large sets of
 * collisions in bins, we just XOR some shifted bits in the
 * cheapest possible way to reduce systematic lossage, as well as
 * to incorporate impact of the highest bits that would otherwise
 * never be used in index calculations because of table bounds.
 */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

​		直接看源码，如果key为null，就返回0；否则，调用key的hashCode函数得到值h，用h和h无符号右移后的数做异或运算，并返回结果。具体细节，为啥是右移16位？这个需要深入研究，在此主要研究的不是这个，就不做深入讨论。

​		然后再看`V putIfAbsent(K key, V value)`函数的源码，如下：

```java 
/**
 * If the specified key is not already associated with a value (or is mapped
 * to {@code null}) associates it with the given value and returns
 * {@code null}, else returns the current value.
 * @since 1.8
 */
@Override
public V putIfAbsent(K key, V value) {
    return putVal(hash(key), key, value, true, true);
}
```

​		当前方法的注释太长，就只截取了一部分，只截取了一部分，这部分的意思就是：如果指定的键还没有与某个值相关联(或者被映射到null)，则将其与给定的值相关联并返回null，否则返回当前值。这个方法是1.8之后的新方法。

​		可以看到，这个方法调用的也是`putVal`函数，所以，接下来重点讲解这个方法。源码如下：

```java
/**
 * Implements Map.put and related methods
 *
 * @param hash hash for key
 * @param key the key
 * @param value the value to put
 * @param onlyIfAbsent if true, don't change existing value
 * @param evict if false, the table is in creation mode.
 * @return previous value, or null if none
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
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

​		先看注释：实现了Map.put()方法及相关方法。没啥重点，再看下参数注释：

- `boolean onlyIfAbsent`：if true, don't change existing value.如果为true，不改变已经存在的值。
- `boolean evict` ： if false, the table is in creation mode.如果为false，hash表处于创建模式。

先看第一部分代码

```java
Node<K,V>[] tab; Node<K,V> p; int n, i;
if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
```

代码做了以下几件事：

1. 定义了一系列的变量；从这儿可以看出，tab是HashMap的数组,n是数组的长度，其余的变量暂时不知道作用；
2. 用全局变量给局部变量赋值；`table`赋给了`tab`，`n`为`table`的长度；
3. 如果全局变量`table`为`null`，或者`table`的长度为0，则进行了扩容，并把扩容后的长度赋值给了`n`;

我们再来看函数`resize()`，源码如下：

```java
/**
 * Initializes or doubles table size.  If null, allocates in
 * accord with initial capacity target held in field threshold.
 * Otherwise, because we are using power-of-two expansion, the
 * elements from each bin must either stay at same index, or move
 * with a power of two offset in the new table.
 * 翻译：初始化或者翻倍表的大小。如果表为null,按照字段threshold中保留的初始容量进行分配。否则，由于使用2的幂次方进行扩容，
 * 容器的中每个元素必须保持同样的索引，或者在新的表中以2的偏移量进行移动。
 * 
 * @return the table
 */
final Node<K,V>[] resize() {
    //旧的数组
    Node<K,V>[] oldTab = table;
    //旧的数组长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    //旧的阈值
    int oldThr = threshold;
    int newCap, newThr = 0;
    //旧数组长度大于0
    if (oldCap > 0) {
        //旧数组长度大于最大数组长度,阈值去Integer的最大值，并返回当前数组；
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //旧数组长度乘以2之后小于最大数组长度且旧数组大于初始化长度，新的阈值乘以2
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    //如果旧的阈值大于0，则新的数组长度等于旧的阈值
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    //其余情况，新的数组长度和阈值取默认值
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    //如果新的阈值等于0时，ft=新数组长度*负载因子，如果新数组长度小于最大数组长度且ft小于最大容量，取ft；否则，取Integer最大值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    //把新阈值赋给全局阈值，按照新数组长度创建新数组，并把数组赋值给全局数组变量；
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    //如果旧数组不为null，进行数据复制
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                //释放旧数组空间
                oldTab[j] = null;
                //
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    //返回新数组
    return newTab;
}
```







