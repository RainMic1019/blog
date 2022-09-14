---
title: Repository模式及其在C语言中的应用
date: 2020-11-29
categories:
 - 程序设计
tags:
 - C
---

## 1 什么是Repository模式？

Repository是一个独立的层，介于领域层（业务逻辑层）与数据映射层（数据访问层）之间。它的存在让领域层感觉不到数据访问层的存在，它提供一个类似集合的接口提供给领域层进行领域对象的访问。Repository是仓库管理员，领域层需要什么东西只需告诉仓库管理员，由仓库管理员把东西拿给它，并不需要知道东西实际放在哪。

- Repository 模式是架构模式，在设计架构时，才有参考价值；
- Repository 模式主要是封装数据查询和存储逻辑；
- Repository 模式实际用途：更换、升级 ORM 引擎，不影响业务逻辑；
- Repository 模式能提高测试效率，单元测试时，用 Mock 对象代替实际的数据库存取，可以成倍地提高测试用例运行速度。


## 2 Repository模式在C语言中的应用

问题描述：实现图书管理系统中的book_repository_t基类，包含获取全部图书以及增删改查的接口，在子类book_repository_json中实现其具体功能。

### 2.1 基类头文件

```c
/* book_repository.h */
#ifndef _BOOK_REPOSITORY_H
#define _BOOK_REPOSITORY_H

#include "../book/book.h"
#include "tkc/darray.h"
#include "tkc/types_def.h"

BEGIN_C_DECLS

struct _book_repository_t;
typedef struct _book_repository_t book_repository_t;

typedef darray_t* (*book_repository_get_all_t)(book_repository_t* br, darray_t* darray);
typedef ret_t (*book_repository_add_t)(book_repository_t* br, book_t* book);
typedef ret_t (*book_repository_remove_t)(book_repository_t* br, book_t* book, int32_t number);
typedef ret_t (*book_repository_update_t)(book_repository_t* br, book_t* new_book,
                                          book_t* old_book);
typedef book_t* (*book_repository_find_t)(book_repository_t* br, book_t* book);
typedef ret_t (*book_repository_clear_t)(book_repository_t* br);
typedef ret_t (*book_repository_destroy_t)(book_repository_t* br);

/**
 * @class book_repository_t
 * 图书仓库抽象基类。
 * 
 */
struct _book_repository_t {
  book_repository_get_all_t get_all;
  book_repository_add_t add;
  book_repository_remove_t remove;
  book_repository_update_t update;
  book_repository_find_t find;
  book_repository_clear_t clear;
  book_repository_destroy_t destroy;
};

/**
 * @method book_repository_get_all
 * 获取图书仓库中的所有图书。
 * @param {book_repository_t*} book_repository对象。
 * @param {darray_t*} darray 动态数组指针。
 *
 * @return {darray_t*} 返回存储所有book对象的动态数组指针，失败返回NULL。
 */
darray_t* book_repository_get_all(book_repository_t* br, darray_t* darray);

/**
 * @method book_repository_add
 * 向图书仓库中增加一个图书。
 * @param {book_repository_t*} book_repository对象。
 * @param {book_t} book 待增加的图书。
 *
 * @return {ret_t} 返回RET_OK表示成功，否则表示失败。
 */
ret_t book_repository_add(book_repository_t* br, book_t* book);

/**
 * @method book_repository_remove
 * 从图书仓库中删除指定数量的图书。
 * @param {book_repository_t*} book_repository对象。
 * @param {book_t*} book 图书对象。
 * @param {int32_t} number 数量（0为不删除，大于图书数量则全部删除）。
 *
 * @return {ret_t} 返回RET_OK表示成功，否则表示失败。
 */
ret_t book_repository_remove(book_repository_t* br, book_t* book, int32_t number);

/**
 * @method book_repository_update
 * 在图书仓库中修改指定的图书。
 * @param {book_repository_t*} book_repository对象。
 * @param {book_t*} new_book 修改后的图书对象。
 * @param {book_t*} old_book 修改前的图书对象。
 *
 * @return {ret_t} 返回RET_OK表示成功，否则表示失败。
 */
ret_t book_repository_update(book_repository_t* br, book_t* new_book, book_t* old_book);

/**
 * @method book_repository_find
 * 在图书仓库中查找指定的图书。
 * @param {book_repository_t*} book_repository对象。
 * @param {book_t*} book 图书对象。
 *
 * @return {book_t*} 如果找到，返回满足条件的book对象，否则返回NULL。
 */
book_t* book_repository_find(book_repository_t* br, book_t* book);

/**
 * @method book_repository_clear
 * 清除图书仓库中的图书。
 * @param {book_repository_t*} book_repository对象。
 *
 * @return {ret_t} 返回RET_OK表示成功，否则表示失败。
 */
ret_t book_repository_clear(book_repository_t* br);

/**
 * @method book_repository_deinit
 * 销毁图书仓库。
 * @param {book_repository_t*} book_repository对象。
 *
 * @return {ret_t} 返回RET_OK表示成功，否则表示失败。
 */
ret_t book_repository_destroy(book_repository_t* br);

END_C_DECLS

#endif /*_BOOK_REPOSITORY_H*/
```

