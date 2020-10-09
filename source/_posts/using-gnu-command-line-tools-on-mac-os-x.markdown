---
layout: post
title: "Using GNU Command Line Tools on Mac OS X"
date: 2015-03-22 23:05
comments: true
categories: 
- dev
- tools
tags:
- tools
---
<p><center><img src="/images/apple_mac_logo.jpg" width="225" height="225"></center>
Unlike Ubuntu shipped with GNU command line tools,Mac OS X shipped with the BSD version,which make me WTF when I encountered with some compatibility issues.Fortunately,it's quit easy to install gnu tools by using homebrew in Mac OS X and set them as default.

<!-- more -->

First comes with the most important one -- GNU coreutils,which contains the most essential commands like ``ls``,``cat`` and bla,bla...

``brew install coreutils``,and ``export PATH="$(brew --prefix)/opt/coreutils/libexec/gnubin:/usr/local/bin:$PATH"``,and add this alias to your ``.bashrc`` or ``.zshrc``.

```bash
alias man="a() { echo $1; man -M $(brew --prefix)/opt/coreutils/libexec/gnuman $1 1>/dev/null 2>&1;  if [ "$?" -eq 0 ]; then man -M $(brew --prefix)/opt/coreutils/libexec/gnuman $1; else man $1; fi }; a"
```

Next you may probably wanna install the following ones(for some of the packages,you need to run ``brew tap homebrew/dups`` first,the ``brew tap`` command allows you to add more GitHub repos to the list of formulae that ``brew`` tracks, updates and installs from.)

```bash
brew install binutils
brew install diffutils
brew install ed --with-default-names
brew install findutils --with-default-names
brew install gawk
brew install gnu-indent --with-default-names
brew install gnu-sed --with-default-names
brew install gnu-tar --with-default-names
brew install gnu-which --with-default-names
brew install gnutls
brew install grep --with-default-names
brew install gzip
brew install screen
brew install watch
brew install wdiff --with-gettext
brew install wget
brew install vim --override-system-vi
brew install macvim --override-system-vim --custom-system-icons
```

Originally,the ``--with-default-names`` option was designed to prevent homebrew from prepending *g*s to the newly installed commands, thus we could use these commands as default ones over the ones shipped by OS X,but when I tried with this argument,it just didn't work,lol.Alternatively, ``PATH`` tricks works.

