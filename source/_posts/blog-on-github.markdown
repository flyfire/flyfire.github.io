---
layout: post
title: "使用Octopress和Github写博客"
date: 2012-07-27 14:50
comments: true
categories: 
- tools
tags:
- tools

---
<center><em>Ubuntu下建立Octopress博客过程</em></center>
<center><p><img src="/images/octopress_logo.jpg" alt="Octopress"></p></center>

+ 首先安装``git``
```bash
sudo apt-get install git
```

+ 安装``rvm``和``ruby 1.9.3``
```bash
curl -L https://get.rvm.io | bash -s stable --ruby=1.9.3
```

+ 克隆``octopress``到本地
```bash
git clone git://github.com/imathis/octopress.git 
```

+ 安装依赖
```bash
gem sources -a http://ruby.taobao.org/
gem sources -r http://rubygems.org/
gem sources -l #确保只有淘宝的ruby镜像
gem install bundle
bundle install #建议更改Gemfile第一行source为taobao镜像
```

+ 安装``slash``主题
```bash
git clone git://github.com/tommy351/Octopress-Theme-Slash.git .themes/slash
rake install['slash']
```
可以定制``slash``主题，由于众所周知的原因，我把博客的分享换成了国内的jiathis，评论系统换成了国内的多说，需要更改的地方也不是很多，就不详细叙述了，具体可以参考我的代码。当然你也可以使用其他的分享、评论系统，或者使用默认的也行。至于其他的第三方帐号，我只使用了Google Analytics，这个也很简单，就不赘述了。

+ 生成网站、预览
```bash
rake generate && rake preview
```
如果你没有改动``Rakefile``里的``server_port``的话，默认就可以在 http://localhost:4000/ 查看博客了。

<!-- more -->

+ 部署到github

首先你得有一个github帐号，比如说yourname，然后create repo，名字是yourname.github.com。新手的话可能还需要设置下git和github服务器通信，可以参考github的帮助页，[setup-git](https://help.github.com/articles/set-up-git)和[generating-ssh-keys](https://help.github.com/articles/generating-ssh-keys)等来进行设置。
OK，以上搞定后，可以在本地操作了。
```bash
rake setup_github_pages
```
按照提示输入你的github repo url就可以了。
然后部署。
```bash
rake generate && rake deploy
```
等几分种刷新下http://yourname.github.com 应该就不会显示404页面而是你的博客页了。

+ 把修改后的``octopress``和``slash``主题也push到github
```bash
git add . 
git commit -m 'push source'
git push origin source
```
这样子的话，这个repo就有了2个branch，一个是master，就是``rake generate``生成的博客网站，一个是source，就是你修改的octopress配置文件和slash主题。

+ 设置域名
```bash
echo "youdomain.com" >>source/CNAME
```
到域名服务商那里设置DNS就可以了。

+ 写文章、发表
```bash
rake new_post['new-post-title']
```
这个命令会在source/_posts目录下生成一个markdown文件，编辑这个markdown文件即可。markdown的语法可以在[gitcafe](http://gitcafe.com/riku/Markdown-Syntax-CN/blob/master/syntax.md)上查看。

写了新文章后，push到github。
```bash
git add . 
git commit -m 'new post'
git push origin source
```
然后deploy到github page
```bash
rake generate
rake deploy
```
过不了多久就可以在http://yourname.github.com 上查看最新写的文章了。

+ 更换工作环境后，恢复octopress和原来的文章
```bash
git clone -b source  git@github.com:yourname/yourname.github.com.git
```
剩下的安装``rvm``等参考前面文章。
获得博客网站有两种方式，一是``rake generate``直接在本地生成，另一种是从github服务器上取回来。
```bash
cd yourname.github.com
mkdir _deploy && cd _deploy
git init
git remote add origin git@github.com:yourname/yourname.github.com.git
git pull origin master
```
OK，现在可以继续写博客，然后po了。

---------------------------------
后记：

+ 写了文章，执行``rake deploy``可能会显示错误，修改Rakefile文件里的deploy函数中的``push``为``push -f``即可，注意文件中的第**18**行是修改过的。

{% include_code ruby/push2forcepush.rb %}

+ 学习git可以参考<a href="/downloads/files/learninggit.pdf">看日记学git</a>和[pro-git-zh](https://github.com/numbbbbb/progit-zh-pdf-epub-mobi)。
