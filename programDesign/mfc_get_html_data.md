---
title: VC++6.0 MFC 获取并解析网页数据
date: 2018-11-14
categories:
 - 程序设计
tags:
 - C++
 - 网络
 - MFC
---

## 1 前言

本文利用 `CInternetSession` 和 `CHttpFile` 从 [天气预报网站](http://qq.ip138.com/weather/guangdong/GuangZhou.htm) 获取近三天的日期、天气、温度、风向。

## 2 示例代码

```cpp
//添加头文件
#include <afxinet.h>
 
//获取网络数据
void CSensorSysDlg::GetNetworkData()
{
	UpdateData();//刷新
	CInternetSession Session("");
	CHttpFile *HttpFile=NULL;
	CString strData;
	CString strUrl; 
	CString strReturnStr="";
	CString strEdit;
	
	//获取ip138天气信息
	//获取URL
	strUrl.Format( _T("http://qq.ip138.com/weather/guangdong/GuangZhou.htm"),"广州");
	//打开URL
	HttpFile = (CHttpFile*)Session.OpenURL(strUrl,1,INTERNET_FLAG_TRANSFER_ASCII|INTERNET_FLAG_DONT_CACHE|INTERNET_FLAG_RELOAD|INTERNET_FLAG_PRAGMA_NOCACHE);
	if(HttpFile==NULL)
	{
		AfxMessageBox("无法打开网页，请检查网络连接！");
		return;
	}
	
	//将数据读取到m_ReturnStr
	while( HttpFile->ReadString( strData ) )
	{
		strReturnStr += strData;
	}
 
	UpdateData();//刷新数据
	
	strReturnStr.Delete(0,strReturnStr.Find("<tr class=\"bg5\">"));//定位到所需文本
 
	strReturnStr.Delete(0,strReturnStr.Find("h=")+8);//获得今天日期
	strEdit=strReturnStr.Left(strReturnStr.Find(" "));
	m_strDate1.Format(strEdit);
 
	strReturnStr.Delete(0,strReturnStr.Find("h=")+8);//获得明天日期
	strEdit=strReturnStr.Left(strReturnStr.Find(" "));
	m_strDate2.Format(strEdit);
 
	strReturnStr.Delete(0,strReturnStr.Find("h=")+8);//获得后天日期
	strEdit=strReturnStr.Left(strReturnStr.Find(" "));
	m_strDate3.Format(strEdit);
 
	strReturnStr.Delete(0,strReturnStr.Find("<br/>")+5);//获得今天天气
	strEdit=strReturnStr.Left(strReturnStr.Find("<"));
	m_strWeather1.Format(strEdit);
	
	strReturnStr.Delete(0,strReturnStr.Find("<br/>")+5);//获得明天天气
	strEdit=strReturnStr.Left(strReturnStr.Find("<"));
	m_strWeather2.Format(strEdit);
 
	strReturnStr.Delete(0,strReturnStr.Find("<br/>")+5);//获得后天天气
	strEdit=strReturnStr.Left(strReturnStr.Find("<"));
	m_strWeather3.Format(strEdit);
	
	strReturnStr.Delete(0,strReturnStr.Find("</tr><tr>")+13);//获得今天气温
	strEdit=strReturnStr.Left(strReturnStr.Find("</td>"));
	m_strTemperature1.Format(strEdit);	
	
	strReturnStr.Delete(0,strReturnStr.Find("<td>")+4);//获得明天气温
	strEdit=strReturnStr.Left(strReturnStr.Find("</td>"));
	m_strTemperature2.Format(strEdit);
	
	strReturnStr.Delete(0,strReturnStr.Find("<td>")+4);//获得后天气温
	strEdit=strReturnStr.Left(strReturnStr.Find("</td>"));
	m_strTemperature3.Format(strEdit);
 
	strReturnStr.Delete(0,strReturnStr.Find("</tr><tr>")+13);//获得今天风向
	strEdit=strReturnStr.Left(strReturnStr.Find("</td>"));
	m_strWind1.Format(strEdit);
 
	strReturnStr.Delete(0,strReturnStr.Find("<td>")+4);//获得明天风向
	strEdit=strReturnStr.Left(strReturnStr.Find("</td>"));
	m_strWind2.Format(strEdit);
 
	strReturnStr.Delete(0,strReturnStr.Find("<td>")+4);//获得后天风向
	strEdit=strReturnStr.Left(strReturnStr.Find("</td>"));
	m_strWind3.Format(strEdit);
	
	//释放资源
	HttpFile->Close();
	Session.Close();
    UpdateData(FALSE);//刷新对话框
}
```
