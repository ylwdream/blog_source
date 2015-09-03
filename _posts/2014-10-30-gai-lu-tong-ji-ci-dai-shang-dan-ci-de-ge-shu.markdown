---
layout: post
title: "概率统计磁带上单词的个数"
date: 2014-10-30 23:03:59 +0800
comments: true
categories: algorithm 
---


假设磁带上记录有Shakespeare全集，如何统计 其中使用了多少个不同的单词？为简单起见，同一词 的复数，被动语态等可作为不同项。

<!--more-->
## 算法设计
方法一：对磁带上的单词进行外部排序，时间θ(NlgN),空间需求较大

方法二：在内存中建一散列表，表中只存储首次出现的单 词，平均时间O(N)，空间Ω(n)

方法三：若能忍受某种误差及已知n或N的上界M，则存在 一个时空性能更好的概率算法解此问题。 设U是单词序列的集合，设参数m稍大于lgM，可令
 `m = 5 + int(lgM)`

此种方法要知道单词的上限，而且是概率的算法。
取一个y[m+1]位的数组。

对每个单词采用hash映射为一个m为的数。若单词的从左到右最后一个出现的1的位置index，另y[index] = 1,记 y[m+1]中从左到右第一个为0的位置M。则单词的个数大概就是2<<M个。

下面是我写的代码。
## 实现代码
``` C++ WordCount.cpp
#include <string.h>
#include <iostream>
#include <fstream>
#include <cmath>

using namespace std;

class wordCount
{
public:
	wordCount(int max)
	{
		this->len = (int)log(max) + 1;
		this->max = (2 << len);
		sum = 0;
	}
	int get_first_num_0(string str)
	{
		unsigned int sum =0;
		
		for(int i=0; i< str.length(); ++i)
		{
			sum = (str[i] - '0') * 10 + sum;
		}
		return sum % max;
	}
	void set_num_1(int index)
	{
		sum = sum | (1 << index);
	}
	int get_num_1(unsigned short word) 
	{
		int index = 0;         //找1出现的最高位置 
		
		for(int i = 0; i <= len; ++i)
		{
			if((word >> i) && 1)
			{
				index = i; 
			}
		}	
		return index;
	}
	int result()
	{
		int index = 0;
		for(int i=0; i<=len; ++i)
		{
			if(((sum >> i))&& 1)
			{
				index = i;
			}
		}
		return index;
	}
	
private:
	unsigned int sum;
	unsigned int len;
	unsigned int max;	
};
 
int main(int argc, char *argv[])
{
	
	ifstream infd;
	infd.open("in.txt");
	
	std::string word;
	wordCount wcount(20);

	while(getline(infd, word, ' ')) 
	{ 
		if(word == "exit")
		break;
		int index = wcount.get_num_1(wcount.get_first_num_0(word));
		wcount.set_num_1(index);
	}
	
	int total_word = int((2 << wcount.result()) / 1.54703);
	printf("the total words is %d\n", total_word);
	infd.close();
	return 0;
}
```
## 总结

至于最后为什么要除那个系数，我现在也不懂。

