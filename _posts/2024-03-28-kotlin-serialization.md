
---
layout: post
category: computer
title: "Kotlin-Serialization 介绍，Kotlin 上最好用的序列化工具"
author: ZhangKe
date:   2024-03-28 09:49:30 +0800
---

# 介绍

Kotlin Serialization 是 Kotlin 提供的**跨平台**序列化和反序列的库，它可以将对象树序列化成一些常见的格式，纯天然支持 Kotlin，扩展性很强，几乎可以满足所有业务场景，而且不需要使用反射，性能很好，可以说是目前 Kotlin 语言序列化工具的不二之选。

`kotlinx.serialization` 目前支持的格式有如下几种：

- JSON: [kotlinx-serialization-json](https://github.com/Kotlin/kotlinx.serialization/blob/master/formats/README.md#json)
- Protocol Buffers: [kotlinx-serialization-protobuf](https://github.com/Kotlin/kotlinx.serialization/blob/master/formats/README.md#protobuf)
- CBOR: [kotlinx-serialization-cbor](https://github.com/Kotlin/kotlinx.serialization/blob/master/formats/README.md#cbor)
- Properties: [kotlinx-serialization-properties](https://github.com/Kotlin/kotlinx.serialization/blob/master/formats/README.md#properties)
- HOCON: [kotlinx-serialization-hocon (only on JVM)](https://github.com/Kotlin/kotlinx.serialization/blob/master/formats/README.md#hocon)

大部分情况下我们使用的是 JSON，所以本篇文章主要会介绍 JSON 相关的使用。

Kotlin 序列化分为两个过程，第一步是将对象树转换成由基础数据类型组成的序列，第二步是将这个序列按照格式编码输出。

![](/assets/img/post/kotlin-serialization/Untitled.png)

# 集成

Kotlin 序列化工具在单独的组件中：https://github.com/Kotlin/kotlinx.serialization 。

[https://github.com/Kotlin/kotlinx.serialization](https://github.com/Kotlin/kotlinx.serialization)

其中包含了如下几个部分：

- Gradle 编译插件：org.jetbrains.kotlin.plugin.serialization
- 运行时依赖库

首先需要在 gradle 中添加编译插件：

```jsx
plugins {
    id 'org.jetbrains.kotlin.plugin.serialization' version '1.9.23'
}
```

然后添加依赖库：

```jsx
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-core:1.6.2")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.2")
}

```

此外，`kotlinx.serialization` 有自己独立的版本，跟 Kotlin 并不同步，具体版本[看这里](https://github.com/Kotlin/kotlinx.serialization/releases)。

# Json 编码

将数据转换成指定格式的过程称之为编码，对于 Kotlin 序列化编码来说，通过使用扩展函数`Json.encodeToString` 实现。

```jsx
@Serializable
class Project(val name: String, val language: String)

fun main() {
    val data = Project("kotlinx.serialization", "Kotlin")
    println(Json.encodeToString(data))
}
```

Kotlin 序列化并不是使用反射，所以对于支持序列化/反序列化的类应该使用 `@Serializable` 注解标记。

# Json 解码

相反的过程称为解码。要将 JSON 字符串解码为对象，我们将使用`Json.decodeFromString`扩展函数。为了指定我们想要获得的结果类型，我们向该函数提供一个类型参数。

```jsx
@Serializable
data class Project(val name: String, val language: String)

fun main() {
    val data = Json.decodeFromString<Project>("""
        {"name":"kotlinx.serialization","language":"Kotlin"}
    """)
    println(data)
}
```

# @Serializable 注解

该注解用于标记一个可被序列化的类，序列化规则如下：

- 只有具有 backing fields 的属性才会参与序列化过程，代理属性或者有 get/set 的属性不会参与。
- 主构造器中的参数必须是对象属性
- 对于想在序列化完成之前验证数据的场景，可以在类的 init 块中验证入参。

### 可选属性

在将 Json 字符串反序列化成对象时，如果 Json 字符串中缺失类中的某个属性，反序列化将会失败，但我们可以通过向属性添加默认值来避免这种情况。

```jsx
@Serializable
data class Project(val name: String, val language: String = "Kotlin")

fun main() {
    val data = Json.decodeFromString<Project>("""
        {"name":"kotlinx.serialization"}
    """)
    println(data)
}
```

### @Required

@Required 注解表示在反序列过程中，用该注解修饰的属性必须非空。

如果一个字段有默认值的同时，我们又期望反序列过程中输入的 Json 中必须包含这个属性，那么可以使用该注解，使用之后如果 Json 中不存在这个属性将会反序列化失败。

### @Transient

如果某个属性不需要被序列化，那么可以用该注解修饰，且该属性必须有默认值。

默认情况下，如果一个标记了 `@Transient` 的属性在序列化后的字符串中有同名属性，那么反序列化成该对象时会报错，可以在构建 Json 时 使用 `ignoreUnknownKeys = true` 避免报错。
`ignoreUnknownKeys` 的含义是忽略序列化字符串中的未知字段。默认是 false，这样如果反序列化字符串中包含了额外的字段反序列化时也不会报错。

### 默认值不参与序列化

如果一个属性有默认值，且这个对象中该属性的值也是默认值，那么这个值就不会参与反序列化。

```jsx
@Serializable
data class User(val firstname: String, val lastname: String = "Zhang")

fun main() {
    val data = User("Ke")
    println(Json.encodeToString(data))
    // 输出： {"firstname":"Ke"}
}
```

如上，因为构建的 `data` 中的 `language` 属性是默认值，所以序列化数据不会包含 `lastname` .

如果给 `lastname` 设置一个不等于默认值的值的话，那么就会参与序列化了。

```jsx
fun main() {
    val data = User("Ke", "Li")
    println(Json.encodeToString(data))
    // 输出： {"firstname":"Ke","lastname":"Li"}
}
```

当然，也有办法规避这一点。

### @EncodeDefault

这个注解就是为了解决上面的问题，它可以让默认值同样参与序列化。

另外，`@EncodeDefault` 注解还可以通过使用 `EncodeDefault.Mode` 参数将其调整为相反的行为。

## **Serial field names**

Kotlin Serialization 支持自定义序列化和反序列化的属性名，通过 `@SerialName` 注解实现。

# 枚举

Kotlin Serialization 支持枚举类，并且不需要在枚举类上使用 `@Serializable` 注解。

当然，如果你想自定义序列化后的属性名，也可以加上 `@Serializable` 注解，然后通过`@SerialName` 设置。

# 预先支持的类型

Kotlin Serialization 除了支持基本类型和 String 之外，还预先支持了一些复合数据类型。

- Pair
- Triple
- Array
- List
- Set
- Map
- Unit
- 单例类
- Duration
- Nothing

其中的 Unit 和单例类的序列化/反序列化一样，由于 Unit 本身也是单例类，所以他们序列化的内容都是一个空的 Json 字符串。

# **Serializers/序列化器**

如上所说，从对象到基本数据类型的过程称为序列化，而序列化过程就是由序列化器 *Serializer* 控制的。

上面介绍的几个预先支持序列化的数据类型就是默认提供了对应的 *Serializer，*或者是基本类型的 Serializer*。*

## 基本类型序列化器

要获取基本类型序列化器，可以直接使用扩展函数。

```kotlin
val intSerializer: KSerializer<Int> = Int.serializer()
println(intSerializer.descriptor)
// output: PrimitiveDescriptor(kotlin.Int)
```

## 预先提供的序列化器

要获取 Kotlin 预先提供的序列化器，可以通过顶级函数 *`serializer()`* 获取。

```kotlin
enum class Status { SUPPORTED }

val pairSerializer: KSerializer<Pair<Int, Int>> = serializer()
val statusSerializer: KSerializer<Status> = serializer()
println(pairSerializer.descriptor)
println(statusSerializer.descriptor)
// output: 
// kotlin.Pair(first: kotlin.Int, second: kotlin.Int)
// com.zhangke.algorithms.Status(SUPPORTED)
```

`serializer()` 函数接受一个范型参数用于获取指定类型的 Serializer，所以实际上我们可以通过这个函数获取所有可序列化类的 Serializer。

## 编译器插件生成的 Serializer

我们在给一个类添加 `@Serializable` 注解之后编译插件会自动生成对应的 Serializer，然后通过这个类对象的扩展函数即可获取到。

```kotlin
@Serializable
class User(val name: String)

val userSerializer:KSerializer<User> = User.serializer()
println(userSerializer.descriptor)
// output: com.zhangke.algorithms.User(name: kotlin.String)
```

## 编译器插件生成的范型类的 Serializer

对于支持序列化的范型类，范型类型也需要支持序列户才行，范型类对应的 `serializer()` 函数需要传入范型类型的 Serializer。

```kotlin
@Serializable
class Box<T>(val contents: T)

val userSerializer:KSerializer<Box<User>> = Box.serializer(User.serializer())
println(userSerializer.descriptor)
// output: com.zhangke.algorithms.Box(contents: com.zhangke.algorithms.User)
```

## 集合类型的 Serializer

集合类型的 Serializer 有这三种：ListSerializer() / MapSerializer() / SetSerializer()，集合类型的 Serializer 同样需要按照范型类的方式传入参数。

```kotlin
val stringListSerializer: KSerializer<List<String>> = ListSerializer(String.serializer()) 
println(stringListSerializer.descriptor)
```

## 自定义 Serializer

大部分情况下我们直接使用注解以及预制的 Serializer 就够了，但有些时候我们可能想控制序列化过程，或者给某些无法添加注解的类序列化，就需要用到自定义 Serializer 了。

要自定义 Serializer，我们需要创建一个类，实现 KSerializer 接口。

```kotlin
class Color(val rgb: Int)

object ColorAsStringSerializer : KSerializer<Color> {
    override val descriptor: SerialDescriptor = PrimitiveSerialDescriptor("Color", PrimitiveKind.STRING)

    override fun serialize(encoder: Encoder, value: Color) {
        val string = value.rgb.toString(16).padStart(6, '0')
        encoder.encodeString(string)
    }

    override fun deserialize(decoder: Decoder): Color {
        val string = decoder.decodeString()
        return Color(string.toInt(16))
    }
}
```

重写的 `descriptor` 属性是这个 Serializer 的描述符，你可以选择自己去实现 `SerialDescriptor`，也可以向上面一样创建一个基础类型的 `PrimitiveSerialDescriptor`。

然后另外两个函数一目了然，分别是序列化和反序列化。

### 委托 Serializer

Serializer 可以将序列化/反序列化过程委托给其它的 Serializer 来完成，比如我们可以先把 Color 转成整数数组，然后委托给 `IntArraySerializer` 。

```kotlin
class ColorIntArraySerializer : KSerializer<Color> {
    private val delegateSerializer = IntArraySerializer()
    override val descriptor = SerialDescriptor("Color", delegateSerializer.descriptor)

    override fun serialize(encoder: Encoder, value: Color) {
        val data = intArrayOf(
            (value.rgb shr 16) and 0xFF,
            (value.rgb shr 8) and 0xFF,
            value.rgb and 0xFF
        )
        encoder.encodeSerializableValue(delegateSerializer, data)
    }

    override fun deserialize(decoder: Decoder): Color {
        val array = decoder.decodeSerializableValue(delegateSerializer)
        return Color((array[0] shl 16) or (array[1] shl 8) or array[2])
    }
}
```

创建好 Serializer 之后需要根据不同的场景来使用。

### 直接设置在类上

```kotlin
@Serializable(with = ColorAsStringSerializer::class)
class Color(val rgb: Int)
```

### 设置在属性上

```kotlin
@Serializable
class Table(
    @Serializable(with = ColorAsStringSerializer::class) val color: Color
)
```

### 序列化时使用

将 Serializer 作为第一个参数传给 `Json.encodeToString` 函数。

```kotlin
Json.encodeToString(ColorAsStringSerializer, Color(0xFF0000))
```

### 给范型设置 Serializer

@Serializable 注解可以用于范型类型。

```kotlin
@Serializable          
class ProgrammingLanguage(
    val name: String,
    val releaseDates: List<@Serializable(DateAsLongSerializer::class) Date>
)
```

### 为文件指定 Serializer

Kotlin 同样允许给文件设置序列化器。

```kotlin
@file:UseSerializers(DateAsLongSerializer::class)
```

这样设置之后，该文件中的类将会自动应用这个 Serializer。

## 上下文序列化

截止目前，我们上面的序列化过程都是静态的，但有些时候，我们需要给同一个类动态选择不同的 Serializer。

例如，我们希望给 Date 类序列化成不同标准的字符串，但同时这取决于不同的接口版本，我们只有在运行时才能确定应该使用哪个标准的 Serializer。

这种情况下，我们可以先不给 Date 设置具体的 Serializer，而是添加一个 @Contextual 标记，这表示这个属性将会根据 Json 中的上下文来决定使用哪个 Serializer。

```kotlin
@Serializable          
class ProgrammingLanguage(
    val name: String,
    @Contextual 
    val stableReleaseDate: Date
)
```

为了提供上下文，我们需要创建一个 SerializersModule 实例，它描述了在运行时应该使用哪些 Serializer 来序列化那些标记了 Contextual 的类。

```kotlin
private val module = SerializersModule { 
    contextual(Date::class, DateAsLongSerializer)
}
val format = Json { serializersModule = module }
```

现在，上下文模块信息已经存储在了 `format` 对象中，只要使用这个 `format` 对象，上面的 Date 类就会使用 `DateAsLongSerializer` 序列化。

设置好之后，就可以使用 `format` 对象序列化了。

```kotlin
fun main() {
    val data = ProgrammingLanguage("Kotlin", SimpleDateFormat("yyyy-MM-ddX").parse("2016-02-15+00"))
    println(format.encodeToString(data))
}
```

# 多态类的序列化

对于类的多态，也就是有继承关系的类，序列化时会遇到一些问题，Kotlin Serialization 对此也提供了解决方案。

## 密封类-sealed class

一种解决方案是使用 sealed class，Kotlin 序列化是支持密封类序列化的，只要加上 `@Serializable` 注解即可。

```jsx

@Serializable
sealed class Project {
    abstract val name: String
}
            
@Serializable
class OwnedProject(override val name: String, val owner: String) : Project()

fun main() {
    val data: Project = OwnedProject("kotlinx.coroutines", "kotlin")
    println(Json.encodeToString(data)) // Serializing data of compile-time type Project
}

// output: {"type":"com.zhangke.OwnedProject","name":"kotlinx.coroutines","owner":"kotlin"}
```

 可以看到，Kotlin 为了解决多态类序列化问题，会在序列化内容中新增一个 `type` 字段用于表示对象的具体类型，在反序列化时会根据这个类型来创建相应的对象。

当然，type 字段的值是可以自定义的，有些时候你可能不希望用默认的包名，那么也可以选择自定义。

```jsx
@Serializable         
@SerialName("owned")
class OwnedProject(override val name: String, val owner: String) : Project()
```

如上所示，自定义 `type` 只需要给相应的子类添加 `@SerialName` 注解即可。

## 注册子类

除了上面的使用 `sealed class` 之外，还可以使用注册子类的方式序列化多态类。

注册子类是指需要在 Json 中构建序列化 Module，其中提供接口和子类的对应关系，让 `serializer` 知道应该选择哪些子类序列化。

```kotlin
@Serializable
abstract class Project {
    abstract val name: String
}

@Serializable
@SerialName("owned")
class OwnedProject(override val name: String, val owner: String) : Project()

@Serializable
class JavaProject(override val name: String): Project()

val module = SerializersModule {
    polymorphic(
        baseClass = Project::class,
        actualClass = OwnedProject::class,
        actualSerializer = serializer(),
    )
    polymorphic(
        baseClass = Project::class,
        actualClass = JavaProject::class,
        actualSerializer = serializer(),
    )
}

val format = Json { serializersModule = module }

fun main() {
    val list = listOf(
        OwnedProject("kotlinx.coroutines", "kotlin"),
        JavaProject("sun.java")
    )
    println(format.encodeToString(list))
}
// output: [{"type":"owned","name":"kotlinx.coroutines","owner":"kotlin"},{"type":"com.zhangke.algorithms.JavaProject","name":"sun.java"}]
```

先构建 `SerializersModule`，通过 `polymorphic` 函数注册子类信息，然后将这个 Module 设置到 Json 中就行了。

## 接口序列化

上面注册子类的方式可以解决抽象类的场景，但是接口仍然没办法序列户，因为 `@Serializable` 注解不允许加在接口上，不过 Kotlin 序列化会使用 `PolymorphicSerializer` 策略隐式序列化接口，这意味着我们不需要给接口添加 `@Serializable` 注解，剩下的按照上面注册子类的方式来就行了。

```kotlin
interface Project {
    val name: String
}

@Serializable
@SerialName("owned")
class OwnedProject(override val name: String, val owner: String) : Project

@Serializable
class JavaProject(override val name: String) : Project

val module = SerializersModule {
    polymorphic(
        baseClass = Project::class,
        actualClass = OwnedProject::class,
        actualSerializer = serializer(),
    )
    polymorphic(
        baseClass = Project::class,
        actualClass = JavaProject::class,
        actualSerializer = serializer(),
    )
}

val format = Json { serializersModule = module }

fun main() {
    val list = listOf(
        OwnedProject("kotlinx.coroutines", "kotlin"),
        JavaProject("sun.java")
    )
    println(format.encodeToString(list))
}
// output: [{"type":"owned","name":"kotlinx.coroutines","owner":"kotlin"},{"type":"com.zhangke.algorithms.JavaProject","name":"sun.java"}]
```

# Json 配置

上面一直在说 serializer 的属于序列化过程，下面开始介绍编码过程，本文只介绍 Json 编码相关的内容。

Json 编码/解码通过  kotlinx 中的 Json 类实现，我们可以直接调用 Json 获取到一个全局默认唯一的 Json 对象，也可以自己构建一个 Json 对象。

因为 Json 内部可能会有缓存，考虑到性能问题，建议将构建好的 Json 对象保存起来然后复用，最好不要用一次构建一次。

### 输出格式化

默认情况下，Json 输出的是单行字符串，但你可以通过将 `prettyPrint` 设置为 true 使其输出一个漂亮的 Json 格式。

```kotlin
val format = Json { prettyPrint = true }
```

现在他将输出一个漂亮的 Json 字符串：

```kotlin
{
    "name": "kotlinx.serialization",
    "language": "Kotlin"
}
```

### 宽松的解析

默认情况下，Json 将按照严格的规范来解析 Json，比如键值必须用引号，整型和字符串类型的限制等，但可以通过 `isLenient = true` 使用宽松模式。

在该模式下，带引号的值可以因为 Kotlin 对象中对应的类型是整型而尝试解析成整型，键值也可以不带引号。

### 忽略未知键

默认情况下，Json 在反序列化过程中遇到未知的键会报错，比如 Json 字符串中有个 id 的字段，但是反序列化的目标类中并没有 id 属性，那么就会报错。

可以通过设置 `ignoreUnknownKeys = true` 忽略未知键来避免报错，使其正常解析。

### 替换 Json names

我们上面介绍的 `@SerialName` 可以给一个字段设置一个在 Json 中的名字，但同时这个字段本身的名字就无法被解析了。而 `@JsonNames` 注解可以给字段设置多个名字，并且这个字段原本的值仍然可以被解析。

```kotlin
@Serializable
data class Project(@JsonNames("title") val name: String)

fun main() {
  val project = Json.decodeFromString<Project>("""{"name":"kotlinx.serialization"}""")
  println(project)
  val oldProject = Json.decodeFromString<Project>("""{"title":"kotlinx.coroutines"}""")
  println(oldProject)
}
```

另外，对 `@JsonNames` 注解的支持由 `JsonBuilder.useAlternativeNames` 标志控制。与大多数配置标志不同，此标志默认启用。

### 强制使用默认值

通过使用 `coerceInputValues = true` 可以将默写无效的输入转为默认值。

目前支持的无效输入只有如下两种：

- null 不可空的输入
- 枚举的未知值

这意味着，如果某个类的某个属性有默认值，那么在反序列化的时候，如果这个字段在 Json 中满足如上两个条件之一，那将会使用这个属性的默认值来反序列化。

```kotlin
val format = Json { coerceInputValues = true }

@Serializable
data class Project(val name: String, val language: String = "Kotlin")

fun main() {
    val data = format.decodeFromString<Project>("""
        {"name":"kotlinx.serialization","language":null}
    """)
    println(data)
}
// output: Project(name=kotlinx.serialization, language=Kotlin)
```

### 显示空值

默认情况下，null 也会被编码到 Json 中，可以通过设置 `explicitNulls = false` 设置 null 值不序列化到 Json 中。

### 结构化 Json 键

JSON 格式本身不支持结构化的键，一般来说只能是字符串。

但可以通过使用  `allowStructuredMapKeys = true` 属性启用对结构化键的非标准支持。

```kotlin
val format = Json { allowStructuredMapKeys = true }

@Serializable
data class Project(val name: String)

fun main() {
    val map = mapOf(
        Project("kotlinx.serialization") to "Serialization",
        Project("kotlinx.coroutines") to "Coroutines"
    )
    println(format.encodeToString(map))
}
```

具有结构化键的映射表示为包含以下项目的 JSON 数组：`[key1, value1, key2, value2,...]`。

```kotlin
[{"name":"kotlinx.serialization"},"Serialization",{"name":"kotlinx.coroutines"},"Coroutines"]
```

### 其它

除此之外，Json 中还有很多设置，可以满足非常丰富的场景。

- `allowSpecialFloatingPointValues` - 使用 NaN 和无穷大等特殊浮点值。
- `classDiscriminator` - 设置多态数据的类型的键的名称。
- `decodeEnumsCaseInsensitive` - 以不区分大小写的方式解码枚举
- `namingStrategy` - 全局命名策略，设置为 JsonNamingStrategy.SnakeCase 可以从驼峰命名法的字段解析成蛇形命名法。

## Json Element

Json 作为编码/解码的工具，也同样将其内部的 JsonElement 相关的类和工具暴露出来供大家使用。

可以通过 `Json.parseToJsonElement` 函数解析出 JsonElement 对象。

JsonElement 类型和 Gson 中的以及大多数的 Json 工具中的类型都几乎一致，这里就不做赘述了。

### Json Element Builder

Json 提供了一些 DSL 用于构建 JsonElement。

```kotlin
fun main() {
    val element = buildJsonObject {
        put("name", "kotlinx.serialization")
        putJsonObject("owner") {
            put("name", "kotlin")
        }
        putJsonArray("forks") {
            addJsonObject {
                put("votes", 42)
            }
            addJsonObject {
                put("votes", 9000)
            }
        }
    }
    println(element)
}
```

然后也可以通过 `Json.decodeFromJsonElement` 函数将其直接反序列化成对象。

```kotlin
@Serializable
data class Project(val name: String, val language: String)

fun main() {
    val element = buildJsonObject {
        put("name", "kotlinx.serialization")
        put("language", "Kotlin")
    }
    val data = Json.decodeFromJsonElement<Project>(element)
    println(data)
}
```

## Json transformations

Json 提供了自定义编码/解码 Json 数据的能力，可以影响序列化后的 Json 内容。

自定义 Json 转换器通过 `JsonTransformingSerializer` 实现，它也实现了 `KSerializer` 接口。

下面举个例子。

现在有一个 User 类，我希望在序列化成 Json 时能在序列化后的 Json 数据中自动添加一个 time 字段表示序列化发生的时间。

```kotlin
@Serializable
class User(
    val name: String,
    val age: Int,
)

object UserSerializer : JsonTransformingSerializer<User>(User.serializer()) {
    override fun transformSerialize(element: JsonElement): JsonElement {
        return buildJsonObject {
            element.jsonObject.forEach { key, value ->
                put(key, value)
            }
            put("time", System.currentTimeMillis())
        }
    }
}
fun main() {
    val user = User("zhangke", 18)
    println(format.encodeToString(UserSerializer, user))
}
// output: {"name":"zhangke","age":18,"time":1711556475153}
```

如上所示，首先定义 UserSerializer，然后在返回的 JsonObject 中先添加原有的字段，然后添加一个 time 字段就行了。

---

好了，本文就介绍到这里了，更详细的内容可以去官网查看。

[https://kotlinlang.org/docs/serialization.html](https://kotlinlang.org/docs/serialization.html)