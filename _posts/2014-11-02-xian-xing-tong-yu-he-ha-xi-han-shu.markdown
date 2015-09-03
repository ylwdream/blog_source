---
layout: post
title: "线性同余和哈希函数"
date: 2014-11-02 03:20:43 -0800
comments: true
categories: algorithm
---


这几天在学习概率算法，觉得在误差允许的范围内,概率算法是可以接收的。而且往往在现实应用中，应用很广。
<!--more-->

## 线性同余 ##
线性同余方法（LCG）是个产生伪随机数的方法。

它是根据递归公式：

$$ N_j + 1 \equiv (A \ast Nj + B)(\mod M) $$

其中A,B,M是产生器设定的常数。

LCG的周期最大为M，但大部分情况都会少于M。(生成M个数之后序列重复)
## rand 函数
c/c++中的rand()函数就是采用这个方法实习的。

C语言中伪随机数生成算法实际上是采用了"线性同余法"。
具体的计算如下：

    Xi = (Xi-1 * A + C ) mod M

其中A,C,M都是常数（一般会取质数）。当C=0时，叫做乘同余法。引出一个概念叫seed，它会被作为X0被代入上式中，然后每次调用rand()函数都会用上一次产生的随机值来生成新的随机值。可以看出实际上用rand()函数生成的是一个递推的序列，一切值都来源于最初的 seed。所以当初始的seed取一样的时候，得到的序列都相同。

C语言里面有RAND_MAX这样一个宏，定义了rand()所能得到的随机值的范围。在C里可以看到RAND_MAX被定义成0x7fff，也就是32767。rand()函数里递推式中M的值就是32767。

现在一种较好的生成随机数的方法是：[梅森旋转算法](http://zh.wikipedia.org/wiki/%E6%A2%85%E6%A3%AE%E6%97%8B%E8%BD%AC%E7%AE%97%E6%B3%95)

随机数在概率算法中很重要，尤其是MC算法。


## 哈希函数 ##

哈希表（Hash table，也叫散列表），是根据关键码值(Key value)而直接进行访问的数据结构。也就是说，它通过把关键码值映射到表中一个位置来访问记录，以加快查找的速度。这个映射函数叫做散列函数，存放记录的数组叫做散列表。

哈希函数映射的过程一般是不可逆的，例如MD5算法。将任意长的字符串对应到128bytes上，一般用于摘要的提取和信息指纹的提取。

哈希表在查找和插入的复杂度基本都是o(1),所以效率极高。
(不知道数据库中的索引是hash还是b树，后者空间占用下，但是会慢一点，估计是后者吧。因为hash我感觉是封闭状态的应用。。)

hash表有多种实现方式，后面的参考中都进行了详细的介绍，综合效率和准确度等等。
## 哈希实现

看一种最简单实用的版本,把字符串转换为无符号的长整数。

``` c 
unsigned long HashString(char *lpszString)
{
unsigned long ulHash = 0xf1e2d3c4;

while (*lpszString != 0)
{
ulHash <<= 1;
ulHash += *lpszString++;
}

return ulHash;
}
```
## Bloom Filter 算法
下面说一下**Bloom Filter的算法 **

Bloom Filter算法如下：

创建一个m位BitSet，先将所有位初始化为0，然后选择k个不同的哈希函数。第i个哈希函数对字符串str哈希的结果记为h（i，str），且h（i，str）的范围是0到m-1 。

下面给出一种C++的实现版本吧。
``` c++ BloomFilter.cpp
#include <vector>
#include <string>
#include <iostream>
#include <string.h>

using namespace std;

#define DEFAULT_SIZE (1 << 25)//分配2^24个bit
// c++ 中bool 为1个字节，占8个位
 
class simple_hash
{
public:
	simple_hash(int cap, int seed)
	{
		this->cap = cap;
		this->seed = seed;
	}
	int hash(const string &str)
	{
		int result = 0;
		int len = str.length();
		for(int i=0; i< len; ++i)
			result = seed * result + str[i];
		return (cap - 1) & result;      //取模的意思,等价于 result % cap
	}
private:
	int cap;
	int seed;
};

class BloomFilter 
{
public:
	BloomFilter()
	{
		maxSize = DEFAULT_SIZE;
		int size = maxSize / 8;
		bits = new char[size];
		memset(bits, 0, sizeof(bits));
		int len = sizeof(seeds) / sizeof(int);
		for(int i=0; i<len; ++i)
		{
			func[i] = new simple_hash(maxSize, seeds[i]);
		} 
	}
	void addString(const string &value)
	{
		auto len = func.size();
		for(int i=0; i<len; i++)
		{
			auto code = func[i]->hash(value);
			auto pos = code / 8;
			auto bit_index = 1 << (code % 8);
			bits[pos] |= bit_index;
		}
	}
	
	//判断是否所有的位都被标记了 
	bool cantians(const string &value)
	{
		char ret = 1;      //00000001 
		int len = func.size();
		for(int i=0; i<len; ++i)
		{
			auto code = func[i]->hash(value);             //  c++ 没有位操作，真麻烦。。。。 
			auto pos = code / 8;                         // 000010000 
			auto bit_index = 1 << (code % 8);            // 先求出哪一位然后再把那一位右移动到第0位 
			ret = ret && ((bits[pos] && bit_index) >> (code % 8)); 
		}
		if(ret&1) return true;
		else return false;
	}
	
private:
	vector<int> seeds{13, 31, 131, 1313}; 
	vector<simple_hash*> func;
	static int maxSize;
	char *bits;    //一个char占8位 
};

int main()
{
	cout << sizeof(bool) <<endl;
	return 0;
}
```
## 位级运算
再次吐槽c++没有不支持原始的位运算真麻烦。。。c++11中的位容器貌似只支持到长整形那么多个字节。
先总结到这吧。


## **参考**

- [纯线性同余随机数生成器](http://www.cnblogs.com/xkfz007/archive/2012/03/27/2420154.html)
- [解析Hash表算法](http://blog.csdn.net/v_july_v/article/details/6256463) 
- [BloomFilter——大规模数据处理利器](http://www.cnblogs.com/heaad/archive/2011/01/02/1924195.html)
- [http://sfsrealm.hopto.org/inside_mopaq/chapter2.htm](http://sfsrealm.hopto.org/inside_mopaq/chapter2.htm)
