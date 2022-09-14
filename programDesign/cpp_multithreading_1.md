---
title: C++并发与多线程笔记一：基本概念和用法
date: 2022-03-06
categories:
 - 程序设计
tags:
 - C++
 - 并发
 - 多线程
---

## 1 基本概念

### 1.1 并发、进程、线程

#### 1.1.1 并发

并发是指两个或者更多的任务（独立的活动）同时发生（进行）：**一个程序同时执行多个独立的任务**。

以往的计算机通常是单核cpu，某一个时刻只能执行一个任务，此时由操作系统调度实现并发，即每秒钟进行多次所谓的“任务切换”，**造成并发的假象**，这种切换（上下文切换）存在时间开销，比如操作系统需要保存切换时的各种状态、执行进度等信息，并且切换回来时好需要复原这些信息。

随着硬件的发展，出现了多处理器计算机，用于服务器和高新能计算领域，比如一块芯片上有个核心（cpu）：双核、4核、8核等等，它们能实现真正的并行执行多个任务（**硬件并发**）。

使用并发的目的：可以同时干多个事，提高性能。

#### 1.1.2 进程

在了解进程之前，首先需要知道什么叫程序，**程序是指令、数据及其组织形式的描述，而进程就是程序的实体**。

简单理解，一个可执行程序运行起来了，就叫创建了一个进程。进程就是运行起来的可执行程序。

#### 1.1.3 线程

每个进程，都有唯一的一个主线程。执行可执行程序产生一个进程后，这个主线程就随着这个进程默默启动起来了。

**线程**：是操作系统能够进行运算调度的最小单位。它被包含在进程之中，是进程中的实际运作单位。一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。

简单理解，线程就是用来执行代码的，可以理解为一条代码的执行通路。

```cpp
/* 主线程启动时执行main()函数 */
int main()
{
    /* 各种代码 */
    return 0;
}
/* 
 * 主线程执行完main()函数return后，表示整个进程允许完毕，
 * 此时主线程结束运行，整个进程也结束运行。
 */
```

主线程由系统创建，除了它之外，可以通过写代码来创建其他线程，其他线程走的是别的道路，甚至去不同的地方。

每创建一个新线程，就可以在同一时刻，多干一个不同的事（多走一条不同的代码执行路径）。

程序中同时运行多个线程时，即实现了并发，但线程并不是越多越好，每个线程，都需要一个独立的堆栈空间，线程之间的切换要保存很多中间状态，会耗费本该属于程序运行的时间。

### 1.2 并发的实现方法

实现并发的手段：

- 通过多个进程实现并发
- 在一个进程中，创建多个线程实现并发

#### 1.2.1 多进程并发

- 比如账号服务器一个进程，游戏服务器一个进程，二者之间存在通信；
- 服务器进程之间存在通信，比如同一电脑下的管道，文件，消息队列，共享内存等；不同电脑下的 socket 通信等。

#### 1.2.2 多线程并发

**线程**：感觉像是轻量级的进程。每个进程有自己独立的运行路径，但一个进程中的所有线程共享地址空间（共享内存），全局变量、全局内存、全局引用都可以在线程之间传递，所以多线程开销远远小于多进程，但这也会引入一个新问题：数据一致性问题。

多进程并发和多线程并发可以混合使用，但通常优先考虑多线程技术。

> 备注：使用多线程并发时，创建线程的数量最大不建议超过 200-300个，至于多少合适，需要根据实际项目情况进行调整，有时线程数量过多反而会导致效率降低。

#### 1.2.3 总结

和进程比，线程的优点如下：

1. 线程启动速度更快，更轻量级；
2. 系统资源开销更少，执行速度更快，比如共享内存这种通信方式比任何其他的通信方式都快。

缺点：使用有一定难度，要小心处理数据的一致性问题。

### 1.3 C++11新标准线程库

以往的多线程代码通常调用系统平台提供的接口实现，不能跨平台运行，比如在 Windows 平台创建线程使用 CreateThread() 接口，但 Linux 平台则使用 pthread_create() 接口。

当然，使用 POSIX thread（pthread）库也可以实现跨平台，但需要在不同的平台上进行配置，用起来也不是特别方便。

**从 C++ 11 标准开始，C++语言本身增加了对多线程的支持**，意味着增强了可移植性（跨平台），减少了开发人员的工作量。

## 2 C++线程基本用法

### 2.1 线程运行的开始和结束

