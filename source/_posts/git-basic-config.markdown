---
layout: post
title: "Git配置(持续更新)"
date: 2013-11-29 23:14
comments: true
categories: 
- tools
- git
tags:
- tools
---
<center><p><img src="/images/git-logo-black.png" width="182" height="76" alt="git"></p></center>

用git大概有2年多了吧，不过大都停留在``add``，``commit``，``clone``，``push``这些简单的操作上，最近看了git-scm网站的维护者，Pro.Git的作者的一个演讲，准备好好研究下git。以下是笔记，会不间断更新。

最基础的git config莫过于``user.name``和``user.email``了吧。
```bash
git config --global user.name Solarex
git config --global user.email i@solarex.name
```

git ``commit``的时候有时会发现在comment中需要换行，在命令行直接输入的话非常不方便，其实可以在自己喜欢的编辑器里面操作。
```bash 
git config --global core.editor vim
```
这样git在执行``commit``操作时就会打开vim，就可以像编辑文本一样写comment了。

在打印git log的时候，如果有色彩区分会好看很多。
```bash
git config --global color.ui auto
```

看一些视频的时候，经常会看到他们输入一些非常短的命令就执行了相应的操作，其实，这些可以通过``alias``实现。
```bash
git config --global alias.st "status"
git config --global alias.ci "commit -a -v"
```

另外强烈推荐``tig``来查看git log，非常方便。Ubuntu下通过命令``sudo apt-get install tig``，Mac下通过``brew install tig``均可直接安装。
