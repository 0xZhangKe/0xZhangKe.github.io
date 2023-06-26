---
layout: post
category: computer
title: "MVI 架构在 Compose 中的事件处理方式"
author: ZhangKe
date:   2023-06-26 22:44:00 +0800
---

# MVI

MVI 架构核心思想是**单一可信数据源**。

`ViewModel` 中需要维护着一个 `UiState` ( 一般来说会是个 `data class` )，这个 `UiState` 包含了 UI 层所需要的所有数据，或者说描述了 UI 层的所有状态，同时这个 `UiState` 应该具备通知观察者更新的能力，例如使用 `**StateFlow**`。

 UI 层应该仅通过该 `UiState` 渲染。那么对于这个页面的 `Composable` 函数来说，入参中表示数据的部分应该只有一个 `UiState`。

```kotlin
fun NavGraphBuilder.registerLoginPage(navController: NavController){
    composable("login"){
        val viewModel: LoginViewModel = viewModel()
        val uiState = viewModel.uiStateFlow.collectAsState().value
        LoginPage(uiState = uiState)
    }
}

@Composable
private fun LoginPage(
    uiState: LoginUiState,
) {
// do somethings
}
```

# 事件上浮

事件上浮是指应该将事件处理尽可能上浮，一般来说应该上浮到页面的 `Composable` 函数的上一级。

也就是说，页面的 `Composable` 函数应该包含了这个页面所有的事件回调。

上一级是指页面路由注册的地方，我们可以将其视为 `Activity/Fragment`，在这里拿到所有的事件回调，并交给 `ViewModel`。

```kotlin
fun NavGraphBuilder.registerLoginPage(navController: NavController) {
    composable("login") {
        val viewModel: LoginViewModel = viewModel()
        val uiState = viewModel.uiStateFlow.collectAsState().value
        LoginPage(
            uiState = uiState,
            onBackClick = navController::popBackStack,
            onLoginClick = viewModel::onLoginClick,
        )
    }
}

@Composable
private fun LoginPage(
    uiState: LoginUiState,
    onBackClick: () -> Unit,
    onLoginClick: () -> Unit,
) {
// do somethings
}
```

# ViewModel 中的事件

一般来说事件是在 UI 层通过监听用户手势而被触发的，但也有一些事件是**在 `ViewModel` 层被触发**的。

例如网络请求失败后的错误消息提示，结束页面或者打开新页面等等。

我们可以先考虑**网络请求成功后打开新页面**这个场景。

鉴于 Compose 提供了副作用相关的一些函数，以及 MVI 单一可信数据源的思想，我们可能会考虑在 `UiState` 中提供一个 `Boolean` 值表示是否需要打开页面，并将其作为 Key 通过副作用函数打开新页面。

真的这么做的话可能会发现一些问题，例如打开新的页面后退出回到当前页面，结果又自动打开了新页面，此时我们会意识到应该在页面打开后更新字段为 `false`，然后可能还会发现其他问题。

这么做无疑是很麻烦的，本质上是因为我们**混淆了事件与数据这两者的概念**。

**数据是用于填充 UI 元素的对象，这些元素会因为不同的数据而有所区别，并在视觉上有所体现。**

**事件是指软件在运行过程中的某个时间点由于某些特定的原因 ( 例如用户手势 ) 而需要做出的一系列变更中的一个。**

那么显而易见的是，打开页面就应该属于事件。

具体而言，我们应该如何处理呢。

我们可以在 `ViewModel` 中定义一个叫 `openMainPageFlow` 的对象，它应该是个 `SharedFlow`，然后 UI 层通过监听这个 Flow 来打开页面。

```kotlin
// LoginViewModel.kt
class LoginViewModel : ViewModel() {

		private val _openMainPageFlow: MutableSharedFlow<Unit> = MutableSharedFlow()
    val openMainPageFlow: SharedFlow<Unit> = _openMainPageFlow
		// other code ...
}

// LoginNavigation.kt
fun NavGraphBuilder.registerLoginPage(navController: NavController) {
    composable("login") {
        val viewModel: LoginViewModel = viewModel()
        val uiState = viewModel.uiStateFlow.collectAsState().value
        LoginPage(
            uiState = uiState,
            onBackClick = navController::popBackStack,
            onLoginClick = viewModel::onLoginClick,
        )
				val openMainPageFlow = viewModel.openMainPageFlow
        LaunchedEffect(openMainPageFlow) {
            openMainPageFlow.collect {
                navController.navigate("main")
            }
        }
    }
}
```