- 程序运行起来，生成一个进程，该进程所属的主线程开始自动运行；当主线程从main()函数返回，则整个进程执行完毕。
- 主线程从main()开始执行，那么我们自己创建的线程，也需要从一个函数开始运行（初始函数），一旦这个函数运行完毕，线程也结束运行。
- 整个进程是否执行完毕的标志是**主线程是否执行完**，如果主线程执行完毕就代表整个进程执行完毕了，**此时如果其他子线程还没有执行完，也会被强行终止**。

准备工作：

1. 添加头文件 thread.h。
2. 添加 std 命名空间。
3. 定义一个线程入口函数。

```cpp
#include <iostream>
#include <thread>
using namespace std;

/* 线程入口函数 */
void myThreadEntry()
{
	cout << "My thread start!!!" << endl;
	/* 线程执行代码 */
	cout << "My thread end!!!" << endl;
}
```

#### 2.1.1 thread()

创建一个线程对象：

```cpp
int main()
{
	/* 创建一个thread对象， 并以myThreadEntry()作为线程入口函数 */
	/* 其中myThreadEntry是可执行对象（函数指针） */
    thread newThread(myThreadEntry);
    return 0;
}
```

> 备注：线程类（thread类）参数是一个可调用对象。一组可执行的语句称为可调用对象，C++中的可调用对象可以是函数、函数指针、lambda表达式、bind创建的对象或者重载了函数调用运算符的类对象。

#### 2.1.2 join()

等待线程执行执行完毕：

```cpp
int main()
{
    thread newThread(myThreadEntry);
    /* 阻塞主线程，等待子线程执行完毕 */
	/* 当myThreadEntry执行完毕，join()就执行完毕，主线程继续往下执行 */
	newThread.join();
	cout << "Hello World!" << endl;
    return 0;
}
```

输出结果：

```bash
My thread start!!!
My thread end!!!
Hello World!
```

#### 2.1.3 detach()

传统多线程程序主线程必须等待子线程执行完毕后，自己才能最终退出，但 C++11 中提供了 detach() 接口，它用于分离主线程和子线程，即主线程不必等待子线程运行结束，主线程是否退出不影响子线程的运行。

一旦 detach() 之后，与主线程关联的thread对象就会失去与主线的关联（变成孤儿），此时这个子线程就会驻留在后台运行，这个子线程就相当于被C++运行时库接管了，当这个子线程运行完成后，由运行时库负责清理该线程相关的资源（守护线程）。

```cpp
int main()
{
    thread newThread(myThreadEntry);
    /* 分离主线程与子线程 */
	/* 子线程驻留在后台运行，被C++运行时库接管 */
	newThread.detach();
	cout << "Hello World!" << endl;
	cout << "Hello World!" << endl;
	cout << "Hello World!" << endl;
	cout << "Hello World!" << endl;
	cout << "Hello World!" << endl;
	cout << "Hello World!" << endl;
	cout << "Hello World!" << endl;
	cout << "Hello World!" << endl;
	cout << "Hello World!" << endl;
	cout << "Hello World!" << endl;
    return 0;
}
```

输出结果：

```bash
Hello World!
Hello World!
Hello World!
Hello World!
Hello World!
Hello World!
Hello World!
My thread start!!!
Hello World!
My thread end!!!
Hello World!
Hello World!
```

> 备注：主线程和子线程分离后，输出结果交替打印，且每次运行的输出结果都不一样。

一旦调用 detach()，就不能对子线程使用 join() 了。detach() 使我们完全失去了对子线程的控制，因此不建议这样用，还是调用 join() 正常等待子线程退出更加安全可靠谱。

#### 2.1.4 joinable()

判断是否可以成功使用 join() 或者 detach()。

```cpp
int main()
{
    thread newThread(myThreadEntry);
  
    /* 如果返回true，证明可以调用join()或者detach() */
	if (newThread.joinable()) {
		cout << "1.joinable() == true" << endl;
	} else {
		cout << "1.joinable() == false" << endl;
	}

	newThread.detach();
	
    /* 如果返回false，证明调用过join()或detach()，二者都不能再调用了 */
	if (newThread.joinable()) {
		cout << "2.joinable() == true" << endl;
	} else {
		cout << "2.joinable() == false" << endl;
	}

    return 0;
}
```

输出结果：

```bash
1.joinable() == true
2.joinable() == false
```

### 2.2 其他创建线程的方法

#### 2.2.1 用类

定义一个TA类型，并重载一个无参数的()操作，**让其变成一个可调用对象**：

