---
layout: post
category: computer
title: "CopyOnWriteArrayList 使用与分析"
author: ZhangKe
date:   2020-08-14 23:14:27 +0800
---

# CopyOnWriteArrayList
先看看百科上关于 [COW 的介绍](https://zh.wikipedia.org/wiki/%E5%AF%AB%E5%85%A5%E6%99%82%E8%A4%87%E8%A3%BD "COW 的介绍")：
> 写入时复制（英语：Copy-on-write，简称COW）是一种计算机程序设计领域的优化策略。其核心思想是，如果有多个调用者（callers）同时请求相同资源（如内存或磁盘上的数据存储），他们会共同获取相同的指针指向相同的资源，直到某个调用者试图修改资源的内容时，系统才会真正复制一份专用副本（private copy）给该调用者，而其他调用者所见到的最初的资源仍然保持不变。这过程对其他的调用者都是透明的（transparently）。此作法主要的优点是如果调用者没有修改该资源，就不会有副本（private copy）被创建，因此多个调用者只是读取操作时可以共享同一份资源。
<!--more-->
简单来说，就是读取时直接读取不用加锁同步，写入数据时会**复制一份副本**，然后将新的数据写入到副本中，然后再把副本替换成原来的数据。

因此 CopyOnWriteArrayList 是**线程安全**的，另外也允许 null  元素。

这种方式导致了读取速度很快，写入速度较慢，**适合多线程环境中经常读取但写入很少的场景。**

所以如果有个场景需要在多线程环境中使用，频繁读取，但写入次频率很低，例如黑名单白名单，每天更新一次，那就可以使用 CopyOnWriteArrayList 来实现。

其中的迭代器是不允许对数据修改的，调用`remove`,`set`,`add`这些方法会直接抛异常。

一般可以按照常规的 List 来使用，其中没有什么特殊的方法，就直接开始看源码吧。

CopyOnWriteArrayList 下面简称 COWList。

## 源码
我们先看其中两个最重要的方法：
```java
private transient volatile Object[] array;
final Object[] getArray() {
    return array;
}
final void setArray(Object[] a) {
    array = a;
}
```
上面定义的`array`变量就是 COWList 中存储数据的地方。
所有的读取写入操作都是通过对这个数组的操作。

因此构造函数也是创建一个空的数组然后调用`setArray`方法设置数组。
```java
public CopyOnWriteArrayList() {
    setArray(new Object[0]);
}
```

COWList 的优势之一就是即使在多线程环境中，读取也是不需要加锁的，所以效率很高，那么我们来看看读取方法是如何实现的：
```java
public E get(int index) {
    return get(getArray(), index);
}
private E get(Object[] a, int index) {
    return (E) a[index];
}
```
可以看到这里直接通过索引读取了数组的值，没有加锁。这根我们使用`Collections.synchronizedList`方法获取到的`SynchronizedList`效率要高。

可以看下`SynchronizedList`是如何实现读取及写入的：
```java
//SynchronizedList
public E get(int index) {
    synchronized (mutex) {
        return list.get(index);
    }
}
public E set(int index, E element) {
    synchronized (mutex) {
        return list.set(index, element);
    }
}
```
对比一下显而易见，读取不加锁的话效率就高多了。

那么 COWList 是如何实现多线程环境下的安全问题的呢，看下写入方法就明白了。
```java
public boolean add(E e) {
    synchronized (lock) {
        Object[] elements = getArray();
        int len = elements.length;
        //复制数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    }
}
```
可以看到添加元素会加锁，加锁之后会将原数组复制到一个更大（`len + 1`）的数组中去，并将追加元素添加到数组末尾，完成操作后调用`setArray`方法设置新数组。

所以写入的效率是很低的，需要进行一次数组复制操作，用 COWList 时应该尽量降低写入操作的频率，如果需要经常写入，那说明可能不适合用这个。
