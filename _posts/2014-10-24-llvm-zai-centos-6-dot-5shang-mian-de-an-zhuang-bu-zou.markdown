---
layout: post
title: "LLVM 在Centos 6.5上面的安装步骤"
date: 2014-10-24 19:16:54 -0700
comments: true
categories: compiler
tags:
- llvm 
---

下面整理的是我在自己的电脑上安装llvm时遇到的问题，以前用makefile自己安装了几次，每次都没有成功，昨天寝室师兄告诉我要看软件的版本问题，才明白到以前根本没有注意到这些问题。好了下面是安装的步骤，记录下，也为了后面的人做个参考。
<!--more-->
llvm的官方的User Guide链接：[http://llvm.org/docs/GettingStarted.html](http://llvm.org/docs/GettingStarted.html)

## 1. 首先看一下llvm依赖的编译环境

```
Package	Version	Notes

GNU Make 3.79, 3.79.1	Makefile/build processor

GCC >=4.7.0	C/C++ compiler1
python >=2.5	Automated test suite2

GNU M4 1.4	Macro processor for configuration3

GNU Autoconf 2.60	Configuration script builder3

GNU Automake 1.9.6	aclocal macro generator3

libtool 1.5.22	Shared library manager3

zlib >=1.2.3.4	Compression library4
```

Gcc的版本要大于4.7.0
GNU ld 2.16.X 链接器的版本要大于2.16
GNU binutils 2.17 bitnutils是一个二进制工具包，版本要大于2.17
如果你的电脑上的版本比要求的低。就`yum install binutils` 相关项到最新的版本

## 2. 安装Gcc
``` shell
	wget ftp://ftp.gnu.org/gnu/gcc/gcc-4.8.2/gcc-4.8.2.tar.bz2
	tar -xvjf gcc-4.8.2.tar.bz2
	cd gcc-4.8.2
	./contrib/download_prerequisites
	cd ..
	mkdir gcc-4.8.2-build
	cd gcc-4.8.2-build
	$PWD/../gcc-4.8.2/configure --prefix=$HOME/toolchains --enable-languages=c,c++
	make -j$(nproc)
	make install
```
make -j4就是根据处理器的个数，并行加快makefile的执行，我系统的$(nproc)=1,不过我建议直接make -j4。

## 3. 下载llvm源代码

首先就是在你的电脑上安装svn

	sudo yum install subversion

然后根据官方的文档，下载llvm和clang前端。


### 3.1 下载llvm

    cd where-you-want-llvm-to-live
    svn co http://llvm.org/svn/llvm-project/llvm/trunk llvm

### 3.2 检出Clang

    cd where-you-want-llvm-to-live
    cd llvm/tools
    svn co http://llvm.org/svn/llvm-project/cfe/trunk clang

### 3.3 检出Compiler-RT：

    cd where-you-want-llvm-to-live
	cd llvm/projects
    svn co http://llvm.org/svn/llvm-project/compiler-rt/trunk compiler-rt

### 3.4 得到测试套件的代码(可选的)

	cd where-you-want-llvm-to-live
	cd llvm/projects
	svn co http://llvm.org/svn/llvm-project/test-suite/trunk test-suite

### 3.5 配置环境并且编译LLVM和Clang

	cd where-you-want-to-build-llvm
	mkdir build (为了不污染源代码，在这个build文件夹下编译)
	cd build
	../llvm/configure [options]Some common options:
	--prefix=directory — 设置llvm编译的安装路径(default/usr/local).
	--enable-optimized — 是否选择优化(defaultis NO)，yes是指安装一个Release版本.
	--enable-assertions — 是否断言检查(default is YES).

我的命令是 

	../llvm/configure --prefix=/usr/local/llvm --enable-optimized --enable-targets=host-only
	

## 4. 编译make llvm

	cd build
    make -j4

编译make的时候会提示version `GLIBCXX_3.4.15' not found 。  
这是因为系统找不到这个版本的原因。输入下面的命令，可以看到并没有我们要的版本。

	strings /usr/lib/libstdc++.so.6 | grep GLIBCXX  

这时候要更新库的版本.这时候要更新系统 usr/lib目录下的Gcc动态链接库的版本。
将你上一部编译gcc产生的库的libstdc++.so.6.0.20 复制到/usr/lib目录下
  
	cp /home/wyl/build_gcc4.9/prev-i686-pc-linux-gnu/libstdc++-v3/src/.libs /usr/lib

然后更新软链接

    ln libstdc++.so.6.0.17 libstdc++.so.6
    strings /usr/lib/libstdc++.so.6 | grep GLIBCXX


![](/img/blog-img/gccstdc++.png)

更新完成后继续执行 make -j4，然后就是漫长的等待编译过程了。
安装完成后把llvm的bin目录加入到PATH
``` shell
vim ~/.bashrc
export PATH=$PATH:/usr/local/llvm
clang++ -v
```

## 参考资料 ##

- [llvm官方文档](http://llvm.org/docs/GettingStarted.html) 
- [gcc动态库的更新](http://www.cnblogs.com/yingsi/p/3290958.html) 
