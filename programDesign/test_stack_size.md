---
title: C/C++测试程序运行时所需的栈大小
date: 2021-11-21
categories:
 - 程序设计
tags:
 - C
 - C++
---

## 1 前言

由于工作需要，最近在研究怎么测试程序用掉的栈大小，这里感谢领导给我提供了一个很不错的思路，只需要要一个 alloca 函数就能搞定，在此记录分享一下，以备不时之需。

## 2 测试原理

### 2.1 alloca函数

在讲原理之前，我们需要先了解一个关键的函数—— **alloca 函数**。

alloca 函数是一个内存分配函数，与malloc、calloc、realloc类似，但是注意一个重要的区别：**alloca 是在栈(stack)上申请空间的，并且用完马上就释放。**

它包含在头文件malloc.h中，在某些系统中会宏定义成_alloca使用，声明入下：

```c
void * __cdecl  alloca(size_t);
```

需要注意的是：

1. 在调用 alloca 的函数返回时，**它分配的内存会自动释放**。也就是说，用 alloca 分配的内存在栈上。
2. alloca 不具可移植性, 而且在没有传统堆栈的机器上很难实现。当它的返回值直接传入另一个函数时会带来问题，因为他分配在栈上。

由于这些原因，alloca不宜用在必须广泛移植的程序中。

示例代码：

```c
int main() {
  int *p = (int *)alloca(sizeof(int)*10);
  free(p);  //此时不能用free()去释放，会导致错误
  return0;
}
```

### 2.2 原理分析

既然了解了 alloca 函数的特性，并且已知栈是一块连续的内存空间，那么原理就很好理解了。

首先，在程序初始化的时候，我们可以用 alloca 函数**在栈上分配一块足够大的内存并记录内存起始地址**，然后**给这块内存数据打上标记**，例如将每个字节都设置成 0xAB。

然后，在程序运行的过程中，肯定会消耗栈空间，此时，**我们给栈标记上的 0xAB 数据就会被修改**，当我们需要知道当前程序所消耗的栈峰值时，就可以计算我们上个步骤中分配的栈内存中还剩多少仍然被标记的数据。

最后，**使用分配的栈内存大小减仍然被标记的内存大小，就可以得到程序在运行过程中所消耗的栈峰值。**

## 3 示例代码

首先，我们实现 stack_usage_fill 函数，用于从栈上分配一块空间，并对其进行标记填充，代码如下：

```c
static const char* stack_usage_fill(int tag, size_t stack_size) {
    char* top = (char*)alloca(stack_size);
    if (top) {
        memset(top, tag, stack_size);
    }
    return top;
}
```

然后，我们实现 stack_usage_test 函数，用于计算程序使用的栈大小，代码如下：

```c
static size_t stack_usage_test(int tag, size_t stack_size, const char* top) {
    size_t unused = 0;
    char ctag = (char)tag;
    for (unused = 0; unused < stack_size && top[unused] == ctag; unused++) {
    }
    return stack_size - unused;
}
```

接下来，我们写个简单的 foo 函数来测试一下：

```c
#include <stdio.h>
#include <string.h>
#include <malloc.h>

#define STACK_TEST_TAG 0xAB
#define STACK_TEST_SIZE 1024 * 64

······

static int foo() {
    char buf[1024];
    memset(buf, 0x11, sizeof(buf));
    int sum = 0;
    for (int i = 0; i < sizeof(buf); i++) {
        sum += buf[i];
    }
    return sum;
}

int main() {
    const char* top = stack_usage_fill(STACK_TEST_TAG, STACK_TEST_SIZE);
    size_t stack_max_used1 = stack_usage_test(STACK_TEST_TAG, STACK_TEST_SIZE, top);
    int sum = foo();
    size_t stack_max_used = stack_usage_test(STACK_TEST_TAG, STACK_TEST_SIZE, top);
    printf("stack_max_used %u\n", stack_max_used1);
    printf("stack_max_used %u, sum=%d\n", stack_max_used, sum);
    return 0;
}
```

输出结果：

```bash
stack_max_used 4
stack_max_used 1428, sum=17408
```

将 foo 函数中申请的局部数据 buf 的大小改为 2048 之后，输出结果如下：

```bash
stack_max_used 0
stack_max_used 2824, sum=34816
```

可以看出使用以上方法可以大致估算出程序运行时所需的栈大小。

## 4 注意事项

需要注意的是，在程序中使用 printf 函数，会根据实际平台的处理消耗一些栈空间，从而导致数据不准确，比如，如果将上一章节中的 main 函数代码修改一下，将 printf 函数提前：

```c
int main() {
    const char* top = stack_usage_fill(STACK_TEST_TAG, STACK_TEST_SIZE);
    size_t stack_max_used1 = stack_usage_test(STACK_TEST_TAG, STACK_TEST_SIZE, top);
    printf("stack_max_used %u\n", stack_max_used1);
    int sum = foo();
    size_t stack_max_used = stack_usage_test(STACK_TEST_TAG, STACK_TEST_SIZE, top);
    printf("stack_max_used %u, sum=%d\n", stack_max_used, sum);
    return 0;
}
```

输入结果如下，可以明显的看到，此时的结果算上了第一次调用 printf 函数消耗的栈空间，从而导致数据不准确：

```bash
stack_max_used 0
stack_max_used 7244, sum=17408
```
