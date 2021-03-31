---
title: Android编译变体
categories:
  - Android
tags:
  - Android
  - 编译
  - 编译变体
cover: 'https://cdn.littlecorgi.top/blog/Keynote_Blog_FINAL.png'
abbrlink: 9900843a
date: 2020-01-20 18:16:42
updated: 2020-01-20 18:16:42
description:
---
# 编译流程
我们都知道Android编译就是将我们自己写的Java文件编译成了.apk，但是他到底是如何将java编译成.apk的呢？

我们引入一张Google官方的流程图：
![](https://cdn.littlecorgi.top/mweb/2020-08-16/15975726163198.jpg)

我们可以看到，一个APP编译的大致流程就是这样的：
1. 先将项目的源代码编译成dex文件(Android虚拟机可识别的字节码文件，至于为啥不是.class，最根本的原因就是Android并没有直接使用Java的虚拟机。并且单个dex文件可被引用的方法总数被限制为65536，如果我们的项目过于庞大超出这个限制时，可能会被打包成多个dex文件)，其中这个源代码除了我们自己写的代码之外还包括我们导入的依赖库的代码；并将其他除源文件之外比如资源文件等等所有文件都转成编译的资源。
2. APK打包器将dex文件和编译的资源组合成单个APK，不过必须先为APK签名才能将打包的应用安装在设备上。
3. APK打包器通过调试或者发布密钥库为APK签名：
  1. 如果编译的版本是调试版本，打包器会使用调试密钥库为应用签名。AndroidStudio会自动使用调试密钥库配置新项目。
  2. 如果编译的版本是最终正式发布版本，则需要你自己提供签名信息，然后生成密钥用于打包。
4. 生成APK之前，Android首先会使用zipalign工具对应用进行优化。

# 认识Android项目结构
一般情况下，我们在新建Android项目时，AndroidStudio会默认为我们创建一下文件：
![](https://cdn.littlecorgi.top/mweb/2020-08-16/15975728777993.jpg)


其中最顶层是Project，而Project里面可以有多个module，module下面有sourceSet来存放我们编写的代码。
而我们对编译变体的操作主要就是在module的build.gradle中进行编辑。

# 配置版本类型
如果我们看一个AndroidStudio为我们创建的默认项目的app module的build.gradle，我们可以看到AndroidStudio默认为我们创建好的版本类型：
```
buildTypes {
    release {
        minifyEnabled true
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
    }
}
```
在buildTypes这个块中，又一个类型：release。

但是如果我们通过build variants进行查看的话，却会发现我们当前app这个module却又默认为debug这个类型，而如果我们打开选项框却又有debug和release两种类型。
![](https://cdn.littlecorgi.top/blog/%E7%BC%96%E8%AF%91%E5%8F%98%E4%BD%931.png)

我们明明没有配置这个debug类型，那这为什么会有debug类型供我们选择呢？

这是因为Android Studio会默认为我们的应用创建一个debug类型，以供我们可以方便的对我们的app进行安装调试。

但是如果我们需要对debug类型进行些设置的话，我们也可以在buildType中添加debug项，然后在debug块中进行设置。比如我们想让debug的包的应用程序ID后面添加一个"debug"后缀：
```
buildTypes {
    release {
        minifyEnabled true
        proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
    }

    debug {
        applicationIdSuffix ".debug"
        debuggable true
    }
}
```

这样我们如果需要生成debug的包，生成的包的应用程序ID后面就会多一个.debug的后缀。

除了可以设置应用程序ID之外，我们还可以设置是否生成可调式的apk(debugable)、是否启用Mutli-Dex等等(multiDexEnabled)。
具体buildTypes块中的可配置项，可以看此处-http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.BuildType.html

# 配置产品特性
创建产品特性和配置版本类型类似，也是在相应module的build.gradle中进行操作。

区别就是：
1. 版本类型故名思议就是打包成不同的版本以供不同类型调试的需求；而产品特性就是说我们可以根据我们应用业务上的需求将同一个项目打包成不同的产品，比方说免费版和付费版、普通版和lite版、国内版和海外版等等。
2. 版本类型是在android - buildTypes中进行配置，主要是对代码编译时的处理或者优化进行配置；而产品特性是在andorid - productFlabors中进行配置，主要是用于指出我们的整个项目可以分成哪几个子项目，并且对应的子项目的特有的代码该写在哪个源集中。

配置产品特性主要分两步走：flavorDimensions 和 productFlavors。
## flavorDimensions
这个属性主要是用来设定有哪几种类型维度。也就是说他指定的维度用于给下面的具体的产品特性进行归类。
```
flavorDimensions "location", "pay"
```
上面我就定义了两个维度，分别是
- 区分APP投放地区的location（如果在中国投放则是CN版，如果美国投放则是US，日本投放则是JP等等）
- 区分用户是否需要付费使用的pay（基础版也就是非付费版stand和多功能版也就是付费版pro）

## productFlavors
定义好了类型维度flavorDimensions，我们接下来就可以去定义估计的产品类型了。

基于我们上面定义的flavorDimensions。
我们首先需要location对用的类型：
```
flavorDimensions "location", "pay"

productFlavors {
    CN {
        dimension "location"
        resConfigs "zh"  //让该版本的语言只支持中文
    }
    US {
        dimension "location"
        resConfigs "en"  //让该版本的语言只支持英文
    }
```
这样我们就已经定义了两个属于location维度的类型，其中CN设定为只支持中文，US设定为只支持英文。这个时候在Sync一下，再打开Build Variants，就能看到如下信息：
![](https://cdn.littlecorgi.top/blog/%E7%BC%96%E8%AF%91%E5%8F%98%E4%BD%932.png)
              
            

分别对应着我们刚刚定义好的产品特性和版本类型，对每个产品特性，都有每个类型的版本类型。


然后我们再对pay进行配置：
```
android {
    compileSdkVersion 29
    buildToolsVersion "29.0.2"
    defaultConfig {
        //...省略
    }
    buildTypes {
        //...省略
    }

    flavorDimensions "location", "pay"
    productFlavors {
        CN {
            dimension "location"
            resConfigs "zh"
        }
        US {
            dimension "location"
            resConfigs "en"
        }
        free {
            dimension "pay"
        }
        pro {
            dimension "pay"
        }
    }
}
```
可以看到我们在之前的基础上，又新定义了两个产品类型free和pro，他两属于pay这个类型维度。具体的区别我们可以在代码中去体现。老规矩，Sync之后看build variants：
![](https://cdn.littlecorgi.top/blog/%E7%BC%96%E8%AF%91%E5%8F%98%E4%BD%933.png)
              
            

可以看到，现在我们的项目有8个build类型。
也就是我们设定的buildTypes和productFlavors排列组合组成的。

那这时可能会有个小问题，为什么是CNFreeDebug而不是FreeCNDebug或者DebugCNFree呢？

这块就涉及到一个优先级的问题。当gradle为每个编译变体或相应的APK命名时，属于较高优先级类型维度的产品特性会优先展示，然后再是较低的产品特性，最后才是版本类型。所以说，在flavorDimensions定义时，location定义在pay的前面，所以CN和US肯定在free和pro的前面。而版本类型在最后面，所以我们在buildTypes中定义的debug和release在最后面。

所以以上面的配置为例，我们总共有8个编译变体:
编译变体：[CN, US][free, pay][debug, release]

例如：
- 编译变体：CN free debug
- Apk：app-cn-free-debug.apk


## variantFilter
有些时候，我们可能会不需要某些编译变体，这个时候variantFilter就得派上用场了。

我们先来看他的定义方法：
```
      variantFilter { variant ->
          def names = variant.flavors*.name
          // To check for a certain build type, use variant.buildType.name == "<buildType>"
          if (names.contains("US") && names.contains("free")) {
              // Gradle ignores any variants that satisfy the conditions above.
              setIgnore(true)
          }
      }
    }
```

以上是Google官方给的例子，我就把产品特性名更改了下，简单来说就是当US和free组合在一起时，就忽略此编译变体。

# 源集
Android Studio 按逻辑关系将每个模块的源代码和资源分组为源集。模块的 main/ 源集包含其所有编译变体共用的代码和资源。其他源集目录是可选的，在您配置新的编译变体时，Android Studio 不会自动为您创建这些目录。不过，创建类似于 main/ 的源集有助于让 Gradle 只有在编译特定应用版本时才应使用的文件和资源井然有序：

- /src/main：所有的编译变体都会使用此源集
- /src/buildType：此源集用于编写特定版本的专用代码和资源
- /src/productFlavor：此源集用于编写特定编译类型专用的代码和资源
- /src/productFlavorBuildType：此源集用于编写特定编译变体特定的代码和资源

例如：
编译变体CNProRelease需要合并来自一下源集的代码和资源：
- /src/cnProRelease
- /src/release
- /src/pro
- /src/cn
- /src/main
- 依赖库
并且源集间的优先级就按从上到下的顺序排序，也就是说，cnProRelease的优先级是最高的，其次是release，其次再是产品特性，最后才是main和依赖库。所以说，如果这几个源集中有相同的变量或者资源，则以优先级高的源集中的为主。

编译变体 > 编译版本类型 > 正式版类型 > 主源集 > 库依赖项
