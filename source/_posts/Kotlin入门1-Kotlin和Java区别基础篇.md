---
title: 'Kotlin入门1:Kotlin和Java区别基础篇'
tags:
  - Kotlin
  - 语言基础
categories: Kotlin
comments: true
abbrlink: 343b01ae
cover: https://cdn.littlecorgi.top/mweb/2020-05-27/kotlin%E5%85%A5%E9%97%A8-%E4%B8%80-%E2%80%94%E2%80%94%E5%A4%B4%E5%9B%BE.png
date: 2019-06-02 17:16:17
updated: 2019-06-02 17:16:17
copyright: true
---
# Koltin入门
# Kotlin简介
> 科特林岛(Котлин)是一座俄罗斯的岛屿,位于圣彼得堡以西约30公里处,形状狭长,东西长度约14公里,南北宽度约2公里,面积有16平方公里,扼守俄国进入芬兰湾的水道。科特林岛上建有喀琅施塔得市,为圣彼得堡下辖的城市。

而我们虽说的kotlin，就是一门根据它命名的一种现代程序设计语言。Kotlin 是一个用于现代多平台应用的静态编程语言，由 JetBrains 开发。Kotlin可以编译成Java字节码，也可以编译JavaScript，方便在没有JVM的设备上运行。Kotlin已正式成为Android官方支持开发语言。

<!-- More -->
-------


# 变量
kotlin和Java的最基本的区别就是kotlin中**万物皆对象**，Java中还存在着int、float等基本类型，但是在kotlin中，它把这些都定义成了对象，类似于Java中的封装类
## 变量的声明
上面说了，kotlin万物皆对象，所以所有的变量也都是对象。
在kotlin定义对象和Java有点小区别。
kotlin定义对象的格式为
>  声明类型 变量名： 变量类型

其中：
* 声明类型分为`val`和`var`。`val`是不可变类型，类似于const，定义时必须赋值，赋值后不能被修改。`var`是可变类型。
* 变量名就是你定义的这个变量的名称。
* 变量类型就是你这个变量对应的类的名字。

## 类型推断
### 省去变量类型
kotlin里面类似c++的`auto`类型，对于基本类型，你可以不写变量类型，kotlin会自动帮你判断。

```kotlin
var a = 5
println(a is Int)

var b = “123
println(b is String)

var c = 1.3
println(c is Float)

var d = true
println(d is Boolean)
```

### `is`关键字
`is`顾名思义，就是判断这个变量是不是这个类型的实例。
例子见上。

## 数字类型

|  类型  |  宽度(bit)  |
| --- | --- |
|  Byte |   8 |
| Short |  16  |
|  Int  |  32  |
|  Long  |  64  |
|  Float  |  32  |
| Double   |  64  |

这些类型都继承自`Number`和`Comparable`类。

### 字面常量值
* 十进制：`123`
* 十六进制：`0x0f`
* 二进制：`0b0010`
* Long类型：`123L`
* double类型：123.4
* Float类型：`123.4f`或者`123.4F`

我们也可以使用下划线`_`来方便我们阅读

```kotlin
1_000_000 // 1000000
0xFF_EC // 0xFFEC
```
### 显示转换
kotlin中不可隐式转换
比如Java 中

```java
int a = 2;
long b = a;
```
但是在kotlin中

```kotlin
var a : Int? = 2
var b : Long? = a; // error
var b : Long? = a.toLong()
```

## Char类型
kotlin中的`Char`表示字符。但是和Java不同，他不能直接当ASCII码值。

```kotlin
fun check(c : Char) {
    if (c == 1) { // error 
    
    }
}
```

## Boolean类型
kotlin中的布尔类型用`Boolean`来表示，他有两个值`true`和`false`。用法和Java一样。

## String类型
和Java一样，kotlin中的字符串也是`String`。但是kotlin中`String`是不可变的。所以kotlin中`String`必须是`val`类型。同时，`String`是`final`不可继承的。

## Array类型
kotlin中数组必须使用`Array`表示。
基本写法
> val array: Array<类型> = arrayOf(..)

例如
```kotlin
/**整型Int的数组*/
val arrayOfInt: IntArray = intArrayOf(1,3,5,7,9)
/**字符Char类型的数组*/
val arrayOfChar: CharArray = charArrayOf('H','e','l','l','o','W','o','r','l','d')
/**字符串String数组*/
val arrayOfString: Array<String> = arrayOf("Hello","World")

fun main(args: Array<String>) {
    //查看有多少个元素
    println(arrayOfInt.size)
    //遍历数组
    for (char in arrayOfChar){
        println(char)
    }

    //根据所引获取数据,数组是从0开始的，现在获取第二个东京大学
    println(arrayOfUniversity[1])
    //重新给数组赋值，早稻田大学
    arrayOfUniversity[1] = University("早稻田大学")
    println(arrayOfUniversity[1])

    //将char连接成一个字符串,默认是自动由逗号","分割的，输出H, e, l, l, o, W, o, r, l, d
    println(arrayOfChar.joinToString())
    //如果想要连成HelloWorld
    println(arrayOfChar.joinToString (""))

    //数组的切片,输出3，5,结尾需要arrayOfInt-1，不然会报索引越界异常
    println(arrayOfInt.slice(1..2))

    println(arrayOfInt.size)
}
```