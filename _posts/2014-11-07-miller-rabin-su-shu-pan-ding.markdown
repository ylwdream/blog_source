---
layout: post
title: "Miller–Rabin 素数判定"
date: 2014-11-07 23:46:08 +0800
comments: true
categories: 
- algorithm 
---

## Miller–Rabin
Miller–Rabin是MonteCarlo算法的一种，也就是概率算法，但是在实际应用中因为效率比较高，而出错概率跟小，所以应用很广。
<!--more-->


在实际应用中因为其复杂度$o(lgN^3)$远小于$o(sqrt(10^N))$,其中N为要排定的素数的位数。
## 费马定理
介绍这个算法，首先要介绍费马小定理。
根据费马小定理：如果p是素数，$ 1 \le a \le p $，那么

>* $$ a^{p-1} \equiv 1 \pmod{p} $$

上述命题的逆命题不是正确的，即满足$ a^{p-1} \equiv 1 \pmod{p} $不一定是素数，但是不满足肯定不是素数。。。

首先就是找出来强伪素数证据。

要测试 N 是否为素数，首先将 N-1 分解为 $ 2^s*d $。在每次测试开始时，先随机选一个介于 [1, N-1]的整数 a，之后如果对所有的$ r \in [0, s-1] $，若 $ a^d \mod N \neq 1 $ 且 $ a^{2^{r}d} \mod N \neq -1 $，则 N 是合数。

## 伪代码
``` 
Miller-Rabin(n,t)
　　输入：一个大于3的奇整数n和一个大于等于1的安全参 数t(用于确定测试轮数)。 
　　输出：返回n是否是素数(概率意义上的，一般误判概率小于(1/2)80即可) 。 
　　1、将n-1表示成2sr，(其 中 r是奇数)
　　2、 对i从1到 循t 环作下面的操作： 
　　2.1选择一个随机整数a(2≤a ≤n-2)
　　2.2计算y ←ar mod n
　　2.3如果y≠1并且y ≠n-1作下面的操作,否则转3： 
　　2.3.1 j←1;
　　2.3.2 当j≤s-1 并且y≠n-1循环作下面操作,否则跳到 2.3.3：
　　{计算y ←y2 mod n;
　　如果 y=1返回 &quot;合数 &quot;；
　　否则 j←j+1; }
　　2.3.3如果y ≠n-1 则返回&quot; 合数&quot; ； 
　　3、返回&quot;素数&quot;。 说明：本算法2.3.2循环中的&quot;y=1返回&quot;合数&quot; &quot;是基于如下定理：
　　定理： 设x、y和n是整数，如果x2=y2 (mod n) 但x ≠±y
　　(mod n)，则(x-y)和n的公约数中有n的非平凡因子。 在算法2.3.2循环中，如果y=1则a2(j-1)r =1(mod n)，而由此也可知a2(j-1)r≠±1(mod n) ，由此通过上面的定理可以知道，(a2(j-1)r －1)和n有非平凡公因子，从而可判断n是合数。

```
## 实验结果
下面是代码，和朴素的素数排定算法的误差为0，输出的素数是一样的。

``` c++ Miller-Rabin
#include <cmath>
#include <ctime>
#include <random>
#include <memory>
#include <iostream>

using namespace std;

template <typename T>
class Prime
{
public:
	Prime()
	{
		gen = new std::mt19937(rd());
	}
	T Mod(T& a, T& t, T& n)     //快速幂模运算
	{
		T ret = 1;
		T tmp = a;
		while (t)
		{
			if (t & 1) ret = ret * tmp % n;
			tmp = (tmp * tmp) % n;
			t >>= 1;
		}
		return ret;
	}

	bool Btest(T &a, T &n) 
	{
		T s = 0;
		T t = n - 1;
		do{
			s++;
			t = t >> 1;
		} while (t % 2 == 0); 把n-1分解为2^s * d
		T x = Mod(a, t, n);
		if (x == 1 || x == n - 1)
			return true;

		for (T i = 1; i <= s - 1; ++i)
		{
 			x = (x * x) % n;
			if (x == n - 1)
				return true;
		}
		return false;
	}

	T uniform(T begin, T end)
	{
		T temp = gen->operator()();
		return temp % (end - begin) + begin;
	}

	bool MillRab(T &n)
	{
		T a = uniform(2, n - 2);  //从2~n-2中随机选一个数
		return Btest(a, n);
	}

	bool RepeatMillRob(T n, T k)  //重复检测k次MillRab,因为算法是偏假的，所以有一次假酒可以排定n非素
	{
		for (T i = 1; i <= k; ++i)
		if (MillRab(n) == false)
			return false;
		return true;
	}

private:
	std::random_device rd;
	std::mt19937 *gen;
};

int main()
{

	long long i;
	Prime<long long> prime;
	int sum = 0;
	for (i = 2; i < 1000; ++i)
	{
		
		if (i & 1 && prime.RepeatMillRob(i, 3))      //首先排定n是否为奇数，重复运行三次。 
		{
			cout << i << ' ';
			sum++;
			if (sum % 10 == 0) cout << endl;
		}
			
//		else
//			cout << "the number" << i << "is not prime" << endl;
	}
	cout << endl;
	cout << "the total num is" << sum << endl;
	return 0;
}
```


## **参考**

- [Miller–Rabin](http://en.wikipedia.org/wiki/Miller%E2%80%93Rabin_primality_test)
- [费马小定理](http://zh.wikipedia.org/wiki/%E8%B4%B9%E9%A9%AC%E7%B4%A0%E6%80%A7%E6%A3%80%E9%AA%8C)
