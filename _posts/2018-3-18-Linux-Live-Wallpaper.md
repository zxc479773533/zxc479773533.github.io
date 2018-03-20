---
layout: post
title: 两种方法实现Linux下的动态桌面
tags:
- Beautify
- Arch
categories: Linux
---

最近打算给Linux换一个桌面，突然想到我在Windows下用的是Wallpaper Engine的动态桌面，简直美爆。于是就开始了这方面的折腾。

## 基本原理

Wallpaper Engine下载的文件放在`steam/steamapps/workshop/content/431960`文件夹内，我们可以从中提取出一些动态桌面文件，大概有如下三种格式：

* html
* pkg格式
* 视频文件

因此只需要想办法在Linux下，让某种媒体文件在桌面的后层播放即可。

经过一些尝试并没有找到让前两种文件播放的方法，于是我从简单的视频文件来着手。

## 轻量级播放器xwinwrap

一开始确实是想使用`VLC`的直接将视频置底的功能，执行：

```
$ cvlc --video-wallpaper --no-audio [video path]
```

然后发现这根本就是一个假的wallpaper，只是单纯将一个视频全屏播放并置底而已。

后来在网上找到了`xwinwrp`这个软件，它是一个非常轻量的程序，源码甚至只有700行左右，主要的功能是播放视频。

你可以直接用源码安装： https://github.com/zxc479773533/xwinwrap

Archlinux用户可以在aur中直接安装：

```
yaourt -S shantz-xwinwrap-bzr
```

xwinwrp的几个主要的参数如下：

```
 -g      - Specify Geometry (w=width, h=height, x=x-coord, y=y-coord. ex: -g 640x480+100+100)
 -ni     - Ignore Input
 -d      - Desktop Window Hack. Provide name of the "Desktop" window as parameter
 -argb   - RGB
 -fs     - Full Screen
 -s      - Sticky
 -st     - Skip Taskbar
 -sp     - Skip Pager
 -a      - Above
 -b      - Below
 -nf     - No Focus
 -o      - Opacity value between 0 to 1 (ex: -o 0.20)
 -loop   - loop times (0 means infinite)
 -sh     - Shape of window (choose between rectangle, circle or triangle. Default is rectangle)
 -ov     - Set override_redirect flag (For seamless desktop background integration in non-fullscreenmode)
 -debug  - Enable debug messages
```

这里可以看到，关键的参数是`-b`和`-fs`实现全屏和在后层播放视频。

一个有效的例子如下：

```
xwinwrap -ni -o 1.0 -fs -s -st -sp -b -nf -- mplayer -wid WID -quiet -nosound -loop 0 [video path]
```

但是xwinwrap的缺点也很明显，运行之后会把顶栏给遮掉，而且有时可能会卡住，总的来说体验一般，优点在于程序轻量，资源占用很少。

运行时的效果如下：

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Linux-Live-Wallpaper-01.png)

## 更好用的komorebi

在网上经过一番查找后又发现了一个好用的可以在底层播放视频的软件`komorebi`，github地址如下：https://github.com/iabem97/komorebi

`komorebi`是一个桌面壁纸管理器，下载后自带一些壁纸/动态壁纸，并且自带一个桌面时钟，可以尝试一下看看效果。

`komorebi`在安装好后会自带一个`Wallpaper Creator`，但是经过我使用发现这东西是个假的，还存在些许bug。

因此，我找到了其存放素材的地点和桌面的配置文件，下面我给出添加桌面的方法。

`komorebi`默认的壁纸素材存放在`/System/Resources/Komorebi/`目录下，你需要先找好一个视频[mp4格式]，一个缩略图[jpg格式]，然后在该目录下创建一个文件夹，自行命名即可。至于视频的来源，你可以在Windows下提取Wallpaper Engine下载的文件，利用Pr/格式工厂等软件去掉声音，重新渲染，或者自行剪辑视频，注意这里选取的视频不要太长太大即可。这里以创建`霞ヶ丘詩羽`的一个动态桌面示例。

```
cd /System/Resources/Komorebi/
sudo mkdir kasumigaoka_Utaha
```

接下来将动态桌面的视频和缩略图放在文件里，其中缩略图必须命名为`wallpaper.jpg`，否则会报错(如果利用自带的Wallpaper Creator，生成的文件名不对)。

```
cd kasumigaoka_Utaha
sudo mv ~/kasumigaoka_Utaha.mp4 kasumigaoka_Utaha.mp4
sudo mv ~/kasumigaoka_Utaha.jpg wallpaper.jpg
```

然后创建配置文件`config`，具体的解释如下：

```
sudo vim config
```

```
[Info]
WallpaperType=video // 壁纸类型，有image和video两种
VideoFileName=Kasumigaoka_Utaha.mp4 // 视频路径

[DateTime]
Visible=true // 桌面时钟是否可见
Parallax=false // 视是否有视差
MarginTop=0 // 上侧边距
MarginRight=0 // 右侧边距
MarginLeft=40 // 左侧边距
MarginBottom=25 // 下侧边距
RotationX=0 // x轴旋转
RotationY=0 // y轴旋转
RotationZ=0 // z轴选择
Position=bottom_left // 时钟的位置，有"top/center/bottom" + "_" + "left/right/center"九种情况
Alignment=center //字体对齐方式，有left/right/center三种
AlwaysOnTop=true //始终在上方
Color=#dd22dd22dd22 // 时钟字体颜色
Alpha=255 // 时钟字体透明度
ShadowColor=#dd22dd22dd22 // 字体描边颜色
ShadowAlpha=255 // 字体描边透明度
TimeFont=Lato Light 40 // 时间字体和大小
DateFont=Lato Light 26 // 日期字体和大小
```

这个时候，对着桌面右键更换桌面，应该就可以看到我们创建的动态桌面了。

![](https://github.com/zxc479773533/zxc479773533.github.io/raw/master/_posts/images/Linux-Live-Wallpaper-02.gif)

## 附:Linux下字体安装

在配置桌面时钟时可能会用到一些自己喜欢的好用的字体，比如一些萌系字体之类的，这里再附加一个Linux下安装字体的方法。

你可以从网上下载很多你想要的字体，这里以海报中常用的圆滑的”华康少女字体“为例。

```
sudo mkdir /usr/share/fonts/FrLt_DFGirl
sudo cp 你的字体文件  /usr/share/fonts/FrLt_DFGirl
sudo chmod 644 /usr/share/fonts/FrLt_DFGirl/*
cd /usr/share/fonts/FrLt_DFGirl
sudo mkfontscale
sudo mkfontdir
sudo fc-cache -fv
```