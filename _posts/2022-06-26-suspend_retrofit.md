---
layout: post
category: computer
title: "一种 Kotlin suspend 函数结合 Retrofit 使用的姿势"
author: ZhangKe
date:   2022-06-26 17:40:03 +0800
---


在 Kotlin 协程以前我们在使用 Retrofit 的时候一般会结合 RxJava 一起使用，通过 `Single` 来表示一个已经创建的请求。

```kotlin
@GET("/info")
fun requestUserInfo(): Single<BaseEntry<UserInfo>>
...
api.requestUserInfo()
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe({
        //...
    },{
        //...
    })
```

`Single` 在 RxJava 中表示只包含一个事件的流，所以用它来充当网络请求也是相对合理的。

不过既然是 HTTP 网络请求，那么返回值肯定是只有一次或者抛出异常，用流来形容似乎不太合适，毕竟用单个事件组成的流听起来就很奇怪。所以对于发起网络请求这种形式的函数来说，直接用返回数据或者抛出异常是更贴合实际的。

那么对于 Kotlin 来说，这种函数就是 `suspend` 函数。并且 Retrofit 也已经支持了 `suspend` 函数。

我们再看一下 `suspend` + Retrofit 的网络请求是怎样的。

```kotlin
@GET("/info")
suspend fun requestUserInfo(): BaseEntry<UserInfo>
...
viewModelScope.launch {
    try {
        val response = withContext(Dispatchers.IO) {
            api.requestUserInfo()
        }
        //...
    } catch (e: Exception) {
        //...
    }
}
```

这样看起来是比较符合直觉的，调用请求函数直接返回响应数据，或者抛出异常。

但看起来还是有点问题，有点啰嗦，而且还要写 `try catch`，不够简单优雅。

其实也可以换成 Flow，但感觉仍然不是最佳方案，Flow 的设计就不适合用在这里，肯定有更好的解决办法。

前段时间在转协程的时候遇到了这个问题，当时想到了一个看起来还不错的解决方案，在这里分享出来。

目前要解决的问题就是期望使用 `suspend` 函数发起网络请求并直接返回响应数据，但同时又不希望写 `try catch`。

我的想法是在 Retrofit 获取到请求后构建一个响应类包装数据和异常，调用方通过这个类来获取到数据或者异常。其他的仍然按照之前的逻辑既可，写法上类似下面这样。

```kotlin
@GET("/info")
suspend fun requestUserInfo(): Response<BaseEntry<UserInfo>, ErrorEntry>
...
viewModelScope.launch {
    val response = withContext(Dispatchers.IO){
        api.requestUserInfo()
    }
    response.onSuccess { 
        //...
    }
    response.onError { 
        //...
    }
}
```

`Response` 就是上面说的包装类， 其中的 `onSuccess` 及 `onError` 代码块对应成功或者失败，看起来比之前简单多了吧。

下面介绍一下如何实现。

既然需要给 Retrofit 添加一个返回类型，那就需要先创建一个 `Response` 对应的 `CallAdapterFactory`。在获取到响应数据或者异常发生之后构建一个 `Response`，并将数据或异常存入其中，大体上就是这个思路，比较简单，然后我们分步骤看一下具体实现。

### Response

先看一下上面说的 Response 是如何定义的。

```kotlin
open class Response<S, E: ErrorResponse.ErrorEntryWithMessage> {

    var success: Boolean = true

    var successData: S? = null

    var errorData: ErrorResponse<E>? = null

    inline fun onSuccess(block: (S) -> Unit) {
        if (success) {
            block(successData!!)
        }
    }

    inline fun onError(block: (ErrorResponse<E>) -> Unit) {
        if (!success) {
            block(errorData!!)
        }
    }
}
```

这比较简单，`success` 表示该请求是否成功，这会在 `Response` 构建后由外部赋值。泛型 S 表示请求成功后返回的数据类型，泛型 E 则是请求成功但是接口返回了错误数据的类型。

