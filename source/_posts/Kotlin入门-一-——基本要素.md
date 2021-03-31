---
title: Kotlin入门(一)——基本要素
tags:
  - Kotlin
  - 语言基础
categories: Kotlin
cover: https://cdn.littlecorgi.top/mweb/2020-05-27/kotlin%E5%85%A5%E9%97%A8-%E4%B8%80-%E2%80%94%E2%80%94%E5%A4%B4%E5%9B%BE.png
abbrlink: 2ed82411
date: 2020-05-27 15:52:53
updated: 2020-05-27 15:52:53
---
# Kotlin入门(一)——基本要素

> 本章内容包括：
> - kotlin的HelloWorld
> - 变量、控制流、函数
> - 包和类（最基础的内容，具体的后面会介绍）
> - 智能转换

# 0. 前言
我之前写过一篇[《Kotlin入门》](https://www.littlecor/gi.top/%2Fposts%2F343b01ae.html)博客，但是一方面是这篇博客写的比较早，写的时候单纯是为了学习anko而写的，所以感觉写的并不好，另一方面，在写的时候只是写了点基础知识，当时也没有系统的学习kotlin。

所以在看完kotlin实战后，就想回过头来写一篇总结博客

写这个主要是为了几个目的：
- 对于看我博客的人来说有一个更加系统更加全面更加完善的kotlin入门指南。
- 对于我自己，单纯的看书还是不够好，就写一篇总结博客总结下书本中的知识。

注：由于我之前只会Java，并且目前主要是在做Android开发，所以我的博客更多的会将kotlin与Java进行比较，并且可能会偏向Android开发中的实践。

<!-- More -->

# 1. Hello World
万恶之源，我们从程序员的第一个程序“Hello World”开始。

我们先来看下kotlin版的Hello World是怎么写的，如果你之前学习过Java，你会立马被kotlin语法的简洁所震惊：
```kotlin
package kotlinstudy

fun main() {
    println("Hello World")
}
```

如果你之前是Java工程师，或者你之前学过Java，是不是会感觉到非常的惊讶：
- 熟悉的`public static void main(String[] args)`不见了，取而代之的是`fun main()`
- 熟悉的`System.out.println()`不见了，取而代之的是`println()`
- 熟悉的main方法必须在一个class里面不见了，取而代之的是方法可以直接写在最外面。~~*这个叫顶层层函数，我们后面会讲到*~~

我们都知道，kotlin也是运行在JVM之上的，也就是他也必须遵守JVM的一些约定，那么他是如何能做到这样的语法的呢？

如果你使用IDEA或者Android Studio的话，你可以在上面的Tools-Kotlin里面看到一个`Show Kotlin Bytecode`的选项：
![-w743](https://cdn.littlecorgi.top/mweb/2020-05-27/15895156953362.jpg)

我们打开它，就能看到我们刚刚编写的kotlin代码的字节码文件了，但是这个时候还是看不懂，不急，我们可以点一下这个`Decompile`按钮，然后就能转成更易懂的语句了。
![-w1393](https://cdn.littlecorgi.top/mweb/2020-05-27/15895157853281.jpg)
![-w765](https://cdn.littlecorgi.top/mweb/2020-05-27/15895158024195.jpg)

看到转过来的语法，你就能懂了，为什么kotlin能运行在JVM只上了。其实kotlin运行的最终本质仍然还是JVM的class文件，只不过他使用了一层语法糖，所谓的`fun main()`、`println()`等等，最终都会转成Java语法对应的东西。

所以我们可以得出一个结论：kotlin的一切方便，只不过都是一层语法糖而已。（可能存在一定的过与绝对，但是道理也确实还是这个道理）


# 2. 函数
## 2.1 函数声明
上面我们通过一个简单的HelloWorld了解到了main函数的定义方法，我们就一个简单的函数来大致了解下kotlin中函数的定义：
```kotlin
fun max(a: Int, b: Int): Int {
    return if (a > b) a else b
}
```

我们就来将这个函数拆解一下，逐步讲一下他的各个组成部分：
- `fun`：fun关键字是kotlin中函数的标识，就是说想在kotlin中写一个函数的话，必须以fun开头，否则kotlin是不会将它识别成一个函数的，反而还可能会报错。

- `max`：紧跟着fun的，就是函数的名称了。
- `(a: Int, b: Int)`：函数名后面就是用括号括起来的形参列表了，每个形参之间用逗号(',')分隔开来。而参数的定义方法或者说变量的定义方法想必大家已经看出来了，`a`就是变量名，然后`Int`就是变量类型。由于kotlin没有基本类型，也就是说他不像Java有int、float等这些基本类型，所以变量类型就是`Int`了。
- `: Int`：大家应该已经看出来了，和定义类型类似，这个就代表函数的返回值是Int类型的。
- 后面的就和Java类似，就是用一组大括号括起来的函数体。只不过kotlin没有三目运算符，所以Java中的`?:`在kotlin中就成了`if else`。

## 2.2 表达式函数体
在kotlin中，你可以让简单的事物尽可能简单的展现出来(在后面你可以看到更多这类例子)。

比方说我们上面的`max`函数，他的功能就是接受`a`和`b`两个参数，然后返回他们中的较大值。我们写的代码已经够简洁了，但是，这个只是对于Java来说够简洁，但是对于kotlin来说，他还不够。

下面让我们来看一下如何用表达式函数来重写这个`max`函数：
```kotlin
fun max(a: Int, b: Int): Int = if (a > b) a else b
```

对于上面那个初级版本的`max`函数来说，这个已经够简洁了。

对应的，如果函数体写在大括号中，我们就说这个函数有代码块体。如果他只是简单的一个表达式，他就有表达式体。

但是，我们刚刚说到，kotlin想让简单的事物尽可能简单的展现出来。所以，对于这个已经非常明显的返回值类型，我们何必要把它显式的定义出来呢：
```kotlin
fun max(a: Int, b: Int) = if (a > b) a else b
```

## 2.3 参数

### 2.3.1 默认参数
有时候，我们想要缺省函数的参数时，我们就可以对参数使用默认值：
```kotlin
fun max(a: Int, b: Int = 0) = if (a > b) a else b
```

这个时候我们调用它的时候，就可以缺省有默认值的参数：
```kotlin
print(max(1))
```

但是当在设置了默认值参数的后面有没有设置默认值的参数的话，我们调用的时候必须显式指明这个值到底是属于哪个参数的：
```kotlin
// 函数定义
fun max(a: Int = 0, b: Int): Int = return if (a > b) a else b

// 函数调用
print(max(b = 1))
```


# 3. 变量(类和对象)
## 3.1 不可变变量和可变变量
和Java不同，在kotlin中，变量的定义主要通过`val`和`var`。这个语法有点类似于JS。

- `val`：用于值从不更改的变量。您不能为使用 val 声明的变量重新赋值。对应Java是`final`。
- `var`：用于值可以更改的变量。对应Java是普通(非final)的变量。

```kotlin
val count: Int = 10
var num: Int = 10
count = 1 // Error：因为count是val，并且已经初始化过了
num = 1
```

在默认情况下，应该尽可能的去使用`val`来声明所有的kotlin变量。

同Java的final一样，被val修饰的变量，它本身是不能变的，但是他指向的对象是可以变的：
```kotlin
val a = ArrayList<String>()
a.add("1234") // 改变了所引用的对象
a.add("1324") // 改变了所引用的对象
a = ArrayList<String>() // Error：改变了他自身
```

## 3.2 类型推断
在某些时候，变量的类型可以省略不写，然后编译器会根据所赋值的类型来推断类型：
```kotlin
val languageName = "Kotlin"
val upperCaseName = languageName.toUpperCase()
```

但是一旦变量的类型在定义时被确定下来后，后面是不许再更改类型了：
```kotlin
var answer = 42 // answer的类型是Int
answer = "no answer" //Error：类型不匹配
```

## 3.3 字符串模板
我们前面写过HelloWorld，但是我们现在想根据人名，输出Hello Name，那么这个用kotlin该怎么实现：
```kotlin
fun main() {
    val name = "Kotlin"
    println("Hello $name")
}
```

我们可以看到字符串中有一个`$变量名`类型的结构，其实这个结构完整是这样的`${变量名}`，只不过对于单纯的变量，可以省略花括号。

这个就是字符串模板。

除了变量外，字符串模板中还可以写入表达式：
```kotlin
fun main() {
    val name = "Kotlin"
    println("Hello ${name.toUpperCase()}")
}
```

甚至更复杂的都可以写：
```kotlin
fun main() {
    val sc = Scanner(System.`in`)
    val name = sc.next()
    println("Hello ${if (name.isNotEmpty()) name else "Kotlin"}")
}
```

# 4. 控制流
在kotlin中，控制流主要有`if`、`when`、`for`、`while`。
## 4.1 if
在大部分语言中，`if`就是简单的判断，kotlin也不例外。

但是在kotlin中，`if`是一个表达式，她是有返回值的。这也是为什么在kotlin中没有`?:`，因为`if else`就可以替代他。

```kotlin
// 传统用法
var max = a 
if (a < b) max = b

// With else 
var max: Int
if (a > b) {
    max = a
} else {
    max = b
}
 
// 作为表达式
val max = if (a > b) a else b
```

## 4.2 when 
有一个很常见的案例，当我们需要判断一个变量的值的时候，当在某个区间执行某个代码，在另一个区间执行另一个代码(比如我们高中是学习过的那种区间的二元一次方程求解)，这个时候如果我们使用`if`的话，就需要特别多的`if else`。

而如果我们有一种东西，他可以只需要指定判断的变量，然后后面就可以下对应的区间并执行对应的代码，这样是不是比单纯的大量的`if else`简洁的多。

所以我们就需要使用`when`。

### 4.2.1 when的基础用法
在kotlin中，用`when`取代了C、C++、Java中的`switch`，功能一模一样，只不过有一小区别，我们边看代码边说：
```kotlin
val x = 0

when (x) {
    0 -> {
        println("x == 0")
    }
    1 -> {
        println("x == 1")
    }
    2 -> {
        println("x == 2")
    }
    else -> {
        println("没有被定义")
    }
}
```

和`switch`一样，括号里面是用来匹配的变量，然后下面就是匹配的值，当匹配到某个值的时候，就执行这个值后面的代码块。

但是和Java及C++的`switch`不同的是，他不需要写`break`语句。

当其他分支都不满足条件的时候，就会自动进入`else`分支，去执行`else`分支的代码。在kotlin中，`else`分支是必须有的。

当有多个分支需要执行同样的代码时，就可以把多个分支的条件放在一起：
```kotlin
val x = 0

when (x) {
    0, 1 -> {
        println("x == 0")
    }
    2 -> {
        println("x == 2")
    }
    else -> {
        println("没有被定义")
    }
}
```

同时我们也可以使用区间(这个我们后面将for循环的时候会讲到)作为分支的判断条件：
```kotlin
val x = 0

when (x) {
    in 0..10 -> {
        println("x == 0")
    }
    else -> {
        println("没有被定义")
    }
}
```

### 4.2.2 在when中使用任意对象
`when`比Java中的`switch`要强大的多，在`switch`中只能用常量作为分支条件，但是在`when`中，可以使用任何对象：
```kotlin
val a: Int = 1
val b: Int = 0
when (setOf(a, b)) {
    setOf(0, 0) -> {
    }
    setOf(0, 1) -> {
    }
    setOf(1, 0) -> {
    }
    setOf(1, 1) -> {
    }
}
```

### 4.2.3 类型检测与智能转换
#### 类型检测
使用`when`的时候还可以去判断当前这个对象是属于哪个类的对象。

kotlin给我们提供了一个关键字`is`，使用这个关键字，我们可以去判断当前这个对象是否符合给定的类型：
```kotlin
if (obj is String) {
    print(obj.length)
}

if (obj !is String) { // 与 !(obj is String) 相同
    print("Not a String")
} else {
    print(obj.length)
}
```

#### 智能转换
`is`的厉害之处不在于他能判断对象是否符合某个给定的类型，而是在于当他检测过某个变量是某种类型的时候，后面就不用再进行转换了，可以就把他当做检查过的类型使用，这就是智能转换：
```kotlin
fun demo(x: Any) {
    if (x is String) {
        print(x.length) // x 自动转换为字符串
    }
}
```

#### 在when中使用
```kotlin
when (x) {
    is Int -> print(x + 1)
    is String -> print(x.length + 1)
    is IntArray -> print(x.sum())
}
```

## 4.3 while
相信一定有很多人都和我一样，看到这块会有点疑问。因为我看过的大部分语言学习的书，比方说C、Java、C++的大部分语言入门类书，都会将`for循环`放在`while循环`的前面。而当我在看《kotlin实战》这本书的时候，却发现它把`while`放在了`for`的前面。

但是当我看了会之后就懂了，kotlin的`while`和Java以及C的`while`没有啥区别，但是kotlin的`for`和Java的`for`有点区别，并且有很多操作。所以会把`while`放在前面，并且会一笔带过。

kotlin的while也分为`while`和`do-while`。用法和Java的没有任何区别：
```kotlin
while (x > 0) { // 当条件符合才进入循环
    x--
}

do { // 先会进入代码块一只，然后执行完毕后才会去判断条件，如果符合就再去执行循环
  val y = retrieveData()
} while (y != null) // y 在此处可见
```

## 4.4 for
我们刚刚说到过，kotlin的`for`和Java的`for`有着较大的区别。

我们在Java中常用的for有两种：`简单的for`和`for-each`：
- 简单的for就是`for(变量定义; 执行的条件; 变量每次循环需要执行的语句，一般都是控制变量的值的变化)`

```java
for (int i = 0; i < 10; i += 2) {
    
}
```
- for-each就是`for(临时变量 in 集合)`

```java
ArrayList<Integer> arrayList = new ArrayList<>();
for (i in arrayList) {

}
```

此时i就相当于是arrayList这个集合里面存着的每一个对象。

而kotlin的`for`就和Java的`for-each`一致。

### 4.4.1 for-each
`for`循环可以对任何提供了`迭代器(iterator)`的对象进行遍历。
```kotlin
for (item in collection) print(item)
```

for可以循环遍历任何提供了迭代器的对象。即：
- 有一个成员函数或者扩展函数`iterator()`，它的返回类型
    - 有一个成员函数或者扩展函数`next()`，并且
    - 有一个成员函数或者扩展函数`hasNext()`返回`Boolean`

### 4.4.2 迭代数字：区间..
对于我来说，kotlin的使用`for`迭代数字比Java更为方便，但是有些时候却反而很不方便。

在使用`for`迭代数字之前，我们先来看kotlin的区间表达式。

区间本质上就是两个值之间的间隔，这两个值通常就是数字：一个代表了起始值，一个代表了结束值。
```kotlin
if (i in 1..4) { // 等同于1 <= i <= 4
    print(i)
}
```

而`in`这个函数实际上是`rangeTo()`。所以说，实际上我们调用的是`rangeTo()`函数。

但是我们可以看下[kotlin官方文档](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.ranges/range-to.html)对这个函数的定义：
![kotlin入门-一-——基本要素4](https://cdn.littlecorgi.top/mweb/2020-05-27/kotlin%E5%85%A5%E9%97%A8-%E4%B8%80-%E2%80%94%E2%80%94%E5%9F%BA%E6%9C%AC%E8%A6%81%E7%B4%A04.png)

翻译过来就是:
> 创建一个从这个继承自Comparable接口的value到that的range(区间)
> value必须小于that，否则这个range将会是空的。

所以就是说`rangeTo()`或者说`..`只能从小到大的遍历，反过来则不行。

但是我们上面的例子是从1逐步到4，也就是说打印出来的是1，2，3，4。
但是如果我只想要输出1到4之间的奇数呢？
也就是说，如果我想在1到4之间每隔2个输出一次，那么该怎么操作呢：

```kotlin
for (i in 1..4 step 2) {
    print(i)
}
```

这样就能从1开始每隔2执行一次，也就是输出1，3。

`..`相当于是闭区间，那么开区间呢？
如果要实现开区间，我们可以使用`until`
```kotlin
for (i in 1 until 4 step 2) {
    print(i)
}
```

我们刚刚说的全部都是从小到大，那么从大到小呢？
```kotlin
for (i in 8 downTo 1) print(i)
```

这样就会实现从8开始输出字符直到1为止。

### 4.4.2 为什么我会说kotlin的for在有些地方没有Java的for方便
在使用`for (i in 0..10`的时候，如果我们需要操作指针是没法操作的，举个例子：
```kotlin
for (i in 1..2) {
    i-- // Error：val不能被更改
}
```

现在我们就懂了，因为kotlin自动把i这个变量定义为val的。

那么如果我们把它手动设定为var呢：
```kotlin
for (var i in 1..2) { // Error：'var'不允许被放在循环的参数中
    i--
}
```
也就是说kotlin根本不允许var出现在这，或者说任何东西都不允许出现在这(val也不行)，因为这样不符合kotlin的语法。

所以在kotlin中，如果你想在循环中操作指针，就只能通过while了。