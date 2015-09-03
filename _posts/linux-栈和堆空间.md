title: linux 栈和堆空间
date: 2015-05-23 11:21:14
categories:
- linux
toc: true
tags:
 - memory
---

## 运行时栈

程序的运行离不开栈空间和堆空间。栈中保存着程序运行的局部变量还有最重要的是，维持了函数调用的活动记录和关系。因为程序语言设计的允许递归调用，所以程序的局部变量必须存储在栈空间中，每一次函数调用都会有不同的活动记录。而函数执行完成后会回收当前的栈帧，所以无法通过函数返回值来访问回收后的局部变量，当然，`static` 变量除外。 因为编译器遇到`static` 变量时是按全局变量处理的，在符号表中有`static`变量的位置。只不过其访问权限只能在函数内。

<!--more-->

## 活动记录
活动记录又称为栈帧，一般保存如下几个方面的内容。

![]( /img/blog/stack_frame.png)

- 函数的返回地址和参数
- 临时变量：包括函数的非静态局部变量以及编译器自动生成的其他的临时变量。
- 保存的上下文：包括在函数调用前后需要保持不变的寄存器。

----

函数调用的过程如下：
- 把函数的参数压入栈中，或者通过寄存器传递参数，或者是内存的共享变量
- 把当前指令的下一条指令压入栈中。(call 指令)
- 跳转到具体函数执行。(设置eip的值为调用函数的入口地址)


## 栈大小

1. 栈默认是向下生长的，也就是栈底在搞地址，栈顶在底地址。如下图所示

![]( /img/blog/stack_model.png)

2. 栈空间默认大小在我的机器上是10M。可以通过`ulimit -a` 来查看系统设置的资源。其中设置的栈空间默认大小为10M。但是我却不知道栈底到底是怎么设置和映射的，应该是`exec`函数在读取可执行文件后，把 elf 文件的数据段和代码段分别映射到对应的虚拟地址空间后，剩下的空间映射为进程的栈空间和堆空间。

![]( /img/blog/stack_size.png)

3. 我写了一个程序来测试机器上的栈空间大小，其中有一段嵌入式汇编。基本想法是：利用函数执行前和执行后esp指针之差来估算栈的大小，基本也是10M大小。
``` c 
#include <stdio.h> 
#include <pthread.h> 

int i = 1; 

unsigned int stack_begin = 0;
unsigned int stack_end = 0;
unsigned int stack_size = 0;

// 嵌入式汇编
#define read_esp(stack_addr) \ 
({ \
	__asm__ (\
		"movl %%esp, %%eax; \
		 movl %%eax, %0;"	\
		:"=r"(stack_addr)); \
})

void *test() 
{ 

	if(i == 1)
	{
		read_esp(stack_begin);  //第一次读取esp 的值
	}
	
	read_esp(stack_end);
	printf("esp size = %d\n", stack_begin - stack_end);   //栈自顶向下
	
	int buffer[1024]; 
	printf("i=%d\n", i); 
	i++; 
	test(); 
} 


int main() 
{ 

	pthread_t p; 
	read_esp(stack_begin);
	pthread_create(&p, NULL, &test, NULL); 
	
	sleep(100); 
} 
```
其中栈空间大小可以通过`ulimit -s` 来设置大小。在此我就不测试了。
![]( /img/blog/stack_size_test.png)
通过程序的运行结果可以看出来，运行过程中出现的`segement fault` 就是栈溢出了。

> 通过`cat proc/stack/maps` 查看进程的映射信息发现栈空间开始地址为`bf885000`

![]( /img/blog/stack_mem.PNG)


## 函数返回值传递

函数的参数一般是通过eax作为返回值，但是当函数参数大于4字节。对于5~8 字节的数据，默认是按`eax, edx` 联合返回的。但是一个结构体或者一个类对象，那么是如何处理呢? 

### 结构体

测试的代码如下，为了方便，也给出了对应的AT&T 的汇编代码。
首先声明了一个结构体，大小为100*4 = 400 个字节。 400 = 0x190 (16进制)
用 `objdump -d test.o` 可以看到程序对应的源代码。