ErrorResponse 定义如下。

```kotlin
sealed class ErrorResponse<T : ErrorResponse.ErrorEntryWithMessage>(val message: String) {

    data class ApiError<T : ErrorEntryWithMessage>(val data: T?, val code: Int) :
        ErrorResponse<T>(data?.errorMessage ?: appContext.getString(R.string.unknown_error))

    data class NetworkError<T : ErrorEntryWithMessage>(val error: Throwable) :
        ErrorResponse<T>(appContext.getString(R.string.network_error))

    interface ErrorEntryWithMessage {
        val errorMessage: String
    }
}
```

考虑到对调用方友好，这里限制错误数据至少要包含一个 errorMessage.

### 创建 CallAdapterFactory

先创建一个 `ResponseCallAdapterFactory` 继承 `CallAdapter.Factory` 并实现其中的 `get` 方法。

```kotlin
class ResponseCallAdapterFactory : CallAdapter.Factory() {

    override fun get(
        returnType: Type,
        annotations: Array<out Annotation>,
        retrofit: Retrofit
    ): CallAdapter<*, *>? {
        return null
    }
}
```

Retrofit 会调用这个方法尝试获取对应类型的 `CallAdapter`，通过它来构建最终的返回值。

### 判断请求返回类型是否是 Response

在这个 `get` 方法中需要先通过入参判断是否是我们需要处理的返回类型，不是则返回 `null`。

```kotlin
// Kotlin suspend function will wrap result with retrofit.Call
if (Call::class.java != getRawType(returnType)) return null
if (returnType !is ParameterizedType) {
    return null
}
val responseType = getParameterUpperBound(0, returnType)
val responseRawType = getRawType(responseType)
if (!Response::class.java.isAssignableFrom(responseRawType)) {
    return null
}
if (responseType !is ParameterizedType) return null
@Suppress("UNCHECKED_CAST")
val responseInstance =
    getRawType(responseType).newInstance() as Response<*, ErrorResponse.ErrorEntryWithMessage>
val successType = getParameterUpperBound(0, responseType)
val errorType =
    if (responseRawType == Response::class.java) {
        fetchErrorTypeFromResponse(responseType)
    } else {
        fetchErrorType(responseType) ?: return null
    }
if (!ErrorResponse.ErrorEntryWithMessage::class.java.isAssignableFrom(getRawType(errorType))) {
    return null
}
```

上面代码主要是判断是否是 `suspend` 函数以及返回值是否为我们的上面说的 `Response`，如果不是则不需要处理，直接返回 `null` 结束。另外还需要获取具体的 Response 类型（Response 被设计为可继承的）以及泛型类型。

### 构建 CallAdapter

代码按照上面的逻辑走到这里，就意味着遇到了一个我们需要处理的函数，此时需要结合上面获取到的类型来创建并返回对应的 CallAdapter 对象。

```kotlin
val errorBodyConverter: Converter<ResponseBody, ErrorResponse.ErrorEntryWithMessage> =
    retrofit.nextResponseBodyConverter(null, errorType, annotations)
return ResponseCallAdapter(
    responseInstance,
    successType,
    errorBodyConverter
)
```

上面返回的 ResponseCallAdapter 也是我们创建的类。

```kotlin
class ResponseCallAdapter<S, E : ErrorResponse.ErrorEntryWithMessage>(
    private val responseInstance: Response<S, E>,
    private val successType: Type,
    private val errorConverter: Converter<ResponseBody, E>
) : CallAdapter<S, Call<Response<S, E>>> {

    override fun responseType(): Type = successType

    override fun adapt(call: Call<S>): Call<Response<S, E>> {
        return ResponseCall(call, responseInstance, errorConverter)
    }
}
```

这个类主要还是用来确定处理类型已经再构建一个 `Call` 对象。

### 创建 ResponseCall

ResponseCall 需要继承 Call 并设置泛型类型。

