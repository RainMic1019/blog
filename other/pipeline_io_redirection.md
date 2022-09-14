---
title: 管道通信：输入输出重定向
date: 2021-05-05
categories:
 - 其他
tags:
 - C
 - C++
 - 通信
 - Windows
---

## 1 前言

近期工作内容需要研究进程间的通信，项目拟使用管道实现进程通信，因此将学习的内容简单整理一下，希望对自己与各位有所帮助。

## 2 示例介绍

本文简单介绍输入输出的重定向问题，首先看一个示例，假如有一个程序的输入输出作为标准输入输出，即从键盘上输入，然后输出（打印）到屏幕，程序的示例代码如下：

```c
#include <stdio.h>
int main()
{
	int n = 0;
	/* 标准输入时，按 ctrl + z 可输入EOF */
	while (scanf("%d", &n) != EOF) {
		printf("%d\n", n);
	}
}
```

现在要重定向输入法输出，使程序从某一文件中读取数据，处理后输出到另一个文件，有两种实现方式：

1. 使用C语言的freopen()函数；
2. 使用C++的ifstream和ofstream类。

## 3 实现输入输出重定向

### 3.1 使用freopen()函数

#### 3.1.1 函数原型

```c
FILE *freopen(
   const char *path,  // 文件路径
   const char *mode,  // 打开方式，"w"表示写，"r"表示读，"a"表示追加，详情请查阅MSDN
   FILE *stream       // FILE类型的指针，传入stdin表示标准输入，传入stdout表示标准输出
);
```

#### 3.1.2 使用方法

freopen()函数的使用方法很简单，比如 freopen("stdin.txt", "r", stdin); 就可以使程序中的 scanf() 函数从 stdin.txt 文件中读取数据作为输入，同样 freopen("stdout.txt", "w", stdout); 可以使程序中的 printf() 函数将输出由标准输出改为输出到 stdout.txt 文件。

> 将程序的输入输出重定向到文件后，如果想改回到标准输入输出，操作如下：Windows平台第一个参数 path 传入 "CON"，Linux为 "/dev/console"。

#### 3.1.3 示例代码

```c
#include <stdio.h>

int main()
{
	int n = 0;
	FILE *pFileRead = freopen("stdin.txt", "r", stdin);
	FILE *pFileWrite = freopen("stdout.txt", "w", stdout);
 
	while (scanf("%d", &n) != EOF) {
		printf("%d\n", n);
	}
	fclose(pFileRead);
	fclose(pFileWrite);
 
	// 改回控制台的标准输入输出（Windows为"CON" Linux为"/dev/console"）
	freopen("CON", "r", stdin);
	freopen("CON", "w", stdout); 
	while (scanf("%d", &n) != EOF) {
		printf("%d\n", n);
	}
	return 0;
}
```

### 3.2 使用i/ofstream类

关于ifstream类和ofstream类的详细信息请查阅MSDN，下面直接介绍如果使用这两个类实现输入输出重定向。

#### 3.2.1 使用方法

ifstream类和ofstream类可以以读的方式和写的方法打开一个文件，在 iosfwd 文件中可见以下代码：

```cpp
typedef basic_ifstream<char, char_traits<char> > ifstream;
typedef basic_ofstream<char, char_traits<char> > ofstream;
```

并且可以在 fstream 文件中找到以下代码，表明 basic_ifstream 类继承于 basic_istream 类，basic_ofstream 类继承于basic_ostream类：

```cpp
...
	class basic_ifstream
		: public basic_istream<_Elem, _Traits>
...
	class basic_ofstream
		: public basic_ostream<_Elem, _Traits>
...
```

最后，我们再来看看常用的 iostream 文件，里面有 cin 和 cout 的定义，二者实际上是 istream 类型和 ostream 类型的变量，代码如下：

```cpp
__PURE_APPDOMAIN_GLOBAL extern istream cin, *_Ptr_cin;
__PURE_APPDOMAIN_GLOBAL extern ostream cout, *_Ptr_cout;
```

根据以上代码可以判断 cin 和 cout 与 ifstream 类和 ofstream 类有着非常密切关系：

- cin 是 basic_istream 类的变量，而 ifstream 则是 basic_istream 类的派生类。
- cout 是 basic_ostream 类的变量，而 ofstream 则是 basic_ostream 类的派生类。

事实上在 basic_istream 类实际是虚继承于 basic_ioso 类，basic_ostream 类实际是虚继承于 basic_ios 类。

basic_ioso 类 和 basic_ios 类都有 rdbuf() 成员函数，这个函数允许我们访问和修改类中一个类型为 basic_streambuf 的成员变量。修改该成员就能重定向输入输出。

因此对 cin 和 cout 调用 rdbuf() 函数并传入 ifstream 和 ofstream 的 rdbuf() 就可以将控制台的标准输入输出改为从文件中读取和输出到文件。

#### 3.2.2 示例代码

```cpp
#include <iostream>
#include <fstream>
using namespace std;

int main()
{
	ifstream stdin("stdin.txt");
	ofstream stdout("stdout.txt");
	// 保存标准输入输出方式 streambuf 类就是 basic_streambuf 类
	streambuf *strmin_buf = cin.rdbuf();
	streambuf *strmout_buf = cout.rdbuf();
 
	// 重定向到文件
	cin.rdbuf(stdin.rdbuf());
	cout.rdbuf(stdout.rdbuf());
 
	int n = 0;
	while (cin >> n) {
		cout << n << endl;
		if (n == 0)
		  break;
	}
 
	inFile.close();
	outFile.close();
 
	// 改回控制台的标准输入输出
	cin.rdbuf(strmin_buf);
	cout.rdbuf(strmout_buf);
 
	while (cin >> n) {
		cout << n << endl;
		if (n == 0)
		  break;
	}
 
	return 0;
}
```
