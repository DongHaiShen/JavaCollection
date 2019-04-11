[TOC]

## 一、顶部注释分析

### 1.1 首句定义

+ Doubly-linked list implementation of the List and Deque interfaces
+ LinkedList 是 List 接口和 Deque 接口的**双向链表**实现



### 1.2 从注释中得到的结论

+ `permits all elements (including null)`：LinkList **允许null元素**
+ `All of the operations perform as could be expected for a doubly-linked`：所有的**操作都针对双向链表**
- `Note that this implementation is not synchronized`：LinkedList**不是线程安全**的
- `Collections.synchronizedList`方法可以实现线程安全的操作
- 由 iterator() 和 listIterator() 返回的迭代器是 `fail-fast` 的



## 二、源码分析

### 2.1 定义

```java
public class LinkedList<E> extends AbstractSequentialList<E>
		implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

+ LinkedList\<E\>：说明它支持泛型。
+ extends **AbstractSequentialList**\<E\>：继承自 AbstractSequentialList， 只支持按次序访问
+ implements **List**\<E\>：实现了 List 接口
+ implements **Deque**\<E\>：Deque，Double ended queue，双端队列。LinkedList可用作队列或双端队列就是因为实现了它。
+ implements **Cloneable**：可以调用 clone() 方法来返回实例的 `field-for-field` 拷贝
+ implements **java.io.Serializable**：具有序列化功能
+ **没有实现 RandomAccess 接口**，因此不像 ArrayList 一样支持快速随机访问



### 2.2 字段

```java
// LinkedList节点个数
transient int size = 0;

// 指向头节点的指针
transient Node<E> first;

// 指向尾节点的指针
transient Node<E> last;

// Node类定义
private static class Node<E> 
{
	E item;
	Node<E> next;
	Node<E> prev;

	Node(Node<E> prev, E element, Node<E> next) 
	{
		this.item = element;
		this.next = next;
		this.prev = prev;
	}
}
```



### 2.3 构造方法

1. `public LinkedList() {}`：构造一个空链表
2. `public LinkedList(Collection<? extends E> c)`：根据指定集合构造LinkedList。步骤为先构造一个空Linkedlist，再把指定集合中的所有元素都添加到LinkedList中



### 2.4 添加元素

+ add 方法实际上会调用 linkLast 方法，即在链表最后添加元素

```java
public boolean add(E e) 
{
	linkLast(e);
	return true;
}

void linkLast(E e) 
{
    // 使节点l指向原来的尾结点
	final Node<E> l = last; 
	
	//新建节点newNode，节点的前指针指向l，后指针为null
	final Node<E> newNode = new Node<>(l, e, null); 
    
    // 尾指针指向新的头节点newNode
	last = newNode;
    
    // 如果原来的尾结点为null，更新头指针，否则将原来尾结点的后指针指向新节点newNode，构成双向
	if (l == null)
		first = newNode;
	else
		l.next = newNode;
	size++;
	modCount++;
}
```



### 2.5 get、set、remove

+ 三者中都调用了 `node(index)` 方法，用于返回在指定索引处的非空元素
+ 在查找时有一个优化点是：**若 index 小于总长度的一半，则从头开始遍历；否则从尾开始遍历**

```java
Node<E> node(int index) 
{
	// assert isElementIndex(index);

	if (index < (size >> 1)) 
	{
		Node<E> x = first;
		for (int i = 0; i < index; i++)
			x = x.next;
		return x;
	} 
	else 
	{
		Node<E> x = last;
		for (int i = size - 1; i > index; i--)
			x = x.prev;
		return x;
	}
}
```



+ get先检测范围，然后找到指定元素

```java
public E get(int index) 
{
	checkElementIndex(index);
	return node(index).item;
}
```



+ set先检测范围，然后找到指定元素替换，并返回旧值

```java
public E set(int index, E element) 
{
	checkElementIndex(index);
	Node<E> x = node(index);
	E oldVal = x.item;
	x.item = element;
	return oldVal;
}
```



+ remove先检测范围，然后找到指定元素更新连接，并返回旧值

```java
public E remove(int index) {

	checkElementIndex(index);
	return unlink(node(index));
}

E unlink(Node<E> x) 
{
	// assert x != null;
	final E element = x.item;
	final Node<E> next = x.next;
	final Node<E> prev = x.prev;

	if (prev == null) 
	{
		first = next;
	} 
	else 
	{
		prev.next = next;
		x.prev = null;
	}

	if (next == null)
    {
		last = prev;
	} 
	else 
	{
		next.prev = prev;
		x.next = null;
	}

	x.item = null;
	size--;
	modCount++;
	return element;
}
```



### 2.6 队列操作（添加、删除、获取）

+ 总结：
    1. 常用的插入、删除、获取方法都存在两种形式：在操作失败时，一种抛出异常，另一种返回一个特殊值（null 或 false，具体取决于操作）
    2. 其中插入操作的 offer 方法是用于专门为有容量限制的 Queue 实现设计的



+ 添加元素：offer 和 add，实际上 offer 内部就是调用了 add ()，但是两者的区别为：

| 方法  |            操作失败时的处理方式             |      来源      |
| :---: | :-----------------------------------------: | :------------: |
|  add  | 抛出异常，如队列满抛出IllegalStateException | Collection接口 |
| offer |         不抛出异常，只是返回 false          |   Queue接口    |

```java
public boolean offer(E e) 
{
	return add(e);
}

public boolean add(E e) 
{
	linkLast(e);
	return true;
}
```



+ 删除**头部**元素：poll 和 remove，两者最终调用的都是 `unlinkFirst(f)`，但是两者的区别为：

|  方法  |            操作失败时的处理方式            |
| :----: | :----------------------------------------: |
| remove | 若头部元素为空会抛出NoSuchElementException |
|  poll  |         不抛出异常，只是返回 null          |

```java
public E poll() 
{
	final Node<E> f = first;
	return (f == null) ? null : unlinkFirst(f);
}

public E remove() 
{
	return removeFirst();
}

public E removeFirst() 
{
	final Node<E> f = first;
	if (f == null)
		throw new NoSuchElementException();
	return unlinkFirst(f);
}
```



+ 获取头部元素：element 和 peek，两者最终都是访问 `f.item`，但是两者的区别为：

|  方法   |            操作失败时的处理方式            |
| :-----: | :----------------------------------------: |
| element | 若头部元素为空会抛出NoSuchElementException |
|  peek   |         不抛出异常，只是返回 null          |

```java
public E peek() 
{
	final Node<E> f = first;
	return (f == null) ? null : f.item;
}

public E element() 
{
	return getFirst();
}

public E getFirst() 
{
	final Node<E> f = first;
	if (f == null)
		throw new NoSuchElementException();
	return f.item;
}
```



### 2.7 栈操作（入栈、出栈、获取栈顶）

+ 具体实现方式和队列类似，当利用LinkedList实现栈时使用

```java
public void push(E e) 
{
	addFirst(e);
}

public E pop() 
{
	return removeFirst();
}

public E peek() 
{
	final Node<E> f = first;
	return (f == null) ? null : f.item;
}
```




