---
title: SQLite的基本用法
date: 2020-12-06
categories:
 - 数据库
tags:
 - C
 - SQL
 - 数据库
---

## 1 什么是SQLite？

SQLite是一个软件库，实现了自给自足的、无服务器的、零配置的、事务性的 SQL 数据库引擎。SQLite 是在世界上最广泛部署的 SQL 数据库引擎。SQLite 源代码不受版权限制。

## 2 SQL语句

**特点：** 不区分大小写，每条语句后加";"结尾。

**关键字：** select、insert、update、delete、from、creat、where、desc、order、by、group、table、alter、view、index等，数据库中不能使用关键字命名表和字段。

### 2.1 表的增删

#### 2.1.1 新建表

```sql
CREATE TABLE database_name.table_name(
   column1 datatype  PRIMARY KEY(one or more columns),
   column2 datatype,
   column3 datatype,
   .....
   columnN datatype,
);
```

#### 2.1.2 删除表

```sql
DROP TABLE database_name.table_name;
```

### 2.2 记录的增删改查

#### 2.2.1 增

```sql
指定列：
INSERT INTO TABLE_NAME [(column1, column2, column3,...columnN)]  
VALUES (value1, value2, value3,...valueN);

所有列：
INSERT INTO TABLE_NAME VALUES (value1,value2,value3,...valueN);
```

#### 2.2.2 删

```sql
DELETE FROM table_name
WHERE [condition];
如果需要删除表内所有记录：DELETE FROM table_name
```

#### 2.2.3 改

```sql
UPDATE table_name
SET column1 = value1, column2 = value2...., columnN = valueN
WHERE [condition];
```

#### 2.2.4 查

```sql
指定列：
SELECT column1, column2, columnN FROM table_name;
所有列：
SELECT * FROM table_name;
```

