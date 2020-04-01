---
title: Java_集合知识
categories:
  - Java
top: false
date: 2020-03-31 10:11:19
tags:
- Java
- Java集合
keywords: java、ArrayList、linkedList、HashMap
description: 
---

## java集合

### ArrayList与LinkedList
- **非线程安全** ：`ArrayList`与`LinkedList`都是 ***不同步*** 的，都是 **非** 线程安全的。
- **底层数据结构** ：`ArrayList`底层使用的是`Object[]`数组，`LinkedList`底层使用的是`双向链表`。
- **插入和删除元素** ：
    - `ArrayList`由于底层是使用Object[]数组实现的，所以插入和删除元素的时间复杂度受元素位置的影响，**时间**复杂度 *近似O(n)*。如果ArrayList是在末尾插入和删除，则时间负责度就是O(1)。但是如果是在中间插入或删除，则时间复杂度为O(n-i)。因为在第i个位置插入或删除元素，会影响n-i个元素的位置。
    - `LinkedList`采用的是链表存储，所以在插入或删除元素不受位置的影响，都是 *近似O(1)*。只需要改变相连元素的pre或next的指针。
- **随机访问** ： `ArrayList`实现了`RandomAccess`标识接口，支持快速随机访问，可以通过下标快速找到。LinkedList不支持高效的随机访问。在`Collections`类里binarySearch如下
    ```java
    public static <T> int binarySearch(List<? extends Comparable<? super T> list, T key){
      if(list instanceof RandomAccess || list.size() < BINARYSEARCH_THRESHOLD)
        return Collections.indexedBinarySearch(list,key);
      else
        return Collections.iteratorBinarySearch(list,key);
    }
    ```
#### 总结遍历方式选择
- 实现了RandomAccess接口的list，优先使用普通for循环
- 未实现RandomAccess接口的list，优先使用iterator遍历，大size的数据，千万不要使用普通for循环

### ArrayList与Vector
- `ArrayList`是 **非线程安全** 的
- `Vector`是 **线程安全** 的，`Vector`的所有 *方法* 都使用`synchronized`关键字。

### HashMap的底层实现
> **数据结构**：`数组`加`链表(或当链表的size > 8 时会转换为红黑树)`。数组元素是通过key的hashCode经过扰动函数处理后得到的hash值，然后通过(n-1) & hash 判断当前元素的存放位置(这里的n为数组的长度)，如果当前元素存在，则判断该元素与要存入的元素的hash值以及key是否相同，如果相同则直接覆盖，否则就通过拉链法解决冲突。 以下为HashMap的hash扰动函数
  ```java
  static final int hash(Object key){
    int h;
    return (key == null) ? 0 : (h=key.hashCode()) ^ (h >>> 16);
  }
  ```

### HashMap与HashTable
1. `HashMap`是 **非线程安全**的；而`HashTable`是 **线程安全** 的，`HashTable`内部的方法基本都经过`synchronized`修饰。如果要保证线程安全，建议使用`ConcurrentHashMap`！
2. `HashMap的key`可以有一个为null，多个value为null；`HashTable`的key和null任何一个为null都会抛出`NullPointerException`
3. `HashMap`的默认初始容量为 **16**，之后每次扩充都是原来的*2*倍；如果给予了初始大小，则会将其扩充为 ***2的幂次方***大小；`HashTable`默认初始大小为**11**，之后每次扩充，容量为原来的***2n+1***；如果给定了初始大小则会使用给定的大小。

### HashMap的长度是2的幂次方
> 为了让`HashMap`存取高效，尽量减少碰撞，也就是尽量把数据分配均匀。Hash值的范围值-2147483648到2147483647，前后加起来大概 **40亿** 的映射空间，只要哈希函数映射比较均匀，一般应用很难出现碰撞。但是一个40亿长度的数组，内存是放不下的。所以这个哈希值不能直接用。用之前还要先做数组的长度取模运算，得到的余数才能用来要存放的位置，也就是对应的数组下标。这个数组下标的计算方法(n-1) % hash 。（n代表数组长度）。这也就是解释了HashMap的长度为什么是2的幂次方。

#### 算法说明
> 我们首先会想到采用%取余的操锁实现。但是，取余`（%）`操作中如果除数是`2的幂次`则等价于与其除数减一的与（`&`）操作。（也即是说 `hash % length == hash&(length-1)`的前提是 ***length是2的n次方***）。并且采用二进制位操作`&`相对于`%`能够提高运算效率。

### HashMap多线程操作死循环问题
- 主要是发生在多线程下hashMap发生扩容调用resize方法引起的，JDK8已解决。

### HashMap与HashSet
- 查看源码发现，HashSet的底层存储数据使用的是HashMap，HashSet很多方法都是直接调用HashMap的方法。HashSet自己只实现了clone、readObject、writeObject方法。HashMap比HashSet效率更好。

### ConcurrentHashMap线程安全的具体实现、底层原理
- 数据结构：Node数组 + 链表 + 红黑树
- 并发控制：使用synchronized和CAS来操作，synchronized只锁定丹铅链表或红黑树的首节点，这样只要hash不冲突，就不会产生并发锁问题。 


### 集合底层数据结构总结
#### Collection
-  **ArrayList**：Object数组
-  **Vector**：Object数组
-  **LinkedList**：双向链表

#### Set
-  **HashSet**：*无序，唯一*；基于HashMap实现的，底层采用HashMap来保存元素
-  **LinkedHashSet**：`LinkedHashSet`继承于HashSet，并且其内部是通过linkedHashMap来实现的
-  **TreeSet**：有序，唯一；红黑树（自平衡的排序二叉树）

#### Map
-  **HashMap**：数组+链表或红黑树
-  **LinkedHashMap**：linkedHashMap继承于HashMap，并增加了一条双向链表，使得可以保持键值对的插入顺序。同时通过对链表的相应操作，实现了访问顺序逻辑
-  **HashTable**：数组+链表
-  **TreeMap**：红黑树(自平衡的排序二叉树)