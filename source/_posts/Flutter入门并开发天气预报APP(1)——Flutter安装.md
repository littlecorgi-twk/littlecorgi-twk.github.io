---
title: Flutter入门并开发天气预报APP(1)——Flutter安装
categories:
  - Flutter
tags:
  - Flutter
  - Android
comments: true
copyright: true
abbrlink: 4bf666bf
date: 2019-10-08 19:55:33
updated: 2019-10-08 19:55:33
feature_img: 'https://cdn.littlecorgi.top/blog/Flutter.png'
cover: 'https://cdn.littlecorgi.top/blog/Flutter2.png'
keywords:
description:
---
# Flutter是什么
Flutter是由谷歌推出的一个移动UI框架，可以让开发者快速的在Android和iOS上构建高质量的原生用户界面。

<!-- More -->

通过它来编写APP的好处在于：
- 它内置了Android的Material Design风格和iOS的Cupertino风格的UI；
- 它通过Dart语言编写，只需要编写一份代码就能在Android和iOS设备上得到同样的运行效果；
- 他有热重载功能，只需要运行APP后，代码更改一点然后点击热重载键就能快速显示出修改后的界面。

# 使用镜像安装
由于一些众所周知的原因，Flutter在国内访问有时候会受到限制，因此Flutter官方为国内开发者搭建了临时镜像，大家可以将下面的环境变量添加到系统中：
```bash
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
```
由于我用的是macOS，因此我就只演示macOS的。
如果有Linux的用户则和macOS 的安装方式类似，甚至可以说一样；
但是对于Windows用于，需要大家桌面右键此`电脑->属性->弹出页面左侧栏的高级系统设置->环境变量`，在下面的系统环境变量栏中点击新建，变量名输入上面等号左边的大写内容，不要export，值输入等号右边的内容，输入一组之后再点击新建接着输入下一组。
![Windows下的环境变量配置](https://img-blog.csdn.net/20180616140532293?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ExMjAzOTkxNjg2/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

接下来我们开始说macOS。
## 首先你得确定你用的是什么shell
如果你没更改过你macOS终端设置，那么你默认就是`bash`，但是如果你更改过，比方说我就换成了`zsh`，那么我觉得你应该知道在哪设置环境变量。
## 如果你是bash
首先打开你macOS的终端，在启动台里面可以找到，接着对`bash`的环境变量进行设置，输入
```
vim ~/.bash_profile
```
回车，进入`.bash_profile`文件的编辑页面，然后按下键盘`I键`，进入vim的编辑模式，然后直接把上面的export环境变量赋值粘贴进去。然后按下`Esc键`退出vim的编辑模式，然后直接输入`:wq`回车保存并退出。
接着输入
```
source ~/.bash_profile
```
让刚刚配置的环境变量生效即可。
![bash配置环境变量](https://cdn.littlecorgi.top/mweb/2019-10-08/bash%E9%85%8D%E7%BD%AE%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F.png)

## 如果你是zsh
首先打开你macOS的终端，在启动台里面可以找到，接着对`zsh`的环境变量进行设置，输入
```
vim ~/.zshrc
```
回车，进入`.zshrc`文件的编辑页面，找到如图所示部分（重点在于有`# export MANPATH="/Usrs/......`的地方，因为我还有其它的环境变量，所以你的可能和我的不同）：
![zshr](https://cdn.littlecorgi.top/mweb/2019-10-08/zshrc.png)


在`# export`下面，按下键盘`I键`，进入vim的编辑模式，然后直接把上面的export环境变量赋值粘贴进去。
见我图中最下面标红的那地方，至于最后一行`export PATH=~/development/flutter/bin:$PATH`先不管。然后按下`Esc键`退出vim的编辑模式，然后直接输入`:wq`回车保存并退出。
接着输入
```
source ~/.zshrc
```
让刚刚配置的环境变量生效即可。
![zsh配置环境变量](https://cdn.littlecorgi.top/mweb/2019-10-08/zsh%E9%85%8D%E7%BD%AE%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F.png)

# 下载Flutter SDK
## 先去Flutter的官网去下载你对应系统的Flutter SDK
1. 打开官网下载页：[Flutter SDK release](https://flutter.dev/docs/development/tools/sdk/releases)
2. 选择你的系统对应的目录，然后默认选择Stable channel下载。(*Master channel这个是直接从github获取的github上的最新版，也就是说Flutter的开发者他写了点什么改了点什么都会里面在这个版本上显示出来，所以这个版本意味着非常不稳定，可能bug很多；Dev channel是开发版，Master channel经过一定测试之后的版本会发布到Dev channel上，但是不是大量的测试，所以可能还是不太稳定；Beta channel是测试版，每月发布一次，会把上个月所释放的所有Dev channel中最稳定的版本释放到Beta channel上，但是相对来说还是有点不稳定，适合想尝鲜的开发者；State channel是标准版，当Flutter维护人员认为某个版本足够稳定之后，就会发送到这个版本上去，一般开发者最好还是使用这个版本)

## 解压安装包到你想要的目录
例如
```
// Linux/macOS
cd ~/development
unzip ~/Downloads/flutter_macos_v0.5.1-beta.zip
```

## 添加Fluuter路径到环境变量中
回顾第2项，把如下内容添加到环境变量中去
```bash
export PATH=/你上一步flutter解压的目录/flutter/bin:$PATH
```
## 运行Flutter doctor
如果是macOS/Linux，直接在终端中输入`flutter doctor`即可。
如果是Windows，则右键开始，选择PowerShell，然后输入`flutter doctor`即可。

这个命令的功能是检查你的Flutter是否正常安装。

如果正常安装的话会出现如下信息：
```bash
Doctor summary (to see all details, run flutter doctor -v):
[✓] Flutter (Channel stable, v1.9.1+hotfix.4, on Mac OS X 10.13.6 17G65, locale zh-Hans-CN)
 
[✓] Android toolchain - develop for Android devices (Android SDK version 29.0.1)
[✗] Xcode - develop for iOS and macOS
    ✗ Xcode installation is incomplete; a full installation is necessary for iOS development.
      Download at: https://developer.apple.com/xcode/download/
      Or install Xcode via the App Store.
      Once installed, run:
        sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
    ✗ CocoaPods not installed.
        CocoaPods is used to retrieve the iOS and macOS platform side's plugin code that responds to
        your plugin usage on the Dart side.
        Without CocoaPods, plugins will not work on iOS or macOS.
        For more info, see https://flutter.dev/platform-plugins
      To install:
        sudo gem install cocoapods
        pod setup
[✓] Android Studio (version 3.5)
[!] IntelliJ IDEA Community Edition (version 2019.2.3)
    ✗ Flutter plugin not installed; this adds Flutter specific functionality.
    ✗ Dart plugin not installed; this adds Dart specific functionality.
[✓] VS Code (version 1.38.1)
[✓] Connected device (1 available)

! Doctor found issues in 2 categories.
```

我的电脑运行之后会出现两个错误，因为Flutter doctor会自动检查电脑中所有能运行Flutter的编辑器或者IDE，然后判断他们配没配置Flutter。我的电脑是因为没有安装Xcode以及我没有在IDEA中配置Flutter，所以我这两项出错。

其实你只要保证除了IDE之外的项目都是正常的就行了，如果有错的话直接把报错信息放到百度里面进行搜索就可以了。至于IDE项目，如果你是iOS开发者，则保证Xcode或者是AppCode啥的前面是✔️就行了的，至于你电脑上的其它例如IDEA或者vscode啥的可以不用管；如果你是Android开发者，则保证你常用的IDE如Android Studio前面是✔️就行了的，至于你电脑上的其它例如IDEA或者vscode啥的可以不用管。

总而言之，确保除了IDE之外的其它项前面是对勾，然后确定一个你常用的写代码的工具前面有对勾就行了，如果你除了IDE之外的其它项有问题，就把错误信息直接百度；如果是你常用的IDE前面没对勾，那就百度你常用的IDE+Flutter来参考别人的配置教程。

---
*由于我没有用过xcode所以就不讲xcode如何配置Flutter了，就分别讲如何在AndroidStudio和vscode配置Flutter。*

---

# Android Studio配置Flutter
1. 打开你Android Studio的`设置/首选项/Preferences`，然后选择`Plugins`，在`Marketplace`里面搜索`Flutter`：    ![Androidstudio Flutte](https://cdn.littlecorgi.top/mweb/2019-10-08/Androidstudio%20Flutter.png)

2. 接着点击`INSTALL`安装就行了，并且安装Flutter插件会自动顺带安装Dart插件，如果没有安装Dart插件，你就一样的搜索`Dart`然后安装即可；
3. 安装完成并重启Android Studio之后，仍然打开`设置/首选项/Preferences`，选择`Languages & Frameworks`->`Flutter`，在最上面的SDK中的`Flutter SDK path`选择你第三步Flutter SDK解压缩的路径，不出意外的话下面`Version`会自动显示你Flutter SDK的版本，并且`Languages & Frameworks`->`Dart`里面的`Dart SDK path`也会自动出现。
    ![Android Studio Flutter Setting](https://cdn.littlecorgi.top/mweb/2019-10-08/Android%20Studio%20Flutter%20Setting.png)
4. 接着和之前一样运行`flutter doctor`,看看Android Studio前面有没有✔️，正常情况下是会有的。

# vscode 配置Flutter
1. 在vscode拓展里面搜索Flutter和Dart，安装；![vscode Flutte](https://cdn.littlecorgi.top/mweb/2019-10-08/vscode%20Flutter.png)

2. 接着和之前一样运行`flutter doctor`,看看vscode前面有没有✔️，正常情况下是会有的。