---
title: Koin-2-Setup(初始化)
categories:
  - Koin
tags:
  - Koin
  - Kotlin
  - Android
cover: 'https://pbs.twimg.com/profile_banners/936214891731603456/1523947778/1500x500'
hide: true
abbrlink: b075de90
date: 2020-10-07 21:23:05
updated: 2020-10-07 21:23:05
---
> https://start.insert-koin.io/#/setup/index

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

# 为您的项目配置Koin

### [最新版本](https://start.insert-koin.io/#/setup/index?id=current-version)

Koin的最新版本：

```groovy
// 最新的stable版本
koin_version= "2.1.6"

// 最新的unstable版本
koin_version= "2.2.0-rc-1"
```

### [Gradle 依赖](https://start.insert-koin.io/#/setup/index?id=gradle-dependencies)

添加下面这些Gradle依赖来将Koin添加到您的项目中：

> Koin已经在Jcenter上发布

```groovy
// Add Jcenter to your repositories if needed
repositories {
    jcenter()
}
```

Kotlin

```groovy
// Koin for Kotlin
implementation "org.koin:koin-core:$koin_version"

// Koin Extended & experimental features
implementation "org.koin:koin-core-ext:$koin_version"

// Koin for Unit tests
testImplementation "org.koin:koin-test:$koin_version"
```

Gradle Plugin

```groovy
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

Android

```groovy
// Koin for Android
implementation "org.koin:koin-android:$koin_version"

// Koin Android Scope feature
implementation "org.koin:koin-android-scope:$koin_version"

// Koin Android ViewModel feature
implementation "org.koin:koin-android-viewmodel:$koin_version"
```

AndroidX

```groovy
// Koin AndroidX Scope feature
implementation "org.koin:koin-androidx-scope:$koin_version"

// Koin AndroidX ViewModel feature
implementation "org.koin:koin-androidx-viewmodel:$koin_version"

// Koin AndroidX Fragment Factory
implementation "org.koin:koin-androidx-fragment:$koin_version"

// Koin AndroidX Work Manager (unstable version)
implementation "org.koin:koin-androidx-workmanager:$koin_version"

// Koin AndroidX Compose (unstable version)
implementation "org.koin:koin-androidx-compose:$koin_version"
```

Ktor

```groovy
// Koin for Ktor Kotlin
implementation "org.koin:koin-ktor:$koin_version"
```