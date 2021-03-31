---
title: Aria2macOS安装
categories:
  - 杂项
tags:
  - Aria2
comments: true
copyright: true
abbrlink: 4f46fdbb
date: 2019-08-02 10:13:16
updated: 2019-08-02 10:13:16
---
> 对不起大家，在我配置好这个软件之后，我兴冲冲的打开了github想下载下ssr4.0forWindows，迫不及待想感受下aria2的快感，结果发现我错了。  
> 下载速度就才几十k/s，那一瞬间，我感触很多，我突然想到我昨晚的迅雷下载同样的资源同样的网同样的vpn下大几百k/s 的速度，然后又看到了此时我给予希望的aria，我陷入了沉思，或许这就是人生吧。  
> 突然觉得macOS迅雷，也还挺好看的，也挺好用的，完全没有广告，要下载打开，下载完了就退出，唯一问题就是Chrome的迅雷拓展似乎太灵敏了点，我啥都没点就莫名其妙弹出下载页面，除了这个，似乎非常完美。  
> 所以，在我配置好aria2并写下这篇博客后的不到1个小时时间，我卸载了aira2，并重新用上了迅雷。所以，我最后只想说一句，财大nb！财大nb！财大nb！对不起，走错片场了，重来，IDM牛逼！IDM牛逼！IDM牛逼！  
>如果哪位牛逼的大哥看到了这篇博客，记得帮我给IDM说声，一个Windows的IDM正版用户急需IDM macOS版！！！

