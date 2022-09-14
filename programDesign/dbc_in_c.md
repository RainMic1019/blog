---
title: 契约式设计（Dbc）以及其在C语言中的应用
date: 2020-11-08
categories:
 - 程序设计
tags:
 - C
---

## 1 前言

本周培训中接触到了契约式设计，查阅了许多资料后仍然是一知半解，与同事和大学导师交流了后，受到了启发，本文记录总结了个人对契约式设计的学习感悟。

## 2 什么是契约式设计？

### 2.1 概述

契约式设计/契约式编码（Design by Contract，以下简称 DbC ）是一种设计计算机软件的方法。这种方法描述了，软件设计者应该为软件组件定义正式的、准确的、可验证的接口规范，它扩展了抽象数据类型对于先验条件、后验条件和不变式的一般定义。这些规范称为“契约”，它是一个比喻，类似于商业契约/合同的条件和职责，对于以上概念，下文会详细描写。

> 备注：Dbc起源于Eiffel语言，但它的应用范围与语言无关。

### 2.2 目的

Dbc的主要目的是希望程序员能够在设计程序时明确地规定一个模块单元（具体到面向对象，就是一个类的实例）在调用某个操作前后应当属于何种状态。

**Dbc不是一种编程范型，它是一种设计风格，一种语法规范。**

### 2.3 思路

契约式设计强调三个概念：先验条件，后验证条件和不变式。

- **先验条件**：对于函数来讲，期望所有调用它的模块都保证一定的进入条件，这样它就不用去处理不满足先验条件的情况，先验条件发生在每个函数的最开始。
- **后验条件**：保证退出时给出特定的属性，也就是说函数保证能做到的事情，函数完成时的状态，函数有这一事实表示它会结束，不会无休止的循环，后验条件发生在每个函数的最后。
- **不变式**：在进入时假定，并在退出时保持一些特定的属性，不变式实际上是前置条件和后置条件的交集，违反这些操作会导致程序抛出异常。从调用者的角度来看，该条件总是为真，在函数的内部处理过程中，不变项可以为变，但在函数结束后，控制返回调用者时，不变项必须为真。

### 2.4 六大原则

（1）区分命令和查询。查询返回一个结果，但不改变对象的可见性质。命令改变对象的状态，但不一定返回结果。

（2）将基本查询同派生查询分开。派生查询可以用基本查询来定义。

（3） 针对每个派生查询，设定一个后验条件，使用一个或多个基本查询的结果来定义它。这样我们只要知道基本查询的值，也就能知道派生查询的值。

（4） 对于每个命令都撰写一个后验条件，规定每个基本查询的值。结合“用基本查询定义派生查询”的原则，我们已经能够知道每个命令的全部可视效果。

（5）对于每个查询和命令，采用一个合适的先验条件。先验条件限定了客户调用查询和命令的时机。

（6） 撰写不变式来定义对象的恒定特性。类是某种抽象的体现，应当将注意力集中在最重要的属性上，以帮助读者建立关于类抽象的正确概念模型。

## 3 为什么要使用Dbc

- **获得更优秀的设计**：Dbc鼓励程序员思考诸如“函数的先验条件是什么”这样的问题，这样有助于程序员理清概念，函数获得了更清晰的描述，同时限制了函数得调用，对非法调用得结果也很清楚。
- **保证系统健壮性**：系统地运用异常 当例程被非法使用（先验条件失败）或者例程没有遵循契约的规定（后验条件或者不变式失败）时，就会发生异常。
- **更有效得组织模块间通信**：尽可能准确地规定模块彼此通信得责任和权力。
- **提高可靠性**：编写契约可以帮助开发者更好地理解代码，契约有助于测试（契约可以随时关闭或者启用）。
- **简化调试**：契约能把错误牢牢地定位，开发期间的支持 由于断言判断为假而在运行时展现出来的错误将被精确地定位。
- ......

等等。

## 4 如何在编程中使用Dbc

上文简单描述了Dbc的定义、它的核心概念以及大致的思路，可以看出在平时编程的时候很多程序员都在或多或少地做一些这些工作。比如在函数定义开始时判断给定的指针参数是否不能为NULL，或者程序退出时释放临时分配的内存块。但在具体细节则基本依靠个人习惯，有人喜欢调用方检查，有人则喜欢被调用方检查。

Eiffel与这些实践的不同之处在于它是唯一一种以语法形式将先验条件、后验条件和不变式规定为被调用方代码中独立语法块的语言，其关键字分别require、ensure、invariant。

以下例子为Eiffel语言，来自：[http://www.eiffel.com/developers/design_by_contract_in_detail.html](http://www.eiffel.com/developers/design_by_contract_in_detail.html)

- **功能描述**：定义一个范型类DICTIONARY [ELEMENT]的put方法；
- **先验条件**：当前的元素个数count小于容量capacity，键值key不是空字符串；
- **后验条件**：字典中存在元素x；使用健值key获取的元素是x；操作后元素个数count等于操作前元素个数old count加一；
- **不变式**：元素个数count大于0，小于容量capacity；

```eiffel
class DICTIONARY [ELEMENT]                
    feature
put (x: ELEMENT; key: STRING) is                
         -- Insert x so that it will be retrievable               
         -- through key.              
require
count <= capacity
not key.empty
        ensure  
has (x)        
item (key) = x  
count = old count + 1
    end  
     ... Interface specifications of other features ...                  
invariant  
0 <= count  
          count <= capacity  
end
```

将其改写为C语言，如下所示：

```c
#include "stdio.h"
#include "assert.h"

/* 动态数组元素结构体（假设） */
typedef struct _element_t {
 int x;
 char* key;
} element_t;

bool_t put(darray_t* darray, int x, const char* key) {
 /* 先验条件(require) */
 assert(darray->size >= 0 && darray->size < darray->capacity);

 /* 不变式（invariant） */
 assert(darray->size >= 0 && darray->size < darray->capacity);

 /* put操作 */
 uint32_t old_count = darray->size;
 element_t elm;
 elm.x = x;
 elm.key = TKMEM_CALLOC(1, tk_strlen(key) + 1);
 tk_strcpy(elm.key, key);
 darray_push(darray, &elm);

 /* 后验条件（ensure） */
 element_t* elm_find = darray_find(darray, &elm);
 assert(elm_find != NULL && elm_find->x == x && tk_str_eq(elm_find->key, key) && darray->size == old_count + 1);

 /* 不变式（invariant） again */
 assert(darray->size >= 0 && darray->size < darray->capacity);
    
 return TRUE;
}
```

> 备注：
> 1. 以上代码调用了AWTK中的部分库函数，如有不懂请在评论提问。
> 2. AWTK的github仓库：[https://github.com/zlgopen/awtk](https://github.com/zlgopen/awtk)

## 5 总结

通过以上的学习，可以看出Dbc的本意其实很简单，就是在设计程序中加入断言。而所谓断言，实际就是必须为真的假设，只有这些假设为真，程序才可以做到正确无误。Dbc的主要断言就包括以上所讲的先验条件，后验证条件和不变式。

最后，在此感谢与我讨论交流的同事和抽空回答我问题的大学导师。
