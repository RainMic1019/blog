---
title: C++并发与多线程笔记五：unique_lock详解
date: 2022-06-22
categories:
 - 程序设计
tags:
 - C++
 - 并发
 - 多线程
---

## 1 前言

本文接上文 [C++并发与多线程笔记四：互斥量(概念、用法、死锁)](./cpp_multithreading_4.md) 的内容，主要纪录 `unique_lock` 的使用方法以及原理。

## 2 uniqie_lock取代lock_quard

uniqie_lock 是个类模板，它的功能跟 lock_quard 类似，但比 lock_quard 更灵活。在工作中，一般用 lock_quard （推荐使用）就足够了，但在一些特殊的场景下会用到 uniqie_lock。

在上篇文章中讲到了 lock_quard 取代了 mutex 的 lock() 和 unlock()，在 lock_quard 的构造函数中上锁，在析构函数中解锁，这点其实在 uniqie_lock 中也是一样的。

uniqie_lock 在使用上比 lock_quard 灵活，但代价就是效率会低一点，并且内存占用量也会相对高一些。

uniqie_lock 的缺省用法实际上与  lock_quard 一样，可以直接替换，代码如下：

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <list>
using namespace std;

class A {
 public:
  /* 把收到的消息（玩家命令）存到队列中 */
  void inMsgRecvQueue() {
    for (int i = 0; i < 100000; ++i) {
      cout << "inMsgRecvQueue exec, push an elem " << i << endl;
      std::unique_lock<std::mutex> m_guard1(m_mutex1);
      msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
    }
  }
  /* 消息队列不为空时，返回并弹出第一个元素 */
  bool outMsgLULProc(int& command) {
    std::unique_lock<std::mutex> m_guard1(m_mutex1);
    if (!msgRecvQueue.empty()) {
      command = msgRecvQueue.front(); /* 返回第一个元素 */
      msgRecvQueue.pop_front();       /* 移除第一个元素 */
      return true;
    }
    return false;
  }
  /* 把数据从消息队列中取出 */
  void outMsgRecvQueue() {
    int command = 0;
    for (int i = 0; i < 100000; ++i) {
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
  mutex m_mutex1;         /* 创建互斥量1 */
};

int main() {
  A obj;
  thread myInMsgObj(&A::inMsgRecvQueue, &obj);
  thread myOutMsgObj(&A::outMsgRecvQueue, &obj);
  myInMsgObj.join();
  myOutMsgObj.join();

  cout << "Hello World!" << endl;
  return 0;
}
```

## 3 uniqie_lock的第二个参数

uniqie_lock 的第二个参数是一个标志位，其可取参数详见下文。

### 3.1 std::adopt_lock

该标记表示这个互斥锁已经被 lock() 了，uniqie_lock 不会再重复上锁。

也就是说该标记的效果是：假设调用方线程已经拥有了互斥锁的所有权，通知 uniqie_lock 不需要再构造函数中 lock 这个互斥锁了。

```cpp
  /* 把收到的消息（玩家命令）存到队列中 */
  void inMsgRecvQueue() {
    for (int i = 0; i < 100000; ++i) {
      cout << "inMsgRecvQueue exec, push an elem " << i << endl;
      m_mutex1.lock(); /* 先lock()，才能在 unique_lock 中用 adopt_lock 标准 */
      std::unique_lock<std::mutex> m_guard1(m_mutex1, std::adopt_lock);
      msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
    }
  }
```

> 备注：lock_quard 中该标记的含义相同。

### 3.2 std::try_to_lock

假设我们在 myOutMsgObj 线程的回调函数中拿到互斥锁后，sleep 20 秒，就存在一个问题，myOutMsgObj 线程占用了互斥锁资源，却不向下执行，导致另一条线程 myInMsgObj 一直都没办法拿到互斥锁，也要等 20 秒，造成计算资源浪费。

```cpp
  /* 消息队列不为空时，返回并弹出第一个元素 */
  bool outMsgLULProc(int& command) {
    std::unique_lock<std::mutex> m_guard1(m_mutex1);

    std::chrono::milliseconds dura(20000); /* 20秒 */
    std::this_thread::sleep_for(dura);     /* sleep 20秒 */

    if (!msgRecvQueue.empty()) {
      command = msgRecvQueue.front(); /* 返回第一个元素 */
      msgRecvQueue.pop_front();       /* 移除第一个元素 */
      return true;
    }
    return false;
  }
```

为解决这个问题，uniqie_lock 引入了 try_to_lock 参数，它表示代码会尝试上锁，即使没有成功，也会立即返回，不会阻塞的等待。

```c
  /* 把收到的消息（玩家命令）存到队列中 */
  void inMsgRecvQueue() {
    for (int i = 0; i < 100000; ++i) {
      cout << "inMsgRecvQueue exec, push an elem " << i << endl;
      /* 尝试上锁 */
      std::unique_lock<std::mutex> m_guard1(m_mutex1, std::try_to_lock);
      if (m_guard1.owns_lock()) {  /* 如果拿到了锁 */
        msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
      } else {
        cout << "try_to_lock fail, do something else!!!" << endl;
      }
    }
  }
```

> 备注：使用 try_to_lock 参数前，线程中不能调用 lock()，否则会造成死锁。

### 3.3 std::defer_lock

该标记表示并没有给 mutex 加锁，即初始化了一个没有加锁的 mutex。使用该标记初始化的 uniqie_lock 对象可以灵活的调用 uniqie_lock 的成员函数，这点将在下文中一起演示。

## 4 uniqie_lock的成员函数

### 4.1 lock()

使用 `std::defer_lock` 参数初始化 uniqie_lock 对象可以调用unique_lock的成员函数上锁，并且无需在代码中解锁，它会自动解锁，有点类似智能指针。

```cpp
  /* 把收到的消息（玩家命令）存到队列中 */
  void inMsgRecvQueue() {
    for (int i = 0; i < 100000; ++i) {
      cout << "inMsgRecvQueue exec, push an elem " << i << endl;
      /* 初始化了没有加锁的m_mutex1 */
      std::unique_lock<std::mutex> m_guard1(m_mutex1, std::defer_lock);
      m_guard1.lock(); /* 调用unique_lock的成员函数上锁，并且无需在代码中解锁，它会自动解锁 */
      msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
    }
  }
```

### 4.2 unlock()

根据 4.1 的代码，我们可以知道 unique_lock 的成员函数 lock() 上锁后，在对象析构的时候会自动解锁，那为什么 unique_lock 还需要提供 unlock() 函数呢？

这里就体现了 unique_lock 的灵活性，它既具备智能指针自动销毁的特性，又可以在代码中手动解锁，方便程序在一段代码中对其中某几个部分上锁的需求，将一些非共享代码从锁中提取出来，从而提高效率。

> 备注：通常，在代码中，上锁的代码越少（粒度越小），效率越高。

```cpp
  /* 把收到的消息（玩家命令）存到队列中 */
  void inMsgRecvQueue() {
    for (int i = 0; i < 100000; ++i) {
      cout << "inMsgRecvQueue exec, push an elem " << i << endl;
      /* 初始化了没有加锁的m_mutex1 */
      std::unique_lock<std::mutex> m_guard1(m_mutex1, std::defer_lock);
      m_guard1.lock(); /* 调用unique_lock的成员函数上锁，并且无需在代码中解锁，它会自动解锁 */
      msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
      m_guard1.unlock();
      /* 非共享代码...... */
      m_guard1.lock();
      /* 共享代码...... */
      m_guard1.unlock();  /* 这个unlock() 可调可不调，m_guard1 对象析构时会自动解锁 */
    }
  }
```

### 4.3 try_lock()

这个成员函数与 `std::try_to_lock` 参数类似，尝试上锁，如果拿到锁了，则返回 true，否则返回 false。

```cpp
  /* 把收到的消息（玩家命令）存到队列中 */
  void inMsgRecvQueue() {
    for (int i = 0; i < 100000; ++i) {
      cout << "inMsgRecvQueue exec, push an elem " << i << endl;
      /* 初始化了没有加锁的m_mutex1 */
      std::unique_lock<std::mutex> m_guard1(m_mutex1, std::defer_lock);
      if (m_guard1.try_lock()) {   /* 如果拿到锁了 */
        msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
      } else {
        cout << "try_to_lock fail, do something else!!!" << endl;
      }
    }
  }
```

### 4.4 release()

该成员函数的作用是返回它所管理的 mutex 对象指针，并释放所有权；也就是说，这个 unique_lock  和 mutex 不再有关系。

调用以下代码创建 unique_lock 类型的对象 m_guard1 时，实际上是把 m_guard1 与 m_mutex1 对象绑定在一起了，release() 函数可以将这两个对象解绑，并返回绑定的 mutex 指针，即此处的 m_mutex1 。注意这里是解绑，并不是销毁。

```cpp
std::unique_lock<std::mutex> m_guard1(m_mutex1);
```

调用 release() 函数解绑后，我们必须保存返回的 mutex 指针，并在接下来的代码中自行管理。

```c
  /* 把收到的消息（玩家命令）存到队列中 */
  void inMsgRecvQueue() {
    for (int i = 0; i < 100000; ++i) {
      cout << "inMsgRecvQueue exec, push an elem " << i << endl;
      /* 初始化了没有加锁的m_mutex1 */
      std::unique_lock<std::mutex> m_guard1(m_mutex1);
      std::mutex* pmutex = m_guard1.release(); /* 解绑后返回之前绑定的m_mutex1，接下来我们要自行管理m_mutex1 */
      msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
      pmutex->unlock();          /* 解绑后需要自己解锁 */
    }
  }
```

## 5 uniqie_lock所有权的传递

当 uniqie_lock 与 mutex 对象绑定在一起才是一个完整的、能发挥作用的 uniqie_lock 实例，也就是说 uniqie_lock 需要和 mutex 配合使用，可以理解为 uniqie_lock 需要管理一个 mutex。

一个 mutex 对象只能被一个 uniqie_lock 对象所有（拥有），即同一个 mutex 无法被两个 uniqie_lock 对象同时使用，这就是 uniqie_lock 的所有权概念。

```cpp
/* 上锁两次，造成死锁 */
std::unique_lock<std::mutex> m_guard1(m_mutex1);
std::unique_lock<std::mutex> m_guard2(m_mutex1); /* 复制所有权（程序崩溃） */
```

m_guard1 拥有 m_mutex1 的所有权，并且 m_guard1 可以把自己 mutex(m_mutex1) 的所有权转移给其他的 uniqie_lock 对象。

> 备注：所有权可以转移，但不能复制，这与智能指针 unique_ptr 类似，智能指针指向的对象同样也是可以移动，但不能复制。

将 m_guard1 的所有权转移给 m_guard2，代码如下：

```cpp
std::unique_lock<std::mutex> m_guard1(m_mutex1);
/* 移动语义，相当于m_guard1与m_mutex1解绑；m_guard2与m_mutex1绑定 */
std::unique_lock<std::mutex> m_guard2(std::move(m_guard1)); 
```

转移unique_lock所有权的扩展方法：

```cpp
  std::unique_lock<std::mutex> rtn_unique_lock() {
    std::unique_lock<std::mutex> temp_guard(m_mutex1);
    /* 从函数返回一个局部的temp_guard（移动构造函数）*/
    /* 返回局部对象temp_guard会让系统创建临时的unique_lock对象，并调用unique_lock的移动构造函数 */
    return temp_guard;
  }
  /* 把收到的消息（玩家命令）存到队列中 */
  void inMsgRecvQueue() {
    for (int i = 0; i < 100000; ++i) {
      cout << "inMsgRecvQueue exec, push an elem " << i << endl;
      /* rtn_unique_lock 函数中 temp_guard 对象的所有权转移到 m_guard1 中了 */
      std::unique_lock<std::mutex> m_guard1 = rtn_unique_lock();
      msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
    }
  }
```
