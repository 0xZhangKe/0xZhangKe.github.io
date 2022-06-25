---
layout: post
category: computer
title: "Android 多点触控及应用（画板控件 DrawView）"
author: ZhangKe
date:   2019-06-12 22:19:03 +0800
---

多指触控是指监听多个手指的触控事件，我们可以重写 View 中的 onTouchEvent 方法，或者使用 setOnTouchListener 方法来处理触摸事件。

首先我们来看一下如何判断多指触摸时的事件类型。

# MotionEvent 中的事件类型
一般而言，我们通过判断 MotionEvent 的 action 来判断输入事件类型，从而做出相应的处理。
在不考虑多指的情况下，我们一般只关注如下几个事件类型：
- **MotionEvent.ACTION_DOWN**
第一根手指点击屏幕

- **MotionEvent.ACTION_UP**
最后一根手指离开屏幕

- **MotionEvent.ACTION_MOVE**
屏幕上有手指在滑动

- **MotionEvent.ACTION_CANCEL**
事件被拦截


那么对于多指触控来说，除了上述常用的几种事件类型之外，我们还需要关注另外两个事件类型：
- **MotionEvent.ACTION_POINTER_DOWN**
点击前屏幕上已存在手指

- **MotionEvent.ACTION_POINTER_UP**
当屏幕上一根手指被抬起，此时屏幕上仍有别的手指

需要注意的是，上述的两个类型我们不能像以前那样使用 MotionEvent#getAction 方法获取到，需要使用 getActionMasked 才行。

所以再处理多指触控时我们的 onTouch 方法一般可以写成这样：
```java
public boolean onTouchEvent(MotionEvent event) {
    switch (event.getActionMasked()) {
        case MotionEvent.ACTION_DOWN: break;
        case MotionEvent.ACTION_UP: break;
        case MotionEvent.ACTION_MOVE: break;
        case MotionEvent.ACTION_POINTER_DOWN: break;
        case MotionEvent.ACTION_POINTER_UP: break;
    }
    return true;
}
```

当有多跟手指同时触碰屏幕时，我们需要追踪不同的手指。这里面又涉及到另外几个概念。

# 手指追踪
MotionEvent 中提供了几个用于追踪不同手指的方法，再介绍它们之前我先来说一下几个概念。
- **ActionIndex**
事件索引

- **PointerId**
手指 ID

- **PointerIndex**
手指索引

与之对应的几个方法：

| 方法 | 描述 | 备注 |
| ------ | ------ | ------ |
| getActionMasked() | 获取事件类型 | 可获取到多指触控事件类型 |
| getPointerCount() | 获取当前屏幕上的手指个数 |  |
| getActionIndex() | 获取事件索引 |  |
| getPointerId(int) | 获取手指 ID | 参数为 ActionIndex ||
| findPointerIndex(int) | 获取手指索引 | 参数为 PointerId |

### ActionIndex（事件索引）
ActionIndex 通过 getActionIndex 方法可以直接获取到，可以大概地将其理解为用来描述当前事件发生在第几根手指上，例如我们监听到手指抬起时，可能想知道是哪一根手指抬起，那么可以通过 ActionIndex 来判断。
另外，对于同一根手指来说，ActionIndex 的值可能会随着手指的按下与抬起变化的，所以我们不能用它来标识某个手指。
看起来，ActionIndex 唯一的作用就是用来获取 PointerId。
特别需要说明一点，该方法只针对 **ACTION_POINTER_DOWN** 及 **ACTION_POINTER_UP** 事件有效，**ACTION_MOVE** 事件是没法准确的获取到该值的，我们需要结合其他事件综合判断。

### PointerId（手指 ID）
PointerId 通过 getPointerId(int) 方法获取，参数为 ActionIndex。
我们可以通过 PointerId 来标识一根手指。对于同一根手指来说，从按下到抬起整个过程中，PointerId 是固定不变的。
同样需要注意的是，这个值可能会被重用，例如一个手指的 id 是 0，当它抬起后重新按下一根手指，id 可能同样为 0。

### PointerIndex（手指索引）
PointerIndex 通过 findPointerIndex(int) 来获取，参数为 PointerId。
这个值是用来获取该事件更多内容的。
如果我们想获取该事件的点击点位置，当我们通过 getX()/getY() 方法获取坐标时获取到的只能是第一根手指的位置，但是这两个方法提供了一个重载：
```java
float getX(int pointerIndex);
float getY(int pointerIndex);
```
我们可以将 PointerIndex 当做参数来获取到当前手指的点击坐标。
此外还有很多方法都提供了类似的重载，这里就不一一叙述了。


# 应用
通过上面的介绍我们已经大概明白了多点触控的一些关键点了，现在我们来实际应用一下吧。

我这里将做一个用于绘制手指运动轨迹的 DrawView，并且可以同时跟踪多个手指的轨迹，效果图如下：

![DrawView 效果图](/assets/img/post/multi-touch/1.webp)

上图是同时滑动四根手指时的效果。

## 分析
要实现这样的效果，主要有两点问题需要考虑。
第一是如何准确的追踪某根手指的滑动轨迹，因为上面也说了，ACTION_MOVE 是无法获取到 ActionIndex 的。但上帝再关了门同时肯定也会开个窗户，我们可以通过 PointerId 来追踪，首先监听 ACTION_DOWN 及 ACTION_POINTER_DOWN 两个事件，在此处获取到新手指的 PointerId，ACTION_MOVE 事件中遍历所有的手指，然后对比 PointerId 既可。

