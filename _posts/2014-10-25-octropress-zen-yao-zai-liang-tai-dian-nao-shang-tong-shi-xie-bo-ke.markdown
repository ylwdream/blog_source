---
layout: post
title: "Octropress 怎么在两台电脑上同时写博客"
date: 2014-10-25 23:23:33 +0800
comments: true
toc: false
categories: blog
tags:
- octopress
---

由于在寝室和实验室都有电脑，所以希望两边可以同步的来写博客，现阶段是还在配置中，由于老是忘记配置完要push到github上去，所以经常脑残的不同步，还有就是如果会点html和js，用这个配置网站简直是利器啊。
<!--more-->
## 创建一个本地的Octopress仓库 ##

重新创建一个已经存在的本地Octopress仓库只需要以下几个简单的步骤即可。

克隆(clone)博客到新机器

首先将博客的源文件clone到本地的octopress文件夹内。

   > git clone -b source git@github.com:ylwdream/ylwdream.github.io.git octopress

然后将博客文件clone到octopress的_deploy文件夹内

> cd octopress
> git clone git@github.com:ylwdream/ylwdream.github.io.git _deploy 

更新和推送

当你要在一台电脑写博客或做更改时，首先更新source仓库。更新master并不是必须的，因为你更改源文件之后还是需要rake generate的。

> cd octopress
> git pull origin source  # update the local source branch
> cd ./_deploy
> git pull origin master  # update the local master branch
如果有需要merge的文件，可以加上-f选项，强制执行。千万不要去github项目中修改配置文件。

写完博客之后不要忘了推送到remote，下面的步骤在每次更改之后都必须做一遍。（**千万要记住啊**）

> rake generate
> git add .
> git commit -am "Some comment here." 
> git push origin source  # update the remote source branch 
> rake deploy             # update the remote master branch

----

主要参考

- [超链接](http://boboshone.com/blog/2013/06/05/write-octopress-blog-on-multiple-machines/)
