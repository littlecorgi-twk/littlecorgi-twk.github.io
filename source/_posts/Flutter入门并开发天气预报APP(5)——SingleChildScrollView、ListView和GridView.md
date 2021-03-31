---
title: Flutter入门并开发天气预报APP(5)——SingleChildScrollView、ListView和GridView
categories:
  - Flutter
tags:
  - Flutter
  - Android
comments: true
copyright: true
abbrlink: 30ff421d
date: 2019-10-10 22:08:38
updated: 2019-10-10 22:08:38
feature_img: 'https://cdn.littlecorgi.top/blog/Flutter.png'
cover: 'https://cdn.littlecorgi.top/blog/Flutter2.png'
keywords:
description:
---
下面我们来介绍下Flutter中的滑动控件`SingleChildScrollView`、列表`ListView`和表格`GridView`。
<!--More-->
# SingleChildScrollView
`SingleChildScrollView`类似于Android中的`ScrollView`，我使用的较浅，在我目前看来，他和`ScrollView`的唯一区别就是它还可以横向滚动。

我们先来看下他的定义：
```java
SingleChildScrollView({
  this.scrollDirection = Axis.vertical, //滚动方向，默认是垂直方向
  this.reverse = false, 
  this.padding, 
  bool primary, 
  this.physics, 
  this.controller,
  this.child,
})
```

- `scrollDirection`：设定滚动的方向，可以设定`Axis.vertical`或者`Axis.horizon`；
- `reverse`：是否按照阅读方向的反方向滑动，emmm这个可能有点觉得莫名其妙，但是如阿拉伯语言地区，他们阅读和我国古时候一样是从右向左的，所以如果`reverse`为`true`，且滚动方向是水平滚动的话，如果系统时阿拉伯语等从右向左阅读的语言时，方向是从右向左的；
- `padding`：留白的大小，和`Padding`的用法一样，通过`EdgeInsets`来设定，具体可以看前几章将Flutter基础Widget；
- `primary`：指是否使用widget树中默认的`PrimaryScrollController`；当滑动方向为垂直方向（`scrollDirection`值为`Axis.vertical`）并且没有指定`controller`时，`primary`默认为`true`；
- `physics`：用于控制滚动方式，这个等会重点讲解；
- `controller`：用于滚动监听及控制；
- `child`：子Widget。

