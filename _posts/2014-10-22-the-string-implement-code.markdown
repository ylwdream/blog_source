---
layout: post
title: "The String Implement Code"
date: 2014-10-22 03:26:16 -0700
comments: true
toc: false
categories: larbin
tags:
- 爬虫
---
最近在学习开源爬虫larbin的代码，里面有我觉得比较好的代码，想要记录下也想和大家分享下。
<!--more-->
1. 扩大字符串容量的算法。先扩大一倍，然后不行就扩大到需要的大小。可以看看stl中vector的实现。
``` c
	void LarbinString::addBuffer (char *s, uint len) 
	{
		if(size <= pos + len){
			size *= 2;
    		if (size <= pos + len) size = pos + len + 1;
			char *tmp = new char[size];
			memcpy(tmp, chaine, pos);
			delete [] chaine;
			chaine = tmp;
		}
		memcpy(chaine+pos, s, len);
		pos += len;
		chaine[pos] = 0;
	}
```
在stl中vector的实现方法与此类似，底层是一个数组，第一次扩大也是原来的二倍。
   
2. 查找匹配特定的字符串可以带\*模式
``` c
    /* test if b is forbidden by pattern a */
	bool robotsMatch (char *a, char *b) 
	{
		int i=0;
		int j=0;
		while (a[i] != 0) 
		{
		    if (a[i] == '*') 
			{
	      		i++;
      			char *tmp = strchr(b+j, a[i]);
      			if (tmp == NULL) return false;
      			j = tmp - b;
    		} 
			else 
			{
      			if (a[i] != b[j]) return false;
      		i++; j++;
    		}
		}
	  return true;
	}
```
3. 从后向前找是否是字符串匹配，如果从前向后匹配的话会多浪费一个变量
``` c
//从后向前找
bool caseContain (char *a, char *b) 
{
	size_t la = strlen(a);
	int i = strlen(b) - la;
	while (i >= 0) 
	{
		if (!strncasecmp(a, b+i, la)) 
		{
  			return true;
		}
		i--;
		}
  return false;
}
//如果从前向后匹配的话
bool caseContain (char *a, char *b) 
{
	size_t la = strlen(a);
	size_t lb = strlen(b);
	int i = 0;
	while (i <= lb - la) {
	if (!strncasecmp(a, b+i, la)) 
	{
  		return true;
	}
	i++;
	}
	return false;
}
```

	
