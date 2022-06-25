---
layout: post
category: computer
title: "Android Matrix 详解"
author: ZhangKe
date:   2019-06-13 22:19:03 +0800
---

我们在自定义 View 控件时随处可见 Matrix 的身影，主要用于坐标转换映射，我们可以通过 Matrix 矩阵来控制视图的变换。

Matrix 本质上是一个如下图所示的矩阵：

![](/assets/img/post/matrix/1.webp)


上面每个值都有其对应的操作。
Matrix 提供了如下几个操作：
- **缩放（Scale）**
对应 **MSCALE_X** 与 **MSCALE_Y**

- **位移（Translate）**
对应 **MTRANS_X** 与 **MTRANS_Y**

- **错切（Skew）**
对应 **MSKEW_X** 与 **MSKEW_Y**

- **旋转（Rotate）**
旋转没有专门的数值来计算，Matrix 会通过计算缩放与错切来处理旋转。

## 数学原理
我们先来简单的复习一下矩阵乘法规则。

![](/assets/img/post/matrix/2.webp)

我们在使用 Matrix 处理视图变换时本质上是通过矩阵映射坐标。
所以上述的几个操作都是对矩阵的操作，我们新建一个 Matrix 后其矩阵为默认状态，其值如下：

![](/assets/img/post/matrix/3.webp)

可以看到默认状态下的数据都是初始值，即不做任何变换处理，所有坐标保持原样。

### 缩放（Scale）
对于单个坐标来说，缩放只要将其坐标值值乘以缩放值即可。
假设对某个点宽度缩放 k1 倍，高度缩放 k2 倍，该点坐标为 x0、y0，缩放后坐标为 x、y，那么缩放的公式如下：

![](/assets/img/post/matrix/4.webp)

我们现在知道了缩放对应矩阵中的两个值的位置以及上面的公式，那现在在用矩阵来描述缩放操作：

![](/assets/img/post/matrix/5.webp)


等号左边的矩阵就是计算后的缩放结果。

Matrix 中用于缩放操作的方法有如下两个：
```java
void setScale(float sx, float sy);
void setScale(float sx, float sy, float px, float py);
```
前面两个参数 sx、sy，分别是宽和高的缩放比例。
第二个重载方法多了两个参数 px、py，这两个参数用来描述缩放的枢轴点，关于枢轴点的含义可以看下注释：

*Set the matrix to scale by sx and sy, with a pivot point at (px, py). The pivot point is the coordinate that should remain unchanged by the specified transformation.*

大概说枢轴点是指定转换应保持不变的坐标。
当我们不传这两个参数时，枢轴点默认为左上角的点，缩放都是向下和向右，所以枢轴点可以大概的理解为缩放的锚点，缩放从这个点开始向四周扩散。
我们用矩阵来描述一下就能明白了。
初始化一个矩阵之后调用缩放方法：
```java
Matrix matrix = new Matrix()
matrix.setScale(0.5F, 0.5F, 300F, 300F);
```
缩放 0.5 倍，枢轴点为 300，调用该方法后矩阵变换为：

![](/assets/img/post/matrix/6.webp)

实际上我们设置了枢轴点后 Matrix 会做一次位移操作，平移距离就是 s * p.

### 位移（Translate）
位移操作是指将坐标（x0,y0）平移一定的距离，我们直接将坐标加上平移的距离即可得到平移后的坐标：

![](/assets/img/post/matrix/7.webp)

用矩阵表示：

![](/assets/img/post/matrix/8.webp)

用于设置位移操作的只有一个方法：
```java
void setTranslate(float dx, float dy);
```

### 错切（Skew）
错切我们平时用的不多，先用一张图来帮助理解。

![](/assets/img/post/matrix/9.webp)

上图是通过下面的代码进行错切的前后对比图。
```java
matrix.setSkew(0.3F, 0.3F);
```
分别设置了水平错切垂直错切的值为 0.3，效果就是上面的样子。
错切公式如下：

