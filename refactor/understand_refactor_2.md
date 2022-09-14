---
title: 从零开始了解重构（二）
date: 2022-06-14
categories:
 - 重构
tags:
 - C
 - 重构
---

## 1 前言

上篇笔记 [从零开始了解重构（一）](./understand_refactor_1.md) 主要介绍了何为重构、重构的准备工作、以及一些拆分函数的方法，比如提炼函数、查询取代临时变量、内联变量手法。本文紧接着上一篇笔记内容，继续以戏剧演出团计算收费的例子来了解重构。

## 2 通过示例了解重构

**示例程序的详细介绍以及代码详见**： [从零开始了解重构（一）](./understand_refactor_1.md) 。

上篇笔记中我们通过以下方法拆分了 statement() 函数，接下来我们继续这一过程，提高代码的可读性和扩展性。

- 提炼函数；
- 查询取代临时变量；
- 内联变量手法。

### 2.1 拆解statement()函数

#### 2.1.1 提炼计算观众量积分的逻辑

现在 statement() 函数的内部实现是这样的：

```c
str_t* statement(tk_object_t* invoice, tk_object_t* plays) {
  double_t total_amount = 0;
  uint32_t volume_credits = 0;
  str_t* result = TKMEM_ZALLOC(str_t);
  const char* customer = tk_object_get_prop_str(invoice, "[0].customer");
  tk_object_t* performances = tk_object_get_prop_object(invoice, "[0].performances");
  uint32_t perf_num = tk_object_get_prop_uint32(performances, TK_OBJECT_PROP_SIZE, 0);

  str_init(result, 0);
  str_append_format(result, STATEMENT_SIZE, "Statement for %s\n", customer);

  for (uint32_t i = 0; i < perf_num; i++) {
    value_t v;
    uint32_t audience = 0;
    tk_object_t* perf = NULL;
    const char* play_name = NULL;
    const char* play_type = NULL;

    object_array_get(performances, i, &v);
    perf = value_object(&v);
    audience = tk_object_get_prop_uint32(perf, "audience", 0);

    play_name = tk_object_get_prop_str(play_for_perf(perf, plays), "name");
    play_type = tk_object_get_prop_str(play_for_perf(perf, plays), "type");

    /* 增加观众量积分 */
    volume_credits += tk_max(audience - 30, 0);
    /* 每增加10名戏剧观众可以获得额外积分 */
    if (tk_str_eq(play_type, "comedy")) {
      volume_credits += floor(audience / 5);
    }

    total_amount += amount_for_perf(perf, plays);

    /* 格式化输出每个剧目的收费 */
    str_append_format(result, STATEMENT_SIZE, "  %s: $%0.2f (%d seats)\n", play_name,
                      amount_for_perf(perf, plays) / 100, audience);
  }

  /* 格式化输出总收费和获得的积分 */
  str_append_format(result, STATEMENT_SIZE, "Amount owed is $%.2f\n", total_amount / 100);
  str_append_format(result, STATEMENT_SIZE, "You earned %d credits\n", volume_credits);

  return result;
```

这里我们可以看到移除了 play 变量的好处，移除了一个局部作用域的变量，提炼观众量积分的计算逻辑又更简单一些。

这里我们仍然需要处理其他两个局部变量。 `perf` 和 `volume_credits` 同样可以作为参数传入，将它们的获取逻辑提炼到新函数中并修改新函数中的变量名，代码如下：

```c
static tk_object_t* getPerf(tk_object_t* performance_list, uint32_t index) {
  char str_i[4] = {0};
  return tk_object_get_prop_object(performance_list, tk_itoa(str_i, sizeof(str_i), index));
}
```

```c
static uint32_t volume_credits_perf(tk_object_t* a_performance, tk_object_t* plays) {
  uint32_t result = 0;
  result += tk_max(tk_object_get_prop_uint32(a_performance, "audience", 0) - 30, 0);
  if (tk_str_eq(tk_object_get_prop_str(play_for_perf(a_performance, plays), "type"), "comedy")) {
    result += floor(tk_object_get_prop_uint32(a_performance, "audience", 0) / 5);
  }
  return result;
}
```

```c
str_t* statement(tk_object_t* invoice, tk_object_t* plays) {
  double_t total_amount = 0;
  uint32_t volume_credits = 0;
  str_t* result = TKMEM_ZALLOC(str_t);
  const char* customer = tk_object_get_prop_str(invoice, "[0].customer");
  tk_object_t* performances = tk_object_get_prop_object(invoice, "[0].performances");
  uint32_t perf_num = tk_object_get_prop_uint32(performances, TK_OBJECT_PROP_SIZE, 0);

  str_init(result, 0);
  str_append_format(result, STATEMENT_SIZE, "Statement for %s\n", customer);

  for (uint32_t i = 0; i < perf_num; i++) {
    uint32_t audience = tk_object_get_prop_uint32(getPerf(performances, i), "audience", 0);
    const char* play_name = tk_object_get_prop_str(play_for_perf(getPerf(performances, i), plays), "name");

    volume_credits += volume_credits_perf(getPerf(performances, i), plays);
    total_amount += amount_for_perf(getPerf(performances, i), plays);

    /* 格式化输出每个剧目的收费 */
    str_append_format(result, STATEMENT_SIZE, "  %s: $%0.2f (%d seats)\n", play_name,
                      amount_for_perf(getPerf(performances, i), plays) / 100, audience);
  }

  /* 格式化输出总收费和获得的积分 */
  str_append_format(result, STATEMENT_SIZE, "Amount owed is $%.2f\n", total_amount / 100);
  str_append_format(result, STATEMENT_SIZE, "You earned %d credits\n", volume_credits);

  return result;
}
```

这里只展示一步到位的结果，在实际操作时，每编辑一个步骤都需要执行编译、测试、提交。

