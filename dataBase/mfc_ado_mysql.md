---
title: VC++6.0 MFC利用ADO连接到MySQL数据库
date: 2018-11-04
categories:
 - 数据库
tags:
 - C++
 - MFC
 - SQL
 - 数据库
---

## 1 前言

安装MySQL数据库，并为其安装驱动！

两种连接方式：ODBC连接和非ODBC连接。

## 2 ODBC方式链接

### 2.1 导入动态库

在应用程序的 `stdafx.h` 头文件中（也可以在其他合适的地方）包含如下语句。

```cpp
//导入msado15.dll动态链接库，不要命名空间，将EOR改成adoEOR，避免与文件结尾冲突
#import "C://program files//common files//system//ado//msado15.dll" no_namespace rename ("EOF", "adoEOF")
```

### 2.2 链接数据库

```cpp
CoInitialize(NULL); //初始化Com组件
_ConnectionPtr conPtr; //数据库链接指针(可在头文件中定义)
//对conPtr初始化
//conPtr.CreateInstance("ADODB.Connection"); //Connection用于与数据库服务器的链接方法一
conPtr.CreateInstance(__uuidof(Connection)); //Connection用于与数据库服务器的链接方法二
 
try
{
    conPtr->ConnectionTimeout = 5; //设置连接时间
    //MySql为数据源名 localhost表示本地 root表示用户名 123表示密码
 
    //链接方式1，使用这种方式链接时要注意在设置数据源时一定要选择一个数据库(测试成功)
    //conPtr->Open("DSN=MySql;server=localhost;","root","123",adModeUnknown);
 
    //链接方式2，采用这种方式链接时在创建数据源时没有必要选择一个数据库(测试成功)
    conPtr->Open("DSN=MySql;server=localhost;database=test;","root","123",adModeUnknown);
}
catch(_com_error e) //捕捉异常
{
    AfxMessageBox(e.ErrorMessage());
}
CoUninitialize(); //释放com组件(不知是否有必要)
```

### 2.3 访问数据库

```cpp
RecordsetPtr recordPtr;//数据集指针（可在头文件中定义）
//对recordPtr初始化
recordPtr.CreateInstance(__uuidof(Recordset));

try	
{
    //number是int类型的字段
	m_pRec=m_pCon->Execute("select number from data",NULL,adCmdText);
}
catch (_com_error e)	
{	
	AfxMessageBox(e.Description());	
}
while (!(m_pRec->adoEOF))	
{   //获取记录
	CString number=(LPCSTR)(_bstr_t)m_pRec->GetCollect("number");
	AfxMessageBox(number);
	m_pRec->MoveNext();
}
m_pRec->Close();//关闭记录集
m_pRec.Release();//释放空间
m_pCon->Close();//关闭连接
m_pCon.Release();//释放空间
```

## 3 非ODBC方式链接

### 3.1 链接数据库

```cpp
CoInitialize(NULL); //初始化Com组件
_ConnectionPtr conPtr; //数据库链接指针
//对conPtr初始化
//conPtr.CreateInstance("ADODB.Connection"); //Connection用于与数据库服务器的链接方法一
conPtr.CreateInstance(__uuidof(Connection)); //Connection用于与数据库服务器的链接方法二
 
//MySQL ODBC 5.1 Driver为驱动名称（取决于我们为MySql安装的驱动），localhost为服务器地址，test为数据库名，root为用户名（MySql默认用户名为root），123为密码
CString conStr=_T("Driver={ MySQL ODBC 5.1 Driver };Server= localhost;Database=test;");
 
try
{
    conPtr->ConnectionTimeout = 5; //设置连接时间
    conPtr->Open((_bstr_t)conStr, _T("root"),_T("123"), adModeUnknown);
}
catch(_com_error e) //捕捉异常
{
    AfxMessageBox(e.ErrorMessage());
}
CoUninitialize(); //释放com组件
```

### 3.2 访问数据库

获取记录集的方式与ODBC方式相同。
