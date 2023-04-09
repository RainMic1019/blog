---
title: C++并发与多线程笔记八：async、future、packaged_task、promise
date: 2023-04-08
categories:
 - 程序设计
tags:
 - C++
 - 并发
 - 多线程
---

## 1 前言

本文接上文 [C++并发与多线程笔记七：condition_variable、wait、notify_one/all](./cpp_multithreading_7.md) 的内容，主要记录 async、future、packaged_task、promise 概念以及用法。

## 2 std::async、std::future 创建后台任务并返回值

### 2.1 基本用法

`std::async` 是个函数模板，用来启动一个异步任务，启动一个异步任务后，它返回一个 `std::future` 类模板对象。

上述"启动一个异步任务"的本质含义就是自动创建一个线程并开始执行对应的线程入口函数，它返回一个 `std::future` 类模板对象，这个对象内就含有线程入口函数的返回值（即线程执行的返回结果）。

我们可以通过调用 `std::future` 对象的成员函数 `get()` 来获取结果。即 `std::future` 提供了一种访问异步操作结果的机制，就是说这个结果可能没办法马上拿到，在线程执行完毕后，才能拿到。

> 注：`std::future` 对象里会保存一个值，在将来的某个时刻能够拿到。

在下面的示例代码中，`std::future` 对象的 `get()` 成员函数等待线程结束并返回结果，也就是说 `get()` 函数在拿不到值时是阻塞的。另外，`std::future` 对象还有个 `wait()` 成员函数，等待线程返回，但函数本身并不返回结果，类似线程对象中的 `join()` 函数。

```cpp
#include <thread>
#include <future> /* 需要包含该头文件 */
#include <iostream>
using namespace std;

/* 线程入口函数 */
int myThread() {
  cout << "myThread() start thread ID = " << std::this_thread::get_id() << endl;
  std::chrono::milliseconds duration(5000); /* 定义间隔时间为 5000 ms */
  std::this_thread::sleep_for(duration);    /* 睡眠指定的间 */
  cout << "myThread() end thread ID = " << std::this_thread::get_id() << endl;
  return 5;
}

int main() {
  cout << "main() start thread ID = " << std::this_thread::get_id() << endl;

  /* 定义一个std::future类模板对象，变量类型为 int， */
  /* 并通过 std::async 启动一个异步任务 myThread */
  std::future<int> result = std::async(myThread);

  cout << "continue......!" << endl;

  /* get() 函数会阻塞在这，等待 myThread 线程执行完毕后返回  */
  cout << result.get() << endl;
  // result.wait(); /* 等待线程执行完毕，但不返回值 */

  cout << "main() end thread ID = " << std::this_thread::get_id() << endl;
  return 0;
}
```

输出结果：

```bash
main() start thread ID = 1860
continue......!
myThread() start thread ID = 3300
# 这里会阻塞 5000 ms
myThread() end thread ID = 3300
5
main() end thread ID = 1860
```

> 注：`std::future` 对象的 `get()` 成员函数只能调用一次，否则运行时会发生异常：`std::future_error`。

使用类成员函数作为 `std::async` 启动的线程回调函数有些额外的细节，示例代码如下，具体详见注释：

```cpp
#include <thread>
#include <future> /* 需要包含该头文件 */
#include <iostream>
using namespace std;

class A {
 public:
  /* 线程入口函数 */
  int myThread(int myParameter) {
    cout << myParameter << endl;
    cout << "myThread() start thread ID = " << std::this_thread::get_id() << endl;
    std::chrono::milliseconds duration(5000); /* 定义间隔时间为 5000 ms */
    std::this_thread::sleep_for(duration);    /* 睡眠指定的间 */
    cout << "myThread() end thread ID = " << std::this_thread::get_id() << endl;
    return 5;
  }
};

int main() {
  A objA;
  int tempParameter = 12;
  cout << "main() start thread ID = " << std::this_thread::get_id() << endl;

  /* std::async 传入类成员函数作为线程回调函数：第二个参数为类对象引用，第三个参数开始为线程回调函数的入参 */
  std::future<int> result = std::async(&A::myThread, &objA, tempParameter);

  /* std::async 传入全局函数作为线程回调函数：第二个参数开始为线程回调函数的入参 */
  // std::future<int> result = std::async(myThread, tempParameter);

  cout << "continue......!" << endl;
  cout << result.get() << endl;

  cout << "main() end thread ID = " << std::this_thread::get_id() << endl;
  return 0;
}
```

