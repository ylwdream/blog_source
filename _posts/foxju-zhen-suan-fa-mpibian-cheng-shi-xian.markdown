---
layout: post
title: "FOX矩阵算法MPI编程实现"
date: 2014-12-16 00:23:20 +0800
comments: true
toc: false
categories: parallel
tags:
- MPI
---



Fox算法同样通过循环移位的办法来达到节省存储空间的目的。

<!--more-->

设处理器个数p=q2，则算法的要点如下：
1. 所选中的对角线Aii向所在行的q个处理器进行一对多广播；
2. 各处理器将自己所拥有的A和B的子块进行矩阵相乘运算；
3. B矩阵的块向上循环移动一位，从下面接受一个新的B矩阵块；
4. 选择A的一个矩阵块作为广播源，选择方法：如果Aij是上次的广播源，则本次的广播源是Ai,(j+1)%q。其中'%'表示取模运算。转步骤2。

附链接：
[http://202.197.191.206:8080/06/text/ch06/se03/6_3_4_3.htm#](http://202.197.191.206:8080/06/text/ch06/se03/6_3_4_3.htm#)

>* 程序运行 mpirun -n proncs fox
>* 其中n必须是平方数，例如：4、8、16

``` c FOX_MPI.c
// wyl 
// c[n][n] = a[n][n]*b[n][n]

#include <mpi.h>
#include <stdio.h>
#include <math.h>
#include <string.h>
#include <malloc.h>
#include <stdlib.h>

#define a(i, j) a[i*stock + j]
#define b(i, j) b[i*stock + j]
#define c(i, j) c[i*stock + j]
#define A(i, j) A[i*Len + j]
#define B(i, j) B[i*Len + j]
#define C(i, j) C[i*Len + j]
//令进程数为Len

int myrank, numprocs;
int i, j, k;
int step;
int send_i, send_j;      //广播的块号
int send_m, recv_m;         //广播的线程号 
MPI_Status status;
int Len;
int *A, *B, *C;
int *buf; //缓冲区
int stock, sqrt_p; //每块大小和处理器个数的根号值 
MPI_Datatype colum_data; //每个块的大小stock*stock 

void clm(int *a, int *b, int *c, int stock)
{   
	for(i=0; i<stock; ++i)
	{
	  for(j=0; j<stock; ++j)
	  {
	      for(k=0; k<stock; k++)
	      c(i,j) += a(i,k) * b(k,j);
	  }
	}
}

void init_data(int *A, int *B, int Len)
{
	for(i=0; i<Len; i++)
	for(j=0; j<Len; j++)
	{
		A[i*Len+j] = rand() % 100;
		B[i*Len+j] = rand() % 100;
	}
}

void out_data(int *A, int Len)
{
	for(i=0; i<Len; i++)
	{
		for(j=0; j<Len; j++)
			printf("%d ", A[i*Len + j]);
		printf("\n");
	}
}

void Environment_Finalize(int *a, int *b, int *c)
{
	free(a);
    free(b);
    free(c);
    free(buf);
}

void send_ms(int rank, int *a)
{
	if(myrank == rank)
	{
		send_i = myrank / sqrt_p;
		send_j = myrank % sqrt_p;
		for(i=0; i<sqrt_p; i++)
		{
			if(i % sqrt_p != send_j)
			{
				MPI_Send(a,stock*stock, MPI_INT, send_i*sqrt_p + i, 0, MPI_COMM_WORLD);
			}
		}
	}
	else
	{
		MPI_Recv(a,stock*stock, MPI_INT, rank, 0, MPI_COMM_WORLD, &status);  
	}	
		
}


int main(int argc, char *argv[])
{
	int *a, *b, *c;
	int u, v;
	double start_time, end_time;
		
	srand(time(NULL));
	
	MPI_Init(&argc, &argv);
	MPI_Comm_rank(MPI_COMM_WORLD, &myrank);
	MPI_Comm_size(MPI_COMM_WORLD, &numprocs);
  	
	Len = 10;
	sqrt_p = sqrt(numprocs);
	stock = Len / sqrt_p; 

	printf("%d\n",stock);

  	//stock行的数据,每行的数据为stock,每个块间隔为len 
  	MPI_Type_vector(stock, stock, Len, MPI_INT, &colum_data);  
	MPI_Type_commit(&colum_data);
  	
  	a = (int*)malloc(stock*stock*sizeof(int));
  	b = (int*)malloc(stock*stock*sizeof(int));
	c = (int*)malloc(stock*stock*sizeof(int));
  	buf = (int*)malloc(stock*Len*sizeof(int));
	
  	if(myrank == 0)
  	{
  		A = (int*)malloc(Len*Len*sizeof(int));
  		B = (int*)malloc(Len*Len*sizeof(int));
  		C = (int*)malloc(Len*Len*sizeof(int));
  		init_data(A, B, Len);
  		
		out_data(A,Len);
  		
  		for(i=0; i<stock; ++i)
  		for(j=0; j<stock; ++j)
  			a(i, j) = A(i, j);
  			
  		for(i=0; i<Len; i += stock)
  		for(j=0; j<Len; j += stock)
  		{
  			if(i || j )
  			{
  				MPI_Send(&A(i,j), 1, colum_data,(i/stock)*sqrt_p + j/stock, 0, MPI_COMM_WORLD);
  				MPI_Send(&B(i,j), 1, colum_data, (i/stock)*sqrt_p + j/stock, 1, MPI_COMM_WORLD);
  			}
	  	}
  	}
	else 
	{
		MPI_Recv(buf, 1, colum_data, 0, 0, MPI_COMM_WORLD, &status);
		for(i=0; i<stock; i++)
		{
			for(j=0; j<stock; j++)
				a(i, j) = buf[i*Len+j];
		}

		MPI_Recv(buf, 1, colum_data, 0, 1, MPI_COMM_WORLD, &status);
		for(i=0; i<stock; i++)
		{
			for(j=0; j<stock; j++)
				b(i, j) = buf[i*Len+j];
		}
	}
	
	MPI_Barrier(MPI_COMM_WORLD);

	printf("this is rank %d\n", myrank);
	for(i=0; i<stock; ++i)
	{
  		for(j=0; j<stock; ++j)
  			printf("%d ", a(i,j));
  			printf("\n");
  	}
	
//-------------------------------分配完毕--------------------------------------- 

	for(step = 0; step<sqrt_p; step++)          //执行sqrt(p)次迭代 
	{	
		if(step == 0)         //第一步 
		{
			if(myrank / sqrt_p == myrank % sqrt_p)
			{
				send_m = myrank;
			   	send_i = myrank / sqrt_p;
				
				for(i=0; i<sqrt_p; i++)
				{
					if(send_m % sqrt_p != i)
						MPI_Send(&send_m, 1, MPI_INT, send_i * sqrt_p + i, 0, MPI_COMM_WORLD);
				}
			}
			else
			{
				send_i = myrank / sqrt_p;
				MPI_Recv(&send_m, 1, MPI_INT, send_i * sqrt_p + send_i, 0, MPI_COMM_WORLD, &status);
			}	 
			
			send_ms(send_m, a);
		
			MPI_Barrier(MPI_COMM_WORLD);
		}
	
		
		clm(a, b, c, stock);             //计算 
 	
		     				    			//第三步：B阵向上循环一步 
		if((myrank / sqrt_p) % 2 == 0)      //偶数进程先发送再接收 
		{
			v = myrank / sqrt_p;
			u = myrank % sqrt_p;
			MPI_Send(b,stock*stock, MPI_INT, ((v - 1 + sqrt_p) % sqrt_p) * sqrt_p + u, 0, MPI_COMM_WORLD);
			MPI_Recv(buf,stock*stock, MPI_INT,((v + 1) % sqrt_p) * sqrt_p + u, 0, MPI_COMM_WORLD, &status);
			memcpy(b,buf, sizeof(stock*stock*sizeof(int)));      
		}
		else
		{
			v = myrank / sqrt_p;
			u = myrank % sqrt_p;
			MPI_Recv(buf,stock*stock, MPI_INT,((v + 1) % sqrt_p) * sqrt_p + u, 0, MPI_COMM_WORLD, &status);
			MPI_Send(b,stock*stock, MPI_INT, ((v - 1 + sqrt_p) % sqrt_p) * sqrt_p + u, 0, MPI_COMM_WORLD);
			memcpy(b,buf, sizeof(stock*stock*sizeof(int)));
		}
		
		if(myrank == send_m)        //第四步     
		{ 
			   
//同一组中的，其他线程并不知道send_m改变了，怎么通知？
			int n;
			n =  (myrank/sqrt_p)*sqrt_p + (myrank % sqrt_p + 1) % sqrt_p;      
//用上一次的主发送进程告诉其他人下次主发送进程改变了
			send_i = send_m / sqrt_p;
			for(i=0; i<sqrt_p; i++)
			if(send_m % sqrt_p != i)
				MPI_Send(&n,1, MPI_INT, send_i * sqrt_p + i, 0, MPI_COMM_WORLD);
			send_m = n;    
		}
		else
		{
			MPI_Recv(&send_m,1, MPI_INT, send_m, 0, MPI_COMM_WORLD, &status);
		} 
		
		
		send_ms(send_m, a);
		
		MPI_Barrier(MPI_COMM_WORLD);

	}
 		
 	//-----------------------------------计算完成，收集结果---------------------------------------
	 
	 
	if(myrank != 0)
	{
		MPI_Send(c,stock*stock, MPI_INT, 0, myrank, MPI_COMM_WORLD);
	}
 	else if(myrank == 0)
 	{	
 		for(i=0; i<stock; i++)
 		for(j=0; j<stock; j++)
 		C(i,j) = c(i,j);
 		
 		for(i=1; i<numprocs; i++)
 		{
 			MPI_Recv(c,stock*stock, MPI_INT, i, i, MPI_COMM_WORLD, &status);

			v = i / sqrt_p;
			u = i % sqrt_p;
			
 			for(j=v*stock; j<(v+1)*stock; j++)
 			for(k=u*stock; k<(u+1)*stock; k++)
 			C(j,k) = c(j%stock,k%stock);
 		}
		
 	}
 	
 	
 	if(myrank == 0)
 	{
		printf("the result C muterix \n");
 		out_data(c, Len);
 		end_time = MPI_Wtime();
 		printf("the date M*M=%d p=%d cost time:%lf\n", Len, numprocs, end_time-start_time);
 	}
	 	
	Environment_Finalize(a,b,c);
	MPI_Finalize();
	
	return 0;
}
```
