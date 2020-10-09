---
layout: post
title: "Moving bitbucket repo to github"
date: 2014-09-30 15:38
comments: true
categories: 
- git
- tools
tags:
- tools
---

```bash
git clone https://bitbucket.org/username/repos.git local_dir
cd local_dir
git remote rename origin bitbucket
git remote add origin git@github.com:username/repos.git
git push -u origin master
git remote rm bitbucket
```