如果使用 `std::async` 启动异步任务后，不调用 `std::future` 的 `get()` 函数或 `wait()` 函数，程序在主线程退出前依旧会等待异步任务（子线程）结束，但通常不建议这样操作，可能会有风险。

### 2.2 扩展用法

#### 2.2.1 std::launch::deferred

我们可以通过额外向 `std::async` 传递一个 `std::launch` 类型（枚举）参数来达到一些特殊目的。比如 `std::launch::deferred` 表示线程入口函数调用被延迟到 `std::future` 对象的 `get()` 或者 `wait()` 函数被调用时才执行。

```cpp
/*......*/
std::future<int> result = std::async(std::launch::deferred, &A::myThread, &objA, tempParameter);
cout << result.get() << endl;
// result.wait()
/*......*/
```

输出结果：

```bash
main() start thread ID = 2572
continue......!
12
myThread() start thread ID = 2572
myThread() end thread ID = 2572
5
main() end thread ID = 2572
```

可以看到，异步任务（线程）中打印的线程 ID 与主线程 ID 一样，也就是说 `std::async` 传入 `std::launch::deferred` 参数后，调用 `std::future` 对象的 `get()` 或者 `wait()` 函数启动异步任务（线程），**实际上并没有创建新的线程，只起到一个延迟执行任务的功能**。

> 注：如果 `std::async` 传入 `std::launch::deferred` 参数后，没有调用 `std::future` 对象的 `get()` 或者 `wait()` 函数，那么这个异步任务（线程）不会被创建，也不会执行。

#### 2.2.2 std::launch::async

`std::launch` 还有另一个枚举类型 `async`，它是默认参数，表示调用 `std::async` 函数时就开始创建线程，并立即开始执行，也就是上文一开始讲的用法。

## 3 std::packaged_task

`std::packaged_task` 是个类模板，它的模板参数是各种可调用对象。通过 `std::packaged_task` 来把各种可调用对象包装起来，方便将来作为线程入口函数调用，并且 `std::packaged_task` 包装的对象中可以拿到线程的 `std::future` 对象，示例代码如下：

```cpp
#include <thread>
#include <future> /* 需要包含该头文件 */
#include <iostream>
using namespace std;

/* 线程入口函数 */
int myThread(int myParameter) {
  cout << myParameter << endl;
  cout << "myThread() start thread ID = " << std::this_thread::get_id() << endl;
  std::chrono::milliseconds duration(5000); /* 定义间隔时间为 5000 ms */
  std::this_thread::sleep_for(duration);    /* 睡眠指定的间 */
  cout << "myThread() end thread ID = " << std::this_thread::get_id() << endl;
  return 5;
}

int main() {
  cout << "main() start thread ID = " << std::this_thread::get_id() << endl;

  /* 把 myThread 函数通过 std::packaged_task 包装起来 */
  /* myThread 函数的返回值和入参都是 int 类型，因此模板类型为 int(int) */
  std::packaged_task<int(int)> myThreadTask(myThread);

  /* 使用包装好的 myThreadTask 对象，第二个参数为线程回调函数的入参，创建线程并开始执行  */
  std::thread mThreadObj(std::ref(myThreadTask), 12);

  mThreadObj.join(); /* 等待线程结束 */

  /* 根据 std::packaged_task 包装的对象获取 std::future 对象 */
  std::future<int> result = myThreadTask.get_future();
  cout << result.get() << endl;

  cout << "main() end thread ID = " << std::this_thread::get_id() << endl;

  return 0;
}
```

输出结果：

```bash
main() start thread ID = 7512
12
myThread() start thread ID = 3916
myThread() end thread ID = 3916
5
main() end thread ID = 7512
```

使用 `std::packaged_task` 包装 lambda 表达式的示例代码如下，这个 lambda 表达式的功能与上文的 `myThread()` 函数一样：