《重构》的作者指出，临时变量往往会带来麻烦，它们只在对其进行处理的代码块中有用，因此临时变量实质上是鼓励我们写又长又复杂的函数，代码往往变得晦涩难懂。因此，重构代码的过程中非常重视替换这些临时变量，有时甚至存在“将函数赋值给临时变量”的场景，比如书中 JS 代码的 format 变量，这里由于我是用 C 语言写的，所以没有这种情况。作者在重构时将他的 format 变量替换为一个明确声明的函数，并使用了**改变函数声明的手法**优化该函数，最终结果如下：

```js
function usd(aNumber) {
  return new Intl.NumberFormat("en-US", { style: "currency", currency: "USD", minimumFractionDigits: 2}).format(aNumber/100);
}
```

> 备注：在写代码的时候，好的命名非常重要。只有恰如其分地命名，才能彰显出将大函数分解成小函数的价值。有了好的名称，就不必通过阅读函数体来了解其行为。但要一次把名取好并不容易，这是一个不断优化的过程。通常我们需要花时间通读更多代码，才能发现最好的名称是什么。

#### 2.1.2 移除观众量积分总和

接着我们来重构 volume_credits，处理这个变量更加微妙，因为它是在循环的迭代过程中累加得到的。第一步，就是应用拆分循环手法将 volume_credits 的累加过程分离出来，代码如下：

```c
str_t* statement(tk_object_t* invoice, tk_object_t* plays) {
  double_t total_amount = 0;
  uint32_t volume_credits = 0;
  str_t* result = TKMEM_ZALLOC(str_t);
  const char* customer = tk_object_get_prop_str(invoice, "[0].customer");
  tk_object_t* performances = tk_object_get_prop_object(invoice, "[0].performances");
  uint32_t perf_num = tk_object_get_prop_uint32(performances, TK_OBJECT_PROP_SIZE, 0);

  str_init(result, 0);
  str_append_format(result, STATEMENT_SIZE, "Statement for %s\n", customer);

  for (uint32_t i = 0; i < perf_num; i++) {
    uint32_t audience = tk_object_get_prop_uint32(getPerf(performances, i), "audience", 0);
    const char* play_name =tk_object_get_prop_str(play_for_perf(getPerf(performances, i), plays), "name");

    total_amount += amount_for_perf(getPerf(performances, i), plays);
    /* 格式化输出每个剧目的收费 */
    str_append_format(result, STATEMENT_SIZE, "  %s: $%0.2f (%d seats)\n", play_name,
                      amount_for_perf(getPerf(performances, i), plays) / 100, audience);
  }

  for (uint32_t i = 0; i < perf_num; i++) {
    uint32_t audience = tk_object_get_prop_uint32(getPerf(performances, i), "audience", 0);
    volume_credits += volume_credits_perf(getPerf(performances, i), plays);
  }

  /* 格式化输出总收费和获得的积分 */
  str_append_format(result, STATEMENT_SIZE, "Amount owed is $%.2f\n", total_amount / 100);
  str_append_format(result, STATEMENT_SIZE, "You earned %d credits\n", volume_credits);

  return result;
}
```

完成这一步，我就可以使用移动语句手法将变量声明挪动到紧邻循环的位置，代码如下：

```c
str_t* statement(tk_object_t* invoice, tk_object_t* plays) {
  ...
  uint32_t volume_credits = 0;
  for (uint32_t i = 0; i < perf_num; i++) {
    uint32_t audience = tk_object_get_prop_uint32(getPerf(performances, i), "audience", 0);
    volume_credits += volume_credits_perf(getPerf(performances, i), plays);
  }
  ...
  return result;
}
```

把与更新 volume_credits  变量相关的代码都集中到一起，有利于以查询取代临时变量手法的施展。第一步同样是先对变量的计算过程应用提炼函数手法。

```c
static uint32_t total_volume_credits(tk_object_t* performance_list, tk_object_t* plays) {
  uint32_t result = 0;
  for (uint32_t i = 0; i < tk_object_get_prop_uint32(performance_list, TK_OBJECT_PROP_SIZE, 0); i++) {
    result += volume_credits_perf(getPerf(performance_list, i), plays);
  }
  return result;
}
```

```c
str_t* statement(tk_object_t* invoice, tk_object_t* plays) {
  double_t total_amount = 0;
  str_t* result = TKMEM_ZALLOC(str_t);
  const char* customer = tk_object_get_prop_str(invoice, "[0].customer");
  tk_object_t* performances = tk_object_get_prop_object(invoice, "[0].performances");
  uint32_t perf_num = tk_object_get_prop_uint32(performances, TK_OBJECT_PROP_SIZE, 0);

  str_init(result, 0);
  str_append_format(result, STATEMENT_SIZE, "Statement for %s\n", customer);

  for (uint32_t i = 0; i < perf_num; i++) {
    uint32_t audience = tk_object_get_prop_uint32(getPerf(performances, i), "audience", 0);
    const char* play_name = tk_object_get_prop_str(play_for_perf(getPerf(performances, i), plays), "name");

    total_amount += amount_for_perf(getPerf(performances, i), plays);
    /* 格式化输出每个剧目的收费 */
    str_append_format(result, STATEMENT_SIZE, "  %s: $%0.2f (%d seats)\n", play_name,
                      amount_for_perf(getPerf(performances, i), plays) / 100, audience);
  }

  uint32_t volume_credits = total_volume_credits(performances, plays);

  /* 格式化输出总收费和获得的积分 */
  str_append_format(result, STATEMENT_SIZE, "Amount owed is $%.2f\n", total_amount / 100);
  str_append_format(result, STATEMENT_SIZE, "You earned %d credits\n", volume_credits);

  return result;
}
```

完成函数提炼后，再应用内联变量手法内联 total_volume_credits 函数。

