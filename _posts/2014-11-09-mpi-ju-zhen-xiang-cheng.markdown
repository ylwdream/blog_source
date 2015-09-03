---
layout: post
title: "MPI矩阵相乘"
date: 2014-11-09 04:34:43
comments: true
toc: false
categories: 
- parallel 
tags:
- MPI
---



## MPI 介绍

MPI和openMP都可以用于并行计算，但是MPI的并行粒度是进程级别的。一般运行在SMP集群中。MPI 是通过消息传递模型进行并行计算，所以mpi编写的程序即可以运行在统一的共享模型机器上或者运行在同一编制的共享内存模型中。
MPI 提供的只是消息传递的语言接口，有 `C` 和 `C++` 接口。

<!--more-->

下面说一下如何利用MPI来实现对举证相乘的并行计算。
## MPI 数据类型
由于mpi支持的原生类型并没有对二维矩阵的支持，所以我们要想发送二维矩阵，首先要对矩阵进行数据打包。

### 1. MPI提供了全面而强大的构造函数(Constructor Function)来定义派生数据类型.  
``` c    
MPI_Datatype EvenElements;  
···
MPI_Type_vector(50, 1, 2, MPI_DOUBLE, &EvenElements);  
MPI_Type_commit(&EvenElements);  
MPI_Send(A, 1, EvenElements, destination, ···);  
```

首先声明一个类型为MPI_Data_type的变量EvenElements 

调用构造函数MPI_Type_vector(count, blocklength, stride, oldtype, &newtype)来定义派生数据类型。 
新的派生数据类型必须先调用函数MPI_Type_commit获得MPI系统的确认后才能调用MPI_Send进行消息发送。 

### 2. 下面来看收集操作函数：

收集MPI_GATHER是典型的多对一通信的例子 在收集调用中 每个进程 包括根进 程本身 将其发送缓冲区中的消息发送到根进程 根进程根据发送进程的进程标识的序号即 进程的rank值 将它们各自的消息依次存放到自已的消息缓冲区中。

收集调用每个进程的发送数据个数sendcount和发送数据类型sendtype都是相同的 都和 根进程中接收数据个数recvcount和接收数据类型recvtype相同 注意根进程中指定的接收数 据个数是指从每一个进程接收到的数据的个数 而不是总的接收个数 此调用中的所有参数对根进程来说都是有意义的,而对于其它进程只有sendbuf sendcount sendtype root和comm是有意义的 其它的参数虽然没有意义 但是却不能省略 root和comm在所有进程中都必须是一致的。 

### 3. 在计算过程中为了控制同步。

MPI_BARRIER阻塞所有的调用者直到所有的组成员都调用了它 各个进程中这个调用 才可以返回 

### 4. 最后一个散发函数
MPI_SCATTER是一对多的组通信调用，但是和广播不同 ROOT向各个进程发送的数据可以是不同的 MPI_SCATTER和MPI_GATHER的效果正好相反，两者互为逆操作。
其效果相当于根进程调用了N次send函数，然后每一个进程调用了一次recv操作，接收的元素个数相同，但是内容不同。
 
## 矩阵相乘
``` c mpi.c
// wyl 
// c[n][n] = a[n][n]*b[n][n]

#include <mpi.h>
#include <stdio.h>
#include <math.h>
#include <string.h>
#include <malloc.h>
#include <stdlib.h>

#define Len 5
//令进程数为Len

int myrank, numprocs;
int i, j;
int tag; 
MPI_Status status;
int a[Len][Len], b[Len][Len], result[Len][Len];
int buf[Len]; //缓冲区
char processor_name[MPI_MAX_PROCESSOR_NAME];
MPI_Datatype colum_data;

void clm(int buf[Len], int b[Len][Len])
{

	int *rbuf = (int*)malloc(Len*sizeof(int));
		
	for(i=0; i<Len; ++i)
	{
		for(j=0; j<Len; ++j)
		{
			rbuf[i] += buf[j]*b[j][i];
		}
	}
	memcpy(buf, rbuf, Len*sizeof(int));
	free(rbuf);
	return;
}

int main(int argc, char *argv[])
{

	MPI_Init(&argc, &argv);
	MPI_Comm_rank(MPI_COMM_WORLD, &myrank);
	MPI_Comm_size(MPI_COMM_WORLD, &numprocs);
	
	tag = 0;
	
	MPI_Type_vector(1, Len, Len, MPI_INT, &colum_data);  //1行的数据
	MPI_Type_commit(&colum_data);
	
	if(myrank == 0)
	{
		for(i = 0; i < Len; ++i)
		for(j = 0; j < Len; ++j)
		{
			a[i][j] = i + j;
			b[i][j] = (i+1) * (j+1);
			result[i][j] = 0;
		}	
		
		for(i=1; i<numprocs; ++i)
			MPI_Send(&b, Len, colum_data, i, tag, MPI_COMM_WORLD);
	
		for(i=1; i<numprocs; i++)
			MPI_Send(&a[i], 1, colum_data, i, tag, MPI_COMM_WORLD); //root进程不发送数据
	
		// MPI_Scatter(a, 1, colum_data, buf, 1, colum_data, 0, MPI_COMM_WORLD);
		memcpy(buf, a, Len*sizeof(int));
		clm(buf, b);

		MPI_Barrier(MPI_COMM_WORLD);
		MPI_Gather(buf, 1, colum_data, result, 1, colum_data, 0, MPI_COMM_WORLD);
	
		printf("the result is\n");	
		for(i=0; i<Len; ++i)
		{
			for(j=0; j<Len; ++j)
				printf("%d ", result[i][j]);
			printf("\n");
		}
	}
	else
	{

		printf("this is rank %d\n", myrank);
		
		MPI_Recv(b, Len, colum_data, 0, tag, MPI_COMM_WORLD, &status);
		
		for(i=0; i<Len; ++i)
		{
			for(j=0; j<Len; ++j)
				printf("%d ", b[i][j]);
			printf("\n");
		}
		
		MPI_Recv(buf, 1, colum_data, 0, tag, MPI_COMM_WORLD, &status);
		
		clm(buf, b);
		
		MPI_Barrier(MPI_COMM_WORLD); //	确定都收到了数组b的消息，且计算完成
		MPI_Gather(buf, 1, colum_data, result, 1, colum_data, 0, MPI_COMM_WORLD);
	}

	MPI_Finalize();

	return 0;
}
```

***参考：***
[http://www.mpich.org/static/docs/v3.1/www3/MPI_Type_vector.html](http://www.mpich.org/static/docs/v3.1/www3/MPI_Type_vector.html)
