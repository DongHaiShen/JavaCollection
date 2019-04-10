[TOC]

## 一、顶部注释分析

### 1.1 数据结构

![HashMap](https://ws3.sinaimg.cn/large/006oCwEfly1g1xghjaxycj30j60evwfm.jpg)

+ HashMap的数据结构是 **数组+链表+红黑树** (JDK1.8)
+ 数组中的每一个节点可称为是**桶**。当向 HashMap 中添加一个键值对时，首先计算键值对中key的hash值，以此确定数组中对应的桶，但是可能存在同一hash值的元素已经被放在该桶中，这种现象称为 **hash碰撞**
+ 这时按照**尾插法** (JDK1.8，JDK1.7及以前为头插法) 的方式把键值对添加到同一个桶的后面，从而形成链表
+ 当链表长度超过8 (TREEIFY_THRESHOLD)时，链表就转换为 [红黑树](https://blog.csdn.net/chen_zhang_yu/article/details/52415077)



### 1.2 从注释中得到的结论

+ **底层**：`HashMap` 是 `Map` 接口基于哈希表的实现。
+ **是否允许null**：key 和 value **都允许为 null**
+ **是否有序**：不保证映射的顺序，特别是它不保证顺序恒久不变
+ **何时rehash**：超出当前允许的最大容量。`initial capacity * load factor` 就是当前允许的最大元素数目，超过该值之后就会进行 rehash 操作来进行扩容，**扩容后的的容量为之前的两倍**
+ **初始化容量对性能的影响**：设置太小或太大都不好：小了虽然节省空间但会频繁 rehash 增加时间开销；大了增加空间开销同时影响遍历效率
+ **加载因子对性能的影响**：同理设置太小或太大都不好：小了会频繁 rehash 增加时间开销；大了会影响遍历效率也增加时间开销，**0.75**是个折中的选择
+ **是否同步**：HashMap不是同步的，即非线程安全，可以使用 `Collections.synchronizedMap` 进行同步
+ **迭代器**：返回的迭代器是 `fail-fast` 的
+ **与 Hashtable 的区别**：HashMap 除了是非同步和允许 null 值外，和 Hashtable 大致相同 （The HashMap class is roughly equivalent to Hashtable, except that it is unsynchronized and permits nulls）



## 二、源码分析

### 2.1 定义

```java
public class HashMap<K,V> extends AbstractMap<K,V> 
		implements Map<K,V>, Cloneable, Serializable
```

+ HashMap<K,V>：HashMap 是以 key-value 形式存储数据的
+ extends **AbstractMap**<K,V>：继承自 AbstractMap，大大减少了实现 Map 接口时需要的工作量
+ implements **Map**<K,V>：实现了Map接口，提供了所有可选的 Map 操作
+ implements **Cloneable**：可以调用 clone() 方法来返回实例的 `field-for-field` 拷贝
+ implements **Serializable**：可以序列化



### 2.2 字段

```java
// 默认初始化容量，值为16
// 该值必须是2的n次幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// 最大容量, 如果一个更大的初始化容量在构造函数中被指定，将被MAXIMUM_CAPACITY替换
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认加载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 当添加一个元素被添加到有至少TREEIFY_THRESHOLD个节点的桶中，桶中的链表将被转化为树形结构
static final int TREEIFY_THRESHOLD = 8;

// 当桶中的节点数小于等于该值，把树结构恢复成链式结构
static final int UNTREEIFY_THRESHOLD = 6;

// 当哈希表的大小超过这个阈值，才会把链式结构转化成树型结构，否则仅采取扩容方式
static final int MIN_TREEIFY_CAPACITY = 64;

// 存储键值对的数组
transient Node<K,V>[] table;

// 维护entrySet()的缓存
transient Set<Map.Entry<K,V>> entrySet;

// 键值对实际个数
transient int size;

// 扩容的临界值，可通过 capacity * load factor 计算得到
int threshold;

// 加载因子
final float loadFactor;
```



### 2.3 Node静态内部类

```java
static class Node<K,V> implements Map.Entry<K,V>
{
	// hash值，键值对以及指向下个键值对的指针，形成链表
	final int hash;
	final K key;
	V value;
	Node<K,V> next;

	Node(int hash, K key, V value, Node<K,V> next) 
	{
		this.hash = hash;
		this.key = key;
		this.value = value;
		this.next = next;
	}
}
```



### 2.4 构造方法

1. `public HashMap(int initialCapacity, float loadFactor)`：使用指定的初始化容量和加载因子构造一个空 HashMap
2. `public HashMap(int initialCapacity)`：使用指定的初始化容量和默认加载因子 (0.75) 构造一个空 HashMap
3. `public HashMap()`：使用默认的初始化容量 (16) 和加载因子 (0.75) 构造一个空 HashMap
4. `public HashMap(Map<? extends K, ? extends V> m)`：使用指定 Map 构造新的HashMap。初始化容量和加载因子均为默认值



### 2.5 threshold 赋值

+ 在构造方法中 threshold 的初始化为：`this.threshold = tableSizeFor(initialCapacity);`
+ `tableSizeFor` 方法的作用：**返回一个大于等于输入值且为2的整数次幂的数**。如输入10，返回16
+ 此处只是给 threshold 一个初始化值，在后续使用中会**重新赋值为 capacity * load factor**

```java
// 返回一个大于等于输入值且为2的整数次幂的数
static final int tableSizeFor(int cap) 
{
	// 减1操作使得该方法可以得到等于原值的数，主要是针对原先就是2的整数次幂的数
    // 例如8在不减1的情况下会返回16，减1后会返回8
    int n = cap - 1;
	n |= n >>> 1;
	n |= n >>> 2;
	n |= n >>> 4;
	n |= n >>> 8;
	n |= n >>> 16;
	return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```



### 2.6 hash值计算

#### 2.6.1 计算方式

+ 在 HashMap 中，经常需要计算 key 的 hash 值来定位具体的桶
+ 定位方式为：`table[(n-1) & hash]`，其中 n 为容量，hash 为 key 的哈希值
+ hash值的计算方式如下，可分为两个步骤：
    1. 取 key 的 hashCode，调用了Object类的 hashCode 方法，这是一个 native 方法；
    2. 将 hashCode 的**高16位和低16位做异或运算**；

```java
static final int hash(Object key) 
{
	int h;
	return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

![hash运算](https://ws3.sinaimg.cn/large/006oCwEfly1g1xghj9vwkj30ii094jsh.jpg)

#### 2.6.2 计算原理

+ 这里有一个问题，即**为什么要对 key 的hashCode 做异或运算**？
+ 原因在于：
    1. 上述提到在定位时是通过 `(n-1) & hash` 的方式；
    2. 若 n 很小，如初始值16，则显然做 & 操作时**只有后4位有效**；
    3. 因此如果当哈希值的**高位变化很大，低位变化很小**，这样很容易造成碰撞；
    4. 因此设计者**将哈希值的高低位也参与到计算中**，增加了随机性，减少了碰撞冲突的可能性



### 2.7 resize 扩容

+ 向HashMap对象里不停的添加元素，当其内部数组无法装载更多的元素时，就需要进行扩容；
+ 数组是无法自动扩容的，扩容方法是使用一个新数组代替已有数组；
+ resize 方法非常巧妙：因为每次扩容都是翻倍，与原来位置`(n-1) & hash` 的结果相比，节点要么就在原来的位置，要么就被分配到 “原位置+旧容量” 这个位置
+ 大致步骤如下：
    1. 如果旧容量已经超过最大阈值，则无法继续扩容，返回旧 HashMap；
    2. 否则将容量阈值**加倍**，即 `newThr = oldThr << 1`；
    3. 根据新容量阈值新建数组，将 HashMap 的 table 引用指向新数组；
    4. 将旧 HashMap 的元素复制到新HashMap中，需要根据结构为树还是链表选取不同的方法；
    5. 最后返回新 HashMap

```java
final Node<K,V>[] resize()
```



### 2.8 put 操作

#### 2.8.1 put 方法
+ put 添加元素分三个步骤：
    1. 通过 `hash(Object key)`方法计算 key 的哈希值
    2. 通过 `putVal(hash(key), key, value, false, true)` 方法实现功能
    3. 返回 putVal 方法返回的结果

```java
public V put(K key, V value) 
{
	return putVal(hash(key), key, value, false, true);
}
```

#### 2.8.2 putVal 方法

```java
// onlyIfAbsent为true时，会替换已经存在的值
// evict为false时, the table is in creation mode
// 由于最后是返回当前key的旧值，因此中间需要保存key所对应的节点
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict)
```

执行步骤如下：

1. 如果哈希表为空，调用 resize() 创建一个哈希表；
2. 如果指定 key 值没有发生碰撞，直接加入哈希表即可；
3. 如果有碰撞，先找到 key 对应的桶：
    + 如果桶中的第一个节点就匹配了，则将其记录下来；
    + 如果桶中的第一个节点没有匹配，且桶中结构为红黑树，则调用红黑树对应的方法插入键值对；
    + 如果桶中结构为链表链表，则遍历：
        + 如果找到了key映射的节点，就记录这个节点并退出；
        + 如果没有找到，在链表**尾部插入**节点
        + 插入后，如果链的长度大于TREEIFY_THRESHOLD这个临界值，则把链表转为红黑树
4. 若找到了 key 所对应的节点，即该 key 本身就存在：
    + 记录节点的 value；
    + 如果参数 onlyIfAbsent 为false，或者 oldValue 为null，替换value，否则不替换；
    + 返回记录的 value；
5. 如果没有找到 key 对应的节点，即该 key 本身不存在：
    + 那么插入节点后 size 会加1，同时检查size是否大于临界值threshold，如果大于需要使用 resize 方法进行扩容，最后返回 null



### 2.9 get 操作

#### 2.9.1 get 方法

+ get 获取元素分三个步骤：
    1. 通过 `hash(Object key)`方法计算 key 的哈希值 hash；
    2. 通过 `getNode(int hash, Object key)` 方法获取node；
    3. 如果 node 为 null，返回null，否则返回 node.value

```java
public V get(Object key) 
{
	Node<K,V> e;
	return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

#### 2.9.2 getNode 方法

```java
// 该方法根据key的hash值和具体key值获取对应节点
final Node<K,V> getNode(int hash, Object key)
```

执行步骤如下：

1. 如果哈希表为空，或key对应的桶为空，直接返回null；
2. 如果桶中的第一个节点就和指定参数hash和key匹配，直接返回这个节点；
3. 如果桶中的第一个节点没有匹配上，而且有后续节点： 
    + 如果当前的桶结构为红黑树，则调用红黑树的 getTreeNode 方法去获取节点；
    + 如果当前的桶结构为链表，则遍历直到 key 匹配或遍历结束；
4. 最终找到节点则返回，否则返回null



### 2.10 remove 操作

#### 2.10.1 remove方法

+ remove 删除元素分三个步骤：
    1. 通过 `hash(Object key)` 方法计算key的哈希值；
    2. 通过 `removeNode` 方法实现功能；
    3. 返回被删除的 node 的 value

```java
public V remove(Object key) 
{
	Node<K,V> e;
	return (e = removeNode(hash(key), key, null, false, true)) == null ? null : e.value;
}
```

#### 2.10.2 removeNode方法

```java
// mathValue为true时，则必须value也相等才会删除
// movable为false时，删除该节点不会移动其他节点，主要针对红黑树结构
final Node<K,V> removeNode(int hash, Object key, Object value, boolean matchValue, 									boolean movable) 
```

执行步骤如下：

1. 如果数组 table 为空或 key 映射到的桶为空，返回null；
2. 如果桶中的第一个节点就和指定参数hash和key匹配，记录这个节点；
3. 如果桶中的第一个节点没有匹配上，而且有后续节点： 
    + 如果当前的桶结构为红黑树，则调用红黑树的 getTreeNode 方法去获取节点；
    + 如果当前的桶结构为链表，则遍历直到 key 匹配或遍历结束；
4. 如果被记录下来的node不为null，且value也匹配：
    + 根据桶结构调用对应方法删除节点；
    + size减1，返回被删除的节点

### 2.11 entrySet 说明

+ 在针对 HashMap 遍历时会调用 `map.entrySet()` 方法来获取一个**集合视图**，再进行后续操作

```java
// 返回HashMap中所有键值对的set视图
public Set<Map.Entry<K,V>> entrySet() 
{
	Set<Map.Entry<K,V>> es;
	return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
}
```

+ 其中的 entrySet 实际上为 HashMap 的属性，但在 put() 操作时并没有对 entrySet 进行修改，**那么它是如何变化的**？
+ 实际上利用了 `entrySet = new EntrySet()` 这一操作，其中 EntrySet 是 HashMap 中的内部类，其中有一个关键方法 **forEach**
+ 从代码中可以看出：
    1. 在进行 foreach 遍历 EntrySet 的时候，**实际上是遍历 table**，即 HashMap 中存储实际数据的数组；
    2. 这也进一步验证了 entrySet() 方法返回的是一个集合视图：**视图的概念类似于数据库**，即视图没有具体的数据，真正获取数据时还是从 table 中得到

```java
final class EntrySet extends AbstractSet<Map.Entry<K,V>> 
{
	public final void forEach(Consumer<? super Map.Entry<K,V>> action) 
	{
		Node<K,V>[] tab;
		if (action == null)
			throw new NullPointerException();
		if (size > 0 && (tab = table) != null) 
		{
			int mc = modCount;
			for (int i = 0; i < tab.length; ++i) // 实际上是在遍历table数组 
			{
				for (Node<K,V> e = tab[i]; e != null; e = e.next)
                	action.accept(e);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
	}
}
```



## 三、与Hashtable对比

+ HashMap 与 Hashtable 从存储结构和实现来讲基本上都是相同的
+ 两者最大的不同是 Hashtable 是线程安全的，且它不允许 key 和 value 为null（**两个都不允许为null**）
+ Hashtable是个**过时**的集合类，不建议在新代码中使用，在不需要线程安全的场合可以用 HashMap 替换，需要线程安全的场合可以用 ConcurrentHashMap 替换



|         区别         |      HashMap       |               Hashtable               |
| :------------------: | :----------------: | :-----------------------------------: |
|       数据结构       |  数组+链表+红黑树  |               数组+链表               |
|       继承的类       | 继承自 AbstractMap |         继承自继承 Dictionary         |
|    默认初始化容量    |         16         |                  11                   |
|       扩容方式       |    原始容量 x2     |            原始容量 x2 + 1            |
|       容量要求       | 必须为2的整数次幂  |                不要求                 |
|     计算索引位置     |  `(n - 1) & hash`  |  `(hash & 0x7FFFFFFF) % tab.length`   |
|       遍历方式       |  Iterator(迭代器)  | Iterator(迭代器)和Enumeration(枚举器) |
| Iterator遍历数组顺序 |    索引从小到大    |             索引从大到小              |
|     是否线程安全     |         否         |                  是                   |
|       性能高低       |         高         |                  低                   |


















