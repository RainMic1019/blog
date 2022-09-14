---
title: C++并发与多线程笔记四：互斥量(概念、用法、死锁)
date: 2022-04-05
categories:
 - 程序设计
tags:
 - C++
 - 并发
 - 多线程
---

## 1 前言

本文接上文 [C++并发与多线程笔记三：数据共享](./cpp_multithreading_3.md) 的内容，主要包含互斥量的基本概念、用法、死锁演示以及解决方案。

## 2 互斥量的基本概念

互斥量就是个类对象，可以理解成一把锁，多个线程尝试用lock()成员函数来加锁，只有一个线程能锁定成功，如果没有锁成功，那么流程将卡在lock()这里不断尝试去锁定。

> 备注：互斥量使用要小心，上锁的代码需要根据实际情况考虑（只保护需要保护的数据），少了达不到效果，多了影响效率。

## 3 互斥量的用法

首先，需要包含头文件`#include <mutex>`，然后使用 mutex 类即可创建锁对象：

```cpp
#include <iostream>
#include <mutex>
using namespace std;

class A
{
private:
	mutex my_mutex;			/* 创建一个互斥锁 */
};
```

### 3.1 lock()和unlock()

在代码中 `lock()` （上锁）和 `unlock()` （解锁）必须**成对使用**，步骤如下：

* 先 `lock()` 上锁；
* 然后操作共享数据；
* 再 `unlock()` 解锁

> 备注：代码中使用互斥量的时绝不允许非对称调用，即 `lock()` 和 `unlock()` 一定是成对出现的。

以上篇文章的示例代码为例，我们需要保护的共享数据为消息队列 `msgRecvQueue` ，在读写这个队列前就需要上锁，读写完毕后需要解锁，代码如下：

```cpp
#include <iostream>
#include <thread>
#include <mutex>
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
			my_mutex.lock();		   /* 上锁 */
			msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
			my_mutex.unlock();		   /* 解锁 */
		}
	}
	/* 消息队列不为空时，返回并弹出第一个元素 */
	bool outMsgLULProc(int &command)
	{
		my_mutex.lock(); /* 上锁 */
		if (!msgRecvQueue.empty())
		{
			command = msgRecvQueue.front(); /* 返回第一个元素 */
			msgRecvQueue.pop_front();			/* 移除第一个元素 */
			my_mutex.unlock();					/* 解锁（每个分支都需要解锁，别漏了） */
			return true;
		}
		my_mutex.unlock(); /* 解锁（每个分支都需要解锁，别漏了） */
		return false;
	}
	/* 把数据从消息队列中取出 */
	void outMsgRecvQueue()
	{
		int command = 0;
		for (int i = 0; i < 100000; ++i)
		{
			bool result = outMsgLULProc(command);
			if (result)
				cout << "outMsgLULProc exec, and pop_front: " << command << endl;
			else
				cout << "outMsgRecvQueue exec, but queue is empty!" << i << endl;
			cout << "outMsgRecvQueue exec end!" << i << endl;
		}
	}

private:
	list<int> msgRecvQueue; /* 容器（实际上是双向链表）：存放玩家发生命令的队列 */
	mutex my_mutex;			/* 创建一个互斥锁 */
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

### 3.2 std::lock_guard类模板

我们在代码中上锁后，一定要记得解锁，如果忘记解锁会导致程序运行异常，而且通常很难排查。为了防止开发者忘记解锁，C++11引入了一个叫做 `std::lock_guard` 的类模板，它在开发者忘记解锁的时候，会替开发者自动解锁。

> 备注：`std::lock_guard` 可以直接取代 `lock()` 和 `unlock()`，也就说使用 `std::lock_guard` 后，就不能再使用 `lock()` 和 `unlock()` 了。

```cpp
#include <iostream>
#include <thread>
#include <mutex>
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
			lock_guard<mutex> m_guard(my_mutex);
			msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
		}
	}
	/* 消息队列不为空时，返回并弹出第一个元素 */
	bool outMsgLULProc(int &command)
	{
		/**
		 * m_guard 是一个 lock_guard 对象。
		 * lock_guard 构造函数里执行了 lock()。
		 * lock_guard 析构函数里执行了 unlock()。
		 */
		lock_guard<mutex> m_guard(my_mutex);
		if (!msgRecvQueue.empty())
		{
			command = msgRecvQueue.front(); /* 返回第一个元素 */
			msgRecvQueue.pop_front();			/* 移除第一个元素 */
			return true;
		}
		return false;
	}
	/* 把数据从消息队列中取出 */
	void outMsgRecvQueue()
	{
		int command = 0;
		for (int i = 0; i < 100000; ++i)
		{
			bool result = outMsgLULProc(command);
			if (result)
				cout << "outMsgLULProc exec, and pop_front: " << command << endl;
			else
				cout << "outMsgRecvQueue exec, but queue is empty!" << i << endl;
			cout << "outMsgRecvQueue exec end!" << i << endl;
		}
	}

private:
	list<int> msgRecvQueue; /* 容器（实际上是双向链表）：存放玩家发生命令的队列 */
	mutex my_mutex;			/* 创建一个互斥锁 */
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

