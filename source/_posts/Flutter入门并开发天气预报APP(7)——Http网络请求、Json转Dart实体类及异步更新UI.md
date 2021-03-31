---
title: Flutter入门并开发天气预报APP(7)——Http网络请求、Json转Dart实体类及异步更新UI
categories:
  - Flutter
tags:
  - Flutter
  - Android
comments: true
copyright: true
abbrlink: 163d319c
date: 2019-10-12 22:13:21
updated: 2019-10-12 22:13:21
feature_img: 'https://cdn.littlecorgi.top/blog/Flutter.png'
cover: 'https://cdn.littlecorgi.top/blog/Flutter2.png'
keywords:
description:
---

> 相关Demo源码可见Github [a1203991686/CoolWeather_Flutter](https://github.com/a1203991686/CoolWeather_Flutter)

<!--More-->
# Flutter Http 网络请求
Flutter网络请求可分为两种方式，一种为`Dart:IO`库中为我们提供的`HttpClient`，另一种为Dart第三方库`Dio`。

## HttpClient
`HttpClient`是`Dart:IO`库自带的一个类，通过他来实现网络请求最底层、也可以完完全全的自定义设置，但是相对于其他人封装好的第三方库来说显得复杂。

### 引入
导入`Dart:IO`库：
```java
import 'dart:io';
```

### 创建一个HttpClient:
```java
 HttpClient httpClient = new HttpClient();
```

### 创建一个Uri
如果你是`Http`请求：
```java
var uri = new Uri.http(
    host,
    queryParameters: {
        "xx":"xx",
        "yy":"dd"
    }
);
```
如果你是`Https`请求：
```java
var uri = new Uri.https(
    host,
    queryParameters: {
        "xx":"xx",
        "yy":"dd"
    }
);
```
### 根据uri获取返回数据
```java
HttpClientRequest request = await httpClient.getUrl(uri);
HttpClientResponse response = await request.close();
```

### 读取内容
进行这一步得导入`dart:convert`库：
```java
import 'dart:convert';

... //省略中间代码 

String responseBody = await response.transform(Utf8Decoder()).join();

print(respinseBody)
```
### 最后关闭Client
```java
httpClient.close();
```

## Dio
`Dio`是Dart社区上别人上传的第三方库，他封装了`HttpClient`，相较来说更为简单、方便。

### 引入
在项目的`pubspec.yaml`文件的`dependencies`中导入`Dio`：
```java
dependencies:
  dio: 
  // 如果冒号后面不带具体版本信息则表示自动下载最新版
```

接着在需要使用的代码文件里面，`import`导入他：
```java
import 'package:dio/dio.dart';
```
接着我们就可以使用`Dio`了。

### 示例
首先得创建一个`Dio`对象。
```java
Dio dio =  Dio();
```

发起`GET`请求：
```java
Response response;
response=await dio.get(uri)
print(response.data.toString());
```

发起`POST`请求：
```java
response=await dio.post(uri);
```

# Json转Dart
##  手动生成Dart实体类
首先得大家安利一个网站，因为Dart实体类比Java实体类多了几个方法，所以相对来说麻烦，通过这个网站就可以自动生成json数据对应的实体类:
[https://caijinglong.github.io/json2dart/](https://caijinglong.github.io/json2dart/)

![json2dart](https://cdn.littlecorgi.top/mweb/2019-10-12/json2dart.png)

比方说我们使用这个链接(http://guolin.tech/api/china)，并让他自动转dart：
![Province](https://cdn.littlecorgi.top/mweb/2019-10-12/Province.png)

接下来只需要创建dart文件，然后再把代码复制进去就行了。

不出意外，代码肯定会报错。
![dart实体类](https://cdn.littlecorgi.top/mweb/2019-10-12/dart%E5%AE%9E%E4%BD%93%E7%B1%BB.png)

报错信息为:
```java
error: Target of URI hasn't been generated: 'province.g.dart'. (uri_has_not_been_generated at [cool_weather] lib/bean/province.dart:3)

error: The method '_$ProvinceFromJson' isn't defined for the class 'Province'. (undefined_method at [cool_weather] lib/bean/province.dart:31)

error: The method '_$ProvinceToJson' isn't defined for the class 'Province'. (undefined_method at [cool_weather] lib/bean/province.dart:33)
```

这是因为我们项目下还没有`province.g.dart`这个文件，这个文件是根据dart实体类自动生成的，那么我们该怎么生成他呢？

这个时候我们需要在`pubspec.yaml`文件中导入三个依赖包：
![转dart](https://cdn.littlecorgi.top/mweb/2019-10-12/%E8%BD%ACdart.png)
我们重点只需要管三个写着需要导入的。输入之后点击`pubspec.yaml`文件右上角的`packages get`，就会自动下载包了。

然后在终端中，转到项目根目录下，输入
```
flutter packages pub run build_runner build
```
接着你就会惊喜的发现，在你的dart实体类下面多了一个文件，也就是你所缺失的~.g.dart文件，并且实体类中那些报错也都没了。
![dart.g.dart](https://cdn.littlecorgi.top/mweb/2019-10-12/dart.g.dart.png)

## 将请求回来的Response转为Dart实体类

接着HttpClient返回来的responseBody或者Dio返回的response

```java
List<Province> provinceList;
provinceList = ProvinceList.getProvinceList(jsonResponse);
```

provinceList就是返回来的数据转成的实体类的对象了。

# 异步更新UI
在此处介绍两种方法，第一种是我自己想出来的沙雕方法，第二种是通过Flutter提供的专门用于异步更新UI的组件FutureBuilder。
## 沙雕方法
在此说一下我这个方法的想法。

大家可以回顾下我之前讲Widget的时候说过，Widget有一个方法initState可以用来加载UI，以及一个setState方法可用来提醒Flutter重新加载UI。

说到这很多同学一定想到了，我们可以在获取到数据之后通过setState方法来更新UI。

### 获取到数据
这个不同多说，上面讲的全都是获取数据。

### 设置一个空的实体类
此处我还是拿上面的Province.dart来做例子。我们正常情况下获取到的数据转成的实体类的对象是`List<Province> provinceList`。所以我们此处先设置一个空的对象:`List<Province> _provinceList`。

接着在initState里面调用网络请求的方法：
```java
@override
void initState() {
    getProvince();
    super.initState();
}
```

然后在getProvince方法里面获取数据，并将数据provinceList传给_provinceList：

```java
void getProvince() async {
    var response = await Dio().get("http://guolin.tech/api/china");
    provinceList = ProvinceList.getProvinceList(response);
    setState(() {
        _provinces = provinceList;
    });
}
```

这样就可以当获取到数据的时候就通知Flutter更新状态了。这个时候有的同学可能会问我，你这个也只是获取到数据的时候才更新啊，那当还没获取到数据的时候呢？

这块就体现到了为什么我们要设置一个_provinces了。

我们完全可以在build方法设置下，判断下_provinces为不为空:如果为空，就证明没数据，就加载其它页面；如果不为空，就证明有数据了，那就加载数据。
```java
body: _provinces == null
          ? new Text("正在请求")
          : new ListView.builder(
          ... //省去代码，总的来说就是将_provinces的数据加载ListView里面，一个ListView的基本构造方法
          )
```

## FutureBuilder
回归正题，我上面那个方法真的是有够沙雕的。

那我们来看看这个名正言顺的Flutter亲生儿子FutureBuilder。

首先我们看下定义：
```java
FutureBuilder({
  this.future,
  this.initialData,
  @required this.builder,
})
```
- future：异步任务；
- initialData：初始化数据；
- builder：builder方法。

### 示例
```java
body: FutureBuilder(
    future: Dio().get("http://guolin.tech/api/china"),
    builder: (BuildContext context, AsyncSnapshot snapshot) {
        if (snapshot.connectionState == ConnectionState.done) {
            Response response = snapshot.data;
            print(response.toString());
            //发生错误
            if (snapshot.hasError) {
                return Text(snapshot.error.toString());
            }

            provinceList = ProvinceList.fromJson(response.data);
            //请求成功，通过项目信息构建用于显示项目名称的ListView
            return ListView.builder(
                itemCount: provinceList.provinces.length,
                itemBuilder: (context, index) {
                    return GestureDetector(
                        child: ListTile(
                            title: Text("${provinceList.provinces[index].name}"),
                        ),
                        onTap: () {
                            Navigator.push(context,
                                MaterialPageRoute(builder: (context) {
                                    return CityPageWidget(
                                        cityID: provinceList.provinces[index].id);
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