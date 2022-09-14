---
title: VC++6.0 MFC将列表控件中内容保存到EXCEL
date: 2018-11-23
categories:
 - GUI
tags:
 - C++
 - MFC
---

## 1 获取工作路径

```cpp
//获取工作路径 
CString GetWorkDir()
{
	char pFileName[MAX_PATH];   
	int nPos=GetCurrentDirectory( MAX_PATH, pFileName);   
	CString csFullPath(pFileName);
	csFullPath+="\\";
	if(nPos<0)   
		return CString("");   
	else   
		return csFullPath;   
}
```

## 2 保存文件按钮代码

```cpp
  //获取当前时间
  CTime time=CTime::GetCurrentTime();
	CString strRptTime =time.Format("%Y%m%d%H%M%S");
	// 实现存盘
	CString buff0[1024],buff1[1024],buff2[1024],buff3[1024],buff4[1024],buff5[1024],buff6[1024],buff7[1024],buff8[1024];
	CString fileName = GetWorkDir()+ strRptTime+".xls";//保存路径是工作路径5555
	CFile file(fileName, 
    CFile::modeCreate|CFile::modeReadWrite|CFile::shareExclusive);
	CString Listcol="序号\t日期\t星期\t时间\t温度(℃)\t湿度(%RH)\t光照强度(lx)\tCO2浓度(ppm)\t备注\n";//文件表头
	file.Write(Listcol,Listcol.GetLength());
	int i=0;
	int j=0;
	j=m_ListData.GetItemCount();//获取行号（m_ListData是列表控件变量）
	if(j>0)
	{
		for(i=0;i<j;i++)
		{
			buff0[i] = m_ListData.GetItemText(i,0);
			buff1[i] = m_ListData.GetItemText(i,1);
			buff2[i] = m_ListData.GetItemText(i,2);
			buff3[i] = m_ListData.GetItemText(i,3);
			buff4[i] = m_ListData.GetItemText(i,4);
			buff5[i] = m_ListData.GetItemText(i,5);
			buff6[i] = m_ListData.GetItemText(i,6);
			buff7[i] = m_ListData.GetItemText(i,7);
			buff8[i] = m_ListData.GetItemText(i,8);
			CString msg;		
msg.Format("%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t%s\t\n",
buff0[i],buff1[i],buff2[i],buff3[i],buff4[i],buff5[i],buff6[i],buff7[i],buff8[i]);
			file.Write(msg,msg.GetLength());
		}
	}
	file.Close();
	AfxMessageBox("导出文件完成！");
}
```
