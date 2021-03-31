---
title: AndroidView之PopupWindow
tags:
  - Android
  - View
categories: Android
comments: true
abbrlink: 1900f7eb
date: 2019-05-31 19:21:15
updated: 2019-05-31 19:21:15
copyright: true
---
# AndroidView之PopupWindow

因为项目中有很多地方都需要使用PopupWindow，所以特别查了一下，做了一个简单的总结，过两天就加到项目中去。

<!-- More -->

# PopupWindow
PopupWindow，顾名思义，就是一个用来显示弹窗的组件。

# 创建步骤
1. 创建PopupWindow实例
2. 设置一些基本参数
3. 显示PopupWindow
# 构造方法

```java
PopupWindow window = new PopupWindow(View contentView, int width, int height, boolean focusable);
```

这个方法有四个参数，第一个参数是用于PopupWindow中的View，第二个参数是PopupWindow的宽度，第三个参数是PopupWindow的高度，第四个参数指定PopupWindow能否获得焦点。

示例代码

```java
View contentView = LayoutInflater.from(MainActivity.this).inflate(R.layout.popuplayout, null);
PopupWindwo window = PopupWindow (contentView, ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT, true);
```

# 基本方法
- window.setBackgroundDrawable(Drawable background); 设置PopupWindow的背景
- window.setOutsideTouchable(boolean touchable); 设置PopupWindow是否能响应外部点击事件
- window.setTouchable(boolean touchable); 设置PopupWindow是否能响应点击事件
只有同时设置PopupWindow的背景和可以响应外部点击事件，它才能“真正”响应外部点击事件。

# 显示PopupWindow
- window.showAtLocation(View parent, int gravity, int x, int y); 第一个参数是PopupWindow的父View，第二个参数是PopupWindow相对父View的位置，第三和第四个参数分别是PopupWindow相对父View的x、y偏移
- window.showAsDropDown(View anchor, int xoff, int yoff, int gravity); 第一个参数是PopupWindow的锚点，第二和第三个参数分别是PopupWindow相对锚点的x、y偏移

# 为PopupWindow添加动画
- 进入时动画：(context_menu_enter.xml)

```java
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <translate
        android:duration="@android:integer/config_shortAnimTime"
        android:fromXDelta="0"
        android:fromYDelta="100%p"
        android:interpolator="@android:anim/accelerate_decelerate_interpolator"
        android:toXDelta="0"
        android:toYDelta="0"/>
 
</set>
```
- 退出时动画：(context_menu_exit.xml)

```java
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android" >
 
    <translate
        android:duration="@android:integer/config_shortAnimTime"
        android:fromXDelta="0"
        android:fromYDelta="0"
        android:interpolator="@android:anim/accelerate_decelerate_interpolator"
        android:toXDelta="0"
        android:toYDelta="100%p" />
 
</set>
```
- 生成style

```
<style name="contextMenuAnim" parent="@android:style/Animation.Activity">
    <item name="android:windowEnterAnimation">@anim/context_menu_enter</item>
    <item name="android:windowExitAnimation">@anim/context_menu_exit</item>
</style>
```