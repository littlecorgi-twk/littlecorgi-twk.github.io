---
title: OpenGL ES 2.0笔记1——简介
categories:
  - OpenGL
tags:
  - OpenGL
  - Android
  - 图像处理
comments: true
copyright: true
abbrlink: a92b4a54
date: 2019-10-17 10:43:11
updated: 2019-10-17 10:43:11
feature_img: 
cover: 'https://gss2.bdstatic.com/9fo3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike150%2C5%2C5%2C150%2C50/sign=8045afbe12d5ad6ebef46cb8e0a252be/78310a55b319ebc4e51be2e38b26cffc1e17166c.jpg'
keywords:
description:
---
# OpenGL是什么？OpenGL ES又是什么？
## 简介
OpenGL是一个跨平台的软件接口语言，用于调用硬件的2D、3D图形处理器。

然而受限于现在的移动设备性能，如果将OpenGL直接用在他们上面将会特别卡，于是就出现了OpenGL ES。 OpenGL ES是OpenGL的分支，他专门作用于嵌入式设备，去掉了OpenGL很多不必要的功能。

<!--More-->

## 应用场景
- 游戏
- 视频播放器
- 视频编辑应用
- 图片编辑应用

等对图像处理的及时性要求较高的应用场景。

## Android对OpenGL ES 的支持
|OpenGL ES 版本|基于OpenGL的版本|Android引入的版本|兼容性|功能、特色|
|:-:|:-:|:-:|:-:|:-:|
|1.0&1.1|1.3&1.5|Android 1.0|-|固定的图像管道，开发难度相比2.0低|
|2.0|2.0|Android 2.2|不兼容1.x|可编程的渲染管道，性能效率更高，开发难度更高|
|3.0|3.x|Android 4.3|兼容2.0|性能更高，支持ETC2格式的透明纹理压缩|
|3.1|4.x|Android 5.0|兼容2.0/3.0|新增计算着色器、单独的着色器对象等新特性|

## 版本的选择
由于此篇博客是学习阶段写的，而我在网上找到的大部分博客都是2.0，所以就以2.0为主。

其实我觉得应该是3.0，2.0确实太老了，现在Android4.3的手机都较少见了，而3.1又太新了。

## 导入
首先你得在AndroidManifest.xml中加入
`<uses-feature android:glEsVersion="0x00020000" android:required="true" />`

其中不同版本对应值：

|OpenGL ES 版本|glEsVersion 版本|
|:-:|:-:|
|2.0|0x00020000|
|3.0|0x00030000|
|3.1|0x00030001|

# 基本框架
Android框架里面提供了两个类来给你使用OpenGL ES API创建和操作图形：`GLSrufaceView`和`GLSurfaceView.Renderer`。

## GLSrufaceView
这是一个视图类，你可以通过OpenGL ES API来绘制和操作图形对象，他在功能上很类似与`SurfaceView`。你可以通过创建一个`SurfaceView`的实例并添加你的渲染器来使用这个类。

他的常用方法有：
- `setEGLContextClientVersion`：设置OpenGL ES版本，2.0则设置2
- `onPause`：暂停渲染，最好是在`Activity`、`Fragment`的`onPause()`方法内调用，减少不必要的性能开销，避免不必要的崩溃
- `onResume`：恢复渲染，用法类比`onPause()`
- `setRenderer`：设置渲染器
- `setRenderMode`：设置渲染模式
- `requestRender`: 请求渲染，由于是请求异步线程进行渲染，所以不是同步方法，调用后不会立刻就进行渲染。渲染会回调到`Renderer`接口的`onDrawFrame()`方法。
- `queueEvent`：插入一个`Runnable`任务到后台渲染线程上执行。相应的，渲染线程中可以通过`Activity`的`runOnUIThread`的方法来传递事件给主线程去执行

其中，GLSurfaceView的渲染模式有：
- `RENDERMODE_CONTINUOUSLY`：不停地渲染
- `RENDERMODE_WHEN_DIRTY`：只有调用了`requestRender()`之后才会触发渲染回调onDrawFrame方法

## GLSurfaceView.Renderer
此接口定义了在GLSurfaceView中绘制图形所需的方法。必须将此接口的实现作为单独的类提供，并使用GLSurfaceView.setRenderer()将其添加到GLSurfaceView的实现类中。

