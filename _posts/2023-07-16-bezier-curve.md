---
layout: post
category: computer
title: "使用 Compose 绘制渐变贝塞尔曲线趋势图"
author: ZhangKe
date:   2023-07-16 22:56:30 +0800
---

要么说 Compose 优雅呢，假如你想画个东西，用安卓 View 的话你要继承 View 并且实现其中的 `onDraw` 方法，然后才能拿到 `Canvas` 开始绘制，但 Compose 你只需要这样：

```kotlin
Canvas(modifier = Modifier.size(100.dp)) {
    // draw in DrawScope
}
```

非常的自然，没有比这个更自然的事情了。

先看下效果图吧。

![](/assets/img/post/bezier_curve/preview.png)

# 贝塞尔曲线

贝塞尔曲线相关的科普网上有非常多的文章，这里就不详细介绍了，本文主要内容是如何通过 Compose 绘制上面样式的贝塞尔曲线趋势图。

首先简单回顾一下贝塞尔曲线，贝塞尔曲线一般分为一阶贝塞尔曲线（直线），二阶、三阶等更高阶的贝塞尔曲线，通过设置不同的控制点我们可以得到几乎所有类型的曲线。

![](/assets/img/post/bezier_curve/Bezier_3_big.gif)

我们可以通过这个特性绘制出一些很优美的曲线出来。

同样 Compose 的 Canvas 也提供了相关的 API。

```kotlin
// Path.kt

// 二阶贝塞尔曲线
fun quadraticBezierTo(x1: Float, y1: Float, x2: Float, y2: Float)
// 三阶贝塞尔曲线
fun cubicTo(x1: Float, y1: Float, x2: Float, y2: Float, x3: Float, y3: Float)
```

起始点 P0 是 Path 的当前位置，对于初始 Path 来说可以通过 `moveTo` 方法设置初始位置。

# 需求分析

既然我们是希望绘制趋势图，那么趋势图应该是包含了一系列的趋势，趋势我们可以通过 Float 浮点类型来表示。

那么我们的趋势图就会根据这个浮点数据列表进行绘制，因此需要通过一些计算将浮点数转换为坐标值，转换规则也比较简单，计算出最大值和最小值，然后按照每个浮点数的比例分配 x 轴和 y 轴即可。

考虑到上面的样式，如果做的漂亮一点的话，最好是使用三阶贝塞尔曲线。

![](/assets/img/post/bezier_curve/design_3.png)

按照这样的方式设置的控制点绘制出来的曲线比较漂亮。

假如把红点（startPoint）和黄点（endPoint）理解为两个连续的趋势，那么绿点和蓝点就是两个控制点。

绿点的坐标公式为：

```kotlin
val firstControlPoint = Offset(
        x = startPoint.x + (endPoint.x - startPoint.x) / 2F,
        y = startPoint.y,
)
```

蓝点的坐标公式为：

```kotlin
val secondControlPoint = Offset(
        x = startPoint.x + (endPoint.x - startPoint.x) / 2F,
        y = endPoint.y,
)
```

我们只要按照上面的方式计算出控制点位置，然后设置到三阶贝塞尔曲线，就可以了。

另外，考虑到丰富多变的需求，我们尽可能在不增加太多工作量前提下多支持一些样式。

趋势图主要包含三种样式：

- 只有趋势线段（效果图一）
- 只有趋势图着色部分（效果图二）
- 包含线段和着色区（效果图三）

此外还需要支持渐变色。

# 接口设计

首先考虑下样式，上面说了包含三种样式，那我们可以通过 Kotlin 的 `sealed class` 来表示样式。

```kotlin
sealed class BezierCurveStyle {

    // 只有趋势图着色部分
    class Fill(val brush: Brush) : BezierCurveStyle()

		// 只有趋势线段
    class CurveStroke(
        val brush: Brush,
        val stroke: Stroke,
    ) : BezierCurveStyle()

    // 包含线段和着色区
    class StrokeAndFill(
        val fillBrush: Brush,
        val strokeBrush: Brush,
        val stroke: Stroke,
    ) : BezierCurveStyle()
}
```

因为要支持渐变色，所以颜色通过 `Brush` 替代，这样的话即使使用者希望只使用纯色也可以使用 `SolidColor` 来设置参数。

然后我们定义一个名为 `BezierCurve` 的 Composable 函数。

