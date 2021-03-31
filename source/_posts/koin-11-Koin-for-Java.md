---
title: Koin-11-Koin for Java
categories:
  - Koin
tags:
  - Koin
  - Kotlin
  - Android
cover: 'https://pbs.twimg.com/profile_banners/936214891731603456/1523947778/1500x500'
hide: true
abbrlink: ebed2a69
date: 2020-10-07 21:25:27
updated: 2020-10-07 21:25:27
---
> https://start.insert-koin.io/#/getting-started/koin-for-java

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

下面是 `koin-java` 特性的简短描述。这个项目是帮助java开发人员的静态utils的一个小子集。

## [开始](https://start.insert-koin.io/#/getting-started/koin-for-java?id=start-koin)

Just use the `startKoin()` static function (with the static import to reduce the syntax):

只需要使用 `startKoin()` 静态函数(通过静态导入来减少语法)：

```java
import static org.koin.core.context.GlobalContext.start;

// Build KoinApplication instance
// Builder API style
KoinApplication koinApp = KoinApplication.create()
                .printLogger()
                .modules(koinModule);

// Statr KoinApplication instance
start(koinApp);
```

您可以访问与Kotlin `startKoin()` [相同的选项](({{ site.baseurl }}/docs/{{ site.docs_version }}/quick-references/koin-core/#start-koin))。

## [Kotlin中的Java友好模块](https://start.insert-koin.io/#/getting-started/koin-for-java?id=java-friendly-module-in-kotlin)

你需要使用 `@JvmField` 标记你的module变量，以便其可以从Java中读取：

```kotlin
@JvmField
val koinModule = module {
    single { ComponentA() }
    single { ComponentB(get()) }
    single { ComponentC(get(), get()) }

    module("anotherModule") {
        single { ComponentD(get()) }
    }
}
```

## [在静态函数中注入](https://start.insert-koin.io/#/getting-started/koin-for-java?id=inject-with-static-functions)

`KoinJavaComponent` 类是第一个静态的helper，他能将Koin的功能引入Java：

- `inject()` - 延迟注入一个实例
- `get()` - 检索实例
- `getKoin()` - 获取Koin上下文

```java
import static org.koin.java.standalone.KoinJavaComponent.*;

ComponentA a = get(ComponentA.class);
Lazy<ComponentA> lazy_a = inject(ComponentA.class);
```