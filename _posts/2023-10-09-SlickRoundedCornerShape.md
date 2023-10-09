---
layout: post
category: computer
title: "【Compose】平滑的解决圆角矩形长度过短的显示问题"
author: ZhangKe
date:   2023-10-09 21:47:40 +0800
---

最近在用到圆角矩形时发现一个问题，如果控件宽度太小，小于圆角直径的话，Compose 会做一些特殊的处理，即按照比例缩小圆角半径，让其仍然看起来是个圆角矩形，如下图。

![Untitled](/assets/img/post/slick-rounded-corner-shape/Untitled.png)

对于有些场景，例如上面的场景，这样做就不太合理了，我们希望圆角半径仍然保持不变，就是这样。

![Untitled](/assets/img/post/slick-rounded-corner-shape/Untitled1.png)

本篇文章内容就是关于如何实现上面的效果。

# Shape

本来我是想直接通过 `Canvas` 来绘制，但发现这么做不够通用，Compose 中一般通过 `*Shape*` 来描述形状，而上面的情况也属于一种圆角矩形形状，那么如果我可以通过定义一个能解决上述情况的 *`Shape`* 的话就可以应用在很多地方了。

# SlickRoundCornerShape

Compose 中提供了一个 `Shape` 接口，其中包含一个方法，我们自定义 `Shape` 时主要就是实现这个方法。

```kotlin
@Immutable
interface Shape {
    /**
     * Creates [Outline] of this shape for the given [size].
     *
     * @param size the size of the shape boundary.
     * @param layoutDirection the current layout direction.
     * @param density the current density of the screen.
     *
     * @return [Outline] of this shape for the given [size].
     */
    fun createOutline(size: Size, layoutDirection: LayoutDirection, density: Density): Outline
}
```

入参就是当前控件的信息，`size` 是控件的大小，返回值是个 `Outline`。

在创建 `Outline` 时我们可以参考已有的 `RoundedCornerShape`.

```kotlin
override fun createOutline(
        size: Size,
        topStart: Float,
        topEnd: Float,
        bottomEnd: Float,
        bottomStart: Float,
        layoutDirection: LayoutDirection
    ) = if (topStart + topEnd + bottomEnd + bottomStart == 0.0f) {
        Outline.Rectangle(size.toRect())
    } else {
        Outline.Rounded(
            RoundRect(
                rect = size.toRect(),
                topLeft = CornerRadius(if (layoutDirection == Ltr) topStart else topEnd),
                topRight = CornerRadius(if (layoutDirection == Ltr) topEnd else topStart),
                bottomRight = CornerRadius(if (layoutDirection == Ltr) bottomEnd else bottomStart),
                bottomLeft = CornerRadius(if (layoutDirection == Ltr) bottomStart else bottomEnd)
            )
        )
    }
```

我们要处理的是 `RoundedCornerShape` 的特殊情况，即宽度小于圆角直径的情况，所以在这基础上我们需要添加一个判断分支来处理。

```kotlin
if (topStart + topEnd + bottomEnd + bottomStart == 0.0f) {
    Outline.Rectangle(size.toRect())
} else if (topStart == bottomStart && size.width < (topStart * 2F)) {
    ...
} else {
    Outline.Rounded(
        RoundRect(
            rect = size.toRect(),
            topLeft = CornerRadius(if (layoutDirection == Ltr) topStart else topEnd),
            topRight = CornerRadius(if (layoutDirection == Ltr) topEnd else topStart),
            bottomRight = CornerRadius(if (layoutDirection == Ltr) bottomEnd else bottomStart),
            bottomLeft = CornerRadius(if (layoutDirection == Ltr) bottomStart else bottomEnd)
        )
    )
}
```

这里为了简单起见，我们只处理 `topStart == bottomStart` 的情形。

现在新增的 `if` 分支就是我们要处理的场景了。

这里还包含了两种情况：

- 控件高度小于等于圆角直径，此时左侧只有一个半圆。

