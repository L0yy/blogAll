---
title: Com组建检测虚拟沙箱
index_img: https://w.wallhaven.cc/full/2k/wallhaven-2kmoxm.jpg
banner_img: https://w.wallhaven.cc/full/2k/wallhaven-2kmoxm.jpg
date: 2019-09-04 19:11:22
tags:
    - 检测沙箱
categories: 逆向
---


因为沙箱的仿真度不全问题，可能造成仿真系统上的音频设备功能与真机的差异，通过这来实现检测是否在沙箱中运行。

但经测试 `any.run` 和`微步` 都未被识别为沙箱

这里只是介绍下可以使用COM对系统基础设备的操作，来检测虚拟沙箱。
```c
#include "stdafx.h"
#include <conio.h>
#include <windows.h>
#include <dshow.h>
#include <objbase.h>

#pragma comment(lib, "Strmiids.lib")

void useCom()
{
	/*
	这些只是一些随机检查，以确保恶意软件在真实系统上执行。
	只有安装了音频设备，此处的沙箱才会被视为真实系统。
	大多数仿真器都会失败，因为几乎不可能为现代操作系统中的每个COM接口实现适当的支持。
	*/
	CoInitialize(0);
	wchar_t * filerName = L"random_name";
	IGraphBuilder * pGraph;
	CoCreateInstance(CLSID_FilterGraph, NULL, CLSCTX_INPROC_SERVER, IID_IGraphBuilder, (void**)&pGraph);
	if (E_POINTER != pGraph->AddFilter(NULL, filerName))
	{
		MessageBox(0,L"E_POINTER != pGraph->AddFilter",0,0);
		ExitProcess(-1);
	}

	//对一个简单的音频渲染器进行硝化，不检查错误代码，但是失败后pBaseFilter将被设置为NULL
	IBaseFilter *pBaseFile;
	CoCreateInstance(CLSID_AudioRender, NULL, CLSCTX_INPROC_SERVER, IID_IBaseFilter, (void**)&pBaseFile);

	//试图找到刚刚添加的过滤器;如果以前未检查任何错误（或错误的仿真），此功能将无法找到过滤器，并且将成功检测到沙箱/仿真器。
	pGraph->AddFilter(pBaseFile, filerName);
	IBaseFilter *pBaseFile2;
	pGraph->FindFilterByName(filerName, &pBaseFile2);

	if (pBaseFile2 == NULL)
	{
		MessageBox(0,L"pBaseFile2==Null!!!",0,0);
		ExitProcess(1);
	}
	//检查achName是不是之前添加的过滤器名
	FILTER_INFO info = { 0 };
	pBaseFile2->QueryFilterInfo(&info);
	if (wcscmp(info.achName,filerName)!=0)
	{
		MessageBox(0,L"pBaseFile2 AddFilter error",0,0);
		exit(0);
	}

	IReferenceClock *pClock;
	if (pBaseFile2->GetSyncSource(&pClock))
	{
		MessageBox(0,L"pBaseFile2->GetSyncSource",0,0);
		exit(0);
	}
	if (pClock != NULL)
	{
		exit(0);
	}
	CLSID clsID;
	pBaseFile2->GetClassID(&clsID);
	if (clsID.Data1 == 0)
	{
		MessageBox(0,L"pBaseFile2->GetClassID",0,0);
		exit(1);
	}
	if (pBaseFile2 ==NULL)
	{
		MessageBox(0,L"pBaseFile2 ==NULL",0,0);
		exit(1);
	}

	IEnumPins *pEnum = NULL;
	if (pBaseFile2->EnumPins(&pEnum)!=0)
	{
		MessageBox(0,L"pBaseFile2->EnumPins",0,0);
		exit(-1);
	}
	//AddRef返回的引用计数必须大于0
	if (pBaseFile2->AddRef() == 0)
	{
		MessageBox(0,L"pBaseFile2->AddRef()",0,0);
		exit(-1);
	}
}

int _tmain(int argc, _TCHAR* argv[])
{
	useCom();
	MessageBox(0,L"没有沙箱!!!\n",0,0);

	return 0;
}
```
IDA反编译结果

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190905142026524.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5eUFyYXk=,size_16,color_FFFFFF,t_70)
