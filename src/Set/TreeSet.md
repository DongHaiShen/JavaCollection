[TOC]

## 一、顶部注释分析

### 1.1 从注释中得到的结论

+ A NavigableSet implementation based on a TreeMap：基于 **TreeMap** 的 NavigableSet 实现
+ HashSet 底层实际上是一个 **NavigableMap 接口类型的实例**，如 TreeMap
+ HashSet 是非线程安全的



## 二、源码分析

### 2.1 定义

```java
public class TreeSet<E> extends AbstractSet<E> 
		implements NavigableSet<E>, Cloneable, java.io.Serializable
```

+ TreeSet\<E\>：TreeSet支持泛型
+ extends **AbstractSet**\<E\>：继承自AbstractSet
+ implements NavigableSet：实现了NavigableSet 接口，可以返回特定条件的最近匹配
+ implements Cloneable：可以调用 clone() 方法来返回实例的 `field-for-field` 拷贝
+ implements Serializable：可以序列化



### 2.2 字段

```java
// TreeSet的底层就是一个NavigableMap类型的实例
// 而TreeMap实现了NavigableMap接口，因此可用来构造TreeSet
private transient NavigableMap<E,Object> m;

// 和HashSet相似，创建一个PRESENT表示所有的value
private static final Object PRESENT = new Object();
```



### 2.3 构造方法

1. `TreeSet(NavigableMap<E,Object> m)`：根据指定的 NavigableMap 构造一个 TreeSet
2. `public TreeSet()`：构造一个空的TreeMap并赋值给m，value 为 PRESENT
3. `public TreeSet(Comparator<? super E> comparator)`：构造一个新的空TreeSet，它根据指定比较器进行排序
4. `public TreeSet(Collection<? extends E> c)`：构造一个包含指定 collection 中所有元素的新TreeSet，它按照其元素的自然顺序进行排序
5. `public TreeSet(SortedSet<E> s)`：构造一个与指定 SortedSet 具有相同元素和相同排序规则的新TreeSet




