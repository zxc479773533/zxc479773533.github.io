---
layout: post
title: Grub美化-主题和背景图片（in arch）
tags:
- Beautify
- Arch
categories: Linux
---
我们都知道linux刚装好之后的那个引导界面是一个黑框白字的界面，相信大家用时间长之后也觉得这个界面不怎么好看吧。今天下午我动手修改了一下，让开机启动的界面变得好看了许多，接下来我来总结一下修改的经过。

## 具体修改

首先在命令行里面 `sudo nano /etc/default/grub`，查看grub的配置文件。

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Grub-background-01.png)

在配置文件里面可以看到很多属性，理解也比较简单，这里我就不多说了，加“#”注释的又很多隐藏的功能，你可以去掉注释使用。

其中有两条是：#GRUB_BACKGROUND和#GRUB_THEME，这两条分别是grub的背景和grub的主题配置，如果只是想简单的修改背景，那么只需要将#GRUB_BACKGROUND前面的#去掉，在""内写上图片的路径即可（注意是png格式）。

如果需要修改主题，则要将#GRUB_THEME前面的#去掉，在""内填上该主题的配置文件，现在以Arch Linux装上之后自带的一个starfield主题为例来讲。（注意如果你要修改主题就不需要改上面的GRUB_BACKGROUND）

主题可以在网上自行下载，下载后应放在路径/boot/grub/themes中，进去可以看到里面已经有一个主题，叫starfield，文件夹里的themt.text就是该主题的配置文件。

这里推荐一个grub主题的下载网站：[grub themes](https://www.gnome-look.org/browse/cat/109/)

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Grub-background-02.png)

这里的desktop-image就是你在启动时的大背景，后面会有各处的字体字号等设置，可以根据需要自己修改。

最后在主题那里写上 `GRUB_THEME="/boot/grub/themes/starfield/theme.txt` 即可，如果想要替换自己的背景，可以把自己的图片改名为starfield.png并替换（注意得是有效png格式），或者自行修改配置文件。

最后，不要忘了更新配置文件：`grub-mkconfig -o /boot/grub/grub.cfg`

最终，我们配置好的效果如下：

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Grub-background-03.png)

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Grub-background-04.png)

选择登录系统的具体细节

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Grub-background-05.png)

最后补上我现在用的grub主题(自己稍微用PS处理了下)：

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Grub-background-06.jpg)