```cpp
#include <iostream>
#include <thread>
using namespace std;

/* 定义TA类 */
class TA {
public:
	void operator()() {  /* 重载()操作（无参数） */
		cout << "My thread start!!!" << endl;
		/* 线程执行代码 */
		cout << "My thread end!!!" << endl;
	}
};

int main()
{
	TA ta;                /* 声明TA类对象 */
	thread newThread(ta); /* ta：可调用对象 */
	newThread.join();     /* 等待newThrad执行完毕 */
	cout << "Hello World!" << endl;
    return 0;
}
```

输出结果：

```bash
My thread start!!!
My thread end!!!
Hello World!
```

**一个使用 detach() 的坑：**

```cpp
#include <iostream>
#include <thread>
using namespace std;

class TA {
public:
    int &m_i;             /* 定义一个应用 */
	TA(int &i) :m_i(i) {} /* 创建TA对象时需传入一个引用值 */
	void operator()() {
	    /* 主线程结束后，局部变量my_i值已被释放 */
	    /* 而m_i是my_i的引用，此时将产生不可预料的后果 */
		cout << "1.m_i = " << m_i << endl; 
		cout << "2.m_i = " << m_i << endl;
		cout << "3.m_i = " << m_i << endl;
		cout << "4.m_i = " << m_i << endl;
		cout << "5.m_i = " << m_i << endl;
		cout << "6.m_i = " << m_i << endl;
	}
};

int main()
{
	int my_i = 6; 
	TA ta(my_i);          /* 声明TA类对象 */
	thread newThread(ta); /* ta：可调用对象 */
	newThread.detach();   /* 分离主线程和子线程 */
    return 0;
}
```

**问题一**：调用detach()分离了主线程与子线程后，它们将分别独立运行，当主线程结束后,，局部变量 my_i 将被回收释放，此时子线程中的 m_i 引用了 my_i，这个值就是个无效的值，无法预料会有什么结果。

**问题二**：在主线程中，ta也是局部变量，主线程运行完毕，按常理来说，ta对象也被释放了，为什么调用detach()后子线程还能正常运行呢？

首先，主线程运行完毕后，ta对象肯定是不在了，但是这个对象不在了也没关系，因为这个对象实际上是被**复制**到线程中去的，所以执行完主线程后，ta对象会被销毁，但是所复制的ta对象依旧存在。

只要TA类对象里没有引用、没有指针，那么就不会产生问题。

验证代码：

```cpp
#include <iostream>
#include <thread>
using namespace std;

class TA {
public:
    int &m_i;
	TA(int &i) :m_i(i) {
		cout << "TA()构造函数被执行" << endl;
	}
	TA(const TA &ta) :m_i(ta.m_i){
		cout << "TA()拷贝构造函数被执行" << endl;
	}
	~TA(){
		cout << "~TA()析构函数被执行" << endl;
	}
	void operator()() {
		cout << "1.m_i = " << m_i << endl;
		cout << "2.m_i = " << m_i << endl;
		cout << "3.m_i = " << m_i << endl;
		cout << "4.m_i = " << m_i << endl;
		cout << "5.m_i = " << m_i << endl;
		cout << "6.m_i = " << m_i << endl;
	}
};

int main()
{
	int my_i = 6; 
	TA ta(my_i);          /* 调用TA构造函数 */
	thread newThread(ta); /* 调用TA拷贝构造函数 */
	newThread.detach();   /* 分离主线程和子线程 */
	cout << "Hello World!" << endl;
    return 0;
}
```

输出结果：

```bash
TA()构造函数被执行
TA()拷贝构造函数被执行
Hello World!
~TA()析构函数被执行
```

将 detach() 换成 join() 之后的输出结果：

```bash
TA()构造函数被执行
TA()拷贝构造函数被执行
1.m_i = 6
2.m_i = 6
3.m_i = 6
4.m_i = 6
5.m_i = 6
6.m_i = 6
~TA()析构函数被执行
Hello World!
~TA()析构函数被执行
```

**可以看出 join() 中释放了深度拷贝到子线程中的 ta 对象。**

#### 2.2.2 用lambda表达式

使用lambda表达式创建线程的示例代码如下：

```cpp
#include <iostream>
#include <thread>
using namespace std;

int main()
{
	auto myLamThread = [] {
		cout << "My thread start!!!" << endl;
		/* 线程执行代码 */
		cout << "My thread end!!!" << endl;
	};
	thread newThread(myLamThread); /* myLamThread：可调用对象 */
	newThread.join();              /* 等待线程结束 */
	cout << "Hello World!" << endl;
    return 0;
}
```

输出结果：

```bash
My thread start!!!
My thread end!!!
Hello World!
```
