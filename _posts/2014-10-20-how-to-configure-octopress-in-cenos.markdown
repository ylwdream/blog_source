---
layout: post
title: "how to configure octopress in Cenos"
date: 2014-10-20 21:23:46 -0700
comments: true
categories: blog
tags:
- Octopress
---
# 1 先安装ruby #

    bash < <(curl -s https://raw.github.com/wayneeseguin/rvm/master/binscripts/rvm-installer )
    echo '[[ -s "$HOME/.rvm/scripts/rvm" ]] && . "$HOME/.rvm/scripts/rvm" # Load RVM function' >>$HOME/.bash_profile
    source ~/.bash_profile
    type rvm | head -1  #验证rvm 安装 output: rvm is a function
    rvm install 2.1.0 && rvm use 2.1.0  #安装ruby 2.1.0
    ruby -v

<!--more-->
# 2 安装 git #
    
    yum install git
	git config --global user.name "YOUR NAME"
	git config --global user.email "YOUR EMAIL ADDRESS"
然后去github官网注册一个账号，我是在CentOS 6.5上面用ssh登录git的。ssh登录要在本地产生一个密码对。一个公钥和私钥，公钥添加到github个人的ssh中。官网链接：[https://help.github.com/articles/generating-ssh-keys/](https://help.github.com/articles/generating-ssh-keys/)

    ls -al ~/.ssh
    ssh-keygen -t rsa -C your_email@example.com
把生成的公钥的密码添加到账户里，一直enter到底，不用写密码

    ssh-agent -s
	ssh-add ~/.ssh/id_rsa

测试是否登录成功

    ssh -T git@github.com
# 3 下载cotopress 和 主题 #
	git clone git://github.com/imathis/octopress.git
	cd octopress/
	gem install bundle #安装依赖
	bundle install
	rake install    #安装theme
# 4 设置博客地址 #
    rake setup_github_pages #把博客的主页地址填到这里，记得是ssh的地址 #git@github.com:ylwdream/ylwdream.github.io.git
    rake generate
	rake preview #http:localhost:4000上可以预览博客
    rake deploy
新增一个博客

	rake new_post["在github搭建-Octopress 博客"]
	rake generate   #生成内容
	rake deploy
# 5 把博客发布到github上去#
修改 .git/config 里的"https://"修改，因为是ssh链接，所以要把"https://"删掉
还要把mere修改问source

    [remote "origin"]
		url = git@github.com:$youid/$youid.github.com.git
	    fetch = +refs/heads/*:refs/remotes/origin/*
	[branch "source"]
	  remote = origin
	  merge = refs/heads/source

添加项目文件

    git add .
	git commit -m 'your message'
	git push origin source

然后就可以在http://ylwdream.github.io/ 上访问自己的博客了。

# 6 参考 #

这有一些我配置的时候参考的一些资料

- [http://octopress.org/docs/deploying/github/](http://octopress.org/docs/deploying/github/ "octopress官方链接")
- [http://sharadchhetri.com/2014/01/12/install-octopress-centos-6-rhel-6/](http://sharadchhetri.com/2014/01/12/install-octopress-centos-6-rhel-6/)
- [http://blog.csdn.net/biaobiaoqi/article/details/8760309](http://blog.csdn.net/biaobiaoqi/article/details/8760309)
