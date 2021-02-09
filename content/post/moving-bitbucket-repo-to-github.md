+++
title = "Moving Bitbucket Repo to Github"
date = "2014-09-30"
slug = "2014/09/30/moving-bitbucket-repo-to-github"
Categories = ["git", "tools"]
+++

```bash
git clone https://bitbucket.org/username/repos.git local_dir
cd local_dir
git remote rename origin bitbucket
git remote add origin git@github.com:username/repos.git
git push -u origin master
git remote rm bitbucket
```
