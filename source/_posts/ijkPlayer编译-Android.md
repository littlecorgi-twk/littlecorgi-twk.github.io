---
title: ijkPlayer编译-Android
categories:
  - Android
tags:
  - Android
  - ijkplayer
  - 音视频
comments: true
copyright: true
abbrlink: d61d5542
date: 2019-08-06 19:51:28
updated: 2019-08-06 19:51:28
---
# 1. 简介
ijkplayer是哔哩哔哩的一个开源视频播放框架，支持Android、iOS。底层是ffplay。

Github地址：[bilibili/ijkplayer](https://github.com/bilibili/ijkplayer)
<!-- More -->

# 2. 编译方法
由于通过Gradle编译起来很慢而且一旦失败又得重头来，所以这块就使用AndroidNDK的方式来编译。

## 2.1 编译之前
首先你得配置好等会编译需要的东西。这块我们都会使用[Homebrew](https://brew.sh/index_zh-cn)来安装git和yasm。
Homebrew类似于Ubuntu的dpkg、RedHat和centOS的yum，他是macOS上的一个软件包管理器。但是后来出了Linux版。
由于ijkplayer官方说用的它，那咱们就用它吧。

### Ubuntu
#### 1. 先把目前已有的包更新
```bash
// 从镜像站下载软件列表，并检查有没有需要更新的包
sudo apt update

// 更新需要更新的包
sudo apt upgrade

// 自动卸载掉当初为了安装其他软件或其他原因而安装但是目前已经没用的包
sudo apt autoremove
```

#### 2. 安装Homebrew


```bash
1. 首先确定是否安装ruby  
    apt install ruby'
    
2. 然后通过ruby安装Homebrew  
    ruby -c "$(curl -fsSL https://raw.githubusercontent.com/Linuxbrew/install/master/install.sh)"
    
3. 再通过Homebrew安装git和yasm
    brew install git
    brew install yasm
```

#### 3. 配置Android SDK和NDK
1. Android SDK
    你可以打开你的AndroidStudio，进入`Preferences -> Appearance & Behavior -> System Settings -> Android SDK`。这块就会显示你SDK安装目录。
    然后在终端中输入`vim ~/.bash_profile`。然后输入以下内容：
    
    ```bash
    export ANDROID_SDK_HOME=AndroidSDK的目录/android-sdk-linux
    export PATH=$PATH:${ANDROID_SDK_HOME}/tools
    export PATH=$PATH:${ANDROID_SDK_HOME}/platform-tools
    ```
    保存，然后再在终端输入`source ~/.bash_profile`，就可以了。
2. Android NDK
    首先你得到官网下载NDK：[NDK 归档](https://developer.android.com/ndk/downloads/older_releases.html?hl=zh-cn)。而且你只能下载r10e版本的，下载其它版本编译会报错。
    下载下来解压后，同样和上面一样得配置环境变量：
    
    ```bash
    export ANDROID_NDK_HOME=NDK目录
    export PATH=$PATH:${ANDROID_SDK_HOME}
    ```
    保存，然后再在终端输入`source ~/.bash_profile`，就可以了。

### macOS
安装Homebrew，由于macOS自带ruby，所以我们直接可以开始安装Homebrew。
```bash
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
brew install git
brew install yasm
```
然后就是配置你的Android SDK和Android NDK：
和上面Ubuntu下一样。

## 2.2 编译Android

```bash
git clone https://github.com/Bilibili/ijkplayer.git ijkplayer-android
cd ijkplayer-android
git checkout -B latest k0.8.8

./init-android.sh

cd android/contrib
./compile-ffmpeg.sh clean
// 这步特慢，耗时很长
./compile-ffmpeg.sh all

cd ..
./compile-ijk.sh all
```
上面这些执行完就可以了。
然后你就能在你的`/ijkplayer-android/android/ijkplayer`路径下就能看到编译后的项目了，其中每个`module的/src/mian/libs`里面就是so文件。

# 3. 最后
你可以直接通过AndroidStudio打开`/ijkplayer-android/android/ijkplayer`，然后在里面的`/ijkplayer-example`下谢你自己的代码就可以了。