---
title: Flutter入门并开发天气预报APP(2)——Flutter计数器Demo
categories:
  - Flutter
tags:
  - Flutter
  - Android
comments: true
copyright: true
abbrlink: 956d1e66
date: 2019-10-09 10:53:57
updated: 2019-10-09 10:53:57
feature_img: 'https://cdn.littlecorgi.top/blog/Flutter.png'
cover: 'https://cdn.littlecorgi.top/blog/Flutter2.png'
keywords:
description:
---

本文样例都是使用的AndroidStudio，如果你使用Xcode或者vscode，代码都是一样的，只是IDE创建项目的方式以及运行按键位置不同而已。

<!-- More -->

# 创建Flutter项目
AndroidStudio中选择`File`->`New`->`New Flutter Project`->`Flutter Application`->`NEXT`。接着填写你的项目名称等一系列信息后就创建成功了，我们就会看到AndroidStudio已经为我们创建好了一个Demo，接着连上你的手机或者打开你的虚拟机，按下AndroidStudio的运行键，不一会儿，你就会惊喜的发现你的手机上就有一个Flutter计数器Demo了。
![计数器运行Demo](https://cdn.littlecorgi.top/mweb/2019-10-09/%E8%AE%A1%E6%95%B0%E5%99%A8%E8%BF%90%E8%A1%8CDemo.png)

# 介绍下AndroidStudio界面
![AndroidStudio界面](https://cdn.littlecorgi.top/mweb/2019-10-09/AndroidStudio%E7%95%8C%E9%9D%A2.png)
这是我的AndroidStudio界面，接下来我分三个部分介绍下AndroidStudio的大致界面。

## 控制区
![控制区](https://cdn.littlecorgi.top/mweb/2019-10-09/%E6%8E%A7%E5%88%B6%E5%8C%BA.png)

一次介绍每一个按钮的功能。
1. 第一个是用来选择运行设备的，比方说我现在就选的是我的虚拟机，然后我按下运行后就可以将项目运行到我的虚拟机上而不会运行到连着我的电脑的手机上去；
2. 第二个是选择运行哪个Flutter module，如果你同一个项目里面有多个module，你就可以在此选择你想要运行的module；
3. 第三个灰色的选项框我也没弄懂她是干嘛的，我至今都没能选择过，一直都是灰的；
4. 绿色的三角形，大家熟知，运行键；
5. 红色的瓢虫，大家熟知，debug键；
6. 不知道干嘛的；
7. FlutterAPP的性能监视器；
8. 闪电，这个是重点，也是Flutter的一大特性，热重载键，他能做到不需要重新编译代码就能将你修改的代码的效果在你的APP上显示出来；
9. 不知道干嘛的；
10. 红色的正方形，停止运行按钮。

## Run运行区
![Run运行区](https://cdn.littlecorgi.top/mweb/2019-10-09/Run%E8%BF%90%E8%A1%8C%E5%8C%BA.png)
就是Flutter的信息显示区，类似于AndroidStudio的Logcat，能显示FlutterAPP运行中的一些信息。

## Flutter Outline/Flutter Inspector区
![Flutter Outline](https://cdn.littlecorgi.top/mweb/2019-10-09/Flutter%20Outline.png)
![Flutter Inspecto](https://cdn.littlecorgi.top/mweb/2019-10-09/Flutter%20Inspector.png)
用于显示当前界面的一些构建信息。

# Flutter项目结构
![Flutter项目结构](https://cdn.littlecorgi.top/mweb/2019-10-09/Flutter%E9%A1%B9%E7%9B%AE%E7%BB%93%E6%9E%84.png)

我们挑一些重要的说一下：
1. .idea文件夹，AndroidStudio识别项目的必备文件夹，不用管；
2. Android文件夹，存放Flutter构建的Android项目的文件夹；
3. ios文件夹，存放Flutter构建的iOS项目的文件夹；
4. lib文件夹，存放Flutter项目和资源的文件夹；
5. pubspec.yaml文件，存放Flutter项目相关信息，如项目名称、版本、依赖等等，类似于Android项目的build.gradle文件。

# main.dart文件
我们通过讲这个文件来给大家大致介绍下Flutter项目代码结构。

全部源代码在此
```dart
import 'package:flutter/material.dart';

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: MyHomePage(title: 'Flutter Demo Home Page'),
    );
  }
}

class MyHomePage extends StatefulWidget {
  MyHomePage({Key key, this.title}) : super(key: key);
  final String title;

  @override
  _MyHomePageState createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  int _counter = 0;

  void _incrementCounter() {
    setState(() {
      _counter++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Text(
              'You have pushed the button this many times:',
            ),
            Text(
              '$_counter',
              style: Theme.of(context).textTheme.display1,
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _incrementCounter,
        tooltip: 'Increment',
        child: Icon(Icons.add),
      ),
    );
  }
}

```

## import
首先我们看最开始的`import`区。

这块相当于C/C++的`include`，反而不像java的`import`。我为什么这么说呢，你想一下，你写java程序的时候，你是需要哪个类就直接在代码中输入类名，然后IDE会自动给你显示有这个类的所有的包，然后让你选包去导入；而写dart是需要你先导入包，才能在代码中写这个包中的类，如果你没有先导入包而直接到下面写的话，IDE是不会提示你这个类可能会在哪个包中，而是直接报错，这点和C/C++比较类似。

那我们为什么导入的是`flutter/material.dart`这个包呢？我在第一节里面说过，Flutter他内置了Android的Material Design风格和iOS的Cupertino风格的UI；所以这块就是导入Material Design风格UI文件，而如果你要使用Cupertino风格UI文件的话，把它改成Cupertino就行了。

## main()
```dart
void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ), 
      home: MyHomePage(title: 'Flutter Demo Home Page'),
    );
  }
}
```
- 第一行，指明项目的入口就是`MyApp()`。固定格式，可以不用管。

- 第二行就定义了一个类`MyApp`继承自`StatelessWidget`，并重写了父类的`build()`方法。`Widget`可以看成Flutter中的页面，只要是Flutter中的页面，都必须继承自`Widget`。他有点类似于Android中的`View`，但是不同于`View`的是，Flutter中的`padding`、`align`、`layout`等居然也是`Widget`。

- 如果是`StatelessWidget`，则代表她是一个“状态少”的`Widget`，我的理解就是状态相当于`Widget`的一些数值，比方说页面要显示计数器的计数`count`，那么这个`count`就是一个状态，只要有状态就使用`StatefulWidget`。像`MyApp`它没有状态，于是就用`StatelessWidget`。

- 继承自`StatelessWidget`的类必须实现`build()`方法，相当于这个类是一个死页面，只需要展示固定的内容，也就是不需要状态，所以直接通过`build()`直接写入界面就行。

- `build()`方法必须同样返回一个`Widget`，所以我们就使用`MaterialApp`，其中我们给他的`title`、`theme`、`home`进行了赋值，同学们应该都能大致读懂啥意思，首页`home`则给了`MyHomePage`.

## MyHomePage
```dart
class MyHomePage extends StatefulWidget {
  MyHomePage({Key key, this.title}) : super(key: key);
  final String title;

  @override
  _MyHomePageState createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  int _counter = 0;

  void _incrementCounter() {
    setState(() {
      _counter++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Text(
              'You have pushed the button this many times:',
            ),
            Text(
              '$_counter',
              style: Theme.of(context).textTheme.display1,
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _incrementCounter,
        tooltip: 'Increment',
        child: Icon(Icons.add),
      ),
    );
  }
}
```

### MyHomePage
我们在前面说过，如果一个页面有数据也就是“状态”的话，那么他就得继承自`StatefulWidget`。

如果一个类继承自`StatefulWidget`，那这个类会很简单，只需要创建一个构造方法，构造方法中写入`key`以及该`Widget`需要的数据。然后重写`createState()`方法。

### MyHomePageState
如果一个类继承自`StatefulWIdget`，那么一定会有一个他的`State`类并继承自`State<该类>`，就像`class _MyHomePageState extends State<MyHomePage>`一样。

那我们来看看State类里面有些啥东西:

首先先定义“状态”：由于我们这个APP主要的功能是用来计数，所以必须定义一个int型变量来计数：
```dart
int _counter = 0;
```
接着如果我们这个_counter变量变了，那就得通知flutter说状态变了，需要更新UI：
```dart
void _incrementCounter() {
  setState(() {
     _counter++;
  });
}
```
其中，`setState()`方法就是用来通知`Flutter`这个状态改变了，要去更新UI的。

最后重写`build()`方法来写入界面：

- `Scaffold`是一个组件，它提供了默认的导航栏、标题和包含主屏幕`widget`树（后同“组件树”或“部件树”）的`body`属性。
- `body`后面是一个`Center`组件，它的作用是将它的子`Widget`定位到父`Widget`的中间。
- `Center`后面是一个`Column`组件，这个组件类似于Android中的`LinearLayout`的`orientation: Vertical`，也就是垂直向的线性布局。同时Flutter也提供了`Row`这个水平向的线性布局。他们的子`Widget`是`children`属性，也就是可以写多个`Widget`。
- `Column`里面是两个`Text`，这个没啥讲的
- 接下来是一个`FloatingActionButton`，这就是一个浮动按钮，Android里面有这个控件，核心是他的`onPressed`属性，这个就相当于`onClick()`，用于处理点击事件。

# 体验热重启
首先先让项目在虚拟机或者手机上运行着，然后我们把页面Text中的"You have pushed the button this many times:"改为"You have clicked the button this many times:"，然后按热重启键（macOS版AndroidStudio默认是commond+s键），接着你就能里面在虚拟机或者手机上看到修改之后的结果。
![flutter热重启](https://cdn.littlecorgi.top/mweb/2019-10-09/%E6%9C%AA%E5%91%BD%E5%90%8D%20-2-.gif)
