---
layout: post
title: "设计模式之工厂模式"
date: 2014-10-28 23:41:38 +0800
comments: true
toc: false
categories: 
- 设计模式

---

今天看了一点设计模式，是关于工厂模式的，简单的说就是你不会生产一个工具，交给工厂去生产，你只用告诉工厂工具的尺寸就好。
<!--more-->
比如你想要印100元大钞，你没有印钞机，只要告诉有印钞机的工厂，他就会给你返回一个100元的大钞。

下面是《大话设计模式》书上的设计计算器例子，原例子用c#写的，我把他改为了c++，记录下，也为了以后的学习。

> Operation 是基础类，体现了面向对象的封装/继承和多态。

``` C++ Operation.h

#ifndef _OPERATION_H
#define _OPERATION_H
#include <string>

class Operation
{
public:
	virtual double getResult() 
	{
		result = 0;
		return result;
	}
	inline void set_A(double val) { this->numberA = val; }
	inline double get_A() { return numberA; }
	inline void set_B(double val) { this->numberB = val; }
	inline double get_B() { return numberB; }
	virtual ~Operation(){}

protected:
	double numberA, numberB;
	double result;
};

class OperationFactory
{

public:
	static Operation * createOperate(std::string);
};
#endif
```
> 下面是四则运算和工厂类的实现

``` C++ Operation.cpp

#include "Operation.h"

class OperationAdd : public Operation
{
public:

	double getResult()
	{
		result = numberA + numberB;
		return result;
	}

	~OperationAdd(){}
};

class OperationSub : public Operation
{
public:

	double getResult()
	{
		result = numberA - numberB;
		return result;
	}
	~OperationSub(){}
};

class OperationMul : public Operation
{
public:

	double getResult()
	{
		result = numberA * numberB;
		return result;
	}
	~OperationMul(){}
};

class OperationDiv : public Operation
{
public:

	double getResult()
	{
		try
		{
			result = numberA / numberB;
			return result;
		}
		catch (...)
		{
		}
		 
	}
	~OperationDiv(){}
};

Operation * OperationFactory::createOperate(std::string operate)
{
	Operation * oper = NULL;       //出了作用域对象自动销毁

	if (operate == "+")          //c++ switch不支持string
	{
		oper = new OperationAdd();
	}
	else if (operate == "-")
	{
		oper = new OperationSub();
	}
	else if (operate == "*")
	{
		oper = new OperationMul();
	}
	else if (operate == "/")
	{
		oper = new OperationDiv();
	}

	return oper;
}

```

> mian.cpp是测试代码。我理解的工厂模式就是封装了要使用的类的new操作。不让用户看到工厂内具体是如何生产工具的细节。

``` C++ main.cpp
#include "Operation.h"
#include <iostream>

using namespace std;

int main()
{
	Operation *oper;
	oper = OperationFactory::createOperate("+");
	oper->set_A(1);
	oper->set_B(2);
	double result = oper->getResult();
	cout << result << endl;
	oper->~Operation();       //调用析构函数
	cin >> result;
}
```