> 备注：更多SQLite的SQL语句规则请查阅：[https://www.runoob.com/sqlite/sqlite-tutorial.html](https://www.runoob.com/sqlite/sqlite-tutorial.html)

## 3 SQLite的基本用法

1. 下载SQLite源码，主要包含：shell.c、sqlite3.c、sqlite3.h和sqlite3ext.h。

> 备注：SQLite源代码下载地址：[https://www.sqlite.org/index.html](https://www.sqlite.org/index.html)

2. 包含头文件：

```c
#include "sqlite3.h"
```

3. 初始化SQLite：

```c
int sqlite3_initialize(void)
```

4. 连接数据库：

```c
/* 根据文件路径打开数据库，如果不存在，则会创建一个新的数据库。 */
int sqlite3_open(const char *zFilename, sqlite3 **ppDb)
```

5. 执行SQL语句：

```c
int sqlite3_exec(
  sqlite3 *db,                /* The database on which the SQL executes */
  const char *zSql,           /* The SQL to be executed */
  sqlite3_callback xCallback, /* Invoke this callback routine */
  void *pArg,                 /* First argument to xCallback() */
  char **pzErrMsg             /* Write error messages here */
)
```

6. 使用sqlite3_prepare查询数据集（sqlite3_stmt*），示例如下：

```c
char sql = "SELECT * FROM Book;";
sqlite3_stmt* stmt = NULL; /* 数据集 */
/* -1代表系统会自动计算SQL语句的长度 */
sqlite3_prepare(br_sqlite3->db, sql, -1, &stmt, NULL)
/* 每调一次sqlite3_step()函数，stmt就会指向下一条记录 */
while(sqlite3_step(stmt) == SQLITE_ROW) {
  sqlite3_column_int(stmt, 0);  /* 获取第0列的int值 */
  sqlite3_column_text(stmt, 1); /* 获取第1列的text值 */
  ...
 }
sqlite3_finalize(stmt);  /* 释放数据集 */
```

7. 关闭数据库：
```c
int sqlite3_close(sqlite3 *db)
```

8. 关闭SQLite：

```c
int sqlite3_shutdown(void)
```

## 4 SQLite在C语言中的应用

功能描述：基于SQLite数据库实现图书管理系统中的book_repository_sqlite3_t类，该类包含创建SQLite3图书仓库、获取全部图书、增删改查、清空图书以及销毁SQLite3图书仓库的功能。

> 注意：book_repository_sqlite3_t是book_repository_t的子类，book_repository_t基类代码详见文章：[Repository模式及其在C语言中的应用](../Other/repository_mode.md)。

### 4.1 子类头文件

```c
/* book_repository_sqlite3.h */
#ifndef _BOOK_REPOSITORY_SQLLITE3_H
#define _BOOK_REPOSITORY_SQLLITE3_H

#include "book_repository.h"
#include "../../3rd/sqlite3/sqlite3.h"

BEGIN_C_DECLS

/**
 * @class book_repository_sqlite3_t
 * @parent book_repository_t
 * 
 * sqlite3图书仓库。
 * 
 */
typedef struct _book_repository_sqlite3_t {
  book_repository_t br;

  /**
   * @property {char*} app_name
   * @annotation ["readable"]
   * sqlite3数据库文件（book.db）的目录。
   */
  char* app_name;
  /**
   * @property {sqlite3*} db
   * @annotation ["readable"]
   * sqlite3数据库实例。
   */
  sqlite3* db;
} book_repository_sqlite3_t;

/**
 * @method book_repository_sqlite3_create
 * 创建sqlite3图书仓库。
 * 
 * @param {char*} app_name  sqlite3数据库文件（book.db）的目录。。
 *
 * @return {book_repository_t*} book repository对象。
 */
book_repository_t* book_repository_sqlite3_create(char* app_name);

END_C_DECLS

#endif /*_BOOK_REPOSITORY_SQLLITE3_H*/
```

### 4.2 子类源文件

以下代码均来自子类源文件 `book_repository_sqlite3.c`。

#### 4.2.1 创建sqlite3图书仓库

```c
/* 准备数据库文件路径 */
static ret_t prepare_database_file(char db_filename[MAX_PATH + 1], const char* app_name,
                                   const char* db_name) {
  char home[MAX_PATH + 1];
  char path[MAX_PATH + 1];

  fs_get_user_storage_path(os_fs(), home);
  path_build(path, MAX_PATH, home, app_name, NULL);

  if (!path_exist(path)) {
    fs_create_dir(os_fs(), path);
  }
  path_build(db_filename, MAX_PATH, path, db_name, NULL);

  return RET_OK;
}

/* 初始化数据库（创建book表） */
static ret_t database_init(char db_filename[MAX_PATH + 1], book_repository_sqlite3_t* br_sqlite3) {
  char* err_msg = NULL;

  sqlite3_initialize();
  ret_t ret_file = prepare_database_file(db_filename, br_sqlite3->app_name, "book.db");
  return_value_if_fail(ret_file == RET_OK, RET_FAIL);
  int rc = sqlite3_open(db_filename, &(br_sqlite3->db));
  return_value_if_fail(rc == SQLITE_OK, RET_FAIL);

  /* create table */
  const char* sql =
      "CREATE TABLE IF NOT EXISTS Book ("
      "id INT PRIMARY KEY NOT NULL, "
      "title TEXT NOT NULL, "
      "author TEXT NOT NULL, "
      "press TEXT NOT NULL, "
      "year INT NOT NULL, "
      "month INT NOT NULL, "
      "day INT NOT NULL, "
      "number INT NOT NULL);";

  /* exec sql(callback is NULL) */
  rc = sqlite3_exec(br_sqlite3->db, sql, NULL, NULL, &err_msg);
  return_value_if_fail(rc == SQLITE_OK, (sqlite3_free(err_msg), RET_FAIL));

  return RET_OK;
}

/* 创建sqlite3图书仓库 */
book_repository_t* book_repository_sqlite3_create(char* app_name) {
  return_value_if_fail(app_name != NULL, NULL);

  book_repository_sqlite3_t* br_sqlite3 = TKMEM_CALLOC(1, sizeof(book_repository_sqlite3_t));
  book_repository_t* br = (book_repository_t*)br_sqlite3;
  return_value_if_fail(br != NULL, NULL);

  br->get_all = book_repository_sqlite3_get_all;
  br->add = book_repository_sqlite3_add;
  br->remove = book_repository_sqlite3_remove;
  br->update = book_repository_sqlite3_update;
  br->find = book_repository_sqlite3_find;
  br->clear = book_repository_sqlite3_clear;
  br->destroy = book_repository_sqlite3_destroy;

  br_sqlite3->app_name = tk_strdup(app_name);
  br_sqlite3->db = NULL;
  char db_filename[MAX_PATH + 1] = {0};

  ret_t ret = database_init(db_filename, br_sqlite3);
  return_value_if_fail(ret == RET_OK, (br->destroy(br), NULL));

  return br;
}
```

#### 4.2.2 获取全部图书

```c
static darray_t* book_repository_sqlite3_get_all(book_repository_t* br, darray_t* darray) {
  book_repository_sqlite3_t* br_sqlite3 = (book_repository_sqlite3_t*)br;
  darray_clear(darray);
  char sql[MAX_SQL] = {0};
  sqlite3_stmt* stmt = NULL; /* sava data */
  tk_snprintf(sql, sizeof(sql), "SELECT * FROM Book;");
  int rc = sqlite3_prepare(br_sqlite3->db, sql, -1, &stmt, NULL);
  return_value_if_fail(rc == SQLITE_OK, darray);

  while (sqlite3_step(stmt) == SQLITE_ROW) {
    if (darray->size < darray->capacity) {
      book_t* book = book_create();
      book_set_id(book, sqlite3_column_int(stmt, 0));
      book_set_title(book, sqlite3_column_text(stmt, 1));
      book_set_author(book, sqlite3_column_text(stmt, 2));
      book_set_press(book, sqlite3_column_text(stmt, 3));
      book_set_year(book, sqlite3_column_int(stmt, 4));
      book_set_month(book, sqlite3_column_int(stmt, 5));
      book_set_day(book, sqlite3_column_int(stmt, 6));
      book->number = sqlite3_column_int(stmt, 7);
      darray_push(darray, book);
    }
  }
  sqlite3_finalize(stmt);
  return darray;
}
```

#### 4.2.3 增加图书

```c
static ret_t book_repository_sqlite3_add(book_repository_t* br, book_t* book) {
  book_repository_sqlite3_t* br_sqlite3 = (book_repository_sqlite3_t*)br;
  book_t* bk = book_repository_sqlite3_find(br, book);
  if (bk != NULL) {
    bk->number++;
    book_repository_sqlite3_update(br, bk, bk);
    book_destroy(bk);
  } else {
    book->number = 1;
    char sql[MAX_SQL] = {0};
    char* err_msg = NULL;
    tk_snprintf(sql, sizeof(sql), "INSERT INTO Book VALUES (%d, '%s', '%s', '%s', %d, %d, %d, %d);",
                book->id, book->title, book->author, book->press, book->year, book->month,
                book->day, book->number);
    int rc = sqlite3_exec(br_sqlite3->db, sql, NULL, NULL, &err_msg);
    return_value_if_fail(rc == SQLITE_OK, (sqlite3_free(err_msg), RET_FAIL));
  }
  return RET_OK;
}
```

#### 4.2.4 删除图书

```c
static ret_t book_repository_sqlite3_remove(book_repository_t* br, book_t* book, int32_t number) {
  book_repository_sqlite3_t* br_sqlite3 = (book_repository_sqlite3_t*)br;
  book_t* bk = book_repository_sqlite3_find(br, book);
  if (bk != NULL) {
    int32_t book_number = bk->number - number;
    if (book_number > 0) {
      bk->number = book_number;
      book_repository_sqlite3_update(br, bk, bk);
    } else {
      char sql[MAX_SQL] = {0};
      char* err_msg = NULL;
      tk_snprintf(sql, sizeof(sql), "DELETE FROM Book WHERE id = %d;", bk->id);
      int rc = sqlite3_exec(br_sqlite3->db, sql, NULL, NULL, &err_msg);
      return_value_if_fail(rc == SQLITE_OK, (sqlite3_free(err_msg), RET_FAIL));
    }
    book_destroy(bk);
  } else {
    return RET_FAIL;
  }
  return RET_OK;
}
```

#### 4.2.5 修改图书

```c
static ret_t book_repository_sqlite3_update(book_repository_t* br, book_t* new_book,
                                            book_t* old_book) {
  book_repository_sqlite3_t* br_sqlite3 = (book_repository_sqlite3_t*)br;
  book_t* bk = book_repository_sqlite3_find(br, old_book);
  if (bk == NULL) {
    return RET_FAIL;
  }
  char sql[MAX_SQL] = {0};
  char* err_msg = NULL;
  tk_snprintf(sql, sizeof(sql),
              "UPDATE Book SET id = %d, title = '%s', author = '%s', press = '%s', year = %d, "
              "month = %d, day = %d, number = %d WHERE id = %d;",
              new_book->id, new_book->title, new_book->author, new_book->press, new_book->year,
              new_book->month, new_book->day, new_book->number, old_book->id);
  int rc = sqlite3_exec(br_sqlite3->db, sql, NULL, NULL, &err_msg);
  return_value_if_fail(rc == SQLITE_OK, (sqlite3_free(err_msg), RET_FAIL));
  book_destroy(bk);
  return RET_OK;
}
```

#### 4.2.6 查询图书

```c
static book_t* book_repository_sqlite3_find(book_repository_t* br, book_t* book) {
  book_repository_sqlite3_t* br_sqlite3 = (book_repository_sqlite3_t*)br;
  char sql[MAX_SQL] = {0};
  sqlite3_stmt* stmt = NULL; /* sava data */

  tk_snprintf(sql, sizeof(sql), "SELECT * FROM Book WHERE id = %d;", book->id);
  int rc = sqlite3_prepare(br_sqlite3->db, sql, -1, &stmt, NULL);
  return_value_if_fail(rc == SQLITE_OK, NULL);

  if (sqlite3_step(stmt) == SQLITE_ROW) {
    book_t* ret_book = book_create();
    book_init(ret_book, sqlite3_column_int(stmt, 0), sqlite3_column_text(stmt, 1),
              sqlite3_column_text(stmt, 2), sqlite3_column_text(stmt, 3),
              sqlite3_column_int(stmt, 4), sqlite3_column_int(stmt, 5),
              sqlite3_column_int(stmt, 6));
    ret_book->number = sqlite3_column_int(stmt, 7);
    sqlite3_finalize(stmt);
    return ret_book;
  }
```

#### 4.2.7 清空图书

```c
static ret_t book_repository_sqlite3_clear(book_repository_t* br) {
  book_repository_sqlite3_t* br_sqlite3 = (book_repository_sqlite3_t*)br;
  char* err_msg = NULL;
  const char* sql = "DELETE FROM Book;";
  int rc = sqlite3_exec(br_sqlite3->db, sql, NULL, NULL, &err_msg);
  return_value_if_fail(rc == SQLITE_OK, (sqlite3_free(err_msg), RET_FAIL));
  return RET_OK;
}
```

#### 4.2.8 销毁sqlite3图书仓库

```c
static ret_t book_repository_sqlite3_destroy(book_repository_t* br) {
  book_repository_sqlite3_t* br_sqlite3 = (book_repository_sqlite3_t*)(br);
  TKMEM_FREE(br_sqlite3->app_name);
  sqlite3_close(br_sqlite3->db);
  sqlite3_shutdown();
  TKMEM_FREE(br);
  return RET_OK;
}
```
