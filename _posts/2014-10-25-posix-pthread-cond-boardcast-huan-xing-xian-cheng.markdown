---
layout: post
title: "posix pthread_cond_boardcast 唤醒线程"
date: 2014-10-25 22:31:30 -0700
comments: true
categories: larbin 
tags: 
- 爬虫
---
看larbin的代码的时候看到里面的fifo队列里有信号量和条件变量的使用，里面有一个函数是`pthread_cond_boardcast`使用。百度了下和`pthread_cond_signal`的区别，发现还是不明确，先记录下吧。
<!--more-->

## 函数原型

下面是函数的原型
``` C
    #include <pthread.h>

	int pthread_cond_signal(pthread_cond_t *cond);
	int pthread_cond_broadcast(pthread_cond_t *cond)
```
[官方文档](http://pubs.opengroup.org/onlinepubs/007908799/xsh/pthread_cond_broadcast.html)上说，`pthread_cond_signal`唤醒在cond条件变量上阻塞的线程，至少唤醒一个，视具体的处理器的调度策略和线程的优先级还有处理器的个数而定，而`pthread_cond_broadcast`会唤醒所有在条件变量上等待的线程。
## 条件变量
条件变量一般是和互斥量一起使用的，以防多个线程同时去访问条件变量。当`pthread_cond_broadcast`后，  
因为此时互斥量还没有unlock所以，那些等待这个条件变量的线程还是无法执行，只有当前拥有该mutex的线程unlock该mutex后，那些等待的线程才可以竞争去lock(mutex)去执行。
所以如果执行pthread_cond_broadcast 线程没有拥有mutex，会很容易出问题吧(RT:这个真想不明白到底会出什么问题)。

下面是larbin里的一个template代码和我加的测试代码。
```	C++ SyncFifo.h
/* fifo in RAM with synchromisations */
	
	#ifndef SYNCFIFO_H
	#define SYNCFIFO_H
	
	#define std_size 100
	#define mypthread_cond_init(x,y) pthread_cond_init(x,y)
	#define mypthread_cond_destroy(x) pthread_cond_destroy(x)
	#define mypthread_cond_wait(c,x,y) while (c) { pthread_cond_wait(x,y); }
	#define mypthread_cond_broadcast(x) pthread_cond_broadcast(x)
	
	#define mypthread_mutex_init(x,y) pthread_mutex_init(x,y)
	#define mypthread_mutex_destroy(x) pthread_mutex_destroy(x)
	#define mypthread_mutex_lock(x) pthread_mutex_lock(x)
	#define mypthread_mutex_unlock(x) pthread_mutex_unlock(x)
	
	template <class T>
	class SyncFifo
	{
	protected:
		uint in, out;
		uint size;
		T **tab;
		pthread_mutex_t lock;
		pthread_cond_t nonEmpty;
	
	public:
		SyncFifo(uint size = std_size);
	
		~SyncFifo();
		
		// get the first object
		T* get();
	
		T* tryGet();
	
		/* add an object in the Fifo */
		void put(T *obj);
		
		int getLength();
	};
	
	template<class T>
	SyncFifo<T>::SyncFifo(uint size)
	{
		tab = new T*[size];
		this->size = size;
		in = out = 0;
		mypthread_mutex_init(&lock, NULL);
		mypthread_cond_init(&nonEmpty, NULL);
	}
	
	template <class T>
	SyncFifo<T>::~SyncFifo()
	{
		delete [] tab;
		mypthread_mutex_destroy(&lock);
		mypthread_cond_destroy(&nonEmpty);
	
	}
	
	template <class T>
	T *SyncFifo<T>::get()
	{
		T *tmp;
		mypthread_mutex_lock(&lock);
		mypthread_cond_wait(in == out, &nonEmpty, &lock);
		tmp = tab[out];
		out = (out + 1) % size;
		mypthread_mutex_unlock(&lock);
		return tmp;
	}
	
	template <class T>
	T *SyncFifo<T>::tryGet()
	{
		T *tmp;
		mypthread_mutex_lock(&lock);
		if(in != out)
		{
			tmp = tab[out];
			out = (out + 1) % size;
		}
		mypthread_mutex_unlock(&lock);
	}
	
	template <class T>
	void SyncFifo<T>::put(T *obj)
	{
		mypthread_mutex_lock(&lock);
		tab[in] = obj;
		if(in == out) //通知因为nonEmpty为空而阻塞的调用线程
		{
			mypthread_cond_broadcast(&nonEmpty);
		}
	
		in = (in + 1) % size;
		if(in == out)   //必须再次判断，因为有可能已经非满了
		{
			T **tmp;
			tmp = new T*[2*size];
			for(uint i =out; i<size; ++i)
			{
				tmp[i] = tab[i];
			}
			for(uint i=0; i<in; ++i)
			{
				tmp[i+size] = tab[i];
			}
	
			in += size;
			size *= 2;
			delete [] tab;
			tab = tmp;
		}
	
		mypthread_mutex_unlock(&lock);
	}
	
	template <class T>
	int SyncFifo<T>::getLength()
	{
		int tmp;
		mypthread_mutex_lock(&lock);
		tmp = (in + size - out) % size;
		mypthread_mutex_unlock(&lock);
		return tmp;
	}
	#endif
```
## 测试程序
我自己写了一个main函数来调用此模板，目前还是没发生死锁，但是我觉得如果先调用get是会发生死锁的。如果执行get的线程一直运行，而执行put的线程得不到调用，因为pthread_cond_wait是阻塞调用的，也就是条件不满足，会一直在那个条件上等待，那么就死掉了，好歹调度系统不会像我想的这么笨。。
``` C++ test.main	
	#include <iostream>
	#include <pthread.h>
	#include <unistd.h>
	#include <stdio.h>
	#include <stdlib.h>
	#include <sys/types.h>
	#include "SyncFifo.h"
	
	using namespace std;
	
	#define size 10
	
	struct data
	{
		static int number;
		data()
		{
			number ++;
		}
		void print()
		{
			printf(" the number is %d\n", number);
		}
	};
	int data::number = 0;
	SyncFifo<data> * fifo;
	
	void *put(void *)
	{
		while(true)	
		{
			data *tmp = new data;
			printf("add data : fifo size %d", fifo->getLength());
			tmp->print();
			fifo->put(tmp);
			sleep(1);
		}
	}
	
	void *get(void *)
	{
		while(true)
		{
			data *tmp = fifo->get();
			printf("get data : fifo size %d", fifo->getLength());
			tmp->print();
			sleep(3);
		}
	}
	int main()
	{
		fifo = new SyncFifo<data>;
		pthread_t thread[5];
	
		for(int i=0; i<5; ++i)
		{
	
			if(i%2)
			pthread_create(&thread[i], NULL, put, NULL);
			else
				pthread_create(&thread[i], NULL, get, NULL);
		}
	
		pthread_join(thread[0], NULL);
		pthread_join(thread[1], NULL);
		pthread_join(thread[2], NULL);
	
		return 0;
	}
```
## 总结
关于多线程的问题，头大，目前就是遇到一点，学习一点，并没有专门的去学习过。
