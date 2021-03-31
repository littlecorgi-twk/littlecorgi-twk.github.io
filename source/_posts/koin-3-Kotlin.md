---
title: Koin-3-Kotlin
categories:
  - Koin
tags:
  - Koin
  - Kotlin
  - Android
cover: 'https://pbs.twimg.com/profile_banners/936214891731603456/1523947778/1500x500'
hide: true
abbrlink: f88fedb6
date: 2020-10-07 21:23:21
updated: 2020-10-07 21:23:21
---
> https://start.insert-koin.io/#/quickstart/kotlin?id=getting-started-with-kotlin-app

> 开始
> - [Intro(介绍)](/posts/fd6f0996.html)
> - [Setup(初始化)](/posts/b075de90.html)
> 
> Koin in 5 minutes (5分钟快速入手Koin)
> - [Kotlin](/posts/f88fedb6.html)
> - [Android](/posts/2a806fe1.html)
> - [Android Java](/posts/e7ad0613.html)
> - [Android ViewModel](/posts/6d1e0fe7.html)
> 
> Getting Started (开始)
> - [Starting Koin(开始使用Koin)](/posts/c99907a8.html)
> - [Modules & definitions(模块和定义)](/posts/d8ca0532.html)
> - [Koin Components(Koin组件)](/posts/f85e8eae.html)
> - [Testing(测试)](/posts/63fa5724.html)
> - [Koin for Java](/posts/ebed2a69.html)
> - [Koin for Android](/posts/f5a785d7.html)

# [在Kotlin app中使用](https://start.insert-koin.io/#/quickstart/kotlin?id=getting-started-with-kotlin-app)

> 本教程将会告诉你如何使用Koin注入和检索组件来编写一个Kotlin app。

## [获取实例代码](https://start.insert-koin.io/#/quickstart/kotlin?id=get-the-code)

可以直接在Github上查看项目或者下载zip

> 🚀 Go to [Github](https://github.com/InsertKoinIO/getting-started-koin-core) or [download Zip](https://github.com/InsertKoinIO/getting-started-koin-core/archive/master.zip)

## [配置](https://start.insert-koin.io/#/quickstart/kotlin?id=setup)

首先，检查下 `koin-core` 依赖是不是按照如下形式添加了：

```groovy
// Add Jcenter to your repositories if needed
repositories {
    jcenter()    
}
dependencies {
    // Koin for Kotlin apps
    compile "org.koin:koin-core:$koin_version"
    // Testing
    testCompile "org.koin:koin-test:$koin_version"
}
```

## [Application](https://start.insert-koin.io/#/quickstart/kotlin?id=the-application)

在我们的小型App上，我们只需要有2个组件：

- HelloMessageData - 持有数据(data)
- HelloService - 使用和显示HelloMessageData上的数据
- HelloApplication - 检索和使用HelloService

### [Data holder](https://start.insert-koin.io/#/quickstart/kotlin?id=data-holder)

让我们创建一个 `HelloMessageData` 数据类来持有我们的数据：

```kotlin
/**
 * A class to hold our message data
 */
data class HelloMessageData(val message : String = "Hello Koin!")
```

### [Service](https://start.insert-koin.io/#/quickstart/kotlin?id=service)

让我们创建一个service来显示 `HelloMessageData` 中的数据。写一个 `HelloServiceImpl` 类以及他的接口 `HelloService` ：

```kotlin
/**
 * Hello Service - interface
 */
interface HelloService {
    fun hello(): String
}


/**
 * Hello Service Impl
 * Will use HelloMessageData data
 */
class HelloServiceImpl(private val helloMessageData: HelloMessageData) : HelloService {

    override fun hello() = "Hey, ${helloMessageData.message}"
}
```

## [The application class](https://start.insert-koin.io/#/quickstart/kotlin?id=the-application-class)

为了让HelloService组件能运行，我们还需要创建一个runtime组件。

让我们写一个 `HelloApplication` 类并让他实现 `KoinComponent` 接口。这能让我们稍后可以通过 `by inject()` 函数来检索我们的组件：

```kotlin
/**
 * HelloApplication - Application Class
 * use HelloService
 */
class HelloApplication : KoinComponent {

    // Inject HelloService
    val helloService by inject<HelloService>()

    // display our data
    fun sayHello() = println(helloService.hello())
}
```

## [声明依赖](https://start.insert-koin.io/#/quickstart/kotlin?id=declaring-dependencies)

现在，让我们使用Koin module来将 `HelloMessageData` 和 `HelloService` 组装在一起：

```kotlin
val helloModule = module {

    single { HelloMessageData() }

    single { HelloServiceImpl(get()) as HelloService }
}
```

我们使用 `single` 来将每一个组件声明成单例对象。

- `single { HelloMessageData() }` : 声明一个单例的 `HelloMessageData` 对象
- `single { HelloServiceImpl(get()) as HelloService }` : 使用注入的 `HelloMessageData` 来构造`HelloServiceImpl` 对象，并声明成 `HelloService` 的单例对象。

## [这就完成啦！](https://start.insert-koin.io/#/quickstart/kotlin?id=thats-it)

只需要通过一个 `main` 函数来启动我们的app：

```kotlin
fun main(vararg args: String) {

    startKoin {
        // use Koin logger
        printLogger()
        // declare modules
        modules(helloModule)
    }

    HelloApplication().sayHello()
}
```

