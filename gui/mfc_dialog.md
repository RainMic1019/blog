---
title: VC++6.0 MFC显示模态对话框和非模态对话框
date: 2018-11-21
categories:
 - GUI
tags:
 - C++
 - MFC
---

## 1 模态对话框

```cpp
#include "AddDataDlg.h"//添加头文件
 
CAddDataDlg AddData_Dialog;//在头文件中定义对话框对象（CAddDataDlg为该对话框对应的类）
 
int nReturn = AddData_Dialog.DoModal();//在源文件函数中显示模态对话框,将返回值赋给nReturn
if (nReturn ==IDCANCEL)//整形的结果如果是取消
{
  return FALSE;//就返回FALSE，对于当前这个按钮按下事件的处理结束。
}
if (nReturn == IDOK)//整形的结果如果是确定
{
  return TRUE;//就返回TRUE，对于当前这个按钮按下事件的处理结束。
｝
```

## 2 非模态对话框

在主对话框类头文件中定义指针：

```cpp
#include "SDataDLG.h"//添加头文件
CSDataDLG *m_SelectData_Dialog;//定义对话框指针（CSDataDLG为对话框所对应的类）
```

在主对话框类的构造里面初始化指针：

```cpp
m_SelectData_Dialog=NULL;//初始化对话框指针
```

在显示对话框函数中添加显示代码：

```cpp
//判定对话框是否有所指向
if (m_SelectData_Dialog == NULL)
{
  m_SelectData_Dialog = new CSDataDLG();//指向一个非模对话框示例
  m_SelectData_Dialog->Create(IDD_Select_DIALOG, this);//创建
}
m_SelectData_Dialog->ShowWindow(SW_SHOW);//显示
```

为主对话框类添加析构函数：

```cpp
//在头文件中的构造函数下方添加
~ CSensorSysDlg();//析构函数

//在源文件中的构造函数下方添加
//析构函数
CSensorSysDlg::~CSensorSysDlg() 
{
}
```

在析构函数中对指针进行析构：

```cpp
//析构函数
CSensorSysDlg::~CSensorSysDlg() 
{
  //析构非模态对话框
  if (m_SelectData_Dialog != NULL)
  {
    delete m_SelectData_Dialog;
    m_SelectData_Dialog = NULL;
  }
}
```
