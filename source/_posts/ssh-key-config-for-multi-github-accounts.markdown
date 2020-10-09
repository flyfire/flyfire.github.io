---
layout: post
title: "SSH key settings for multi github accounts"
date: 2014-06-02 15:13
comments: true
categories: 
- tools
- git
tags:
- tools
---
在同一台电脑上有多个github账户时，切换ssh key会比较麻烦，可以使用ssh config来简化这一动作。

在使用``ssh-keygen``时，为不同的账户选择不同的ssh key文件。

```bash
hrh@Solarex:~$ ls ~/.ssh/
id_rsa_accountA id_rsa_accountA.pub id_rsa_accountB id_rsa_accountB.pub known_hosts config
``` 

在``~/.bashrc``中添加ssh key。

```bash
ssh-add ~/.ssh/id_rsa_accountA >/dev/null 2>&1
ssh-add ~/.ssh/id_rsa_accountB >/dev/null 2>&1
```

配置ssh config文件``~/.ssh/config``。
```bash
#AccountA
Host github-a.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_accountA

#AccountB
Host github-b.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_accountB
```

这样以后使用AccountA时可以``git clone git@github-a.com:user/repos.git local_dir``这样操作，clone下来后可以``cd local_dir``对``user.name``和``user.email``来进行config来覆盖global config，剩下的就和平时没有什么区别了，使用AccountB时相似操作就可以了。

<!-- more -->

<script src="https://gist.github.com/flyfire/ecdf3b6d623923d73c07.js"></script>