```cpp
  /* 把 lambda 表达式通过 std::packaged_task 包装起来 */
  std::packaged_task<int(int)> myThreadTask([](int myParameter) {
    cout << myParameter << endl;
    cout << "myThread() start thread ID = " << std::this_thread::get_id() << endl;
    std::chrono::milliseconds duration(5000); /* 定义间隔时间为 5000 ms */
    std::this_thread::sleep_for(duration);    /* 睡眠指定的间 */
    cout << "myThread() end thread ID = " << std::this_thread::get_id() << endl;
    return 5;
  });
```

`std::packaged_task` 包装起来的可调用对象是可以直接调用的，所以从这个角度来讲，`std::packaged_task` 对象也是一个可调用对象，比如：

```cpp
#include <future> /* 需要包含该头文件 */
#include <iostream>
using namespace std;

int main() {
  cout << "main() start thread ID = " << std::this_thread::get_id() << endl;

  /* 把 lambda 表达式通过 std::packaged_task 包装起来 */
  std::packaged_task<int(int)> myThreadTask([](int myParameter) {
    cout << myParameter << endl;
    cout << "myThread() start thread ID = " << std::this_thread::get_id() << endl;
    std::chrono::milliseconds duration(5000); /* 定义间隔时间为 5000 ms */
    std::this_thread::sleep_for(duration);    /* 睡眠指定的间 */
    cout << "myThread() end thread ID = " << std::this_thread::get_id() << endl;
    return 5;
  });

  /* 直接执行，并获取函数返回值 */
  myThreadTask(105);
  std::future<int> result = myThreadTask.get_future();
  cout << result.get() << endl;

  cout << "main() end thread ID = " << std::this_thread::get_id() << endl;

  return 0;
}
```

输出结果：

```bash
main() start thread ID = 15236
105
myThread() start thread ID = 15236
myThread() end thread ID = 15236
5
main() end thread ID = 15236
```

`std::packaged_task` 可以实现很多灵活的操作，比如创建一个容器，里面可以放入很多 `std::packaged_task` 对象，需要的时候再拿出来用：

```cpp
#include <vector>
#include <future> /* 需要包含该头文件 */
#include <iostream>
using namespace std;

vector<std::packaged_task<int(int)>> myTasks; /* 容器 */

int main() {
  cout << "main() start thread ID = " << std::this_thread::get_id() << endl;

  /* 把 lambda 表达式通过 std::packaged_task 包装起来 */
  std::packaged_task<int(int)> myThreadTask([](int myParameter) {
    cout << myParameter << endl;
    cout << "myThread() start thread ID = " << std::this_thread::get_id() << endl;
    std::chrono::milliseconds duration(5000); /* 定义间隔时间为 5000 ms */
    std::this_thread::sleep_for(duration);    /* 睡眠指定的间 */
    cout << "myThread() end thread ID = " << std::this_thread::get_id() << endl;
    return 5;
  });

  /* 将 myThreadTask 对象移动到容器中，移动完毕后容器size为1，myThreadTask为空 */
  /* 此处只能用 move，不能用 copy，因为 std::packaged_task 的拷贝构造函数被指定为删除的（delete），只能移动或者引用，无法拷贝。 */
  myTasks.push_back(std::move(myThreadTask));

  /* ...... */

  /* 将容器中的 std::packaged_task 对象取出来用 */
  std::packaged_task<int(int)> myThreadTaskBak;
  auto iter = myTasks.begin();        /* 用迭代器获取容器中的第一个元素 */
  myThreadTaskBak = std::move(*iter); /* 同样用 move 取出来 */
  myTasks.erase(iter);                /* 删除容器中的第一个元素，后续代码不可以在再使用 iter */
  myThreadTaskBak(123);               /* 调用拿出来的 std::packaged_task 对象 */

  cout << "main() end thread ID = " << std::this_thread::get_id() << endl;

  return 0;
}
```

输出结果：

```bash
main() start thread ID = 13080
123
myThread() start thread ID = 13080
myThread() end thread ID = 13080
main() end thread ID = 13080
```

## 4 std::promise

`std::promise` 也是一个类模板，我们可以在某个线程中给它赋值，然后我们可以在其他线程中把这个值取出来，示例代码如下：

