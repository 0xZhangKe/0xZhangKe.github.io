---
layout: post
category: computer
title: "LinkedList 使用与分析"
author: ZhangKe
date:   2020-08-13 18:00:03 +0800
---

# LinkedList
实现了 **List** 以及 **Deque** 的**双向链表**，元素允许为 null，所以 LinkedList 同时具备 List 以及 Deque 的特性。
跟 ArrayList 一样，LinkedList 也是**非线程安全**的，可以使用包装方法获取同步对象：
```java
List list = Collections.synchronizedList(new LinkedList(...));
```
`iterator`以及`listIterator`同样也被设计为 [fail-fast](https://en.wikipedia.org/wiki/Fail-fast "fail-fast")。
<!--more-->
## 使用特性
LinkedList 内部实现上是个**链表**，所以可以把它当作**链表**使用。

LinkedList 同样还可以代替 Stack 类当做**栈**来使用（官方也推荐这么做）。

但不合适作为 **List** 使用，List 相关的方法会比较耗时，性能低。

链表和栈相关的操作基本都可以在**固定时间**内完成，但是 List 特性的操作，也就是通过索引访问元素的方法`get`需要 **O(n)** 的时间，不推荐使用。

## 公开方法
因为双端队列也是基于队列的，所以 LinkedList 中的方法大体可以分为三种：
- 列表方法
- 队列方法
- 双端队列方法

下面分别看下。

### 列表方法
列表方法比较简单，这里就不赘述了，可以看 ArrayList 那一节文章。

### 队列
队列是一种先进先出（FIFO）的数据结构，Java 中的接口 Queue 描述了队列这种数据结构。

|方法名|描述|备注|
| :------------ | :------------ | :------------ |
|add(e)|将元素插入队列|未超出容量限制则添加成功返回 true，否则抛出 IllegalStateException|
|offer(e)|将元素插入队列|未超出容量限制的情况下添加，否则返回 false|
|remove()|返回并删除队列的头|队列为空:NoSuchElementException|
|poll()|返回并删除队列的头|队列为空返回 null|
|element()|返回队列的头|队列为空:NoSuchElementException|
|peek()|返回队列的头|队列为空返回 null|

我们可以将上表按照功能分为三类：插入、移除、检索。
每种操作又提供了两种方法：抛出异常或返回特殊值。

|操作|抛出异常|返回特殊值|
| :------------ | :------------ | :------------ |
|插入|add(e)|offer(e)|
|移除|remove()|poll()|
|检索|element()|peek()|

上面就是队列中提供的方法了，Java 大概是考虑到使用方便问题，所以给每种方法都提供了抛出异常或者返回特殊值的方式，也导致方法数量多了一倍，而对于双端队列来说，上述的每种方法又会多出两个，看起来很多，但我们只要知道本质上就是上述的三种操作就行，其他的都是变种而已。

## 双端队列
作为双端队列，比队列多出的特性将通过给队列中的每个操作方法提供两个变种方法来实现，比如队列中的 add 方法，双端队列中将提供`addFirst`和`addLast`方法，用于将元素添加到**头**或者**尾**。

由于双端队列的特性导致还可以很轻易的实现**栈**的功能，可以用来替换 Stack 类来使用，所以 Deque 中又提供了两个用于实现栈结构的方法。

|方法名|描述|备注|
| :------------ | :------------ | :------------ |
|push(E e)|将元素压入栈顶||
|pop()|弹出栈的顶部元素|列表为空抛出 NoSuchElementException|

Deque 还提供了从后向前遍历的`descendingIterator`的迭代器，元素将从最后一个开始遍历到第一个元素。

## 总结
下面来总结一下。
LinkedList 作为队列（**FIFO**）使用时的方法：

|队列方法|等效的双端队列方法|
| ------------ | ------------ |
|add(e)|addLast(e)|
|offer(e)|offerLast(e)|
|remove()|removeFirst()|
|poll()|pollFirst()|
|element()|getFirst()|
|peek()|peekFirst()|

LinkedList 作为堆栈（`index`）使用时的方法：

|堆栈方法|等效的双端队列方法|
| ------------ | ------------ |
|push(e)|addFirst(e)|
|pop(e)|removeFirst(e)|
|peek()|peekFirst()|

## 源码
我们先看看链表的节点是如何定义的：
```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```
因为是双向链表，所以每个节点除了保存下一个节点的引用外，还保存了上一个节点的引用。
所以与 ArrayList 不同的是，LinkedList 并不是使用数组存储数据，而是通过上面 **Node 形成的链表来存储**。

我们先来通过一个简单的`add`方法看看实现：
```java
public boolean add(E e) {
    linkLast(e);
    return true;
}
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```
主要就调用了`linkLast`方法将元素添加到链表的末尾，这就是个链表的操作。
除了`linkLast`之外还有`linkFirst`用于将元素添加到链表头部。
诸如`addFirst`,`push`,`offer`之类的添加元素的操作最终都是通过上面两个方法实现的。

再来看看移除操作`poll`:
```java
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```
使用`unlinkFirst`方法将元素从链表顶部移除，对应的还有`unlinkLast`方法。
同样，类似`removeFirst`,`pop`,`pollFirst`等操作也是通过上面两个方法实现的。

上面的两种操作都是直接对链表的头尾操作，都可以在**固定时间复杂度**内完成，实现也比较简单。
但考虑到 LinkedList 还实现了 List 接口，具备了 List 的特性，例如通过下标获取到指定值，这个实现就比较复杂了，也会消耗掉 **O(n)** 的时间。
```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
Node<E> node(int index) {
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
        x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
        x = x.prev;
        return x;
    }
}
```

`get`方法通过调用`node`方法获取到指定索引的值，由于链表的特性使然，无法直接通过索引获取到元素，需要从端点开始逐个向内遍历，直到遍历到指定索引再将元素返回。
`node`方法考虑到性能问题，在遍历前会先判断`index`值靠前还是靠后，然后选择是从头还是尾开始遍历，充分利用了双向链表的特性。

好了，上面几个方法就是 LinkedList 源码中最具有代表性的几个方法了。