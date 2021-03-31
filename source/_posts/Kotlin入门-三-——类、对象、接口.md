---
title: Kotlin入门(三)——类、对象、接口
tags:
  - Kotlin
  - 语言基础
categories: Kotlin
cover: >-
  https://cdn.littlecorgi.top/mweb/2020-05-27/kotlin%E5%85%A5%E9%97%A8-%E4%B8%80-%E2%80%94%E2%80%94%E5%A4%B4%E5%9B%BE.png
abbrlink: 756a6103
date: 2020-06-10 15:41:23
updated: 2020-06-10 15:41:23
---
> 本章内容包括：
> - 类的基本要素
> - 类的继承结构
> - 修饰符
> - 接口

# 0. 前言
在[上一篇](https://blog.csdn.net/a1203991686/article/details/106421100)的末尾，我们提到了Kotlin的包和导入。

原本我是准备把这篇的内容也放在上一篇的，但是后来一想，这张的内容会很有点多，放进去的话可能会导致上一篇太大了，所以就单独分成一篇了。

在说类之前，我们先来看下一个类的Java版和Kotlin版的对比，这个会一下子就让你对Kotlin感兴趣。

<!--More-->

我们现在有一个需求，需要定义一个JavaBean类Person，这个类中包含这个人的姓名、电话号码以及地址。

我们先来看下Java的实现：
```java
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

这是一个很基本的Java类，我们定义了四个private属性，然后给定了一个构造函数，然后对每个属性都给了get和set方法。

相信大家学Java一定都写过这个类。但是我们想想，就写一个功能这么简单的类，Java却需要我们写这么多内容，有的同学会说：Idea和Eclipse都不是提供了自动生成代码的工具吗。但是如果你看了Kotlin的实现，一定会觉得连自动生成工具都麻烦：

```kotlin
class Person(
    val firstName: String,
    val lastName: String,
    val telephone: String,
    val address: String
)
```

对，你没有看错，Kotlin的类就是这么的简单：
- 由于Kotlin的属性默认的修饰符就是`public`。但是由于我们这个方法设置成了`val`，所以除了构造方法外，没法对这个属性的值进行更改。但是从Kotlin编译后会自动将`val`的属性转为`private final String firstName;`，`var`的属性转为`private String firstName;`

- 由于Kotlin会自动为属性生成`get`和`set`方法，所以没必要去显式的写`get`和`set`方法，除非你需要自定义的`get`和`set`方法。但是由于这个类的属性都是`val`，所以只会生成`get`方法。
- Kotlin的默认构造方法是直接写在类名后面的

接下来我们就把这个代码进行分解，逐步来讲解Kotlin的类。

# 1. 类与继承
## 1.1 类
与Java类似，Kotlin也是使用`class`关键字来表示类。
```kotlin
class Person() {}
```

类声明由类名、类头（指定其类型参数、主构造函数等）以及由花括号包围的类体构成。类头和类体都是可选的，一个类如果没有类体，可以省略花括号：
```kotlin
class Person()
```

## 1.2 构造函数
### 1.2.1 主构造函数
Kotlin的一个类可以有一个主构造函数以及一个或者多个次构造函数。主构造函数是类头的一部分，跟在类名之后：
```kotlin
class Person constructor(val name: String){}
```
如果主构造函数没有任何注解或者可见性修饰符，可以省略这个`constructor`关键字。
```kotlin
class Person(val name: String){}
```

主构造方法主要有两种目的：表明构造方法的参数，以及定义使用这些参数初始化的属性。

但是主构造方法不允许直接有代码块，所以如果需要在主构造方法中添加初始化代码，可以放到init关键字的代码块中：
```kotlin
class Person(val _name: String) {
    val name: String
    init {
        name = _name
        println(name)
    }
}
```

但是同时，这个例子中，_name赋值给name，这个语句可以放在name的定义中去，所以可以改成：
```kotlin
class Person(val _name: String) {
    val name: =  _name
    init {
        println(name)
    }
}
```

但是，如果主构造方法需要添加注解或者修饰符的话，这个`constructor`是不能省略的：
```kotlin
class Person private constructor(val name: String) {
}
```

### 1.2.2 次构造函数
类也可以单纯的声明次构造方法而不声明主构造方法：
```kotlin
class Person {
    val name: String

    constructor(_name: String) {
        name = _name
        println(name)
    }
}
```

如果类有一个主构造函数，每个次构造函数需要委托给主构造函数， 可以直接委托或者通过别的次构造函数间接委托。委托到同一个类的另一个构造函数用 this 关键字即可：
```kotlin
class Person(val name: String) {
    var children: MutableList<Person> = mutableListOf<>()
    constructor(name: String, parent: Person) : this(name) {
        parent.children.add(this)
    }
}
```

值的注意下的是，初始化语句(init)实际上会成为主构造方法的一部分。委托给主构造函数会作为次构造函数的第一条语句，因此所有初始化块与属性初始化器中的代码都会在次构造函数体之前执行。即使该类没有主构造函数，这种委托仍会隐式发生，并且仍会执行初始化块。
简单点来说就是，不管你有没有主构造方法，只要你有次构造方法，并且有init语句，他都会在执行次构造方法的函数体内的代码之前，先去执行init语句：
```kotlin
fun main() {
    val person = Person("1")
}

class Person {
    val name: String

    constructor(_name: String) {
        name = _name
        println(name)
    }

    init {
        println("init: 1")
    }
}

/* 
输出结果为
init: 1
1
 */
```

## 1.3 类的实例
回忆一下[上一篇](https://blog.csdn.net/a1203991686/article/details/106421100)讲基本类型：
```kotlin
val one = 1 // Int
val threeBillion = 3000000000 // Long
```

- 首先我们说过Kotlin和Java不一样，所有的东西都是对象，包括基本类型。所以说我们创建的Kotlin的`Int`型参数，实际上就是`new`了一个`Int`这个类的对象
- 其次Kotlin创建对象是通过`val`和`var`关键字的
- 最后就是一点大家应该都发现了，Kotlin是没有`new`这个关键字的

所以从上面我们可以推出如果在Kotlin中创建一个对象：
- 首先，我们需要根据对象需要的场景，选择到底是val的对象还是var的对象；
- 然后写对象名；
- 紧跟着对象名的就是对象的类型，但是由于Kotlin有类型推断，所以此处这个显式声明类型可以省略，因为他可以通过等号右边给定的内容自动推断出该对象的类型（此处暂时先不考虑lateinit或者其他的操作）
- 之后就可以写等号`=`
- 等号右边就是我们需要赋值给这个对象的初始化语句了，但是Kotlin没有`new`关键字，所以是直接写，不需要加`new`

```kotlin
val person1: Person = Person("1") // Kotlin没有 new 关键字

val person2 = Person("1") // 由于等号右边已经给出了具体的内容，所以可以省略掉显式的指定类型
```

当然也有特殊情况：
- 使用`lateinit`关键字

```kotlin
lateinit var person3: Person
```
在这种情况下必须显式的指定变量类型，因为使用了`lateinit`关键字，可以延迟初始化，但是从现在开始，直到初始化，期间如果使用了这个变量，**运行后**就会报错`lateinit property person3 has not been initialized`

- 在方法或者类中定义变量

```kotlin
fun main() {
    val person4: Person
    println(person4) // 此时IDE就会报红，Variable 'person4' must be initialized
    person4 = Person("4")
    println(person4)
}
```

在方法中或者类中还可以这样，先不初始化，先定义变量，但是此时必须显式指出其类型，并且在初始化之前都不可使用变量，如果使用了，**在编译前也就是还在编辑时**，IDE就会报红，`Variable 'person4' must be initialized`。但是一旦初始化之后就可以正常使用。

## 1.4 继承
### 1.4.1 `Object`和`Any`、`extends`和`:`
我们都知道，Java中存在着一个基类`Object`，所有的对象都会继承自这个类，哪怕你自己创建的对象没有指明具体继承自哪个类，但是Java会让他继承自Object类。而这个类里面也有一些每个类必有的方法如`getClass()`、`hashCode()`、`equals()`、`toString()`等一系列方法。

同样的，Kotlin也有这样的基类，只不过叫做`Any`。但是不同的是Kotlin的Any只有三个方法：`hashCode()`、`equals()`和`toString()`。

而在Java中，想要继承某一个类的话，就需要在这个类的后面用`extends`关键字 + 超类名的方法去指明这个类继承自那个类：
```java
class Staff extends Person{
}
```

而在Kotlin中，就没有`extends`这个关键字了，取而代之的是我们的老朋友`:`：
```kotlin
open class Person(val name: String)
class Staff(name: String) : Person(name) 
```

这个就代表了`Staff`类继承自`Person`类。同时基类(`Person`)必须得被`open`修饰符修饰，因为Kotlin默认所有的类都是`final`的，所以不能被继承，所以就需要`open`修饰符修饰它。

并且如果派生类有一个主构造函数，其基类可以（并且必须） 用派生类主构造函数的参数就地初始化。

如果派生类没有主构造函数，那么每个次构造函数必须使用`super`关键字初始化其基类型，或委托给另一个构造函数做到这一点。 注意，在这种情况下，不同的次构造函数可以调用基类型的不同的构造函数：

```kotlin
// 代码非常不规范，仅作为例子参考
open class Person {
    val name: String
    var address: String = ""
    constructor(_name: String) {
        name = _name
    }
    constructor(_name: String, _address: String) {
        name = _name
        address = _address
    }
}

class Staff : Person {
    constructor(name: String) : super(name)
    constructor(name: String, address: String) : super(name, address)
}
```

### 1.4.2 `override`

#### 覆盖方法
和Java一样，Kotlin也是通过`override`关键字来标明覆盖，只不过不同的是Java是`@override注解`而Kotlin是`override`修饰符。
```kotlin
open class Person(val name: String) {
    open fun getName() {
        println("这是$this, name: $name")
    }
}

class Staff(name: String) : Person(name) {
    override fun getName() {
        println("这是$this, name: $name")
    }
}
```

我们可以看到`Staff`继承自`Person`并重写了`getName()`方法。
此时必须在`Staff`重写的`getName()`方法前加上`override`修饰符，否则编译器会报错。

同时，与继承类时一样，Kotlin默认方法也是`final`的，如果想让这个方法被重写，就需要加上`open`关键字。但是重写后的方法，也就是有`override`修饰的方法，默认是开放的，但是如果你想让他不再被重写，就需要手动添加`final`修饰符：
```kotlin
class Staff(name: String) : Person(name) {
    final override fun getName() {
        println("这是$this, name: $name")
    }
}
```

#### 覆盖属性
这个就是Kotlin有但是Java没有的了。和覆盖方法一样，也就是在需要覆盖的属性前面加上`override`：
```kotlin
open class Person {
    open val name = "123"
}

class Staff : Person() {
    override val name = "2"
}
```

同时你可以使用`var`属性去覆盖一个`val`的属性，但是反过来就不行了。因为`var`默认会有`get()`和`set()`方法，而`val`只有`get()`方法，如果用`val`去覆盖`var`，那么`var`的`get()`方法会无法处理。

### 1.4.3 初始化顺序
在构造派生类的新实例的过程中，第一步完成其基类的初始化（在之前只有对基类构造函数参数的求值），因此发生在派生类的初始化逻辑运行之前。
```kotlin
open class Base(val name: String) {

    init { println("Initializing Base") }

    open val size: Int = 
        name.length.also { println("Initializing size in Base: $it") }
}

class Derived(
    name: String,
    val lastName: String
) : Base(name.capitalize().also { println("Argument for Base: $it") }) {

    init { println("Initializing Derived") }

    override val size: Int =
        (super.size + lastName.length).also { println("Initializing size in Derived: $it") }
}

fun main() {
    println("Constructing Derived(\"hello\", \"world\")")
    val d = Derived("hello", "world")
}
```
运行结果是：
```text
Constructing Derived("hello", "world")
Argument for Base: Hello
Initializing Base
Initializing size in Base: 5
Initializing Derived
Initializing size in Derived: 10
```

### 1.4.5 调用超类实现
子类可以通过`super`关键字访问超类中的内容：
```kotlin
open class Rectangle {
    open fun draw() { println("Drawing a rectangle") }
    val borderColor: String get() = "black"
}

class FilledRectangle : Rectangle() {
    override fun draw() {
        super.draw()
        println("Filling the rectangle")
    }

    val fillColor: String get() = super.borderColor
}
```

但是如果在一个内部类中访问外部类的超类的内容，可以通过外部类名限定的super关键字`super@Outer`来实现：
```kotlin
class FilledRectangle: Rectangle() {
    fun draw() { /* …… */ }
    val borderColor: String get() = "black"
    
    inner class Filler {
        fun fill() { /* …… */ }
        fun drawAndFill() {
            super@FilledRectangle.draw() // 调用 Rectangle 的 draw() 实现
            fill()
            println("Drawn a filled rectangle with color ${super@FilledRectangle.borderColor}") // 使用 Rectangle 所实现的 borderColor 的 get()
        }
    }
}
```

### 1.4.6 覆盖规则
在 Kotlin 中，实现继承由下述规则规定：如果一个类从它的直接超类继承相同成员的多个实现， 它必须覆盖这个成员并提供其自己的实现（也许用继承来的其中之一）。 为了表示采用从哪个超类型继承的实现，我们使用由尖括号中超类型名限定的`super`，如`super<Base>`：
```kotlin
open class Rectangle {
    open fun draw() { /* …… */ }
}

interface Polygon {
    fun draw() { /* …… */ } // 接口成员默认就是“open”的
}

class Square() : Rectangle(), Polygon {
    // 编译器要求覆盖 draw()：
    override fun draw() {
        super<Rectangle>.draw() // 调用 Rectangle.draw()
        super<Polygon>.draw() // 调用 Polygon.draw()
    }
}
```

## 1.5 抽象类
Kotlin中的抽象类用abstract关键字。抽象成员可以在本类中不用实现。

同时不用说的就是，在抽象类中不需要使用open标注。

```kotlin
open class Polygon {
    open fun draw() {}
}

abstract class Rectangle : Polygon() {
    abstract override fun draw()
}
```

# 2. 属性和字段
属性的定义我们已经在[前面](https://blog.csdn.net/a1203991686/article/details/106379486)说到过了，主要是`val`和`var`两个关键字，而现在首先要说的，就是`Getter`和`Setter`。

## 2.1 `Getter`和`Setter`
声明一个属性完整的语法是：
```kotlin
var <propertyName>[: <PropertyType>] [= <property_initializer>]
    [<getter>]
    [<setter>]
```
其中，`property_initializer`、`getter`、`setter`都是可选的，并且如果类型可以从`property_initializer`推断出来，则也是可选的。

而对于不管是`val`还是`var`，如果你访问这个属性的时候就直接返回这个属性的值的话，`getter`是可以省略的，而如果你想返回的时候后做些操作的话，就可以自定义`getter`（比方说我们现在有一个`Person`类，里面有`name`、`age`以及`isAdult`三个属性，其中`isAdult`我们需要去设置他的`get`方法，当`age`大于等于18的时候就返回`true`，否则返回`false`）：
```kotlin
class Person(_name: String, _age: Int) {
    val name: String = _name
    val age: Int = _age
    val isAdult: Boolean
        get() = age >= 18
}
```

而同时我们需要在设置这个人的age的时候，做一个判断，如果输入的值小于0的话，就抛异常，否则才更改age的值。这个时候我们就需要自定义set方法了：
```kotlin
class Person(_name: String, _age: Int) {
    val name: String = _name
    var age: Int = _age
        set(value) {
            if (value <= 0) {
                throw Exception("年龄必须大于0")
            } else {
                field = value
            }
        }
    val isAdult: Boolean
        get() = age >= 18
}

fun main() {
    try {
        val person = Person("314", 18)
        person.age = 0
    } catch (e: Exception) {
        println(e.message)
    }
}
```
运行结果就是`年龄必须大于0`。但是这块有个小要点，就是属性的set方法在对象初始化的时候是不起作用的，也就是说，如果我给上面这个Person类创建对象的时候，给age传入0或者负数的话：
```kotlin
fun main() {
    try {
        val person = Person("314", 0)
        println(person.age)
    } catch (e: Exception) {
        println(e.message)
    }
}
```
运行结果没有任何异常，输出`0`。

不知道大家注意到了没有，我们给`setter`传入的是`value`，而用`field`承接了传入的`value`。

其实这个`value`是我们自定义的，也就是说`set()`这个括号里面的名字你可以随便写，只要符合Kotlin命名规范。
但是这个`field`是不可变的，这个`field`相当于是`this.属性`，也就相当于是set的这个值本身，也就是说，如果你想在`setter`中改变这个属性的值的话，就必须得把最终的值传给`field`，`field`就相当于是这个属性，而`setter`中`this.属性`是没有意义的，你写了的话，IDEA反而会提示你让你改成`field`。
![1](https://cdn.littlecorgi.top/mweb/2020-06-09/1.png)

## 2.2 编译器常量
如果只读属性的值在编译器是已知的，就可以使用`const`去修饰将其标记为编译器常量，这种属性需要满足下列要求：
- 位于顶层或者是`object`声明 或`companion object`的一个成员
- 以`String`或原生类型值初始化
- 没有自定义`getter`

## 2.3 延迟初始化属性与变量
一般，属性声明为非空类型就必须得在构造函数中去初始化。但是这样也会不是很方便，例如像Android中的view的对象(TextView、Button等view的对象，需要被`findViewById`)。在这种情况下，我们没法去提供一个构造器去让其初始化，这个时候就可以使用`lateinit`修饰符：
```kotlin
class MainActivity : AppCompatActivity() {

    private lateinit var mTextView: TextView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        mTextView = findViewById(R.id.textView_base_model)
    }
}
```

在被`lateinit`修饰的变量被初始化前，如果访问这个变量的话，就会抛一个异常。

# 3. 接口
在Kotlin中使用interface来定义接口：
```kotlin
interface Food {
    fun cook() {
        // 可选的方法体
    }
    fun eat()
}
```

## 3.1 实现接口
和Java一样，一个类可以实现多个接口：
```kotlin
class Person : Food, Action {
    override fun eat() {
    }

