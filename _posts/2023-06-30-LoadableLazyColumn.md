---
layout: post
category: computer
title: "亲手封装一个简单灵活的下拉刷新上拉加载 Compose Layout"
author: ZhangKe
date:   2023-06-30 23:30:40 +0800
---

# 亲手封装一个简单灵活的下拉刷新上拉加载 Compose Layout

Compose 的下拉刷新有现成的 [Material 库](https://developer.android.com/reference/kotlin/androidx/compose/material/pullrefresh/package-summary)可以直接使用，非常简单方便。

但是上拉加载目前没看到有封装的特别好的库，Paging 有些场景无法满足，而且上拉加载也是个比较简单的功能，没必要再去依赖一个质量未知的库。我们可以基于目前的 LazyList 简单的封装一个灵活的组件。

基本原则是仍然基于现有的 PullRefresh 以及 LazyList API 实现，不依赖三方库，使用简单灵活好用。

# 接口设计

首先我们将这个可以上拉加载下拉刷新的 Compose 函数命名为 `LoadableLazyColumn`。

上面说到我们需要基于 PullRefresh 以及 LazyList API 实现，这两个组件都具备各自的 State。

- `PullRefreshState` ：下拉刷新的 State
- `LazyListState` ：LazyColumn 的 State

由于我们需要在此基础上提供上拉加载的能力，那还需要一个上拉加载的 State，我们可以将其命名为 `LoadMoreState` ，目前 `LoadMoreState` 需要包含两个参数：

1. `loadMoreRemainCountThreshold` ：加载更多的剩余 Item 个数阈值，当剩余个数小于等于这个阈值时开始发起加载更多请求。
2. `onLoadMore` ：加载更多的事件回调。

既然提供了 `LoadMoreState` ，我们还应该提供一个对应的 remember 函数。

```kotlin
@Composable
fun rememberLoadMoreState(
    loadMoreRemainCountThreshold: Int,
    onLoadMore: () -> Unit,
): LoadMoreState {
    return remember {
        LoadMoreState(loadMoreRemainCountThreshold, onLoadMore)
    }
}
```

上面我们只是单纯的定义了 `LoadMoreState`，同时我们也知道了 `LoadableLazyColumn` 还包含另外两个 State，总共也就是三个 State。

现在我们需要创建 `LoadableLazyColumnState` ，它需要包含上面说的三个 State。

```kotlin
@OptIn(ExperimentalMaterialApi::class)
data class LoadableLazyColumnState(
    val lazyListState: LazyListState,
    val pullRefreshState: PullRefreshState,
    val loadMoreState: LoadMoreState,
)
```

以及对应的 `remember` 方法。

不过上面说的三个 state 只是我们的内部实现，这不是调用者需要考虑的事情，对于使用者来说这只是一个 state，因此我们的 `remember` 方法的参数应该是这三个 state 的合集。

```kotlin
@Composable
@ExperimentalMaterialApi
fun rememberLoadableLazyColumnState(
    refreshing: Boolean,
    onRefresh: () -> Unit,
    onLoadMore: () -> Unit,
    refreshThreshold: Dp = PullRefreshDefaults.RefreshThreshold,
    refreshingOffset: Dp = PullRefreshDefaults.RefreshingOffset,
    loadMoreRemainCountThreshold: Int = 5,
    initialFirstVisibleItemIndex: Int = 0,
    initialFirstVisibleItemScrollOffset: Int = 0
): LoadableLazyColumnState {
    val pullRefreshState = rememberPullRefreshState(
        refreshing = refreshing,
        onRefresh = onRefresh,
        refreshingOffset = refreshingOffset,
        refreshThreshold = refreshThreshold,
    )

    val lazyListState = rememberLazyListState(
        initialFirstVisibleItemScrollOffset = initialFirstVisibleItemScrollOffset,
        initialFirstVisibleItemIndex = initialFirstVisibleItemIndex,
    )

    val loadMoreState = rememberLoadMoreState(loadMoreRemainCountThreshold, onLoadMore)

    return remember(pullRefreshState, lazyListState, loadMoreState) {
        LoadableLazyColumnState(
            lazyListState = lazyListState,
            pullRefreshState = pullRefreshState,
            loadMoreState = loadMoreState,
        )
    }
}
```

这样我们就创建了 `LoadableLazyColumnState`。

然后 `LoadableLazyColumn` 这个函数的入参就显而易见了。

```kotlin
@OptIn(ExperimentalMaterialApi::class)
@Composable
fun LoadableLazyColumn(
    modifier: Modifier = Modifier,
    state: LoadableLazyColumnState,
    refreshing: Boolean,
    loading: Boolean,
    contentPadding: PaddingValues = PaddingValues(0.dp),
    reverseLayout: Boolean = false,
    verticalArrangement: Arrangement.Vertical =
        if (!reverseLayout) Arrangement.Top else Arrangement.Bottom,
    horizontalAlignment: Alignment.Horizontal = Alignment.Start,
    flingBehavior: FlingBehavior = ScrollableDefaults.flingBehavior(),
    userScrollEnabled: Boolean = true,
    loadingContent: (@Composable () -> Unit)? = null,
    content: LazyListScope.() -> Unit,
)
```

# 实现方案

这里会根据 `LazyList` 滑动事件来触发加载更多事件，当滑动事件结束后，判断用户是否为向下滑动，并且剩余元素的个数小于等于设定的阈值。

所幸 `lazyListState` 提供了这些状态，我们可以通过它那计算出上面的情况。

```kotlin
val lazyListState = state.lazyListState
// 获取 lazyList 布局信息
val listLayoutInfo by remember { derivedStateOf { lazyListState.layoutInfo } }
```

可以通过下面的方法获取到 LazyList 是否正在滑动：

```kotlin
// Whether this [ScrollableState] is currently scrolling by gesture, 
// fling or programmatically ornot.
lazyListState.isScrollInProgress
```

然后通过下面的两个方法获取到最后一个可见的 `index`，以及 `item` 总数：

```kotlin
listLayoutInfo.visibleItemsInfo.lastOrNull()?.index
listLayoutInfo.totalItemsCount
```

上面说的几个方法都是获取当前状态，但我们的目的是判断状态的变化，主要是下面两个事件变化：

- 滑动停止事件
- 最后一个可见 index 变化事件

如果我们能在滑动事件停止后判断最后一个可见 index 与上次滑动结束后的最后一个可见 index 相比的大小，就知道是向上滑动还是向下滑动了。再加上最后一个可见 index 与阈值相比，就可以判断触发加载更多事件了。

这里我们使用 `remember` 函数来实现，即 `remember` 上次的值，与当前值做对比。

```kotlin
// 上次是否正在滑动
var lastTimeIsScrollInProgress by remember {
    mutableStateOf(lazyListState.isScrollInProgress)
}
// 上次滑动结束后最后一个可见的index
var lastTimeLastVisibleIndex by remember {
    mutableStateOf(listLayoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0)
}
// 当前是否正在滑动
val currentIsScrollInProgress = lazyListState.isScrollInProgress
// 当前最后一个可见的 index
val currentLastVisibleIndex = listLayoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
```

通过上面的代码我们就拿到了所有需要的状态了，然后简单对比一下即可。

```kotlin
if (!currentIsScrollInProgress && lastTimeIsScrollInProgress) {
    if (currentLastVisibleIndex != lastTimeLastVisibleIndex) {
        val isScrollDown = currentLastVisibleIndex > lastTimeLastVisibleIndex
        val remainCount = listLayoutInfo.totalItemsCount - currentLastVisibleIndex - 1
        if (isScrollDown && remainCount <= state.loadMoreState.loadMoreRemainCountThreshold) {
            LaunchedEffect(Unit) {
                state.loadMoreState.onLoadMore()
            }
        }
    }
    // 滑动结束后再更新值
    lastTimeLastVisibleIndex = currentLastVisibleIndex
}
lastTimeIsScrollInProgress = currentIsScrollInProgress
```

这样就差不多了，看下所有的代码。

```kotlin
package com.zhangke.framework.loadable.lazycolumn

import androidx.compose.foundation.gestures.FlingBehavior
import androidx.compose.foundation.gestures.ScrollableDefaults
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.LazyListScope
import androidx.compose.foundation.lazy.LazyListState
import androidx.compose.foundation.lazy.rememberLazyListState
import androidx.compose.material.CircularProgressIndicator
import androidx.compose.material.ExperimentalMaterialApi
import androidx.compose.material.pullrefresh.PullRefreshDefaults
import androidx.compose.material.pullrefresh.PullRefreshIndicator
import androidx.compose.material.pullrefresh.PullRefreshState
import androidx.compose.material.pullrefresh.pullRefresh
import androidx.compose.material.pullrefresh.rememberPullRefreshState
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.derivedStateOf
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp

@OptIn(ExperimentalMaterialApi::class)
@Composable
fun LoadableLazyColumn(
    modifier: Modifier = Modifier,
    state: LoadableLazyColumnState,
    refreshing: Boolean,
    loading: Boolean,
    contentPadding: PaddingValues = PaddingValues(0.dp),
    reverseLayout: Boolean = false,
    verticalArrangement: Arrangement.Vertical =
        if (!reverseLayout) Arrangement.Top else Arrangement.Bottom,
    horizontalAlignment: Alignment.Horizontal = Alignment.Start,
    flingBehavior: FlingBehavior = ScrollableDefaults.flingBehavior(),
    userScrollEnabled: Boolean = true,
    loadingContent: (@Composable () -> Unit)? = null,
    content: LazyListScope.() -> Unit,
) {
    val lazyListState = state.lazyListState
    // 获取 lazyList 布局信息
    val listLayoutInfo by remember { derivedStateOf { lazyListState.layoutInfo } }
    Box(
        modifier = modifier
            .pullRefresh(state.pullRefreshState)
    ) {
        LazyColumn(
            contentPadding = contentPadding,
            state = state.lazyListState,
            reverseLayout = reverseLayout,
            verticalArrangement = verticalArrangement,
            horizontalAlignment = horizontalAlignment,
            flingBehavior = flingBehavior,
            userScrollEnabled = userScrollEnabled,
            content = {
                content()
                item {
                    if (loadingContent != null) {
                        loadingContent()
                    } else {
                        if (loading) {
                            Box(modifier = Modifier.fillMaxWidth()) {
                                CircularProgressIndicator(
                                    modifier = Modifier
                                        .size(30.dp)
                                        .align(Alignment.Center)
                                )
                            }
                        }
                    }
                }
            },
        )
        PullRefreshIndicator(
            refreshing,
            state.pullRefreshState,
            Modifier.align(Alignment.TopCenter)
        )
    }
    // 上次是否正在滑动
    var lastTimeIsScrollInProgress by remember {
        mutableStateOf(lazyListState.isScrollInProgress)
    }
    // 上次滑动结束后最后一个可见的index
    var lastTimeLastVisibleIndex by remember {
        mutableStateOf(listLayoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0)
    }
    // 当前是否正在滑动
    val currentIsScrollInProgress = lazyListState.isScrollInProgress
    // 当前最后一个可见的 index
    val currentLastVisibleIndex = listLayoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
    if (!currentIsScrollInProgress && lastTimeIsScrollInProgress) {
        if (currentLastVisibleIndex != lastTimeLastVisibleIndex) {
            val isScrollDown = currentLastVisibleIndex > lastTimeLastVisibleIndex
            val remainCount = listLayoutInfo.totalItemsCount - currentLastVisibleIndex - 1
            if (isScrollDown && remainCount <= state.loadMoreState.loadMoreRemainCountThreshold) {
                LaunchedEffect(Unit) {
                    state.loadMoreState.onLoadMore()
                }
            }
        }
        // 滑动结束后再更新值
        lastTimeLastVisibleIndex = currentLastVisibleIndex
    }
    lastTimeIsScrollInProgress = currentIsScrollInProgress
}

@Composable
@ExperimentalMaterialApi
fun rememberLoadableLazyColumnState(
    refreshing: Boolean,
    onRefresh: () -> Unit,
    onLoadMore: () -> Unit,
    refreshThreshold: Dp = PullRefreshDefaults.RefreshThreshold,
    refreshingOffset: Dp = PullRefreshDefaults.RefreshingOffset,
    loadMoreRemainCountThreshold: Int = 5,
    initialFirstVisibleItemIndex: Int = 0,
    initialFirstVisibleItemScrollOffset: Int = 0
): LoadableLazyColumnState {
    val pullRefreshState = rememberPullRefreshState(
        refreshing = refreshing,
        onRefresh = onRefresh,
        refreshingOffset = refreshingOffset,
        refreshThreshold = refreshThreshold,
    )

    val lazyListState = rememberLazyListState(
        initialFirstVisibleItemScrollOffset = initialFirstVisibleItemScrollOffset,
        initialFirstVisibleItemIndex = initialFirstVisibleItemIndex,
    )

    val loadMoreState = rememberLoadMoreState(loadMoreRemainCountThreshold, onLoadMore)

    return remember(pullRefreshState, lazyListState, loadMoreState) {
        LoadableLazyColumnState(
            lazyListState = lazyListState,
            pullRefreshState = pullRefreshState,
            loadMoreState = loadMoreState,
        )
    }
}

@Composable
fun rememberLoadMoreState(
    loadMoreRemainCountThreshold: Int,
    onLoadMore: () -> Unit,
): LoadMoreState {
    return remember {
        LoadMoreState(loadMoreRemainCountThreshold, onLoadMore)
    }
}

data class LoadMoreState(
    val loadMoreRemainCountThreshold: Int,
    val onLoadMore: () -> Unit,
)

@OptIn(ExperimentalMaterialApi::class)
data class LoadableLazyColumnState(
    val lazyListState: LazyListState,
    val pullRefreshState: PullRefreshState,
    val loadMoreState: LoadMoreState,
)
```
