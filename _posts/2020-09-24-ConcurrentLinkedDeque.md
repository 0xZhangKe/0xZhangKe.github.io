---
layout: post
category: computer
title: "ConcurrentLinkedDeque 使用与分析"
author: ZhangKe
date:   2020-09-24 23:16:27 +0800
---

# ConcurrentLinkedDeque
ConcurrentLinkedDeque 是基于**链表**的**无限双端队列**，**线程安全**，不允许 null 元素。

ConcurrentLinkedDeque 内部通过 CAS 来实现线程同步，一般来说，如果需要使用线程安全的双端队列，那么推荐使用该类。

由于双端队列的特性，该类同样可以当做**栈**来使用，所以如果需要在并发环境下使用栈，也可以使用该类。

迭代器设计为[弱一致性](https://en.wikipedia.org/wiki/Weak_consistency "弱一致性")的（weakly consistent），此外还可以通过`descendingIterator`方法获取一个通过相反方向遍历的迭代器。

虽然跟 LinkedList 一样都是双端队列的链表实现，但由于其并发特性，导致无法简单点的通过计数来确定队列的长度，所以`size`方法将会**以线性时间**运行，并且如果在执行期间被其它线程修改可能返回**不准确的结果**。
<!--more-->
对于批量操作，例如：`addAll，removeAll，retainAll，containsAll，equals,toArray`来说，不能保证这些操作的原子性。

# 方法

ConcurrentLinkedDeque 作为队列（**FIFO**）使用时的方法：

|队列方法|等效的双端队列方法|
| ------------ | ------------ |
|add(e)|addLast(e)|
|offer(e)|offerLast(e)|
|remove()|removeFirst()|
|poll()|pollFirst()|
|element()|getFirst()|
|peek()|peekFirst()|

ConcurrentLinkedDeque 作为堆栈（**FILO**）使用时的方法：

|堆栈方法|等效的双端队列方法|
| ------------ | ------------ |
|push(e)|addFirst(e)|
|pop(e)|removeFirst(e)|
|peek()|peekFirst()|

# 源码
因为 ConcurrentLinkedDeque 是基于链表的数据结构，我们先看看该类中使用的链表节点 Node 的相关代码。
## Node
```java
static final class Node<E> {
    volatile Node<E> prev;
    volatile E item;
    volatile Node<E> next;

    private static final sun.misc.Unsafe UNSAFE;
    private static final long prevOffset;
    private static final long itemOffset;
    private static final long nextOffset;

    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = Node.class;
            prevOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("prev"));
            itemOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("item"));
            nextOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("next"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}

```
上半部分代码就是一个标准的 Node 节点的定义，我们主要看静态初始化块中的代码。

首先介绍一下 Unsafe 类，关于这个类的详细信息及使用可以看美团技术的[这篇文章](https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html "这篇文章")，这里大概介绍一下，Unsafe 中提供了一些执行级别低，不安全的方法，例如内存操作以及待会会看到的 CAS，一般来说不推荐使用，因此命名为 Unsafe。

Node 内部类中定义的三个 offset 变量：`prevOffset`、`itemOffset`、`nextOffset` 分别对应 `prev`、`item`、`next` 这三个变量的内存偏移量（相对内存地址），通过这三个偏移量可以拿到实际的内存中的值，这个后面会看到怎么用。

然后静态初始化块中获取了 Unsafe 实例，接着对三个 offset 变量进行初始化。

下面再来看看 Node 中其它几个方法：
```java
Node(E item) { UNSAFE.putObject(this, itemOffset, item); }
boolean casItem(E cmp, E val) {
    return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
}
void lazySetNext(Node<E> val) {
    UNSAFE.putOrderedObject(this, nextOffset, val);
}
boolean casNext(Node<E> cmp, Node<E> val) {
    return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
}
void lazySetPrev(Node<E> val) {
    UNSAFE.putOrderedObject(this, prevOffset, val);
}
boolean casPrev(Node<E> cmp, Node<E> val) {
    return UNSAFE.compareAndSwapObject(this, prevOffset, cmp, val);
}
```
第一个构造方法意思就是将参数中的对象设置到指定的地址偏移量处。

下面的几个方法分为两类，一类是 cas 开头的操作一类是 lazySet 操作，先说下 cas 操作，例如`casItem`方法，其作用是将对应的 item 值替换为参数 val 的值，该方法会先对比内存中的值与 cmp 参数值是否相等，相等则替换，返回值表示操作是否完成，这里调用了`UNSAFE.compareAndSwapObject`方法来保证并发环境下的安全问题。

而`lazySetNext`方法则是直接将参数 val 设置到内存中，调用了`UNSAFE.putOrderedObject`方法。

上面我们已经介绍了 Node 类中的代码，其中已经实现了部分多线程同步代码，毕竟对链表的操作本质上都是对节点的操作，虽然其他还有很多方法，但基本都是层层嵌套调用，最终可能都会调用到那几个重点方法上面，下面我们看看这几个重点。

## linkFirst
`linkFirst`方法用于将指定的节点添加到链表头位置，例如`addFirst`、`offerFirst`最终都是调用了该方法。
```java
private void linkFirst(E e) {
  checkNotNull(e);
  final Node<E> newNode = new Node<E>(e);
  restartFromHead:
  for (;;)
    for (Node<E> h = head, p = h, q;;) {
      if ((q = p.prev) != null &&
              (q = (p = q).prev) != null)
        // Check for head updates every other hop.
        // If p == q, we are sure to follow head instead.
        p = (h != (h = head)) ? h : q;
      else if (p.next == p) // PREV_TERMINATOR
        continue restartFromHead;
      else {
        // p is first node
        newNode.lazySetNext(p); // CAS piggyback
        if (p.casPrev(null, newNode)) {
          // Successful CAS is the linearization point
          // for e to become an element of this deque,
          // and for newNode to become "live".
          if (p != h) // hop two nodes at a time
            casHead(h, newNode);  // Failure is OK.
          return;
        }
        // Lost CAS race to another thread; re-read prev
      }
    }
}
```
上面代码先开启了一个死循环用于自旋，第二个循环用于在每次自旋时找到`head`节点，因为多线程环境下`head`节点或者说链表随时可能被修改，所以每次都需要重新获取。

找到`head`节点后，也就是`p`变量，开始进行替换操作，将`newNode`设置为`head`节点。先调用`newNode.lazySetNext(p)`将原来的`head`节点设置为`newNode`的next节点，然后开始 cas 操作，调用`p.casPrev(null, newNode)`将原本`head`节点的`prev`节点设置为`newNode`。`casPrev`的返回值表示是否操作成功，操作失败则继续下一次自旋重试，直到操作成功为止。

好了源码就说这么多吧，其他几个方法也都大同小异，大差不差，这个明白了其他的也就明白了。

