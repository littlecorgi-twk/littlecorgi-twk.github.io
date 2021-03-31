---
title: Flutter入门并开发天气预报APP(8)——天气预报第二步-选择省、市、区界面及网络请求
categories:
  - Flutter
tags:
  - Flutter
  - Android
comments: true
copyright: true
abbrlink: f0e3723
date: 2019-10-14 16:55:26
updated: 2019-10-14 16:55:26
feature_img: 'https://cdn.littlecorgi.top/blog/Flutter.png'
cover: 'https://cdn.littlecorgi.top/blog/Flutter2.png'
keywords:
description:
---

> 项目Github地址：[a1203991686/CoolWeather_Flutter](https://github.com/a1203991686/CoolWeather_Flutter)

在第六章中我们写了天气预报的页面， 但是你作为天气预报肯定能选择城市吧。所以我们现在来写选择省、市、区的界面。

我们使用的是郭霖大神在第一行代码最后面酷欧天气的API。

<!-- More -->

# 1. 实现界面
既然是一个选择省市区的界面，那么我们就用`ListView`。

首先看一下大致界面：
![Screenshot_1570967184](https://cdn.littlecorgi.top/mweb/2019-10-14/Screenshot_1570967184.png)


就直接使用`ListView`的`builder()`方法，忘了的同学可以看前面第5章。



## 省
在`view`文件夹下新建一个类`provinces_page.dart`，接下来就在这个文件里面写代码：
```java
class ProvincesPageWidget extends StatefulWidget {
    ProvincesPageWidget({Key key}) : super(key: key);

    @override
    ProvincesPageStateWidget createState() => new ProvincesPageStateWidget();
}

class ProvincesPageStateWidget extends State<ProvincesPageWidget> {
    @override
    Widget build(BuildContext context) {
        return Scaffold(
            appBar: new AppBar(
                title: Text(
                    "省份",
                    style: TextStyle(fontSize: 25.0),
                ),
            ),
            body: ListView.builder(
                itemCount: 30,
                itemBuilder: (context, index) {
                    return ListTile(
                            title: Text("$index"),
                    );
                },
            ),
        );
    }
}
```


## 市
和省一样。在`view`文件夹下新建一个类`city_page.dart`，接下来就在这个文件里面写代码。照着省的依葫芦画瓢，写一个`ListView`。

## 区
和省一样。在`view`文件夹下新建一个类`counties_page.dart`，接下来就在这个文件里面写代码。照着省的依葫芦画瓢，写一个`ListView`。

# 2. 网络请求
网络请求可以看下之前第7章的内容。主要是通过`Dio`进行网络请求。大家可以看下第7章网络请求的内容。

我们网络请求主要需要以下几个API：
- 请求省份列表：`http://guolin.tech/api/china`
- 请求对应市列表：`http://guolin.tech/api/china/provinceID`
- 请求对应区县列表：`http://guolin.tech/api/china/provinceID/cityID`

由于我们在之前文章中使用的省作为例子，这块我们就用区县作为例子：

## 转为Dart类
首先使用网站[https://caijinglong.github.io/json2dart/](https://caijinglong.github.io/json2dart/)来讲Json数据转为Dart实体类：
![Json](https://cdn.littlecorgi.top/mweb/2019-10-14/Json.png)

然后在项目`lib`目录新建一个文件夹。名为`bean`。这个文件夹主要存放实体类的代码。然后在这个文件夹下面新建一个dart文件，命名为`Counties`。然后把生成的Dart代码复制粘贴进去。但是你会发现复制进去后会报错，这个不用急，接下来我们来处理错误。

## 生成.g.dart文件
接下来在项目根目录的`pubspec.yaml`文件的`dependencies`项下面添加依赖`json_annotation`、`dev_dependencies`项下面添加`build_runner`和`json_serializable`。

接着打开终端，定位到项目根目录，输入
`flutter packages pub run build_runner build`，运行即可。运行完毕你就会发现你项目的存实体类的`lib/bean`目录多了一个文件，名为`Counties.g.dart`。同时你`Counties.dart`里面的错误也没有了。

## 网络请求
最后你就可以通过Dio来进行网络请求了。
```java
Dio().get(http://guolin.tech/api/china/$_provinceID/$_cityID);
```

那么到这有的同学可能就会问了，你网址里面有`provinceID`和`cityID`，那这两个怎么获取，这个时候就得用上带值路由跳转了。

我们一般都是这样的一个逻辑:先选择省份、再选择城市、最后选择区和县。所以我们这个`CountyPageWidget`肯定是由`CityPageWidget`调用的。那么我们可以在`CityPageWidget`的`ListView`里面将`Text`通过`GestureDetector`包起来，然后将`GestureDetector`的`onTap`方法设置为跳转到`CountyPageWidget`。`CityPageWidget`的`ListView`代码如下：
```java
ListView.builder(
    itemCount: _cityList.cities.length,
    itemBuilder: (context, index) {
        return GestureDetector(
            child: ListTile(
                title: Text("${_cityList.cities[index].name}"),
            ),
            onTap: () {
                Navigator.push(context,
                    MaterialPageRoute(builder: (context) {
                        return CountiesPageWidget(
                            provinceID: _provinceID, //由CityWidget的上一级ProvinceWidget传过来，
                            cityID: _cityList.cities[index].id, // 由网络请求回来的数据传入
                        );
                    }));
            },
        );
    },
);
```

这样`CountyPageWidget`所需的`provinceID`和`cityID`都是由`CityPageWidget`传入的，那么我们怎么对他传入的值进行处理呢，怎么把它添加到网址里去？接下来大家看代码就行了：
```java
class CountyPageWidget extends StatefulWidget {
  //在Widget中定义两个，方便由其他Widget传入
  final int provinceID;
  final int cityID;

  // 设置构造方法，在构造方法中传入两个变量
  CountyPageWidget({Key key, this.provinceID, this.cityID}) : super(key: key);

  @override
  CountyPageWidgetState createState() => CountyPageWidgetState();
}

class CountyPageWidgetState extends State<CountyPageWidget> {
  // 在State中定义两个变量
  int _provinceID;
  int _cityID;
  List<County> _county;

  @override
  void initState() {
    // 在此将Widget中的两个变量赋值给State中的变量
    _provinceID = widget.provinceID;
    _cityID = widget.cityID;
    super.initState();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: new AppBar(
        title: Text(
          "区县",
          style: TextStyle(fontSize: 25.0),
        ),
      ),
      body: FutureBuilder( // UI异步更新组件
        // 这块就可以调用了
        future: Dio().get("http://guolin.tech/api/china/$_provinceID/$_cityID"),
        builder: (BuildContext context, AsyncSnapshot snapshot) {
        ...省略后面所有代码
}
```

# 3. UI异步更新
与异步更新相关的知识大家也可以看我们之前的第7章博客，这块只讲应用。

## 省
将Scaffold的body参数设置为FutureBuilder，并在FutureBuilder里面调用ListView:
```java
FutureBuilder(
    future: Dio().get("http://guolin.tech/api/china"),
    builder: (BuildContext context, AsyncSnapshot snapshot) {
        if (snapshot.connectionState == ConnectionState.done) {
            Response response = snapshot.data;
            //发生错误
            if (snapshot.hasError) {
                return Text(snapshot.error.toString());
            }

            provinceList = getProvinceList(response.data);
            //请求成功，通过项目信息构建用于显示项目名称的ListView
            return ListView.builder(
                itemCount: provinceList.length,
                itemBuilder: (context, index) {
                    return GestureDetector(
                        child: ListTile(
                            title: Text("${provinceList[index].name}"),
                        ),
                        onTap: () {
                            Navigator.push(context, MaterialPageRoute(builder: (context) {
                                return CityPageWidget(
                                    provinceID: provinceList[index].id,
                                );
                            }));
                        },
                    );
                },
            );
        }
        // 请求未完成时弹出loading
        return CircularProgressIndicator();
    },
)
```


## 市
与省同理:
```java
FutureBuilder(
    future: Dio().get("http://guolin.tech/api/china/$_provinceID"),
    builder: (BuildContext context, AsyncSnapshot snapshot) {
        if (snapshot.connectionState == ConnectionState.done) {
            Response response = snapshot.data;
            //发生错误
            if (snapshot.hasError) {
                return Text(snapshot.error.toString());
            }

            _cityList = getCityList(response.data);
            //请求成功，通过项目信息构建用于显示项目名称的ListView
            return ListView.builder(
                itemCount: _cityList.length,
                itemBuilder: (context, index) {
                    return GestureDetector(
                        child: ListTile(
                            title: Text("${_cityList[index].name}"),
                        ),
                        onTap: () {
                            Navigator.push(context, MaterialPageRoute(builder: (context) {
                                return CountiesPageWidget(
                                    provinceID: _provinceID,
                                    cityID: _cityList[index].id,
                                );
                            }));
                        },
                    );
                },
            );
        }
        // 请求未完成时弹出loading
        return CircularProgressIndicator();
    },
)
```



## 区县
与省同理:
```java
FutureBuilder(
    future: Dio().get("http://guolin.tech/api/china/$_provinceID/$_cityID"),
    builder: (BuildContext context, AsyncSnapshot snapshot) {
        if (snapshot.connectionState == ConnectionState.done) {
            Response response = snapshot.data;
            //发生错误
            if (snapshot.hasError) {
                return Text(snapshot.error.toString());
            }

            _county = getCountyList(response.data);
            //请求成功，通过项目信息构建用于显示项目名称的ListView
            return ListView.builder(
                itemCount: _county.length,
                itemBuilder: (context, index) {
                    return GestureDetector(
                        child: ListTile(
                            title: Text("${_county[index].name}"),
                        ),
                        onTap: () {
                            _saveCityID(_county[index].weatherId);
                            Navigator.push(context, MaterialPageRoute(builder: (context) {
                                return MainPage(
                                    cityID: _county[index].weatherId,
                                );
                            }));
                        },
                    );
                },
            );
        }
        // 请求未完成时弹出loading
        return CircularProgressIndicator();
    },
)
```