```c
str_t* statement(tk_object_t* invoice, tk_object_t* plays) {
  double_t total_amount = 0;
  str_t* result = TKMEM_ZALLOC(str_t);
  const char* customer = tk_object_get_prop_str(invoice, "[0].customer");
  tk_object_t* performances = tk_object_get_prop_object(invoice, "[0].performances");
  uint32_t perf_num = tk_object_get_prop_uint32(performances, TK_OBJECT_PROP_SIZE, 0);

  str_init(result, 0);
  str_append_format(result, STATEMENT_SIZE, "Statement for %s\n", customer);

  for (uint32_t i = 0; i < perf_num; i++) {
    uint32_t audience = tk_object_get_prop_uint32(getPerf(performances, i), "audience", 0);
    const char* play_name = tk_object_get_prop_str(play_for_perf(getPerf(performances, i), plays), "name");

    total_amount += amount_for_perf(getPerf(performances, i), plays);
    /* 格式化输出每个剧目的收费 */
    str_append_format(result, STATEMENT_SIZE, "  %s: $%0.2f (%d seats)\n", play_name,
                      amount_for_perf(getPerf(performances, i), plays) / 100, audience);
  }

  /* 格式化输出总收费和获得的积分 */
  str_append_format(result, STATEMENT_SIZE, "Amount owed is $%.2f\n", total_amount / 100);
  str_append_format(result, STATEMENT_SIZE, "You earned %d credits\n", total_volume_credits(performances, plays));

  return result;
}
```

#### 2.1.3 移除总收费

我们再以同样的手法移除 total_amount 变量，代码如下：

```c
static double_t total_amount(tk_object_t* performance_list, tk_object_t* plays) {
  double_t result = 0;
  for (uint32_t i = 0; i < tk_object_get_prop_uint32(performance_list, TK_OBJECT_PROP_SIZE, 0); i++) {
    result += amount_for_perf(getPerf(performance_list, i), plays);
  }
  return result;
}
```

```c
str_t* statement(tk_object_t* invoice, tk_object_t* plays) {
  str_t* result = TKMEM_ZALLOC(str_t);
  tk_object_t* performances = tk_object_get_prop_object(invoice, "[0].performances");

  str_init(result, 0);
  str_append_format(result, STATEMENT_SIZE, "Statement for %s\n", tk_object_get_prop_str(invoice, "[0].customer"));

  for (uint32_t i = 0; i < tk_object_get_prop_uint32(performances, TK_OBJECT_PROP_SIZE, 0); i++) {
    /* 格式化输出每个剧目的收费 */
    str_append_format(result, STATEMENT_SIZE, "  %s: $%0.2f (%d seats)\n", 
                      tk_object_get_prop_str(play_for_perf(getPerf(performances, i), plays), "name"),
                      amount_for_perf(getPerf(performances, i), plays) / 100, 
                      tk_object_get_prop_uint32(getPerf(performances, i), "audience", 0));
  }

  /* 格式化输出总收费和获得的积分 */
  str_append_format(result, STATEMENT_SIZE, "Amount owed is $%.2f\n",  total_amount(performances, plays) / 100);
  str_append_format(result, STATEMENT_SIZE, "You earned %d credits\n", total_volume_credits(performances, plays));

  return result;
}
```

这里顺便把一些多余的局部变量优化掉了。

#### 2.1.4 小结

针对以上的重构内容，我们可以明显的看到代码中重复的循环变多了，在重构时不免怀疑是否存在降低性能的问题，但大多数时候，重复一次这样的循环对性能的影响都可忽略不计。如果在重构前后进行计时，很可能甚至都注意不到运行速度的变化——通常也确实没什么变化。

> 备注：在聪明的编译器、现代的缓存技术面前，我们很多直觉都是不准确的。软件的性能通常只与代码的一小部分相关，改变其他的部分往往对总体性能贡献很小。

**当然，有时，一些重构手法也会显著地影响性能。但即便如此，我们也可以先不去管它，继续重构，因为有了一份结构良好的代码，回头调优其性能也容易得多。**

如果在重构时引入了明显的性能损耗，我们可以在后面花时间进行性能调优。进行调优时，可能会回退早先做的一些重构——但更多时候，**因为重构我们可以使用更高效的调优方案。最后得到的是既整洁又高效的代码。**

因此对于重构过程的性能问题，作者的总体的建议是：大多数情况下可以忽略它。如果重构引入了性能损耗，先完成重构，再做性能优化。

这里我们再来看一下代码的全貌：