``` c
typedef struct big_thing
{
	int buf[100];
}big_thing;

big_thing return_test()
{
	big_thing b;
	b.buf[0] = 1;
	return b;
}

int main()
{
	big_thing n;
	n = return_test();
}
```
``` c
00000000 <return_test>:
   0:	55                   	push   %ebp
   1:	89 e5                	mov    %esp,%ebp
   3:	57                   	push   %edi
   4:	56                   	push   %esi
   5:	53                   	push   %ebx
   6:	81 ec 90 01 00 00    	sub    $0x190,%esp       	//为 b 分配空间
   c:	c7 85 64 fe ff ff 01 	movl   $0x1,-0x19c(%ebp) 	// b的起始地址
  13:	00 00 00 
  16:	8b 45 08             	mov    0x8(%ebp),%eax  		// eax = &n;
  19:	89 c3                	mov    %eax,%ebx        	// ebx = &n;
  1b:	8d 85 64 fe ff ff    	lea    -0x19c(%ebp),%eax	// eax = &b;
  21:	ba 64 00 00 00       	mov    $0x64,%edx			// edx = 100;(10进制)
  26:	89 df                	mov    %ebx,%edi			// edi = &n;
  28:	89 c6                	mov    %eax,%esi 			// esi = &b
  2a:	89 d1                	mov    %edx,%ecx			// ecx = 100;
  2c:	f3 a5                	rep movsl %ds:(%esi),%es:(%edi) //复制
  2e:	8b 45 08             	mov    0x8(%ebp),%eax		// eax = &n
  31:	81 c4 90 01 00 00    	add    $0x190,%esp			// 平衡栈空间
  37:	5b                   	pop    %ebx
  38:	5e                   	pop    %esi
  39:	5f                   	pop    %edi
  3a:	5d                   	pop    %ebp
  3b:	c2 04 00             	ret    $0x4

0000003e <main>:
  3e:	55                   	push   %ebp
  3f:	89 e5                	mov    %esp,%ebp
  41:	81 ec 90 01 00 00    	sub    $0x190,%esp     //为 n 分配空间
  47:	8d 85 70 fe ff ff    	lea    -0x190(%ebp),%eax //eax = n的地址值
  4d:	50                   	push   %eax              //压栈
  4e:	e8 fc ff ff ff       	call   4f <main+0x11>    //调用函数
  53:	c9                   	leave  
  54:	c3                   	ret    
```

> 我把程序的空间图画出来，如下图所示:  

![]( /img/blog/stack_struct.png)

从上图可知，是先把 n的地址通过eax压入到堆栈中，作为 `return_test` 的参数。实际的函数可以理解为：`void return_test(big_thing *n)`。 可以推测编译器做了一定的优化，通过eax作为变量来传递结构体的地址，而`return_test`运行结束后，eax 仍然保存着变量n的地址。这也保证了main函数中栈空间eax值仍然不变。即eax是作为保护寄存器参数。
### 类对象

c++ 中的类对象与c中的对象有所不同，因为c++ 中可以定义赋值构造函数可以重载赋值操作符还有c++ 中的this 关键字。
``` c++
#include <iostream>

using namespace std;

struct big_thing
{
public:
	big_thing()
	{
		cout << "big_thing construct" << endl;
	}
	big_thing(const big_thing &rhs)
	{
		cout << "big_thing copy construct" << endl;
	}

	big_thing& operator=(const big_thing &rhs)
	{
		cout << "big_thing operator =" << endl;
	}
	~big_thing()
	{
		cout << "big_thing destruct" << endl;
	}
	int buf[100];
	
};

big_thing return_test()
{
	big_thing b;
	b.buf[0] = 1;
	return b;
}

int main()
{
	big_thing n;
	n = return_test();
}
```
运行结果
``` c
big_thing construct
big_thing construct
big_thing operator =
big_thing destruct
big_thing destruct
```
## 总结
堆栈是程序运行过程中很重要的空间分配策略，但是当访问到不能访问的地址，或者栈空间不足时都会发生内存访问错误。要对错误进行分析，可以结合gdb的调试功能。
未完，下一篇谢谢关于malloc堆的分配问题。

## 参考
- 《程序员的自我修养》
- [gcc 嵌入汇编](http://oss.org.cn/kernel-book/ch02/2.6.3.htm)