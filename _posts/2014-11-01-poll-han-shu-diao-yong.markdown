---
layout: post
title: "poll 函数调用"
date: 2014-11-01 19:41:18 -0700
comments: true
categories: linux
tags:
- socket 
---



使用非阻塞 I/O 的应用程序常常使用 poll, select, 和 epoll 系统调用. 
poll, select 和 epoll 本质上有相同的功能: 每个允许一个进程来决定它是否可读或者写一个或多个文件而不阻塞. 这些调用也可阻塞进程直到任何一个给定集合的文件描述符可用来读或写. 因此, 它们常常用在必须使用多输入输出流的应用程序, 而不必粘连在它们任何一个上.
<!--more-->
## select 函数
select 在 BSD Unix 中引入, 而 poll 是 System V 的解决方案. epoll 调用[23]添加在 2.5.45, 作为使查询函数扩展到几千个文件描述符的方法
``` c
unsigned int (*poll) (struct file *filp, poll_table *wait); 
```
这个驱动方法被调用, 无论何时用户空间程序进行一个 poll, select, 或者 epoll 系统调用, 涉及一个和驱动相关的文件描述符. 这个设备方法负责这 2 步:

1. 在一个或多个可指示查询状态变化的等待队列上调用 poll_wait. 如果没有文件描述符可用作 I/O, 内核使这个进程在等待队列上等待所有的传递给系统调用的文件描述符.

2. 返回一个位掩码, 描述可能不必阻塞就立刻进行的操作.
## poll 函数
poll 函数的声明
``` c
#include <poll.h>
int poll(struct pollfd fds[], nfds_t nfds, int timeout)；
```
fds：是一个struct pollfd结构类型的数组，用于存放需要检测其状态的Socket描述符；每当调用这个函数之后，系统不会清空这个数组，操作起来比较方便；特别是对于socket连接比较多的情况下，在一定程度上可以提高处理的效率；这一点与select()函数不同，调用select()函数之后，select()函数会清空它所检测的socket描述符集合，导致每次调用select()之前都必须把socket描述符重新加入到待检测的集合中。

-----
nfds：nfds_t类型的参数，用于标记数组fds中的结构体元素的总数量；

timeout：是poll函数调用阻塞的时间，单位：毫秒。设置为-1时表示阻塞调用

返回值:

·>0：数组fds中准备好读、写或出错状态的那些socket描述符的总数量

·=0：数组fds中没有任何socket描述符准备好读、写，或出错；此时poll超时，超时时间是timeout毫秒

·-1： poll函数调用失败，同时会自动设置全局变量errno

-----
下面是pollfd的定义
``` c
    struct pollfd {
    int fd; /*文件描述符*/
    short events; /* 等待的需要测试事件 */
    short revents; /* 实际发生了的事件，也就是返回结果 */
    };
```
与select()十分相似，当返回正值时，代表满足响应事件的文件描述符的个数，如果返回0则代表在规定时间内没有事件发生。如发现返回为负则应该立即查看 errno，因为这代表有错误发生。
如果没有事件发生，revents会被清空，所以你不必多此一举。

poll是采用轮询的方式进行监控的，所以对于监控的文件描述符越多，效率越低。
## poll 实例
下面是一个例子：

``` c poll.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <errno.h>
 
#include <poll.h>
#include <fcntl.h>
#include <unistd.h>
 
#define MAX_BUFFER_SIZE 1024
#define IN_FILES 3
#define TIME_DELAY 60*5
#define MAX(a,b) ((a>b)?(a):(b))
 
int main(int argc ,char **argv)
{
  struct pollfd fds[IN_FILES];
  char buf[MAX_BUFFER_SIZE];
  int i,res,real_read, maxfd;
  fds[0].fd = 0;
  if((fds[1].fd=open("data1",O_RDONLY|O_NONBLOCK)) < 0)
    {
      fprintf(stderr,"open data1 error:%s",strerror(errno));
      return 1;
    }
  if((fds[2].fd=open("data2",O_RDONLY|O_NONBLOCK)) < 0)
    {
      fprintf(stderr,"open data2 error:%s",strerror(errno));
      return 1;
    }
  for (i = 0; i < IN_FILES; i++)
    {
      fds[i].events = POLLIN;
    }
  while(fds[0].events || fds[1].events || fds[2].events)
    {
      if (poll(fds, IN_FILES, -1) <= 0)
    {
     printf("Poll error\n");
     return 1;
    }
      for (i = 0; i< IN_FILES; i++)
    {
     if (fds[i].revents)
       {
		   printf("fd %d start\n", i);
         memset(buf, 0, MAX_BUFFER_SIZE);
         real_read = read(fds[i].fd, buf, MAX_BUFFER_SIZE);
         if (real_read < 0)
        {
         if (errno != EAGAIN)
           {
             return 1;
           }
        }
         else if (!real_read)
        {
         close(fds[i].fd);
         fds[i].events = 0;
        }
         else
        {
         if (i == 0)
           {
             if ((buf[0] == 'q') || (buf[0] == 'Q'))
            {
             return 1;
            }
           }
         else
           {
             buf[real_read] = '\0';
             printf("%s", buf);
           }
        }
	 printf("fd %d end\n", i);
       }
    }
    }
  exit(0);
}
```
