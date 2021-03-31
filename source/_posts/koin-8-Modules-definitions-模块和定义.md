---
title: Koin-8-Modules & definitions(模块和定义)
categories:
  - Koin
tags:
  - Koin
  - Kotlin
  - Android
cover: 'https://pbs.twimg.com/profile_banners/936214891731603456/1523947778/1500x500'
hide: true
abbrlink: d8ca0532
date: 2020-10-07 21:24:36
updated: 2020-10-07 21:24:36
---
> https://start.insert-koin.io/#/getting-started/modules-definitions

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

用Koin编写定义是通过Kotlin函数完成的，该函数描述了实例的构建方式。 

配置完Koin应用程序后，让我们来编写一些模块和定义。

## [模块 & 定义](https://start.insert-koin.io/#/getting-started/modules-definitions?id=module-amp-definitions)

给出我们需要去注入的类：

```kotlin
class DataRepository()
interface Presenter
class MyPresenter(val repository : Repository) : Presenter
class HttpClient(val url : String)
```

下面展示如何定义这些组件：

```kotlin
// declare a module
// 定义一个module
val myModule = module {

    // Define a singleton for type DataRepository
    // 为DataRepository定义一个单例
    single { DataRepository() }

    // Define a factory (create a new instance each time) for type Presenter (infered parameter in <>)
    // Resolve constructor dependency with get()
    // 为Presenter(在<>中推断类型)定义一个factory(每次都创建一个新的实例)
    // 使用 get() 解析构造函数依赖
    factory<Presenter> { MyPresenter(get()) }

    // Define a singleton of type HttpClient
    // inject property "server_url" from Koin properties
    // 定义一个HttpClient类的单利
    // 从Koin属性中注入"server_url"这个属性
    single { HttpClient(getProperty("server_url")) }
}
```

## [修饰符](https://start.insert-koin.io/#/getting-started/modules-definitions?id=qualifiers)

你可以给组件提供一个修饰符。这个修饰符可以是string或者type，并通过 `named()` 函数进行配置：

```kotlin
val myModule = module {

    // Define a singleton for type  DataRepository
    // 为DataRepository定义一个单例
    single { DataRepository() }

    // Mock
    single(named("mock")) { MockDataRepository() }
}
```

> 修饰符可以与枚举类关联，也可以直接与类型关联：

```kotlin
// Type qualifier
single(named<MyClass>()) { MockDataRepository() }

// Enum qualifier
single(named<MyEnum.MyValue>()) { MockDataRepository() }
```

## [其它类型](https://start.insert-koin.io/#/getting-started/modules-definitions?id=additional-types)

在DLS模块中，对于一个定义，你可以使用 `bind` 操作符(KClass列表使用 `binds`)来给定一些额外的类型去绑定：

```kotlin
module {
    single { Component1() } bind ComponentInterface1::class
}
```

然后你就能通过 `get<Component1>()` 或者 `get<ComponentInterface1>()` 来请求到你的实例了。

你也能给多个定义绑定同一个type：

```kotlin
module {
    single { Component1() } bind ComponentInterface1::class
    single { Component2() } bind ComponentInterface1::class
}
```

但是在这你不能通过 `get<Simple.ComponentInterface1>()` 来请求一个实例。你只能使用 `koin.bind<Component1,ComponentInterface1>()` 来检索一个具有 `Component1` 实现的 `ComponentInterface1` 的实例。

请注意，您还可以查找绑定给定类型的所有组件： `getAll<ComponentInterface1>()` 将请求所有绑定 `ComponentInterface1` 类的实例。

## [组合模块](https://start.insert-koin.io/#/getting-started/modules-definitions?id=combining-several-modules)

Koin没有导入(import)的概念。所以只需结合几个互补的模块定义。

让我们在两个模块中分发定义：

```kotlin
val module1 = module {
    single { DataRepository() }
}
val module2 = module {
    factory<Presenter> { MyPresenter(get()) }
}
```

我们只需要为Koin列出他们：

```kotlin
startKoin {
    modules(module1,module2)
}
```

## [启动后加载](https://start.insert-koin.io/#/getting-started/modules-definitions?id=loading-after-start)

在Koin通过 `startKoin { }` 函数启动后，我们还可以使用 `loadKoinModules(modules...)` 函数来加载其它定义模块。

## [删除定义](https://start.insert-koin.io/#/getting-started/modules-definitions?id=dropping-definitions)

一旦一个模块被加载到了Koin中，我们就可以取消加载他然后删除掉与这些定义相关的定义和实例。为此，我们可以使用 `unloadKoinModules(modules...)` ：

```kotlin
val module = module {
    single { (id: Int) -> Simple.MySingle(id) }
}
startKoin {
    printLogger(Level.DEBUG)
    modules(module)
}

get<Simple.MySingle> { parametersOf(42) } -> id is 42

// unload definitions for given module
unloadKoinModules(module)
// load definitions for given module
loadKoinModules(module)

get<Simple.MySingle> { parametersOf(24) } -> id is 24
```

## [即时声明](https://start.insert-koin.io/#/getting-started/modules-definitions?id=declare-on-the-fly)

Koin 1.0的最后一个向后移植*(译者注：原文是backport)*功能之一是能够动态声明实例。现在可以在Koin.declare（）或Scope.declare（）上使用：

```kotlin
val koin = koinApplication {
    // no def
    modules()
}.koin

// Create an instance
val a = Simple.ComponentA()

// declare it
koin.declare(a)

// retrieve it
assertEquals(a, koin.get<Simple.ComponentA>())
```

你也能通过一个修饰符或者次要类型来帮助你创建定义：

- `koin.declare(myInstance, named("qualifier"))`
- `koin.declare(myInstance, secondaryTypes = listOf())`

## [DSL回顾](https://start.insert-koin.io/#/getting-started/modules-definitions?id=dsl-recap)

快速回顾下这些Koin关键字：

- `module { }` - 创建一个Koin模块或者子模块(在一个模块里面)
- `factory { }` - 提供一个 *factory工厂* bean定义
- `single { }` - 提供一个bean定义
- `get()` - 解析一个组件依赖
- `named()` - 通过type、枚举或者字符串来定义一个修饰符
- `bind` - 给给定的bean定义绑定其它Kolin类型
- `binds` - 给给定的bean定义绑定其他Kotlin类型列表
- `getProperty()` - 解析一个Koin属性