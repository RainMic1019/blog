---
title: 斐波那契数列两种算法的时间复杂度
date: 2019-10-09
categories:
 - 算法
tags:
 - C++
---

## 1 简介

斐波那契数列，又称黄金分割数列，指的是这样一个数列：0、1、1、2、3、5、8、13、21、34、……在数学上，斐波纳契数列以如下被以递归的方法定义：F（0）=0，F（1）=1，F（n）=F(n-1)+F(n-2)（n≥2，n∈N*）在现代物理、准晶体结构、化学等领域，斐波纳契数列都有直接的应用，为此，美国数学会从1963起出版了以《斐波纳契数列季刊》为名的一份数学杂志，用于专门刊载这方面的研究成果。

## 2 题目要求

求解斐波那契数列的F(n)有两种常用算法：递归算法和非递归算法。试分析两种算法的时间复杂度。

## 3 算法分析

### 3.1 递归算法

实现代码如下：

```cpp
#include<iostream>
using namespace std;

long Fibonacci(int n) {
    if (n == 0)
        return 0;
    else if (n == 1)
        return 1;
    else
        return Fibonacci(n - 1) + Fibonacci(n-2);
}

int main() {
    cout << "Enter an integer number:" << endl;
    int N;
    cin >> N;
    cout << Fibonacci(N) << endl;
    system("pause");
    return 0;
}
```

**时间复杂度分析：**求解F(n),必须先计算F(n-1)和F(n-2),计算F(n-1)和F(n-2)，又必须先计算F(n-3)和F(n-4)。。。。。。以此类推，直至必须先计算F(1)和F(0),然后逆推得到F(n-1)和F(n-2)的结果，从而得到F(n)要计算很多重复的值，在时间上造成了很大的浪费，算法的时间复杂度随着N的增大呈现指数增长，时间的复杂度为O(2^n)，即2的n次方。

### 3.2 非递归算法

实现代码如下：

```cpp
#include<iostream>
using namespace std;

long Fibonacci(int n) {
    if (n <= 2)
        return 1;
    else {
        long num1 = 1;
        long num2 = 1;
        for (int i = 2;i < n - 1;i++) {
            num2 = num1 + num2;
            num1 = num2 - num1;
        }
        return num1 + num2;
    }
}

int main() {
    cout << "Enter an integer number:" << endl;
    int N;
    cin >> N;
    cout << Fibonacci(N) << endl;
    system("pause");
    return 0;
}
```

**时间复杂度分析**：从n(>2)开始计算，用F(n-1)和F(n-2)两个数相加求出结果，这样就避免了大量的重复计算，它的效率比递归算法快得多，算法的时间复杂度与n成正比，即算法的时间复杂度为O(n)。
