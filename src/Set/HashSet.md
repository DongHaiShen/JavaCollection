[TOC]

## 一、顶部注释分析

### 1.1 从注释中得到的结论

+ This class implements the Set interface, backed by a hash table (actually a HashMap instance)：该类实现了 Set 接口，由哈希表进行支持（实际上是 HashMap 的实例）
+ HashSet **不保证迭代的顺序恒久不变**
+ HashSet 允许 null 元素 （由于HashMap支持null元素）
+ HashSet 同样是非线程安全的
+ **对 HashSet 的操作实际上就是对 HashMap 的操作**



## 二、源码分析

### 2.1 定义

```java
public class HashSet<E> extends AbstractSet<E> 
		implements Set<E>, Cloneable, java.io.Serializable
```

+ HashSet \<E\>：存储的是单个泛型元素
+ extends **AbstractSet**\<E\>：继承自AbstractSet，实现Set接口时需要实现的工作量大大减少了
+ implements Set\<E\>：实现了Set接口，提供了所有可选的 Set 操作
+ implements Cloneable：可以调用 clone() 方法来返回实例的 `field-for-field` 拷贝
+ implements Serializable：可以序列化



### 2.2 字段

```java
// HashSet的底层就是一个HashMap实例
private transient HashMap<E,Object> map;

// HashMap是保存键值对的，而HashSet实际上只想保存key
// 因此创建一个PRESENT表示所有的value
private static final Object PRESENT = new Object();
```



### 2.3 构造方法

#### 2.3.1 常规构造方法

1. `public HashSet()`：实际上就是构造一个空的HashMap，然后赋值给 map 字段
2. `public HashSet(Collection<? extends E> c)`：构造一个包含指定集合中所有元素的新set
3. `public HashSet(int initialCapacity, float loadFactor)`：构造一个新的空set，其底层HashMap实例具有指定的初始容量和加载因子
4. `public HashSet(int initialCapacity)`：构造一个新的空set，其底层HashMap实例具有指定的初始容量和默认的加载因子 (0.75)



#### 2.3.2 特殊构造方法

+ 这个构造方法是一个**包私有** (package private) 的方法，可以看到它的**修饰符不是 public**
+ 该方法**仅用于构造 LinkedHashSet**，创建的是 **LinkedHashMap** 而非之前的 HashMap，用于维护 Set 内元素的顺序
+ 在 LinkedHashSet 进行初始化时，通过增加 dummy  参数来调用该方法
```java
HashSet(int initialCapacity, float loadFactor, boolean dummy) 
{
	map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```












