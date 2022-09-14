---
title: 状态机基本概念以及使用状态机实现单词计数
date: 2020-10-22
categories:
 - 算法
tags:
 - C
---

## 1 前言

在本周的培训内容中，接触到了“状态机”一词，这是什么意思？用来做什么？怎么做？以下记录了初识状态机的学习感悟，并使用状态机原理实现了简单的单词计数实例。

## 2 什么是状态机？

根据查阅到的资料总结，状态机是一个有向图形，又可称状态转移图，由一组节点和一组相应的转移函数组成。

举一个简单的例子：人有三个状态（节点）：健康、感冒、康复中。触发的条件（转移函数）有淋雨（t1）、吃药（t2）、打针（t3）、休息（t4），所以状态机就是在不同的条件下跳转到自己或不同状态的图。

```bash
健康   - (t3) -> 健康
健康   - (t1) -> 感冒
感冒   - (t3) -> 健康
感冒   - (t2) -> 康复中
康复中 - (t4) -> 健康
```

使用状态机编写程序时，常用到4个概念：

1. 状态（State）：一个状态机至少包括两个状态，如上面例子，有健康、感冒、康复中三个状态；
2. 事件（Event）：事件就是执行某个操作的触发条件或者口令。对于“淋雨”就是一个事件。
3. 动作（Action）：事件发生后要执行动作，动作执行完毕后，可以迁移到新的状态，也可以仍旧保持原状态。动作不是必需的，当事件发生后，也可以不执行任何动作，直接迁移到新状态。编程时，一个Action一般对应一个函数。
4. 转换（Transition）：也就是从一个状态变化为另一个状态，例如从健康到感冒就是一个变换。

## 3 状态机用来做什么？

状态机是一个对真实世界的抽象，而且是逻辑严谨的数学抽象，所以非常时候用在数学领域。可以应用到各个层面上，例如硬件设计、编译器设计、以及编程实现各种具体业务逻辑，如下面的使用状态机实现单词计数，从单词计数可扩展至解析xml文件等等。

举一些生活中常见的例子：电灯的开关逻辑、自动贩卖机的售卖货物的逻辑、空调的控制逻辑等都可以抽象为状态机。

## 4 使用状态机实现单词计数

问题描述：一个字符串由多个单词组成，这些单词由空格、逗号、句点、换行符等多种符号隔开，请写一个程序统计输入的字符串中有多少个单词。

构建状态机模型：

1. 字符类型：英文字符、符号；
2. 字符状态：起始状态、单词状态、符号状态、结束状态；
3. 状态转换规则：

   （1）起始状态下读到非符号，进入单词状态；
   （2）单词状态下读到符号，进入符号状态；
   （3）符号状态下读到非符号，进入单词状态；
   （4）在起始状态、单词状态、符号状态下读到‘ \0’，进入结束状态。

4. 动作：每次进入单词状态，单词计数加1。

实现代码如下：

```c
/**
 * 判断字符是否为符号。
 */
#define IS_SIGN(c) ((c) == ' ' || (c) == ',' || (c) == '.' || (c) == '\n' || (c) == '?' || (c) == '!')

/**
 * @enum char_state_t
 * 字符在字符串中的状态。
 */
typedef enum _char_state_t {
  /**
   * @const STATE_INIT
   * 起始状态。
   */
  STATE_START = 0,
  /**
   * @const STATE_WORD
   * 单词状态。
   */
  STATE_WORD,
  /**
   * @const STATE_SIGN
   * 符号状态。
   */
  STATE_SIGN,
  /**
   * @const STATE_END
   * 结束状态。
   */
  STATE_END
} char_state_t;
 
/**
 * @method count_word
 * 使用状态机计算字符串中单词个数。
 * 
 * @param {const char*} text 需要计算的字符串。
 * 
 * @return {int32_t} 单词个数（text为NULL时返回-1）。
 */
int32_t count_word(const char* text) {
  if(text == NULL) return -1;
  int32_t count = 0;
  char_state_t state = STATE_START;
  char curr = '\0';
  while (curr = *text++) {
    switch (state) {
      case STATE_START: /* 起始状态 */
        if (IS_SIGN(curr)) {
          state = STATE_SIGN;
        } else {
          state = STATE_WORD;
          count++;
        }
        break;
      case STATE_WORD: /* 单词状态 */
        if (IS_SIGN(curr)) {
          state = STATE_SIGN;
        }
        break;
      case STATE_SIGN: /* 符号状态 */
        if (!IS_SIGN(curr)) {
          state = STATE_WORD;
          count++;
        }
        break;
      default:
        break;
    }
  }
  state = STATE_END;
  return count;
}

int main(int argc, char* argv[]) {
  int ret = 0;
  if (argc == 2) {
    ret = count_word_state(argv[1]);
    printf("Word Number = %d\n", ret);
  } else {
    printf("Usage: [\"string\"]\n");
    ret = 0;
  }
  return ret;
}
```

## 5 总结

状态机是一个数学模型，可理解为有向图，通常体现为一个状态转换图，涉及到状态（State）、事件（Event）、动作（Action）、转换（Transition）。状态机是计算机科学的重要基础概念之一，也可以说是一种总结归纳问题的思想，应用非常广泛。
