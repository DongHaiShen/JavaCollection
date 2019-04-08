[TOC]

## Vector

### 一、顶部注释分析

+ Vector是一个 growable 的数组，它的大小可以根据需要增加或减少
+ Vector is synchronized. If a thread-safe implementation is not needed, it is recommended to use ArrayList in place of Vector，即Vector 和 ArrayList 的最大的不同是 Vector 是**线程安全的**，但是如果不需要线程安全，还是推荐使用 ArrayList 提高效率



### 二、源码分析

#### 2.1 定义

+ `public class Vector<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable`

+ 与 ArrayList 相同，继承自 AbstractList，实现 List 接口，可随机访问，可 clone，可序列化


#### 2.2 字段

```java
// 保存vector中元素的数组
// vector的容量是数组的长度，数组的长度最小值为vector的元素个数
protected Object[] elementData;

// vector中实际的元素个数
protected int elementCount;

/** 
 * vector需要自动扩容时增加的容量
 * 如果capacityIncrement小于或等于0，vector的容量需要增长时将会成倍增长。
 */
protected int capacityIncrement;

// 分派给vector的最大容量
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

#### 2.3 构造方法

1. `Vector(int initialCapacity, int capacityIncrement)`：构造一个指定容量为capacity、自增容量为capacityIncrement 的空vector
2. `Vector(int initialCapacity)`：构造一个指定容量为initialCapacity、自增容量为0的空vector
3. `Vector()`：调用方法2 构造一个指定容量为10、自增容量为0的空vector，即 `Vector(10)`
4. `Vector(Collection<? extends E> c)`：使用指定的 Collection 构造vector


#### 2.4 常用方法

+ Vector的常用方法实现与 ArrayList 有很多相似之处，如 add、set、get、remove等，最大的区别在于 Vector 中大多数方法都是线程安全的，即使用了 synchronized 关键字，以 add 为例：

```java
public synchronized boolean add(E e) 
{
	modCount++;
	ensureCapacityHelper(elementCount + 1);
	elementData[elementCount++] = e;
	return true;
}
```

#### 2.5 扩容

1. 该方法的实现和 ArrayList 中大致相同，不同的是在第一次扩容时，vector的逻辑是： 
```java
newCapacity = oldCapacity + ((capacityIncrement > 0) ? capacityIncrement : oldCapacity); 
```

2. 即如果 capacityIncrement > 0，就加 capacityIncrement，如果不是就**增加一倍**；
3. ArrayList是即增加现有的一半

```java
private void grow(int minCapacity) 
{
// overflow-conscious code
	int oldCapacity = elementData.length;
	int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
 									capacityIncrement : oldCapacity);
	if (newCapacity - minCapacity < 0)
		newCapacity = minCapacity;
	if (newCapacity - MAX_ARRAY_SIZE > 0)
		newCapacity = hugeCapacity(minCapacity);
	elementData = Arrays.copyOf(elementData, newCapacity);
}
```
