![Untitled](/assets/img/post/slick-rounded-corner-shape/Untitled2.png)

- 控件高度大于圆角直径，此时左侧包含上下两个圆角和一条连接直线。

![Untitled](/assets/img/post/slick-rounded-corner-shape/Untitled3.png)

针对上述两种情况，我们可以通过构建不同的 Path 来实现。

```kotlin
val radius = topStart
val path = Path()
if (height > radius * 2) {
    buildSlickRoundCornerPath(path, size, radius)
} else {
    buildSingleArcPath(path, size, radius)
}
Outline.Generic(path)
```

第一种情况是比较容易处理的。

```kotlin
private fun buildSingleArcPath(path: Path, size: Size, radius: Float) {
    path.moveTo(size.width, 0F)
    path.arcTo(
        rect = Rect(0F, 0F, radius * 2F, radius * 2F),
        startAngleDegrees = 90F,
        sweepAngleDegrees = 180F,
        forceMoveTo = true,
    )
    path.close()
}
```

逻辑比较简单，先移动到空间的右上角，然后绘制一个圆弧，这个圆弧的半径就是圆角半径，然后再闭合这个 `Path` 就行了。

我们主要来看第二种情况。

比较麻烦的是需要根据空间的宽度动态调整控件绘制区域的高度。

![Untitled](/assets/img/post/slick-rounded-corner-shape/Untitled4.png)

图中黄色线断表示控件的宽度，此时我们需要计算红线的长度，然后绘制这部分的圆弧。

还好我们有老祖宗的智慧：**勾股定理**。

```kotlin
val arcHeight = sqrt(radius * radius - (radius - width) * (radius - width))
```

那么圆弧到控件顶部和底部的距离就是：

```kotlin
val yOffset = radius - arcHeight
```

因此上半部分的圆弧可以这么绘制出来：

```kotlin
path.arcTo(
    rect = Rect(
        left = 0F,
        top = yOffset,
        right = width * 2F,
        bottom = yOffset + arcHeight * 2F,
    ),
    startAngleDegrees = 180F,
    sweepAngleDegrees = 90F,
    forceMoveTo = true,
)
```

然后我们把 `Path` 移动到下面圆弧的最底端：

```kotlin
val bottomArcBottom = height - yOffset
path.lineTo(x = width, y = bottomArcBottom)
path.arcTo(
    rect = Rect(
        left = 0F,
        top = bottomArcBottom - arcHeight * 2,
        right = width * 2F,
        bottom = bottomArcBottom,
    ),
    startAngleDegrees = 90F,
    sweepAngleDegrees = 90F,
    forceMoveTo = true,
)
```

上下两个圆弧的大小一致，参数也都差不多，就不用详细介绍了。

然后再封闭 `Path` 即可。

```kotlin
path.lineTo(0F, yOffset + arcHeight)
path.close()
```

这样差不多就能实现上面的效果了。

使用起来也跟普通的 `Shape` 一样。

```kotlin
modifier
    .clipToBounds()
    .background(
        color = Color.Blue.copy(alpha = 0.3F),
        shape = SlickRoundedCornerShape(radius),
    )
// or
modifier
    .clip(SlickRoundedCornerShape(radius))
    .background(color = Color.Blue.copy(alpha = 0.3F))
```

好了，这篇文章就这么多了，实现总体上比较简单，但有些小细节需要注意，所以直接把它分享出来，大家遇到这个问题的话可以直接拿去用。

Compose 发展比较晚，生态还不是特别完善，在应对复杂需求时可能会遇到很多诸如此类的小问题，原本用 View 可以快速实现的东西到了 Compose 这里就需要花不少时间，所以 Compose 社区生态还是要靠我们开发者逐渐完善。

[点击这里](https://github.com/0xZhangKe/compose-ext/blob/main/SlickRoundedCornerShape/src/main/java/com/zhangke/compose/ext/ui/SlickRoundedCornerShape.kt)查看完整代码。
