---
title: 管道通信：匿名管道
date: 2021-05-09
categories:
 - 其他
tags:
 - C
 - C++
 - 通信
 - Windows
---

## 1 前言

上篇笔记（[管道通信：输入输出重定向](./pipeline_io_redirection.md）记录了输入输出重定向的实现方法，本文总结一下匿名管道的内容，希望对自己与各位有所帮助。

## 2 管道的概念

简单理解，计算机中的管道（pipe）类似现实世界中的水管，从一端放入水流，另一端就会流出来。回到计算机中，这个水流就是数据。

管道（pipe）又分为 **匿名管道** 和 **命名管道**。本文主要介绍如何使用匿名管道来实现输入输出重定向功能。

## 3 匿名管道的创建和使用

### 3.1 函数原型

#### 3.1.1 CreatePipe

```cpp
/* 创建管道 */
BOOL WINAPI CreatePipe(
  PHANDLE hReadPipe,                      // 管道输出（读取）端句柄
  PHANDLE hWritePipe,                     // 管道输入（写入）端句柄
  LPSECURITY_ATTRIBUTES lpPipeAttributes, // 管道的安全属性
  DWORDnSize                              // 管道缓冲区容量，设置0时使用默认大小
);
```

管道的安全属性通常设置如下：

```cpp
SECURITY_ATTRIBUTES sa = {0};
sa.nLength = sizeof(sa);
sa.bInheritHandle = TRUE;
sa.lpSecurityDescriptor = NULL;
```

> 注意：不能在管道的输出端写入数据，也不能在管道的输入端读取数据。

#### 3.1.2 ReadFile

```cpp
/* 从管道中读取数据 */
BOOL ReadFile(
  HANDLE hFile,                // 管道输出端（读取）句柄
  LPVOID lpBuffer,             // 数据缓冲区指针，读取的数据将写在该缓冲区中
  DWORD nNumberOfBytesToRead,  // 指定读取的字节数
  LPDWORD lpNumberOfBytesRead, // 返回实际读取到的字节数
  LPOVERLAPPED lpOverlapped    // 用于异步操作，一般传入NULL即可
);
```

#### 3.1.3 WriteFile

```cpp
/* 向管道写入数据 */
BOOL WriteFile(
  HANDLE hFile,                   // 管道输入端（写入）句柄，也可以是CreateFile()接口创建的文件句柄
  LPVOID lpBuffer,                // 待写入管道的数据缓冲区指针
  DWORD nNumberOfBytesToWrite,    // 指定写入的字节数
  LPDWORD lpNumberOfBytesWritten, // 返回实际写入管道的字节数
  LPOVERLAPPED lpOverlapped       // 用于异步操作，一般传入NULL即可
);
```

#### 3.1.4 CloseHandle

```cpp
/* 关闭管道端口句柄 */
BOOL CloseHandle(
  HANDLE hObject  // 想要关闭的句柄
);
```

> 注意：当管道输入输出端都被关闭后，系统会自动关闭该管道，并回收相关资源。

### 3.2 管道使用技巧

从以上管道相关的函数原型来看，管道的读写其实和文件的读写非常类似，实际上管道也是一种特殊的文件——内存映射文件。

使用管道时需要注意：读取和写入数据时，一定要注意顺序，如果管道中没有数据，调用ReadFile()会造成阻塞，直到有其它线程将数据写入管道。同样，当有线程正在管道中读取数据时，其它试图将数据写入管道的的线程也会被阻塞，更多内容请查阅MSDN。

### 3.3 示例代码

此处以[管道通信：输入输出重定向](./pipeline_io_redirection.md)中的示例程序作为子进程，在父进程中创建两个管道（输入/输出 管道），分别用于存放输入输出数据，将这两个管道重定向到输入输出文件（stdin.txt 和 stdout.txt）。

此时，在父进程中启动子进程，将子进程的标准输入输出改成从上述创建的 输入/输出 管道句柄，即实现了子进程的输入输出重定向，示例代码如下：

```cpp
#include <windows.h>
#include <stdio.h>

int main()
{
	char sz[3][50] = { "示例程序.exe", "infile.txt", "outfile.txt" };
	HANDLE hPipeInputRead = NULL;
	HANDLE hPipeInputWrite = NULL;
	HANDLE hPipeOutputRead = NULL;
	HANDLE hPipeOutputWrite = NULL;

	SECURITY_ATTRIBUTES sa;
	sa.nLength = sizeof(sa);
	sa.bInheritHandle = TRUE;
	sa.lpSecurityDescriptor = NULL;

	CreatePipe(&hPipeInputRead, &hPipeInputWrite, &sa, 0);   // 创建输入管道
	CreatePipe(&hPipeOutputRead, &hPipeOutputWrite, &sa, 0); // 创建输出管道

	HANDLE hInputFile = CreateFile(L"stdin.txt", GENERIC_READ, 0, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_READONLY, NULL);
	HANDLE hOutputFile = CreateFile(L"stdout.txt", GENERIC_READ | GENERIC_WRITE, FILE_SHARE_WRITE, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);

	CHAR Buffer[1024 * 4] = { 0 };
	DWORD dwNumberOfBytesRead = 0;
	DWORD dwNumberOfBytesWritten = 0;
	BOOL  bRet =  FALSE;

	while (TRUE) { // 从文件stdin.txt中读取数据,写到输入管道中
		bRet = ReadFile(hInputFile, Buffer, sizeof(Buffer), &dwNumberOfBytesRead, NULL);
		if (!bRet || dwNumberOfBytesRead == 0)
			break;
		bRet = WriteFile(hPipeInputWrite, Buffer, dwNumberOfBytesRead, &dwNumberOfBytesWritten, NULL);
		if (!bRet)
			break;
	}
	// 关闭输入管道的写句柄
	CloseHandle(hInputFile);
	CloseHandle(hPipeInputWrite);

	// 启动demo.exe作为子进程
	STARTUPINFO si;
	si.cb = sizeof(si);
	GetStartupInfo(&si);
	si.hStdInput = hPipeInputRead;    // 输入由标准输入 -> 从管道中读取
	si.hStdOutput = hPipeOutputWrite; // 输出由标准输出 -> 输出到管道
	si.wShowWindow = SW_HIDE;
	si.dwFlags = STARTF_USESHOWWINDOW | STARTF_USESTDHANDLES;
	PROCESS_INFORMATION pi;

	CreateProcess(L"demo.exe", NULL, NULL, NULL, TRUE, 0, NULL, NULL, &si, &pi);
	WaitForSingleObject(pi.hProcess, INFINITE);

	// 关闭进程相关句柄
	CloseHandle(pi.hThread);
	CloseHandle(pi.hProcess);
	CloseHandle(hPipeInputRead);
	CloseHandle(hPipeOutputWrite);

	while (TRUE) { // 从输出管道中读取数据,写到文件stdout.txt中
		bRet = ReadFile(hPipeOutputRead, Buffer, sizeof(Buffer), &dwNumberOfBytesRead, NULL);
		if (!bRet || dwNumberOfBytesRead == 0)
			break;
		bRet = WriteFile(hOutputFile, Buffer, dwNumberOfBytesRead, &dwNumberOfBytesWritten, NULL);
		if (!bRet)
			break;
	}

	//关闭输出管道的读句柄
	CloseHandle(hOutputFile);
	CloseHandle(hPipeOutputRead);

	return 0;
}
```

## 4 总结
从上述示例来看，匿名管道可以在不修改程序源码的情况下实现输入输出重定向，也就是说只要有exe文件，即可实现输入输出重定向，并且上述示例中，父进程也可以使用 输入/输出 管道写入或读取数据，即实现了父子进程间的通信。
