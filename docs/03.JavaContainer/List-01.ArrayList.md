# ArrayList源码解读

[TOC]

## ArrayList简介

ArrayList底层是动态数组(Resizable-array), 它的元素可以是任意值，包括`null`。与`Vector`基本相同, `Vector`是线程安全的

## 继承体系



![](https://raw.githubusercontent.com/hyman213/FigureBed/master/2019/07/20190701224710.png)

ArrayList实现了List, RandomAccess, Cloneable, java.io.Serializable等接口。

ArrayList实现了List，提供了基础的添加、删除、遍历等操作。

ArrayList实现了RandomAccess，提供了随机访问的能力。

ArrayList实现了Cloneable，可以被克隆。

ArrayList实现了Serializable，可以被序列化。

## 源码解析

### 属性

```java
    /**
     * Default initial capacity.
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Shared empty array instance used for empty instances.
     * 空数组，如果传入的容量为0时使用
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     * 空数组，传传入容量时使用，添加第一个元素的时候会重新初始为默认容量大小
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * The size of the ArrayList (the number of elements it contains).
     * 真正储存元素的个数，而不是
     */
    private int size;
```

（1）DEFAULT_CAPACITY

默认容量为10，也就是通过new ArrayList()创建时的默认容量。

（2）EMPTY_ELEMENTDATA

空的数组，这种是通过new ArrayList(0)创建时用的是这个空数组。

（3）DEFAULTCAPACITY_EMPTY_ELEMENTDATA

也是空数组，这种是通过new ArrayList()创建时用的是这个空数组，与EMPTY_ELEMENTDATA的区别是在添加第一个元素时使用这个空数组的会初始化为DEFAULT_CAPACITY（10）个元素。

（4）elementData

真正存放元素的地方，使用transient是为了不序列化这个字段。

至于没有使用private修饰，后面注释是写的“为了简化嵌套类的访问”，但是楼主实测加了private嵌套类一样可以访问。

private表示是类私有的属性，只要是在这个类内部都可以访问，嵌套类或者内部类也是在类的内部，所以也可以访问类的私有成员。

（5）size

真正存储元素的个数，而不是elementData数组的长度。

### ArrayList构造方法

```java
// 传入初始容量
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        // 如果传入的初始容量大于0，就新建一个数组存储元素
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        // 如果传入的初始容量等于0，使用空数组EMPTY_ELEMENTDATA
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        // 如果传入的初始容量小于0，抛出异常
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

// 不传初始容量，初始化为DEFAULTCAPACITY_EMPTY_ELEMENTDATA空数组，会在添加第一个元素的时候扩容为默认的大小，即10。
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

// 传入集合并初始化elementData，这里会使用拷贝把传入集合的元素拷贝到elementData数组中，如果元素个数为0，则初始化为EMPTY_ELEMENTDATA空数组。
public ArrayList(Collection<? extends E> c) {
    // 集合转数组
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // c为空，初始化为EMPTY_ELEMENTDATA
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

### add(E e)方法

添加元素到末尾

```java
public boolean add(E e) {
	// 检查是否需要扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
	// 把元素插入到最后一位
    elementData[size++] = e;
    return true;
}


private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

// 计算需要的capacity, 通过ArrayList()构造的数组，在插入第一个元素时，初始化长度为DEFAULT_CAPACITY
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

// 计算具体的数组大小，判断是否需要扩容
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

// 扩容
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    // 新容量为旧容量的1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 如果新容量发现比需要的容量还小，则以需要的容量为准
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    // 如果新容量已经超过最大容量了，则使用最大容量
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // 以新容量拷贝出来一个新数组
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

### add(int index, E element)

添加元素到指定位置

```java

public void add(int index, E element) {
    // 检查是否越界
    rangeCheckForAdd(index);
    // 检查是否需要扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    // 将inex及其之后的元素往后挪一位
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    // 元素插入到index位置
    elementData[index] = element;
    // 大小增1
    size++;
}

private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

System.arraycopy
public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);
```

### addAll(Collection<? extends E> c)

求两个集合的并集

```java
// 将集合中的所有元素添加到数组末尾
public boolean addAll(Collection<? extends E> c) {
    // 集合转数组
    Object[] a = c.toArray();
    // 集合长度
    int numNew = a.length;
    // 检查是否需要扩容
    ensureCapacityInternal(size + numNew);  // Increments modCount
    // 将元素复制进elementData
    System.arraycopy(a, 0, elementData, size, numNew);
    // 长度加numNew
    size += numNew;
    return numNew != 0;
}
```

### get(int index)

```java
public E get(int index) {
    rangeCheck(index);
    return elementData(index);
}

private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

E elementData(int index) {
    return (E) elementData[index];
}
```

### remove(int index)

```java
public E remove(int index) {
    // 越界检查
    rangeCheck(index);

    modCount++;
    // 需要删除的元素
    E oldValue = elementData(index);
	// 需要移动的元素的个数
    int numMoved = size - index - 1;
    // 将index后面的元素向前移
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    // 最后一个元素删除
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

### remove(Object o)

```java
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

// 比remove(int index)少一个越界检查，而且没有返回值
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

### retainAll(Collection<?> c)

求两个集合的交集

```java
public boolean retainAll(Collection<?> c) {
    // 集合c不能为null
    Objects.requireNonNull(c);
    return batchRemove(c, true);
}

// Objects.requireNonNull
public static <T> T requireNonNull(T obj) {
	if (obj == null)
        throw new NullPointerException();
	return obj;
}

/**
* 批量删除元素
* complement为true表示删除c中不包含的元素
* complement为false表示删除c中包含的元素
*/
private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    // 使用读写两个指针同时遍历数组    
	// 读指针每次自增1，写指针放入元素的时候才加1    
	// 这样不需要额外的空间，只需要在原有的数组上操作就可以了
    int r = 0, w = 0;
    boolean modified = false;
    try {
        // 遍历整个数组，如果c中包含该元素，则把该元素放到写指针的位置（以complement为准）
        for (; r < size; r++)
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
        // Preserve behavioral compatibility with AbstractCollection,
        // even if c.contains() throws.
		// 正常来说r最后是等于size的，除非c.contains()抛出了异常
        if (r != size) {
            // 如果c.contains()抛出了异常，则把未读的元素都拷贝到写指针之后
            System.arraycopy(elementData, r,
                             elementData, w,
                             size - r);
            w += size - r;
        }
        if (w != size) {
            // clear to let GC do its work
			// 将写指针之后的元素置为空，帮助GC
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            // 新大小等于写指针的位置（因为每写一次写指针就加1，所以新大小正好等于写指针的位置）
            size = w;
            modified = true;
        }
    }
	// 有修改返回true
    return modified;
}
```

步骤解析：

（1）遍历elementData数组；

（2）如果元素在c中，则把这个元素添加到elementData数组的w位置并将w位置往后移一位；

（3）遍历完之后，w之前的元素都是两者共有的，w之后（包含）的元素不是两者共有的；

（4）将w之后（包含）的元素置为null，方便GC回收；

### removeAll(Collection c)

求两个集合的差集，只保留当前集合中不在c中的元素。与retainAll类似，batchRemove中implement参数传入false

```java
public boolean removeAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, false);
}
```

### 序列化&反序列化

```java
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException{
    // Write out element count, and any hidden stuff
    // 防止序列化期间有修改
    int expectedModCount = modCount;
    // 写出非transient非static属性（会写出size属性）
    s.defaultWriteObject();

    // Write out size as capacity for behavioural compatibility with clone()
    s.writeInt(size);

    // Write out all elements in the proper order.
    for (int i=0; i<size; i++) {
        s.writeObject(elementData[i]);
    }

    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}


private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    elementData = EMPTY_ELEMENTDATA;

    // Read in size, and any hidden stuff
    // 读入非transient非static属性（会读取size属性）
    s.defaultReadObject();

    // Read in capacity
    s.readInt(); // ignored

    if (size > 0) {
        // be like clone(), allocate array based upon size not capacity
        int capacity = calculateCapacity(elementData, size);
        SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, capacity);
        // 检查是否需要扩容
        ensureCapacityInternal(size);

        Object[] a = elementData;
        // Read in all elements in the proper order.
        for (int i=0; i<size; i++) {
            a[i] = s.readObject();
        }
    }
}
```

查看writeObject()方法可知，先调用s.defaultWriteObject()方法，再把size写入到流中，再把元素一个一个的写入到流中。

一般地，只要实现了Serializable接口即可自动序列化，writeObject()和readObject()是为了自己控制序列化的方式，这两个方法必须声明为private，在java.io.ObjectStreamClass#getPrivateMethod()方法中通过反射获取到writeObject()这个方法。

在ArrayList的writeObject()方法中先调用了s.defaultWriteObject()方法，这个方法是写入非static非transient的属性，在ArrayList中也就是size属性。同样地，在readObject()方法中先调用了s.defaultReadObject()方法解析出了size属性。

elementData定义为transient的优势，自己根据size序列化真实的元素，而不是根据数组的长度序列化元素，减少了空间占用。



## 总结

（1）ArrayList内部使用数组存储元素，当数组长度不够时进行扩容，每次加一半的空间（1.5倍），ArrayList不会进行缩容；

（2）ArrayList支持随机访问，通过索引访问元素极快，时间复杂度为O(1)；

（3）ArrayList添加元素到尾部极快，平均时间复杂度为O(1)；

（4）ArrayList添加元素到中间比较慢，因为要搬移元素，平均时间复杂度为O(n)；

（5）ArrayList从尾部删除元素极快，时间复杂度为O(1)；

（6）ArrayList从中间删除元素比较慢，因为要搬移元素，平均时间复杂度为O(n)；

（7）ArrayList支持求并集，调用addAll(Collection c)方法即可；

（8）ArrayList支持求交集，调用retainAll(Collection c)方法即可；

（9）ArrayList支持求单向差集，调用removeAll(Collection c)方法即可；

## 参考

- [彤哥读源码之集合篇](https://mp.weixin.qq.com/s/kpBAIRoMvqPzC-wfLELP3Q)
