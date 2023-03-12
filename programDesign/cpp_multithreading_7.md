---
title: C++并发与多线程笔记七：condition_variable、wait、notify_one/all
date: 2023-03-12
categories:
 - 程序设计
tags:
 - C++
 - 并发
 - 多线程
---

## 1 前言

本文接上文 [C++并发与多线程笔记六：单例模式下的数据共享](./cpp_multithreading_6.md) 的内容，主要纪录条件变量std::condition_variable、wait、notify_one、notify_all 概念以及用法。

## 2 条件变量 std::condition_variable、wait、notify_one

假设现在有两条线程，线程 A 负责处理消息（等待一个条件满足），线程 B 负责往消息队列中添加消息，当线程 B 通知线程 A 条件满足时，线程 A 才继续向下执行，这就是**条件变量**的作用。

首先，我们回顾一下 [C++并发与多线程笔记五：unique_lock详解](./cpp_multithreading_5.md) 中的示例代码：

```cpp
#include <list>
#include <mutex>
#include <thread>
#include <iostream>
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

  return 0;
}
```

其中，`outMsgLULProc()` 函数会被频繁调用，每次调用都会去上锁，会占用不必要的资源，再加上双重检查（锁定）后，可以缓解这种情况，代码如下：

```cpp
  bool outMsgLULProc(int& command) {
    if (!msgRecvQueue.empty()) {
      std::unique_lock<std::mutex> m_guard1(m_mutex1);
      if (!msgRecvQueue.empty()) {
        command = msgRecvQueue.front(); /* 返回第一个元素 */
        msgRecvQueue.pop_front();       /* 移除第一个元素 */
        return true;
      }
    }
    return false;
  }
```

但双重检查仍然无法避免频繁上锁的问题，如果队列中有消息存在时，能通过主动通知线程处理，则能够进一步提高程序运行效率，也就是上文提到的**条件变量**的作用。

`std::condition_variable` 实际上是一个和条件相关的类，即等待一个条件达成，它需要跟互斥量配合工作，用的时候需要生成一个类的对象。

基于上文的示例代码，我们在  `class A` 中添加一个私有的 `condition_variable` 类型对象，并做以下处理：

1. 在 `outMsgRecvQueue()` 函数（线程A）中调用条件变量的 `wait()` 函数**等待**队列消息；
2. 在 `inMsgRecvQueue()` 函数（线程B）中调用条件变量的 `notify_one()` 函数**通知**线程A处理消息。

```cpp
#include <list>
#include <mutex>
#include <thread>
#include <iostream>
using namespace std;

class A {
 public:
  /* 把收到的消息（玩家命令）存到队列中 */
  void inMsgRecvQueue() {
    for (int i = 0; i < 100000; ++i) {
      cout << "inMsgRecvQueue exec, push an elem " << i << endl;
      std::unique_lock<std::mutex> m_guard1(m_mutex1);
      msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
      /* 通知并尝试唤醒阻塞在 wait() 函数的线程 */
      m_cond.notify_one();
    }
  }
  /* 把数据从消息队列中取出 */
  void outMsgRecvQueue() {
    int command = 0;
    while (true) {
      std::unique_lock<std::mutex> m_guard1(m_mutex1);
      /* 
       * 此处的条件变量对象，等待一个 lambda 表达式（可调用对象，类似函数）。
       * 如果这个 lambda 表达式返回 true，那么 wait() 直接返回；
       * 如果这个 lambda 表达式返回 false，那么 wait() 会将解锁互斥量，并在本行阻塞（睡眠/挂起），
       * 阻塞状态会一直持续到其他某个线程调用 notify_one() 函数通知为止。
       * 如果 wait() 不指定第二个参数：m_cond.wait(m_guard1)，那么默认与 lambda 表达式返回 false 的效果一样。
       * 
       * 当其他线程调用 notify_one() 函数唤醒 wait() 后，行为如下：
       * 1. wait() 函数会不断尝试重新获取互斥锁，如果拿不到锁，它依然会处于阻塞状态；
       * 2. 如果 wait() 函数拿到了互斥锁，则上锁并重复判断参数二中的 lambda 表达式（或回调函数）：
       *   (1) 如果表达式为 false 则解锁并继续阻塞；
       *   (2) 如果表达式为 true 则 wait() 返回，继续执行后续代码（注意：此时仍处于上锁状态）。
       * 3. 如果 wait() 没有指定第二个参数，则直接返回，即无条件唤醒。
       */
      m_cond.wait(m_guard1, [this] {
        if (!msgRecvQueue.empty()) {
          return true;
        }
        return false;
      });

      /* 处理队列中的消息 */
      command = msgRecvQueue.front(); /* 返回第一个元素 */
      msgRecvQueue.pop_front();       /* 移除第一个元素 */
      cout << "outMsgRecvQueue exec, pop an elem " << command << endl;
      m_guard1.unlock(); /* 如果后续还有处理代码，则可以提前手动解锁，减少另一个线程的等待时间 */
    }
  }

 private:
  list<int> msgRecvQueue; /* 容器（实际上是双向链表）：存放玩家发生命令的队列 */
  mutex m_mutex1;         /* 创建互斥量1 */
  condition_variable m_cond; /* 创建条件变量对象 */
};

int main() {
  A obj;
  thread myInMsgObj(&A::inMsgRecvQueue, &obj);
  thread myOutMsgObj(&A::outMsgRecvQueue, &obj);
  myInMsgObj.join();
  myOutMsgObj.join();

  return 0;
}
```

