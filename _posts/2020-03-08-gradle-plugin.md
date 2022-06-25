---
layout: post
category: computer
title: "Gradle 依赖与插件"
author: ZhangKe
date:   2020-03-08 18:16:03 +0800
---

关于 Gradle 中的基础、Task 等知识，可以看我的[上一篇文章](/computer/2020/03/08/gradle-basic.html)。 

# Gradle 中的依赖
Gradle 中的依赖可以分为脚本文件依赖、插件依赖以及包依赖。
## 脚本文件依赖
随着项目结构的复杂，一个 build.gradle 已经无法满足我们的需求了，尤其是对依赖库版本的配置，如果多个 project 都需要用到某个依赖库，稍有不慎版本就会错乱，从而引发一些问题。

此时我们期望可以把所有用到的依赖库版本都配置在同一个文件中，build.gradle 使用这个文件中的版本来依赖相应的版本，Gradle 提供了 **apply** 方法来依赖其他文件。
```
apply from: 'config.gradle'
```
我们可以在 build.gradle 文件中添加上述代码来依赖 config.gradle 文件，这样就可以把这个文件中的设置应用到对应的 Project 中去，**包括其中的 Task**。

## 插件依赖
插件依赖是指依赖**编译插件**，最常见的是我们新建一个 Android 项目时，对应 module 的 build.gradle 文件第一行自动添加的安卓插件：
```
apply plugin: 'com.android.application'
```
与上一节的脚本文件依赖一样，都使用 apply 函数添加依赖，其中 plugin 后面跟的是插件的 ID 或者全限定类名，我们后面会介绍如何设置插件名。
通过上面的一行代码就可以把这个插件应用到当前的编译脚本中去，但对于自定义插件来说，我们还需要做一些其他的工作。
对于 Gradle 来说，依赖一个插件需要分如下三步：

