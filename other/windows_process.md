---
title: Windows平台进程创建和退出
date: 2021-05-01
categories:
 - 其他
tags:
 - C++
 - Windows
---

## 1 前言

近期工作内容需要研究 Windows 平台的进程，因此将学习的内容简单整理一下，希望对自己与各位有所帮助。

## 2 进程的概念

进程是**资源分配的基本单位**，也是**独立运行的基本单位**。通俗讲就是一段程序执行的过程。

进程由两部分构成，一部分指一个**内核对象**，操作系统用它来管理进程，也是系统保存进程统计信息的地方。另一部分由**地址空间**构成，包括文本区、数据区、堆栈区。文本区存储处理器执行的代码，数据区存储变量和进程执行期间使用的的动态内存分配，堆栈区存储活动过程中调用的指令和本地变量。

## 3 进程的三种状态

进程在运行过程中不断改变其运行状态，通常情况下，一个运行进程必须具有以下三种状态：

1. **就绪（Ready)**：当进程分配到除了CPU以外的所有必要资源，只要处理器分配资源就能立马执行时的状态。
2. **运行(Running)**：当进程已经获得处理器的资源分配，进程正在运行的状态。
3. **阻塞(Blockjer)**：正在执行的进程，由于等待某个事件发生而无法执行时，便放弃资源分配而处于阻塞状态，例如等待I/O完成，申请缓冲区不能满足，等待信号等等。

## 4 进程的创建

Windows平台创建进程的常用方法：

1. Winexec()；
2. ShellExcute()；
3. CreateProcess()。

### 4.1 Winexec

#### 4.1.1 函数原型

```cpp
UINT WinExec( 
LPCSTR lpCmdLine,  // 路径名（可以带cmd命令行）
UINT uCmdShow      // 显示状态);
```

如果 lpCmdLine 中可执行文件的名称不包含目录路径，则系统将按以下顺序搜索可执行文件的路径：

1. 加载应用程序的目录；
2. 当前目录；
3. Windows系统目录：GetSystemDirectory()接口获取此目录的路径；
4. Windows目录：GetWindowsDirectory()接口获取此目录的路径；
5. PATH环境变量中列出的目录；

#### 4.1.2 示例代码

1. 打开进程：

```cpp
UINT nRet = ::WinExec(
    "C:\\Users\\shadow\\Desktop\\depends.exe",  // 进程路径名
    SW_SHOW     // 显示状态
);
```

Winexec()接口还可以启动 cmd，包括带参参数启动，比如：

（1）/k 并显示控制台窗口；
（2）/c  执行命令并关闭控制台窗口。

2. 打开注册表：

```cpp
UINT nRet = ::WinExec("cmd.exe /c regedit", SW_SHOW);
```

3. 修改uac权限：

```cpp
UINT nRet = ::WinExec("cmd /k %windir%\\System32\\reg.exe ADD HKLM\\SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Policies\\System /v EnableLUA /t REG_DWORD /d 0 /f",SW_SHOW);
```

### 4.2 ShellExcute

#### 4.2.1 函数原型

```cpp
HINSTANCE ShellExecute(
      HWND hwnd,            // 指定窗口的句柄
      LPCTSTR lpOperation,  // 指定要进行的操作
      LPCTSTR lpFile,       // 文件路径
      LPCTSTR lpParameters, // 若lpFile是可执行文件，则此参数指定命令行参数
      LPCTSTR lpDirectory,  // 指定默认目录
      INT nShowCmd          // 指定程序窗口初始化显示方式
   );
```

#### 4.2.2 示例代码

```cpp
/* 打开进程 */
ShellExecute(NULL, "open","C:\\Users\\shadow\\Desktop\\depends.exe",NULL, NULL,SW_SHOWNORMAL);
 
/* 打开网址 */
ShellExecute(NULL, "open", "www.baidu.com/s?wd=\"拼音\"", NULL, NULL,SW_SHOW);

/* 浏览文件夹 */
ShellExecute(NULL, "explore", "D:\\360", NULL, NULL, SW_SHOWNORMAL);

/* 打印指定文件 */
ShellExecute(NULL, "print", "C:\\Users\\shadow\\Desktop\\t.txt", NULL, NULL, SW_HIDE);

/* 自动搜索exe路径 */
ShellExecute(NULL, "open", "cmd.exe", "/k dir", NULL, SW_SHOWNORMAL);    

```

### 4.3 CreateProcess

#### 4.3.1 函数原型

```cpp
BOOL CreateProcess
(
LPCTSTR lpApplicationName,                 // 应用程序名称（路径）
LPTSTR lpCommandLine,                      // 命令行字符串
LPSECURITY_ATTRIBUTES lpProcessAttributes，// 进程安全性
LPSECURITY_ATTRIBUTES lpThreadAttributes,  // 线程安全性
BOOL bInheritHandles,                      // 是否继承父进程属性
DWORD dwCreationFlags,                     // 创建标志符，比如CREATE_NEW_CONSOLE表示创建一个新的控制台，具体请查阅MSDN
LPVOID lpEnvironment,                      // 指向新的环境块的指针，指向一个新进程的环境块，如果该参数为空，新进程使用调用进程环境
LPCTSTR lpCurrentDirectory,                // 指向一个以NULL结尾的字符串，该字符串用来指定子进程的工作路径。
LPSTARTUPINFO lpStartupInfo,               // 传递给新进程的信息
LPPROCESS_INFORMATION lpProcessInformation // 新进程返回的信息
);
```

#### 4.3.2 示例代码

```cpp
void CProcessDlg::OnBnClickedButton() {
    STARTUPINFO si = { 0 };
    si.cb = sizeof(si);
    PROCESS_INFORMATION pi = { 0 };
 
    BOOL bRet = ::CreateProcess(
        "C:\\Users\\shadow\\Desktop\\depends.exe",
        NULL, //命令行参数
        NULL,
        NULL,
        FALSE,
        0,    //没有别的需求，创建标志填0
        NULL, //环境块
        NULL, //当前目录
        &si,  //启动进程一些配置
        &pi
    );
 
    if (!bRet) {
        AfxMessageBox(_T("Create Process FAIL!"));
    }
}
```

## 5 进程的退出

1. ExitProcess(0);执行后直接退出进程，后边代码将无法执行；
1. TerminateProcess （不推荐使用，因为目标进程没有机会清理资源）；
2. 进程中的所有线程自行终止运行（这种情况几乎从未发生）；
3. 主线程的入口函数返回（最好使用这个方法)；
