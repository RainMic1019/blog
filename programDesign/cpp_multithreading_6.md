---
title: C++并发与多线程笔记六：单例模式下的数据共享
date: 2022-09-04
categories:
 - 程序设计
tags:
 - C++
 - 并发
 - 多线程
publish: false
---

## 1 前言

本文接上文 [C++并发与多线程笔记五：unique_lock详解](./cpp_multithreading_5.md) 的内容，主要纪录单例设计模式下的数据共享以及 `stb::call_once`。

## 2 设计模式概述

设计模式是软件开发人员在软件开发过程中面临的一般问题的解决方案。这些解决方案是众多软件开发人员经过相当长的一段时间的试验和错误总结出来的。

设计模式是一套被反复使用的、多数人知晓的、经过分类编目的、代码设计经验的总结。使用设计模式是为了重用代码、让代码更容易被他人理解、保证代码可靠性。

之前我学习《设计模式——可复用面向对象软件的基础》时，也写过很笔记，感兴趣的可以看看：[设计模式相关笔记](https://liuyuxin.site/categories/设计模式/)。

## 3 单例设计模式

单例模式（Singleton Pattern）是最简单的设计模式之一，它的使用频率很高，原理和用法详见：[设计模式学习（五）：单例模式及其优化示例](./../designPatterns/singlet on_pattern.md)。

简单来讲，单例模式就是在整个项目中，有某个或者某些特殊的类，只能创建一个属于该类的对象。

在 C++ 中简单的实现一个单例类 `MyCAS`，代码如下：

```cpp
#include <iostream>
#include <thread>
#include <mutex>
using namespace std;

class MyCAS { /* 这是一个单例类 */
 private:
  static MyCAS* m_instance; /* 私有静态成员变量 */

  /* 私有化构造函数（无法通过 MyCAS a; 来创建对象） */
  MyCAS() {}

 public:
  static MyCAS* GetInstance() {
    if (m_instance == nullptr) {
      m_instance = new MyCAS();
      /* 创建一个 CRecycle 类的静态对象 */
      /* 它在创建 m_instance 时构造，又在程序退出时析构，析构时销毁 m_instance 对象 */
      static CRecycle recycle;
    }
    return m_instance;
  }

  /* 类中套类，用来释放 MyCAS 的单例对象 */
  class CRecycle {
   public:
    ~CRecycle() {
      if (MyCAS::m_instance != nullptr) {
        delete MyCAS::m_instance;
        MyCAS::m_instance = nullptr;
      }
    }
  };
};

/* 静态成员变量必须在类外进行初始化：类内声明，类外初始化 */
/* MyCAS 类静态变量初始化 */
MyCAS* MyCAS::m_instance = nullptr;

int main() {
  MyCAS* p_a = MyCAS::GetInstance(); /* 创建 MyCAS 类的单例对象 */
  return 0;
}
```

需要注意的是，通过以上方式创建的 `p_a` 对象，由于是个指针，在程序结束的时候 `m_instance` 不会自动释放，需要通过特殊的设计来释放 `m_instance` 变量，避免产生野指针。

## 4 单例模式下的数据共享

如果一个单例类对象的数据是只读的，那么在多线程中访问是没有问题的，但注意需要在其他线程运行之前，在主线程中初始化这个单例类的对象。

在实际项目中，我们可能需要在子线程中初始化单例类的对象，甚至这种线程可能不止一个。此时我们写的 `GetInstance()` 函数就必须上锁互斥了。

比如在以下代码中，两个线程 `myThread1` 和 `myThread2` 都使用同一个入口函数，两个流程同时执行 myThreadFun 函数，即同时执行 `GetInstance()`。

```cpp
void myThreadFun() {
  cout << "My thread start!" << endl;
  MyCAS* p_a = MyCAS::GetInstance(); /* 创建 MyCAS 类的单例对象 */
  cout << "My thread end!" << endl;
  return;
}

int main() {
  /* 创建两条线程 */
  std::thread myThread1(myThreadFun);
  std::thread myThread2(myThreadFun);

  myThread1.join();
  myThread2.join();

  return 0;
}
```

有一种情况：线程1刚执行完 `m_instance == nullptr` 的判断，还没进行 `new` 操作，此时操作系统立马转头去执行线程2（这个切换随时都可能出现），这时线程2判断 `m_instance` 确实为空，因此就先 `new MyCAS()` 了，等到操作系统切回到线程1又会再 `new` 出来一个新对象。

因此，我们需要给 `GetInstance()` 函数上个互斥锁，代码如下:

```cpp
...
std::mutex resMutex;

class MyCAS { /* 这是一个单例类 */
 ...
 public:
  static MyCAS* GetInstance() {
    std::unique_lock<std::mutex> myMutex(resMutex); /* 自动上锁，作用域结束后解锁 */
    if (m_instance == nullptr) {
      m_instance = new MyCAS();
      static CRecycle recycle;
    }
    return m_instance;
  }
  ...
};
```

但这么写有个问题，每次调用 `GetInstance()` 函数都需要上锁解锁，会导致效率降低。这个案例中其实只需要初始化 `m_instance` 对象的时候上锁，非初始化时调用 `GetInstance()` 函数不上锁也行。

这里我们加多一个判断——双重检查，优化代码效率：

```cpp
  static MyCAS* GetInstance() {
    if (m_instance == nullptr) { /* 双重检查，提高效率 */
      std::unique_lock<std::mutex> myMutex(resMutex); /* 自动上锁，作用域结束后解锁 */
      if (m_instance == nullptr) {
        m_instance = new MyCAS();
        /* 创建一个 CRecycle 类的静态对象 */
        /* 它在创建 m_instance 时构造，又在程序退出时析构，析构时销毁 m_instance 对象 */
        static CRecycle recy le;
      }
    }
    return m_instance;
  }
```

## 5 std::call_once()

这是一个函数模板，是 C++11 引入的，函数代码如下：

```cpp
/* Microsoft Visual Studio/2019/Community/VC/Tools/MSVC/14.28.29910/include/mutex */
void(call_once)(once_flag& _Once, _Fn&& _Fx, _Args&&... _Ax) noexcept(
    noexcept(_STD invoke(_STD forward<_Fn>(_Fx), _STD forward<_Args>(_Ax)...))) /* strengthened */ {
    // call _Fx(_Ax...) once
    // parentheses against common "#define call_once(flag,func) pthread_once(flag,func)"
    int _Pending;
    if (_RENAME_WINDOWS_API(__std_init_once_begin_initialize)(&_Once._Opaque, 0, &_Pending, nullptr) == 0) {
        _CSTD abort();
    }

    if (_Pending != 0) {
        _Init_once_completer _Op{_Once, _Init_once_init_failed};
        _STD invoke(_STD forward<_Fn>(_Fx), _STD forward<_Args>(_Ax)...);
        _Op._DwFlags = 0;
    }
}
```

该函数的第二个参数是一个函数名 a()：

* `call_once` 的功能是能够保证函数 a 只被调用一次。
* `call_once` 具备互斥量这种能力，而且效率上，比互斥量消耗的资源更少。
* `call_once` 需要雨一个标记结合使用，这个标记 `std::once_flag` 其实是一个结构。
* `call_once` 就是通过这个标记来决定对应的函数 a() 是否执行，调用 `call_once()` 成功后，`call_once()` 就把这个标记设置为已调用状态。
* 后续再次调用 `call_once()`，只要 `std::once_flag` 被设置为了已调用状态，那么对应的函数 a() 就不会再被执行了。

结合上一小节中的例子，我们只需要把 `m_instance` 对象的初始化代码放到 `call_once` 中即可实现执行一次的效果，也就不用上锁了。

```cpp
#include <iostream>
#include <thread>
#include <mutex>
using namespace std;

std::once_flag g_flag; /* 定义标记 */

class MyCAS { /* 这是一个单例类 */
 private:
  static MyCAS* m_instance; /* 私有静态成员变量 */

  /* 私有化构造函数（无法通过 MyCAS a; 来创建对象） */
  MyCAS() {
  }

  static void CreateInstance() {
    m_instance = new MyCAS();
    /* 创建一个 CRecycle 类的静态对象 */
    /* 它在创建 m_instance 时构造，又在程序退出时析构，析构时销毁 m_instance 对象 */
    static CRecycle recycle;
  }

 public:
  static MyCAS* GetInstance() {
    std::call_once(g_flag, CreateInstance); /* 只调用一次 CreateInstance */
    return m_instance;
  }

  /* 类中套类，用来释放 MyCAS 的单例对象 */
  class CRecycle {
   public:
    ~CRecycle() {
      if (MyCAS::m_instance != nullptr) {
        delete MyCAS::m_instance;
        MyCAS::m_instance = nullptr;
      }
    }
  };
};

/* 静态成员变量必须在类外进行初始化：类内声明，类外初始化 */
/* MyCAS 类静态变量初始化 */
MyCAS* MyCAS::m_instance = nullptr;

void myThreadFun() {
  cout << "My thread start!" << endl;
  MyCAS* p_a = MyCAS::GetInstance(); /* 创建 MyCAS 类的单例对象 */
  cout << "My thread end!" << endl;
  return;
}

int main() {
  std::thread myThread1(myThreadFun);
  std::thread myThread2(myThreadFun);

  myThread1.join();
  myThread2.join();

  return 0;
}
```
