---
title: 使用匿名管道和CreateProcess隐式调用控制台程序
date: 2021-05-12
categories:
 - 其他
tags:
 - C
 - C++
 - 通信
 - Windows
---

## 1 前言

近期工作内容需要在一个程序中包装一个控制台程序，用于执行cmd命令获取结果，经过对Windows平台进程和管道通信的学习后，采用 匿名管道 和 CreateProcess 隐式调用控制台程序来实现这个需求。

## 2 核心内容

[Windows平台进程创建和退出](./windows_process.md) 记录了 CreateProcess 的基本用法。

[管道通信：匿名管道](./pipeline_anonymous.md)) 记录了匿名管道的定义及其基本用法。

本文使用匿名管道和CreateProcess隐式调用控制台程序，**核心内容**为在程序中利用 CreateProcess 创建 控制台子进程，并利用管道技术重定向控制台子程序与主进程间的标准输入输出，从而达到在程序中封装控制台的目的，用于在程序运行中执行cmd命令，并获取其结果，希望对自己与各位有所帮助。

## 3 示例代码

```cpp
#include <stdio.h>
#include <string.h>
#include <windows.h>

int main() {
	SECURITY_ATTRIBUTES sa = { 0 };
	STARTUPINFO         si = { 0 };
	PROCESS_INFORMATION pi = { 0 };
	HANDLE              hPipeOutputRead = NULL;
	HANDLE              hPipeOutputWrite = NULL;
	HANDLE              hPipeInputRead = NULL;
	HANDLE              hPipeInputWrite = NULL;
	BOOL                bTest = 0;
	DWORD               dwNumberOfBytesRead = 0;
	DWORD               dwNumberOfBytesWrite = 0;
	CHAR                szMsg[1024];
	CHAR                szBuffer[1024 * 4];

	sa.nLength = sizeof(sa);
	sa.bInheritHandle = TRUE;
	sa.lpSecurityDescriptor = NULL;

	CreatePipe(&hPipeOutputRead, &hPipeOutputWrite, &sa, 0); //创建标准输出管道
	CreatePipe(&hPipeInputRead, &hPipeInputWrite, &sa, 0);   //创建标准输入管道

	// 创建控制台子进程，进行输入输出重定向,并隐藏其窗口
	si.cb = sizeof(si);
	si.dwFlags = STARTF_USESHOWWINDOW | STARTF_USESTDHANDLES;
	si.wShowWindow = SW_HIDE;
	si.hStdInput = hPipeInputRead;
	si.hStdOutput = hPipeOutputWrite;
	si.hStdError = hPipeOutputWrite;

	CreateProcess(NULL, "cmd.exe", NULL, NULL, TRUE, 0, NULL, NULL, &si, &pi);

	// 关闭输出管道的写句柄和输入管道的读句柄
	CloseHandle(hPipeOutputWrite);
	CloseHandle(hPipeInputRead);

	// 向输入管道写入cmd命令，即执行命令
	sprintf(szMsg, "systeminfo\nexit\n");
	WriteFile(hPipeInputWrite, &szMsg, strlen(szMsg), &dwNumberOfBytesWrite, NULL);

	while (TRUE) {
		bTest = ReadFile(hPipeOutputRead, &szBuffer, sizeof(szBuffer), &dwNumberOfBytesRead, NULL);
		if (!bTest) {
			sprintf(szMsg, "Error #%d reading pipe.", GetLastError());
			MessageBox(NULL, szMsg, "WinPipe", MB_OK);
			break;
		}
		szBuffer[dwNumberOfBytesRead] = 0;  // 结尾处添加'\0'
		MessageBox(NULL, szBuffer, "WinPipe", MB_OK);
	}

	WaitForSingleObject(pi.hProcess, INFINITE); // 等待子进程结束 

	// 关闭相关句柄
	CloseHandle(pi.hThread);
	CloseHandle(pi.hProcess);
	CloseHandle(hPipeOutputRead);
	CloseHandle(hPipeInputWrite);

	return 0;
}
```
