---
layout: post
category: computer
title: "Hashtable 使用与分析"
author: ZhangKe
date:   2020-10-12 23:17:27 +0800
---

# Hashtable
Hashtable 实现了 Map 接口，用于存储键值对，**禁止 null 元素**。

用作键的对象必须保证`hashCode`与`equals`的可用性。

Hashtable 是**线程安全**的，但一般来说，如果不需要考虑线程安全问题可以使用 **HashMap** 作为替代，如果需要线程安全的高并发哈希表可以使用 **ConcurrentHashMap**，总的来说就是，一般不需要使用 HashTable。

Hashtable 具有两个性能相关的参数：**初始容量(_initial capacity_)**和**负载因子(_load factor_)**， 容量是指 HashTable **桶（_buckets_）**的容量，初始容量即 HashTable 在创建时桶的默认大小。在哈希冲突的情况下，每个桶会通过**链表**存储多条数据。
<!--more-->
负载因子默认是 **0.75**，一般来说该值是最佳值。

当 HashTable 容量超出负载因子时将会进行 **rehash** 操作，该操作会对所有数据的位置进行重新分配，是耗时操作。

HashTable 的迭代器被设计为 **fail-fast** 的，如果迭代器创建完成后 Hashtable 的结构被修改，迭代器将会抛出**ConcurrentModificationException**异常。但 Hashtable 本身的`get`及`put`方法并不是 fail-fast 的。

但迭代器也并不能保证准确的 fail-fast，只会尽最大努力的抛出异常。

# 源码
Hashtable 存储的是键值对，在内部会将键值封装成一个`Entry<K,V>`对象，我们先看看这个类：
```java
private static class Entry<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Entry<K,V> next;

    public boolean equals(Object o) {
        if (!(o instanceof Map.Entry))
            return false;
        Map.Entry<?,?> e = (Map.Entry<?,?>)o;

        return (key==null ? e.getKey()==null : key.equals(e.getKey())) &&
                (value==null ? e.getValue()==null : value.equals(e.getValue()));
    }

    public int hashCode() {
        return hash ^ Objects.hashCode(value);
    }
}
```
Entry 类主要是对键值对的封装，其中`hash`是键的`hashCode`，`next`表示下一个节点，我们上面也说过了，每个位置可能会存储多条数据，而存储方式就是链表。

我们再看看类中定义的几个主要的参数：
```java
private transient Entry<?,?>[] table;
private transient int count;//当前Hashtable的实际大小
private int threshold;//扩容阈值 = capacity * loadFactor
private float loadFactor;//负载因子，默认0.75
```
负载因子可以用过构造器指定，`threshold`会在初始化时被设置，每次扩容也会更新。

`table`数组则是 Hashtable 的底层数据结构，用于存储数据，也就是上面说的桶，而数组的每个元素都是个链表。

## put(K,V)
再来通过一个典型的 put 方法看看其内部实现：
```java
public synchronized V put(K key, V value) {
    // Make sure the value is not null
    if (value == null) {
        throw new NullPointerException();
    }

    // Makes sure the key is not already in the hashtable.
    Hashtable.Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    @SuppressWarnings("unchecked")
    Entry<K,V> entry = (Entry<K,V>)tab[index];
    for(; entry != null ; entry = entry.next) {
        if ((entry.hash == hash) && entry.key.equals(key)) {
            V old = entry.value;
            entry.value = value;
            return old;
        }
    }

    addEntry(hash, key, value, index);
    return null;
}
```
我们上面说过需要保证键的`hashCode`与`equals`的可用性，这里就可以看到原因，计算数组位置的依据就是`hashCode`。

通过键来计算对应数组索引的算法很简单：
```java
int index = (hash & 0x7FFFFFFF) % tab.length;
```
`0x7FFFFFFF`是`Integer.MAX_VALUE`的值，按位与这一步是为了防止索引值超出整型范围，然后根据数组长度求余可以将索引控制在数组范围内。

下面的 for 循环是用来搜索 HashTable 中是否已经包含当前键对象，如果已经包含则直接更新`value`的值。

否则调用`addEntry`方法将该键值对添加到 HashTable 中去。

## addEntry(int, K, V, int)
```java
private void addEntry(int hash, K key, V value, int index) {
    modCount++;// 尽可能保证 fail-fast

    Hashtable.Entry<?,?> tab[] = table;
    if (count >= threshold) {
        // Rehash the table if the threshold is exceeded
        rehash();

        tab = table;
        hash = key.hashCode();
        index = (hash & 0x7FFFFFFF) % tab.length;
    }

    // Creates the new entry.
    @SuppressWarnings("unchecked")
    Entry<K,V> e = (Entry<K,V>) tab[index];
    tab[index] = new Entry<>(hash, key, value, e);
    count++;
}
```
首先判断当前数组是否已达到扩容阈值，达到则先进行 **rehash** 操作，该操作会重新创建一个更大的数组，并重新分配所有的数据。

rehash 操作完成后就可以将键值对信息组装成 Entry 并添加到`table`中。

## rehash()
rehash 操作时耗时操作，主要负责两个任务，一是扩容，二是重新分配数据位置。
```java
protected void rehash() {
    int oldCapacity = table.length;
    Entry<?,?>[] oldMap = table;

    // overflow-conscious code
    int newCapacity = (oldCapacity << 1) + 1;
    if (newCapacity - MAX_ARRAY_SIZE > 0) {
        if (oldCapacity == MAX_ARRAY_SIZE)
            // Keep running with MAX_ARRAY_SIZE buckets
            return;
        newCapacity = MAX_ARRAY_SIZE;
    }
    Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

    modCount++;
    threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    table = newMap;

    for (int i = oldCapacity ; i-- > 0 ;) {
        for (Entry<K,V> old = (Entry<K,V>)oldMap[i]; old != null ; ) {
            Entry<K,V> e = old;
            old = old.next;

            int index = (e.hash & 0x7FFFFFFF) % newCapacity;
            e.next = (Entry<K,V>)newMap[index];
            newMap[index] = e;
        }
    }
}
```
每次的扩容大小是`(oldCapacity << 1) + 1`也就是原来长度的**两倍 + 1**。

如果长度已达到最大值，则不会进行扩容，而且在原来长度的基础上继续添加，此时数组上链表的长度会越来越长，扩容完成后会遍历所有数据重新分配位置。

## get(Object)
通过`get`方法获取对应键的值，没有则返回`null`。
```java
public synchronized V get(Object key) {
    Entry<?,?> tab[] = table;
    int hash = key.hashCode();
    int index = (hash & 0x7FFFFFFF) % tab.length;
    for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) {
            return (V)e.value;
        }
    }
    return null;
}
```
先通过哈希值获取到数组的索引，然后遍历该索引处的链表，直到找到哈希值相同且`equals`方法返回`true`的值，然后返回该值，或者返回`null`。