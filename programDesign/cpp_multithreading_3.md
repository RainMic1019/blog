---
title: C++并发与多线程笔记三：数据共享
date: 2022-04-02
categories:
 - 程序设计
tags:
 - C++
 - 并发
 - 多线程
---

## 1 前言

本文接上文 [C++并发与多线程笔记二](./cpp_multithreading_2.md) 的内容，主要包含创建多个线程、数据共享问题分析和案例代码。

## 2 创建和等待多个线程

这里创建十个线程，并且使用同一个 入口函数 `my_thread()`，代码如下：

```cpp
#include <iostream>
#include <thread>
#include <vector>
using namespace std;

void my_thread(int num)
{
	cout << "my thread start, num = " << num << endl;
	/* 线程代码 */
	cout << "my thread end, num = " << num << endl;
	return;
}

int main()
{
	vector<thread> m_threads;
	/* 创建10个线程，线程回调统一用 my_thread() */
	for (int i = 0; i < 10; i++)
	{
		m_threads.push_back(thread(my_thread, i)); /* 创建线程并开始执行 */
	}
	for (auto iter = m_threads.begin(); iter != m_threads.end(); ++iter)
	{
		iter->join(); /* 等待线程结束 */
	}

	cout << "Hello World!" << endl;
	return 0;
}
```

输出结果：

```bash
my thread start, num = 0my thread start, num = 2
my thread start, num = my thread start, num = 6my thread start, num = my thread start, num = 9
my thread end, num = 9

my thread end, num = 6
my thread start, num = 1
my thread end, num = 1
8
my thread end, num = 8
my thread start, num = 7
my thread end, num = 7

my thread end, num = 0
my thread start, num = 4
my thread end, num = 4
my thread start, num = 3
my thread end, num = 3
5
my thread end, num = 5
my thread end, num = 2
Hello World!
```

1. 可以看到线程并不是按照创建顺序执行的，先创建的线程不一定先执行，这个跟操作系统内部对线程的运行调度机制有关。
2. 主线程等待所有子线程运行结束，最后主线程结束，使用 `join()` 更容易写出稳定的程序。
3. 把thread对象放入到 vector 容器中（类似对象数组），方便管理。

## 3 数据共享问题分析

### 3.1 只读数据

如果一份数据是只读的，提供给多个线程使用，每个线程读到的数据都是一样的，这份数据仍然是**安全稳定的**，比如以下代码中的 `g_value`：

```cpp
#include <iostream>
#include <thread>
#include <vector>
using namespace std;

vector<int> g_value = {1, 2, 3}; /* 共享数据(只读) */

void my_thread(int num)
{
	cout << "thread id = " << this_thread::get_id();
	cout << "\t g_v: " << g_value[0] << g_value[1] << g_value[2] << endl;
	return;
}

int main()
{
	vector<thread> m_threads;
	/* 创建10个线程，线程回调统一用 my_thread() */
	for (int i = 0; i < 10; i++)
	{
		m_threads.push_back(thread(my_thread, i)); /* 创建线程并开始执行 */
	}
	for (auto iter = m_threads.begin(); iter != m_threads.end(); ++iter)
	{
		iter->join(); /* 等待线程结束 */
	}

	cout << "Hello World!" << endl;
	return 0;
}
```

输出结果：

```bash
thread id = thread id = 12104    g_v: 1thread id = 9116  g_v: 123thread id = 4540        g_v: 123
thread id = 5108         g_v: 123
thread id = 2616         g_v: 123
thread id = 18056        g_v: 123
thread id = 2420         g_v: 123

thread id = 7440         g_v: 123
23
thread id = 4484         g_v: 123
4764     g_v: 123
Hello World!
```

### 3.2 读写数据

比如两个线程写`g_value`，八个线程读`g_value`，如果代码没有特别处理，那程序大概率会崩溃，或者得到错误的结果，发生各种不可预料的结果。

最简单的处理就是读的时候不能写，写的时候不能读，多个线程之间不能同时写，也就说我们经常说的线程同步问题。

### 3.3 其他案例

假设目前在售卖从北京到深圳的火车票，车次为T123，共有10个售票窗口，其中1号窗口和2号窗口的顾客同时都要订99号座位，那这个时候在程序中的处理逻辑应该如下：

1. 假设先处理1号窗口的订单，首先需要查看这个99号座位是否为空，这里是读数据；
2. 如果为空，则帮顾客订购99号座位，并录入系统，这里是写数据；
3. 2号窗口在1号窗口的操作的过程中必须等着，只有等1号窗口操作完成，2号窗口才能开始操作。

## 4 共享数据案例代码

假设做一个网络游戏服务器，有两个子线程：

- 子线程1收集玩家命令，并把命令数据写到一个队列中。
- 子线程2从队列中取出玩家发出来的命令并解析，然后执行玩家需要的动作。

此处为了简化实现，用一个数字代表玩家发出来的命令，并且基于面向对象的思想，使用成员函数作为线程回调函数。

```cpp
#include <iostream>
#include <thread>
#include <list>
using namespace std;

class A
{
public:
	/* 把收到的消息（玩家命令）存到队列中 */
	void inMsgRecvQueue()
	{
		for (int i = 0; i < 100000; ++i)
		{
			cout << "inMsgRecvQueue exec, push an elem " << i << endl;
			msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
		}
	}
	/* 把数据从消息队列中取出 */
	void outMsgRecvQueue()
	{
		for (int i = 0; i < 100000; ++i)
		{
			if (!msgRecvQueue.empty())
			{
				int command = msgRecvQueue.front(); /* 返回第一个元素 */
				msgRecvQueue.pop_front();			/* 移除第一个元素 */
			}
			else
			{
				/* 消息队列为空 */
				cout << "outMsgRecvQueue exec, but queue is empty!" << i << endl;
			}
			cout << "outMsgRecvQueue exec end!" << i << endl;
		}
	}

private:
	list<int> msgRecvQueue; /* 容器（实际上是双向链表）：存放玩家发生命令的队列 */
};

int main()
{
	A obj;
	thread myInMsgObj(&A::inMsgRecvQueue, &obj);
	thread myOutMsgObj(&A::outMsgRecvQueue, &obj);
	myInMsgObj.join();
	myOutMsgObj.join();

	cout << "Hello World!" << endl;
	return 0;
}
```

这里的消息队列 msgRecvQueue 就是共享数据，以上程序运行时会出现崩溃或数据乱套的现象，因为两个线程的回调函数 `inMsgRecvQueue()` 和 `outMsgRecvQueue()` 分别在不断读写消息队列 `msgRecvQueue`，由于不做任何限制，可能会出现 in 线程还没 push 完一个数据时，out 线程就已经把消息队列中的首元素删除的情况，这时 in 线程 push 的数据就不知道飞到哪里去了。

**这里抛出问题，下篇我们通过互斥量来解决这个问题。**
