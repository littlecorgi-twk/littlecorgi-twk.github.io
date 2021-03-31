---
title: Kotlin Anko入门
tags:
  - Kotlin
  - Android
categories: Kotlin
comments: true
abbrlink: 62bb1dd7
date: 2019-06-01 12:30:56
updated: 2019-06-01 12:30:56
copyright: true
---
# Kotlin Anko入门

# 简介
Anko的官网就是他的GitHub地址
> https://github.com/Kotlin/anko

官方对Anko的解释是
> Anko是一个 [Kotlin](https://www.kotlinlang.org/) 库，它使Android应用程序开发更快更容易。它使您的代码清晰易读，让您忘记Android SDK for Java的粗糙边缘。

<!-- More -->
为什么这样说呢？
比方说如果你写Android，你在xml中定义了一个`Button`，他的ID是`button_login`。
如果是Java：
```java
setContentView(R.layout.activity_words_detail);
Button button = findViewById(R.Id.button_login);
```
而如果你使用kotlin的话，你可以就按如下代码写
```kotlin
import kotlinx.android.synthetic.main.activity_kotlin_main.*
...
button_login.setText("Kotlin Android Extensions 我不太喜欢");
```
这样确实比Java方便多了，不需要对每一个组件都定义再findViewById，可以就直接输入组件的id然后就能使用了。但是我们还是觉得不够啊，为什么我们不能就直接在代码中写入各个组件呢，于是，Anko来了。

# 导入Anko
Anko由几部分组成：
* *Anko Commons*：一个轻量级的库，包含用于Layouts，Intent，Log等的帮助程序;
* *Anko Layouts*：一种快速且类型安全的方式来编写动态Android布局;
* *Anko SQLite*：Android SQLite的查询DSL和解析器集合;
* *Anko Coroutines*：基于[kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines)库的实用程序。

你可以在app的`build.gradle`中添加
```java
dependencies {
     // Anko Layouts
    implementation "org.jetbrains.anko:anko-sdk25:$anko_version" // sdk15, sdk19, sdk21, sdk23 are also available
    implementation "org.jetbrains.anko:anko-appcompat-v7:$anko_version"

    // Coroutine listeners for Anko Layouts
    implementation "org.jetbrains.anko:anko-sdk25-coroutines:$anko_version"
    implementation "org.jetbrains.anko:anko-appcompat-v7-coroutines:$anko_version"}
```
这个依赖可以直接导入所有的可用特性（包括Commons, Layouts, SQLite)。

# 使用Anko layout

## 创建简单布局
使用Anko创建布局很简单：

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        verticalLayout {
            padding = dip(30)
            editText {
                hint = "Name"
                textSize = 24f
            }
            editText {
                hint = "Password"
                textSize = 24f
            }
            button("登录") {
                textSize = 26f
            }
        }
    }
}
```

你只需要在Activity中写入DSL代码就能使用它。

## AnkoComponent
尽管我们现在可以直接在Activity中写入DSL代码，但是我们还是觉得把代码和布局文件放在一起不太好，希望把Activity和DSl代码放到两个不同的类里面，所以AnkoComponent就出来了。
代码如下
```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        MainActivityUI().setContentView(this)
    }
}

class MainActivityUI : AnkoComponent<MainActivity> {
    override fun createView(ui: AnkoContext<MainActivity>) = with(ui) {
        verticalLayout {
            padding = dip(30)
            editText {
                hint = "Name"
                textSize = 24f
            }
            editText {
                hint = "Password"
                textSize = 24f
            }
            button("登录") {
                textSize = 26f
            }
        }
    }
}
```

## Theme
在Anko如果你想设置Theme需要使用`themeable`
```kotlin
verticalLayout {
    padding = dip(30)
    themedButton("登陆", theme = R.style.Base_TextAppearance_AppCompat_Button)
}
```

## LayoutParams
在Anko中也可以使用`LayoutParams`。
比方说我现在要显示一个button，下面显示一个图片。

如果使用XML:
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".MainActivity">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello World!"
        android:gravity=center
        android:textSize="20sp" />

    <ImageView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="@color/colorAccent" />

</LinearLayout>
```

但是使用Anko:

```kotlin
verticalLayout {
    textView ("Hello World!") {
        textSize = sp(12).toFloat()
    }.lparams(height = wrapContent, width = wrapContent) {
        horizontalGravity = Gravity.CENTER_HORIZONTAL
    }
    imageView {
        backgroundColor = Color.BLUE
    }.lparams(height = matchParent, width = matchParent)
}
```

注意：
* `horizontalMargin` 同时设置 left 和 right margins
* `verticalMargin` 同时设置 top 和 bottom
* `margin` 同时设置4个方向的 margins

## Listeners
在Anko中设置`Listeners`非常简单，我下面以`Button`的`OnClick`举例：
```kotlin
button("登录") {
    id = buttonLogin
    textSize = 26f
    onClick { toast("OnClickLoginButton") }
}
```

## 使用Fragment加载
先创建一个`Activity`，把`Fragment`加进去
```kotlin
class AnkoFragmentActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        linearLayout {
            id = R.id.fragment_id
            supportFragmentManager.beginTransaction().replace(id, AnkoFragment.newInstance()).commit()
        }
    }
}
```
然后在`Fragment`的`onCreateView()`中加入DSL代码，然后返回`View`即可
```kotlin
class AnkoFragment : Fragment() {
    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        // return inflater.inflate(R.layout.fragment_anko, container, false)
        return UI {
            verticalLayout {
                editText()
                button("OK")
            }
        }.view
    }
    companion object {
        fun newInstance(): AnkoFragment {
            return AnkoFragment()
        }
    }
}
```

## 不足
AnkoLayout确实挺好的，因为他把UI代码集成到了代码文件中，不用再像以前一样写一个点击事件还得先`findViewBuId`，然后再`setOnClickListeners`。代码能非常简洁。

但是AnkoLayout还是不够完美，感觉写起来还是没有XML那个顺手（可能是我写惯了XML，刚开始用Anko还不够熟练），而且Anko还有很多控件都不支持。
最重要的是，**他没有实时预览**！！！
> Android Studio里面有一个叫`Anko Support`的插件，可以实现anko的预览，但是他必须是先将项目构建了再预览的，不算是实时预览