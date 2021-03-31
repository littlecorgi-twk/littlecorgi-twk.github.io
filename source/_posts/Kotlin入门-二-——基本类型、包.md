---
title: Kotlin入门(二)——基本类型、包
tags:
  - Kotlin
  - 语言基础
categories: Kotlin
cover: >-
  https://cdn.littlecorgi.top/mweb/2020-05-27/kotlin%E5%85%A5%E9%97%A8-%E4%B8%80-%E2%80%94%E2%80%94%E5%A4%B4%E5%9B%BE.png
abbrlink: eef353d0
date: 2020-06-10 15:41:08
updated: 2020-06-10 15:41:08
---
> 本章内容包括：
> - kotlin的基本类型
> - 包
> - 类与对象

# 0. 前言
在[上篇文章](https://blog.csdn.net/a1203991686/article/details/106379486)我们涉及到了kotlin的一些最基本的语法内容，并完成了kotlin的HelloWorld。但是上篇我们在谈到类的时候，说了只介绍下最基本的类，于是在这篇，我们就着重看下类和对象。(更新：由于类和对象的内容过多，我会放在下一篇来说这个)

但是在说到类之前，我们先来看下基本类型。

<!--More-->
# 1. kotlin的基本类型
在说基本类型之前，我们先提及一个Kotlin的基本定义，也是Kotlin和Java最明显的区别之一：
#### _**在kotlin中，所有的东西都是对象**_

那么熟悉Java的同学可能要说了：不对啊，你不是说Kotlin的基础是Java吗，那么Java的基本类型不是对象啊，那Kotlin的基本类型怎么可能不是对象了。

但是事实就是，**Kotlin的基本类型，都是对象**。

Kotlin的基本类型也被分为了：数字、字符、布尔值、数组与字符串。我们就来逐一谈下。

## 1.1 数字
### 1.1.1 整型
和Java一样，Kotlin也有四种不同大小的整型类型：

|类型|大小（比特）|最小值|最大值|
|:---|:---|:---|:---|
|Byte|8|-128|127
|Short|16|-32768|32767|
|Int|32|-2^31|2^31 - 1|
|Long|64|-2^63|2^63 - 1|


我们上一篇说到了类性推断，所以Kotlin默认给所有未超过`Int`范围的整型值初始化时会推断为`Int`类型。如果初始值超过了`Int`，就会默认为`Long`。如果想直接定义Long的值，就需要在值后面加`L`，like:`123L`。

```kotlin
val one = 1 // Int
val threeBillion = 3000000000 // Long
val oneLong = 1L // Long
val oneByte: Byte = 1
```

### 1.1.2 浮点型
对于浮点型，Kotlin也提供了两种类型：

|类型|大小（比特）|有效数字比特数|指数比特数|十进制位数|
|:---|:---|:---|:---|:---|			
|Float|32|24|8|6-7|
|Double|64|53|11|5-16|

对于小数初始化的变量，Kotlin会默认推断为`Double`类型。而如果需要将一个数值显示指定为`Float`型，就需要在数值后面加`F`，like：`3.0F`。

```kotlin
val pi = 3.14 // Double
val e = 2.7182818284 // Double
val eFloat = 2.7182818284f // Float，实际值为 2.7182817
```

但是需要注意的是，Kotlin的数字没有隐式转换（自动类型转换）。

比如Java会在需要的时候，能自动将低级类型数据转换成高级类型数据：
- 数值型数据的转换：byte→short→int→long→float→double。
- 字符型转换为整型：char→int。

但是Kotlin不会：
```kotlin
fun main() {
    fun printDouble(d: Double) { print(d) }

    val i = 1    
    val d = 1.1
    val f = 1.1f 

    printDouble(d)
//    printDouble(i) // 错误：类型不匹配
//    printDouble(f) // 错误：类型不匹配
}
```

如果需要将数值转换成不同类型，就需要使用显示转换(`as`)。

### 1.1.3 下划线
Kotlin支持在数字中使用下划线`_`，这样能使数字常量更易读：
```kotlin
val oneMillion = 1_000_000
val creditCardNumber = 1234_5678_9012_3456L
val socialSecurityNumber = 999_99_9999L
val hexBytes = 0xFF_EC_DE_5E
val bytes = 0b11010010_01101001_10010100_10010010
```

### 1.1.4 基本类型和包装类
我们在开头说到过，Kotlin的基本类型全是对象。

但是熟悉Java的同学都知道Java中有一个基本类型和包装类。在定义普通的数值的时候，就是使用基本类型int等基本类型；但是如果需要使用泛型——比方说集合的时候——就需要使用Integer(int的包装类)等包装类。

我们可以使用[第一篇](https://blog.csdn.net/a1203991686/article/details/106379486)中看Kotlin的字节码的方法来看下下面的Kotlin代码最终被转成了什么样的代码：
```kotlin
fun main() {
    val a: Int = 3

    println(a)

    val array = ArrayList<Int>()
    array.add(1)
}
```
代码很简单，就是定义了一个基本类型Int型的变量3，然后输出他，然后有定义了一个泛型为Int的ArrayList的集合。

那我们现在来看下他的字节码：
```kotlin
import java.util.ArrayList;
import kotlin.Metadata;

@Metadata(
   mv = {1, 1, 16},
   bv = {1, 0, 3},
   k = 2,
   d1 = {"\u0000\b\n\u0000\n\u0002\u0010\u0002\n\u0000\u001a\u0006\u0010\u0000\u001a\u00020\u0001¨\u0006\u0002"},
   d2 = {"main", "", "KotlinStudy"}
)
public final class MainKt {
   public static final void main() {
      int a = 3;
      boolean var1 = false;
      System.out.println(a);
      ArrayList array = new ArrayList();
      array.add(1);
   }

   // $FF: synthetic method
   public static void main(String[] var0) {
      main();
   }
}
```

主要是看`public static final void main()`这个方法，我们可以看到，Kotlin在编译的时候，自动帮我们将`Int`型的`a`转换成了`int a`。

但是由于泛型消除特性，所以这个时候我们看不懂ArrayList的泛型，但是我们都知道，Java之所以要弄出一套包装类，就是因为他们是类，可以放在泛型中，这不就是我们Kotlin的Int符合的条件吗。

所以可以得出一个结论，Kotlin对于基本类型在编译时，会根据情境自动转换：
- 对于变量、属性、参数和返回类型，Kotlin的Int类型会被编译成Java的基本类型int
- 对于泛型类，Kotlin的基本类型又会被编译成Java对应的包装类

### 1.1.5 显示转换
每个数字类型支持如下的转换:
- `toByte(): Byte`
- `toShort(): Short`
- `toInt(): Int`
- `toLong(): Long`
- `toFloat(): Float`
- `toDouble(): Double`
- `toChar(): Char`

### 1.1.6 运算
#### 四则运算
虽说在Kotlin中属于对象，但是四则运算仍然支持，可以直接使用`+`、`-`、`*`、`/`、`%`。(整数间的除法永远返回的是整数，小数部分则会被舍弃。除非将其中一个整数以浮点数的格式显式的展示出来)

特别可以注意下的是，Kotlin支持运算符重载。
#### 位运算
对于位运算，没有特殊字符来表示，而只可用中缀方式调用具名函数:
```kotlin
val x = (1 shl 2) and 0x000FF000
```

这是完整的位运算列表（只用于`Int`与`Long`）：
- `shl(bits)` – 有符号左移
- `shr(bits)` – 有符号右移
- `ushr(bits)` – 无符号右移
- `and(bits)` – 位与
- `or(bits)` – 位或
- `xor(bits)` – 位异或
- `inv()` – 位非

#### 比较
- 相等性检测：`a == b` 与 `a != b`
- 比较操作符：`a < b`、 `a > b`、 `a <= b`、 `a >= b`
- 区间实例以及区间检测：`a..b`、 `x in a..b`、 `x !in a..b`

## 1.2 字符
字符是使用`Char`来表示，跟上面的`Int`等一样，他也是对象，也是在编译时根据具体情况编译成`char`还是`Character`。

我们可以显式的将字符转为`Int`数字：
```kotlin
fun decimalDigitValue(c: Char): Int {
    if (c !in '0'..'9')
        throw IllegalArgumentException("Out of range")
    return c.toInt() - '0'.toInt() // 显式转换为数字
}
```

## 1.3 布尔
布尔用`Boolean`类型表示，它有两个值：`true`与`false`。

若需要可空引用布尔会被装箱。

内置的布尔运算有：
- `||` – 短路逻辑或
- `&&` – 短路逻辑与
- `!` - 逻辑非

## 1.4 数组
### 1.4.1 数组基本
我们前面说过，Kotlin中只有对象，所以Kotlin的数组也不会像Java那样，直接使用`int[]`这种类型的，而是引入了一个类`Array`。

这个类定义了`get`和`set`函数(Kotlin会将运算符重载转换为`[]`)，以及`size`属性，以及其他一些成员函数。

我们可以使用`arrayOf()`来创建一个数组，并将数组需要存储的值传给他。但是这个方法只能显式的设置数组的元素，并且没法直接设置数组的大小，只能通过方法中的参数也就是具体的数组的元素的个数以及类型推断出这个数组的大小以及类型：
```kotlin
val array1 = arrayOf("1", "2", "3")
val array2 = arrayOf(1, 2, 3)
```

另外也可以通过`Array`构造函数来构造数组：
```kotlin
// 这个代码涉及到lambda，这个以后会再来说
// 一个String类型的数组，容量为5，里面存的内容全是"1"
val array3 = Array(5) {
    "1"
}
// 一个Int类型的数组，容量为5，里面存的内容是[1, 4, 9, 16, 25]
val array4 = Array(5) {
    it * it
}
```

### 1.4.2 原生类型数组
Kotlin也有专门的类来表示原生类型数组：`IntArray`、`CharArray`、`FloatArray`等等。

但是这些类与Array并没有继承关系，但是它们有同样的方法属性集。它们也都有相应的工厂方法:
```kotlin
val x: IntArray = intArrayOf(1, 2, 3)
x[0] = x[1] + x[2]

// 大小为 5、值为 [0, 0, 0, 0, 0] 的整型数组
val arr = IntArray(5)

// 例如：用常量初始化数组中的值
// 大小为 5、值为 [42, 42, 42, 42, 42] 的整型数组
val arr = IntArray(5) { 42 }

// 例如：使用 lambda 表达式初始化数组中的值
// 大小为 5、值为 [0, 1, 2, 3, 4] 的整型数组（值初始化为其索引值）
var arr = IntArray(5) { it * 1 }
```

## 1.5 字符串
字符串应该没有啥好说的了，字符串模板我在[上一篇](https://blog.csdn.net/a1203991686/article/details/106379486)也已经说到了。

Kotlin的字符串也是`String`，也支持转义字符，Java的`String`的那些方法Kotlin都有。

但是Kotlin新增了一个原始字符串：
原始字符串使用三个引号`"""`分界符括起来，内部没有转义并且可以包含换行以及任何其他字符：
```kotlin
val text = """
    for (c in "foo")
        print(c)
"""
```

## 1.6 Kotlin中的void：Unit
Kotlin中的Unit类型就是Java中的void。

用法和Java中的void没有任何区别。

## 1.7 Nothing：这个函数永不返回
> Nothing 是一个 空类型（uninhabited type），也就是说，程序运行时不会出现任何一个 Nothing 类型对象。Nothing 还是其他所有类型的子类型。
> (摘自其它地方)

关于这块，我觉得我也不是很懂，所以还是不在这班门弄斧了，大家可以看下这篇文章：[传送门](https://medium.com/@kazafchen/kotlin%E4%BD%BF%E7%94%A8%E5%BF%83%E5%BE%97-%E4%BA%8C-any%E8%88%87nothing%E9%96%93%E7%9A%84%E5%A5%87%E5%A6%99%E9%97%9C%E4%BF%82-%E4%BB%A5%E5%8F%8A%E8%88%87java%E4%B8%ADvoid%E7%9A%84%E6%84%9B%E6%81%A8%E6%83%85%E4%BB%87-f60477cab752)

# 2. 包与导入
## 2.1 包
和Java一样，Kotlin的包也是在文件最前面用`package`开头：
```kotlin
package kotlinstudy

fun main() {
}
```

## 2.2 导入
和Java一样，Kotlin导入包也是使用`import`关键字。

并且也是可以单独导入一个独特的名字：
```kotlin
import org.example.Message
```
也可以导入这个下面的所有内容：
```kotlin
import org.example.*
```

如果出现的名字发生冲突，比方说导入的最终的名字都是`Message`，只不过属于不同的包下面的，这个时候就可以使用`as`关键字来在本地进行重命名：
```kotlin
import org.example.Message // Message 可访问
import org.test.Message as testMessage // testMessage 代表“org.test.Message”
```