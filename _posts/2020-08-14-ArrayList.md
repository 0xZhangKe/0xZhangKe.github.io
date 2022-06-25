---
layout: post
category: computer
title: "ArrayList 使用与分析"
author: ZhangKe
date:   2020-08-14 23:06:27 +0800
---

# List
先看下 ArrayList 实现的接口 List 的相关概念。
- List 可以称为有序集合或者序列，通过整数索引访问元素
- 允许插入相同元素
- 一般来说也允许插入 null 值
<!--more-->
List 接口中还提供了一个特殊的迭代器：ListIterator

## ListIterator
ListIterator 专门为了 List 打造，在 Iterator 基础上还提供了插入和替换元素以及双向访问的功能。
我们来看下使用：
```java
List<String> list= new ArrayList<>();
list.add("1");
list.add("2");
list.add("7");
list.add("4");
ListIterator<String> iterator = list.listIterator();
while (iterator.hasNext()){
    String next = iterator.next();
    if(next.equals("7")){
        iterator.set("3");
        iterator.previous();
    }else{
        System.out.println(next);
    }
}
```
# ArrayList
Java 原生提供了数组数据结构，但由于本身设计存在诸多问题，例如无法扩容、类型不安全等，不够灵活，所以大部分时候可以使用 ArrayList 来替代，效率上没有太大的差异。

ArrayList 是 List 接口可调整大小的数组实现。

`size`,`isEmpty`,`get`,`set`,`iterator`以及 `listIterator`方法调用的执行时间都是**固定时间**。

`add`操作时间是**摊销固定时间（amortized constant time）**，也就是添加 **n** 个元素需要 **O(n)** 的时间。其他操作都是**线性时间**。

ArrayList 中有个 **capacity** 参数用于描述 ArrayList 中数组的长度，`add`操作会先进行扩容，引起`capacity`的增长，这是个耗时操作。如果将会发生多次`add`操作，可以在此之前先调用`ensureCapacity`方法扩容，以此减小扩容的次数，提升性能。

需要注意的是 ArrayList 是**非线程安全**的，多线程环境下，如果希望修改其结构，必须进行同步，也可以使用 Collections 中的包装方法获取同步对象：
```java
List list = Collections.synchronizedList(new ArrayList(...));
```
iterator 以及 listIterator 也被设计为 [fail-fast](https://en.wikipedia.org/wiki/Fail-fast "fail-fast") 的，多线程环境下使用 iterator 将会直接抛出异常。

ArrayList 长度**默认为 0**，可以通过构造器设置初始长度。

扩容机制是每次添加元素时会检测是否需要扩容，**每次增加的长度为当前长度的一半**，可以通过`ensureCapacity`方法使其扩容到指定长度。

## 源码
我们先从构造函数开始看
```java
//ArrayList 中存储数据的数组
transient Object[] elementData;
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+initialCapacity);
    }
}
public ArrayList() {
    //DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```
可以看到`elementData`默认长度是 **0**，如果设置了默认长度就会初始化成改长度的数组。

再来看看`add`方法：
```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);
    elementData[size++] = e;
    return true;
}
```
注意这里的`size`字段表示当前 ArrayList 的实际长度，不代表`elementData`的长度。
`add`方法会先调用`ensureCapacityInternal`方法，传入的参数`size+1`并不代表本次的扩容长度，而是**最小**扩容长度。
然后该方法会先判断是否需要进行扩容，判断依据就是当前数组是否还有空余空间添加新元素。
```java
//minCapacity = size + 1
if (minCapacity - elementData.length > 0)
        grow(minCapacity);
```
那么再看看扩容是如何实现的，可以先猜想下，既然内部使用数组来存储数据，但数组是不可变长度的，那么最可行的方案应该就是重新创建一个更大的数组，然后将数据拷贝进去，我们可以看看是不是这么做的。
```java
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    //右移一位等于整除2
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        //hugeCapacity 返回的最大值为 Integer.MAX_VALUE
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```
可以看到每次扩容的长度是当前长度 + 当前长度的一半，也就是**扩容 50%** 。
`MAX_ARRAY_SIZE`是`Integer.MAX_VALUE - 8`，一般来说数组长度不会超过这个值，但如果非要继续进行扩容将会扩张到 `Integer.MAX_VALUE`。

那么既然可以扩张到`Integer.MAX_VALUE`为什么还要一个`MAX_ARRAY_SIZE`呢，代码里面的解释是一些 JVM 会在数组中添加一些保留字段，如果将数组长度扩张到`Integer.MAX_VALUE`可能会导致`OutOfMemoryError`。