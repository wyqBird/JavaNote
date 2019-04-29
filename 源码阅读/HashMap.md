# 1. HashMap 简介

```Java

public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable{...}
```

存储键值对；是哈希表的Map接口实现

# 2. 数据结构

## 2.1 对比：

- jdk1.8前：数组+链表，数组是HashMap主体，链表是解决hash冲突；
- jdk1.8后：数组+链表+红黑树，链表长度大于阈值（默认8），链表转化为红黑树，提高效率。红黑树插入效率 < 链表，但删、改、查的效率 > 链表。

## 2.2 执行过程：

- HashMap 通过 key 的 hashCode 经过扰动函数处理过后得到 hash 值
- 通过 (n - 1) & hash 判断当前元素存放的位置（这里的 n 指的是数组的长度）
- 如果当前位置存在元素的话，就判断(equals)该元素与要存入的元素的 hash 值以及 key 是否相同
- 如果相同的话，直接覆盖，不相同就通过拉链法解决冲突。

## 2.3 扰动函数:

- 指的就是 HashMap 的 hash 方法。
- 使用 hash 方法也就是扰动函数是为了防止一些实现比较差的 hashCode() 方法 
- 换句话说使用扰动函数之后可以减少碰撞。

## 2.4 hash 比较：

- 1.8 的 hash 方法比 1.7 的更加简洁，但原理不变。
- 1.8 的 hash 方法比 1.7 的性能更高，因为 7 中扰动四次。

## 2.5 hash()源码

1.8 hash 方法源码:

```Java

static final int hash(Object key) {
      int h;
      // key.hashCode()：返回散列值也就是hashcode
      // ^ ：按位异或
      // >>>:无符号右移，忽略符号位，空位都以0补齐
      return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

```

1.7 hash 方法源码：

```Java

static int hash(int h) {
    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).

    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

```

# 3. 源码解读

## 3.1 类的属性

```Java

// 序列号
private static final long serialVersionUID = 362498820763181265L;
// 默认的初始容量是16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
// 最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;
// 默认的填充因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 当桶(bucket)上的结点数大于这个值时会转成红黑树
static final int TREEIFY_THRESHOLD = 8;
// 当桶(bucket)上的结点数小于这个值时树转链表
static final int UNTREEIFY_THRESHOLD = 6;
// 桶中结构转化为红黑树对应的table的最小大小
static final int MIN_TREEIFY_CAPACITY = 64;
// 存储元素的数组，总是2的幂次倍
transient Node<K,V>[] table;
// 存放具体元素的集合
transient Set<Map.Entry<K,V>> entrySet;
// 存放元素的实际个数，注意这个不等于数组的长度。
transient int size;
// 每次扩容和更改map结构的计数器
transient int modCount;
// 临界值 当实际大小(容量*填充因子)超过临界值时，会进行扩容
int threshold;
// 加载因子
final float loadFactor;

```

说明：

- loadFactor加载因子：控制数组的装填量。给定的数组默认容量为 16，负载因子为 0.75。Map 在使用过程中不断的往里面存放数据，当数量达到了 16 * 0.75 = 12 就需要将当前 16 的容量进行扩容，而扩容这个过程涉及到 rehash、复制数据等操作，所以非常消耗性能。
- threshold：衡量数组是否需要扩增的一个标准。threshold = capacity * loadFactor，当Size>=threshold的时候，那么就要考虑对数组的扩增了

## 3.2 Node节点类

```Java

static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;//哈希值，存放元素到hashmap中时用来与其他元素hash值比较
    final K key;//键
    V value;//值
    Node<K,V> next;// 指向下一个节点

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }
	// 重写hashCode()方法
    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }
	// 重写 equals() 方法
    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}

```

## 3.3 树节点类

```Java

static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;// 父
    TreeNode<K,V> left;// 左 
    TreeNode<K,V> right;// 右
    TreeNode<K,V> prev; // 需要在删除后取消链接
    boolean red;// 判断颜色
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }

	// 返回根节点
    final TreeNode<K,V> root() {
        for (TreeNode<K,V> r = this, p;;) {
            if ((p = r.parent) == null)
                return r;
            r = p;
        }
    }

	...
}

```

## 3.4 构造方法

