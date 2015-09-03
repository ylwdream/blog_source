---
layout: post
title: "pthread_attr_t 线程属性"
date: 2014-10-22 09:14:39 -0700
comments: true
categories: larbin
tags:
- pthread
- 爬虫 
---
'pthread_attr_t' 是posix线程库函数在创建线程时的属性，默认为NULL
 <!--more-->
```	
intpthread_create(pthread_t*tidp,constpthread_attr_t*attr,(void*)(*start_rtn)(void*),void*arg);
```
> pthread_aar_t 在pthread.h中的定义为：

``` c
typedef struct
{
	int  	detachstate;     线程的分离状态
    int 	schedpolicy;   线程调度策略
    struct  sched_param  schedparam;   线程的调度参数
    int  	inheritsched;    线程的继承性
    int  	scope;          线程的作用域
    size_t  guardsize; 线程栈末尾的警戒缓冲区大小
    int     stackaddr_set;
    void *  stackaddr;      线程栈的位置
    size_t  stacksize;       线程栈的大小
}pthread_attr_t;
```
 线程的分离状态决定一个线程以什么样的方式来终止自己。
           
我们已经在前面已经知道，在默认情况下线程是非分离状态的，这种情况   
下，原有的线程等待创建的线程结束。只有当pthread_join() 函数返回       
时，创建的线程才算终止，才能释放自己占用的系统资源。   

分离线程没有被其他的线程所等待，自己运行结束了，线程也就终止了，
马上释放系统资源。

通俗的说也就是：我们知道一般我们要等待(pthread_join)一个线程的结束，
主要是想知道它的结束状态，否则等待一般是没有什么意义的！但是if有一
些线程的终止态我们压根就不想知道，那么就可以使用“分离”属性，那么我
们就无须等待管理，只要线程自己结束了，自己释放src就可以咯！这样更
方便！

-----
``` c
#include <pthread.h>
int pthread_attr_getdetachstate(const pthread_attr_t * attr, int * detachstate);
int pthread_attr_setdetachstate(pthread_attr_t * attr, int detachstate);
```
参数：attr:线程属性变量

detachstate:分离状态属性   
若成功返回0，若失败返回-1。

设置的时候可以有两种选择：

<1>.detachstate参数为：PTHREAD_CREATE_DETACHED     分离状态启动

<2>.detachstate参数为：PTHREAD_CREATE_JOINABLE    正常启动线程

-----

下面是一个例子小程序，说明在线程创建的过程中主线程必须要等待子线程的结束以后才可以终止。其中等待是调用`pthread_join()`函数，此函数默认是阻塞函数。只有等子线程结果以后才可以返回。也就是说只有此函数返回以后才可以继续执行下面的代码。线程返回一是代码执行完成，二是调用`pthread_exit()`函数。

``` c
/*************************************************************************
> File Name: thread.c
> Author: wyl
> Mail: 
> Created Time: Wed 22 Oct 2014 06:58:43 AM PDT
************************************************************************/

#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
#include <stdlib.h>

#define MAX1 10
#define MAX2 20

pthread_t thread[2];
pthread_mutex_t mut;
int number, i;

void *thread1()
{
	printf("htread1: i am thread 1\n");
	for(i =0; i < MAX1; ++i)
	{
		printf("thread1: number = %d i = %d\n", number, i);
		pthread_mutex_lock(&mut);
		number++;
		pthread_mutex_unlock(&mut);
		sleep(2);
	}
	printf("thread1: is main function waiting for me\n");
	pthread_exit(NULL);
}

void *thread2()
{
	printf("htread2: i am thread 2\n");
	for(i =0; i < MAX2; ++i)
	{
		printf("thread2: number = %d i = %d\n", number, i);
		pthread_mutex_lock(&mut);
		number++;
		pthread_mutex_unlock(&mut);
		sleep(3);
	}
	printf("thread2: is main function waiting for me\n");
	pthread_exit(NULL);
}

void thread_create(void)
{
	int temp;
	pthread_attr_t attr;
	pthread_attr_init(&attr);
	pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
	if((temp = pthread_create(&thread[0], &attr, thread1, NULL)) != 0)
		pthread_attr_destroy(&attr), printf("thread1 create failed\n");
	else
		printf("thread1 is established\n");

	if((temp = pthread_create(&thread[1], NULL, thread2, NULL)) != 0)
		printf("thread2 create failed\n");
	else
		printf("thread2 is established\n");

}

void thread_wait(void)
{
	if(thread[0] != 0)
	{
		pthread_join(thread[0], NULL);
		printf("thread 1 is over\n");
	}
	if(thread[1] != 0)
	{
//		pthread_detach(thread[1]);
		pthread_join(thread[1], NULL);
		printf("thread 2 is over\n");
	}
}

int main()
{

	pthread_mutex_init(&mut, NULL);
	printf("the main function\n");
	thread_create();
	printf("the main function wait for threads\n");
	thread_wait();

	return 0;
}

```
> larbin下有一段代码是创建的一个默认的线程。创建完线程就使它分离，也就是线程的作用只是执行函数，不用等待返回状态

``` c
void startThread (StartFun run, void *arg) {
	pthread_t t;
	pthread_attr_t attr;
	if (pthread_attr_init(&attr) != 0
  		|| pthread_create(&t, &attr, run, arg) != 0
  		|| pthread_attr_destroy(&attr) != 0
  		|| pthread_detach(t) != 0) 
	{
		cerr << "Unable to launch a thread\n";
		exit(1);
	}
}
```