第二，因为 MotionEvent 是会将连续多个滑动轨迹打包成一个 MotionEvent，我们需要使用 getHistoricalX 获取此次滑动的历史轨迹，该方法签名如下：
```java
float getHistoricalX(int pointerIndex, int pos);
```
第一个参数 pointerIndex 好解决，第一个问题里已经讲过了，主要是第二个参数。
因为 HistoricalX 是个列表，我们需要通过索引去一个一个读取，第二个 pos 参数就是索引，但前提是我们得知道这个列表的长度才行啊。这样只需要一个 for 循环既可解决。
MotionEvent 里面提供了获取这个列表长度的方法：
```java
int getHistorySize();
```
但这提供了这一个方法，没有其他重载，所以没法通过 pointerIndex 获取某跟手指此次滑动的历史轨迹列表长度！
不过经过我的测试，不管哪根手指滑动都可以通过 getHistorySize 方法获取历史轨迹长度，然后调用 getHistoricalX 方法获取历史轨迹坐标，虽然不知道为什么要这么设计，但也算确实解决了这个问题。

## 实现
我们先定义一个内部类用于充当绘制元数据：
```java
private static class DrawPath {

    /**
     * 手指 ID，默认为 -1，手指离开后置位 -1
     */
    private int pointerId = -1;
    /**
     * 曲线颜色
     */
    private int drawColor;
    /**
     * 曲线路径
     */
    private Path path;
    /**
     * 轨迹列表，用于判断目标轨迹是否已添加进来
     */
    private Stack<List<PointF>> record;

    DrawPath(int pointerId, int drawColor, Path path) {
        this.pointerId = pointerId;
        this.drawColor = drawColor;
        this.path = path;
        record = new Stack<>();
    }
}
```
上面的一个 DrawPath 对应一次手指滑动的生命周期，也就是从 DOWN 到 UP 这个中间所经历的的轨迹。

然后再定义一个存放 DrawPath 的列表及画笔，轨迹颜色数组等变量：
```java
    /**
     * 绘制画笔
     */
    private Paint mPaint = new Paint();
    /**
     * 历史路径
     */
    private List<DrawPath> mDrawMoveHistory = new ArrayList<>();
    /**
     * 用于生成随机数，随机取出颜色数组中的颜色
     */
    private Random random = new Random();
```
初始化一下下：
```java
private void init() {
    mPaint.setAntiAlias(true);
    mPaint.setStyle(Paint.Style.STROKE);
    mPaint.setStrokeCap(Paint.Cap.ROUND);
    mPaint.setStrokeWidth(dip2px(getContext(), 5));
}
```
现在我们来重写 onTouchEvent 方法吧：
```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    //多指触控需要使用 getActionMasked
    switch (event.getActionMasked()) {
        case MotionEvent.ACTION_DOWN: {
            //处理点击事件
            performClick();
            //重置所有 PointerId 为 -1
            clearTouchRecordStatus();
            //新增一个轨迹
            addNewPath(event);
            //重绘
            invalidate();
            return true;
        }
        case MotionEvent.ACTION_MOVE: {
            if (mDrawMoveHistory.size() > 0) {
                for (int i = 0; i < event.getPointerCount(); i++) {
                    //遍历当前屏幕上所有手指
                    int itemPointerId = event.getPointerId(i);//获取到这个手指的 ID
                    for (DrawPath itemPath : mDrawMoveHistory) {
                        //遍历绘制记录表，通过 ID 找到对应的记录
                        if (itemPointerId == itemPath.pointerId) {
                            int pointerIndex = event.findPointerIndex(itemPointerId);
                            //通过 pointerIndex 获取到此次滑动事件的所有历史轨迹
                            List<PointF> recordList = readPointList(event, pointerIndex);
                            if (!listEquals(recordList, itemPath.record.peek())) {
                                //判断该 List 是否已存在，不存在则添加进去
                                itemPath.record.push(recordList);
                                addPath(recordList, itemPath.path);
                            }
                        }
                    }
                }
                invalidate();
            }
            return true;
        }
        case MotionEvent.ACTION_POINTER_UP:
            //屏幕上有一根指头抬起，但有别的指头未抬起时的事件
            int pointerId = event.getPointerId(event.getActionIndex());
            for (DrawPath item : mDrawMoveHistory) {
                if (item.pointerId == pointerId) {
                    //该手指已绘制结束，将此 PointerId 重置为 -1
                    item.pointerId = -1;
                }
            }
            break;
        case MotionEvent.ACTION_POINTER_DOWN:
            //屏幕上已经有了手指，此时又有别的手指点击时事件
            addNewPath(event);
            invalidate();
            break;
        case MotionEvent.ACTION_UP:
            //最后一根手指抬起，重置所有 PointerId
            clearTouchRecordStatus();
            break;
        case MotionEvent.ACTION_CANCEL:
            //事件被取消
            clearTouchRecordStatus();
            break;
    }
    return true;
}
```
上面都有注释，我就不详细说了。
然后是重写 onDraw 方法，这虽然是个绘制轨迹的控件，但其实 onDraw 方法里反倒没多少代码：
```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    if (mDrawMoveHistory == null || mDrawMoveHistory.isEmpty()) {
        return;
    }
    for (DrawPath item : mDrawMoveHistory) {
        mPaint.setColor(item.drawColor);
        canvas.drawPath(item.path, mPaint);
    }
}
```
这样就实现了一个简易的支持多指绘制控件啦，我们还可以向其中添加一些诸如撤销上一步等操作的方法，这里就不多说了。
DrawView 的完整代码已经放到了 Github 上，欢迎查看：

[https://github.com/0xZhangKe/Collection/blob/master/DrawView/DrawView.java](https://github.com/0xZhangKe/Collection/blob/master/DrawView/DrawView.java)
