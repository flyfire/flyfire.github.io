+++
title = "Mac Dev Setup"
date = "2015-05-05"
slug = "2015/05/05/mac-dev-setup"
Categories = ["dev", "tools"]
+++
<p><center><img src="/images/apple_mac_logo.jpg" width=225 height=225/></center></p>

<h2 id="system-preferences">System preferences</h2>

In Apple Icon > System Preferences:

+ Trackpad > Tap to click
+ Keyboard > Key Repeat > Fast (all the way to the right)
+ Keyboard > Delay Until Repeat > Short (all the way to the right)
+ Dock > Automatically hide and show the Dock

<!-- more -->

<h2 id="homebrew">Homebrew</h2>

+ install command line tools ``xcode-select --install``,``xcode-select -p
/Library/Developer/CommandLineTools`` to check if command line tools is installed

+ install homebrew ``ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"`` from [brew.sh](http://brew.sh/)

+ ``brew doctor``,``brew update``,``brew list``,``brew search wget``,``brew install wget``,``brew outdated``,``brew upgrade``,``brew upgrade wget``,``brew uninstall wget --force``,``brew info wget``,``brew deps wget``,``brew edit wget``,``brew list --versions``,``brew cleanup``

+ ``brew install caskroom/cask/brew-cask`` from [brew.sh](https://brew.sh/index_zh-cn)
+ ``brew cask search thunder``,``brew cask info thunder``,``brew cask edit thunder``,``brew cask uninstall thunder``

```bash
$ brew list
ant	  automake   coreutils	gcc49  gnu-sed	     libksba   libyaml	pkg-config  tig
apktool   brew-cask  dex2jar	git    isl011	     libmpc08  mpfr2	readline    tree
autoconf  cloog018   findutils	gmp4   libgpg-error  libtool   openssl	rename	    wget
houruhou at MacPro in ~
$ brew cask list
android-file-transfer	  dash			    go2shell		      neteasemusic		sublime-text
android-studio		  diffmerge		    google-chrome	      pycharm			vlc
appcleaner		  flux			    iterm2		      qq
bilibili		  foxmail		    jd-gui		      skim
ccleaner		  gitbook-editor	    macdown		      sogouinput
```

+ Sublime Text

```json
{
    "font_face": "Consolas",
    "font_size": 13,
    "rulers":
    [
        79
    ],
    "highlight_line": true,
    "bold_folder_labels": true,
    "highlight_modified_tabs": true,
    "tab_size": 4,
    "translate_tabs_to_spaces": true,
    "word_wrap": false,
    "indent_to_bracket": true
}
```

+ Go2Shell ``open -a go2shell --args config`` no longer needed

<h2 id="python">Python</h2>

+ ``brew install python --with-brew-openssl``,``brew install python3 --with-brewed-openssl``
+ ``sudo easy_install pip``
+ pip.conf

```bash
$ cat ~/.pip/pip.conf
; http://www.pypi-mirrors.org/
[global]
use-mirrors=true
; mirrors=http://pypi.douban.com
index-url=http://pypi.douban.com/simple
trusted-host=pypi.douban.com
; ln -s pip.conf ~/.pip/pip.conf
```

+ ``sudo pip install virtualenv virtualenvwrapper``
+ virtualenv setup

```bash
#virtualenv
# virtualenvwrapper
export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python
export WORKON_HOME=/Users/houruhou/Workspace/Solarex/PythonWorkspace
[ -f /usr/local/bin/virtualenvwrapper.sh ] && source /usr/local/bin/virtualenvwrapper.sh
[ -f /etc/bash_completion.d/virtualenvwrapper ] && source /etc/bash_completion.d/virtualenvwrapper
export PIP_VIRTUALENV_BASE=$WORKON_HOME
export PIP_RESPECT_VIRTUALENV=true
```

+ ``mkvirtualenv --no-site-packages test``

<h2 id="java">Java</h2>
+ ``brew cask install java``
+ ``brew tap caskroom/versions``,``brew cask install java6``
+ ``/usr/libexec/java_home -V``查看安装了哪些jdk
+ ``JAVA_HOME=`/usr/libexec/java_home -v 1.6``
+ open eclipse prompt to install java

```bash
/Library/Java/JavaVirtualMachines/jdk1.7.0_45.jdk/Contents/Info.plist

diff

<key>JVMCapabilities</key>
    <array>
<string>CommandLine</string>
</array>

<key>JVMCapabilities</key>
<array>
    <string>JNI</string>
    <string>BundledApp</string>
    <string>WebStart</string>
    <string>Applets</string>
    <string>CommandLine</string>
</array>
```

+ ``eclipse.ini``

```bash
--launcher.XXMaxPermSize
2048m
...
-vmargs
...
-Xms512m
-Xmx850m
-XX:PermSize=512m
-XX:MaxPermSize=1024m
```

<h2 id="command-line">command line</h2>

+ ``defaults write com.apple.finder AppleShowAllFiles -boolean true ; killall Finder`` 显示隐藏文件，``defaults write com.apple.finder AppleShowAllFiles -boolean false ; killall Finder``，不显示隐藏文件
+ ``defaults write com.apple.dock ResetLaunchPad -bool true; killall Dock``新的应用被安装后，经常会跑到 Launchpad 的第一屏，所以它们的位置跟安装的顺序有关系，而我更希望它们可以按照某种更加稳定的顺序排列，比如按照系统默认的顺序，在默认顺序中，Launchpad 第一屏只有 Apple 自家应用。
+ ``defaults write com.apple.screencapture location ~/Pictures/ScreenShots;killall SystemUIServer``,change default screen capture folder
+ ``sudo scutil --set HostName MacPro``修改hostname
+ ``defaults write com.apple.finder _FXShowPosixPathInTitle -bool TRUE;killall Finder``finder显示路径，``defaults delete com.apple.finder _FXShowPosixPathInTitle;killall Finder``恢复默认不显示

<h2 id="reference">reference</h2>

+ [mac-dev-setup](https://github.com/nicolashery/mac-dev-setup)
+ [mac-dev](https://github.com/pubyun/macdev)
+ [setup-mac-dev-zh_cn](https://aaaaaashu.gitbooks.io/mac-dev-setup/content/SystemPreferences/index.html),[setup-mac-dev](http://sourabhbajaj.com/mac-setup/)
+ [mac-configurations](https://github.com/solarex/macconfigurations)
+ [Mac keyboard shortcuts](https://support.apple.com/en-us/HT201236)
+ [Mac keyboard shortcuts for accessibility features](https://support.apple.com/en-us/HT204434)





