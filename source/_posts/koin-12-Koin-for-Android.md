---
title: Koin-12-Koin for Android
categories:
  - Koin
tags:
  - Koin
  - Kotlin
  - Android
cover: 'https://pbs.twimg.com/profile_banners/936214891731603456/1523947778/1500x500'
hide: true
abbrlink: f5a785d7
date: 2020-10-07 21:25:34
updated: 2020-10-07 21:25:34
---
> https://start.insert-koin.io/#/getting-started/koin-for-android

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

## [在Android中开始](https://start.insert-koin.io/#/getting-started/koin-for-android?id=starting-koin-for-android)

在任何一个Android类中：

```kotlin
class MainApplication : Application() {

    override fun onCreate() {
        super.onCreate()

        startKoin {
            // use AndroidLogger as Koin Logger - default Level.INFO
            // 使用AndroidLogger代替KoinLogger - 默认Lovel.INFO
            androidLogger()

            // use the Android context given there 在这使用给定的AndroidContext
            androidContext(this@MainApplication)

            // load properties from assets/koin.properties file
            // 从assets/koin.properties文件中加载属性
            androidFileProperties()

            // module list 模块列表
            modules(offlineWeatherApp)
        }
    }
}
```

如果你不能注入Android上下文或者application，请确保使用 `androidContext()` 函数在你的Koin应用程序声明。

## [使用Android Context](https://start.insert-koin.io/#/getting-started/koin-for-android?id=use-the-android-context)

在你的定义中，你能通过 `androidContext()` 和 `androidApplication()` 函数注入 `Context` 和 `Application` ：

```kotlin
module {
    single { MyAndroidComponent(androidContext()) }
}
```

## [Android组件作为Koin组件](https://start.insert-koin.io/#/getting-started/koin-for-android?id=android-components-as-koincomponents)

Koin对 `Activity`, `Fragment` & `Service` 进行了扩展，以将其视为现成的KoinComponents：

```kotlin
class MyActivity : AppCompatActivity(){

    // Inject MyPresenter
    val presenter : MyPresenter by inject()

    override fun onCreate() {
        super.onCreate()

        // or directly retrieve instance
        val presenter : MyPresenter = get()
    }
}
```

这些类能被使用：

- `get()` or `by inject()` instance retrieving 检索实例
- `getKoin()` to access th current `Koin` instance 访问当前 `Koin` 实例

如果您需要注入来自另一个类的依赖项，并且无法在模块中声明它，则仍然可以使用 `KoinComponent` 接口对其进行标记。

## [Extended Scope API](https://start.insert-koin.io/#/getting-started/koin-for-android?id=extended-scope-api)

> for Android (koin-android-scope or koin-androidx-scope projects)

Scope API更接近Android平台。`Activity` 和 `Fragment` 都具有Scope API的扩展：`currentScope` 获取当前关联的Koin scopr。 此scope已创建并绑定到组件的生命周期。

你能直接使用关联的Koin scope来检索组件：

```kotlin
class DetailActivity : AppCompatActivity(), DetailContract.View {

    // inject from current activity scope instance
    override val presenter: DetailContract.Presenter by currentScope.inject()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        //...
    }
}
```

很容易来声明你的Android组件scope:

```kotlin
module {
    // declare a scope for DetailActivity
    scope(named<DetailActivity>)> {
        scoped<DetailContract.Presenter> { DetailPresenter(get(), get()) }
    }
}
```

任何Activity和Fragment都能直接使用scopeAPI： `createScope()`, `getScope()` and `deleteScope()`。

## [Android ViewModel](https://start.insert-koin.io/#/getting-started/koin-for-android?id=android-viewmodel)

> (koin-android-viewmodel or koin-androidx-viewmodel projects)

Koin也带来了一些特殊的特性来管理ViewModel：

- `viewModel` 特殊的DSL关键来来声明一个ViewModel
- `by viewModel()` & `getViewModel()` 注入ViewModel实例(from `Activity` & `Fragment`)
- `by sharedViewModel()` & `getSharedViewModel()` 从宿主Activity中复用ViewModel实例(来自Fragment)

让我们在模块中声明一个ViewModel：

```kotlin
val myModule : Module = module {

    // ViewModel instance of MyViewModel
    // get() will resolve Repository instance
    viewModel { MyViewModel(get()) }

    // Single instance of Repository
    single<Repository> { MyRepository() }
}
```

在一个Activity中注入它：

```kotlin
class MyActivity : AppCompatActivity(){

    // Lazy inject MyViewModel
    val model : MyViewModel by viewModel()

    override fun onCreate() {
        super.onCreate()

        // or also direct retrieve instance
        val model : MyViewModel = getViewModel()
    }
}
```