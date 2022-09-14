---
title: C语言使用回调函数降低耦合（隔离变化）
date: 2020-10-29
categories:
 - 程序设计
tags:
 - C
---

## 1 前言

耦合性是程序结构中各个模块之间相互关联的度量。它取决于各个模块之间接口的复杂程度、调用模块的方式以及哪些信息通过接口。

编写代码有两个核心点：隔离变化、降低复杂度，而解耦是达到这两个目标的重要手段。在本周的培训中，学习了使用回调函数隔离代码中变化的部分，以下做一个记录。

## 2 使用回调函数降低耦合

问题描述：实现回家函数come_home，传入出发的时间，打印回家的方式以及到家的时间，有两种回家方式：开车、走路。其中开车需要1小时，走路需要3小时。

功能分析：由于回家方式不同，所用时间不同，因此到家的时间不同，可以将come_home函数代码分为变化部分以及不变部分，将变化部分写在回调函数中，实现隔离变化的目的。

- 变化部分：输出回家方式，根据出发时间，计算到家时间。
- 不变部分：输出到家时间。

### 2.1 定义回调函数指针

```c
typedef int (*on_arrival_t)(void* ctx, int departure_time);
```

### 2.2 编写come_home函数

```c
void come_home(int departure_time, on_arrival_t on_arrival, void* ctx)
{
 int arrival_time = on_arrival(ctx, departure_time); /* 变化部分：调用回调函数计算到家时间 */
 printf("arrival_time: %d\n", arrival_time);         /* 不变部分：打印到家时间 */
}
```

### 2.3 实现具体的回调函数

```c
/* 开车回家 */
int on_drive(void* ctx, int departure_time) {
 int  arrival_time = departure_time + 1; /* 计算开车回家时间 */
 printf("drive\n");                      /* 打印回家方式 */

 return arrival_time;
}

/* 走路回家 */
int on_walk(void* ctx, int departure_time) {
 int  arrival_time = departure_time + 3; /* 计算走路回家时间 */
 printf("walk\n");                       /* 打印回家方式 */

 return arrival_time;
}
```

### 2.4 main函数

```c
int main(int argc, char* argv[])
{
 come_home(17, on_drive, NULL); /* 17点开车回家 */
 come_home(17, on_walk, NULL);  /* 17点走路回家 */
 
 return 0;
}
```

### 2.5 输出

```bash
drive
arrival_time: 18
walk
arrival_time: 20
```

## 3 总结

在上面的例子中，当开车、走路的时间发生变化或者需要补充其他代码，又或者需要增加其他回家方式时，我们只需要修改或增加回调函数即可，不需要修改come_home函数，由此可见，回调函数是隔离代码中变化部分的有效手段。
