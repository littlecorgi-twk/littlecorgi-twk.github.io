---
title: Koin-4-Android
categories:
  - Koin
tags:
  - Koin
  - Kotlin
  - Android
cover: 'https://pbs.twimg.com/profile_banners/936214891731603456/1523947778/1500x500'
hide: true
abbrlink: 2a806fe1
date: 2020-10-07 21:23:27
updated: 2020-10-07 21:23:27
---
> https://start.insert-koin.io/#/quickstart/android

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

# [在Android application中使用](https://start.insert-koin.io/#/quickstart/android?id=getting-started-with-android-application)

> 这个教程将会告诉你如何使用Koin注入和检索组件来编写一个Android/Kotlin application。

## [Get the code](https://start.insert-koin.io/#/quickstart/android?id=get-the-code)

可以直接在Github上查看项目或者下载zip

> 🚀 Go to [Github](https://github.com/InsertKoinIO/getting-started-koin-android) or [download Zip](https://github.com/InsertKoinIO/getting-started-koin-android/archive/master.zip)

## [配置Gradle](https://start.insert-koin.io/#/quickstart/android?id=gradle-setup)

通过如下方式添加 Koin Android 的依赖：

```groovy
// Add Jcenter to your repositories if needed
repositories {
    jcenter()    
}
dependencies {
    // Koin for Android
    compile "org.koin:koin-android:$koin_version"
}
```

## [组件](https://start.insert-koin.io/#/quickstart/android?id=our-components)

让我们来创建一个 `HelloRepository` 来提供一些数据：

```kotlin
interface HelloRepository {
    fun giveHello(): String
}

class HelloRepositoryImpl() : HelloRepository {
    override fun giveHello() = "Hello Koin"
}
```

在创建一个presenter类来消费这个数据：

```kotlin
class MySimplePresenter(val repo: HelloRepository) {
    fun sayHello() = "${repo.giveHello()} from $this"
}
```

## [编写Koin Module](https://start.insert-koin.io/#/quickstart/android?id=writing-the-koin-module)

使用 `module` 函数来声明一个module，让我们来声明我们的第一个组件：

```kotlin
val appModule = module {

    // single instance of HelloRepository
    single<HelloRepository> { HelloRepositoryImpl() }

    // Simple Presenter Factory
    factory { MySimplePresenter(get()) }
}
```

我们将我们的 `MySimplePresenter` 类声明为一个 `factory` 来让每当我们的Activity需要一个时就创建一个新的对象。

## [开始使用Koin](https://start.insert-koin.io/#/quickstart/android?id=start-koin)

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

## [注入依赖](https://start.insert-koin.io/#/quickstart/android?id=injecting-dependencies)

`MySimplePresenter` 组件将会被通过 `HelloRepository` 实例所创建。为了在我们的Activity中获取到它，就需要使用 `by inject()` 委托注入器来注入它：

```kotlin
class MySimpleActivity : AppCompatActivity() {

    // Lazy injected MySimplePresenter
    val firstPresenter: MySimplePresenter by inject()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        //...
    }
}
```

> `by inject()` 函数允许我们在Android组件(Activity, fragment, Service…)运行时再来检索Koin实例。

> `get()` 函数在这可以非延迟的直接检索一个实例。