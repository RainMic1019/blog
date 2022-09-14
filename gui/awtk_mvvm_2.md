---
title: AWTK-MVVM学习（二）：数据绑定与命令绑定
date: 2021-01-03
categories:
 - GUI
tags:
 - C
 - GUI
 - AWTK
 - MVVM
---

## 1 前言

上周简单介绍了AWTK-MVVM，并按照规则设计了图书管理系统的Model，详见文章：[AWTK-MVVM学习（一）](./awtk_mvvm_1.md)。本周学习了AWTK-MVVM的数据绑定和命令绑定，本文记录其中的关键内容，更详细的教程请参考 `awtk-mvvm/docs` 下的md文档。

> 备注：前往 GitHub 下载 [awtk-mvvm](http://github.com/zlgopen/awtk-mvvm)。

## 2 数据绑定

在进行数据绑定之前，先要建立 View（UI界面）与 ViewModel的联系。在AWTK 中用XML文件来描述用户界面，即View，在上周的博客中，设计了一个图书管理系统的Model，可根据注释生成 ViewModel 的代码，实现数据绑定的步骤如下：

步骤一：为界面绑定模型，即将 View 与 ViewModel 关联起来，在AWTK-MVVM中有一个 v-model 属性，该属性用来指定 ViewModel 的名称，将控件与模型绑定起来，通常我们在窗口上指定，XML如下：

```xml
<window v-model="book_controller">
...
</window>
```

步骤二：为控件绑定数据，即将某些控件的属性与 Model 中的指定属性（变量）绑定起来，这个绑定的具体操作由 ViewModel 完成，我们只需要在XML文件中指定属性即可，例如将 label 控件的文本（text属性）和 Model 中的书名（title属性）绑定，XML如下：

```xml
<window v-model="book_controller">
  ...
  <label x="center" y="middle" w="50%" h="40" text="title" v-data:text="{item.title}"/>
  ...
</window>
```

其中 `v-data:text="{item.title}"` 的具体说明如下：

- v-data 表示该属性是一个数据绑定规则。
- text 表示控件的属性名称。
- item.title 表示Model中的属性，即变量名称，由于此处绑定的为复合数据，因此使用符号 "." 来表示。

> 备注：此处仅做最简单的示例，详细内容请参考 `awtk-mvvm/docs/data_binding.md`。

## 3 命令绑定

命令绑定与数据绑定类似，在AWTK-MVVM中同样通过在XML文件中添加属性完成。AWTK-MVVM中提供一个 v-on 属性，比如添加图书的指令为 "add" ，那么为按钮绑定该指令的XML如下：

```xml
<window v-model="book_controller">
  ...
<button name="btn_add" x="132" y="b:25%" w="100" h="36" v-on:click="{add}" text="Add"/>
  ...
</window>
```

其中 `v-on:click="{add}"` 的具体说明如下：

- v-on 表示该属性是一个命令绑定规则。
- click 表示绑定一个点击事件。
- add 表示绑定的命令为 add。

目前AWTK-MVVM提供的命令如下，后续还会持续更新：

| 命令                     | 说明                       |
| ------------------------ | -------------------------- |
| click                    | 点击事件                   |
| pointer\_down            | 指针按下事件               |
| pointer\_up              | 指针松开事件               |
| key\_down                | 按键按下事件               |
| key\_long\_press         | 按键长按事                 |
| key\_up                  | 按键松开事件               |
| global\_key\_down        | 全局按键按下事件           |
| global\_key\_long\_press | 全局按键长按事件           |
| global\_key\_up          | 全局按键松开事件           |
| value\_changed           | 值改变事件                 |
| value\_changed\_by\_ui   | 值（通过 UI 修改）改变事件 |

> 备注：此处仅做最简单的示例，详细内容请参考 `awtk-mvvm/docs/command_binding.md`。