## 3 上述代码深入思考

需要注意的是，上述代码再加了条件变量之后并不是 `inMsgRecvQueue()` 和 `outMsgRecvQueue()` 来回交替各执行一次，线程B `notify_one()` 后，线程A的 `wait()` 函数会去获取互斥锁，但线程B同样也会去获取互斥锁，不见得 `wait()` 就一定能获取成功，没准线程B刚 `notify_one()` 完，自己又上锁成功了，然后 push 更多的消息到队列中，也就是说线程A 的 `wait()` 函数返回成功后，消息队列中可能不止有一条数据，实际情况得看 CPU 的调度。

另外，如果线程A处理消息的逻辑比较复杂耗时，当线程A提前解锁，线程B那边则又可以继续 push 并调用 `notify_one()` 通知，此时线程A正在干活，那这次调用的 `notify_one()` 就没啥实际作用了，即当线程A正在被 `wait()` 阻塞时，其他线程调用 `notify_one()` 函数才是有效的。

当线程A处理不过来，最常见的解决办法有：

1. 线程A支持一次性处理多条消息；
2. 多开几条线程处理队列中的消息。
3. ...等等

## 4 notify_all()

上文讲的 `notify_one()` 函数只能通知一个线程，如果开多几个线程，那么每调用一次 `notify_one()` 函数则只能随机通知某一个线程（这里的随机主要看 CPU 的调度），比如这里再创建多一个处理消息的线程：

```cpp
int main() {
  A obj;
  thread myInMsgObj(&A::inMsgRecvQueue, &obj);
  thread myOutMsgObj(&A::outMsgRecvQueue, &obj);
  thread myOutMsgObj2(&A::outMsgRecvQueue, &obj);
  myInMsgObj.join();
  myOutMsgObj.join();
  myOutMsgObj2.join();

  return 0;
}
```

在 `outMsgRecvQueue()` 函数中打印一下当前线程的ID，编译运行可以看见有不同的线程ID打印，也就是说 `notify_one()` 函数具体唤醒哪一个线程是不一定，都有可能：

```cpp
void outMsgRecvQueue() {
    int command = 0;
    while (true) {
      std::unique_lock<std::mutex> m_guard1(m_mutex1);
      m_cond.wait(m_guard1, [this] {
        if (!msgRecvQueue.empty()) {
          return true;
        }
        return false;
      });

      /* 处理队列中的消息 */
      command = msgRecvQueue.front(); /* 返回第一个元素 */
      msgRecvQueue.pop_front();       /* 移除第一个元素 */
      cout << "outMsgRecvQueue exec, pop an elem " << command;
      cout << "  threadID = " << this_thread::get_id() << endl; /* 打印当前线程id */
      m_guard1.unlock(); /* 如果后续还有处理代码，则可以提前手动解锁，减少另一个线程的等待时间 */
    }
  }
```

此时用 `notify_all()` 函数则可以同时通知两个线程：

```cpp
  void inMsgRecvQueue() {
    for (int i = 0; i < 100000; ++i) {
      cout << "inMsgRecvQueue exec, push an elem " << i << endl;
      std::unique_lock<std::mutex> m_guard1(m_mutex1);
      msgRecvQueue.push_back(i); /* 假设数字 i 就是收到的玩家命令 */
      /* 通知并尝试唤醒阻塞在 wait() 函数的线程 */
      m_cond.notify_all();
    }
  }
```

但需要注意的是，由于两个处理消息的线程实际上用的是同一把锁，那么其实只有拿到锁的那个线程能继续执行，另一个线程还是会阻塞在 `wait()` 函数，等待下一次通知，在实际应用中两个线程可能干的是不同的活，每个线程得用不同的互斥锁锁对象和条件变量对象。