## physics
这个参数是用于控制滚动方式，有几种参数：
- `NeverScrollablePhysics`：呈现不可滚动状态；
- `BouncingScrollPhysics`：当列表滑动结束时，会回弹列表，类似于iOS的列表滑动效果；
![BouncingScrollPhysics](https://cdn.littlecorgi.top/mweb/2019-10-10/BouncingScrollPhysics.gif)

- `ClampingScrollPhysics`：滑动结束时会显示水波纹阴影，类似于Android的列表滑动效果；
![ClampingScrollPhysics](https://cdn.littlecorgi.top/mweb/2019-10-10/ClampingScrollPhysics.gif)

- `FixedExtentScrollPhysics`：可以自己制作滑动效果，但是我也不会，所以在此不做解释😝。

# ListView
`ListView`可以沿一个方向上线性排布所有子组件（不仅局限于竖直方向）。

让我们来看下他的定义：
```java
ListView({
  ...  
  //可滚动widget公共参数
  Axis scrollDirection = Axis.vertical,
  bool reverse = false,
  ScrollController controller,
  bool primary,
  ScrollPhysics physics,
  EdgeInsetsGeometry padding,

  //ListView各个构造函数的共同参数  
  double itemExtent,
  bool shrinkWrap = false,
  bool addAutomaticKeepAlives = true,
  bool addRepaintBoundaries = true,
  double cacheExtent,

  //子widget列表
  List<Widget> children = const <Widget>[],
})
```
公共属性我们就不讲了，上面`SingleChildScrollView`已经讲过了，我们现在只讲他特有的：
- `itemExtent`：用于控制`ListView`的长度，如果不为`null`，则强制所有子Widget合起来的长度小于设定的值：如果`ListView`是横向的，则所有子Widget横向长度的和小于它；如果`ListView`是纵向的，则所有子Widget纵向长度的和小于它；
- `shrinkWrap`：该属性表示是否根据子组件的总长度来设置`ListView`的长度，默认值为`false` 。默认情况下，`ListView`的会在滚动方向尽可能多的占用空间。当`ListView`在一个无边界(滚动方向上)的容器中时，`shrinkWrap`必须为`true`；
- `addAutomaticKeepAlives`：该属性表示是否将列表项（子组件）包裹在`AutomaticKeepAlive`组件中；典型地，在一个懒加载列表中，如果将列表项包裹在`AutomaticKeepAlive`中，在该列表项滑出视口时它也不会被`GC（垃圾回收）`，它会使用`KeepAliveNotification`来保存其状态。如果列表项自己维护其`KeepAlive`状态，那么此参数必须置为`false`；
- `addRepaintBoundaries`：该属性表示是否将列表项（子组件）包裹在`RepaintBoundary`组件中。当可滚动组件滚动时，将列表项包裹在`RepaintBoundary`中可以避免列表项重绘，但是当列表项重绘的开销非常小（如一个颜色块，或者一个较短的文本）时，不添加`RepaintBoundary`反而会更高效。和`addAutomaticKeepAlive`一样，如果列表项自己维护其`KeepAlive`状态，那么此参数必须置为`false`；
- `cacheExtent`：设定缓存大小。

默认情况下一个一个设定`children`很麻烦，所以为了方便，Flutter还提供了一个`builder`构造方法：

## ListView.builder
```java
ListView.builder({
  // ListView公共参数已省略  
  ...
  @required IndexedWidgetBuilder itemBuilder,
  int itemCount,
  ...
})
```
- `itemCount`：是需要加载子Widget的长度；
- `itemBuilder`：列表项的构造器，当列表滚动到具体的`index`位置时，会调用该构建器构建列表项。

看例子：
```java
ListView.builder(
  itemCount: _provinces.length,
  itemBuilder: (context, index) {
    return GestureDetector(
      child: ListTile(
        title: Text("$index"),
      ),
      onTap: () {}));
      },
    );
  },
)
```
![ListViewBUilde](https://cdn.littlecorgi.top/mweb/2019-10-10/ListViewBUilder.png)


## ListView.separated
用于添加分割线。相较于`ListView.builder`多了一个`separatorBuilder`参数，该参数是一个分割组件生成器。

下面我们看一个例子：奇数行添加一条蓝色下划线，偶数行添加一条绿色下划线。
```java
class ListView3 extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    //下划线widget预定义以供复用。  
    Widget divider1=Divider(color: Colors.blue,);
    Widget divider2=Divider(color: Colors.green);
    return ListView.separated(
        itemCount: 100,
        //列表项构造器
        itemBuilder: (BuildContext context, int index) {
          return ListTile(title: Text("$index"));
        },
        //分割器构造器
        separatorBuilder: (BuildContext context, int index) {
          return index%2==0?divider1:divider2;
        },
    );
  }
}
```
![-w160](https://cdn.jsdelivr.net/gh/flutterchina/flutter-in-action/docs/imgs/6-3.png)

# GridView
`GridView`可以构造一个网格列表，定义如下:
```java
GridView({
  Axis scrollDirection = Axis.vertical,
  bool reverse = false,
  ScrollController controller,
  bool primary,
  ScrollPhysics physics,
  bool shrinkWrap = false,
  EdgeInsetsGeometry padding,
  @required SliverGridDelegate gridDelegate, //控制子widget layout的委托
  bool addAutomaticKeepAlives = true,
  bool addRepaintBoundaries = true,
  double cacheExtent,
  List<Widget> children = const <Widget>[],
})
```
大部分参数和`ListView`都相同，我们只关注`gridDelegate`这个参数：
`gridDelegate`作用是控制`GridView`子组件如何排列，Flutter中提供了两个类`SliverGridDelegateWithFixedCrossAxisCount`和`SliverGridDelegateWithMaxCrossAxisExtent`，我们可以直接使用，下面我们分别来介绍一下它们。

## SliverGridDelegateWithFixedCrossAxisCount
该子类实现了一个横轴为固定数量子元素的layout算法，其构造函数为：
```java
SliverGridDelegateWithFixedCrossAxisCount({
  @required double crossAxisCount, 
  double mainAxisSpacing = 0.0,
  double crossAxisSpacing = 0.0,
  double childAspectRatio = 1.0,
})
```
- `crossAxisCount`：横轴子元素的数量。此属性值确定后子元素在横轴的长度就确定了，即`ViewPort`横轴长度除以`crossAxisCount`的商。
- `mainAxisSpacing`：主轴方向的间距。
- `crossAxisSpacing`：横轴方向子元素的间距。
- `childAspectRatio`：子元素在横轴长度和主轴长度的比例。由于`crossAxisCount`指定后，子元素横轴长度就确定了，然后通过此参数值就可以确定子元素在主轴的长度。

## SliverGridDelegateWithMaxCrossAxisExtent
该子类实现了一个横轴子元素为固定最大长度的layout算法，其构造函数为：
```java
SliverGridDelegateWithMaxCrossAxisExtent({
  double maxCrossAxisExtent,
  double mainAxisSpacing = 0.0,
  double crossAxisSpacing = 0.0,
  double childAspectRatio = 1.0,
})
```
`maxCrossAxisExtent`为子元素在横轴上的最大长度，之所以是“最大”长度，是因为横轴方向每个子元素的长度仍然是等分的。其它参数和`SliverGridDelegateWithFixedCrossAxisCount`相同。

## GridView.builder
和`ListView`一样，当子Widget数量较多时，也提供了`builder`方法:
```java
GridView.builder(
 ...
 @required SliverGridDelegate gridDelegate, 
 @required IndexedWidgetBuilder itemBuilder,
)
```

这个就不在这举例了，`gridDelegate`和前面一样的，`itemBuilder`和`ListView`的一样的。