---
title: 设计模式学习（四）：基于Builder模式的歌词解析器
date: 2021-02-21
categories:
 - 设计模式
tags:
 - C
 - 设计模式
 - AWTK
---

## 1 前言
上篇文章（[设计模式学习（三）：生成器（Builder）模式](./builder.md)）记录了 Builder 模式的具体内容，这次使用C语言来实现一个实际的例子——基于Builder模式的歌词解析器。

> 本文的示例来自李先静老师基于AWTK实现的多媒体播放器项目（awtk-media-player），可前往  [GitHub开源仓库](https://github.com/zlgopen/awtk-media-player)  下载。

## 2 示例介绍

歌词文件（.lrc）是一种文本文件，用来描述歌曲的歌词。在该文件的帮助下，音乐播放器可以根据相应时间同步显示歌词。歌词文件由时间标签、ID标签和歌词组成。

- 时间标签，例如：[00:23.25]
- ID标签，例如：[ar:谭咏麟]
- 歌词，例如：凄雨冷风中  多少繁华如梦

下面是谭咏麟先生的歌曲 *水中花* 歌词的截取部分：

```bash
[ti:水中花]
[ar:谭咏麟]
[al:心手相连]
[by:孟德良]
[00:00.00]《水中花》
[00:02.00]演唱：谭咏麟
[00:04.00]作词：娃娃
[00:05.50]作曲：简宁
[00:07.00]
[00:09.03]凄雨冷风中  多少繁华如梦
[00:15.25]曾经万紫千红  随风吹落
[00:23.25]蓦然回首中  欢爱宛如烟云
[00:29.57]似水年华流走  不留影踪
[00:36.30]
[00:37.18]我看见  水中的花朵
[00:40.31]强要留住一抹红
[00:44.50]奈何辗转在风尘
[00:48.07]不再有往日颜色
[00:50.84]
......
......
[02:46.85]感怀飘零的花朵
[02:50.01]城市中无从寄托
[02:54.10]任那雨打风吹  也沉默
[02:57.90]仿佛是我
[03:00.12]
[03:01.72]啦…啦…啦…啦…
[03:16.09]啦…啦…啦…啦…
```

## 3 示例代码

以下歌词解析器的代码参考了李先静老师基于AWTK实现的多媒体播放器项目（awtk-media-player），可前往 [GitHub开源仓库](https://github.com/zlgopen/awtk-media-player) 下载，其中歌词解析器位于awtk-media-player\src\media_player\lrc目录中。

### 3.1 Builder接口

```c
/* lrc_builder.h */
#ifndef TK_LRC_BUILDER_H
#define TK_LRC_BUILDER_H

#include "tkc/types_def.h"

BEGIN_C_DECLS

struct _lrc_builder_t;
typedef struct _lrc_builder_t lrc_builder_t;

typedef ret_t (*lrc_builder_on_id_tag_t)(lrc_builder_t* builder, const char* key,
                                         const char* value);
typedef ret_t (*lrc_builder_on_time_tag_t)(lrc_builder_t* builder, uint32_t start_time);
typedef ret_t (*lrc_builder_on_text_t)(lrc_builder_t* builder, const char* text);
typedef ret_t (*lrc_builder_on_error_t)(lrc_builder_t* builder, const char* error);
typedef ret_t (*lrc_builder_destroy_t)(lrc_builder_t* builder);

typedef struct _lrc_builder_vtable_t {
  lrc_builder_on_text_t on_text;
  lrc_builder_on_error_t on_error;
  lrc_builder_on_id_tag_t on_id_tag;
  lrc_builder_on_time_tag_t on_time_tag;
  lrc_builder_destroy_t destroy;
} lrc_builder_vtable_t;

/**
 * @class lrc_builder_t
 * lrc builder
 */
struct _lrc_builder_t {
  const lrc_builder_vtable_t* vt;
};

/**
 * @method lrc_builder_on_id_tag
 * 处理id标签。
 *
 * @param {lrc_builder_t*} builder lrc_builder对象。
 * @param {const char*} id 名称。
 * @param {const char*} value 值。
 *
 * @return {ret_t} 返回RET_OK表示成功，否则表示失败。
 */
ret_t lrc_builder_on_id_tag(lrc_builder_t* builder, const char* id, const char* value);

/**
 * @method lrc_builder_on_time_tag
 * 处理time标签。
 *
 * @param {lrc_builder_t*} builder lrc_builder对象。
 * @param {uint32_t} timestamp 时间。
 *
 * @return {ret_t} 返回RET_OK表示成功，否则表示失败。
 */
ret_t lrc_builder_on_time_tag(lrc_builder_t* builder, uint32_t timestamp);

/**
 * @method lrc_builder_on_text
 * 处理歌词。
 *
 * @param {lrc_builder_t*} builder lrc_builder对象。
 * @param {const char*} text 歌词。
 *
 * @return {ret_t} 返回RET_OK表示成功，否则表示失败。
 */
ret_t lrc_builder_on_text(lrc_builder_t* builder, const char* text);

/**
 * @method lrc_builder_on_error
 * 处理错误。
 *
 * @param {lrc_builder_t*} builder lrc_builder对象。
 * @param {const char*} error 错误。
 *
 * @return {ret_t} 返回RET_OK表示成功，否则表示失败。
 */
ret_t lrc_builder_on_error(lrc_builder_t* builder, const char* error);

/**
 * @method lrc_builder_destroy
 * 销毁lrc builder对象。
 *
 * @param {lrc_builder_t*} builder lrc_builder对象。
 *
 * @return {ret_t} 返回RET_OK表示成功，否则表示失败。
 */
ret_t lrc_builder_destroy(lrc_builder_t* builder);

END_C_DECLS

#endif /*TK_LRC_BUILDER_H*/
```

### 3.2 Builder的实现一

lrc_builder_dump：直接把解析的内容保存为文本，方便打印调试 以及 美化格式较乱的 lrc 文件，其主要代码如下：

```c
/* lrc_builder_dump.c */
#include "tkc/mem.h"
#include "tkc/utils.h"
#include "media_player/lrc/lrc_builder_dump.h"

static ret_t lrc_builder_dump_on_id_tag(lrc_builder_t* builder, const char* id, const char* value) {
  lrc_builder_dump_t* dump = (lrc_builder_dump_t*)builder;

  str_append(&(dump->result), "[");
  str_append(&(dump->result), id);
  str_append(&(dump->result), ":");
  str_append(&(dump->result), value);
  str_append(&(dump->result), "]");

  return RET_OK;
}

static ret_t lrc_builder_dump_on_time_tag(lrc_builder_t* builder, uint32_t timestamp) {
  char buff[64];
  uint32_t m = timestamp / (1000 * 60);
  double s = (timestamp % (1000 * 60)) / 1000.0f;
  lrc_builder_dump_t* dump = (lrc_builder_dump_t*)builder;

  tk_snprintf(buff, sizeof(buff), "[%02d:%2.2f]", m, s);

  str_append(&(dump->result), buff);

  return RET_OK;
}

static ret_t lrc_builder_dump_on_text(lrc_builder_t* builder, const char* text) {
  lrc_builder_dump_t* dump = (lrc_builder_dump_t*)builder;

  str_append(&(dump->result), text);

  return RET_OK;
}

static ret_t lrc_builder_dump_on_error(lrc_builder_t* builder, const char* error) {
  lrc_builder_dump_t* dump = (lrc_builder_dump_t*)builder;

  str_append(&(dump->result), error);

  return RET_OK;
}

static ret_t lrc_builder_dump_destroy(lrc_builder_t* builder) {
  lrc_builder_dump_t* dump = (lrc_builder_dump_t*)builder;
  str_reset(&(dump->result));

  TKMEM_FREE(builder);

  return RET_OK;
}

static const lrc_builder_vtable_t s_lrc_builder_dump_vtable = {
    .on_text = lrc_builder_dump_on_text,
    .on_id_tag = lrc_builder_dump_on_id_tag,
    .on_time_tag = lrc_builder_dump_on_time_tag,
    .on_error = lrc_builder_dump_on_error,
    .destroy = lrc_builder_dump_destroy,
};

lrc_builder_t* lrc_builder_dump_create(void) {
  lrc_builder_dump_t* dump = TKMEM_ZALLOC(lrc_builder_dump_t);
  return_value_if_fail(dump != NULL, NULL);

  str_init(&(dump->result), 0);
  dump->lrc_builder.vt = &s_lrc_builder_dump_vtable;

  return (lrc_builder_t*)dump;
}
```

### 3.3 Builder的实现二

lrc：这是默认的builder，它负责把lrc文件构建成内存中的结构，以便查询，其主要代码如下：

```c
/* lrc.c */
#include "tkc/mem.h"
#include "media_player/lrc/lrc.h"
#include "media_player/lrc/lrc_parser.h"
#include "media_player/lrc/lrc_builder.h"

typedef struct _lrc_builder_default_t {
  lrc_builder_t lrc_builder;

  lrc_t* lrc;
  char* p;
  char* strs;
  uint32_t size;
} lrc_builder_default_t;

#define lrc_isspace(c) ((c) == ' ' || (c) == '\t' || (c) == '\n' || (c) == '\r')

static const char* lrc_builder_default_dup(lrc_builder_default_t* b, const char* text) {
  char* p = b->p;
  uint32_t size = strlen(text);
  const char* start = text;
  const char* end = start + size - 1;

  while (*start && lrc_isspace(*start)) start++;
  while (end > start && lrc_isspace(*end)) end--;

  size = end - start + 1;
  memcpy(p, start, size);
  p[size] = '\0';

  b->p += size + 1;

  return p;
}

#define DUP(text) lrc_builder_default_dup(b, text)

static ret_t lrc_builder_default_on_id_tag(lrc_builder_t* builder, const char* id,
                                           const char* value) {
  lrc_builder_default_t* b = (lrc_builder_default_t*)builder;

  lrc_id_tag_list_append(b->lrc->id_tags, DUP(id), DUP(value));

  return RET_OK;
}

static ret_t lrc_builder_default_on_time_tag(lrc_builder_t* builder, uint32_t timestamp) {
  lrc_builder_default_t* b = (lrc_builder_default_t*)builder;

  lrc_time_tag_list_append(b->lrc->time_tags, timestamp);

  return RET_OK;
}

static ret_t lrc_builder_default_on_text(lrc_builder_t* builder, const char* text) {
  lrc_builder_default_t* b = (lrc_builder_default_t*)builder;

  lrc_time_tag_list_set_text(b->lrc->time_tags, DUP(text));

  return RET_OK;
}

static ret_t lrc_builder_default_on_error(lrc_builder_t* builder, const char* error) {
  log_debug("error:%s\n", error);

  return RET_OK;
}

static ret_t lrc_builder_default_destroy(lrc_builder_t* builder) {
  lrc_builder_default_t* b = (lrc_builder_default_t*)builder;

  b->lrc->strs = b->strs;

  return RET_OK;
}

static const lrc_builder_vtable_t s_lrc_builder_default_vtable = {
    .on_text = lrc_builder_default_on_text,
    .on_id_tag = lrc_builder_default_on_id_tag,
    .on_time_tag = lrc_builder_default_on_time_tag,
    .on_error = lrc_builder_default_on_error,
    .destroy = lrc_builder_default_destroy,
};

lrc_builder_t* lrc_builder_default_init(lrc_builder_default_t* b, lrc_t* lrc, char* strs,
                                        uint32_t size) {
  return_value_if_fail(strs != NULL, NULL);

  b->lrc = lrc;
  b->p = strs;
  b->strs = strs;
  b->size = size;
  memset(strs, 0x00, size);
  b->lrc_builder.vt = &s_lrc_builder_default_vtable;

  return (lrc_builder_t*)b;
}

static lrc_t* lrc_parse(lrc_t* lrc, const char* text) {
  ret_t ret = RET_OK;
  lrc_builder_default_t builder;
  uint32_t size = strlen(text) + 1;
  char* strs = TKMEM_ALLOC(size);
  lrc_builder_t* b = lrc_builder_default_init(&builder, lrc, strs, size);

  ret = lrc_parser_parse(b, text);
  lrc_time_tag_list_sort(lrc->time_tags);
  lrc_builder_destroy(&builder);

  return ret == RET_OK ? lrc : NULL;
}

lrc_t* lrc_create(const char* text) {
  lrc_t* lrc = NULL;
  return_value_if_fail(text != NULL, NULL);

  lrc = TKMEM_ZALLOC(lrc_t);
  return_value_if_fail(lrc != NULL, NULL);

  lrc->id_tags = lrc_id_tag_list_create();
  lrc->time_tags = lrc_time_tag_list_create();

  if (lrc->id_tags == NULL || lrc->time_tags == NULL) {
    lrc_destroy(lrc);
    lrc = NULL;
  }

  return_value_if_fail(lrc != NULL, NULL);

  if (lrc_parse(lrc, text) == NULL) {
    lrc_destroy(lrc);
    lrc = NULL;
  }

  return lrc;
}

ret_t lrc_destroy(lrc_t* lrc) {
  return_value_if_fail(lrc != NULL, RET_BAD_PARAMS);

  lrc_id_tag_list_destroy(lrc->id_tags);
  lrc_time_tag_list_destroy(lrc->time_tags);
  TKMEM_FREE(lrc->strs);
  TKMEM_FREE(lrc);

  return RET_OK;
}
```

### 3.4 产品（Product）

根据上篇 [文章](./builder.md) 中的描述，Product 就是 ConcreteBuilder 产生的结果，不同的 ConcreteBuilder 所产生的 Product 是不同的。

在上面的例子中，lrc_builder_dump 产生的 Product 是一段文本。而Builder默认的实现（lrc）产生的 Product 是一个数据结构，如下：

```c
/**
 * @class lrc_t
 * lrc
 */
typedef struct _lrc_t {
  /**
   * @property {lrc_id_tag_list_t*} id_tags
   * @annotation ["readable"]
   * id tags。
   */
  lrc_id_tag_list_t* id_tags;

  /**
   * @property {lrc_time_tag_list_t*} time_tags
   * @annotation ["readable"]
   * time tags。
   */
  lrc_time_tag_list_t* time_tags;

  /*private*/
  char* strs;
} lrc_t;
```

### 3.5 解析器(Director)的工作过程

解析器的任务就是解析出 lrc 文件最基本的元素：时间标签、ID标签和歌词，然后调用builder相应的函数表示出来，此处仅做例子，完整代码请参考 [lrc_parser.c](https://github.com/zlgopen/awtk-media-player/blob/master/src/media_player/lrc/lrc_parser.c) 。

```c
static ret_t lrc_parser_parse_tag(lrc_parser_t* parser) {
  lrc_parser_skip_chars(parser, "\t \r\n");

  if (parser->p[0] == '\0') {
    return RET_OK;
  }

  if (isdigit(parser->p[0])) {
    return lrc_parser_parse_time_tag(parser);
  } else {
    return lrc_parser_parse_id_tag(parser);
  }
}

static ret_t lrc_parser_parse_impl(lrc_parser_t* parser) {
  lrc_parser_skip_text(parser);

  while (TRUE) {
    char c = parser->p[0];

    if (c == '\0') {
      break;
    }

    if (c == '[') {
      parser->p++;
      lrc_parser_parse_tag(parser);
      if (parser->p[0] == ']') {
        parser->p++;
      }
    } else {
      lrc_parser_parse_text(parser);
    }
  }

  return RET_OK;
}

ret_t lrc_parser_parse(lrc_builder_t* builder, const char* str) {
  lrc_parser_t p;
  ret_t ret = RET_OK;

  return_value_if_fail(lrc_parser_init(&p, builder, str) == RET_OK, RET_BAD_PARAMS);
  ret = lrc_parser_parse_impl(&p);
  lrc_parser_deinit(&p);

  return ret;
}
```

### 3.6  调用者(Client)

有了 解析器（Parser）和 相应的Builder 后，调用者需要把它们组合起来。解析完成时，调用者还希望从 Builder 取出 Product，以便后面使用。例如此处调用 lrc_create() 函数解析歌词文本（text参数）即可，代码如下：

```c
static lrc_t* lrc_parse(lrc_t* lrc, const char* text) {
  ret_t ret = RET_OK;
  lrc_builder_default_t builder;
  uint32_t size = strlen(text) + 1;
  char* strs = TKMEM_ALLOC(size);
  lrc_builder_t* b = lrc_builder_default_init(&builder, lrc, strs, size);

  ret = lrc_parser_parse(b, text);
  lrc_time_tag_list_sort(lrc->time_tags);
  lrc_builder_destroy(&builder);

  return ret == RET_OK ? lrc : NULL;
}

lrc_t* lrc_create(const char* text) {
  lrc_t* lrc = NULL;
  return_value_if_fail(text != NULL, NULL);

  lrc = TKMEM_ZALLOC(lrc_t);
  return_value_if_fail(lrc != NULL, NULL);

  lrc->id_tags = lrc_id_tag_list_create();
  lrc->time_tags = lrc_time_tag_list_create();

  if (lrc->id_tags == NULL || lrc->time_tags == NULL) {
    lrc_destroy(lrc);
    lrc = NULL;
  }

  return_value_if_fail(lrc != NULL, NULL);

  if (lrc_parse(lrc, text) == NULL) {
    lrc_destroy(lrc);
    lrc = NULL;
  }

  return lrc;
}
```
