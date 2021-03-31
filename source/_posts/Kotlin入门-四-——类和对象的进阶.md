---
title: Kotlin入门(四)——类和对象的进阶
tags:
  - Kotlin
  - 语言基础
categories: Kotlin
cover: >-
  https://cdn.littlecorgi.top/mweb/2020-05-27/kotlin%E5%85%A5%E9%97%A8-%E4%B8%80-%E2%80%94%E2%80%94%E5%A4%B4%E5%9B%BE.png
abbrlink: 6f3ca960
date: 2020-06-23 20:10:18
updated: 2020-06-23 20:10:18
description:
---
# Kotlin入门(四)——类和对象的进阶
> 本章内容包括：
> - 可空性
> - 数据类
> - 密封类
> - 枚举类

# 0. 前言
在上一篇[《Kotlin入门(三)——类、对象、接口》](https://blog.csdn.net/a1203991686/article/details/106648619)

我们只聊到了Kotlin中基本类的写法以及继承，但是我们说过，Kotlin的本质就是解决Java的繁琐，如果Kotlin只有这么简单的话怎么还能被称为Kotlin。

<!--More-->

首先我们思考在Java中的几个场景：
- 在方法中每次都得对传进来的对象进行判空，并且很多时候都会忘记判空或者不知道别人在调用你这个方法的时候到底会不会给空，然后就导致程序空指针异常了

```Java
void nullTest(Obj obj) {
    if (obj == null) {
        return
    }
    ...
}
```

- 每次在Java中写JavaBean的时候，一旦数据变多，就得写一大堆的getter、setter、toString、equals等等方法

```Java
public class Person {
    private String firstName;
    private String lastName;
    private String telephone;
    private String address;

    public Person(String firstName, String lastName, String telephone, String address) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.telephone = telephone;
        this.address = address;
    }

    public String getFirstName() {
        return firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

    public String getLastName() {
        return lastName;
    }

    public void setLastName(String lastName) {
        this.lastName = lastName;
    }

    public String getTelephone() {
        return telephone;
    }

    public void setTelephone(String telephone) {
        this.telephone = telephone;
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }
}
```

但是如果在Kotlin中，上面两个问题还刚好可以通过可空性和数据类去解决。

那可能有的同学会问，你上面不还有一个密封类吗，那他有啥方便之处呢？这个我们先卖个关子，我们放到后面来谈这个。

# 1. 可空性
在Kotlin中，可空性是Kotlin和Java最显著的区别之一，他能非常高效的帮助我们开发者去避免`NullPointerException`。

Kotlin对于可空性的处理就是把这个运行时的错误转成了编译期的错误，这样我们在编译时就能发现很多存在的错误，从而减少运行时抛出异常的可能性。

## 1.1 可空类型
首先，Kotlin支持对可空类型的显式。这句话可能读起来觉得莫名其妙，其实简单点说就是：这是一种可以直接指出你的程序中哪些变量和属性允许为`null`的方式。

我们还是从同样的功能的代码的Java版入手。

我们先来看下最常见的一种Java的代码：
```Java
int strLen(String s) {
    return s.length();
}
```
恐怕这个代码一写出来，很多哪怕是新手的Java程序员都能指出他的问题：如果传入的`s`是`null`，这个程序就崩溃了。

那我们现在来试着用Kotlin去重写这个函数，但是在重写之前，我们首先得考虑我们调用这个函数的时候，传入的实参，是否可以为null。

如果我们不希望传入的s为null，我们就可以直接使用最基本的Kotlin函数的写法：
```Kotlin
fun strLen(s: String): Int {
    return s.length
}
```

这个时候如果我们在调用strLen的地方给他传入一个null进去，我们甚至都不用编译这段代码，IDEA就会自动帮我们把这块代码给标注出来(报错)，不能传入一个null进去:
![Kotlin入门-四-1](https://cdn.littlecorgi.top/mweb/2020-06-23/Kotlin%E5%85%A5%E9%97%A8-%E5%9B%9B-1.png)


在这个函数中，由于函数的形参被声明为了String(请注意：这个String只是String)，所以Kotlin就会认为你传入的这个String类型的参数必须为String的对象，而不可以为null。

但是如果我们想让它可以传入null呢？这个时候我们就需要显式的在类型名称后面加上问号了：
```kotlin
fun strLen(s: String?): Int {
    return s.length
}
```

这个时候我们就可以直接像上图中的那种方式去调用这个函数。

问号可以加在任何类型的后面，表示这个类型的变量可以为null。

但是其实你像我上面说的那样改了之后，其实IDEA也还是会报错：
![Kotlin入门-四-2](https://cdn.littlecorgi.top/mweb/2020-06-23/Kotlin%E5%85%A5%E9%97%A8-%E5%9B%9B-2.png)

这是因为如果你让一个变量可空了之后，你就没办法直接对他进行操作，也不能把它赋值给非空类型的变量，也不能把可空类型的值传给拥有非空类型参数的函数。

但是Kotlin和Java一样，你只要在外面对s判断不等于null了之后，就可以在if的函数体中对他直接进行操作了：
```Kotlin
fun strLen(s: String?) = if (s != null) s.length else 0
```

但是这个时候，你一定会满头问号，因为你一定会吐槽，这个代码和Java有啥区别，Java甚至都不需要加问号(`?`)。

其实我讲了这么多，只是为了引出Kotlin对于空的一大堆好用的操作，接下来，我们就先来说一下安全调用运算符。

## 1.2 安全调用运算符：`?.`
回归到刚才那个问题，Kotlin是如何解决`if (s != null)`的。

其实要解决那个`if(s != null)`很简单，就用`?.`就行了：
```kotlin
fun strLen(s: String?) = s?.length
```

其实`?.`他就是把null检查和执行代码合成了一个操作：
![Kotlin入门-四-3](https://cdn.littlecorgi.top/mweb/2020-06-23/Kotlin%E5%85%A5%E9%97%A8-%E5%9B%9B-3.jpg)

值得注意的是，图里面后面的两个表达式其实是返回值，也就是说当`s`为`null`的时候，`s?.length`返回值其实是`null`。

安全调用不止可以调用方法，也可以用来访问属性。

但是这个时候你可能会说，这不对啊，Java的代码的作用是当`s`为`null`的时候返回`0`啊，但是你上面的那个Kotlin代码当`s`为`null`的时候，却返回了`null`。

我只能说你图样图森破，其实这个套路和刚才过度到`?.`的时候一样，我们可以继续用Kotlin给定的特殊语句(也就是Elvis运算符`?:`)去解决这个问题。

## 1.3 Elvis运算符：`?:`
```kotlin
fun strLen(s: String?) = s?.length ?: 0
```

同样，也会有一个流程图去让你更容易理解这个代码的流程，只不过这个时候我们需要将`s?.length`看做是一个整体：
![Kotlin入门-四-4](https://cdn.littlecorgi.top/mweb/2020-06-23/Kotlin%E5%85%A5%E9%97%A8-%E5%9B%9B-4.jpg)

我们可以简化`?:`的用法，其实就是`a ?: b`，也就是说，当`a`的值不为`null`的时候，就返回`a`，但是当`a`的值为`null`的时候，就返回`b`。

也就是说，当`s?.length`的值不为`null`的时候，就返回`s.length`(因为此处`s?.length`的值不为`null`，所以就相当于`s.length`)，但是如果当`s?.length`的值为`null`的时候，就返回`0`。

并且对于`?:`，我们其实还有一个非常方便的操作，就是当我们需要返回null或者需要抛异常的时候：
```kotlin
fun foo(node: Node): String? {
    val parent = node.getParent() ?: return null
    val name = node.getName() ?: throw IllegalArgumentException("name expected")
}
```

## 1.4 安全转换：`as?`
我们在之前说到过，Kotlin主要通过`as`运算符来进行类型转换。

但是和Java的类型转换一样，如果被转换的值不是你试图转换的类型时，就会抛出`ClassCastException`异常。虽说可以结合`is`检查来确定这个值拥有合适的类型，但是Kotlin一定会有更加优雅的方式。

`as?`就可以将值转换成指定的类型，如果不是合适的类型就返回`null`：
![Kotlin入门-四-5](https://cdn.littlecorgi.top/mweb/2020-06-23/Kotlin%E5%85%A5%E9%97%A8-%E5%9B%9B-5.jpg)

一种常见的模式就是可以用于重写`equals()`方法：
```kotlin
class Person(val name: String) {
    override fun equals(other: Any?): Boolean {
        val otherPerson = other as? Person ?: return false
        return otherPerson.name == this.name
    }
}
```

## 1.5 非空断言：`!!`
非空断言是Kotlin提供的最简单粗暴的一个处理可空类型的工具。他的作用就像他的样子一样，表示我就让这个类型转换成非空类型。
![Kotlin入门-四-6](https://cdn.littlecorgi.top/mweb/2020-06-23/Kotlin%E5%85%A5%E9%97%A8-%E5%9B%9B-6.jpg)

这种和Java一样，所以也就没啥好说的。

## 1.6 `let`函数
其实这个`let`函数属于后面标准函数的内容，但是由于标准函数中的每一个函数都是服务于具体某个功能的，所以就放在功能这块来说这个函数。

`let`函数天生就是为`?.`服务的。

我们回到上面那个例子，如果我们想在返回`s`的`length`之前先让他删除调最前面或者最后面的空格：
```kotlin
fun strLen(s: String?) = s?.let { str ->
    {
        str.trim()
        str.length
    }
}
```

一般情况，Kotlin的lambda表达式都会将语句的最后一句作为return。

但是在Kotlin的lambda表达式中，我们可以用自动生成的名字`it`：
```kotlin
fun strLen(s: String?) = s?.let {
    it.trim()
    it.length
}
```

至于为啥是`it`，这个是Kotlin的lambda表达式的特殊字符，就类似于setter和getter中的`field`字段一样的。

## 1.7 可空类型的集合
Kotlin也可以创建值为null的集合，比如：
```kotlin
val nullsArray = arrayOfNulls<Int>(1) // 元素类型为Int，容量为1的初始值全为null的数组
val nullsArrayList = ArrayList<Int?>() // 泛型为Int?的ArrayList
```

如果你有一个可空类型元素的集合，并且想要过滤非空元素，你可以使用`filterNotNull`来实现：
```kotlin
val nullableList: List<Int?> = listOf(1, 2, null, 4)
val intList: List<Int> = nullableList.filterNotNull()
```

# 2. 数据类
我们在前言举的例子说到过，Kotlin对于之前的饱受诟病的JavaBean做了一个非常好的处理。

对于这类只需要保存数据的容器，往往你都需要去重写他们的一些方法，比如`toString()`、`equals()`、`hashCode()`等方法，这些方法的写法又特别的机械，像Idea和eclipse都提供了自动生成的方法。

但是在Kotlin里面，你就不必再去手动去操作这些方法了。在Kotlin中，这些类叫做数据类，并且只需要在`class`的前面添加`data`修饰符：
```kotlin
data class Person(val name: String, val age: Int)
```

在声明为数据类后，Kotlin就能自动的帮你重写以下方法：
- `hashCode()`：这个方法不用多介绍了，和Java中一样
- `equals()`：这个方法也不用多介绍了，和Java中一样
- `toString()`：这个方法仍然不用多介绍了，和Java中一样
- `componentN()`：这个函数是用来[解构声明](https://www.kotlincn.net/docs/reference/multi-declarations.html)的，见下文
- `copy()`：见下文

我们可以看一下上面那个类的字节码([看字节码的方法在此](https://blog.csdn.net/a1203991686/article/details/106379486#t1))：
```Java
public final class Person {
   @NotNull
   private final String name;
   private final int age;

   @NotNull
   public final String getName() {
      return this.name;
   }

   public final int getAge() {
      return this.age;
   }

   public Person(@NotNull String name, int age) {
      Intrinsics.checkParameterIsNotNull(name, "name");
      super();
      this.name = name;
      this.age = age;
   }

   @NotNull
   public final String component1() {
      return this.name;
   }

   public final int component2() {
      return this.age;
   }

   @NotNull
   public final Person copy(@NotNull String name, int age) {
      Intrinsics.checkParameterIsNotNull(name, "name");
      return new Person(name, age);
   }

   // $FF: synthetic method
   public static Person copy$default(Person var0, String var1, int var2, int var3, Object var4) {
      if ((var3 & 1) != 0) {
         var1 = var0.name;
      }

      if ((var3 & 2) != 0) {
         var2 = var0.age;
      }

      return var0.copy(var1, var2);
   }

   @NotNull
   public String toString() {
      return "Person(name=" + this.name + ", age=" + this.age + ")";
   }

   public int hashCode() {
      String var10000 = this.name;
      return (var10000 != null ? var10000.hashCode() : 0) * 31 + this.age;
   }

   public boolean equals(@Nullable Object var1) {
      if (this != var1) {
         if (var1 instanceof Person) {
            Person var2 = (Person)var1;
            if (Intrinsics.areEqual(this.name, var2.name) && this.age == var2.age) {
               return true;
            }
         }

         return false;
      } else {
         return true;
      }
   }
}
```

需要注意的是，如果上面的方法中任何一个已经有了显式的实现，那么数据类在生成的时候，就不会再去重新生成这个函数，而是会直接使用显式的这个函数。

## 2.1 无参构造
如果数据类需要一个无参构造，那么就需要对每个属性都指定默认值：
```kotlin
data class Person(val name: String = "", val age: Int = 0)
```
这样就会有一个无参构造。

## 2.2 在类中声明的属性
数据类也可以在类的里面去声明属性：
```kotlin
data class Person(val name: String, val age: Int) {
    var address: String = ""
}
```

但是需要注意的是，这样的话，数据类帮你生成的那些方法(`equals()`、`toString()`等)，都不会带上`address`这个属性。除非你自己显式重写对应的方法。

## 2.3 copy()
我们上面说到，数据类会自动帮我们生成`copy()`方法，那么这个`copy`到底是干嘛的呢？

其实说句实话，我觉得这个方法的话，也有点鸡肋，也就是那种食之无味，但是又弃之可惜的东西(这么说存在一定的绝对)，但是也无所谓，能多点功能，能让我们少写点代码当然是好的了。

好了，说回来，这个方法到底是干嘛的呢？其实单看名字就能看出来，肯定与复制有关，但是他到底是复制啥呢。

其实在有些时候，如果我们需要生成这个类的另外一个对象，但是很多属性都和这个类的原本的对象都是一样的，我们只需要修改他其中的某一个属性，那么这个时候`copy()`就很有用了：
```kotlin
val jack = Person("Jack", 1)
val oldJack = jack.copy(age = 28)
```
我们就可以通过去调用`jack`的`copy()`方法，在参数中指定我们需要修改的属性，这样就可以返回一个除了指定的属性外，其它属性都和原对象一样的一个新对象。

其实这个方法也还是非常有用的，但是我为什么又在上面说食之无味弃之可惜？因为说实话，我觉得这个东西，我们日常使用的着实少，可以说少之又少，但是单看概念又挺有用，并且如果真的让我们自己去重写这个方法，虽然说在技术层面，实现这个方法着实简单，但是一旦我们属性多了起来之后，重写起来还真的得花点功夫。


## 2.4 解构声明
对于Java来说，解构声明是一个新的东西，而这个，说实话，在我看来和上面那个`copy()`差不多，也是一个食之无味弃之可惜的东西。但是，这是相对于数据类来说的。我为啥这么说呢，继续往下看就知道了。

首先我们直接上代码，看下解构声明到底是什么：
```kotlin
val jack = Person("Jack", 1)
val (name, age) = jack
println("$name's age is $age")
```
其中第二行那就是解构声明，也就是说，我们可以将某个对象的所有属性给单独拎出来。

看到这，是不是也会和我一样产生一个感觉，这个东西着实意义不大，我们想去获取某个属性的话，直接调用这个类的属性的`getter`不就行了吗。

但是我刚刚说了，我觉得这个东西很鸡肋，是针对于数据类来说的，下面我给你看个代码你就会觉得这个东西非常有用了：
```kotlin
val map = HashMap<String, Person>()
for ((name, person) in map) {
    println("$name to (${person.name}, ${person.age})")
}
```

# 3. 密封类
其实密封类很简单，没啥特别的东西，

我们先来看一个例子：
```kotlin
interface Expr
class Num(val value: Int) : Expr
class Sum(val left: Expr, val right: Expr) : Expr

fun eval(e: Expr): Int =
    when (e) {
        is Num -> e.value
        is Sum -> eval(e.left) + eval(e.right)
        else ->
            throw IllegalArgumentException("Unknown expression")
    }
```
我们定义了一个父类(接口)`Expr`，以及他的两个子类：代表数字的`Num`和代表和的`Sum`，然后我们在`when`中去处理所有的操作。

目前来说这样很方便，但是其实有一点很多余，就是我们完全没有必要去写`when`中的`else`分支，因为他完全不可能是其他类型。并且如果我们新增了一个`Expr`的子类，万一忘记了在`when`添加对应的分支，那么程序就存在bug。

这个时候，我们的密封类就派上用场了。在Kotlin官方文档，对密封类的定义很简单：“密封类用来表示受限的类继承结构：当一个值为有限几种的类型、而不能有任何其他类型时。”。也就是说上面这种情况，我们非常明确Expr不可能会再有其它的子类的，所以就没有必要再去写else分支。

而实现密封类也很简单，在class前面加上sealed。接下来我们使用密封类改写一下上面那个例子：
```kotlin
sealed class Expr
class Num(val value: Int) : Expr()
class Sum(val left: Expr, val right: Expr) : Expr()

fun eval(e: Expr): Int =
    when (e) {
        is Num -> e.value
        is Sum -> eval(e.left) + eval(e.right)
        // 不再需要else，
        // 并且如果你写了else，编译器会提示你'when' is exhaustive so 'else' is redundant here
        // else ->
        //     throw IllegalArgumentException("Unknown expression")
    }
```

# 4. 枚举类
对于枚举类，Kotlin和Java的用法没啥区别。

```kotlin
enum class Direction {
    NORTH, SOUTH, WEST, EAST
}
```

## 4.1 初始化
```kotlin
enum class Color(val rgb: Int) {
        RED(0xFF0000),
        GREEN(0x00FF00),
        BLUE(0x0000FF)
}
```

# 5. 嵌套类
## 5.1 嵌套类
在Java中，我们很多时候都会在一个类里面再去声明一个类，这种类就叫做内部类。

而Kotlin把这种类叫做嵌套类。简单来说就是把这个类嵌套在了另一个类里面：
```kotlin
class Person {
    private var age: Int = 0
    private var name: Name = Name("William", "Shakespeare")

    class Name(val firstName: String, val lastName: String)
}
```
但是，你如果把上面这段代码翻译成字节码的话，你会发现其实这个嵌套类，他是`static`的，也就是说，他不持有外部类的引用，并且你在嵌套类中，没法直接使用外部类的属性或者方法。

所以，我们在Android中写`Handler`的时候，我们就直接写一个`Handler`的嵌套类就行了。

## 5.2 内部类
但是如果你就是想写一个普通的内部类，就是一个没有`static`修饰的内部类，那么Kotlin就提供了一个`inner`关键字：
```kotlin
class Person {
    private var age: Int = 0
    private var name: Name = Name("William", "Shakespeare")

    inner class Name(val firstName: String, val lastName: String) {
        fun print() {
            println("$firstName $lastName's age is $age")
        }
    }
}
```
这样，`Name`这个类，就是一个非`static`的内部类了，并且他会持有外部类`Person`的引用，所以我们可以直接访问外部类的属性，上面那个`Name`类中的`print()`方法才可以去访问`Person`的属性`age`。

## 5.3 匿名内部类
在Java中我们经常会使用到匿名内部类，比方说我们写Callback回调的时候：
```java
call.enqueue(new Callback<Translation>() {
    //请求成功时回调
    @Override
    public void onResponse(Call<Translation> call, Response<Translation> response) {
        // 对返回数据进行处理
        response.body().show();
    }

    //请求失败时候的回调
    @Override
    public void onFailure(Call<Translation> call, Throwable throwable) {
        System.out.println("连接失败");
    }
});
```
这个时候，enqueue传入的就是一个继承自Callback类的匿名内部类。

而在Kotlin中，也是差不多的形式，只不过我们需要借用下`object`关键字：
```kotlin
call.enqueue(object : Callback<T> {
    override fun onFailure(call: Call<T>, t: Throwable) {
        println("连接失败")
    }

    override fun onResponse(call: Call<T>, response: Response<T>) {
        response?.body().show();
    }
})
```