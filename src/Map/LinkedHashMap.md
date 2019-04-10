[TOC]

## 一、顶部注释分析

### 1.1 数据结构

![LinkedHashMap](https://ws3.sinaimg.cn/large/006oCwEfly1g1xghjr9qwj30ni0j20xx.jpg)



### 1.2 从注释中得到的结论
+ LinkedHashMap 是 Map 接口的哈希表和链表的实现，具有可预知的迭代顺序
+ LinkedHashMap 和 HashMap的不同之处在于：它包含一个**贯穿于所有 entry 的双向链表**
+ 双向链表定义了迭代的顺序，默认是**插入顺序**。如果一个key被重插入，插入顺序不受影响
+ 与 HashMap 类似，初始化容量和加载因子对其性能影响很大，**但是迭代遍历时不受初始容量影响** [(2.8节)](#2.8-entrySet)
+ LinkedHashMap 也是非线程安全的
+ 由于LinkedHashMap 支持按访问顺序遍历，因此适合通过扩展实现 LRU 缓存 [(2.5节)](#2.5-迭代方式 accessOrder 的含义)



## 二、源码分析

### 2.1 定义

```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
```

+ LinkedHashMap<K,V>：以 key-value 形式存储数据
+ extends **HashMap**<K,V>：继承自HashMap，哈希表部分的功能和 HashMap 相似
+ implements **Map**<K,V>：实现了Map接口
+ HashMap 已经实现了Map接口，LinkedHashMap 虽然继承自 HashMap 但还是再次实现了 Map 接口，一般认为这样做的原因是可以直观地表达出 LinkedHashMap 实现了Map



### 2.2 字段

```java
// 双向链表头指针
transient LinkedHashMap.Entry<K,V> head;

// 双向链表尾指针
transient LinkedHashMap.Entry<K,V> tail;

// 迭代方式
// true代表按访问顺序迭代(access-order)
// false代表按插入顺序迭代(insertion-order)
final boolean accessOrder;
```



### 2.3 Entry 静态内部类

+ Entry 继承自 HashMap 的 Node，并且每个 entry 都包含前指针和后指针
+ 在构建新节点时，构建的是 `LinkedHashMap.Entry`，而 不再是 Node

```java
static class Entry<K,V> extends HashMap.Node<K,V> 
{
	Entry<K,V> before, after;
	Entry(int hash, K key, V value, Node<K,V> next) 
	{
		super(hash, key, value, next);
	}
}
```



### 2.4 构造方法

1. `public LinkedHashMap(int initialCapacity, float loadFactor)`：使用指定的初始化容量和加载因子构造一个空 LinkedHashMap
2. `public LinkedHashMap(int initialCapacity)`：使用指定的初始化容量和默认加载因子 (0.75) 构造一个空 LinkedHashMap
3. `public LinkedHashMap()`：使用默认的初始化容量 (16) 和加载因子 (0.75) 构造一个空 LinkedHashMap
4. `public LinkedHashMap(Map<? extends K, ? extends V> m)`：使用指定 Map 构造新的LinkedHashMap。初始化容量和加载因子均为默认值
5. 即构造方法与 HashMap 类似，只是多了一条 `accessOrder = false;`，即**默认迭代顺序为插入顺序**



### 2.5 迭代方式 accessOrder 的含义

+ 如果按**插入顺序**迭代，则符合我们平时的思维方式，即如果依次插入 1，2，3，则访问时顺序也为1，2，3
+ 如果按**访问顺序**迭代，则每次访问某个值后，会把该值放到链表的最后，即采用类似 LRU 的思想，把**最常用的放在链表的最后，不常用的放在链表的最前**
+ 因此顶部注释中有一句：`This kind of map is well-suited to building LRU caches`，即LinkedHashMap 适合实现 LRU 缓存



### 2.6 常用操作

#### 2.6.1 put

+ LinkedHashMap 本身并没有 put 方法，而是直接继承自 HashMap
+ 但是在 put 方法中创建新节点时，会调用 LinkedHashMap 重写后的 newNode 方法

```java
// 重写HashMap中的newNode方法，创建的是LinkedHashMap.Entry
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) 
{
	LinkedHashMap.Entry<K,V> p = new LinkedHashMap.Entry<K,V>(hash, key, value, e);
	linkNodeLast(p);
	return p;
}
```



#### 2.6.2 get

+ 获取方式与 HashMap 大致相同
+ 只是若迭代方式为按访问顺序迭代，则需要在 get 后把该节点放到链表的最后

```java
public V get(Object key) 
{
	Node<K,V> e;
	if ((e = getNode(hash(key), key)) == null)
		return null;
	if (accessOrder)
		afterNodeAccess(e);
	return e.value;
}

// 该方法实现将节点e放到链表最后
void afterNodeAccess(Node<K,V> e) // move node to last
```



#### 2.6.3 remove

+ LinkedHashMap 本身也没有 remove 方法，同样直接继承自 HashMap
+ 但是在 remove 中有一行 `afterNodeRemoval(node)`，此时同样会调用 LinkedHashMap 重写后的该方法



### 2.7 回调函数

+ 其实在 HashMap 中包含了如下三个空方法，它们大多出现在访问、插入、删除某个节点的方法中

```java
// Callbacks to allow LinkedHashMap post-actions
// 回调函数，允许LinkedHashMap进行后期操作

void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }
```

+ 但是在 HashMap 中均为空，即执行访问、插入、删除等操作后不进行其他修改
+ 而它们在 LinkedHashMap **都有重写**，LinkedHashMap正是通过重写这三个方法，在访问、插入、删除节点后**维护双向链表的有序性**
+ 例如执行 get 方法中若 accessOrder 为 true，则调用 `afterNodeAccess` 把该节点放到链表末尾
+ 执行 remove 方法中调用重写的 `afterNodeRemoval`，来更新双向链表



### 2.8 entrySet

+ `entrySet()` 方法在 LinkedHashMap 中也被重写，返回的是 `LinkedEntrySet`
```java
public Set<Map.Entry<K,V>> entrySet() 
{
	Set<Map.Entry<K,V>> es;
	return (es = entrySet) == null ? (entrySet = new LinkedEntrySet()) : es;
}
```

+ 在 forEach 方法中，它遍历的是 **LinkedHashMap内部维护的双向链表**，而不是类似 HashMap 的 table 数组，因此初始容量对遍历没有影响

```java
public final void forEach(Consumer<? super Map.Entry<K,V>> action) 
{
    if (action == null)
		throw new NullPointerException();
    int mc = modCount;
    
    // 通过head指针遍历双向链表，初始容量对其没有影响
    for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after)
		action.accept(e);
    if (modCount != mc)
		throw new ConcurrentModificationException();
}
```