```kotlin
class ResponseCall<S, E : ErrorResponse.ErrorEntryWithMessage>(
    private val delegate: Call<S>,
    private val responseInstance: Response<S, E>,
    private val errorConverter: Converter<ResponseBody, E>
) : Call<Response<S, E>> {

    override fun enqueue(callback: Callback<Response<S, E>>) {
        TODO("Not yet implemented")
    }

    override fun isExecuted(): Boolean = delegate.isExecuted

    override fun clone(): Call<Response<S, E>> {
        return ResponseCall(delegate.clone(), responseInstance, errorConverter)
    }

    override fun cancel() {
        delegate.cancel()
    }

    override fun isCanceled(): Boolean = delegate.isCanceled

    override fun execute(): retrofit2.Response<Response<S, E>> {
        throw UnsupportedOperationException("ResponseCall not support execute")
    }

    override fun request(): Request = delegate.request()

    override fun timeout(): Timeout = delegate.timeout()
}
```

大部分方法代理给 `delegate` 对象既可，其中 `execute` 会直接抛异常，因为这里 `suspend` 函数。

主要逻辑还是在 `enqueue` 函数内。

```kotlin
override fun enqueue(callback: Callback<Response<S, E>>) {
    return delegate.enqueue(object : Callback<S> {

        override fun onResponse(call: Call<S>, response: retrofit2.Response<S>) {
            if (response.isSuccessful) {
                val body = response.body()
                responseInstance.success = true
                responseInstance.successData = body
            } else {
                val error = response.errorBody()
                val errorBody = when {
                    error == null -> null
                    error.contentLength() == 0L -> null
                    else -> {
                        try {
                            errorConverter.convert(error)
                        } catch (e: Exception) {
                            null
                        }
                    }
                }
                responseInstance.success = false
                responseInstance.errorData = ErrorResponse.ApiError(errorBody, response.code())
            }
            callback.onResponse(this@ResponseCall, retrofit2.Response.success(responseInstance))
        }

        override fun onFailure(p0: Call<S>, p1: Throwable) {
            responseInstance.success = false
            responseInstance.errorData = ErrorResponse.NetworkError(p1)
            callback.onResponse(this@ResponseCall, retrofit2.Response.success(responseInstance))
        }
    })
}
```

仍然是先调用 `delete` 方法的 `enqueue` 方法，`Callback` 中会包含请求结束的 `retrofit2.Response`，我们就拿到它来继续构建我们的 `Response`。

---

另外，一般来说一个服务的接口数据类型都是约定好不变的，即所有的响应数据都具有相同的 `ErrorEntry`，那么为了方便，可以定义一个具体服务的 Response，其中的 Error 泛型就是这个服务的  `ErrorEntry`。这也是上面的 Response 被设计成 `open` 的原因。

例如 Notion 的公开 API 就具备相同的 Error 类型。

```kotlin
// {"object":"error","status":401,"code":"unauthorized","message":"API token is invalid."}
data class ErrorEntry(

    @SerializedName("object")
    val objectType: String,

    val status: Int,

    val code: String?,

    val message: String?,
) : ErrorResponse.ErrorEntryWithMessage {

    override val errorMessage: String
        get() = message ?: appContext.getString(R.string.unknown_error)
}
```

我们就可以给 Notion 的服务定义一个 NotionResponse。

```kotlin
class NotionResponse<T> : Response<T, ErrorEntry>()
...
@POST("v1/search")
suspend fun queryAllPages(@Body body: RequestBody): NotionResponse<NotionListEntry<NotionPage>>
```

好了，到这里这篇文章就结束了，我前段时间学习 Compose 时拿着 Notion 的 API 做了个叫 NotionLight 的开源项目（一个轻便的 Notion 客户端），本篇文章内容也是在做这个项目时遇到的问题，上面所有的代码以及具体的使用案例也都在这个开源项目中，大家可以移步至 GitHub 看到具体的代码。

[https://github.com/0xZhangKe/NotionLight](https://github.com/0xZhangKe/NotionLight)