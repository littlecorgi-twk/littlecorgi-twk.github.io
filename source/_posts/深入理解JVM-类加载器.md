---
title: 深入理解JVM-类加载器
categories:
  - Java
tags:
  - Java
  - JVM
cover: 'https://raw.githubusercontent.com/zxh0/jvmgo-book/master/v1/gophers/cover.png'
toc_number: false
abbrlink: 594da31f
date: 2019-11-24 21:58:54
updated: 2019-11-24 21:58:54
description:
---
# 1. 概念
虚拟机设计团队把类加载阶段中的“**通过一个类的全限定名来获取描述此类的二进制字节流(即字节码)**”这个动作放到Java虚拟机外部去实现，以便让应用程序自己决定如何去获取所需要的类。实现这个动作的代码模块称为“**类加载器**”。

一般来说，Java 虚拟机使用 Java 类的方式如下：
* Java 源程序（.java 文件）在经过 Java 编译器编译之后就被转换成字节码（.class 文件）。
* 类加载器负责读取 Java 字节代码，并转换成 java.lang.Class类的一个实例。每个这样的实例用来表示一个 Java 类。通过此实例的 newInstance()方法就可以创建出该类的一个对象。

实际的情况可能更加复杂，比如 Java 字节代码可能是通过工具动态生成的，也可能是通过网络下载的。更详细的内容可以参考上一篇文章中讲类加载过程中的加载阶段时介绍的几个例子（JAR包、Applet、动态代理、JSP等）。

# 2. 类与类加载器
类加载器虽然只用于实现类的加载动作，但他在Java程序中起到的作用却远远不限于类加载阶段。对于任意一个类，都需要由加载他的类加载器和这个类本身异同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间。

通俗一点就是：比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class文件，被同一个虚拟机加载，只要加载他们的类加载器不同，那这两个类就必定不相等（这里指的相等，包括类的Class对象的equals()方法、isAssignableFrom()方法、isInstance()方法的返回结果，也包括使用instanceof()关键字对做对象所属关系判定等情况）。

# 3. 双亲委派模型
从Java虚拟机角度来讲，只存在两种不同的类加载器：
* 启动类加载器（Bootstrap ClassLoader），这个类加载器用C++实现，是虚拟机自身的一部分
* 所有其他类的加载器，这些类加载器由Java实现，独立于虚拟机外部，并且全都继承自抽象类java.lang.ClassLoader

从Java开发人员角度来看，类加载器还可以划分的更细致一些：
* 启动类加载器（Bootstrap ClassLoader） 此类加载器负责将存放在 <JAVA_HOME>\lib 目录中的，或者被 -Xbootclasspath 参数所指定的路径中的，并且是虚拟机识别的（仅按照文件名识别，如 rt.jar，名字不符合的类库即使放在lib 目录中也不会被加载）类库加载到虚拟机内存中。 启动类加载器无法被 Java 程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给引导类加载器，直接使用null代替即可。

* 扩展类加载器（Extension ClassLoader） 这个类加载器是由ExtClassLoader（sun.misc.Launcher$ExtClassLoader）实现的。它负责将<Java_Home>/lib/ext或者被 java.ext.dir系统变量所指定路径中的所有类库加载到内存中，开发者可以直接使用扩展类加载器。

* 应用程序类加载器（Application ClassLoader） 这个类加载器是由 AppClassLoader（sun.misc.Launcher$AppClassLoader）实现的。由于这个类加载器是ClassLoader中的getSystemClassLoader()方法的返回值，因此一般称为系统类加载器。它负责加载用户类路径（ClassPath）上所指定的类库，开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

由开发人员开发的应用程序都是由这三种类加载器相互配合进行加载的，如果有必要，还可以加入自己定义的类加载器。这些类加载器的关系一般如下图所示：
![](media/15745827002064/15745869705585.jpg)

上图展示的类加载器之间的层次关系，称为类加载器的双亲委派模型（Parents Delegation Model）。该模型要求除了顶层的启动类加载器外，其余的类加载器都应有自己的父类加载器，这里类加载器之间的父子关系一般通过组合（Composition）关系来实现，而不是通过继承（Inheritance）的关系实现。

## 3.1 工作过程
如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求时，子加载器才会尝试自己去加载。

这样的好处是Java类随着它的类加载器一起具备了一种带有优先级的层次关系。无论哪个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载，因此Object类在程序的各种类加载器环境中都是同一个类。

相反，如果没有使用双亲委派模式，由各个类加载器自行去加载的话，如果用户自己编写了一个称为java.lang.Object的类，并放在程序的ClassPath中，那系统中将会出现多个不同的Object类，Java类型体系中最基础的行为也就无法保证，应用程序也将会变得一片混乱。

## 3.2 实现
双亲委派模型的实现很简单，主要都集中在java.lang.ClassLoader的loadClass()方法中：先检查是否已经被加载过，如果没有，就调用父加载器的loadClass()，若父加载器为空，则默认使用启动类加载器作为父加载器。如果父类加载失败，抛出ClassNotFoundException异常后，再调用自己的findClass()方法进行加载。

