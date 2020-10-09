---
layout: post
title: "MacBook Pro 2011中安装Ubuntu 13.04"
date: 2013-11-21 22:55
comments: true
categories: 
- tools
tags:
- tools
---
<center><p><img src="/images/logoubuntu.png" width="200" height="200" alt="ubuntu"></p></center>

之前已经在MacBook上装过Ubuntu了，不知怎么搞的，最近Ubuntu更新后，分辨率一直保持在1024×768，屏幕惨不忍睹，网上搜了好多，都是使用[xrandr](https://wiki.ubuntu.com/X/Config/Resolution)之类的，但是执行``xrandr``的时候老是报错，搜了下，貌似是一个bug。不管了，重装一下吧。

首先在MacBook下用``Disk Utility``分区，Ubuntu需要一个swap分区，一个``/``分区，用U盘安装的话有这两个就够了。但是我没有U盘，就又划分了一个2G的分区，把Ubuntu的镜像写进去，这样也便于以后恢复。分区全部使用FAT32格式。

分好区后就是写Ubuntu的镜像文件了，注意要从Ubuntu官方网站上面下载针对Mac特定版本的iso文件，下载下来后，校验下如果没错的话，就可以用``UNetbooin``来烧Ubuntu的镜像了，可以烧到U盘中也可以像我一样烧到一个专门的分区中。分区的编号可以通过命令``diskutil list``来查看。

<!-- more -->

然后下载[rEFIt](http://refit.sourceforge.net/)并安装，重启，按下option键直到出现Windows启动项，选择进入，就可以看到``UNetbootin``生成的菜单了。选择``Install Ubuntu``剩下的就和平常安装Ubuntu没什么两样了。需要注意的是Ubuntu选择安装分区的时候，要把``bootloader``安装在Ubuntu将要安装的磁盘分区上，也就是在Mac下分区时创建的那个分配给``/``的那个分区。剩下的就没什么了，Ubuntu安装很快，稍等一会就可以重启了。

重启后会看到有企鹅的图标，点进去，运气好的话就可以直接进Ubuntu了，但是我点进去的时候提示``Missing Operating System``，嗯，MBR貌似没有同步，没关系，重启，进Mac。安装[GPT fdisk]("http://sourceforge.net/projects/gptfdisk/")，在终端中敲入``sudo gdisk /dev/disk0``进入GPT fdisk菜单，按下``b``，会提示输入备份当前``mbr``的文件名称，输入并备份。现在需要更改``mbr``了，按下``r``再按下``p``，会打印出当前的分区信息，记住Mac和Ubuntu安装分区的编号，比如我的是2和6，现在按下``h``，会提示你输入分区编号，输入2 6回车，接下来会问是否把efi分区放在最前面，按下``y``，然后会让你输入每个分区的``mbr hex code``，mac的是AF，Windows的是07，Linux的是83，我的情况是2号分区安装mac，6号分区安装ubuntu，所以在提示输入2号分区的时候输入AF，接着提示设置boot flag时输入``n``，提示输入6号分区的时候输入83，接着提示输入设置boot flag时输入``n``，后面可能会提示发现free partition，是否使用其加密，``n``吧，如果使用加密的话启动时会卡在显示Mac和企鹅图标之前，必须按下Option键才能进入启动的选项菜单，太麻烦，还是直接否掉吧。OK，现在设置完了mbr，输入``w``保存覆盖掉旧的mbr文件。重启，选择企鹅图标，进入Ubuntu系统，Over，搞定了。

接下来就是Ubuntu下的操作了。

+ 更换软件源，使用aliyun的镜像。
```bash
deb http://mirrors.aliyun.com/ubuntu/ raring main restricted
deb http://mirrors.aliyun.com/ubuntu/ raring-updates main restricted
deb http://mirrors.aliyun.com/ubuntu/ raring universe
deb http://mirrors.aliyun.com/ubuntu/ raring-updates universe
deb http://mirrors.aliyun.com/ubuntu/ raring multiverse
deb http://mirrors.aliyun.com/ubuntu/ raring-updates multiverse
deb http://mirrors.aliyun.com/ubuntu/ raring-backports main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ raring-security main restricted
deb http://mirrors.aliyun.com/ubuntu/ raring-security universe
deb http://mirrors.aliyun.com/ubuntu/ raring-security multiverse
deb http://archive.canonical.com/ubuntu precise partner
deb-src http://archive.canonical.com/ubuntu precise partner
deb http://cz.archive.ubuntu.com/ubuntu trusty main universe
```

+ 添加ppa
```bash
sudo add-apt-repository ppa:fcitx-team/nightly
sudo add-apt-repository ppa:indicator-multiload/stable-daily
sudo add-apt-repository ppa:ubuntu-wine/ppa
sudo add-apt-repository ppa:synapse-core/ppa
sudo add-apt-repository ppa:fossfreedom/byzanz
sudo add-apt-repository ppa:kilian/f.lux
sudo add-apt-repository ppa:xdlailai/openyoudao
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install fcitx fcitx-config-gtk fcitx-sunpinyin fcitx-googlepinyin fcitx-module-cloudpinyin  fcitx-sogoupinyin fcitx-table-all indicator-multiload wine synapse byzanz fluxgui openyoudao mtp-tools mtpfs
```

+ 安装卸载软件
```bash 
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install python3-dev python3-openssl libxss1 flashplugin-installer mplayer vim git-core tig xclip zathura unrar p7zip-full  p7zip-rar zip unzip rar chmsee bleachbit preload goldendict goldendict-wordnet tcpdump mtr curl nscd ack-grep meld pngquant
sudo apt-get purge ibus ibus-gtk ibus-gtk3 ibus-pinyin ibus-pinyin-db-android ibus-table
sudo apt-get autoremove unity-lens-music unity-lens-photos unity-lens-gwibber unity-lens-shopping unity-lens-video
```

+ sublime_text配置文件在``$HOME/.config/sublime_text_3/``
+ [bashrc](https://gist.github.com/flyfire/3a9edce243c45b28dadd)

