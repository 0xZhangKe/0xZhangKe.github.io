---
layout: post
category: computer
title: "ArrayDeque 使用与分析"
author: ZhangKe
date:   2020-09-23 18:00:03 +0800
---

# ArrayDeque
ArrayDeque 是 Java 集合中**双端队列**的**数组实现**，双端队列的链表实现（**LinkedList**）我们在前几篇文章中讲过了。

ArrayDeque 几乎没有容量限制，设计为**线程不安全的**，**禁止 null 元素**。

ArrayDeque 作为**栈**使用时**比 Stack 类效率要高**，作为**队列**使用时**比 LinkedList 要快**。

ArrayDeque 大多数的额操作都在**固定时间**内运行，例外情况包括 `remove`，`removeFirstOccurrence`，`removeLastOccurrence`，`contains`，`iterator.remove()`，和批量操作，这些将以**线性时间**运行。

`iterator`同样也被设计为 [fail-fast](https://en.wikipedia.org/wiki/Fail-fast "fail-fast")。
<!--more-->
# 方法
ArrayDeque 作为 **队列（FIFO）** 使用时的方法：

|队列方法|等效的双端队列方法|
| ------------ | ------------ |
|add(e)|addLast(e)|
|offer(e)|offerLast(e)|
|remove()|removeFirst()|
|poll()|pollFirst()|
|element()|getFirst()|
|peek()|peekFirst()|

ArrayDeque 作为 **堆栈（FILO）** 使用时的方法：

|堆栈方法|等效的双端队列方法|
| ------------ | ------------ |
|push(e)|addFirst(e)|
|pop(e)|removeFirst(e)|
|peek()|peekFirst()|

# 源码
我们先看看代码中最重要的三个变量：
```java
transient Object[] elements;
transient int head;
transient int tail;
```
我们上面也说了 ArrayDeque 是双端队列的数组实现，上面代码可看到确实定义了一个用于存储链表数据数组：`elements`，此外还定义了`head`及`tail`用于描述链表在数组中的头尾位置。另外说明一下，`elements`长度始终未 2 的次方。

这里需要注意一点，用`elements`存储的是双端队列，`head`和`tail`参数表示双端队列的头和尾的索引，但并不意味着`head`值永远小于`tail`，当`tail`移动至数组末尾，但队列长度小于数组时（意味着此时`head`大于0），再向队列末尾添加数据将会使`tail`移动至数组的开头。

假设下图为当前的`elements`数组，黄色方块表示其中有数据。

![](https://img-blog.csdnimg.cn/20200923143615980.png#pic_center)
当我们向队列末尾添加一个元素时，数组变成了下面这样：

![](https://img-blog.csdnimg.cn/20200923143632138.png#pic_center)

目前都是按照我们预期发展的，现在我们来调用`poll`方法移除队列头部的一个元素，此时`elements`变成了下图：

![](https://img-blog.csdnimg.cn/20200923143645813.png#pic_center)
这个时候因为我们移除了队列的头部元素，数组的开头已经空下来了一个位置，这时候再向队列末尾追加一个元素。

![](https://img-blog.csdnimg.cn/20200923143658734.png#pic_center)
可以看到，此时`head`在`tail`的右侧，ArrayDeque为了不浪费数组空间进行了这样的设计，也不难理解，我们继续向下看。

现在看一下构造器：
```java
public ArrayDeque() {
    elements = new Object[16];
}

public ArrayDeque(int numElements) {
    allocateElements(numElements);
}

public ArrayDeque(Collection<? extends E> c) {
    allocateElements(c.size());
    addAll(c);
}

private static int calculateSize(int numElements) {
    int initialCapacity = MIN_INITIAL_CAPACITY;
    if (numElements >= initialCapacity) {
        initialCapacity = numElements;
        initialCapacity |= (initialCapacity >>>  1);
        initialCapacity |= (initialCapacity >>>  2);
        initialCapacity |= (initialCapacity >>>  4);
        initialCapacity |= (initialCapacity >>>  8);
        initialCapacity |= (initialCapacity >>> 16);
        initialCapacity++;

        if (initialCapacity < 0)   // Too many elements, must back off
            initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
    }
    return initialCapacity;
}

private void allocateElements(int numElements) {
    elements = new Object[calculateSize(numElements)];
}
```

默认构造器会将`elements`初始化为一个长度为 16 的数组，另外两个构造器将根据参数的长度来初始化，最终会调用`calculateSize`方法计算初始长度。

`calculateSize`方法就比较有意思了，其中用了很多位运算（虽然我们日常用的很少，但 JDK 中随处可见位运算），最终结果就是控制输出的`initialCapacity`参数保持在 2 的次方。

下面我们看看添加元素时的源码，添加元素到双向链表最后都会进入`addLast`方法中：

```java
public void addLast(E e) {
    if (e == null)
        throw new NullPointerException();
    elements[tail] = e;
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)
        doubleCapacity();
}
```
上面代码的意思就是把元素添加到队列末尾，并移动`tail`的值，根据上面的介绍，`tail`移动并不会单纯的向后移动，而是会根据当前`elements`数组的空余长度进行移动，但这里移动`tail`的代码写的很巧妙：
```java
tail = (tail + 1) & (elements.length - 1)
```
我们上面说过，因为`elements`的长度永远是 2 的次方，所以`elements.length - 1`的二进制位都是 1，对其进行 & 运算相当于对`elements.length`进行求余，所以下面两个公式是相等的，不过这种关系必须保证`elements`长度是 2 的次方才成立：
```java
(tail + 1) & (elements.length - 1)
(tail + 1) % elements.length
```
由于位运算的效率比算数运算要高很多，所以 JDK 中很多类似的应用。

好了，回到上面的`addLast`方法中去，该方法回判断`tail == head`是否成立，相等则表示数组已满，满了就进行扩容，也就是调用`doubleCapacity()`方法。
```java
private void doubleCapacity() {
    assert head == tail;
    int p = head;
    int n = elements.length;
    int r = n - p; // number of elements to the right of p
    int newCapacity = n << 1;
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");
    Object[] a = new Object[newCapacity];
    System.arraycopy(elements, p, a, 0, r);
    System.arraycopy(elements, 0, a, r, p);
    elements = a;
    head = 0;
    tail = n;
}
```
扩容操作会创建一个新的数组，长度为原数组的两倍，然后通过两次复制操作将原数组复制到新的数组中。

好了，关于 ArrayDeque 就介绍这么多了，其他的方法大体上都是类似的。