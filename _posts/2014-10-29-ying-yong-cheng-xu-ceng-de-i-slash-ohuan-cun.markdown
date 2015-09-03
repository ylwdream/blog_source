---
layout: post
title: "应用程序层的I/O缓存"
date: 2014-10-29 06:30:27 -0700
comments: true
toc: false
categories: larbin 
tags:
- 爬虫 
- io
---

缓存技术在计算机的各个层次中都有涉及，例如cache,i/o，web访问url的缓存，dns的缓存。缓存的目的就是为了减少对资源的调用次数，加速程序的执行。
<!--more-->

我们知道，对文件读写的调用时系统调用，会设计到i/o读写，会发生中断等。如果我们先在内存缓存一部分数据，而不是每次有数据时都去调用i/o操作。而是当缓冲区满的时候才写入磁盘是会提高程序的执行效率的。这也是我今天看到larbin下面对文件的读写提供的应用程序级别的缓存。

今天手贱搜了下别人对larbin 的分析，感觉自己突然坚持的东西就坚持不下去了，因为自己也看了大半了月了，感觉进展真的很慢，也不知道我的学习方式是否是正确的。本来都算改写larbin,现在都开始放弃的念头了。哎，学业很忙，心情又不太好。

以下是larbin中的`PersistentFifo.cc`中的一部分代码。

构造函数

``` C++ PersistentFifo.h
PersistentFifo::PersistentFifo (bool reload, char *baseName) {
  fileNameLength = strlen(baseName)+5;
  fileName = new char[fileNameLength+2];
  strcpy(fileName, baseName);
  fileName[fileNameLength+1] = 0;
  outbufPos = 0;
  bufPos = 0;
  bufEnd = 0;
  mypthread_mutex_init (&lock, NULL);
  if (reload) {
	DIR *dir = opendir(".");
	struct dirent *name;

	fin = -1;
	fout = -1;
	name = readdir(dir);
	while (name != NULL) {
	  if (startWith(fileName, name->d_name)) {
		int tmp = getNumber(name->d_name);
		if (fin == -1) {
		  fin = tmp;
		  fout = tmp;
		} else {
		  if (tmp > fin) { fin = tmp; } //先进先出
		  if (tmp < fout) { fout = tmp; }
		}
	  }
	  name = readdir(dir);
	}
    if (fin == -1) {
      fin = 0;
      fout = 0;
    }
    if (fin == fout && fin != 0) {
      cerr << "previous crawl was too little, cannot reload state\n"
           << "please restart larbin with -scratch option\n";
      exit(1);
    }
	closedir(dir);
	in = (fin - fout) * urlByFile;
	out = 0;   //in 和 out 作为fifo的指针
	makeName(fin);
	wfds = creat (fileName, S_IRUSR | S_IWUSR);
	makeName(fout);
	rfds = open (fileName, O_RDONLY);
  } else {
	// Delete old fifos
	DIR *dir = opendir(".");
	struct dirent *name;
	name = readdir(dir);
	while (name != NULL) {
	  if (startWith(fileName, name->d_name)) {
		unlink(name->d_name);
	  }
	  name = readdir(dir);
	}
	closedir(dir);

	fin = 0;
	fout = 0;
	in = 0;
	out = 0;
	makeName(0);
	wfds = creat (fileName, S_IRUSR | S_IWUSR);
	rfds = open (fileName, O_RDONLY);
  }

```

写缓存，利用一个字符串数组，写入的函数是用的内存`memcopy`函数

``` C++ 
void PersistentFifo::writeUrl (char *s) {
  size_t len = strlen(s);
  assert(len < maxUrlSize + 40 + maxCookieSize);
  if (outbufPos + len < BUF_SIZE) {
    memcpy(outbuf + outbufPos, s, len);
    outbufPos += len;
  } else {
    // The buffer is full
    flushOut ();
    memcpy(outbuf + outbufPos, s, len);
    outbufPos = len;
  }
}
```

当写入满的时候才会一次性调用i/o操作