他的主要方法有三个：
- `onSurfaceCreated(GL10 gl, EGLConfig config)`：GLSurfaceView内的Surface被创建时会被调用到
- `onSurfaceChanged(GL10 gl, int width, int height)`：Surface尺寸改变时调用到
- `onDrawFrame(GL10 gl)`：渲染绘制每一帧时调用到

一般情况下，首次创建GLSurfaceView时，会顺序调用`onSurfaceCreated()` -> `onSurfaceChanged()` -> `onDrawFrame()`。然后每绘制一帧，都会不停地回调`onDrawFrame()`方法。

# 编写流程
首先我们来写一个基本框架。

创建一个Android项目，然后Activity选择EmptyActivity。

**注意：本项目某些类或者方法通过AndroidStudio导入的时候会让你选择是从哪个包里面导入，一致选择`android.opengl.GLES20`**

## 编写Activity
打开我们项目的MainActivity，你就能看到熟悉的onCreate方法，接下来我们要对这个Activity进行一些更改。

首先，我们要创建一个GLSurfaceView的对象,并添加一个新的成员变量：
```java
private GLSurfaceView glSurfaceView;
private boolean rendererSet = false;
```

接下来将`setContentView()`移除，并对glSurfaceView进行初始化：
```java
@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    glSurfaceView = new GLSurfaceView(this);
}
```

由于我们是在写OpenGL ES2.0的代码，所以我们需要检测当前运行APP的系统支不支持OpenGL ES2.0。我们将下面几行代码添加到onCreate()里面去:
```java
final ActivityManager activityManager = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
final ConfigurationInfo configurationInfo = activityManager.deviceConfigurationInfo();
final boolean supportsEs2 = configurationInfo.reqGlEsVersion >= 0x20000;
```

但是这段代码在模拟器上不能工作，因为GPU模拟部分有缺陷。为了使代码在模拟器上运行，我们要按如下代码修改检查条件：
```java
final boolean supportsEs2 = configurationInfo.reqGlEsVersion >= 0x20000
    || (Build.VERSION.SDK_INT >= Build.VERSION_CODES.ICE_CREAM_SANDWICH_MR1
    && (Build.FINGERPRINT.startsWith("generic")
    || Build.FINGERPRINT.startsWith("unknown")
    || Build.MODEL.contains("google_sdk")
    || Build.MODEL.contains("Emulator")
    || Build.MODEL.contains("Android SDK build for x86")))
```

接下来就是配置渲染表面：
```java
if (supportsEs2) {
    // 配置这个surface视图
    glSurfaceView.setEGLContextClientVersion(2);

    // 配置Renderer
    glSurfaceView.setRenderer(FirstOpenGLProjectRenderer());
    rendererSet = true;
} else {
    Toast.makeText(this, "This device does not support OpenGL ES 2.0", Toast.LENGTH_SHORT).show();
    return;
}

setContentView(glSurfaceView);
```

最后我们还得处理下Activity的生命周期，也就是将GLSurfaceView的生命周期跟Activity的生命周期绑定起来：
```java
@Override
void onPause() {
    super.onPause();

    if (rendererSet)
        glSurfaceView.onPause();
}

@Override
void onResume() {
    super.onResume();

    if (rendererSet)
        glSurfaceView.onResume();
}
```

## 编写Renderer
创建一个新的文件，命名为FirstOpenGLProjectRenderer，让让这个类继承自GLSurfaceView.Renderer。

接着就实现三个必备方法:
- `onSurfaceCreate()`：创建时被调用

```java
@Override
public void onSurfaceCreated(GL10 glUnsed, EGLConfig config) {
    glClearColor(1.0f, 0.0f, 0.0f, 0.0f); //设置一个清空屏幕用的颜色
}
```

- `onSurfaceChanged()`：创建后每当Surface尺寸改变时，都调用此方法

```java
@Override
public void onSurfaceChanged(GL10 glUnsed, int width, int height) {
    glViewport(0, 0, width, height); //设置视窗尺寸
}
```

- `onDrawFrame()`：绘制每一帧时调用

```java
@Override
public void onDrawFrame(GL10 glUnsed) {
    glClear(GL_COLOR_BUFFER_BIT); //清空屏幕
}
```

然后就可以运行程序了，你就可以看到整个屏幕都成了红色，这是为啥呢？
因为我们在onSurfaceCreated方法里面设置了glClearColor(1.0f,0.0f,0.0f,0.0f)。这个方法的参数对应的是RGBA，然后我们又在onDrawFrame方法中设置了清屏，所以这样你就会看到的是红色了。