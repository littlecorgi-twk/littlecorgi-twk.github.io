---
title: Koin-9-Koin Components(组件)
categories:
  - Koin
tags:
  - Koin
  - Kotlin
  - Android
cover: 'https://pbs.twimg.com/profile_banners/936214891731603456/1523947778/1500x500'
hide: true
abbrlink: f85e8eae
date: 2020-10-07 21:25:00
updated: 2020-10-07 21:25:00
---
> https://start.insert-koin.io/#/getting-started/koin-components

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

有时不能仅通过Koin声明组件。依赖于你的运行时技术，你可能需要在一个不是用Koin创建的类中从Koin检索实例（例如Android）。

## [Koin组件实例](https://start.insert-koin.io/#/getting-started/koin-components?id=the-koincomponent-interface)

通过 `KoinComponent` 接口来标记你的类就能解锁Koin的注入功能：

- `by inject()` - 延迟注入一个实例
- `get()` - 检索一个实例
- `getProperty()` - 获得一个Koin属性

我们可以将上面的模块注入到类属性中：

```kotlin
// Tag class with KoinComponent
class HelloApp : KoinComponent {

    // lazy inject dependency
    val helloService: HelloServiceImpl by inject()

    fun sayHello(){
        helloService.sayHello()
    }
}
```

然后我们就只需要开启Koin并运行我们的类：

```kotlin
// a module with our declared Koin dependencies 
val helloModule = module {
    single { HelloServiceImpl() }
}

fun main(vararg args: String) {

    // Start Koin
    startKoin {
        modules(helloModule)
    }

    // Run our Koin component
    HelloApp().sayHello()
}
```

#### [引导](https://start.insert-koin.io/#/getting-started/koin-components?id=bootstrapping)

> `KoinComponent` 接口也能被用来协助你从Koin外部引导一个应用程序。另外，您可以通过扩展函数直接在一些目标类上引入“KoinComponent”特性（即：Android中的Activity、Fragment have KoinComponent特性）。

## [Bridge with Koin instance](https://start.insert-koin.io/#/getting-started/koin-components?id=bridge-with-koin-instance)

`KoinComponent` 接口主要带来了如下内容：


```kotlin
interface KoinComponent {

    /**
     * Get the associated Koin instance
     */
    fun getKoin(): Koin = GlobalContext.get().koin
}
```

他带来了如下的可能性：

> 然后可以重新定义 `getKoin()` 函数，以重定向到本地自定义Koin实例