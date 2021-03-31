---
title: Jetpack入门 之 LiveData的map()和switchMap
categories:
  - Android
tags:
  - Android
  - Jetpack
  - MVVM
  - LiveData
cover: 'https://cdn.littlecorgi.top/Android%20Jetpack%E5%A4%B4%E5%9B%BE.png'
abbrlink: b1488352
date: 2020-06-24 23:06:16
updated: 2020-08-14 16:40:20
---
# LiveData的map()和switchMap

# 官方文档中的介绍
在Android Developer官网上，对于map和switchMap的解释是这样的：

- map：

> Applies a function on the value stored in the LiveData object, and propagates the result downstream.

对存储在 LiveData 对象中的值应用函数，并将结果传播到下游。

- switchMap：

> applies a function to the value stored in the LiveData object and unwraps and dispatches the result downstream. **The function passed to switchMap() must return a LiveData object, as illustrated by the following example:**

对存储在 LiveData 对象中的值应用函数，并将结果解封和分派到下游。**传递给 switchMap() 的函数必须返回 LiveData 对象。**

<!--More-->

其实单纯的看上面两个官方文档反而还会觉得更加的莫名其妙。其实这两句话差异的核心在于 **“解封”** 和 **“返回LiveData对象”** 。

# 例子
我们先来举个栗子来说明下这两个方法的区别(例子来源于[此文章](https://medium.com/@elye.project/understanding-live-data-made-simple-a820fcd7b4d0))：

```kotlin
class TransformationMapFragment : Fragment() {

    private val changeObserver = Observer<String> { value ->
        value?.let { txt_fragment.text = it }
    }

    override fun onAttach(context: Context?) {
        super.onAttach(context)
        val transformedLiveData = Transformations.map(
                getLiveDataA()) { "A:$it" }
        transformedLiveData.observe(this, changeObserver)
    }
}
```

我们可以看到运行结果是这样的：

![1_6cTWqtHlJgnzF701QFtZQg](https://cdn.littlecorgi.top/mweb/2020-06-24/1_6cTWqtHlJgnzF701QFtZQg.gif)


结合代码运行结果以及上面文档中对map函数的定义，我们可以得出下面这个结论：map函数主要的功能就是根据原LiveData，对其原LiveData的值进行改变然后生成一个新的LiveData。**基于原LiveData**。

而switchMap，同样的，我们也来看一个例子：

```kotlin
class TransformationSwitchMapFragment : Fragment() {

    private val changeObserver = Observer<String> { value ->
        value?.let { txt_fragment.text = it }
    }

    override fun onAttach(context: Context?) {
        super.onAttach(context)

        val transformSwitchedLiveData =
            Transformations.switchMap(getLiveDataSwitch()) { 
                switchToB ->
                if (switchToB) {
                     getLiveDataB()
                } else {
                     getLiveDataA()
                }
        }

        transformSwitchedLiveData.observe(this, changeObserver)
    }
    // .. some other Fragment specific code ..
}
```

我们可以看到运行结果是这样的：

![1_9RDrsXCBQhZmmc8XMQWw6Q](https://cdn.littlecorgi.top/mweb/2020-06-24/1_9RDrsXCBQhZmmc8XMQWw6Q.gif)


同样的结合这个例子和上面的文档描述，我们也可以得出一个结论：switchMap是根据传入的LiveData的值，然后判断这个值，然后再去切换或者构建新的LiveData。**手动生成新的LiveData**。

# 差异
所以对于这两个函数的区别来说，map，更关注于数值的转换，也就是说，他只会通过你之前的值去生成一个新的值，强调的是这个**转换**，也就是说新的LiveData对象的值，仍然是基于当前这个LiveData的值而生成的。

而switchMap，更关注于数值的**触发**，也就是说，他会监听你这个值的变化，而不关注你这个值本身，你可以理解成触发器或者扳机，当触发之后，你就需要自己去主动的返回任意一个你指定的liveData对象。

所以我们再回过头来看一下上面说的 **“解封”** 和 **“返回LiveData对象” ：**

对于这个，我觉得应该相对的来看，对于map来说，他把值从LiveData中读取了出来，但是他只是在原值的基础上进行操作生成新的LiveData对象，新的LiveData对象的值是基于旧的LiveData对象的，也就是说没有完全的将值独立出来；而对于switchMap，它不仅仅将值从LiveData中取出来了，并且取出来之后，对于新的LiveData来说，他并没有要求新的LiveData的值必须得基于旧的LiveData咋样，而是只是单纯的作为一个触发器，你返回的新的LiveData对象中存储的值，与旧的LiveData对象的值有没有关系，他并不在意。

# 总结
所以，总结一下。

map强调的是，**新的LiveData的值必须基于旧的LiveData中的值**(比方说像上面的例子，取出旧的LiveData的每个值然后加上相同的String从而生成新的LiveData)。

而switchMap，他并不在意这些，他在意的是他会**将旧的LiveData的值作为一个触发，作为一个switch**，他不管你你到底是利用这个switch做判断返回不用的值(像上面的例子)，还是你利用这个值去网络请求生成新的值(比方说旧的LiveData存了用户名，而你根据这个用户名去请求用户的具体信息作为新的LiveData返回)也好，只要你**手动返回了一个新的LiveData**就行。

# 相关文档
- [Understanding LiveData made simple - Elye - Medium](https://medium.com/@elye.project/understanding-live-data-made-simple-a820fcd7b4d0)
- [What is the difference between map() and switchMap() methods? - stack overflow](https://stackoverflow.com/a/47690327/13753159)
- [LiveData 概览 # 转换 LiveData - Android Developer](https://developer.android.google.cn/topic/libraries/architecture/livedata#transform_livedata)