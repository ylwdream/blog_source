---
layout: post
title: "C++ new 运算符"
date: 2014-10-31 03:17:22
comments: true
toc: false
categories: 
- C/C++
tags:
- C/C++
---
new 运算符调用函数 `operator new`。 对于任何类型的数组以及不属于 `class、struct` 或 `union` 类型的对象，调用全局函数 `::operator new` 来分配存储。 类类型对象可基于每个类定义其自己的 `operator new` 静态成员函数。
当编译器遇到用于分配类型 type 的对象的 `new` 运算符时，它将发布对 `type::operator new( sizeof( type ) )` 的调用；或者，如果未定义用户定义的 `operator new`，则为 `::operator new( sizeof( type ) )`。 因此，new 运算符可以为对象分配正确的内存量。

首先看下面这句表达式
``` c++
string *sp = new string("initalliszed");
```
<!--more-->

上面实际发三个步骤。首先，该表达式调用名为operator new 的标准库函数，分配足够大的原始的未类型化的内存，以保持指定类型的一个对象；接下来，运行该类型的一个构造函数，来指定初始化构造对象；最后，返回指向新分配并构造的对象的指针。
``` c++
  string *sp = new string("hello");
0121439F  call        operator new (0121137Ah)  
012143A4  add         esp,4  
012143A7  mov         dword ptr [ebp-0E0h],eax  
012143AD  mov         dword ptr [ebp-4],0  
012143B4  cmp         dword ptr [ebp-0E0h],0  
012143BB  je          main+75h (012143D5h)  
012143BD  push        121CC80h  
012143C2  mov         ecx,dword ptr [ebp-0E0h]  
012143C8  call        std::basic_string<char,std::char_traits<char>,std::allocator<char> >::basic_string<char,std::char_traits<char>,std::allocator<char> > (01211398h)  
012143CD  mov         dword ptr [ebp-0F4h],eax  
012143D3  jmp         main+7Fh (012143DFh)  
012143D5  mov         dword ptr [ebp-0F4h],0  
012143DF  mov         eax,dword ptr [ebp-0F4h]  
012143E5  mov         dword ptr [ebp-0ECh],eax  
012143EB  mov         dword ptr [ebp-4],0FFFFFFFFh  
012143F2  mov         ecx,dword ptr [ebp-0ECh]  
012143F8  mov         dword ptr [sp],ecx  
```

new 运算符调用函数 operator new。 对于任何类型的数组以及不属于 class、struct 或 union 类型的对象，调用全局函数 ::operator new 来分配存储。 类类型对象可基于每个类定义其自己的 operator new 静态成员函数。


>* operator new 的第一个参数的类型必须为 size_t（STDDEF.H 中定义的类型），并且返回类型始终为 void *。

在使用 new 运算符分配内置类型的对象、不包含用户定义的 operator new 函数的类类型的对象和任何类型的数组时，将调用全局 operator new 函数。 在使用 new 运算符分配类类型的对象时（其中定义了 operator new），将调用该类的 operator new。

为类定义的 operator new 函数是静态成员函数（因此，它不能是虚函数），该函数隐藏此类类型的对象的全局 operator new 函数。 

下面是微软官网给出的例子

``` C++ new.cpp
#include <malloc.h>
#include <memory.h>
#include <iostream>
using namespace std;

class Blanks
{
public:
    Blanks(){

		cout << "hello, the number i " << number <<endl;
		number++;
	}
    void *operator new( size_t stAllocateBlock, char chInit );
static int number;
};

int Blanks::number = 1;

void *Blanks::operator new( size_t stAllocateBlock, char chInit )
{
    void *pvTemp = malloc( stAllocateBlock );
    if( pvTemp != 0 )
        memset( pvTemp, chInit, stAllocateBlock );
	cout << "in the operator new function" <<endl;
	Blanks();
    return pvTemp;
}
// For discrete objects of type Blanks, the global operator new function
// is hidden. Therefore, the following code allocates an object of type
// Blanks and initializes it to 0xa5
int main()
{
   Blanks *a5 = new(0xa5) Blanks;
   Blanks *b5;
   b5->operator new(1,0xa5);
   Blanks c5;
   int i;
   cin >> i;
   return 0;
   
}
```
从输出的结果可以看出来new运算符的a5第一个写法是先调用operator new函数，再调用Blanks的构造函数。而b5只是调用了一次operator new函数。c5的输出结果表明是先分配了内存，然后调用了构造函数。下面是看到的一部分汇编代码
``` c
	Blanks *a5 = new(0xa5) Blanks;
012E509E  push        0FFFFFFA5h  
012E50A0  push        4  
012E50A2  call        Blanks::operator new (012E121Ch)  
012E50A7  add         esp,8  
012E50AA  mov         dword ptr [ebp-0F8h],eax  
012E50B0  cmp         dword ptr [ebp-0F8h],0  
012E50B7  je          main+4Ch (012E50CCh)  
012E50B9  mov         ecx,dword ptr [ebp-0F8h]  
012E50BF  call        Blanks::Blanks (012E1212h)  //可以看到Blanks的入口地址是012E1212h
012E50C4  mov         dword ptr [ebp-100h],eax  
012E50CA  jmp         main+56h (012E50D6h)  
012E50CC  mov         dword ptr [ebp-100h],0  
012E50D6  mov         eax,dword ptr [ebp-100h]  
012E50DC  mov         dword ptr [a5],eax  
   Blanks *b5;
   b5->operator new(1,0xa5);
012E50DF  push        0FFFFFFA5h  
012E50E1  push        1  
012E50E3  call        Blanks::operator new (012E121Ch)  
012E50E8  add         esp,8  
   Blanks c5;
012E50EB  lea         ecx,[c5]  //把c5的内存地址送给ecx,这个地方好像并没有分配内存
012E50EE  call        Blanks::Blanks (012E1212h)  
```
其实可以自己写一点简单的类，研究汇编代码。记录下。
还有一种就是定位new表达式，就是在预先分配的内存上来构造对象。这里也不介绍了。

可以看出来系统的new 运算符分为两类，第一是为系统分配内存，第二是调用类的构造函数来完成类的构造过程。下一步研究如何看这两个过程：分配内存与构造。