### 2.2 基类源文件

```c
/* book_repository.c */
#include "book_repository.h"

darray_t* book_repository_get_all(book_repository_t* br, darray_t* darray) {
  return_value_if_fail(br != NULL && darray != NULL, NULL);
  return br->get_all(br, darray);
}

ret_t book_repository_add(book_repository_t* br, book_t* book) {
  return_value_if_fail(br != NULL && book != NULL, RET_BAD_PARAMS);
  return br->add(br, book);
}

ret_t book_repository_remove(book_repository_t* br, book_t* book, int32_t number) {
  return_value_if_fail(br != NULL && book != NULL && number >= 0, RET_BAD_PARAMS);
  return br->remove(br, book, number);
}

ret_t book_repository_update(book_repository_t* br, book_t* new_book, book_t* old_book) {
  return_value_if_fail(br != NULL && new_book != NULL && old_book != NULL, RET_BAD_PARAMS);
  return br->update(br, new_book, old_book);
}

book_t* book_repository_find(book_repository_t* br, book_t* book) {
  return_value_if_fail(br != NULL && book != NULL, NULL);
  return br->find(br, book);
}

ret_t book_repository_clear(book_repository_t* br) {
  return_value_if_fail(br != NULL, RET_BAD_PARAMS);
  return br->clear(br);
}

ret_t book_repository_destroy(book_repository_t* br) {
  return_value_if_fail(br != NULL, RET_BAD_PARAMS);
  return br->destroy(br);
}
```

### 2.3 子类头文件

```c
/* book_repository_json.h */
#ifndef _BOOK_REPOSITORY_JSON_H
#define _BOOK_REPOSITORY_JSON_H

#include "book_repository.h"

BEGIN_C_DECLS

/**
 * @class book_repository_json_t
 * @parent book_repository_t
 * 
 * json文件图书仓库。
 * 
 */
typedef struct _book_repository_json_t {
  book_repository_t br;

  /**
   * @property {char*} app_name
   * @annotation ["readable"]
   * 图书仓库json文件（app_conf.json）的目录。
   */
  char* app_name;
} book_repository_json_t;

/**
 * @method book_repository_json_create
 * 创建json文件图书仓库。
 * 
 * @param {char*} app_name 图书仓库json文件（app_conf.json）的目录。
 *
 * @return {book_repository_t*} book repository对象。
 */
book_repository_t* book_repository_json_create(char* app_name);

END_C_DECLS

#endif /*_BOOK_REPOSITORY_JSON_H*/
```

### 2.4 子类源文件