```c
#include "awtk.h"
#include "tkc/str.h"
#include "tkc/mem.h"
#include "tkc/object_array.h"
#include "tkc/object_default.h"
#include "statement.h"

#define STATEMENT_SIZE 1024

static tk_object_t* play_for_perf(tk_object_t* a_performance, tk_object_t* plays) {
  const char* play_id = tk_object_get_prop_str(a_performance, "playID");
  return tk_object_get_prop_object(plays, play_id);
}

static double_t amount_for_perf(tk_object_t* a_performance, tk_object_t* plays) {
  double_t result = 0;
  uint32_t audience = tk_object_get_prop_uint32(a_performance, "audience", 0);
  const char* play_name = tk_object_get_prop_str(play_for_perf(a_performance, plays), "name");
  const char* play_type = tk_object_get_prop_str(play_for_perf(a_performance, plays), "type");

  if (tk_str_eq(play_type, "tragedy")) {
    result = 40000;
    if (audience > 30) {
      result += 1000 * (audience - 30);
    }
  } else if (tk_str_eq(play_type, "comedy")) {
    result = 30000;
    if (audience > 20) {
      result += 10000 + 500 * (audience - 20);
    }
    result += 300 * audience;
  } else {
    log_info("unknow type: %s", play_type);
  }

  return result;
}

static uint32_t volume_credits_perf(tk_object_t* a_performance, tk_object_t* plays) {
  uint32_t result = 0;
  result += tk_max(tk_object_get_prop_uint32(a_performance, "audience", 0) - 30, 0);
  if (tk_str_eq(tk_object_get_prop_str(play_for_perf(a_performance, plays), "type"), "comedy")) {
    result += floor(tk_object_get_prop_uint32(a_performance, "audience", 0) / 5);
  }
  return result;
}

static tk_object_t* getPerf(tk_object_t* performance_list, uint32_t index) {
  char str_i[4] = {0};
  return tk_object_get_prop_object(performance_list, tk_itoa(str_i, sizeof(str_i), index));
}

static uint32_t total_volume_credits(tk_object_t* performance_list, tk_object_t* plays) {
  uint32_t result = 0;
  for (uint32_t i = 0; i < tk_object_get_prop_uint32(performance_list, TK_OBJECT_PROP_SIZE, 0);
       i++) {
    result += volume_credits_perf(getPerf(performance_list, i), plays);
  }
  return result;
}

static double_t total_amount(tk_object_t* performance_list, tk_object_t* plays) {
  double_t result = 0;
  for (uint32_t i = 0; i < tk_object_get_prop_uint32(performance_list, TK_OBJECT_PROP_SIZE, 0);
       i++) {
    result += amount_for_perf(getPerf(performance_list, i), plays);
  }
  return result;
}

str_t* statement(tk_object_t* invoice, tk_object_t* plays) {
  str_t* result = TKMEM_ZALLOC(str_t);
  tk_object_t* performances = tk_object_get_prop_object(invoice, "[0].performances");

  str_init(result, 0);
  str_append_format(result, STATEMENT_SIZE, "Statement for %s\n", tk_object_get_prop_str(invoice, "[0].customer"));

  for (uint32_t i = 0; i < tk_object_get_prop_uint32(performances, TK_OBJECT_PROP_SIZE, 0); i++) {
    /* 格式化输出每个剧目的收费 */
    str_append_format(result, STATEMENT_SIZE, "  %s: $%0.2f (%d seats)\n", 
                      tk_object_get_prop_str(play_for_perf(getPerf(performances, i), plays), "name"),
                      amount_for_perf(getPerf(performances, i), plays) / 100, 
                      tk_object_get_prop_uint32(getPerf(performances, i), "audience", 0));
  }

  /* 格式化输出总收费和获得的积分 */
  str_append_format(result, STATEMENT_SIZE, "Amount owed is $%.2f\n",  total_amount(performances, plays) / 100);
  str_append_format(result, STATEMENT_SIZE, "You earned %d credits\n", total_volume_credits(performances, plays));

  return result;
}
```

现在代码结构已经好多了。顶层的 statement() 函数现在只负责打印详单相关的逻辑。计算相关的逻辑都从主函数中移走，改由一组函数来支持。每个单独的计算过程和详单的整体结构变得更加容易理解。

### 2.2 拆分计算阶段和格式化阶段

到目前为止，我们的重构主要是改善了原函数的结构，以便更好地理解它，看清它的逻辑结构。这也是重构早期的一般步骤，把复杂的代码块分解为更小的单元，并且改善这些单元的命名。现在，我们可以更多关注功能部分的完善了。

此处，我们为这个示例程序添加一个HTML版本的清单。经过初步重构的代码改起来会更简单。因为计算代码已经被分离出来，而我们只需要为 statement() 函数实现一个HTML的版本。

这时我们就遇到了第二个问题，这些分解出来的函数嵌套在打印文本详单的函数中。无论嵌套函数组织得多么良好，每次复制粘贴一堆代码也是很烦的事情。如果同样的计算函数可以被文本版详单和HTML版详单共用就更好了，即所谓的代码复用。

常用的手段是**拆分阶段**，针对本文的例子，可以把目标逻辑分为两个阶段：

1. 阶段一：计算账单所需的数据，将其保存为一个中转数据结构，并传递给阶段二。
2. 阶段二：接受中转数据结构，并将它渲染成文本或 HTML。

#### 2.2.1 提炼渲染函数

这里先将阶段二拆分出来，在本文的案例中，实际上就是打印账单的代码，也就是 `statemant()` 函数的全部内容，我们将其抽取出来，并命名为 `render_plain_text()`。

```c
static str_t* render_plain_text(tk_object_t* invoice, tk_object_t* plays) {
  str_t* result = TKMEM_ZALLOC(str_t);
  tk_object_t* performances = tk_object_get_prop_object(invoice, "[0].performances");

  str_init(result, 0);
  str_append_format(result, STATEMENT_SIZE, "Statement for %s\n", tk_object_get_prop_str(invoice, "[0].customer"));

  for (uint32_t i = 0; i < tk_object_get_prop_uint32(performances, TK_OBJECT_PROP_SIZE, 0); i++) {
    /* 格式化输出每个剧目的收费 */
    str_append_format(result, STATEMENT_SIZE, "  %s: $%0.2f (%d seats)\n", 
                      tk_object_get_prop_str(play_for_perf(getPerf(performances, i), plays), "name"),
                      amount_for_perf(getPerf(performances, i), plays) / 100, 
                      tk_object_get_prop_uint32(getPerf(performances, i), "audience", 0));
  }

  /* 格式化输出总收费和获得的积分 */
  str_append_format(result, STATEMENT_SIZE, "Amount owed is $%.2f\n",  total_amount(performances, plays) / 100);
  str_append_format(result, STATEMENT_SIZE, "You earned %d credits\n", total_volume_credits(performances, plays));

  return result;
}

str_t* statement(tk_object_t* invoice, tk_object_t* plays) {
  return render_plain_text(invoice, plays);
}
```

接着创建阶段一中的中转数据结构对象，将 `render_plain_text()` 中用到的参数挪到这个中转数据结构中，把它作为参数传递给 `render_plain_text()` 函数：

