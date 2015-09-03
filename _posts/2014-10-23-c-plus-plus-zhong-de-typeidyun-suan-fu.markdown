---
layout: post
title: "c++ 中的typeid运算符"
date: 2014-10-23 10:01:33 -0700
comments: true
categories: C/C++
---
## typeid 运算符


typeid 运算符主要是为了支持c++的RTTI(Runtime Type Identification).在有虚拟集成的类结构中，也就是有多态的情况下，当需要知道动态执行的指针或引用的类的类型时被使用。当然它也可以用在内置类型或者是没有虚函数的类继承的结构中。typeid返回的类型是`const std::type_info`。 
<!--more-->
看网上说typeid是定义在typeinfo头文件里的，但是我并没有找到定义，不知道为什么。

    typeid( expression )

  1. 当expression是内置类型或者是没有多态的情况下，typeid不对表达式评估求值,并且typeid指向的是静态类型，在编译的阶段就可以确定下来。
对于内置类型，MSVC下面是这么定义的。

    VCCORLIB_API Platform::Type^ float64::GetType()
	{
		return default::float64::typeid;
	}

  2. 当expression含有虚函数时，typeid就需要在执行期确定，这也正是为了实现RTTI机制。
typeinfo可以根据指针指向的类型，去取这个类型中的虚表的中的类的类型定义。在MSVC中，将类的typeinfo name 放到虚表的槽号为-1的位置。

{% img /img/blog-img/typeinfo.jpg %}

可以看到继承关系中，父类指针和派生类指针指向的开始地址都是一样的，只是指向的对象类型不同，所以向上转换时是安全的，但是向下转换时却有可能发生bad_typeid的错误。
## typeinfo 定义
下面是标准库中队typeinfo的定义
``` c++
	class type_info {
	public:
	    SECURITYCRITICAL_ATTRIBUTE
	    _CRTIMP virtual __thiscall ~type_info();
	    _CRTIMP int __thiscall operator==(_In_ const type_info& _Rhs) const;
	    _CRTIMP int __thiscall operator!=(_In_ const type_info& _Rhs) const;
	    _CRTIMP bool __thiscall before(_In_ const type_info& _Rhs) const;
	#ifdef _SYSCRT
	    _Check_return_ _CRTIMP const char* __thiscall name() const;
	#else  /* _SYSCRT */
	    _Check_return_ _CRTIMP const char* __thiscall name(_Inout_ __type_info_node* __ptype_info_node = &__type_info_root_node) const;
	#endif  /* _SYSCRT */
	    _Check_return_ _CRTIMP const char* __thiscall raw_name() const;
	private:
	    void *_M_data;
	    char _M_d_name[1];
	    __thiscall type_info(_In_ const type_info& _Rhs);
	    type_info& __thiscall operator=(_In_ const type_info& _Rhs);
	};
	#ifndef _TICORE
```

> type_info类提供了public虚 析构函数，以使用户能够用其作为基类。它的默认构造函数和拷贝构造函数及赋值操作符都定义为private，所以不能定义或复制type_info类型的对象。
其中M_d_name[1]为一个字符串指针，指向的是类的名字。可以看下面的代码实现
    
	char *tmp = (char*)malloc(sizeof(class name));
	M_d_name = tmp

## C++ 四种类型转换

关于c++定义的四种转换的关键字
ANSI-C++标准定义了四个新的转换符：`reinterpret_cast`, `static_cast`, `dynamic_cast` 和 `const_cast`，目的在于控制类(class)之间的类型转换。

代码:

    reinterpret_cast<new_type>(expression)
    dynamic_cast<new_type>(expression)
    static_cast<new_type>(expression)
    const_cast<new_type>(expression)
- reinterpret_cast 是特意用于底层的强制转型，导致实现依赖（implementation-dependent）（就是说，不可移植）的结果，例如，将一个指针转型为一个整数。这样的强制转型在底层代码以外应该极为罕见
- dynamic_cast 主要用于执行“安全的向下转型（safe downcasting）”，也就是说，要确定一个对象是否是一个继承体系中的一个特定类型。它是唯一不能用旧风格语法执行的强制转型，也是唯一可能有重大运行时代价的强制转型。
- static_cast 可以被用于强制隐型转换（例如，non-const 对象转型为 const 对象，int 转型为 double，等等,不进行安全检查，效率比dynamic_cast 效率高。它还可以用于很多这样的转换的反向转换（例如，void* 指针转型为有类型指针，基类指针转型为派生类指针），但是它不能将一个 const 对象转型为 non-const 对象（只有 const_cast 能做到），它最接近于C-style的转换。
- const_cast 一般用于强制消除对象的常量性。它是唯一能做到这一点的 C++ 风格的强制转型。
## 实验
	
下面是我做的小实验的代码，GUN Gcc 和MSVC的输出可能会有不一样。
``` c++	
	#include <iostream>
	#include <typeinfo>
	
	using namespace std;
	
	class Base
	{
	public:
		int number = 0;
		virtual void print()
		{
			cout << "the base " << number << endl;
		}
	};
	
	class Book : public Base
	{
	public:
		void print()
		{
			Base::number++;
			cout << "the book " << Base::number << endl;
		}
	private:
		int price;
	};
	void classType(Base *p)
	{
		cout << typeid(*p).name() << endl;
		if (typeid(*p) == typeid(Book))
		{
			Book *bo = static_cast<Book*>(p);
			cout << "static cast ";
			bo->print();
		}
		try{
	
			// Book *bo = static_cast<Book *>(p);	
			if (Book *bo = dynamic_cast<Book *>(p))
				bo->print();
			//else if(typeid(p) == typeid(Base))
			else if (Base *base = dynamic_cast<Base *>(p))
			{
				//Base *base = static_cast<Base *>(p);
				base->print();
			}
		}
		catch (...)
		{
			cerr << "no type" << endl;
		}
	}
	
	struct Base2 {}; // non-polymorphic
	struct Derived2 : Base2 {};
	
	int main()
	{
		Base *base;
		base = new Book;
		std::cout << "reference to polymorphic base: " << typeid(*base).name() << '\n';
		
		Derived2 d2;
		Base2& b2 = d2;
		std::cout << "reference to no-polymorphic base: " << typeid(b2).name() << '\n';
	
		base->print();
		classType(base);
		classType(new Base);
	
		
		
		int var;
		cout << typeid(var).name() << endl;
		cout << typeid(var).raw_name() << endl;
		delete base;
		cin >> var;
		return 0；
	}
```
## 总结
类中的函数和对象经过mangling会成为一个唯一命名的函数或者数据实例。我认为这也是为什么static 数据定义一次，下次使用是数据的值不会被再次初始化，就是因为编译器把它的定义域扩充为全局的。而这些只在mian函数前面由.init函数进行初始化。

因为base是Base\*类型，所以base指针就可以知道Base类的起始地址，在编译的时候也就是知道了print函数的slots号。（这里为1）所以对print函数的调用，就可能变为 `*base->vptr[1](base)`。在执行的时候，base指向的对象为Book,Book里的vptr指针指向的是Book的虚函数表，而这在编译阶段是无法确定的，编译阶段唯一可以确定的就是base的指针类型。因为base现在是指向Book类，所以就会指针Book类中，槽号为1的虚函数。
暂时就是终结这些。还有好多疑问没有搞清楚。
