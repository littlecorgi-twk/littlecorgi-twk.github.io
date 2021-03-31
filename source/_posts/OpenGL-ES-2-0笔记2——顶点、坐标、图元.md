---
title: OpenGL ES 2.0笔记2——顶点、坐标、图元
categories:
  - OpenGL
tags:
  - OpenGL
  - Android
  - 图像处理
comments: true
copyright: true
abbrlink: 268eb1c2
date: 2019-10-18 21:06:38
updated: 2019-10-18 21:06:38
feature_img: 
cover: 'https://gss2.bdstatic.com/9fo3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike150%2C5%2C5%2C150%2C50/sign=8045afbe12d5ad6ebef46cb8e0a252be/78310a55b319ebc4e51be2e38b26cffc1e17166c.jpg'
keywords:
description:
---

> 上节我们对OpenGL做了一个大致的介绍，并写了一个基本框架，在这节中我们将会介绍顶点、坐标系和图元

<!--More-->

# 顶点
在OpenGL中，所有的东西的结构都是从一个顶点开始的。

所谓顶点，就是一个几何图形的拐点。在介绍顶点之前，我们首先介绍下OpenGL坐标。

## OpenGL坐标系
如果有会Android开发的朋友，一定会默认为从屏幕的左上角开始，水平往右是x轴，竖直向下是y轴。

但是在OpenGL中是不同的，OpenGL是从显示视窗的正中心是中心，也就是$(0, 0)$，而屏幕最左边的x轴坐标是-1，屏幕最右边的x轴坐标是1，屏幕最上面的y轴坐标是1，屏幕最下面的y轴坐标是-1，也就是下面这幅图：

![opengl_coordinate](https://cdn.littlecorgi.top/mweb/2019-10-18/opengl_coordinate.png)

## 在代码中定义顶点
在OpenGL中定义顶点有点特殊，我们第一反应都是直接拿数组将顶端存起来就行了，但是在OpenGL中，你除了要将顶点的坐标用数组存起来以外，还得定义一个常量，用来标记一个顶点有两个分量：
```java
private static final int POSITION_COMPONENT_COUNT = 2;
float[] tableVerticesWithTriangles = {
    // Triangle 1
    -0.5f, -0.5f, 
    0.5f,  0.5f,
    -0.5f,  0.5f,

    // Triangle 2
    -0.5f, -0.5f, 
    0.5f, -0.5f, 
    0.5f,  0.5f,

    // Line 1
    -0.5f, 0f, 
    0.5f, 0f,

    // Mallets
    0f, -0.25f, 
    0f,  0.25f
};
```

我们采用浮点数的顺序列表定义顶点数据，这个数组通常被成为顶点属性。

# 图元
在OpenGL中，只能绘制点、直线和三角形，这三个也被称为图元。

也就意味着，其它所有的图形都得通过这三个基本图元来实现。

如果我们需要绘制一个点，那么一个坐标就可以了，但是如果我们需要绘制一条直线，就需要两个坐标，如果绘制一个三角形，就需要三个坐标。

那么我们就以绘制一个正方形举例：

```java
float[] squareVerticesWithTriangles = {
    -0.5f, -0.5f, 
    0.5f,  0.5f,
    -0.5f,  0.5f,
    0.5f, -0.5f
};
```

![坐标](https://cdn.littlecorgi.top/mweb/2019-10-18/%E5%9D%90%E6%A0%87.png)


如果我要在这个正方形中间加一个横线的话，那么就直接在上面的数组里面去加，到时候要将顶点着色器加载到图形中的时候再去处理：
```java
float[] squareVerticesWithTriangles = {
    // Square
    -0.5f, -0.5f, 
    0.5f,  0.5f,
    -0.5f,  0.5f,
    0.5f, -0.5f,
    
    // Line
    -0.5f, 0,
    0.5f, 0
};
```

![线正方形坐标](https://cdn.littlecorgi.top/mweb/2019-10-18/%E7%BA%BF%E6%AD%A3%E6%96%B9%E5%BD%A2%E5%9D%90%E6%A0%87.png)


# 让数据可以被OpenGL读取
在上面，我们已经完成了顶点的定义，但是现在OpenGL尚还不能读取他们。其原因是这些代码的运行环境与OpenGL运行环境使用了不一样的语言。

当我们运行Java语言的时候，并不是直接运行在硬件上的，而是运行在JVM(纯Java)或者Dalvik(Android)虚拟机上的，并且运行在这些虚拟机上的时候，并不能直接访问本地环境；其次，虚拟机还采用的垃圾回收机制，当检测到变量、对象不再被使用时，就自动释放掉这些内存。

但是本地环境不能这样，因为他是直接运行在硬件上的，他不希望内存会被移来移去或者自动释放。

所以我们就需要一个方案，让Android程序与OpenGL进行通信。
其实Android代码与硬件通信有两种方法：
1. 通过JNI技术；
2. 通过调用android.opengl.GLES20包里面的方法。

我们这块主要是通过第二种方法。

Java中有一种特殊的集合，可以分配本地的内存块，并将Java的数据复制到本地内存。代码如下：
```java
// 因为Java中国一个Float型数据占4个字节
private static final int BYTES_PER_FLOAT = 4;
// 声明一个字节缓冲区
private final FloatBuffer vertexData;
 
vertexData = ByteBuffer
    // 分配一块本地内存，所以是数组长度 * 类型所占字节
    .allocateDirect(tableVerticesWithTriangles.length * BYTES_PER_FLOAT)
    // 按照本地字节序组织内容
    .order(ByteOrder.nativeOrder())
    .asFloatBuffer();
    // 将数据存入
    .put(tableVerticesWithTriangles);
```