<!-- More -->
# 下载Aira2
## 通过github安装
- 打开github主页 [aria2/aria2-Release](https://github.com/aria2/aria2/releases)
- 找到对应的系统下载安装即可

## 通过Homebrew安装
终端输入命令`brew install aria2`安装即可。
没有安装Homebrew的同学可以搜索安装Homebrew即可。

> 这个软件是没有图形界面的，所以安装好之后在启动台里面是找不到Aria2的图标的

# 配置Aira2
## 先创建Aria2的配置文件
Aria2提供了两种工作模式：
1. 直接命令行模式下载
    不太推荐这种，因为命令行下下载比较繁琐，而且也不太好操作
2. RPC模式
    在这种方式，Aria2启动之后就会以后台的方式运行，你就可以通过WebUI或者安装客户端的方式来使用图形界面控制Aria2.
但是这种方式下需要配置文件，现在就告诉大家如何配置：
首先创建文件

```linux
cd ~
mkdir /.aria2
cd /.aria2
touch aria2.conf
```

然后打开Finder，使用快捷键`Shift+Cmd+G`弹出路径输入框，接着复制粘贴`~/.aria2/`回车就进入了`.aria2`文件夹，你就会看到里面的`aria2.conf`文件，接着用文本编辑器打开，复制粘贴以下内容
```
#用户名
#rpc-user=user
#密码
#rpc-passwd=passwd
#上面的认证方式不建议使用,建议使用下面的token方式
#设置加密的密钥
#rpc-secret=token
#允许rpc
enable-rpc=true
#允许所有来源, web界面跨域权限需要
rpc-allow-origin-all=true
#允许外部访问，false的话只监听本地端口
rpc-listen-all=true
#RPC端口, 仅当默认端口被占用时修改
#rpc-listen-port=6800
#最大同时下载数(任务数), 路由建议值: 3
max-concurrent-downloads=5
#断点续传
continue=true
#同服务器连接数
max-connection-per-server=5
#最小文件分片大小, 下载线程数上限取决于能分出多少片, 对于小文件重要
min-split-size=10M
#单文件最大线程数, 路由建议值: 5
split=10
#下载速度限制
max-overall-download-limit=0
#单文件速度限制
max-download-limit=0
#上传速度限制
max-overall-upload-limit=0
#单文件速度限制
max-upload-limit=0
#断开速度过慢的连接
#lowest-speed-limit=0
#验证用，需要1.16.1之后的release版本
#referer=*
#文件保存路径, 默认为当前启动位置
dir=/Users/xxx/Downloads
#文件缓存, 使用内置的文件缓存, 如果你不相信Linux内核文件缓存和磁盘内置缓存时使用, 需要1.16及以上版本
#disk-cache=0
#另一种Linux文件缓存方式, 使用前确保您使用的内核支持此选项, 需要1.15及以上版本(?)
#enable-mmap=true
#文件预分配, 能有效降低文件碎片, 提高磁盘性能. 缺点是预分配时间较长
#所需时间 none < falloc ? trunc « prealloc, falloc和trunc需要文件系统和内核支持
file-allocation=prealloc

#开启BT下载
enable-dht=true
bt-enable-lpd=true
enable-peer-exchange=true
# bt-tracker 更新，解决Aria2 BT下载速度慢没速度的问题
udp://tracker.coppersurfer.tk:6969/announce,http://tracker.internetwarriors.net:1337/announce,udp://tracker.opentrackr.org:1337/announce,udp://9.rarbg.to:2710/announce,udp://9.rarbg.me:2710/announce,udp://tracker.openbittorrent.com:80/announce,http://tracker3.itzmx.com:6961/announce,http://tracker1.itzmx.com:8080/announce,udp://exodus.desync.com:6969/announce,udp://retracker.lanta-net.ru:2710/announce,udp://tracker.torrent.eu.org:451/announce,udp://tracker.tiny-vps.com:6969/announce,udp://bt.xxx-tracker.com:2710/announce,udp://tracker2.itzmx.com:6961/announce,udp://tracker.cyberia.is:6969/announce,udp://open.demonii.si:1337/announce,udp://explodie.org:6969/announce,udp://denis.stalker.upeer.me:6969/announce,udp://open.stealth.si:80/announce,http://tracker4.itzmx.com:2710/announce
```

不用更改啥，也不建议更改啥，大家可以看一下上面的配置内容，至少记得每一项大致是个啥，因为等会配置图形界面需要这些信息。

最后面的BT下载的`bt-tracker`大家配置记得到这个页面更新：https://raw.githubusercontent.com/ngosang/trackerslist/master/trackers_best.txt

然后再把内容复制粘贴过去。

下载路径你可以自己选择，可以就把`/Users/xxx/Downloads`中的`xxx`换成你自己的macOS用户名即可。

# 启动Aria2 RPC模式
终端输入`aria2c --conf-path="/Users/xxx/.aria2/aria2.conf" -D`就可以启动了。一样的，xxx是你的用户名。

# 进入图形界面
Aria2有很多第三方图形界面，客户端有，但是我没用过所以我不做介绍，在这主要介绍WebUI。

## YAAW插件
1. 在chrome的拓展中心搜索`YAAW`，然后安装即可，接着就会出现YAAM的插件
2. 点击打开它，就会进入YAAW的WebUI。
![](https://cdn.littlecorgi.top/mweb/15647112624749.jpg)
3. 点击右上角的🔧图标，进入设置界面
![](https://cdn.littlecorgi.top/mweb/15647113330438.jpg)
4. 如果你上面的`aria2.conf`是直接复制粘贴的我的话，照着我图中的填写就可以了，然后就可以使用了。找到需要下载的资源右键然后选择YAAW就会自动开始下载。

## AriaNg WebUI
这个比上面那个好看点，缺点是没有插件，你只能复制下载链接手动新建下载任务
1. 在浏览器中输入网址 `http://ariang.mayswind.net/latest/#!/downloading`
![](https://cdn.littlecorgi.top/mweb/15647115205711.jpg)


2. 点击AriaNg设置
![](https://cdn.littlecorgi.top/mweb/15647115637006.jpg)
3. 一样，如果你上面的`aria2.conf`是直接复制粘贴的我的话，照着我图中的填写就可以了，然后就可以使用了。
4. 你可以点击Aria2状态，看看是不是已连接，如果不是就看看有没有哪儿和`aria2.conf`设置的不一样。
![](https://cdn.littlecorgi.top/mweb/15647116500131.jpg)

# 关闭Aria2
这个好像官方没有方法关闭，于是我们就用最简单粗暴的方法。
1. 终端输入`ps aux|grep aria2`得到Aria2的进程号
2. 然后输入`kill number`调用系统命令直接杀死这个进程，`number`改为上面的到的进程号即可。

# 配置自启动
## 创建sh文件
进入到希望保存的目录下，新建一个文件`aria2.sh`：
```
touch aria2.sh
```
然后输入下面的代码并保存：
```
#!/bin/bash
echo "start aria2 server"
aria2c --conf-path="/Users/xxx/.aria2/aria2.conf" -D
echo "exiting"
exit
```
## 修改文件权限
1. 给`aria2.sh`文件执行权限：`chmod +x aria2.sh`
2. 让`aria2.sh`默认用自己常用的terminal工具打开。右键文件 －> 显示简介：设置“打开方式->所有应用程序”为自己的terminal即可。

## 添加到开机启动项
1. 在`Mac`桌面顶部菜单中，点击苹果图标，在弹出的菜单中，点击进入系统偏好设置。
2. 在打开系统偏好设置后，然后点击进入用户与群组设置选项。
3. 然后在用户与群组设置界面，先在左侧选择登陆用户-当前用户，然后在右侧切换到登录项
4. 然后点下面的+进行添加，选择刚才我们创建的文件aria2.sh，并勾选隐藏。
这样 aria2 就可以在每次开机的时候自启动了。

### 最后，Aria2卸载方式
终端下执行`sudo pkgutil --forget aria2`以及`sudo pkgutil --forget aria2.path`，然后上面创建的`.aria2`文件夹