```c
...
struct _statement_data_t;
typedef struct _statement_data_t statement_data_t;

typedef tk_object_t* (*data_get_play_t)(statement_data_t* data, uint32_t i);
typedef double_t (*data_get_amount_t)(statement_data_t* data, uint32_t i);

struct _statement_data_t {
  double_t amount;
  tk_object_t* plays;
  const char* customer;
  uint32_t volume_credits;
  tk_object_t* performances;

  data_get_play_t data_get_play;
  data_get_amount_t data_get_amount;
};

static tk_object_t* data_get_play(statement_data_t* data, uint32_t i);
static double_t data_get_amount(statement_data_t* data, uint32_t i);
...
static str_t* render_plain_text(statement_data_t* data) {
  str_t* result = TKMEM_ZALLOC(str_t);
  str_init(result, 0);
  str_append_format(result, STATEMENT_SIZE, "Statement for %s\n", data->customer);

  for (uint32_t i = 0; i < tk_object_get_prop_uint32(data->performances, TK_OBJECT_PROP_SIZE, 0); i++) {
    /* 格式化输出每个剧目的收费 */
    str_append_format(result, STATEMENT_SIZE, "  %s: $%0.2f (%d seats)\n",
                      tk_object_get_prop_str(data->data_get_play(data, i), "name"),
                      data->data_get_amount(data, i) / 100,
                      tk_object_get_prop_uint32(getPerf(data->performances, i), "audience", 0));
  }

  /* 格式化输出总收费和获得的积分 */
  str_append_format(result, STATEMENT_SIZE, "Amount owed is $%.2f\n", data->amount / 100);
  str_append_format(result, STATEMENT_SIZE, "You earned %d credits\n", data->volume_credits);

  return result;
}

str_t* statement(tk_object_t* invoice, tk_object_t* plays) {
  statement_data_t data;

  data.plays = plays;
  data.customer = tk_object_get_prop_str(invoice, "[0].customer");
  data.performances = tk_object_get_prop_object(invoice, "[0].performances");

  data.data_get_play = data_get_play;
  data.data_get_amount = data_get_amount;

  data.amount = total_amount(data.performances, plays);
  data.volume_credits = total_volume_credits(data.performances, plays);
  return render_plain_text(&data);
}

static tk_object_t* data_get_play(statement_data_t* data, uint32_t i) {
  return play_for_perf(getPerf(data->performances, i), data->plays);
}

static double_t data_get_amount(statement_data_t* data, uint32_t i) {
  return amount_for_perf(getPerf(data->performances, i), data->plays);
}
```

这里再把 `statement()` 函数中创建中转数据结构的过程拆分一下：

```c
static statement_data_t* create_statement_data(tk_object_t* invoice, tk_object_t* plays) {
  statement_data_t* data = TKMEM_ZALLOC(statement_data_t);

  data->plays = plays;
  data->customer = tk_object_get_prop_str(invoice, "[0].customer");
  data->performances = tk_object_get_prop_object(invoice, "[0].performances");

  data->data_get_play = data_get_play;
  data->data_get_amount = data_get_amount;

  data->amount = total_amount(data->performances, plays);
  data->volume_credits = total_volume_credits(data->performances, plays);
  return data;
}

str_t* statement(tk_object_t* invoice, tk_object_t* plays) {
  statement_data_t* data = create_statement_data(invoice, plays);
  str_t* result = render_plain_text(data);
  TKMEM_FREE(data);
  return result;
}
```

到此为止，阶段一和阶段二的过程已经彻底分离开了，此时，编写一个渲染 HTML 版本的账单函数就很简单了：

```c
static str_t* render_html(statement_data_t* data) {
  str_t* result = TKMEM_ZALLOC(str_t);
  str_init(result, 0);
  str_append_format(result, STATEMENT_SIZE, "<h1>Statement for %s</h1>\n", data->customer);
  str_append(result, "<table>\n");
  str_append(result, "<tr><th>play</th><th>seats</th><th>cost</th></tr>");

  for (uint32_t i = 0; i < tk_object_get_prop_uint32(data->performances, TK_OBJECT_PROP_SIZE, 0); i++) {
    /* 格式化输出每个剧目的收费 */
    str_append_format(result, STATEMENT_SIZE, " <tr><td>%s</td><td>%d</td>",
                      tk_object_get_prop_str(data->data_get_play(data, i), "name"),
                      tk_object_get_prop_uint32(getPerf(data->performances, i), "audience", 0));
    str_append_format(result, STATEMENT_SIZE, "<td>$%0.2f</td></tr>\n", data->data_get_amount(data, i) / 100);
  }

  /* 格式化输出总收费和获得的积分 */
  str_append(result, "<table>\n");
  str_append_format(result, STATEMENT_SIZE, "<p>Amount owed is <em>$%.2f</em></p>\n", data->amount / 100);
  str_append_format(result, STATEMENT_SIZE, "<p>You earned <em>%d</em> credits</p>\n", data->volume_credits);

  return result;
}
...
str_t* html_statement(tk_object_t* invoice, tk_object_t* plays) {
  statement_data_t* data = create_statement_data(invoice, plays);
  str_t* result = render_html(data);
  TKMEM_FREE(data);
  return result;
}
```

在网页中渲染出来的效果如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/499852fbd4b344369204239c514e30b3.png)

### 2.3 将代码拆分到多个文件

根据现有的代码，statement_data_t 实际上是一个相对独立的类型，因此这里将它拆分到单独的头文件和源文件中，代码如下：

```c
/* statement_data.h */
#ifndef STATEMENT_DATA_H
#define STATEMENT_DATA_H

#include "tkc/str.h"
#include "tkc/types_def.h"

BEGIN_C_DECLS

struct _statement_data_t;
typedef struct _statement_data_t statement_data_t;

typedef tk_object_t* (*data_get_perf_t)(statement_data_t* data, uint32_t i);
typedef tk_object_t* (*data_get_play_t)(statement_data_t* data, uint32_t i);
typedef double_t (*data_get_amount_t)(statement_data_t* data, uint32_t i);

struct _statement_data_t {
  double_t amount;
  tk_object_t* plays;
  const char* customer;
  uint32_t volume_credits;
  tk_object_t* performances;

  data_get_perf_t data_get_perf;
  data_get_play_t data_get_play;
  data_get_amount_t data_get_amount;
};

statement_data_t* create_statement_data(tk_object_t* invoice, tk_object_t* plays);

END_C_DECLS

#endif /*STATEMENT_DATA_H*/
```

