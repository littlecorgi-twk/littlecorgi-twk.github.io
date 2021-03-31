---
title: Koin-6-Android_ViewModel
categories:
  - Koin
tags:
  - Koin
  - Kotlin
  - Android
cover: 'https://pbs.twimg.com/profile_banners/936214891731603456/1523947778/1500x500'
hide: true
abbrlink: 6d1e0fe7
date: 2020-10-07 21:23:46
updated: 2020-10-07 21:23:46
---
> https://start.insert-koin.io/#/quickstart/android-viewmodel

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

# [在Android ViewModel application中使用](https://start.insert-koin.io/#/quickstart/android-viewmodel?id=getting-started-with-android-viewmodel-application)

> 这个教程将会告诉你如何使用Koin注入和检索ViewModel组件来编写一个Android/Kotlin application。

## [Get the code](https://start.insert-koin.io/#/quickstart/android-viewmodel?id=get-the-code)

可以直接在Github上查看项目或者下载zip

> 🚀 Go to [Github](https://github.com/InsertKoinIO/getting-started-koin-android) or [download Zip](https://github.com/InsertKoinIO/getting-started-koin-android/archive/master.zip)

## [配置Gradle](https://start.insert-koin.io/#/quickstart/android-viewmodel?id=gradle-setup)

通过如下方式添加 Koin Android 的依赖：

```groovy
// Add Jcenter to your repositories if needed
repositories {
    jcenter()    
}
dependencies {
    // Koin for Android - Scope feature
    // include koin-android-scope & koin-android
    compile "org.koin:koin-android-viewmodel:$koin_version"
}
```

## [我们的组件](https://start.insert-koin.io/#/quickstart/android-viewmodel?id=our-components)

让我们创建一个 `HelloRepository` 来提供一些数据：

```kotlin
interface HelloRepository {
    fun giveHello(): String
}

class HelloRepositoryImpl() : HelloRepository {
    override fun giveHello() = "Hello Koin"
}
```

让我们创建一个ViewModel类，来消费这些数据：

```kotlin
class MyViewModel(val repo : HelloRepository) : ViewModel() {

    fun sayHello() = "${repo.giveHello()} from $this"
}
```

## [编写Koin module](https://start.insert-koin.io/#/quickstart/android-viewmodel?id=writing-the-koin-module)

使用 `module` 函数来声明一个module。让我们声明我们的第一个组件：

```kotlin
val appModule = module {

    // single instance of HelloRepository
    single<HelloRepository> { HelloRepositoryImpl() }

    // MyViewModel ViewModel
    viewModel { MyViewModel(get()) }
}
```

在一个 `module` 中，我们将我们的 `MyViewModel` 类声明成一个 `viewModel`。Koin将会将这个 `MyViewModel` 交给Lifecycle ViewModelFactory并帮我们将它绑定到当前的组件上。

## [开始使用Koin](https://start.insert-koin.io/#/quickstart/android-viewmodel?id=start-koin)

现在我们已经有了一个module，我们就可以使用Koin了。打开你的Application类，或者创建一个(不要忘了在你的Manifest.xml中声明它)。然后只需要调用 `startKoin()` 函数：

```kotlin
class MyApplication : Application(){
    override fun onCreate() {
        super.onCreate()
        // Start Koin
        startKoin{
            androidLogger()
            androidContext(this@MyApplication)
            modules(appModule)
        }
    }
}
```

## [注入依赖](https://start.insert-koin.io/#/quickstart/android-viewmodel?id=injecting-dependencies)

`MyViewModel` 组件将会被通过 `HelloRepository` 实例所创建。为了在我们的Activity中获取到它，就需要使用 `by viewModel()` 委托注入器来注入它：

```kotlin
class MyViewModelActivity : AppCompatActivity() {

    // Lazy Inject ViewModel
    val myViewModel: MyViewModel by viewModel()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_simple)

        //...
    }
}
```

> `by viewModel()` 函数允许我们从Koin中检索一个ViewModel实例，链接到Android ViewModelFactory。

> `getViewModel()` 函数能非延迟的直接检索一个实例。