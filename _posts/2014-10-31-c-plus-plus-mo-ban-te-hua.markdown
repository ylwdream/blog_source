---
layout: post
title: "C++ 模板特化"
date: 2014-10-31 21:55:13 +0800
comments: true
toc: false
categories: C/C++ 
---



有些时候写的模板函数并不适用于所有的情况，这个时候需要对模板进行特化，意思也就是对于特殊情况下使用特化版本的模板。
<!--more-->
1.模板的特化

下面是compare函数模板

``` C++
template<typename T>
int compare(const T& v1, const T& v2)
{
	if(v1 < v2) return -1;
	if(v2 < v1) return 1;
	return 0;
}
```
如果是两个c语言的const char* 实参调用这个函数模板的定义，函数将比较指针的值，而不是字符串本身。所以要定义一个对比较C风格的字符串的特殊定义。这些版本是特化的，而且对用户使用来说是透明的。

**模板特化**：



- 关键字template后面接一对空的尖括号（<>）
- 再接模板名和一对尖括号，尖括号中指定这个特化定义的模板形参
- 函数体列表
- 函数体

``` c++ 
template<>
int compare<char*>(const char * const &v1, const char * const &v2)
{
	return strcmp(v1, v2);
}

```
2.模板中class 和typename关键字

这个我一直不太理解，就是知道typename是可以制定为类型名称，而不是类的静态成员什么的，但是我在vs下面试了下，发现它自动就可以识别到底能否编译通过。好奇怪,下面是网上找的

在模板定义语法中关键字class与typename的作用完全一样。
typename难道仅仅在模板定义中起作用吗？其实不是这样，typename另外一个作用为：使用嵌套依赖类型(nested depended name)，如下所示：
``` C++
class MyArray 
{ 
public：
typedef int LengthType;
.....
}

template<class T>
void MyMethod( T myarr ) 
{ 
typedef typename T::LengthType LengthType; 
LengthType length = myarr.GetLength; 
}
```
这个时候typename的作用就是告诉c++编译器，typename后面的字符串为一个类型名称，而不是成员函数或者成员变量，这个时候如果前面没有
typename，编译器没有任何办法知道T::LengthType是一个类型还是一个成员名称(静态数据成员或者静态函数)，所以编译不能够通过。 

最后说一下stl中的alloc分配内存的函数。
看侯捷的《STL源码剖析》
SGI stl中的内存分配分为两个层次，大于128字节的内存直接使用malloc从堆区申请空间，否则向stl内部维护的一个内存池申请需要的空间。

-----
第一级配置器
__malloc_alloc_template

- allocae()直接使用malloc()
- deallocate直接使用free()函数
- 模拟C++set_new_handler()处理内存不足的情况

第二级配置器
__default_alloc_template

- 维护16个自由链表，负责16中小型区块的次配置能力
- 如果需求大于128字节，就转调第一级配置器

-----

其中第二配置器重的数据结构有

    union obj {
        union obj * free_list_link;
        char client_data[1];    /* The client sees this.        */
	};
sizeof(obj)的大小为4字节，也就是一个内存指针的大小。
union 结构可以节省空间，并且既可以用来当client_data作为内存指针，又可以用来当链表。虽然client_data是一个字节的大小，但是可以用来指向obj的4个内存的大小也正好是4个字节，所以怎么都不会浪费这4个字节的大小。这个地址值给客端使用。
``` C++
 enum {__ALIGN = 8};
  enum {__MAX_BYTES = 128};
  enum {__NFREELISTS = __MAX_BYTES/__ALIGN};
#endif

template <bool threads, int inst>
class __default_alloc_template {

private:
  // Really we should use static const int x = N
  // instead of enum { x = N }, but few compilers accept the former.
# ifndef __SUNPRO_CC
    enum {__ALIGN = 8};
    enum {__MAX_BYTES = 128};  
    enum {__NFREELISTS = __MAX_BYTES/__ALIGN};  //正好16个数
# endif
  static size_t ROUND_UP(size_t bytes) {
        return (((bytes) + __ALIGN-1) & ~(__ALIGN - 1));  //这一点很有意思，感觉位运算很奇妙
  }
__PRIVATE:
  union obj {
        union obj * free_list_link;
        char client_data[1];    /* The client sees this.        */
  };
private:
# ifdef __SUNPRO_CC
    static obj * __VOLATILE free_list[]; 
        // Specifying a size results in duplicate def for 4.1
# else
    static obj * __VOLATILE free_list[__NFREELISTS]; 
# endif
  static  size_t FREELIST_INDEX(size_t bytes) {
        return (((bytes) + __ALIGN-1)/__ALIGN - 1);  //查找在free_list中的位置
  }

  // Returns an object of size n, and optionally adds to size n free list.
  static void *refill(size_t n);
  // Allocates a chunk for nobjs of size size.  nobjs may be reduced
  // if it is inconvenient to allocate the requested number.
  static char *chunk_alloc(size_t size, int &nobjs);

  // Chunk allocation state.
  static char *start_free;
  static char *end_free;
  static size_t heap_size;

public:

  /* n must be > 0      */
  static void * allocate(size_t n)
  {
    obj * __VOLATILE * my_free_list;
    obj * __RESTRICT result;

    if (n > (size_t) __MAX_BYTES) {
        return(malloc_alloc::allocate(n));
    }
    my_free_list = free_list + FREELIST_INDEX(n);
    result = *my_free_list;
    if (result == 0) {
        void *r = refill(ROUND_UP(n));
        return r;
    }
    *my_free_list = result -> free_list_link;
    return (result);
  };

  /* p may not be 0 */
  static void deallocate(void *p, size_t n)
  {
    obj *q = (obj *)p;
    obj * __VOLATILE * my_free_list;

    if (n > (size_t) __MAX_BYTES) { //大于128直接使用第一级的deallocate
        malloc_alloc::deallocate(p, n);
        return;
    }
    my_free_list = free_list + FREELIST_INDEX(n); //找到是哪个位置 8， 16， 32？？
    q -> free_list_link = *my_free_list;  //使用头插法插入空余的空间，o(1)的复杂度
    *my_free_list = q;
  }

  static void * reallocate(void *p, size_t old_sz, size_t new_sz);

} ;

```
