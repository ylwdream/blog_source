---
layout: post
title: " linux 下文件的元数据"
date: 2015-03-16 20:12:11 -0700
comments: true
toc: false
categories: linux 
tags:
- io
- memory

---

文件的元数据也就是记录文件系统的数据字典。    
操作系统根据文件的元数据来解析文件，文件的元数据一般存储在文件的开头。

<!--more-->

应用程序能够通过调用`stat`和`fstat`函数，检索关于文件的信息。
``` c

#include <unistd.h>
#inlcude <sys/stat.h>

int stat(const char *filename, struct stat *buf);
int fsat(int fd, struct stat *buf);

struct stat {
	dev_t         st_dev;       //文件的设备编号
	ino_t         st_ino;       //节点
	mode_t        st_mode;      //文件的类型和存取的权限
	nlink_t       st_nlink;     //连到该文件的硬连接数目，刚建立的文件值为1
	uid_t         st_uid;       //用户ID
	gid_t         st_gid;       //组ID
	dev_t         st_rdev;      //(设备类型)若此文件为设备文件，则为其设备编号
	off_t         st_size;      //文件字节数(文件大小)
	unsigned long st_blksize;   //块大小(文件系统的I/O 缓冲区大小)
	unsigned long st_blocks;    //块数
	time_t        st_atime;     //最后一次访问时间
	time_t        st_mtime;     //最后一次修改时间
	time_t        st_ctime;     //最后一次改变时间(指属性)
}; 
```
>*测试程序如下：
----

``` c 
/*
 * =====================================================================================
 *
 *       Filename:  statcheck.c
 *
 *    Description:  读取文件的元数据
 *
 *        Version:  1.0
 *        Created:  03/16/2015 07:13:46 PM
 *       Revision:  none
 *       Compiler:  gcc
 *
 *         Author:  WYL (502), ylwzzu@gmail.com
 *   Organization:  
 *
 * =====================================================================================
 */

#include <unistd.h>
#include <sys/stat.h>
#include "vm/csapp.h"

int main(int argc, char *argv[])
{
	struct stat st;
	char *type, *readok;
	stat(argv[1], &st);
	if(S_ISREG(st.st_mode))
		type = "regular";
	else if(S_ISDIR(st.st_mode))
		type = "directory";
	else
		type = "other";
	if((st.st_mode & S_IRUSR))
		readok = "yes";
	else
		readok = "no";
	printf("filename %s ", argv[1]);
	printf("type: %s, read: %s\n", type, readok);
	printf("size %d blocksize %d alloc blocks %d \n", st.st_size, st.st_blksize, st.st_blocks);
	return 0;
}
```
还有存储器的影射机制，使得进程可以通信。

- 附录：
[memcached](https://code.google.com/p/memcached/wiki/ReleaseNotes1410) 有空要读下源代码