```Java
// 默认构造函数
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
// 指定“容量大小”的构造函数
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
// 指定“容量大小”和“加载因子”的构造函数
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
// 包含另一个“Map”的构造函数
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}

final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {// 判断table是否已经初始化
        if (table == null) { // pre-size
			// 未初始化，s为m的实际元素个数
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
			// 计算得到的t大于阈值，则初始化阈值
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        else if (s > threshold)// 已初始化，并且m元素个数大于阈值，进行扩容处理
            resize();
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
			// 将m中的所有元素添加至HashMap中
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}

```

## 3.5 put

### 3.5.1 如何确定哈希桶索引的位置？

- 键的哈希值：h = key.hashCode()
- 高位参与：h=h^(h>>>16)
- 取模：i = h & ((tab = resize()).length - 1)

由于length是2^n的合数， length - 1 就是 0111...1，而 & 运算只有高位0参与，因1&1=1，0&1=0，所以低位不改变h的值，所以是均匀哈希。

### 3.5.2 put的流程

- 1.判断table是否为null或长度是否为0？Y则先扩容，再继续2；N则继续2。
- 2.根据hash来计算数组下标？若对应位置为null，则直接插入。继续5.
- 3.如果对应位置有值，判断key是否相等？Y则直接覆盖。继续5.
- 4.N则判断 tab[i]是否为树节点？Y则红黑树直接插入。若不是，则遍历插入【尾插法】。插入后，若长度大于8，则转换为红黑树，再继续5，否者直接继续5。
- 5.判断是否需要扩容？Y则扩容，put结束，N则put结束结束

HashMap只提供了put用于添加元素，putVal方法只是给put方法调用的一个方法，并没有提供给用户使用。

```Java

public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
	// table未初始化或者长度为0，进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
	// (n - 1) & hash 确定元素存放在哪个桶中，桶为空，新生成结点放入桶中(此时，这个结点是放在数组中)
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {// 桶中已经存在元素
        Node<K,V> e; K k;
		// 比较桶中第一个元素(数组中的结点)的hash值相等，key相等
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;// 将第一个元素赋值给e，用e来记录
		// hash值不相等，即key不相等；为红黑树结点
        else if (p instanceof TreeNode)
			// 放入树中
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {// 为链表结点
			// 尾插法，在链表最末插入结点
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {// 到达链表的尾部
					// 在尾部插入新结点
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
						// 结点数量达到阈值，转化为红黑树
                        treeifyBin(tab, hash);
                    break;
                }
				// 判断链表中结点的key值与插入的元素的key值是否相等
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
				// 用于遍历桶中的链表，与前面的e = p.next组合，可以遍历链表
                p = e;
            }
        }
        if (e != null) { // // 表示在桶中找到key值、hash值与插入元素相等的结点
            V oldValue = e.value;// 记录e的value
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;//用新值替换旧值
            afterNodeAccess(e);// 访问后回调
            return oldValue;
        }
    }
	// 结构性修改
    ++modCount;
	// 实际大小大于阈值则扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);// 插入后回调
    return null;
}


```

### 3.5.3 jdk1.7的 put

- 如果定位到的数组位置没有元素就直接插入。
- 如果定位到的数组位置有元素，遍历以这个元素为头结点的链表，依次和插入的key比较，如果key相同就直接覆盖，不同就采用【头插法】插入元素。


```Java

public V put(K key, V value)
    if (table == EMPTY_TABLE) { 
    	inflateTable(threshold); 
	}  
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);
    int i = indexFor(hash, table.length);
    for (Entry<K,V> e = table[i]; e != null; e = e.next) { // 先遍历
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue; 
        }
    }

    modCount++;
    addEntry(hash, key, value, i);  // 再插入
    return null;
}

```

## 3.6 get

```Java

public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
			// 若第一个数组元素相等
            return first;
		// 遍历其他节点
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
				// 若是树节点，则在树中get
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}

```

## 3.7 resize

- 进行扩容，会伴随着一次重新hash分配，并且会遍历hash表中所有的元素，是非常耗时的。在编写程序中，要尽量避免resize。

```Java

final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;//引用扩容前的Entry数组
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
			// 超过最大值就不再扩充了，随便碰撞吧
            threshold = Integer.MAX_VALUE;//修改阈值为int的最大值(2^31-1)，这样以后就不会扩容了  
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
			// 没超过最大值，就扩充为原来的2倍
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
		// 计算新的resize上限
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
		// 把每个bucket都移动到新的buckets中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
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
						// 原索引
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
						// 原索引+oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
						// 原索引放到bucket里
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
						// 原索引+oldCap放到bucket里
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}

```