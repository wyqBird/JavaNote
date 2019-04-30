# 1. ArrayList简介

```Java

public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{...}

```

- 底层是 Object[]，可动态增长。
- 插入数据时，通过 ensureCapacity 来增加 ArrayList 实例的容量。
- 扩容时，int newCapacity = oldCapacity + (oldCapacity >> 1); 新容量是旧容量的1.5倍
- 插入/删除元素的时间复杂度为O（n）。
- 查询/修改第 i 元素的时间复杂度为O（1）。
-  ArrayList 继承了AbstractList，实现了List。它是一个数组队列，提供了相关的添加、删除、修改、遍历等功能。
-  ArrayList 实现了RandomAccess 接口， RandomAccess 是一个标志接口，表明实现这个这个接口的 List 集合是支持快速随机访问的。在 ArrayList 中，我们即可以通过元素的序号快速获取元素对象，这就是快速随机访问。
-  ArrayList 实现了Cloneable 接口，即覆盖了函数 clone()，能被克隆。
-  ArrayList 实现java.io.Serializable 接口，这意味着ArrayList支持序列化，能通过序列化去传输。
-  和 Vector 不同，ArrayList 中的操作不是线程安全的！所以，建议在单线程中才使用 ArrayList，而在多线程中可以选择 Vector 或者 CopyOnWriteArrayList。


# 2. 源码解读

## 2.1 核心源码

### 2.1.1 变量

```Java

//序列号
private static final long serialVersionUID = 8683452581122892189L;

//默认初始容量大小
private static final int DEFAULT_CAPACITY = 10;

//空数组（用于空实例）
private static final Object[] EMPTY_ELEMENTDATA = {};

//用于默认大小空实例的共享空数组实例
//我把它从EMPTY_ELEMENTDATA数组中区分出来，以知道在添加第一个元素时容量需要增加多少。
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

//保存ArrayList数据的数组
transient Object[] elementData;

//ArrayList 所包含的元素个数
private int size;

//要分配的最大数组大小
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

```

### 2.1.2 构造方法

```Java

//初始容量为10的list
//DEFAULTCAPACITY_EMPTY_ELEMENTDATA 为0.初始化为10，也就是说初始其实是空数组 当添加第一个元素的时候数组容量才变成10
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
//带初始容量参数的构造函数。（用户自己指定容量）
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
//构造一个包含指定集合的元素的列表，按照它们由集合的迭代器返回的顺序。
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
	//如果指定集合元素个数不为0
    if ((size = elementData.length) != 0) {
        //c.toArray 可能返回的不是Object类型的数组所以加上下面的语句用于判断，
		//这里用到了反射里面的getClass()方法
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        //用空数组代替
        this.elementData = EMPTY_ELEMENTDATA;
    }
}

```

### 2.1.3 add

```Java

//将指定的元素追加到此列表的末尾
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
	//这里看到ArrayList添加元素的实质就相当于为数组赋值
    elementData[size++] = e;
    return true;
}

//在此列表中的指定位置插入指定的元素
public void add(int index, E element) {
	//对index进行界限检查
    rangeCheckForAdd(index);
	//保证capacity足够大
    ensureCapacityInternal(size + 1);  // Increments modCount!!
	//将从index开始之后的所有成员后移一个位置
    System.arraycopy(elementData, index, elementData, index + 1,size - index);
	//将element插入index位置
    elementData[index] = element;
	//size加1
    size++;
}

//按指定集合的Iterator返回的顺序将指定集合中的所有元素追加到此列表的末尾
public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
    return numNew != 0;
}

//将指定集合中的所有元素插入到此列表中，从指定的位置开始
public boolean addAll(int index, Collection<? extends E> c) {
    rangeCheckForAdd(index);

    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount

    int numMoved = size - index;
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew,numMoved);

    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}

```

## 2.2 System.arraycopy()和Arrays.copyOf()方法

```Java

public static native void arraycopy(
	Object src, int srcPos, Object dest, int destPos, int length);

public static <T> T[] copyOf(T[] original, int newLength) {
    return (T[]) copyOf(original, newLength, original.getClass());
}
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}

```

联系：

- 实际上，copyOf()内部调用了System.arraycopy()方法

区别：

- arraycopy()需要目标数组，将原数组拷贝到你自己定义的数组里，而且可以选择拷贝的起点和长度以及放入新数组中的位置
- copyOf()是系统自动在内部新建一个数组，并返回该数组。


## 2.3 ArrayList 核心扩容技术

ArrayList的扩容机制提高了性能，如果每次只扩充一个，那么频繁的插入会导致频繁的拷贝，降低性能，而ArrayList的扩容机制避免了这种情况。

```Java

public void ensureCapacity(int minCapacity) {
    int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
        // any size if not default element table
        ? 0
        // larger than default for default empty table. It's already
        // supposed to be at default size.
        : DEFAULT_CAPACITY;

    if (minCapacity > minExpand) {
        ensureExplicitCapacity(minCapacity);
    }
}


private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

//得到最小扩容量
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
		//获取默认的容量和传入参数的较大值
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

//判断是否需要扩容
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    //如果说minCapacity也就是所需的最小容量大于保存ArrayList数据的数组的长度的话，就需要调用grow(minCapacity)方法扩容。
	//这个minCapacity到底为多少呢？举个例子在添加元素(add)方法中这个minCapacity的大小就为现在数组的长度加1
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

//ArrayList扩容的核心方法
private void grow(int minCapacity) {
    //elementData为保存ArrayList数据的数组
	//elementData.length求数组长度
    int oldCapacity = elementData.length;
	//oldCapacity为旧容量，newCapacity为新容量,将新容量更新为旧容量的1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
	//检查新容量是否大于最小需要容量，若还是小于最小需要容量，那么就把最小需要容量当作数组的新容量
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
	//检查新容量是否超出了ArrayList所定义的最大容量
	//若超出了，则调用hugeCapacity()来比较minCapacity和 MAX_ARRAY_SIZE
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
	//如果minCapacity大于MAX_ARRAY_SIZE，则新容量则为Interger.MAX_VALUE，否则，新容量大小则为 MAX_ARRAY_SIZE
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}

```

扩容时用到移位运算符，仅移动位置,不去计算,这样提高了效率,节省了资源。

注意：

- java 中的length 属性是针对数组说的,比如说你声明了一个数组,想知道这个数组的长度则用到了 length 这个属性.
- java 中的length()方法是针对字 符串String说的,如果想看这个字符串的长度则用到 length()这个方法.
- java 中的size()方法是针对泛型集合说的,如果想看这个泛型有多少个元素,就调用此方法来查看!


# 3. 内部类

```Java

public Iterator<E> iterator() {...}

private class ListItr extends Itr implements ListIterator<E> {...}

private class SubList extends AbstractList<E> implements RandomAccess {...}

static final class ArrayListSpliterator<E> implements Spliterator<E> {...}

```

- Itr是实现了Iterator接口，同时重写了里面的hasNext()，next()，remove()等方法
- ListItr继承Itr，实现了ListIterator接口，同时重写了hasPrevious()，nextIndex()，previousIndex()，previous()，set(E e)，add(E e)等方法

区别：ListIterator在Iterator的基础上增加了添加对象，修改对象，逆向遍历等方法，这些是Iterator不能实现的。