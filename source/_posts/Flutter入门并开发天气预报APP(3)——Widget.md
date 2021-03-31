---
title: Flutter入门并开发天气预报APP(3)——Widget
categories:
  - Flutter
tags:
  - Flutter
  - Android
comments: true
copyright: true
abbrlink: fc77cdce
date: 2019-10-09 11:46:56
updated: 2019-10-09 11:46:56
feature_img: 'https://cdn.littlecorgi.top/blog/Flutter.png'
cover: 'https://cdn.littlecorgi.top/blog/Flutter2.png'
keywords:
description:
---

# 简介
在Flutter中，Widget是个非常基本的东西，我在上一章就说过，Flutter中只要是界面都是Widget，你可以把它就理解成是控件，但是又和Android的View控件不同的是，在Flutter中，包括`Padding`、`Align`、手势检测的`GestureDetector`等等，都算是`Widget`。

其实大多数时候，你就可以把Widget直接理解成UI控件就行了，因为`Padding`、`Align`、`GestureDetector`等等也都是为UI服务的，你就可以理解成Flutter中只要与UI有关的属性都可以算是控件。

<!--More-->

# Widget的状态
在Android中，我们可以直接通过更新数据来达到刷新UI的目的。但是如果使用Flutter，就像我们前面说的计数器的Demo，如果直接通过`StatelessWidget`也就是无状态Widget的话，是没法进行刷新UI的，只能写一个死界面，也就是说如果在Flutter中界面写出来的就是一个死界面，只有通过刷新状态才能更新UI。

Widget有两个直接子类：`StatelessWidget`和`StatefulWidget`：
- `StatelessWidget`：这个是无状态Widget，实现`build()`方法后，就不可再变化，哪怕他的状态改变了。这句话可能有点矛盾，但是我来举个例子，比方说现在有一个`StatelessWidget`，他的内容就是一个`Text`，而这个`Text`的内容则显示了`counter`这个变量。当这个页面出现的时候，取了当时`counter`的值并显示了出来，但是我们后续不管`counter`这个值怎么改变，界面都是不会显示出来的。
- `StatefulWidget`：这个是有状态Widget，当你有状态需要改变的时候就可以通过他来改变状态。

State的几种状态：

|名称|状态|
|:-:|:-:|
|initState|create之后被insert到渲染树时调用的，只会调用一次|
|didChangeDependencies|state依赖的对象发生变化时调用|
|didUpdateWidget|Widget状态改变时候调用，可能会调用多次|
|build|构建Widget时调用|
|deactivate|当移除渲染树的时调用|
|dispose|Widget即将销毁时调用|

# StatelessWidget
这个类相对简单，只需要实现`build()`方法就可以了。

`StatelessWidget`用于不需要维护状态的场景，也就是说如果他需要显示的某一个参数的值发生变化了他也不变。

```java
class Test extends StatelessWidget {
  Test({Key key, this.title}) : super(key: key);

  final String title;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: new AppBar(
        title: Text("$title"),
      ),
      body: Text("1234"),
    );
  }
}
```
上面代码就演示了一个`StatelessWidget`的例子，功能是根据传入的`title`在界面的`title`了上显示出来。只实现了`build()`方法和构造方法。其中构造方法参数`Key`是必须得有的，而且如果该Widget还有其他状态的话，也需要写入构造方法。接着调用`build()`方法，将传入的`title`在界面的`title`上显示出来。

例如，如果我们调用它：
```java
@override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: Test(title: 'Flutter Demo Home Page'),
    );
  }
```
结果是：
![Screenshot_20191009-112644](https://cdn.littlecorgi.top/mweb/2019-10-09/Screenshot_20191009-112644.jpg)

# StatefulWidget
与`StatelessWidget`相对的就是`StatefulWidget`。它的重点在于可以根据状态来更新。

他较于`StatelessWidget`，少了我们常用的`build()`方法，多了`createState()`方法。下面我就来讲一下他的用法，我还是拿上面的`Test`做例子。

```java
class Test extends StatefulWidget {
  Test({Key key, this.title}) : super(key: key);

  final String title;

  @override
  _TestState createState() => _TestState();
}

class _TestState extends State<Test> {
  String _title;

  @override
  void initState() {
    setState(() {
      _title = widget.title;
    });
    super.initState();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: new AppBar(
        title: Text("$_title"),
      ),
      body: Text("1234"),
    );
  }
}
```
先看`Test`类，他继承自`StatefulWidget`，由于这个页面需要根据传入的参数来显示`title`，所以有一个状态`title`，并在构造方法中写入了他。接着调用了`createState()`方法。这个方法是所有`StatefulWidget`必须的，主要是为这个Widget创建他对应的State。

于是我们就在下面创建一个新的类，类名叫`_TestState`，继承自`State<Test>`。由于Widget有`title`这个状态，并且需要在界面中显示出来，所以为了降低他们的耦合度，我们在State类中也创建一个叫`_title`的状态，并`_title = widget.title;`。
接下来是一个`initState()`方法，这个方法也是State必须的，在上面Widget的状态中我们讲到过，这个是初始化State的，同时在`initSate()`里面，我们写了一个`setState()`，也就是说我们告诉Flutter说这个方法里面的参数改变了接着Widget的状态也得改变。最后就是`build()`方法。

接下来我们调用它：
```java
@override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: Test(title: '1234'),
    );
  }
```
结果是：
![Screenshot_20191009-114316](https://cdn.littlecorgi.top/mweb/2019-10-09/Screenshot_20191009-114316.jpg)
