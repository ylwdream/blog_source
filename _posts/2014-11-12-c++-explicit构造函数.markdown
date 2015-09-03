---
layout: post
title: "c++ explicit构造函数"
date: 2014-11-12 01:08:43
comments: true
toc: false
categories: 
- C/C++
tags:
- C/C++
---



当c++的构造函数只有一个参数时，有时候在表达式运算时会调用c++隐式转换，生成一个临时对象。
按照默认规定，只有一个参数的构造函数也定义了一个隐式转换，将该构造函数对应数据类型的数据转换为该类对象。

<!--more-->
下面的代码是没有加explicit关键字，可以编译运行，默认会调用A的单参数构造函数，但是加了explicit就会编译错误。

隐式转换的坏处就是有可能会发生错误，令程序很难调试，第二就是会生成临时对象，增加构造和析构的负担。

``` c++
#include <iostream>

class A
{

public:
	
	A(int value)   //没有explicit
	{
		this->value = value;
	}

	A(A& a)
	{
		this->value = a.value;
	}
	void print()
	{
		std::cout << value << std::endl;	
	}
private:
	int value;
};

int main()
{
	A a = 2;    //调用隐式转换

	a.print();

	return 0;
}
```



请记住：

提防隐式转换所带来的微妙问题，尽量控制隐式转换的发生；通常采用的方式包括：  
1. 使用非C/C++关键字的具名函数，用operator as_T()替换operato T()（T为C++数据类型）。  
2. 为单参数的构造函数加上explicit关键字。

***参考：***

[http://www.cnblogs.com/cutepig/archive/2009/01/14/1375917.html](http://www.cnblogs.com/cutepig/archive/2009/01/14/1375917.html)
[http://book.51cto.com/art/201202/317579.htm](http://book.51cto.com/art/201202/317579.htm)
