---
title: Koin-7-Starting_Koin
categories:
  - Koin
tags:
  - Koin
  - Kotlin
  - Android
cover: 'https://pbs.twimg.com/profile_banners/936214891731603456/1523947778/1500x500'
hide: true
abbrlink: c99907a8
date: 2020-10-07 21:24:01
updated: 2020-10-07 21:24:01
---
> https://start.insert-koin.io/#/getting-started/starting-koin

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



> Koin is a DSL, a container & a pragamtic API to leverage your dependencies.
>
> Koin是一个DSL、一个容器、一个实用的API来有效利用我们的依赖

Koin DSL包含这些：

- KoinApplication DSL: 描述如何配置你的Koin Application
- Module DSL: 描述你的定义

开始使用Koin可以通过如下几种使用 `startKoin` 的形式：

## [StartKoin](https://start.insert-koin.io/#/getting-started/starting-koin?id=startkoin)

在一个Kotlin文件中：

```kotlin
fun main(vararg args: String) {

    startKoin {
        // enable Printlogger with default Level.INFO 使用默认Level.INFO来开启Printlogger
        // can have Level & implementation 有Level和implementation
        // equivalent to logger(Level.INFO, PrintLogger()) 相当于Logger
        printlogger() 

        // declare properties from given map 从给定的map中声明属性
        properties( /* properties map */)

        // load properties from koin.properties file or given file name 
        // 从koin.properties文件或者给定的文件名中加载属性
        fileProperties()

        // load properties from environment 从环境中加载属性
        environmentProperties()

        // list all used modules 列出所有的module
        // as list or vararg 通过list或者可变长度的参数
        modules(myModules) 
    }
}
```

## [Starting for Android](https://start.insert-koin.io/#/getting-started/starting-koin?id=starting-for-android)

在任何一个Android类中：

```kotlin
class MainApplication : Application() {

    override fun onCreate() {
        super.onCreate()

        startKoin {
            // use AndroidLogger as Koin Logger - default Level.INFO
            // 使用AndroidLogger作为Koin Logger - 默认Level.INFO
            androidLogger()

            // use the Android context given there 使用这给出的Android context
            androidContext(this@MainApplication)

            // load properties from assets/koin.properties file 从assets/koin.properties文件中加载属性
            androidFileProperties()

            // module list
            modules(myModules)
        }
    }
}
```

如果你不能注入Android context或者application，那就确保在你的Koin application声明中使用 `androidContext()` 函数。

## [Starting for Ktor](https://start.insert-koin.io/#/getting-started/starting-koin?id=starting-for-ktor)

Starting Koin from your `Application` extension function:

```kotlin
fun Application.main() {
    // Install Ktor features
    install(Koin) {
        // Use SLF4J Koin Logger at Level.INFO
        slf4jLogger()

        // declare used modules
        modules(myModules)
    }
}
```

## [自定义Koin实例](https://start.insert-koin.io/#/getting-started/starting-koin?id=custom-koin-instance)

Here below are the KoinApplication builders:

下面这些是KoinApplicatioon builders(构造者)：

- `startKoin { }` - 创建并注册如下的KoinApplication实例
- `koinApplication { }` - 创建KoinApplication实例

```kotlin
// Create and register following KoinApplication instance
startKoin {
    logger()
    modules(coffeeAppModule)
}

// create KoinApplication instance
koinApplication {
    logger()
    modules(coffeeAppModule)
}
```

## [Logging](https://start.insert-koin.io/#/getting-started/starting-koin?id=logging)

开始时，Koin log就需要被定义他的名字或类型(如果log是活跃状态)：(*At start, Koin log what definition is bound by name or type (if log is activated):*)

```
[INFO] [Koin] bind type:'org.koin.example.CoffeeMaker' ~ [type:Single,class:'org.koin.example.CoffeeMaker']
[INFO] [Koin] bind type:'org.koin.example.Pump' ~ [type:Single,class:'org.koin.example.Pump']
[INFO] [Koin] bind type:'org.koin.example.Heater' ~ [type:Single,class:'org.koin.example.Heater']
```

## [DSL](https://start.insert-koin.io/#/getting-started/starting-koin?id=dsl)

快速回顾一下Koin DSL关键字：

- `startKoin { }` - 创建和注册如下的KoinApplication实例
- `koinApplication { }` - 创建KoinApplication实例
- `modules(...)` - 声明使用的modules
- `logger()` - 声明Printlogger
- `properties(...)` - 声明map属性
- `fileProperties()` - 从文件中使用属性
- `environmentProperties()` - 从环境中使用属性
- `androidLogger()` - 声明 Android Koin logger
- `androidContext(...)` - 使用给出的Android context
- `androidFileProperties()` - 使用 Android assets 中的属性文件
- `slf4jLogger(...)` - 使用SLF4J Logger