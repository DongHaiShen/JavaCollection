[TOC]

## 一、顶部注释分析

### 1.1 首句分析

+ A Red-Black tree based NavigableMap implementation. The map is sorted according to the  Comparable natural ordering of its keys, or by a Comparator provided at map creation time, depending on which constructor is used.
+ TreeMap是基于 NavigableMap 的**红黑树**实现。
+ TreeMap根据**键的自然顺序**进行排序，或者根据创建map时提供的**Comparator**进行排序，使用哪种取决于使用哪个构造方法



### 1.2 从注释中得到的结论

+ 底层是红黑树，通过实现 NavigableMap 接口，使得可以根据 key 本身进行排序，或是根据构造时提供的 Comparator 进行排序 ，即 **TreeMap 是有序的**
+ TreeMap 的 key 不能为null
+ TreeMap提供时间复杂度为 **log(n)** 的 containsKey、get、put 、remove操作
+ TreeMap同样是非线程安全，其迭代器也是 `fail-fast` 的



## 二、源码分析

### 2.1 定义

```java
public class TreeMap<K,V> extends AbstractMap<K,V>
		implements NavigableMap<K,V>, Cloneable, java.io.Serializable
```

+ TreeMap<K,V>：TreeMap是以 key-value 形式存储数据的
+ extends **AbstractMap**<K,V>：继承自AbstractMap
+ implements **NavigableMap**：实现了NavigableMap，可以返回特定条件的最近匹配
+ implements **Cloneable**：可以调用 clone() 方法来返回实例的 `field-for-field` 拷贝
+ implements **Serializable**：可以序列化



### 2.2 字段

```java
// TreeMap的排序规则，如果为null，则根据键的自然顺序进行排序
private final Comparator<? super K> comparator;

// 红黑树根节点
private transient Entry<K,V> root;

// 红黑树大小
private transient int size = 0;

// 树节点构造，除了左右指针还包括指向父节点的指针，且为红黑树结构
static final class Entry<K,V> implements Map.Entry<K,V> 
{
	K key;
	V value;
	Entry<K,V> left;
	Entry<K,V> right;
	Entry<K,V> parent;
	boolean color = BLACK;
}
```



### 2.3 构造方法

1. `public TreeMap()`：使用 key 的自然排序构造一个空的 TreeMap
2. `public TreeMap(Comparator<? super K> comparator)`：使用给定的比较器构造一个空的 TreeMap
3. `public TreeMap(Map<? extends K, ? extends V> m)`：根据给定 Map 来构造 TreeMap，排序方式为key 的自然排序
4. `public TreeMap(SortedMap<K, ? extends V> m)`：根据给定 **SortedMap** (可排序的Map) 来构造 TreeMap，且排序方式与 SortedMap 相同



### 2.4 put 操作

+ put 方法共分为以下几个步骤：
  1. 若红黑树为空，且 key 不为空，则新建一棵红黑树；
  2. 根据是否有自定义的 Comparator 方式采用不同的方法遍历红黑树**查找 key 所在位置**；
  3. 第2步中若没有自定义 Comparator 方式，即根据 key 进行自然排序，则 **key 必须实现 Comparable 接口，确保是可比较的**；
  4. 如果找到 key 所在位置，则用新值替换原值，并返回原值；
  5. 否则创建新节点插入红黑树，并对树进行调整以保持平衡，最终返回 null



### 2.5 get 操作

+ get 方法共分为以下几个步骤：
  1. 调用 `getEntry` 查找对应的键值对，若找到则返回 value ，找不到返回 null
  2. 在 `getEntry` 内部根据是否有自定义 comparator 采用不同方式查找，总体逻辑相似，只是 key 之间的比较方式不同

```java
public V get(Object key) 
{
	Entry<K,V> p = getEntry(key);
	return (p==null ? null : p.value);
}

final Entry<K,V> getEntry(Object key) 
{
	// Offload comparator-based version for sake of performance
	if (comparator != null)
		return getEntryUsingComparator(key);
	if (key == null)
		throw new NullPointerException();
	@SuppressWarnings("unchecked")
	Comparable<? super K> k = (Comparable<? super K>) key;
	Entry<K,V> p = root;
	while (p != null) 
    {
		int cmp = k.compareTo(p.key);
		if (cmp < 0)
			p = p.left;
		else if (cmp > 0)
			p = p.right;
		else
			return p;
	}
	return null;
}
```



### 2.6 remove 操作

+ remove 方法共分为以下几个步骤：
  1. 先和 get 方法类似尝试获取对应键值对
  2. 若找不到则直接返回 null
  3. 若找到了则先保存对应的值，然后调用 `deleteEntry(p)` **删除该节点并调整红黑树**
  4. 最后返回保存的值

```java
public V remove(Object key) 
{
	Entry<K,V> p = getEntry(key);
	if (p == null)
		return null;

	V oldValue = p.value;
	deleteEntry(p);
	return oldValue;
}
```



### 2.7 entrySet

+ TreeMap 遍历是使用的是 EntryIterator 这个内部类
+ EntryIterator 大多数的实现均在其父类 PrivateEntryIterator 中
+ PrivateEntryIterator 中的 `lastReturned` 用于存放返回的节点，在调用 `nextEntry` 方法获取下一个节点的过程中，使用了 `successor` 方法
+ 该方法是从树结构中按顺序获取下一个节点。由于红黑树是有序的二叉树，因此可以采取类似**中序遍历**的方式，[剑指Offer上有类似的题目](https://www.nowcoder.com/ta/coding-interviews?query=%E4%BA%8C%E5%8F%89%E6%A0%91%E7%9A%84%E4%B8%8B%E4%B8%80%E4%B8%AA%E7%BB%93%E7%82%B9)
+ 大致思路为：
  1. 若当前节点的右子树不为空，则返回**右子树中最小节点**，即右子树的最左节点；
  2. 如果右子树为空，则不断向上回溯，直至当前节点是其父节点的左孩子

```java
class EntrySet extends AbstractSet<Map.Entry<K,V>> 
{
	public Iterator<Map.Entry<K,V>> iterator() 
	{
		return new EntryIterator(getFirstEntry());
	}
}

final class EntryIterator extends PrivateEntryIterator<Map.Entry<K,V>> 


abstract class PrivateEntryIterator<T> implements Iterator<T>
{
	Entry<K,V> lastReturned;
	
	final Entry<K,V> nextEntry() 
	{
    	Entry<K,V> e = next;
    	if (e == null)
			throw new NoSuchElementException();
    	if (modCount != expectedModCount)
			throw new ConcurrentModificationException();
    	next = successor(e);
    	lastReturned = e;
    	return e;
	}
}

// 按顺序获取树结构中的下一个节点，顺序类似于中序遍历
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) 
{
	if (t == null)
		return null;
	else if (t.right != null) 
    {
		Entry<K,V> p = t.right;
		while (p.left != null)
			p = p.left;
		return p;
	} 
    else 
    {
		Entry<K,V> p = t.parent;
		Entry<K,V> ch = t;
		while (p != null && ch == p.right) 
        {
			ch = p;
			p = p.parent;
		}
		return p;
	}
}
```



### 三、与HashMap对比

|          不同点          |     HashMap      | TreeMap |
| :----------------------: | :--------------: | :-----: |
|         数据结构         | 数组+链表+红黑树 | 红黑树  |
|         是否有序         |        否        |   是    |
|   是否实现NavigableMap   |        否        |   是    |
|    是否允许key为null     |        是        |   否    |
| 增删改查操作的时间复杂度 |       O(1)       | log(n)  |

