[TOC]

## 一、 顶部注释分析

### 1.1 首句定义

+ Resizable-array implementation of the `List` interface：`List` 接口的**大小可变**数组的实现



### 1.2 从注释中得到的结论

+ **底层**：`ArrayList` 是 `List` 接口的大小可变数组的实现，即 `implements List<E>`
+ **是否允许null**：ArrayList **允许null 元素**
+ **时间复杂度**：size、isEmpty、get、set、iterator 和listIterator方法都以固定时间运行，时间复杂度为O(1)；而 add 和 remove 方法需要 O(n) 时间
+ **容量**：ArrayList 的容量可以**自动增长**
+ **是否同步**：ArrayList **不是同步的**，即线程不安全
+ **迭代器**：ArrayList 的 iterator 和 listIterator 方法返回的迭代器是 `fail-fast` 的，若修改会抛出 `ConcurrentModificationException` 异常


## 二、源码分析

### 2.1 定义

```java
public class ArrayList<E> extends AbstractList<E> 
		implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

+ ArrayList\<E\>：说明支持泛型
+ extends **AbstractList**\<E\> ：继承自 AbstractList
+ implements **List**\<E\>：实现了List接口
+ implements **RandomAccess**：支持快速（通常是固定时间）随机访问。此接口的主要目的是允许一般的算法更改其行为，从而在将其应用到随机或连续访问列表时能提供良好的性能。
+ implements **Cloneable**：可以调用 clone() 方法来返回实例的 `field-for-field` 拷贝
+ implements **java.io.Serializable**：具有序列化功能



### 2.2 字段

```java
// 默认初始化容量10
private static final int DEFAULT_CAPACITY = 10;

// 当指定容量为0时，返回该共享空数组
private static final Object[] EMPTY_ELEMENTDATA = {};

// 调用无参构造方法时，返回该默认数组
// 与EMPTY_ELEMENTDATA的区别：该值是在默认构造时返回，而EMPTY_ELEMENTDATA是在用户指定容量为0时返回
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

/**
 * 保存添加到ArrayList中的元素
 * ArrayList的容量就是该数组的长度
 * 当第一次有元素进入ArrayList中时，数组将扩容为DEFAULT_CAPACITY
 * 被标记为transient，在对象被序列化的时候不会被序列化
 */
transient Object[] elementData; // non-private to simplify nested class access

// ArrayList的实际大小（数组包含的元素个数）
private int size;

/**
 * 分派给arrays的最大容量
 * 减去8是因为某些VM会在数组中保留一些头字，尝试分配这个最大存储容量，可能会导致array容量大于VM的limit，最终导致OutOfMemoryError
 */
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```



### 2.3 构造方法

1. `ArrayList(int initialCapacity)`：构造一个指定容量为capacity的空ArrayList，容量为0则`this.elementData = EMPTY_ELEMENTDATA`，不为0则新建 `new Object[initialCapacity]`
2. `ArrayList()`：构造一个空数组，`this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA`
3. `ArrayList(Collection<? extends E> c)`：构造一个包含指定 collection 的元素的列表，这些元素是按照该 collection 的迭代器返回它们的顺序排列的。



### 2.4 添加元素

+ 添加元素分两个步骤：
  1. 空间检查，如果有需要进行扩容；
  2. 插入元素

```java
public boolean add(E e) 
{
	ensureCapacityInternal(size + 1);  // 容量检查，Increments modCount!!
	elementData[size++] = e;
	return true;
}
```



### 2.5 容量检查和扩容

1. 若 `elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA` ，则取minCapacity为默认容量和传入容量参数之间的最大值；
2. 否则执行 3 进行明确的容量检查；
3. 如果所需最小容量比当前数组长度还要大，则扩容；
4. 第一次扩容为 `newCapacity = oldCapacity + (oldCapacity >> 1)`，即**扩容为1.5倍**；
5. 如果一次扩容后还不够则直接扩充到所需的 minCapacity；
6. 如果容量超过 MAX_ARRAY_SIZE 则扩容至 MAX_ARRAY_SIZE 或抛出溢出的异常；
7. 最后把原有数组复制到新容量的数组中

```java
private void ensureCapacityInternal(int minCapacity) 
{
	if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) 
	{
		minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
	}

	ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) 
{
	modCount++;

	// 所需最小容量比当前数组长度还要大，需要扩容
	if (minCapacity - elementData.length > 0)
		grow(minCapacity);
}

private void grow(int minCapacity) 
{
	// overflow-conscious code
	int oldCapacity = elementData.length;
	int newCapacity = oldCapacity + (oldCapacity >> 1);
	if (newCapacity - minCapacity < 0)
		newCapacity = minCapacity;
	if (newCapacity - MAX_ARRAY_SIZE > 0)
		newCapacity = hugeCapacity(minCapacity);
	// minCapacity is usually close to size, so this is a win:
	elementData = Arrays.copyOf(elementData, newCapacity);
}
```



### 2.6 get、set、remove、trimToSize

+ 三者首先都需要进行范围检查

```java
private void rangeCheck(int index) 
{
	if (index >= size)
		throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```



+ 范围检查通过后，get 直接返回元素

```java
public E get(int index) 
{
	rangeCheck(index);

	return elementData(index);
}

E elementData(int index) 
{
	return (E) elementData[index];
}
```



+ set 用新值覆盖旧值，并返回旧值

```java
public E set(int index, E element) 
{
	rangeCheck(index);

	E oldValue = elementData(index);
	elementData[index] = element;
    return oldValue;
}
```



+ remove 删除指定元素，并将该元素之后的元素向左移动，并修改 size，最后返回旧值
+ 将索引为 size-1 处的元素**显示赋值为null，从而让GC能发挥作用**。若不手动赋null值，除非对应的位置被其他元素覆盖，否则原来的对象就一直不会被回收（深入理解 Java 虚拟机中有涉及）

```java
public E remove(int index) 
{
	rangeCheck(index);

	modCount++;
	E oldValue = elementData(index);

	int numMoved = size - index - 1;
	if (numMoved > 0)
		 System.arraycopy(elementData, index+1, elementData, index, numMoved);
	 elementData[--size] = null; // clear to let GC do its work

     return oldValue;
}
```



+ 删除元素时不会减少容量，若希望**减少容量**则调用 trimToSize()

```java
public void trimToSize() 
{
	modCount++;
	if (size < elementData.length) 
	{
		elementData = (size == 0)
  			? EMPTY_ELEMENTDATA
  			: Arrays.copyOf(elementData, size);
	}
}
```



### 2.7 modCount

+ modCount 继承于 AbstractList，它记录的是**集合的修改次数**，例如每次 add 或者 remove 它的值都会加1
+ 它的作用是当执行一些**迭代器操作**时，由于ArrayList是非线程安全的，因此每次执行时：
    1. 设 `expectedModCount = modCount`
    2. 执行迭代器 `next` 或 `remove` 等操作时，先调用checkForComodification来检查expectedModCount和modCount是否相等，若不等则抛出异常

```java
final void checkForComodification() 
{
    if (modCount != expectedModCount)
		throw new ConcurrentModificationException();
}
```