```c
/* book_repository_json.c */
#include "book_repository_json.h"
#include "tkc/mem.h"
#include "tkc/utils.h"
#include "conf_io/app_conf_init_json.h"
#include "conf_io/app_conf.h"

#define KEY_LEN 100

static darray_t* book_repository_json_get_all(book_repository_t* br, darray_t* darray);
static ret_t book_repository_json_add(book_repository_t* br, book_t* book);
static ret_t book_repository_json_remove(book_repository_t* br, book_t* book, int32_t number);
static ret_t book_repository_json_update(book_repository_t* br, book_t* new_book, book_t* old_book);
static book_t* book_repository_json_find(book_repository_t* br, book_t* book);
static ret_t book_repository_json_clear(book_repository_t* br);
static ret_t book_repository_json_destroy(book_repository_t* br);

static ret_t book_repository_json_read_book(book_t* book, int32_t id);
static ret_t book_repository_json_write_book(book_t* book);

static darray_t* book_repository_json_get_all(book_repository_t* br, darray_t* darray) {
  darray_clear(darray);
  char key[KEY_LEN] = {0};
  int32_t i = 0;
  while (TRUE) {
    memset(key, 0x00, sizeof(key));
    tk_snprintf(key, sizeof(key), "books.[%d].title", i);
    if (app_conf_exist(key) && darray->size < darray->capacity) {
      book_t* book = book_create();
      memset(key, 0x00, sizeof(key));
      tk_snprintf(key, sizeof(key), "books.[%d].#name", i);
      book_repository_json_read_book(book, tk_atoi(app_conf_get_str(key, "0")));
      darray_push(darray, book);
    } else {
      break;
    }
    i++;
  }
  return darray;
}

static ret_t book_repository_json_add(book_repository_t* br, book_t* book) {
  book_t* bk = book_repository_json_find(br, book);
  if (bk != NULL) {
    char key[KEY_LEN] = {0};
    tk_snprintf(key, sizeof(key), "books.%d.number", bk->id);
    app_conf_set_int(key, bk->number + 1);
    book_destroy(bk);
  } else {
    book->number = 1;
    book_repository_json_write_book(book);
  }
  app_conf_set_int("books_amount", app_conf_get_int("books_amount", 0) + 1);
  app_conf_save();
  return RET_OK;
}

static ret_t book_repository_json_remove(book_repository_t* br, book_t* book, int32_t number) {
  book_t* bk = book_repository_json_find(br, book);
  if (bk != NULL) {
    int32_t book_number = bk->number - number;
    int32_t amount = app_conf_get_int("books_amount", 0);
    if (book_number > 0) {
      char key[KEY_LEN] = {0};
      tk_snprintf(key, sizeof(key), "books.%d.number", bk->id);
      app_conf_set_int(key, book_number);
      amount = amount - number;
    } else {
      char key[KEY_LEN] = {0};
      tk_snprintf(key, sizeof(key), "books.%d", bk->id);
      app_conf_remove(key);
      amount = amount - bk->number;
    }
    app_conf_set_int("books_amount", amount);
    book_destroy(bk);
    app_conf_save();
  } else {
    return RET_FAIL;
  }
  return RET_OK;
}

static ret_t book_repository_json_update(book_repository_t* br, book_t* new_book,
                                         book_t* old_book) {
  book_t* bk = book_repository_json_find(br, old_book);
  if (bk == NULL) {
    return RET_FAIL;
  }
  book_repository_json_remove(br, old_book, 0xffff);
  book_repository_json_write_book(new_book);
  book_destroy(bk);
  app_conf_save();
  return RET_OK;
}

static book_t* book_repository_json_find(book_repository_t* br, book_t* book) {
  char key_id[KEY_LEN] = {0};
  tk_snprintf(key_id, sizeof(key_id), "books.%d.title", book->id);

  if (app_conf_exist(key_id)) {
    book_t* bk = book_create();
    book_repository_json_read_book(bk, book->id);
    return bk;
  }
  return NULL;
}

static ret_t book_repository_json_clear(book_repository_t* br) {
  char key[KEY_LEN] = {0};
  tk_snprintf(key, sizeof(key), "books");
  app_conf_remove("books");
  app_conf_set_int("books_amount", 0);
  app_conf_save();
  return RET_OK;
}

static ret_t book_repository_json_destroy(book_repository_t* br) {
  book_repository_json_t* br_json = (book_repository_json_t*)(br);
  app_conf_deinit();
  TKMEM_FREE(br_json->app_name);
  TKMEM_FREE(br);

  return RET_OK;
}

static ret_t book_repository_json_read_book(book_t* book, int32_t id) {
  return_value_if_fail(book != NULL && id > 0, RET_BAD_PARAMS);

  char key_id[KEY_LEN] = {0};
  tk_snprintf(key_id, sizeof(key_id), "books.%d", id);

  char key[KEY_LEN] = {0};
  tk_snprintf(key, sizeof(key), "%s.title", key_id);
  const char* title = app_conf_get_str(key, NULL);

  memset(key, 0x00, sizeof(key));
  tk_snprintf(key, sizeof(key), "%s.author", key_id);
  const char* author = app_conf_get_str(key, NULL);

  memset(key, 0x00, sizeof(key));
  tk_snprintf(key, sizeof(key), "%s.press", key_id);
  const char* press = app_conf_get_str(key, NULL);

  memset(key, 0x00, sizeof(key));
  tk_snprintf(key, sizeof(key), "%s.year", key_id);
  int32_t year = app_conf_get_int(key, 0);

  memset(key, 0x00, sizeof(key));
  tk_snprintf(key, sizeof(key), "%s.month", key_id);
  int32_t month = app_conf_get_int(key, 0);

  memset(key, 0x00, sizeof(key));
  tk_snprintf(key, sizeof(key), "%s.day", key_id);
  int32_t day = app_conf_get_int(key, 0);

  memset(key, 0x00, sizeof(key));
  tk_snprintf(key, sizeof(key), "%s.number", key_id);
  int32_t number = app_conf_get_int(key, 0);

  book_init(book, id, title, author, press, year, month, day);
  book->number = number;

  return RET_OK;
}

static ret_t book_repository_json_write_book(book_t* book) {
  return_value_if_fail(book != NULL, RET_BAD_PARAMS);

  char key_id[KEY_LEN] = {0};
  tk_snprintf(key_id, sizeof(key_id), "books.%d", book->id);

  char key[KEY_LEN] = {0};
  tk_snprintf(key, sizeof(key), "%s.title", key_id);
  app_conf_set_str(key, book->title);

  memset(key, 0x00, sizeof(key));
  tk_snprintf(key, sizeof(key), "%s.author", key_id);
  app_conf_set_str(key, book->author);

  memset(key, 0x00, sizeof(key));
  tk_snprintf(key, sizeof(key), "%s.press", key_id);
  app_conf_set_str(key, book->press);

  memset(key, 0x00, sizeof(key));
  tk_snprintf(key, sizeof(key), "%s.year", key_id);
  app_conf_set_int(key, book->year);

  memset(key, 0x00, sizeof(key));
  tk_snprintf(key, sizeof(key), "%s.month", key_id);
  app_conf_set_int(key, book->month);

  memset(key, 0x00, sizeof(key));
  tk_snprintf(key, sizeof(key), "%s.day", key_id);
  app_conf_set_int(key, book->day);

  memset(key, 0x00, sizeof(key));
  tk_snprintf(key, sizeof(key), "%s.number", key_id);
  app_conf_set_int(key, book->number);

  return RET_OK;
}

book_repository_t* book_repository_json_create(char* app_name) {
  return_value_if_fail(app_name != NULL, NULL);

  book_repository_json_t* br_json = TKMEM_CALLOC(1, sizeof(book_repository_json_t));
  book_repository_t* br = (book_repository_t*)br_json;
  return_value_if_fail(br != NULL, NULL);

  br->get_all = book_repository_json_get_all;
  br->add = book_repository_json_add;
  br->remove = book_repository_json_remove;
  br->update = book_repository_json_update;
  br->find = book_repository_json_find;
  br->clear = book_repository_json_clear;
  br->destroy = book_repository_json_destroy;

  br_json->app_name = tk_strdup(app_name);

  app_conf_init_json(app_name);
  if (!app_conf_exist("books_amount")) {
    app_conf_set_int("books_amount", 0);
    app_conf_save();
  }

  return br;
}
```