```java
protected synchronized Class<?> loadClass(String name,boolean resolve)throws ClassNotFoundException{
    // 首先，检查请求的类是否已经加载过了
    Class c = findLoadedClass(name);
    if(c == null){
        try{
            if(parent != null){
                c = parent.loadClass(name,false);
            }else{
                c = findBootstrapClassOrNull(name);
            }
        }catch(ClassNotFoundException e){
            // 如果父类加载器抛出ClassNotFoundException
            // 说明父类加载器无法完成加载请求
        }
        if(c == null){
            // 在父类加载器无法加载的时候
            // 再调用本身的findClass方法进行类加载
            c = findClass(name);
        }
    }
    if(resolve){
        resolveClass(c);
    }
    return c;
}
```

# 4. 破坏双亲委派模型

三种破坏双亲委派模型的方式：
## 4.1 JDK1.2引入的双亲委派模型和JDK1.0就有的ClassLoader
双亲委派模型是JDK1.2之后才被引用的，而在JDK1.0就有ClassLoader了。所以面对已存在的用户自定义类加载器的实现代码，Java设计者引入了双亲委派模型时不得不做出一些妥协。

为了向前兼容，JDK1.2之后的java.lang.ClassLoader添加了一个新的protected方法findClass()。开发者只需要将自己的类加载逻辑代码写到findClass()方法中，在loadClass()方法的逻辑里如果父类加载失败，就会调用自己的findClass()方法来完成加载，这样就可以保证新写出来的类加载器符合双亲委派规则。

## 4.2 线程上下文类加载器
这个是因为双亲委派模式自身设计缺陷导致的。

双亲委派很好解决了各个类加载器的基础类统一的问题，但是如果基础类要调用会用户的代码咋办？

这个使用场景我不太了解，我看书上和别人博客上的例子也看不懂。我只能抄一下别人的例子。

>Java 提供了很多服务提供者接口（Service Provider Interface，SPI），允许第三方为这些接口提供实现。常见的 SPI 有 JDBC、JCE、JNDI、JAXP 和 JBI 等。这些 SPI 的接口由 Java 核心库来提供，如 JAXP 的 SPI 接口定义包含在 javax.xml.parsers包中。这些 SPI 的实现代码很可能是作为 Java 应用所依赖的 jar 包被包含进来，可以通过类路径（ClassPath）来找到，如实现了 JAXP SPI 的 Apache Xerces所包含的 jar 包。SPI 接口中的代码经常需要加载具体的实现类。如 JAXP 中的 javax.xml.parsers.DocumentBuilderFactory类中的 newInstance() 方法用来生成一个新的 DocumentBuilderFactory 的实例。这里的实例的真正的类是继承自 javax.xml.parsers.DocumentBuilderFactory，由 SPI 的实现所提供的。如在 Apache Xerces 中，实现的类是 org.apache.xerces.jaxp.DocumentBuilderFactoryImpl。而问题在于，SPI 的接口是Java 核心库的一部分，是由引导类加载器加载的，而SPI 实现的 Java 类一般是由系统类加载器加载的。引导类加载器是无法找到 SPI 的实现类的，因为它只加载 Java 的核心库。它也不能委派给系统类加载器，因为它是系统类加载器的祖先类加载器。也就是说，类加载器的双亲委派模型无法解决这个问题。
>
>为了解决这个问题，Java设计团队只好引入了一个不太优雅的设计：线程上下文类加载器（Thread Context ClassLoader）。线程上下文类加载器是从 JDK 1.2 开始引入的。类 java.lang.Thread中的方法 getContextClassLoader()和 setContextClassLoader(ClassLoader cl)用来获取和设置线程的上下文类加载器。如果没有通过 setContextClassLoader(ClassLoader cl)方法进行设置的话，线程将继承其父线程的上下文类加载器。Java 应用运行的初始线程的上下文类加载器是应用程序类加载器。在线程中运行的代码可以通过此类加载器来加载类和资源。


## 4.3 程序动态性
这里所说的“动态性”指的是当前一些非常热门的名词：代码热替换（HotSwap）、模块热部署(Hot Deployment)等。即希望应用程序能像计算机的外设一样，接上鼠标、键盘，不用重启就能立即使用，鼠标出了问题或需要升级就换个鼠标，不用停机或重启。

当前业界“事实上”的Java模块化标准是OSGi，而OSGi实现代码热部署的关键则是它自定义的类机载器的实现。

每一个程序模块（OSGi中称为Bundle）都有一个自己的类加载器，当需要更换一个Bundle时，就把Bundle连同类加载器一起换掉以实现代码的热替换。

在OSGi环境下，类加载器不再是双亲委派模型中的树状结构，而是进一步发展为更加复杂的网状结构，当收到类加载请求时，OSGi将按照下面的顺序进行类搜索：
1. 将以java.*开头的类委派给分类加载器加载
2. 否则，将委派列表名单内的类委派给父类加载器加载
3. 否则，将import列表中的类委派给Export这个类的Bundle的类加载器加载
4. 否则，查找当前Bundle的ClassPath，使用自己的类加载器加载
5. 否则，查找类是否在自己的Fragment Bundle中，如果在，则委派给Fragment Bundle的类加载器加载
6. 否则，查找Dynamic Import列表的Bundle，委派给对应Bundle的类加载器加载
7. 否则，类查找失败

