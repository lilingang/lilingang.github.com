---
title: Git创建空白分支
date: 2017-05-15 15:48:37
tags: git
---
&emsp;&emsp;有些情况下，想创建一个从头开始的git分支，例如使用hexo和gitub pages搭建博客，既想将hexo原文件托管在git，又想利用gitub pages。所以我们在xxxx.github.io使用两个分支，master用来发布gitub pages；另一个分支用来托管hexo原文件，两个分支版本毫无关联。另一种场景就是代码重构，需要从头编写。

- 下载原仓库
```
$ git@github.com:lilingang/lilingang.github.com.git
$ cd lilingang.github.com
```

- 创建孤立分支hexo
```
git checkout --orphan hexo
```
&emsp;&emsp;此时孤立分支在`git branch`是看不到的，首次commit之后才可以看到


- 清空分支内容
```
$ git rm -rf .
$ rm .gitignore
```

- 添加新内容
```
echo "empty branch" >> README.txt
```

- 提交推送
```
$ git add .
$ git commit -m "init" .
$ git push origin hexo
```
&emsp;&emsp;commit操作之后，你就可以用```git branch```命令看到新分支的名字了，然后push到远程仓库，在github页面上即可看到新分支