```cpp
#include <thread>
#include <future> /* 需要包含该头文件 */
#include <iostream>
using namespace std;

void myThread(std::promise<int>& promiseValue, int calc) {
  int result;

  calc++;
  calc *= 10;

  /* 这里可以做各种运算，假设事件花了 5000 ms */
  std::chrono::milliseconds duration(5000); /* 定义间隔时间为 5000 ms */
  std::this_thread::sleep_for(duration);    /* 睡眠指定的间 */

  /* 计算出结果后，将其保存到 std::promise 对象中 */
  result = calc;
  promiseValue.set_value(result);

  return;
}

int main() {
  cout << "main() start thread ID = " << std::this_thread::get_id() << endl;

  int calc = 128;
  /* 声明一个 std::promise 对象，保存的值类型为 int */
  std::promise<int> myPpromiseValue;

  /* 创建并执行线程函数，等待其结束 */
  std::thread myThreadObj(myThread, std::ref(myPpromiseValue), 128);
  myThreadObj.join();  /* 用 std::thread 类型的对象，必须用 join 等待线程执行完毕 */

  /* promise 和 future 绑定，用于获取线程返回值 */
  std::future<int> result = myPpromiseValue.get_future();
  cout << result.get() << endl; /* get 只能调用一次  */

  cout << "main() end thread ID = " << std::this_thread::get_id() << endl;

  return 0;
}
```

输出结果：

```bash
main() start thread ID = 7520
1290
main() end thread ID = 7520
```

用法简单来说就是：通过 `std::promise` 保存一个值，在将来某个时刻我们通过把一个 `std::future` 绑定到这个 `std::promise` 上来得到绑定值。

也可以在上述代码中加多一个线程，在第二个子线程中获取第一个子线程设置的 `std::promise` 对象值，改造后的代码如下：

```cpp
#include <thread>
#include <future> /* 需要包含该头文件 */
#include <iostream>
using namespace std;

void myThread(std::promise<int>& promiseValue, int calc) {
  int result;

  calc++;
  calc *= 10;

  /* 这里可以做各种运算，假设事件花了 5000 ms */
  std::chrono::milliseconds duration(5000); /* 定义间隔时间为 5000 ms */
  std::this_thread::sleep_for(duration);    /* 睡眠指定的间 */

  /* 计算出结果后，将其保存到 std::promise 对象中 */
  result = calc;
  promiseValue.set_value(result);

  return;
}

void myThreadEx(std::future<int>& futureValue) {
  auto result = futureValue.get();
  cout << "myThreadEx result = " << result << endl;
  return;
}

int main() {
  cout << "main() start thread ID = " << std::this_thread::get_id() << endl;

  int calc = 128;
  /* 声明一个 std::promise 对象，保存的值类型为 int */
  std::promise<int> myPpromiseValue;

  /* 创建并执行线程函数，等待其结束 */
  std::thread myThreadObj(myThread, std::ref(myPpromiseValue), 128);
  myThreadObj.join();

  /* promise 和 future 绑定，用于获取线程返回值 */
  std::future<int> result = myPpromiseValue.get_future();

  /* 创建并执行第二个线程，等待其结束 */
  std::thread myThreadObjEx(myThreadEx, std::ref(result));
  myThreadObjEx.join();

  cout << "main() end thread ID = " << std::this_thread::get_id() << endl;

  return 0;
}
```

输出结果：

```bash
main() start thread ID = 2892
myThreadEx result = 1290
main() end thread ID = 2892
```

`std::promise` 可以在线程与线程之间传递各种类型的数据。

## 5 总结

C++ 并发与多线程的笔记中介绍了很多库中带的类型和函数，但学习这些东西的目的，并不是要把它们都用在自己的实际开发中，相反，如果能用最少的东西写出一个稳定、高效的多线程程序，这是更好的。代码写的优雅整洁的最终目的是给人看的，能让别人轻易看懂并理解的代码才是好代码。

我们为了成长，必须要阅读一些高手写的代码，从而快速实现自己的代码积累，等量变发生质变时，我们的技术会有一个大幅的提升。

此处学习这些内容的目的主要是方便我们未来能够读懂高手的代码，至少看别人代码，遇到 `std::promise`、`std::future` 等东西时不会一头雾水。