    override fun walk() {
    }

    override fun play() {
    }
}
```
## 3.2 接口中的属性
和Java一样，Kotlin的接口中也可以存在属性。

只不过如果想要在Kotlin中定义属性，必须保证这个属性要么是抽象的，要么指定了访问器。
```kotlin
interface Food {
    val isCookFinished: Boolean // 抽象的属性
    val isEatFinished: Boolean // 指定了访问器的属性
        get() = isCookFinished

    fun cook() {
        // 可选的方法体
    }

    fun eat()
}
```

## 3.3 接口的继承
和类一样，接口也可以继承自另外一个接口，可以在父接口的基础上去添加新的方法或者属性。

# 4. 可见性修饰符
在Kotlin中，主要有4种修饰符：
- private
- protected
- internal
- public

如果没有指定修饰符，默认是`public`。

## 4.1 修饰符在包内
我们之前提到过，Kotlin可以直接直接在顶层声明类、函数和属性。
```kotlin
// 文件名：example.kt
package foo

private fun foo() { …… } // 在 example.kt 内可见

public var bar: Int = 5 // 该属性随处可见
    private set         // setter 只在 example.kt 内可见
    
internal val baz = 6    // 相同模块内可见
```
- 如果不指定任何修饰符，默认为public，意味着随处可见；
- 如果声明为private，它只能在声明他的文件内可见；
- 如果声明为internal，它只能在相同的模块(模块我们会在本文的4.4讲到)中可见；
- 顶层中不可使用protected(理由也很好想到——都没有类，怎么存在子类的概念)

## 4.2 修饰符在类和接口内
对于在类或者接口内的方法或者属性，我们四种修饰符都可用：
```kotlin
open class Outer {
    private val a = 1
    protected open val b = 2
    internal val c = 3
    val d = 4  // 默认 public
    
    protected class Nested {
        public val e: Int = 5
    }
}

class Subclass : Outer() {
    // a 不可见
    // b、c、d 可见
    // Nested 和 e 可见

    override val b = 5   // “b”为 protected
}

class Unrelated(o: Outer) {
    // o.a、o.b 不可见
    // o.c 和 o.d 可见（相同模块）
    // Outer.Nested 不可见，Nested::e 也不可见
}
```

- `private`：只在这个类的内部可见；
- `protected`：只在这个类的内部以及他的子类可见；
- `internal`：只在这个模块内可见；
- `public`：随处可见。

## 4.3 局部
局部变量、函数和类不可以有可见性修饰符。


## 4.4 Kotlin中的模块
可见性修饰符`internal`意味着该成员只在相同模块内可见。更具体地说， 一个模块是编译在一起的一套 Kotlin 文件：
- 一个 IntelliJ IDEA 模块；
- 一个 Maven 项目；
- 一个 Gradle 源集（例外是`test`源集可以访问`main`的`internal`声明）；
- 一次`<kotlinc>`Ant 任务执行所编译的一套文件。

