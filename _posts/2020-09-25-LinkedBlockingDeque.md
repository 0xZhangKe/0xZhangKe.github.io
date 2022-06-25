---
layout: post
category: computer
title: "LinkedBlockingDeque 使用与分析"
author: ZhangKe
date:   2020-09-25 23:16:27 +0800
---

# LinkedBlockingDeque
LinkedBlockingDeque 是基于**链表**的**双端阻塞队列**，**线程安全**，元素不允许为 null。

空间容量最大一般为`Integer.MAX_VALUE`，如果构造器中指定了最大值则队列长度将会被限制在该值以下。

大部分方法都以固定时间运行，批量操作，例如：`remove`， `removeFirstOccurrence`，`removeLastOccurrence`，`contains`，`iterator.remove()`，将以线性时间运行。

LinkedBlockingDeque 是**阻塞队列**，是指对于一些指定的操作，在插入或者获取队列元素时如果队列状态不允许该操作可能会阻塞住该线程直到队列状态变更为允许操作，这里的阻塞一般有两种情况。
<!--more-->
第一种是插入元素时，如果当前队列已满将会进入阻塞状态，一直等到队列有空的位置时再讲该元素插入，该操作可以通过设置超时参数，超时后返回 false 表示操作失败，也可以不设置超时参数一直阻塞，中断后抛出`InterruptedException`异常。

第二种是读取元素时如果当前队列为空会阻塞住直到队列不为空然后返回元素，同样可以通过设置超时参数。

一般多用于**生产者消费者模式**。

# 方法
LinkedBlockingDeque 实现了 BlockingDeque 接口，除了原本双端队列中的方法外还另外提供了一些**阻塞操作**的方法。每种操作分为阻塞与超时两种，阻塞方法调用将会一直阻塞知道队列可用，中间如果中断则抛出`InterruptedException`异常。超时方法同样也会阻塞，允许设置超时时间，超时后返回 false 表示操作失败。

下面我们先看下这几个方法。

`head`操作：

|类型|阻塞|超时|
| ------------ | ------------ |
|插入|putFirst(e)|offerFirst(e, time, unit)|
|移除|takeFirst()|pollFirst(time, unit)|

`tail`操作：

|类型|阻塞|超时|
| ------------ | ------------ |
|插入|putLast(e)|offerLast(e, time, unit)|
|移除|takeLast()|pollLast(time, unit)|

# 源码
我们先看看类中定义的几个重要的属性：
```java
private transient int count;//当前链表长度
private final int capacity;//链表最大长度
final ReentrantLock lock = new ReentrantLock();
private final Condition notEmpty = lock.newCondition();
private final Condition notFull = lock.newCondition();
```
`capacity`表示链表的最大长度，默认是`Integer.MAX_VALUE`，可以在构造器中指定。

下面的`lock`是可重入锁，用于在读取或修改链表时使用，保证线程安全。

`notEmpty`表示不为空时的`Condition`，当不为空时会调用`notEmpty.signal()`通知被`notEmpty`阻塞的线程。`notFull`是未满状态的`Condition`，用法同上。

类中的方法可分为三类，第一类是只保证线程安全，但不会阻塞住的方法，第二是会阻塞方法，第三是超时方法。下面我们通过几个典型的方法看看是如何实现的。

通过阅读源码发现，不管是上面三种的哪一种，最终都会调用同一个方法先对节点进行操作，只不过每个方法内部有不同的处理，以读取`head`节点为例，主要有三个典型的方法：
- `pollFirst()`：保证线程安全，但不会阻塞
- `takeFirst()`：阻塞方法
- `pollFirst(timeout, unit)`：超时方法

上面是三个最终都先调用了`unlinkFirst()`移除`head`节点，那么我们先看看这个。
```java
private E unlinkFirst() {
    // assert lock.isHeldByCurrentThread();
    Node<E> f = first;
    if (f == null)
        return null;
    Node<E> n = f.next;
    E item = f.item;
    f.item = null;
    f.next = f; // help GC
    first = n;
    if (n == null)
        last = null;
    else
        n.prev = null;
    --count;
    notFull.signal();
    return item;
}
```
上面代码其实就是一个移除head节点的操作，重点是最后会调用`notFull.signal()`通知被`notFull`阻塞的线程，例如我们使用阻塞/超时方法添加元素时（例如`putFirst`）如果队列已满则会进入阻塞状态，当有其他线程调用了 remove 操作之后此时队列有了空缺的位置，就可以通知阻塞线程继续将元素添加进去。

### pollFirst
下面我们再看看`pollFirst()`是如何实现线程安全的：
```java
public E pollFirst() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return unlinkFirst();
    } finally {
        lock.unlock();
    }
}
```
这个方法其实很简单，先用`ReentrantLock`锁定之后对链表进行操作，操作完成后在解锁，然后返回数据，操作完成。

### takeFirst
`takeFirst()`大概是阻塞队列中最常用的一个方法，用于移除并返回`head`元素，队列为空则阻塞调用线程直到可以返回数据。
```java
public E takeFirst() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        E x;
        while ( (x = unlinkFirst()) == null)
            notEmpty.await();
        return x;
    } finally {
        lock.unlock();
    }
}
```
这里阻塞方式是通过自旋的方式持续从`unlinkFirst`获取`head`节点，空则调用`notEmpty.await()`阻塞，`signal`之后再重复这一步骤，直到获取到值为止。

另外一点，该方法也会先调用`lock.lock()`获得锁，然后继续操作，这点我开始比较疑惑，因为该方法会阻塞住，如果获得了锁别的线程不就没法获得锁从而操作队列了吗，查找资料之后才发现，`notEmpty.await()`方法出于线程调度目的，**该锁会被原子释放**，线程进入阻塞状态，因此其他线程可以正常获得锁并继续操作。

### pollFirst(timeout, unit)
与`takeFirst`不用的是，`pollFirst`可以设置一个超时时间，当阻塞时间超过该时间之后将会从阻塞状态中恢复并返回 null。
```java
    public E pollFirst(long timeout, TimeUnit unit)
        throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            E x;
            while ( (x = unlinkFirst()) == null) {
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            return x;
        } finally {
            lock.unlock();
        }
    }
```
跟`takeFirst()`基本类似，不过调用的是`notEmpty.awaitNanos(nanos)`方法设置超时时间，而且每次循环都会检测一遍是否超时，超时则直接返回 null 值。

还有一个不同点，`pollFirst`获取锁的方式是调用`lock.lockInterruptibly()`方法，该方法与`lock`不同的是，它允许等待线程被中断，中断后直接抛出`InterruptedException`异常，而调用`lock`方法等待线程即使中断仍然会继续等待获取锁，只不过获取成功后会中断线程。