![](/assets/img/post/matrix/10.webp)

矩阵描述：

![](/assets/img/post/matrix/11.webp)

错切操作的方法：
```java
void setSkew(float kx, float ky);
void setSkew(float kx, float ky, float px, float py);
```
这里的两个方法类似于缩放操作的两个 setScale 方法，包括最后两个参数也具有相同的意义，这里就不多做介绍了。

### 旋转（Rotate）
旋转的公式比较复杂，证明也比较麻烦，这里不做详细解释了，因为我也没看懂，有兴趣的同学可以自己去国外某知名视频网站找到详细的推导证明视频，这里放一张刚刚看了但是没看懂的截图：

![](/assets/img/post/matrix/12.webp)

注意一下，Android 中的坐标系与上图的不同，所以结果有些区别，那么旋转的公式如下：

![](/assets/img/post/matrix/13.webp)

矩阵描述：

![](/assets/img/post/matrix/14.webp)

用于控制旋转的方法：
```java
void setRotate(float degrees);
void setRotate(float degrees, float px, float py);
```
同上。

上面讲了 Matrix 变换的数学原理，以及其中提供的几个方法，不仅如此，Matrix 还提供复合变换的功能，下面来说一下。

## Matrix 复合变换
复合变换是指矩阵同时实现两种或以上变换，例如在平移的同时改变其大小。

Matrix 的复合变换实际上就是矩阵相乘，原理很简单，但是因为矩阵相乘不符合交换律、且执行顺序对结果会有影响，所以想准确的使用好符合变换需要了解其原理。

上面我们在介绍这几种变换的同时也说了他们对应的方法，可以看到他们都是 set 方法，但 Matrix 中实际上提供了三种操作，分别是：设置（set）、前乘（pre）以及后乘（post）。

所以上述介绍的几个 set 方法都有与之对应的 pre 及 post 方法，方法列表如下：
```java
//scale
boolean preScale(float sx, float sy);
boolean preScale(float sx, float sy, float px, float py);
boolean postScale(float sx, float sy);
boolean postScale(float sx, float sy, float px, float py);

//translate
boolean preTranslate(float dx, float dy);
boolean postTranslate(float dx, float dy);

//skew
boolean preSkew(float kx, float ky);
boolean preSkew(float kx, float ky, float px, float py);
boolean postSkew(float kx, float ky);
boolean postSkew(float kx, float ky, float px, float py);

//rotate
boolean preRotate(float degrees);
boolean preRotate(float degrees, float px, float py);
boolean postRotate(float degrees);
boolean postRotate(float degrees, float px, float py);
```

那么这三种方法（set、pre、post）有什么区别呢？

### 设置（set）
如果我们不需要考虑复合变换的情况，一般可以直接使用 set 方法，因为 set 方法可能会重置之前的 Matrix 状态，导致之前设置的变换失效。

### 前乘（pre）
前乘相当于矩阵右乘：

![](/assets/img/post/matrix/15.webp)

假设当前矩阵 M 为：

![](/assets/img/post/matrix/16.webp)

我们使用 pre 方法做一个平移操作：
```java
matrix.preTranslate(100, 100);
```
变换过程如下：

![](/assets/img/post/matrix/17.webp)

### 后乘（post）
后乘相当于矩阵左乘：

![](/assets/img/post/matrix/18.webp)

我们用上面的矩阵 M 举个例子，同样对其做一个平移操作，但是使用 post 方法：
```java
matrix.postTranslate(100, 100);
```
变换过程如下：

![](/assets/img/post/matrix/19.webp)

这里的前乘后乘的概念主要是由于矩阵不符合乘法交换律引起的，我们使用时一定要注意，除此之外，调用顺序的不同对其结果也有影响，所以我们在使用时需要先确定好矩阵的变换方式，过程之后，再决定如何使用这些方法。

如上便是 Matrix 的一些基本原理。