```c
/* statement_data.c */
#include "awtk.h"
#include "tkc/mem.h"
#include "statement_data.h"

static tk_object_t* play_for_perf(tk_object_t* a_performance, tk_object_t* plays) {
  const char* play_id = tk_object_get_prop_str(a_performance, "playID");
  return tk_object_get_prop_object(plays, play_id);
}

static double_t amount_for_perf(tk_object_t* a_performance, tk_object_t* plays) {
  double_t result = 0;
  uint32_t audience = tk_object_get_prop_uint32(a_performance, "audience", 0);
  const char* play_name = tk_object_get_prop_str(play_for_perf(a_performance, plays), "name");
  const char* play_type = tk_object_get_prop_str(play_for_perf(a_performance, plays), "type");

  if (tk_str_eq(play_type, "tragedy")) {
    result = 40000;
    if (audience > 30) {
      result += 1000 * (audience - 30);
    }
  } else if (tk_str_eq(play_type, "comedy")) {
    result = 30000;
    if (audience > 20) {
      result += 10000 + 500 * (audience - 20);
    }
    result += 300 * audience;
  } else {
    log_info("unknow type: %s", play_type);
  }

  return result;
}

static uint32_t volume_credits_perf(tk_object_t* a_performance, tk_object_t* plays) {
  uint32_t result = 0;
  result += tk_max(tk_object_get_prop_uint32(a_performance, "audience", 0) - 30, 0);
  if (tk_str_eq(tk_object_get_prop_str(play_for_perf(a_performance, plays), "type"), "comedy")) {
    result += floor(tk_object_get_prop_uint32(a_performance, "audience", 0) / 5);
  }
  return result;
}

static uint32_t total_volume_credits(statement_data_t* data, tk_object_t* plays) {
  uint32_t result = 0;
  uint32_t perf_size = tk_object_get_prop_uint32(data->performances, TK_OBJECT_PROP_SIZE, 0);
  for (uint32_t i = 0; i < perf_size; i++) {
    result += volume_credits_perf(data->data_get_perf(data, i), plays);
  }
  return result;
}

static double_t total_amount(statement_data_t* data, tk_object_t* plays) {
  double_t result = 0;
  uint32_t perf_size = tk_object_get_prop_uint32(data->performances, TK_OBJECT_PROP_SIZE, 0);
  for (uint32_t i = 0; i < perf_size; i++) {
    result += amount_for_perf(data->data_get_perf(data, i), plays);
  }
  return result;
}

static tk_object_t* data_get_perf(statement_data_t* data, uint32_t i) {
  char str_i[4] = {0};
  return tk_object_get_prop_object(data->performances, tk_itoa(str_i, sizeof(str_i), i));
}

static tk_object_t* data_get_play(statement_data_t* data, uint32_t i) {
  return play_for_perf(data->data_get_perf(data, i), data->plays);
}

static double_t data_get_amount(statement_data_t* data, uint32_t i) {
  return amount_for_perf(data->data_get_perf(data, i), data->plays);
}

statement_data_t* create_statement_data(tk_object_t* invoice, tk_object_t* plays) {
  statement_data_t* data = TKMEM_ZALLOC(statement_data_t);

  data->plays = plays;
  data->customer = tk_object_get_prop_str(invoice, "[0].customer");
  data->performances = tk_object_get_prop_object(invoice, "[0].performances");

  data->data_get_perf = data_get_perf;
  data->data_get_play = data_get_play;
  data->data_get_amount = data_get_amount;

  data->amount = total_amount(data, plays);
  data->volume_credits = total_volume_credits(data, plays);
  return data;
}
```

这样 `statement` 的相关代码就非常简洁了：

```c
/* statement.h */
#ifndef STATEMENT_H
#define STATEMENT_H

#include "tkc/str.h"
#include "tkc/types_def.h"

BEGIN_C_DECLS

str_t* statement(tk_object_t* invoice, tk_object_t* plays);

str_t* html_statement(tk_object_t* invoice, tk_object_t* plays);

END_C_DECLS

#endif /*STATEMENT_H*/
```

