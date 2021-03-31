---
title: Koin-10-Testing(测试)
categories:
  - Koin
tags:
  - Koin
  - Kotlin
  - Android
cover: 'https://pbs.twimg.com/profile_banners/936214891731603456/1523947778/1500x500'
hide: true
abbrlink: 63fa5724
date: 2020-10-07 21:25:15
updated: 2020-10-07 21:25:15
---
> https://start.insert-koin.io/#/getting-started/testing

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

`Koin-test` 项目能提供给你轻量但是强大的工具来测试你的Koin应用程序。

## [获取组件](https://start.insert-koin.io/#/getting-started/testing?id=getting-your-components)

只要用 `KoinTest` 标记你的测试类，你就可以解锁KoinComponent &测试特性：

- `by inject()` - 延迟注入一个实例
- `get()` - 检索一个实例

按照以下来定义：

```kotlin
val appModule = module {
    single { ComponentA() }
    //...
}
```

你就能像下面这样来写一个测试：

```kotlin
class MyTest : KoinTest {

    // Lazy inject property
    val componentA : ComponentA by inject()

    // use it in your tests :)
    @Test
    fun `make a test with Koin`() {
        startKoin { modules(appModule) }

        // use componentA here!
    }
}
```

你能使用 `KoinTestRule` JUnit规则来开启或关闭你的Koin上下文：

```kotlin
class MyTest : KoinTest {

    @get:Rule
    val koinTestRule = KoinTestRule.create {
        modules(appModule)
    }

    // Lazy inject property
    val componentA : ComponentA by inject()

    // use it in your tests :)
    @Test
    fun `make a test with Koin`() {
        // use componentA here!
    }
}
```

## [检查你的模块](https://start.insert-koin.io/#/getting-started/testing?id=checking-your-modules)

我们可以使用Koin Gradle插件来让我们运行我们的模块检查：

```gradle
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath "org.koin:koin-gradle-plugin:$koin_version"
    }
}

apply plugin: 'koin'
```

按照如下来写检查测试：

- 使用一个JUnit `CheckModuleTest` 类别
- 通过 `checkModules { }` API来测试模块

```kotlin
@Category(CheckModuleTest::class)
class ModuleCheckTest : AutoCloseKoinTest() {

    @Test
    fun checkModules() = checkModules {
        modules(appModule)
    }
}
```

让我们通过Gradle命令来检查我们的模块：

```
./gradlew checkModules
```

或者

```
./gradlew checkModules --continuous
```

## [动态Mock](https://start.insert-koin.io/#/getting-started/testing?id=mocking-on-the-fly)

Once you have tagged your class with `KoinTest` interface, you can use the `declareMock` function to declare mocks & behavior on the fly:

一旦你用 `KoinTest` 接口标记了你的类，你就能使用 `declareMock` 函数来动态声明mocks(模拟)或者behavior(行为):

```kotlin
class MyTest : KoinTest {

    @get:Rule
    val koinTestRule = KoinTestRule.create {
        modules(appModule)
    }

    // required to make your Mock via Koin
    @get:Rule
    val mockProvider = MockProviderRule.create { clazz ->
        Mockito.mock(clazz.java)
    }

    val componentA : ComponentA by inject()

    @Test
    fun `declareMock with KoinTest`() {
        declareMock<ComponentA> {
            // do your given behavior here
            given(this.sayHello()).willReturn("Hello mock")
        }
    }
}
```

## [开始或停止测试](https://start.insert-koin.io/#/getting-started/testing?id=starting-amp-stopping-for-tests)

请注意在每个测试之间都需要通知你的Koin实例(如果你使用 `startKoin` 在你的测试中)。否则，请确保对本地koin实例使用 `koinApplication` 或 `stopKoin()` 来停止当前的全局实例。