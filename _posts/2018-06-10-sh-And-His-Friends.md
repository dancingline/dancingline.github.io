---
layout: post
title: "xdoj1287 sh And His Friends 概率DP"
date: 2018-06-09
excerpt: 
tags: [ACM, DP]
feature: 
comments: false
---

[题目链接](http://http://acm.xidian.edu.cn/problem.php?id=1287)

## 题目描述

sh要给朋友们过生日，他在考虑大概要买多少个蛋糕。
假设一年有n天，sh朋友们的生日是均匀分布。
问sh最少要有多少朋友，使得存在某一天至少要买三个蛋糕的概率不小于50%？
（即至少三个朋友在同一天生日的概率不小于50%）

## 输入

若干组数据，每组数据仅一行n（1<=n<=1e4）.

## 输出

输出sh最少要有多少朋友，使得满足题目条件。

## 样例输入

```
365
```

## 样例输出

```
88
```



## 解题思路

求某一天至少三个人过生日的概率不小于50%，转化为反面，求最多一天最多两个人过生日的概率小于50%。

用f\[i][j]表示i个人、j天有两个人过生日的概率，则i和j满足$$j\leq \lfloor i/2 \rfloor,\ i-j\leq n $$，那么问题转化为求使得$$\sum _{j=0}^{\lfloor i/2 \rfloor}f[i][j] <0.5$$成立的最小的i，接下来看如何计算f\[i][j]，要达到这一状态，第i个人的生日有两种可能，可能自己单独一天过生日，也可能跟另一个人同一天过生日，

1. 第i个人单独过生日，则前一个状态是（i-1, j），此时有i-1-2\*j天只有一个人过生日，满足$$j\leq \lfloor (i-1)/2 \rfloor,i-1-j \leq n$$，第i个人在剩下的n-(i-1-2\*j+j)天里的某一天过生日，这种情况下$$f[i][j]=\frac {n-i+j+1}{n}f[i-1][j]$$
2. 第i个人跟另一个人同一天过生日，则前一个状态为（i-1, j-1），此时有i-1-2\*（j-1）天只有一个人过生日，满足$$j-1\leq \lfloor(i-1)/2 \rfloor, i-1-2*(j-1)+j-1 \leq n \Rightarrow j \leq \lfloor i/2 \rfloor, i-j \leq n$$，第i个人在有一个人过生日的i-1-2\*（j-1）天里的某一天过生日，这种情况下$$f[i][j]=\frac{i-2*j+1}{n}f[i-1][j-1]$$

综合两种情况，可以得到f\[i][j]的最终表达式


$$
\begin{equation} f[i][j]=\frac{n-i+j+1}{n}f[i-1][j]+\frac{i-2*j+1}{n}f[i-1][j-1] \end{equation}
$$


## 代码

```c++
#include <bits/stdc++.h>
using namespace std;
double f[1000][500];
int main ()
{
    int n, i;
    while (cin >> n)
    {
        memset (f, 0, sizeof (f));
        int i=1;
        f[0][0] = 1.0;
        while (1)
        {
            double sum=0.0;
          //虽然上面推了很多限制，这里没体现几个，
          //不满足限制的状态的概率都初始化成了0
            for (int j=0; j<=i/2; j++)
            {
                if (i-j>n) continue;
                f[i][j] = f[i-1][j]*(n-i+j+1)/n;
                if (j>0) f[i][j] += f[i-1][j-1]*(i-2*j+1)/n;
                sum += f[i][j];
            }
            if (sum <0.5)
            {
                cout << i << endl;
                break;
            }
            i++;
        }
    }
    return 0;
}
```

