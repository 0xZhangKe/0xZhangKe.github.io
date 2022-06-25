---
layout: post
category: computer
title: "Gradle 扫盲与 Task 基础"
author: ZhangKe
date:   2020-03-08 10:43:03 +0800
---

[Gradle](https://gradle.org/) 是用于构建项目的工具，除了管理依赖库之外，Gradle 还支持我们自己添加编译脚本、添加编译配置等控制项目的构建，通过提供 API 我们可以控制编译的每一步操作。


Gradle 目前使用最广泛的是 [Android 项目的构建](https://developer.android.com/studio/build "Android 项目的构建")，几年前 Google 推出 Android Studio 的同时也把它也推选为默认的构建工具，因此我们也经历了从 Maven 到 Gradle 这一痛苦的转变过程，每天对着满屏的编译失败信息怀疑人生。

实际上 Gradle 也确实对开发者很不友好，用 Gradle 你能遇到各种各样的问题，版本混乱到无以复加，互相还不兼容，甚至对 Android Studio 都有版本要求。不过几乎所有的问题都能 Google 到答案，而我们也能看到确实在逐渐变好。

关于 Gradle 中的依赖与自定义插件，可以点此看我的[下一篇文章](/computer/2020/03/08/gradle-plugin.html)。

# Gradle 基本原理
我们知道 Gradle 是一种以 [Groovy 语言](https://groovy-lang.org/ "Groovy 语言")为基础的**自动化构建工具**，一般通过修改 build.gradle 脚本来完成对项目构建的一些设置，例如依赖管理等等。大多数情况下，我们只需要稍微修改下 gradle 文件即可完成自己的需求。

自动化构建工具听起来似乎比较复杂，本质上来说也是一种程序，跟我们自己写的代码一样，我们开始编译时就启动这个程序，然后读取我们在 gradle 文件中配置的参数来实例化各个类，然后按照顺序依次执行对应的任务即可完成整个构建任务。

所以 build.gradle 文件，或者其他后缀为 gradle 的文件其实就是个**配置文件**，就好像 xml 一样，我们在 gradle 文件中修改各种配置参数，Gradle 通过这些参数来实例化 Project 等等就像构造器一样，只要理解了这点学习 Gradle 就会变得很容易。

当我们新建一个项目后，Gradle 默认会生成一些编译脚本文件，主要有：setting.gradle、build.gradle 以及子项目中的 build.gradle 等等，还会在当前目录下生成一个 gradle 文件夹，下面分别介绍一些这些文件的作用：

- setting.gradle 用来告诉 gradle 这个项目包含了那些**子项目**。
- build.gradle 是默认的构建脚本，当我们在执行 gradle 命令时，会首先到当前目录下寻找该文件，然后通过该文件的配置**实例化一个 Project 对象**。
- 自动生成的 gradle 文件夹是 Gradle 包装器，其中包含一个 jar 文件和一个 配置文件，使用这个包装器可以让 Gradle 运行在一个特定的版本上，目的是创造一个**独立于系统、系统配置和 Gradle 版本的可靠和可重复构建**。

Gradle 中有两个重要的概念，分别是 **Project** 和 **Task**，Project 表示一个正在构建的项目，而 Task 表示某一个具体的任务。

# Project
Project 表示**正在构建的项目**，每个 Project 都对应一个我们在 setting.gradle 中配置的 Project，除此之外还有一个 Root Project。

Project 用来管理、描述当前正在构建的项目，我们可以通过其中提供的 api 进行一些操作来达到自己的目的。

一个 Project  可以创建新的 Task，添加依赖关系和配置，并应用插件和其他的构建脚本。

另外，Project 是通过我们常见的 build.gradle 文件实例化出来的，我们在 build.gradle 文件中进行的各种配置最终都会应用到 Project 中去。
## Property
Project 本身提供了一些默认的属性，我们也可以向 Project 中添加自定义的属性。添加属性通过 ext 函数设置。
```
ext{
    p1 = "p1"
}

ext.p2 = "p2"
```
上面两种写法都可以，向 Project 添加属性之后可以直接访问。
```
task printProperty {
    doLast{
        println p1
        println p2
    }
}
```

# Task
Task 定义了一个当前**任务执行时的最小单元**，一个 Project 中一般会存在很多个 Task，通过执行这些 Task 来完成整个项目的构建。也就是说，项目构建的实际工作是由一个个的 Task 来完成的。
## Gradle 生命周期
Task 的运行与 Gradle 的生命周期息息相关，所以在此之前先介绍一下 Gradle 构建的生命周期，无论什么时候执行 Gradle 构建，都会依次运行三个不同的生命周期阶段：
- **初始化**
在初始化阶段，Gradle 会解析 setting.gradle 文件获取该项目包含几个子项目，然后创建一个 RootProject 以及分别为子项目创建 Project 实例。
- **配置**
初始化完成后进入配置阶段，此时会加载所有 build.gradle 文件配置及插件，然后执行所有 Task 的配置代码块。
- **执行**
执行指的就是依据顺序执行所有 Task 的动作。

需要注意，无论我们是单独运行某一个 Task，还是运行所有的 Task，Gradle 的生命周期都是固定为上述的三个步骤，只不过执行的时候会有选择的执行指定 Task 及其依赖的 Task，这意味着如果一个 Task 设置了在配置阶段执行某项任务，即使我们运行了别的 Task，该任务也会被执行。
## Task 的特征
Task 具备如下几个特征。

**1. 可以依赖于别的 Task**
有时候我们期望该 Task 在某个 Task 执行之后再执行，此时可以使用 dependsOn 方法来设置依赖关系，让 Task 按照顺序执行。

**2. 不同时机的 Task 动作：onConfig（配置块）、doFirst、doLast、action**
Task 具备自己的执行周期，我们可以选择在不同的时机做不同的事情。
onConfig（配置块） 表示项目的配置阶段，也就是上述生命周期中的配置阶段，Task 中的配置代码块将会被执行。
doFirst 以及 doLast 是指在 Task 的动作（action）执行前或执行后定义的动作。
action 指执行阶段被运行的代码块。

**3. 输入/输出**
Gradle 通过比较 两个构建 task 的 inputs 和 outputs 来决定是否是最新的。如果 inputs 和 outputs 没有发生变化，则认为 task 是最新的，因此，只有当 inputs 和 outputs 不同时，task 才会运行，否则将会跳过。

**4. 一些 getter/setter 属性**
除此之外，Task 中提供了很多属性，我们可以通过 get/set 方法来获取或者设置。

## Task 的创建与使用
下面我们来定义一个 Task：

![](/assets/img/post/gradle/1.webp)

上述代码比较简单，配置阶段会创建一个变量，然后在 doFirst 中赋值，doLast 中打印出来，运行结果如下：
```
> Configure project :
printVersion on config
> Task :printVersion
prepare version
version:1
```

doLast 方法还有个等价的方法：**leftShift 可以简写为 <<**
```
printVersion << {
    println("on <<")
}
```

Task 的动作除了上面代码定义的三种外还有一个 action 的动作，可以使用注解 @TaskAction 来实现：
```
class PrintVersion extends DefaultTask{
    @TaskAction
    void start(){
        println("on action")
    }
}
```

Gradle 提供了很多种创建 Task 的方式：
```
task printVersion{}
project.task("printVersion"){}
project.tasks.create("printVersion"){}
```

我们可以通过 api 灵活的根据场景创建 Task。
下面再用一个例子总结一下上述内容：
```
project.task("task1"){
    println "task1 on config"
    doLast {
        println "task1 doLast"
    }
}

project.tasks.create("task2"){
    println "task2 on config"
    doLast {
        println "task2 doLast"
    }
}

task task3{
    println "task3 on config"
    doLast {
        println "task3 doLast"
    }
}

task3.dependsOn('task2')//设置 task3 依赖 task2
```
输入命令：gradlew task3，运行 task3，结果如下：
```
C:\xx\xx\xx>gradlew task3
> Configure project :
task1 on config
task2 on config
task3 on config

> Task :task2
task2 doLast

> Task :task3
task3 doLast
```

我们在运行 task3 时，Gradle 会先初始化然后进入配置阶段，此时不管我们运行的是哪一个 task，既然构建进入了配置阶段就意味着会**配置所有的 task**，所以即使我们只运行了 task3，另外两个 task 也都打印了 on config。

然后进入执行阶段，我们看到执行顺序是 task2 ——> task3，是因为代码中设置了 task3 依赖 task2，Gradle 在执行某个 task 时会先执行它依赖的 task 链表。

## Task 的 inputs 和 outputs
inputs 与 outputs 的也是 Task 的一个很重要的特性，Gradle 通过判断 Task的 **inputs/outputs 的一致性**来判断 Task 是否是最新的，如果 Task 是最新的，那么构建阶段将会跳过这个 Task 来节省构建时间。

inputs 和 outputs 可以是**一个目录、一个或多个文件，或者是任意一个属性**。

这个也很好理解，假设这个 Task 的 inputs 是文件 a.java，输出是 a.class，那么如果上次构建到这次构建期间，这两个文件都未发生变化，那就没必要再执行这个 Task 了。

# 参考文献
[1]实战Gradle.Benjamin Muschko.电子工业出版社,2015.9.

