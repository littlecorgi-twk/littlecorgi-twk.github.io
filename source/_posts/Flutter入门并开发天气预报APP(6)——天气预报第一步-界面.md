---
title: Flutter入门并开发天气预报APP(6)——天气预报第一步-界面
categories:
  - Flutter
tags:
  - Flutter
  - Android
comments: true
copyright: true
abbrlink: 15395a84
date: 2019-10-11 21:21:40
updated: 2019-10-11 21:21:40
feature_img: 'https://cdn.littlecorgi.top/blog/Flutter.png'
cover: 'https://cdn.littlecorgi.top/blog/Flutter2.png'
keywords:
description:
---

经过前面的对于Flutter的介绍，我们现在已经可以开始写我们的天气预报APP的界面了。


项目Github地址：[a1203991686/CoolWeather_Flutter](https://github.com/a1203991686/CoolWeather_Flutter)

<!--More-->
# 1. 大致界面
最终写成的大致界面如图：

<img src="https://cdn.littlecorgi.top/mweb/2019-10-11/Screenshot_1570712111.png" width = 50% />

我们可以把这个界面拆分成如下部分：

<img src="https://cdn.littlecorgi.top/mweb/2019-10-11/Screenshot_1570711214.png" width = 50% />

可以看到我们APP主要有最上面用来显示地点和刷新时间的`Title`、显示温度和天气的两个`Text`、显示3天预报的`ListView`、显示空气质量的`GridView`、以及最后显示生活建议的`ListView`。

此处会用到我们前面学过的所有知识，如果有同学没有看过前面内容的，可以看下本系列前面的文章。

# 2. 创建项目
首先我们创建一个新的Flutter项目：
1. 在AndroidStudio点击`File`->`New Flutter Project`；
2. 接着选择`Flutter Application`->`NEXT`。接着填写你的项目名称等一系列信息后点击`next`；![New Flutter Project](https://cdn.littlecorgi.top/mweb/2019-10-11/New%20Flutter%20Project.png)
3. 接着输入你的组织/公司名称作为包名，最后点击`Finish`即可，这样就新建了一个Flutter项目，并且Flutter会自动为你生成一个计数器Demo；
    ![package ne](https://cdn.littlecorgi.top/mweb/2019-10-11/package%20new.png)
4. 让我们把`mian.dart`里面的所有注释以及`_MyHomePageState`里面的`build`方法里面的代码都删掉；
5. 接着在`lib`文件夹下新建一个文件夹，名叫`view`，到时候我们把所有与界面有关的代码文件都放在这个文件夹下面；
6. 然后我们在`view`文件夹下面新建一个dart文件，命名为`main_page`。这个就是我们的天气详情页。


# 3. 设计好天气详情页框架
首先我们需要导入`material`包，我们项目主要用material风格UI来写。

在开头输入`import 'package:flutter/material.dart';`，这样就导入了`material`包。接着创建我们的界面类`MainPage`。

由于我们在到时候写好网络请求后，所有的数据都得从网络获取，并且使用了异步的方法，也就是说我们先把页面加载了然后等获取的数据回来了在通知Flutter更新状态，所以这块我们得使用`StatefulWidget`。
```java
class MainPage extends StatefulWidget {
  MainPage({Key key}) : super(key: key);

  @override
  MainPageState createState() => new MainPageState();
}

class MainPageState extends State<MainPage> {
  @override
  Widget build(BuildContext context) {
  
  }
}
```

接着我们就得开始写`build()`里面的内容了。由于我们需要一个全屏的背景图片，所以我们就使用`Container`作为我们最外层的Widget，并使用`BoxDecoration`装饰容器配合`DecorationImage`来放置图片：
```java
@override
Widget build(BuildContext context) {
  return Container(
    decoration: BoxDecoration(
      image: DecorationImage(
        image: NetworkImage("http://blog.mrabit.com/bing/today"), //必应每日一图背景
        fit: BoxFit.cover, // 设置为全屏
      ),
    ),
    child: _weatherBody(),
  );
}
```

这个时候我们的运行结果如图(①你背景图片多半和我不一样，因为上面那个Uri获取的是必应的每日一图，每天图片都不一样；②运行的时候记得把`child: _weatherBody(),`给注释掉，因为我们还没有定义`_weatherBody()`这个方法):

<img src="https://cdn.littlecorgi.top/mweb/2019-10-11/Screenshot_1570797585.png" width = 50% />


# 4. 设计title
为了方便我就直接把剩下的布局单领出来放到`_weatherBody()`这个方法:
```java
Widget _weatherBody() {
  return
}
```
由于我们需要实现一个有Title的页面，所以最外层我选用了Scaffold：
```java
Widget _weatherBody() {
  return Scaffold(
    backgroundColor: Colors.transparent, //背景透明
    appBar: AppBar(
      centerTitle: true,
      title: Text(
        "北京",
        style: TextStyle(fontSize: 25.0),
      ),
      backgroundColor: Colors.transparent, //背景透明
      actions: <Widget>[ //右侧Widget，相当于Android Toolbar中的menu
        Container(
          alignment: Alignment.center, //向中间对齐
          child: Text(
            "12:10",
            textAlign: TextAlign.center, //文字向中间对齐
          ),
        )
      ],
    ),
    body: SingleChildScrollView( // 由于到时候整个页面一个屏幕可能放不下，就放置了一个滚动布局
    
    ),
  );
}
```

这个时候的运行结果是：

<img src="https://cdn.littlecorgi.top/mweb/2019-10-11/Screenshot_1570797746.png" width = 50% />


# 5. 设计下面天气预报页面
接下来可以开始下下面的天气显示页面了。
## 当前温度和天气情况
首先写一个纵向线性布局`Column`:
```java
body: SingleChildScrollView(
  child: Column(
    children: <Widget>[
      
    ]
  )
),
```
由于我们要显示在屏幕最右边，所以使用Align:
```java
body: SingleChildScrollView(
  child: Column(
    children: <Widget>[
      Align(
        alignment: Alignment.centerRight,
        child:Text(
          "26°C",
          style: TextStyle(
            fontSize: 50.0,
            color: Colors.white,
          ),
        ),
      ),
      Align(
        alignment: Alignment.centerRight,
        child:Text(
          "阴",
          style: TextStyle(
            fontSize: 20.0,
            color: Colors.white,
          ),
        ),
      ),
    ]
  )
),
```

运行结果是：

<img src="https://cdn.littlecorgi.top/mweb/2019-10-11/Screenshot_1570797847.png" width = 50% />


接下来我们需要显示3天天气预报、天气质量以及生活建议的三个子控件，同样为了方便我们也把他们给单领出来，于是上面的Column接下来可以这样写：
```java
body: SingleChildScrollView(
  child: Column(
    children: <Widget>[
      ... //上面的两个Text
      Padding(
        padding: EdgeInsets.only(
          left: 15.0,
          right: 15.0,
          bottom: 15.0,
        ),
        child: Container(
          color: Colors.black54,
          child: _weatherList(), //3天天气预报
        ),
      ),
      Padding(
        padding: EdgeInsets.only(
          left: 15.0,
          right: 15.0,
        ),
        child: Container(
          color: Colors.black54,
          child: _atmosphereList(), //空气质量
        ),
      ),
      Padding(
        padding: EdgeInsets.only(
          top: 15.0,
          left: 15.0,
          right: 15.0,
          bottom: 15.0,
        ),
        child: Container(
          color: Colors.black54,
          child: _lifestyleList(), //生活建议
        ),
      ),
    ]
  )
),
```

## 3天天气预报
对于3天天气预报我们主要通过ListView来实现，由于到时候数据比较灵活，所以我们就直接使用ListView.Builder来实现：
```java
List<String> dates = ["2019/10/01", "2019/10/02", "2019/10/03"];
List<String> temperatures = ["29/14", "30/18", "29/18"];
List<String> texts = ["阴", "晴", "雨"];

Widget _weatherList() {
    return Column(
        children: <Widget>[
            Align(
                alignment: Alignment.centerLeft,
                child: Text(
                    "预报",
                    style: TextStyle(
                        color: Colors.white,
                        fontSize: 20.0,
                    ),
                ),
            ),
            ListView.builder(
                shrinkWrap: true, //这个是指根据ListView所有子Widget的长度来设定ListView的长度
                physics: NeverScrollableScrollPhysics(), //禁止ListView自己的滑动，因为我们在外面用了个SingleChildScrollView，我们通过他的滑动就可以了
                itemCount: dates.length, //ListView子项个数
                itemBuilder: (BuildContext context, int index) {
                    return ListTile(
                        title: Row(
                            mainAxisAlignment: MainAxisAlignment.spaceBetween, //这个是Row的主轴的子项的分布格式，spaceBetween是指平均分布
                            mainAxisSize: MainAxisSize.max,
                            children: <Widget>[
                                Text(
                                    "${dates[index]}",
                                    style: TextStyle(color: Colors.white),
                                ),
                                Text(
                                    "${temperatures[index]}",
                                    style: TextStyle(color: Colors.white),
                                ),
                                Text(
                                    "${texts[index]}",
                                    style: TextStyle(color: Colors.white),
                                ),
                            ],
                        ),
                    );
                }),
        ],
    );
}
```

这个时候运行结果是：

<img src="https://cdn.littlecorgi.top/mweb/2019-10-11/Screenshot_1570798505.png" width = 50% />


## 空气质量
这块我们需要用到GridView，由于只有两个显示内容，而且我们后期也没有需要动态添加的需求，所以我们就直接使用GridView构造方法，而不去使用GridView的builder方法：
```java
List<String> atmospheres = ["16", "56"];

Widget _atmosphereList() {
    return Column(
        children: <Widget>[
            Align(
                alignment: Alignment.centerLeft,
                child: Text(
                    "空气质量",
                    style: TextStyle(
                        color: Colors.white,
                            fontSize: 20.0,
                    ),
                ),
            ),
            GridView(
                shrinkWrap: true, //见上面3天天气预报的ListView处
                physics: NeverScrollableScrollPhysics(), //见上面3天天气预报的ListView处
                gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
                    crossAxisCount: 2, //横轴三个子widget
                    childAspectRatio: 2 //显示区域宽高相等
                ),
                children: <Widget>[
                    Column(
                        children: <Widget>[
                            Text(
                                "${atmospheres[0]}",
                                style: TextStyle(
                                    color: Colors.white,
                                    fontSize: 40.0,
                                ),
                            ),
                            Text(
                                "能见度",
                                style: TextStyle(
                                    color: Colors.white,
                                    fontSize: 20.0,
                                ),
                            ),
                        ],
                    ),
                    Column(
                        children: <Widget>[
                            Text(
                                "${atmospheres[1]}",
                                style: TextStyle(
                                    color: Colors.white,
                                    fontSize: 40.0,
                                ),
                            ),
                            Text(
                                "湿度",
                                style: TextStyle(
                                    color: Colors.white,
                                    fontSize: 20.0,
                                ),
                            ),
                        ],
                    ),
                ],
            ),
        ],
    );
}
```

运行后的效果是：

<img src="https://cdn.littlecorgi.top/mweb/2019-10-11/Screenshot_1570799107.png" width = 50% />

## 生活建议
这块和3天天气预报一样，而且数据比3天天气预报更多，所以他更适合用ListView.builder：
```java
List<String> _lifestyleWeatherBrf = [
    "较舒适",
    "较舒适",
    "适宜",
    "适宜",
    "弱",
    "较适宜",
    "中"
];
List<String> _lifestyleWeatherTxt = [
    "白天天气晴好，早晚会感觉偏凉，午后舒适、宜人。",
    "建议着薄外套、开衫牛仔衫裤等服装。年老体弱者应适当添加衣物，宜着夹克衫、薄毛衣等。",
    "各项气象条件适宜，无明显降温过程，发生感冒机率较低。",
    "天气较好，赶快投身大自然参与户外运动，尽情感受运动的快乐吧。",
    "天气较好，但丝毫不会影响您出行的心情。温度适宜又有微风相伴，适宜旅游。",
    "紫外线强度较弱，建议出门前涂擦SPF在12-15之间、PA+的防晒护肤品。",
    "较适宜洗车，未来一天无雨，风力较小，擦洗一新的汽车至少能保持一天。",
    ",气象条件对空气污染物稀释、扩散和清除无明显影响，易感人群应适当减少室外活动时间。"
];

Widget _lifestyleList() {
    return Column(
        children: <Widget>[
            Align(
                alignment: Alignment.centerLeft,
                child: Text(
                    "生活建议",
                    style: TextStyle(
                        color: Colors.white,
                        fontSize: 20.0,
                    ),
                ),
            ),
            ListView.builder(
                shrinkWrap: true,
                physics: NeverScrollableScrollPhysics(),
                itemCount: _lifestyleWeatherBrf.length,
                itemBuilder: (BuildContext context, int index) {
                    return ListTile(
                        title: Column(
                            mainAxisAlignment: MainAxisAlignment.start,
                            crossAxisAlignment: CrossAxisAlignment.start,
                            children: <Widget>[
                                Text(
                                    "${_lifestyleWeatherBrf[index]}",
                                    style: TextStyle(
                                        color: Colors.white,
                                    ),
                                ),
                                Text(
                                    "${_lifestyleWeatherTxt[index]}",
                                    style: TextStyle(
                                        color: Colors.white,
                                    ),
                                ),
                            ],
                        ),
                    );
                },
            ),
        ],
    );
}
```

运行结果是:

<img src="https://cdn.littlecorgi.top/mweb/2019-10-11/Screenshot_1570799686.png" width = 50% />