```kotlin
@Composable
fun BezierCurve(
    modifier: Modifier,
    points: List<Float>,
    minPoint: Float? = null,
    maxPoint: Float? = null,
    style: BezierCurveStyle,
)
```

`points` 就是我们刚刚说的趋势点，最大值和最小值这里设置为非必传参数，不传的话我们默认将这个列表中的最大值最小值当作坐标系的最大最小值。

# 具体实现

因为绘制需要计算坐标值，所以需要先获取到画布的大小。

```kotlin
var size by remember {
    mutableStateOf(IntSize.Zero)
}

Canvas(
    modifier = modifier.onSizeChanged {
        size = it
    },
    onDraw = {
        if (size != IntSize.Zero && points.size > 1) {
            drawBezierCurve(
                size = size,
                points = points,
                fixedMinPoint = minPoint,
                fixedMaxPoint = maxPoint,
                style = style,
            )
        }
    },
)
```

 接着需要计算每个点的坐标值：

```kotlin
val total = maxPoint - minPoint
val xSpacing = width / (points.size - 1F)
for (index in points.indices) {
    val x = index * xSpacing
    val y = height - height * ((points[index] - minPoint) / total)
}
```

我们在绘制曲线图时，需要两个分组来绘制，也就是绘制相邻两个点的曲线。

因此我们需要遍历这些点，然后逐个与上一个点构建曲线，最后将构建好的曲线交给 Canvas 绘制。

```kotlin
var lastPoint: Offset? = null
val path = Path()
var firstPoint = Offset(0F, 0F)
for (index in points.indices) {
    val x = index * xSpacing
    val y = height - height * ((points[index] - minPoint) / total)
    if (lastPoint != null) {
        buildCurveLine(path, lastPoint, Offset(x, y))
    }
    lastPoint = Offset(x, y)
    if (index == 0) {
        path.moveTo(x, y)
        firstPoint = Offset(x, y)
    }
}
```

`buildCurveLine` 方法就是用来构建两个点之间的曲线，这里就按照我们上面说的方式计算出贝塞尔曲线的两个控制点即可。

```kotlin
private fun buildCurveLine(path: Path, startPoint: Offset, endPoint: Offset) {
    val firstControlPoint = Offset(
        x = startPoint.x + (endPoint.x - startPoint.x) / 2F,
        y = startPoint.y,
    )
    val secondControlPoint = Offset(
        x = startPoint.x + (endPoint.x - startPoint.x) / 2F,
        y = endPoint.y,
    )
    path.cubicTo(
        x1 = firstControlPoint.x,
        y1 = firstControlPoint.y,
        x2 = secondControlPoint.x,
        y2 = secondControlPoint.y,
        x3 = endPoint.x,
        y3 = endPoint.y,
    )
}
```

由于根据样式的不同，我们可能会需要绘制面，也就是给曲线到坐标系最下方的部分着色，所以还需要提供一个函数用来闭合曲线。

```kotlin
fun closeWithBottomLine() {
    path.lineTo(width.toFloat(), height.toFloat())
    path.lineTo(0F, height.toFloat())
    path.lineTo(firstPoint.x, firstPoint.y)
}
```

到了这里曲线差不多就构建完成了，然后根据样式绘制出来。

```kotlin
when (style) {
    is BezierCurveStyle.Fill -> {
        closeWithBottomLine()
        drawPath(
            path = path,
            style = Fill,
            brush = style.brush,
        )
    }

    is BezierCurveStyle.CurveStroke -> {
        drawPath(
            path = path,
            brush = style.brush,
            style = style.stroke,
        )
    }

    is BezierCurveStyle.StrokeAndFill -> {
        drawPath(
            path = path,
            brush = style.strokeBrush,
            style = style.stroke,
        )
        closeWithBottomLine()
        drawPath(
            path = path,
            brush = style.fillBrush,
            style = Fill,
        )
    }
}
```

代码在这里：[https://gist.github.com/0xZhangKe/0b37c18df37cdcad92e99b91e77d8d54](https://gist.github.com/0xZhangKe/0b37c18df37cdcad92e99b91e77d8d54)

Demo 在这里：[https://gist.github.com/0xZhangKe/f2d4a1771b33f4a046ca9a861ecfc04e](https://gist.github.com/0xZhangKe/f2d4a1771b33f4a046ca9a861ecfc04e)
