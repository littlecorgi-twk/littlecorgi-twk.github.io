---
title: Flutter入门并开发天气预报APP(4)——基础Widget
categories:
  - Flutter
tags:
  - Flutter
  - Android
comments: true
copyright: true
abbrlink: a3b37eac
date: 2019-10-10 20:15:26
updated: 2019-10-10 20:15:26
feature_img: 'https://cdn.littlecorgi.top/blog/Flutter.png'
cover: 'https://cdn.littlecorgi.top/blog/Flutter2.png'
keywords:
description:
---
到这章我们就差不多可以开始写天气预报了。首先我们来看一下一些基础简单的Widget。

<!--More-->

# 基础组件
## 文本
`Text`用于显示简单的文本，包含一些控制文本显示的属性。
```java
Text(
    "1234",
),
Text(
    "1234",
    style: TextStyle(
        color: Colors.purple,
        fontSize: 32.0,
        fontWeight: FontWeight.bold,
    ),
),
```
![Text示例](https://cdn.littlecorgi.top/mweb/2019-10-10/Text%E7%A4%BA%E4%BE%8B.png)

- 需要显示的文本信息直接放到一个双引号里面就可以了；
- `textAlign`：文本对齐方式；
- `maxLines`：文本显示的最大行数；
- `overflow`：指定多余文本的截断方式；
- `textScaleFactor`：指定文本相对于当前字体大小的缩放因子；
- `TextStyle`：设置显示文本的字体、颜色、粗细等样式：
    - `height`：指定行高，但是不是绝对值，而是一个因子，相当于`fontsize * height`；
    - `fontFamily`：设置字体；
    - `fontSize`：设置字体大小

## 按钮
不同的组件库有不同的按钮，我们现在只拿Material组件库中的按钮举例。

### RaisedButton
漂浮按钮，带有阴影和灰色背景。按下后阴影会变大。
```java
RaisedButton(
  child: Text("1234"),
  onPressed: () {},
);
```
<figure class="half">
    <img src="https://cdn.littlecorgi.top/mweb/2019-10-10/RaisedButton%E6%9C%AA%E6%8C%89%E4%B8%8B.png">
    <img src="https://cdn.littlecorgi.top/mweb/2019-10-10/RaisedButton%E6%8C%89%E4%B8%8B.png">
</figure>

### FlatButton
扁平化按钮，背景透明且不带阴影，按下后会有背景色。
```java
FlatButton(
  child: Text("1234"),
  onPressed: () {},
)
```
<figure class="half">
    <img src="https://cdn.littlecorgi.top/mweb/2019-10-10/FlatButton%E6%9C%AA%E6%8C%89%E4%B8%8B.png">
    <img src="https://cdn.littlecorgi.top/mweb/2019-10-10/FlatButton%E6%8C%89%E4%B8%8B.png">
</figure>

### IconButton
可点击的Icon，默认没有背景，按下后出现阴影
```java
IconButton(
  icon: Icon(Icons.thumb_up),
  onPressed: () {},
)
```
![-w88](https://cdn.littlecorgi.top/mweb/2019-10-10/15706115387455.png)

## 图片
我们通过`Image`来显示图片，来源可以是`asset`、网络等位置。

### 从asset加载图片
1. 现在项目根目录(也就是和android、ios、lib等目录同级)新建一个`images`目录，并把图片`main.png`拷进去；
2. 在`pubspec.yaml`中的flutter部分添加一下内容：
    ![assets](https://cdn.littlecorgi.top/mweb/2019-10-10/assets.png)
3. 加载该图片

```java
Image(
  image: AssetImage("images/amoled.png"),
);
```

`Image`也提供了一个快速构造函数：
```java
Image.asset(
  "images/amoled.png",
)
```

### 从网络加载图片
```java
Image(
  image: NetworkImage(
      "https://s2.ax1x.com/2019/05/27/VZrQ3V.png"),
)
```
或者
```java
Image.network(
  "https://s2.ax1x.com/2019/05/27/VZrQ3V.png",
)
```

### 参数
`Image`有一些基本参数
```java
const Image({
  ...
  this.width, //图片的宽
  this.height, //图片高度
  this.fit,//缩放模式
  this.alignment = Alignment.center, //对齐方式
  this.repeat = ImageRepeat.noRepeat, //重复方式
  ...
})
```

- `width`和`height`：宽和高；
- `fit`：缩放模式：
    - `fill`：拉伸图片知道填满；
    - `cover`：按原图长宽比放大图片来填满，多余的部分舍去；
    - `contain`：在保证图片长宽比不变的情况下尽可能去填满；
    - `fitWidth`：宽度会缩放到显示空间的宽度，高度会按比例缩放，如果有多余的部分会被舍去；
    - `fitHeight`：与`fitWidth`同理；
    - `none`：没有适应策略，图片多大就显示多大，如果图片原尺寸小于显示空间就只显示原尺寸；如果大于则舍弃多余部分只显示中间部分；
- `repeat`：当图片小于显示空间时，会将图片重复显示。

# 布局组件
## 线性布局(Row、Column)
线性布局就相当于Android里面的`LinearLayout`，但是不同的是Flutter将竖直和水平布局单独拿了出来。

线性布局分为竖直布局`Column`和水平布局`Row`。他两属性都是一样的。
再说属性之前我们先熟悉两个概念：主轴和交叉轴。如果是`Column`，主轴是竖直轴，交叉轴是水平轴；如果是`Row`，主轴是水平轴，交叉轴是竖直轴。
```java
Row({
  ...  
  MainAxisSize mainAxisSize = MainAxisSize.max,    
  MainAxisAlignment mainAxisAlignment = MainAxisAlignment.start,
  CrossAxisAlignment crossAxisAlignment = CrossAxisAlignment.center,
  List<Widget> children = const <Widget>[],
})
```

- `mainAxisSize`：主轴占用空间。默认是`MainAxisSize.max`，是指占用全部主轴空间。如果设置为`MainAxisSize.min`，那就只占用所有子组件所需要的空间；
- `mainAxisAlignment`：子Widget在主轴的对其方向；
- `crossAxisAlignment`：子Widget在交叉轴的对其方向；
- `children`：所有的子Widget。

## 弹性布局(Flex)
其实这个布局和线性布局很有渊源，为什么这么说呢，因为`Row`和`Column`都继承自它。
因此关于他和线性布局重复的地方我们现在就不再讲了，我们直说他“弹性”的部分。

### Expanded
可以按比例“拉伸”`Row`、`Column`和`Flex`子组件所占的空间。

```java
const Expanded({
  int flex = 1, 
  @required Widget child,
})
```
`flex`为弹性系数，如果为`0`或者`null`，则`child`不会阔伸占用的控件。如果大于`0`，则会按照`flex`的比例来分隔主轴全部空闲空间。其实说白了，就和Android里面`LinearLayout`的`weight`一样的。但是我们还是来举个例子吧。

```java
return Scaffold(
      appBar: new AppBar(
        title: Text("$_title"),
      ),
      body: Center(
        child: Flex(
          children: <Widget>[
            Flex(
              direction: Axis.horizontal,
              children: <Widget>[
                Expanded(
                  flex: 1,
                  child: Container(
                    color: Colors.blue,
                    child: Text("1234"),
                  ),
                ),
                Expanded(
                  flex: 2,
                  child: Container(
                    color: Colors.red,
                    child: Text("1234"),
                  ),
                ),
              ],
            ),
            Flex(
              direction: Axis.horizontal,
              children: <Widget>[
                Expanded(
                  flex: 3,
                  child: Container(
                    color: Colors.blue,
                    child: Text("1234"),
                  ),
                ),
                Expanded(
                  flex: 1,
                  child: Container(
                    color: Colors.red,
                    child: Text("1234"),
                  ),
                ),
              ],
            ),
            Flex(
              direction: Axis.horizontal,
              children: <Widget>[
                Expanded(
                  flex: 1,
                  child: Container(
                    color: Colors.blue,
                    child: Text("1234"),
                  ),
                ),
                Expanded(
                  flex: 1,
                  child: Container(
                    color: Colors.red,
                    child: Text("1234"),
                  ),
                ),
              ],
            ),
          ],
          direction: Axis.vertical,
        ),
      ),
);
```
![Flex示例](https://cdn.littlecorgi.top/mweb/2019-10-10/Flex%E7%A4%BA%E4%BE%8B.png)

## 流式布局(Wrap、Flow)
如果使用线性布局的话，当需要显示的内容超出屏幕边界的时候就会报错。
![超出屏幕边界](https://cdn.littlecorgi.top/mweb/2019-10-10/%E8%B6%85%E5%87%BA%E5%B1%8F%E5%B9%95%E8%BE%B9%E7%95%8C.png)

为了避免这种情况，我们就可以使用流式布局。当需要显示的内容超出屏幕边界的时候，就自动折行来继续显示。

Flutter中通过`Wrap`和`Flow`来实现流式布局。

### Wrap
我们来看下`Wrap`主要的一些参数：
```java
Wrap({
  ...
  this.direction = Axis.horizontal,
  this.alignment = WrapAlignment.start,
  this.spacing = 0.0,
  this.runAlignment = WrapAlignment.start,
  this.runSpacing = 0.0,
  this.crossAxisAlignment = WrapCrossAlignment.start,
  this.textDirection,
  this.verticalDirection = VerticalDirection.down,
  List<Widget> children = const <Widget>[],
})
```

其中很多参数`Row`和`Column`中都有，就不再介绍了，主要介绍点不同的：
- `spacing`：主轴方向子widget的间距
- `runSpacing`：纵轴方向的间距
- `runAlignment`：纵轴方向的对齐方式

下面有一个示例：
```java
Wrap(
  spacing: 8.0, // 主轴(水平)方向间距
  runSpacing: 4.0, // 纵轴（垂直）方向间距
  alignment: WrapAlignment.center, //沿主轴方向居中
  children: <Widget>[
    new Chip(
      avatar: new CircleAvatar(backgroundColor: Colors.blue, child: Text('A')),
      label: new Text('Hamilton'),
    ),
    new Chip(
      avatar: new CircleAvatar(backgroundColor: Colors.blue, child: Text('M')),
      label: new Text('Lafayette'),
    ),
    new Chip(
      avatar: new CircleAvatar(backgroundColor: Colors.blue, child: Text('H')),
      label: new Text('Mulligan'),
    ),
    new Chip(
      avatar: new CircleAvatar(backgroundColor: Colors.blue, child: Text('J')),
      label: new Text('Laurens'),
    ),
  ],
)
```
![Wrap示例](https://cdn.littlecorgi.top/mweb/2019-10-10/Wrap%E7%A4%BA%E4%BE%8B.png)

### Flow
`Flow`较复杂，需要自己实现子Widget的位置转换，一般不推荐使用`Flow`。但是如果需要自定义布局策略，或者对性能要求较高，这个时候就得用`Flow`了。

但是由于太过于复杂，我也没咋用过，我就不在这讲了😝，大家有需要的可以百度，或者看我推荐的这篇：[4.4 流式布局-《Flutter实战》](https://book.flutterchina.club/chapter4/wrap_and_flow.html)

## 层叠布局(Stack)
这个布局类似于Android中的`Frame`，允许在父布局的任意地方放置布局。
```java
Stack({
  this.alignment = AlignmentDirectional.topStart,
  this.textDirection,
  this.fit = StackFit.loose,
  this.overflow = Overflow.clip,
  List<Widget> children = const <Widget>[],
})
```
- `alignment`：决定如何去对齐；
- `textDirection`：确定`alignment`的参考系；
- `fit`：没有定位的子Widget如何去适应`Stack`的大小；
- `overflow`：决定超出`Stack`显示空间的子Widget如何去显示，如果`Overflow.clip`，则超出部分会隐藏，`Overflow.visible`则不会。

## 对齐与相对定位(Align)

Align可以调整子Widget的位置，并且可以根据子Widget的宽高来确定自身的宽高。
```java
Align({
  Key key,
  this.alignment = Alignment.center,
  this.widthFactor,
  this.heightFactor,
  Widget child,
})
```
- alignment：代表子Widget在父Widget的起始位置；
- widthFactor和heightFactor确定Align本身的宽高。

来看一个简单的例子：
```java
Container(
  height: 120.0,
  width: 120.0,
  color: Colors.blue[50],
  child: Align(
    alignment: Alignment.topRight,
    child: FlutterLogo(
      size: 60,
    ),
  ),
)
```
运行结果：
![-w100](https://cdn.littlecorgi.top/mweb/2019-10-10/15706312811307.png)

# 3. 容器类组件
## 填充(Padding)
用过Android的同学一定熟悉，Padding就是负责留白的嘛。
但是跟Android里面不同的是，在Android里面我们一般是在一个View里面添加Padding，但是在Flutter里面，Padding直接变成了一个Widget、一个布局。

使用方法：
```java
Padding ({
  EdgeInsetsGeometry padding,
  Widget child,
})
```
对于EdgeInsetsGeometry我们一般使用EdgeInsets类。

### EdgeInsets
EdgeInsets类提供了几个便捷的方法：
- fromLTRB(double left, double top, double right, double bottom)：分别指定四个方向的填充；
- all(double value) : 所有方向均使用相同数值的填充；
- only({left, top, right ,bottom })：可以设置具体某个方向的填充(可以同时指定多个方向)；
- symmetric({ vertical, horizontal })：用于设置对称方向的填充，vertical指top和bottom，horizontal指left和right；

示例：
```java
body: Column(
  children: <Widget>[
    Padding(
      padding: EdgeInsets.only(left: 12.0, bottom: 10.0),
      child: Text("1234"),
    ),
    Padding(
      padding: EdgeInsets.fromLTRB(0.0, 0.0, 10.0, 10.0),
      child: Text("1234"),
    ),
    Padding(
      padding: EdgeInsets.symmetric(vertical: 10.0),
      child: Text("1234"),
    ),
  ],
),
```
运行结果：
![Padding示例](https://cdn.littlecorgi.top/mweb/2019-10-10/Padding%E7%A4%BA%E4%BE%8B.png)