**1. 设置这个插件对应的仓库地址**
我们首先应该设置一些仓库地址，告诉 Gradle 应该去哪里下载我们配置的这些插件和依赖包。
设置仓库地址通过 **repositories** 函数来实现，当我们创建好一个项目后，在根 build.gradle 文件中一般会自动添加如下两个依赖仓库：
```
allprojects {
    repositories {
        google()
        jcenter()
    }
}
```
上面代码就是在本项目中添加了两个仓库：[google](https://maven.google.com/web/index.html) 以及 [jcenter](https://bintray.com/bintray/jcenter).
因为 google 和 jcenter 两个仓库比较常用，所以默认提供了这两个方法，如果我们希望设置自己的仓库也是可以的，以 Github 的仓库为例：
```
allprojects {
    repositories {
        maven { url "https://jitpack.io" }
    }
}
```
实际上，除了通过上述的方式配置仓库地址外，还有另一种方式设置仓库，就是在项目中创建 buildSrc 子项目，这个后面会详细介绍。

**2. 设置这个插件所在包的全限定名**
仓库地址配置好后，我们就可以使用该仓库中的插件了，插件依赖使用 **classpath** 函数来完成：
```
buildscript {
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.3'
    }
}
```
这一步相当于去上面配置的仓库中下载这个插件所在的依赖包，这个依赖包中可能包含很多个插件，有了依赖包我们才是使用其中的插件。

**3. 通过插件名将该插件设置到对应的编译脚本中**
到了这里，准备工作就已经全部搞定了，该有的东西都有了，现在如果我们希望在当前的编译脚本中使用某个插件，直接使用上面介绍过的 **apply** 函数既可：
```
apply plugin: 'com.android.application'
```
另外，这里面说的插件是指编译脚本插件，是在编译时使用的工具，用来控制编译的，跟实际的项目开发时的代码没有关系，与我们常说的在项目开发中使用的依赖包不同。

## 包依赖
包依赖其实是个笼统的概念，包括上面说的插件依赖其实也属于包依赖，本章所说的包依赖单纯是指在实际项目开发中使用的一些**第三方的依赖库**，例如 [OkHttp](https://square.github.io/okhttp/ "OkHttp")、[Gson](https://github.com/google/gson "Gson") 等等，这其实不属于编译脚本的范畴，但毕竟也是 Gradle 依赖的一部分，所以也大概说一下。

包依赖与脚本依赖本质上都一样，都分为三步：

**1. 设置包所在的仓库**
设置仓库地址跟上面说的一样，都是通过 **repositories** 函数来实现。

**2. 下载需要使用的依赖包**
声明依赖包需要在对应项目的 build.gradle 的 **dependencies** 函数中完成。
```
dependencies {
    // 声明一个本地模块的依赖
    implementation project(":mylibrary")
    // 声明一个本地二进制包的依赖
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    // 声明一个远程二进制包的依赖
    implementation 'com.example.android:app-magic:12.3'
}
```
此外，还有很多种不同的声明的依赖方式，每种方式都具有不同的依赖特质，具体可以看下表：

![](/assets/img/post/gradle-plugin/1.webp)

**3. 使用该依赖包**
通过上面两个步骤之后，我们就可以在 Java 代码或者其他语言的代码中使用依赖包中的代码了。
上面就是依赖包的使用方式。我们可以使用 dependencies Task 查看指定项目的依赖树：
```
//查看 app 模块的依赖树
gradlew app:dependencies
```
上面的命令将会输出一个树形的依赖关系表。

# Gradle 插件
上面叙述的 Task 都是完成一些简单的动能，但对于一些更加复杂，或者可以被更广泛使用的功能来说，直接在 gradle 文件中定义 Task 已经很难满足我们的需求了，因此，Gradle 提供了插件的概念来解决这个问题。

编写 Gradle 插件有两种方式，一种是创建一个名为 **buildSrc** 的子项目，在这个子项目中编写插件代码，buildSrc 是 Gradle 插件的**默认目录**，会把这个目录下的插件自动添加到当前项目中去，我们可以在别的子项目中直接使用。
另一种方式是把插件写在其他目录，然后**上传到 maven 仓库**，其他项目通过 maven 的方式依赖这个插件。
为了方便起见，这里以 buildSrc 为例，由于 buildSrc **不是标准的 Android 项目**，所以我们先手动 New module 一个子模块出来，然后选择 Java Library。

创建好后，打开这个模块的 build.gradle 文件，把最上面一行的 java-library 插件改成 groovy 插件，以及添加需要的依赖包：
```
//apply plugin: 'java-library'
apply plugin: 'groovy'

dependencies {
    implementation gradleApi()
    implementation localGroovy()
}
```

然后，因为插件使用的是 groovy 编写，所以还需要把 buildSrc 下面的 java 目录删掉改成  groovy 目录，完成后如下图所示：

![](/assets/img/post/gradle-plugin/2.webp)

这样一个 groovy 项目就创建好了。
我们现在开始编写插件，编写插件的方式很简单，只需要在 groovy 目录下创建一个**实现了 Plugin 接口的 groovy 类**既可：
```
class PrintVersionPlugin implements Plugin<Project> {
    @Override
    void apply(Project target) {
        target.tasks.create("PrintVersionTask"){
            doLast{ println("PrintVersionPlugin version:1.0") }
        }
    }
}
```
Plugin 接口只包含一个 apply  方法，我们拿到该方法的 project 参数后，就可以做一些自己的操作了，project 提供了丰富的 api，我们可以完成很多事情，例如创建一个 Task。

插件写好后，我们还需要为该插件创建一个 id，使用这个插件时直接使用这个 id 既可。创建 ID 映射很简单，在 groovy **同级目录**下创建一个 resources\META-INF\gradle-plugins 文件夹，然后在这个文件下创建 ID 映射文件，**文件名**就是这个插件的 id，后缀为 **properties**，然后把插件类的全限定名按照如下格式写进去既可。
例如我们把上面插件的 ID 命名为 printVersion，那么我们创建一个 printVersion.properties 文件，然后编写如下代码：
```
implementation-class=com.zhangke.gradle.PrintVersionPlugin
```
此时的 buildSrc 项目目录结构如下：

![](/assets/img/post/gradle-plugin/3.webp)

这个时候我们就可以在别的子项目中使用这个插件了:
```
apply plugin: 'printVersion'
```
如果我们希望在一个单独的目录下编写插件，然后上传到 maven 仓库也是可以的，上传 maven 仓库可以用 maven 插件很容易实现，这里就不具体叙述了。

# 参考文献
[1]实战Gradle.Benjamin Muschko.电子工业出版社,2015.9.


如果觉得还不错的话，欢迎关注我的个人公众号：**zhangke_blog**