`std::lock_guard` 虽然用起来方便，但是不够灵活，它只能在析构函数中 `unlock()`，也就是对象被释放的时候，这通常是在函数返回的时候，或者通过添加代码块 `{ /* 代码块 */ }` 限定作用域来指定释放时机。

## 4 死锁

一个简单的例子：

- 张三在北京说：等李四来了之后，我就去广东。
- 李四在广东说：等张三来了之后，我就去北京。

这两个人一直等待对方就形成了死锁。

同理，假如在代码中有两把锁（至少有两个互斥量存在才会产生死锁）分别称为锁1、锁2，并且有两个线程分别称为线程A和线程B。只有在某个线程同时获得锁1和锁2时，才能完成某项工作：

- 线程A执行时，先上锁1，再上锁2。
- 线程B执行时，先上锁2，再上锁1。

若在程序执行线程A的过程中，上好了锁1后，出现了上下文切换，系统调度转去执行线程B，把锁2给上了，那么后续线程A拿不到锁2，线程B拿不到锁1，两条线程都没法往下执行，即出现了死锁。

### 4.1 死锁演示

```cpp
#include <iostream>
#include <thread>
#include <mutex>
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
			m_mutex1.lock(); /* 实际代码中，两把锁不一定同时上，它们可能保护不同的数据 */
			m_mutex2.lock();
			msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
			m_mutex2.unlock();
			m_mutex1.unlock();
		}
	}
	/* 消息队列不为空时，返回并弹出第一个元素 */
	bool outMsgLULProc(int &command)
	{
		m_mutex2.lock();
		m_mutex1.lock();
		if (!msgRecvQueue.empty())
		{
			command = msgRecvQueue.front(); /* 返回第一个元素 */
			msgRecvQueue.pop_front();			/* 移除第一个元素 */
			m_mutex1.unlock();
			m_mutex2.unlock();
			return true;
		}
		m_mutex1.unlock();
		m_mutex2.unlock();
		return false;
	}
	/* 把数据从消息队列中取出 */
	void outMsgRecvQueue()
	{
		int command = 0;
		for (int i = 0; i < 100000; ++i)
		{
			bool result = outMsgLULProc(command);
			if (result)
				cout << "outMsgLULProc exec, and pop_front: " << command << endl;
			else
				cout << "outMsgRecvQueue exec, but queue is empty!" << i << endl;
			cout << "outMsgRecvQueue exec end!" << i << endl;
		}
	}

private:
	list<int> msgRecvQueue; /* 容器（实际上是双向链表）：存放玩家发生命令的队列 */
	mutex m_mutex1;			/* 创建互斥量1 */
	mutex m_mutex2;			/* 创建互斥量2 */
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

### 4.2 死锁的一般解决方案

通常来讲，只要保证多个互斥量上锁的顺序一致，就不会出现死锁，比如把上面示例代码的两个线程回调函数中的上锁顺序改一下，保持一致就好了（都改为先上锁1，再上锁2）。

### 4.3 std::lock()函数模板

`std::lock()` 函数模板是C++11引入的，它能一次锁住两个或两个以上的互斥量，并且它不存在上述的在多线程中由于上锁顺序问题造成的死锁现象，原因如下：

`std::lock()` 函数模板在锁定两个互斥量时，只有两种情况：

1. 两个互斥量都没有锁住；
2. 两个互斥量都被锁住。

如果只锁了一个，另一个没锁成功，则它会立即把已经锁住的互斥量解锁。

```cpp
	/* 把收到的消息（玩家命令）存到队列中 */
	void inMsgRecvQueue()
	{
		for (int i = 0; i < 100000; ++i)
		{
			cout << "inMsgRecvQueue exec, push an elem " << i << endl;
			std::lock(m_mutex1, m_mutex2);
			msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
			m_mutex2.unlock();		   /* 这里别忘记解锁 */
			m_mutex1.unlock();		   /* 这里别忘记解锁 */
		}
	}
```

### 4.4 std::lock_guard的std::adopt_lock参数

在使用 `std::lock()` 函数模板锁上多个互斥量时，也必须得记得把每个互斥量解锁，此时借助 `std::lock_guard` 的 `std::adopt_lock` 参数可以省略解锁的代码。

```cpp
	/* 把收到的消息（玩家命令）存到队列中 */
	void inMsgRecvQueue()
	{
		for (int i = 0; i < 100000; ++i)
		{
			cout << "inMsgRecvQueue exec, push an elem " << i << endl;
			std::lock(m_mutex1, m_mutex2);									 /* 锁上两个互斥量 */
			std::lock_guard<std::mutex> m_guard1(m_mutex1, std::adopt_lock); /* 构造时不上锁，但析构时解锁 */
			std::lock_guard<std::mutex> m_guard2(m_mutex2, std::adopt_lock); /* 构造时不上锁，但析构时解锁 */
			msgRecvQueue.push_back(i);										 /* 假设数字 i 就是收到的玩家命令 */
		}
	}
```