```c
/* statement.c */
#include "awtk.h"
#include "tkc/str.h"
#include "tkc/mem.h"
#include "statement.h"
#include "statement_data.h"

#define STATEMENT_SIZE 1024

static str_t* render_plain_text(statement_data_t* data) {
  str_t* result = TKMEM_ZALLOC(str_t);
  uint32_t perf_size = tk_object_get_prop_uint32(data->performances, TK_OBJECT_PROP_SIZE, 0);
  str_init(result, 0);
  str_append_format(result, STATEMENT_SIZE, "Statement for %s\n", data->customer);

  for (uint32_t i = 0; i < perf_size; i++) {
    /* 格式化输出每个剧目的收费 */
    str_append_format(result, STATEMENT_SIZE, "  %s: $%0.2f (%d seats)\n",
                      tk_object_get_prop_str(data->data_get_play(data, i), "name"),
                      data->data_get_amount(data, i) / 100,
                      tk_object_get_prop_uint32(data->data_get_perf(data, i), "audience", 0));
  }

  /* 格式化输出总收费和获得的积分 */
  str_append_format(result, STATEMENT_SIZE, "Amount owed is $%.2f\n", data->amount / 100);
  str_append_format(result, STATEMENT_SIZE, "You earned %d credits\n", data->volume_credits);

  return result;
}

static str_t* render_html(statement_data_t* data) {
  str_t* result = TKMEM_ZALLOC(str_t);
  uint32_t perf_size = tk_object_get_prop_uint32(data->performances, TK_OBJECT_PROP_SIZE, 0);
  str_init(result, 0);
  str_append_format(result, STATEMENT_SIZE, "<h1>Statement for %s</h1>\n", data->customer);
  str_append(result, "<table>\n");
  str_append(result, "<tr><th>play</th><th>seats</th><th>cost</th></tr>");

  for (uint32_t i = 0; i < perf_size; i++) {
    /* 格式化输出每个剧目的收费 */
    str_append_format(result, STATEMENT_SIZE, " <tr><td>%s</td><td>%d</td>",
                      tk_object_get_prop_str(data->data_get_play(data, i), "name"),
                      tk_object_get_prop_uint32(data->data_get_perf(data, i), "audience", 0));
    str_append_format(result, STATEMENT_SIZE, "<td>$%0.2f</td></tr>\n",
                      data->data_get_amount(data, i) / 100);
  }

  /* 格式化输出总收费和获得的积分 */
  str_append(result, "<table>\n");
  str_append_format(result, STATEMENT_SIZE, "<p>Amount owed is <em>$%.2f</em></p>\n",
                    data->amount / 100);
  str_append_format(result, STATEMENT_SIZE, "<p>You earned <em>%d</em> credits</p>\n",
                    data->volume_credits);

  return result;
}

str_t* html_statement(tk_object_t* invoice, tk_object_t* plays) {
  statement_data_t* data = create_statement_data(invoice, plays);
  str_t* result = render_html(data);
  TKMEM_FREE(data);
  return result;
}

str_t* statement(tk_object_t* invoice, tk_object_t* plays) {
  statement_data_t* data = create_statement_data(invoice, plays);
  str_t* result = render_plain_text(data);
  TKMEM_FREE(data);
  return result;
}
```

### 2.4 按类型重组计算过程

#### 2.4.1 多态取代条件表达式

接下来，我们来让代码支持更多类型的戏剧（支持价格计算和观众积分计算）。

针对当前的代码，我们需要修改 `amount_for_perf()` 函数，向函数中添加新的分支来支持更多类型的戏剧，但随着迭代，这样的分支逻辑很容易变得臃肿难看。

我们可以使用多态来解决这个问题，戏剧为父类，喜剧(comedy)和悲剧(tragedy)分别作为两个子类，包含独立的计算逻辑，外部只需调用一个多态的 `amount()` 函数就可以跳转到子类的计算。

该方法被作者称为"**多态取代条件表达式**"，此处通过这个方法来重构 `amount_for_perf()` 函数 和 `volume_credits_perf()`函数中的条件分支。

#### 2.4.2 构造戏剧基类

创建戏剧基类，并将价格计算函数和观众积分计算函数放进去。

> 备注：在上述的重构过程中，我们构造了 statement_data_t 结构，这时我们只需要保证接下来的重构过程中，该结构的数据不会改变即可（可以为其添加一些测试代码）。

由于这个基类主要负责计算戏剧的相关数据，因此命名为 `perf_calc`，代码如下：

```c
/* perf_calc.h */
#ifndef PERF_CALC_H
#define PERF_CALC_H

#include "tkc/types_def.h"

BEGIN_C_DECLS

struct _perf_calc_t;
typedef struct _perf_calc_t perf_calc_t;

typedef double_t (*perf_calc_get_amount_t)(perf_calc_t* calc);
typedef uint32_t (*perf_calc_get_volume_credits_t)(perf_calc_t* calc);

struct _perf_calc_t {
  tk_object_t* a_play;
  tk_object_t* a_performance;

  perf_calc_get_amount_t perf_calc_get_amount;
  perf_calc_get_volume_credits_t perf_calc_get_volume_credits;
};

perf_calc_t* perf_calc_create(tk_object_t* a_performance, tk_object_t* a_play);
double_t perf_calc_get_amount(perf_calc_t* calc);
uint32_t perf_calc_get_volume_credits(perf_calc_t* calc);

END_C_DECLS

#endif /*PERF_CALC_H*/
```

```c
/* perf_calc.c */
#include "awtk.h"
#include "tkc/mem.h"
#include "perf_calc.h"
#include "perf_calc_comedy.h"
#include "perf_calc_tragedy.h"

double_t perf_calc_get_amount(perf_calc_t* calc) {
  return_value_if_fail(calc->perf_calc_get_amount != NULL, -1);
  return calc->perf_calc_get_amount(calc);
}

uint32_t perf_calc_get_volume_credits(perf_calc_t* calc) {
  if (calc->perf_calc_get_volume_credits != NULL) {
    return calc->perf_calc_get_volume_credits(calc);
  }
  return tk_max(tk_object_get_prop_uint32(calc->a_performance, "audience", 0) - 30, 0);
}

perf_calc_t* perf_calc_create(tk_object_t* a_performance, tk_object_t* a_play) {
  perf_calc_t* result = NULL;
  const char* play_type = tk_object_get_prop_str(a_play, "type");

  if (tk_str_eq(play_type, "tragedy")) {
    result = create_perf_calc_tragedy(a_performance, a_play);
  } else if (tk_str_eq(play_type, "comedy")) {
    result = create_perf_calc_comedy(a_performance, a_play);
  } else {
    log_info("unknow type: %s", play_type);
  }

  return result;
}
```

#### 2.4.3 构造喜剧子类

构建喜剧子类 `perf_calc_comedy`，代码如下：

```c
/* perf_calc_comedy.h */
#ifndef PERF_CALC_COMEDY_H
#define PERF_CALC_COMEDY_H

#include "perf_calc.h"
#include "tkc/types_def.h"

BEGIN_C_DECLS

perf_calc_t* create_perf_calc_comedy(tk_object_t* a_performance, tk_object_t* a_play);

END_C_DECLS

#endif /*PERF_CALC_COMEDY_H*/
```

