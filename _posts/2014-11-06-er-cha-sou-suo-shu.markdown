---
layout: post
title: "二叉搜索树"
date: 2014-11-06 22:01:16 +0800
comments: true
toc: false
categories: algorithm
tags:
- tree
---


今天这货搞得我很急躁啊，妈蛋，本来以为一会就可以把代码敲完的，但是设计数据结构的时候没有考虑递归，第一遍写了一点，发现没有父指针好麻烦，就修改，修改完还是觉得代码好麻烦，结果百度了下，看到递归这么简单，原来这个结构是天然的递归萌啊。。。。
<!--more-->

今天在书上看到二叉搜索树和AVL树，所以想整理一下。

二叉树的定义：

根节点的值大于左节点小于右节点。
这个定义是递归的。

二叉树的优点就是插入和查找的复杂度是o(logN)的。删除会稍微麻烦一点。

排序好的二叉树中序遍历是从大到小有序的。

下面是源代码，就不做图了，因为参考里面已经将的十分详细了，我就不在浪费时间了。

``` C++ Btree.cpp

#include <iostream>
using namespace std;


template <class T>
struct node
{
	struct node<T> *left;
	struct node<T> *right;

	T value;

	node(T value)
	{
		left = NULL;
		right = NULL;
		this->value = value;
	}

	node()
	{
		left = NULL;
		right = NULL;
		this->value = 0;
	}
	
};

template <class T>
class Btree
{

private:
	node<T> * root;
	bool insert_node(node<T>* &root, T value);
	node<T>* search_node(node<T>* node, T value);
	void printLDR(node<T>* node);
	void delete_node(node<T>*, T value);
	int length(node<T> *);
public:
	
	Btree() :root(NULL){}
	~Btree(){}
	bool insert(T);
	node<T>* search(T value);
	void erase(T value);
	int length();
	void clear();
	void print_LDR();     //中序遍历
};


template <class T>
bool Btree<T>::insert_node(node<T>* &root, T value)
{
	if (root == NULL)
	{
		root = new node<T>(value);
		return true;
	}

	if (root->value > value)
		insert_node(root->left, value);
	else if (root->value < value)
		insert_node(root->right, value);
	return false;
}

template<class T>
bool Btree<T>::insert(T value)
{
	return insert_node(root, value);
}

template <class T>
node<T>* Btree<T>::search_node(node<T>* node, T value)
{
	if (node == NULL)
		return NULL;
	if (node->value > x)
	{
		return search_node(node->left, value);
	}
	else if (node->value < x)
	{
		return search_node(node->right, value);
	}
	else return node;
}

template<class T>
node<T>* Btree<T>::search(T value)
{
	return search_node(root, value);
}

template<class T>
void Btree<T>::delete_node(node<T>* root, T value)
{
	if (root == NULL)
		return;
	if (root->value > value)
		delete_node(root->left, value);
	else if (root->value < value)
		delete_node(root->right, value);
	else
	{
		if (root->left && root->right)
		{
			node<T>* tmp = root->right;
			while (tmp->left != NULL)
				tmp = tmp->left;
			root->value = tmp->value;
			delete_node(root->right, tmp->value);
		}
		else
		{
			node<T>* tmp = root;
			if (root->left == NULL)
				root = root->right;
			else if (root->right == NULL)
				root = root->left;
			delete tmp;
		}
	}
	return;
}
template<class T>
void Btree<T>::erase(T value)
{
	delete_node(root, value);
}

template<class T>
void Btree<T>::printLDR(node<T>* node)
{
	if (node == NULL)
		return;
	printLDR(node->left);
	cout << node->value << " ";
	printLDR(node->right);
}

template<class T>
void Btree<T>::print_LDR()
{
	printLDR(root);
}

template<class T>
int Btree<T>::length(node<T> * root)
{
	if (root == NULL)
		return 0;
	int len_left = length(root->left);
	int len_right = length(root->right);
	return (len_left > len_right ? len_left : len_right) + 1;
}

template<class T>
int Btree<T>::length()
{
	return length(root);
}

```
快速排序的过程其实就是二叉查找树，最坏的情况下，二叉树会是线性的。所以复杂度就会是o(N*N),如果是平衡的，那么查找的一个节点的复杂度就是$o(\log N)$，查找的同时可以完成插入,N个节点排序的复杂度就是$o(N \ast logN)$。


**下面讨论下AVL树，也就是平衡的二叉搜索树。**

在AVL树中任何节点的两个子树的高度最大差别为一，所以它也被称为高度平衡树。查找、插入和删除在平均和最坏情况下都是$O（\log n）$。

AVL树得名于它的发明者G.M. Adelson-Velsky和E.M. Landis。

在AVL树中任何节点的两个子树的高度最大差别为一，所以它也被称为高度平衡树。节点的平衡因子是它的左子树的高度减去它的右子树的高度（有时相反）。带有平衡因子1、0或 -1的节点被认为是平衡的。带有平衡因子 -2或2的节点被认为是不平衡的，并需要重新平衡这个树。平衡因子可以直接存储在每个节点中，或从可能存储在节点中的子树高度计算出来。


