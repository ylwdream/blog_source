---
layout: post
title: "最小堆的性质"
date: 2014-11-02 01:05:16 -0700
comments: true
toc: false
categories: algorithm
---

最小堆是经过排序的完全二叉树。对于查找一组数据的前n个元素具有很好的复杂度。
堆是一种经过排序的完全二叉树，其中任一非终端节点的数据值均不大于（或不小于）其左孩子和右孩子节点的值。  
最大堆和最小堆是二叉堆的两种形式。
- 最大堆：根结点的键值是所有堆结点键值中最大者。
- 最小堆：根结点的键值是所有堆结点键值中最小者。

而最大-最小堆集结了最大堆和最小堆的优点，这也是其名字的由来。  
最大-最小堆是最大层和最小层交替出现的二叉树，即最大层结点的儿子属于最小层，最小层结点的儿子属于最大层。  
以最大（小）层结点为根结点的子树保有最大（小）堆性质：根结点的键值为该子树结点键值中最大（小）项
<!--more-->
性质：

- 根结点的键值是所有堆结点键值中最小者。
- 父节点比左右节点的值都要小。


在数据量比较大的情况下，想要统计前K个最小的数，利用最小堆可以有很好的性能，只需要比较待排定的节点是否比当前最小堆的堆首节点即可，如果比堆首节点小，则插入对中，否则则丢弃此元素。
在网络中统计查询前k的关键字应用最广。结合hash的处理，简直就是利器。

堆插入的时间复杂度为o(lgN).也就是堆的层数。
最小堆最大的优势在于取最小值性能是O(1),如果插入的数据基本有序，那么最小堆能获得比较好的性能

下面是百度百科上的堆的样例代码，利用数组实现的。（我想了下，确实比链表实现的快多了，比如取最后一个元素的值，数组o(1)就可以实现，而链表要便利到最后的节点。）
``` C++ minheap.cpp
#include <iostream>

using namespace std;

template<typename T>
class MinHeap
{
private:
	T *heap;
	int CurrentSize;
	int MaxSize;
	
	void FilterDown(int start, const int end);

	void FilterUp(int start);
public:
	MinHeap(int n=1000);
	
	~MinHeap();
	bool insert(const T &);
	
	T removeMin();
	
	T getMin();
	
	bool isEmpty() const;

	bool isFull() const;

	void print();

	void clear();
};

template<typename T>
MinHeap<T>::MinHeap(int n)
{
	MaxSize = n;
	heap = new T[MaxSize];
	CurrentSize = 0;
}

template <typename T>
MinHeap<T>::~MinHeap()
{
	delete [] heap;
	if(heap != NULL)
		heap = NULL;
}

template <typename T>
void MinHeap<T>::FilterUp(int start) //从下向上调整
{
	int parent = (start-1) / 2; // 指向j的父节点
	T temp = heap[start];

	while (start > 0)
	{
		if(heap[parent] <= temp) //保证父节点比它大就行
			break;
		else
		{
			heap[start] = heap[parent];
			start = parent;
			parent = (parent-1) / 2;
		}
	}
	heap[start] = temp;
}

/*
template<typename T>
void MinHeap<T>::FilterUp(int start)
{
	int parent = (start - 1) / 2;
	if(heap[parent] <= heap[start])
		return;
	else
	{
		swap(&heap[start],&heap[parent]);
		parent = (parent - 1) / 2;
		if(parent)
			FilterUp(parent);
	}
}
*/

template <typename T>
void MinHeap<T>::FilterDown(int start, const int end) //从上往下调整
{
	int left_son = 2 * start + 1;
	T temp = heap[start];

	while(left_son <= end)
	{
		if(left_son <end  && heap[left_son] > heap[left_son+1])  //选择两个儿子节点中数值较大的。
			left_son++;
		if(temp <= heap[left_son])
			break;
		else
		{
			heap[start] = heap[left_son];
			start = left_son;
			left_son = 2 *left_son + 1;
		}
	}
	heap[start] = temp;
}

template<typename T>
bool MinHeap<T>::insert(const T &x)
{
	if(CurrentSize == MaxSize)
		return false;
	heap[CurrentSize] = x;
	FilterUp(CurrentSize);
	CurrentSize++;
	return true;
}

template<typename T>
T MinHeap<T>::removeMin()
{
	T x = heap[0];
	heap[0] = heap[CurrentSize -1];    //换位最后一位元素的值，再调整堆
	CurrentSize--;
	FilterDown(0, CurrentSize);
	return x;
}

template<typename T>
T MinHeap<T>::getMin()
{
	return heap[0];
}

template<typename T>
bool MinHeap<T>::isEmpty() const
{
	return CurrentSize == 0;
}

template<typename T>
bool MinHeap<T>::isFull() const
{
	return CurrentSize == MaxSize;
}

template<typename T>
void MinHeap<T>::clear()
{
	CurrentSize = 0;
}

template<typename T>
void MinHeap<T>::print()
{
	for(int i=0; i<CurrentSize; ++i)
		cout << heap[i] << "  ";
	cout <<endl;
}

int main()
{
	int k, n = 11;
	int a[11] = {0, 5, 2, 4, 9, 7, 3, 1, 10, 8, 6};
	MinHeap<int> test(11);
	for(k=0; k<n; k++)
		test.insert(a[k]);
	test.print();
	cout << test.isFull() << endl;
	for(k=0; k<n; k++)
		cout << test.removeMin()<<endl;
	test.print();
	int i;
	cin >> i;
	return 0;
}
```

**参考：**

- [http://blog.sina.com.cn/s/blog_56e6a0750101b0fo.html](http://blog.sina.com.cn/s/blog_56e6a0750101b0fo.html)
- [http://baike.baidu.com/view/1251056.htm](http://baike.baidu.com/view/1251056.htm)

有时间再系统的学习下各种树的复杂度和各个适用的场景吧。不过我知道linux下对page页表的管理是采用的红黑树。