```c
/* perf_calc_comedy.c */
#include "awtk.h"
#include "tkc/mem.h"
#include "perf_calc.h"

static double_t perf_calc_comedy_get_amount(perf_calc_t* calc) {
  double_t result = 0;
  uint32_t audience = tk_object_get_prop_uint32(calc->a_performance, "audience", 0);
  const char* play_type = tk_object_get_prop_str(calc->a_play, "type");
  return_value_if_fail(tk_str_eq(play_type, "comedy"), -1);

  result = 30000;
  if (audience > 20) {
    result += 10000 + 500 * (audience - 20);
  }
  result += 300 * audience;

  return result;
}

static uint32_t perf_calc_comedy_get_volume_credits(perf_calc_t* calc) {
  uint32_t result = 0;
  const char* play_type = tk_object_get_prop_str(calc->a_play, "type");
  return_value_if_fail(tk_str_eq(play_type, "comedy"), -1);

  result = tk_max(tk_object_get_prop_uint32(calc->a_performance, "audience", 0) - 30, 0);
  result += floor(tk_object_get_prop_uint32(calc->a_performance, "audience", 0) / 5);

  return result;
}

perf_calc_t* create_perf_calc_comedy(tk_object_t* a_performance, tk_object_t* a_play) {
  perf_calc_t* result = TKMEM_ZALLOC(perf_calc_t);

  result->a_play = a_play;
  result->a_performance = a_performance;

  result->perf_calc_get_amount = perf_calc_comedy_get_amount;
  result->perf_calc_get_volume_credits = perf_calc_comedy_get_volume_credits;

  return result;
}
```

#### 2.4.4 构造悲剧子类

构建悲剧子类 `perf_calc_tragedy`，代码如下：

```c
/* perf_calc_tragedy.h */
#ifndef PERF_CALC_TRAGEDY_H
#define PERF_CALC_TRAGEDY_H

#include "perf_calc.h"
#include "tkc/types_def.h"

BEGIN_C_DECLS

perf_calc_t* create_perf_calc_tragedy(tk_object_t* a_performance, tk_object_t* a_play);

END_C_DECLS

#endif /*PERF_CALC_TRAGEDY_H*/
```

```c
/* perf_calc_tragedy.c */
#include "awtk.h"
#include "tkc/mem.h"
#include "perf_calc.h"

static double_t perf_calc_tragedy_get_amount(perf_calc_t* calc) {
  double_t result = 0;
  uint32_t audience = tk_object_get_prop_uint32(calc->a_performance, "audience", 0);
  const char* play_type = tk_object_get_prop_str(calc->a_play, "type");
  return_value_if_fail(tk_str_eq(play_type, "tragedy"), -1);

  result = 40000;
  if (audience > 30) {
    result += 1000 * (audience - 30);
  }

  return result;
}

perf_calc_t* create_perf_calc_tragedy(tk_object_t* a_performance, tk_object_t* a_play) {
  perf_calc_t* result = TKMEM_ZALLOC(perf_calc_t);

  result->a_play = a_play;
  result->a_performance = a_performance;

  result->perf_calc_get_amount = perf_calc_tragedy_get_amount;

  return result;
}
```

### 2.5 使用多态计算器来提供数据

构造好了戏剧基类及其相关子类后，我们在 statement_data.c 中调用基类 `perf_calc` 的代码来计算戏剧的相关数据，如下：

```c
/* statement_data.c */
static double_t amount_for_perf(tk_object_t* a_performance, tk_object_t* plays) {
  double_t result = 0;
  perf_calc_t* calc = perf_calc_create(a_performance, play_for_perf(a_performance, plays));
  result = perf_calc_get_amount(calc);
  TKMEM_FREE(calc);
  return result;
}

static uint32_t volume_credits_perf(tk_object_t* a_performance, tk_object_t* plays) {
  uint32_t result = 0;
  perf_calc_t* calc = perf_calc_create(a_performance, play_for_perf(a_performance, plays));
  result = perf_calc_get_volume_credits(calc);
  TKMEM_FREE(calc);
  return result;
}
```

经过重构后，这时再要添加新戏剧类型，只需要添加 `perf_calc` 的子类，并在构造函数中返回子类对象即可。

本质上这个多态重构的过程，就是把 `amount_for_perf()` 函数 和 `volume_credits_perf()` 中条件分支语句搬到了集中的构造函数 `perf_calc_create()`中，**有越多的函数依赖于同一个条件分支，那么多态继承的方法的效果就越好**。

## 3 总结

跟随《重构：改善既有代码的设计》的作者，对这个简单的例子进行分析，让我逐渐有了点重构的感觉，这个例子虽然简单，但包含了多种重构方法，比如：提炼函数、内联变量、多态取代条件变量等等。

重构早期的主要动力其实就是理解代码，首选需要通读代码，有一定理解后，再通过重构将这些代码逐步理顺。在这个过程中，我们会得到更清晰的代码，这样就比较容易得发现更深层次上的设计问题，让我们后续的重构更顺利，二者相互形成正向反馈。

在阅读《重构：改善既有代码的设计》时，我也觉得作者在重构过程中的扣的细节太小了，操作繁琐又重复，但正是这些细节的积累，让代码逐渐变好，最后发生质变，但好代码的评判标准本来就不是刻板唯一的，在学习该书的时候，得时刻提醒自己，着重学习的应该是重构的思想，通过各种重构手法最终形成：**易读的、易理解的、易维护的、易扩展的、高效率的、运行良好且稳定的代码**。

当然，在重构过程中我们也不一定程序在各方面都能达到最好，可能在某些情况下，为了方便程序后期维护扩展，可能在一定程度上需要牺牲效率，或者反过来，但如何从中取得平衡，做出取舍，也正是我们需要不断学习实践的地方。
