---
title: SEH创建与查找
date: 2019-08-26 9:46:42
index_img: https://w.wallhaven.cc/full/5d/wallhaven-5d2p17.jpg
banner_img: https://w.wallhaven.cc/full/5d/wallhaven-5d2p17.jpg
tags:
    - SEH
    - Windbg
categories: 逆向
---



创建一个SEH处理函数

```
#include "stdafx.h"
#define WIN32_LEAN_AND_MEAN
#include <windows.h>
#include <stdio.h>
#include "stdlib.h"

DWORD scratch;

EXCEPTION_DISPOSITION exceptHandler(struct _EXCEPTION_RECORD *ExceptionRecord,
						void * EstablisherFrame,
						struct _CONTEXT *ContextRecord,
						void * DispatcherContext)
{

	unsigned i;

	MessageBox(0,_T("i am in except!!!"),0,0);

	ContextRecord->Eax = (DWORD)&scratch;//将eax的值修改为一个全局变量的地址，就可以写入了

	//printf("%08X \n ", ContextRecord->Eip);//0x4114c5

	ContextRecord->Eip += 0x2A;//这里修改的EIP，等于修改恢复异常后下一要执行的语句

	return ExceptionContinueExecution;//这是异常处理的返回结果，让程序返回执行ContextRecord->Eip的代码
}
int main()
{
	__asm
	{
		push exceptHandler // handler函数的地址
		push FS:[0] // 保存上一级的SEH链地址
		mov FS:[0],ESP // 安装新的EXECEPTION_REGISTRATION结构
	}
	__asm
	{
		mov eax, 0     // 将EAX清零
		mov[eax], 1 // 向0地址写入，会产生访问异常
	}
	printf("Yes you right!!!\n");
	__asm
	{
		mov	eax, [ESP]    // 获取前一个结构
		mov FS:[0], EAX // 恢复之前的链
		add esp, 8       // 恢复堆栈
	}
	return 0;

	printf("Hello Cray\n");
	exit(0);
}
```



![](D:\git笔记\source\_posts\20190826)

 

Wingdb 调试如图断下，地址访问异常

我来看看在内存中的什么位置

因为SHE链在线程环境块结构偏移为0的地方
```
0:000> !teb
TEB at 7ffdf000
    ExceptionList:        0012fe4c
    StackBase:            00130000
    StackLimit:           0012e000
    SubSystemTib:         00000000
    FiberData:            00001e00
    ArbitraryUserPointer: 00000000
    Self:                 7ffdf000
    EnvironmentPointer:   00000000
    ClientId:             000026a8 . 0000275c
    RpcHandle:            00000000
    Tls Storage:          7ffdf02c
    PEB Address:          7ffd9000
    LastErrorValue:       0
    LastStatusValue:      c0000139
    Count Owned Locks:    0
    HardErrorMode:        0
```
查看teb结构地址。接下来我们将这个地址与TEB结构对应

```
0:000> dt _teb 7ffdf000 .
ntdll!_TEB
   +0x000 NtTib            : 
      +0x000 ExceptionList    : 0x0012fe4c _EXCEPTION_REGISTRATION_RECORD
      +0x004 StackBase        : 0x00130000 Void
      +0x008 StackLimit       : 0x0012e000 Void
      +0x00c SubSystemTib     : (null) 
      +0x010 FiberData        : 0x00001e00 Void
      +0x010 Version          : 0x1e00
      +0x014 ArbitraryUserPointer : (null) 
      +0x018 Self             : 0x7ffdf000 _NT_TIB
   +0x01c EnvironmentPointer : 
   +0x020 ClientId         : 
      +0x000 UniqueProcess    : 0x000026a8 Void
      +0x004 UniqueThread     : 0x0000275c Void
   +0x028 ActiveRpcHandle  : 
   +0x02c ThreadLocalStoragePointer : 
   +0x030 ProcessEnvironmentBlock : 
```

可以看到`_EXCEPTION_REGISTRATION_RECORD ` 结构的地址为


0x0012fe4c 

```0:000> dt _EXCEPTION_REGISTRATION_RECORD 0012fe4c
Exceptions1!_EXCEPTION_REGISTRATION_RECORD
   +0x000 Next             : 0x0012ff70 _EXCEPTION_REGISTRATION_RECORD
   +0x004 Handler          : 0x004110a0     _EXCEPTION_DISPOSITION  Exceptions1!ILT+155(?_except_handler1YAKPAU_EXCEPTION_RECORDPAXPAU_CONTEXT+0
```
Next指向下一个处理函数，Handler指向当前SHE的处理函数

```
0:000> u 0x004110a0     
Exceptions1!ILT+155(?_except_handler1YAKPAU_EXCEPTION_RECORDPAXPAU_CONTEXT:
004110a0 e91b030000      jmp     Exceptions1!_except_handler1 (004113c0)
Exceptions1!ILT+160(__initterm):
004110a5 e984220000      jmp     Exceptions1!initterm (0041332e)
Exceptions1!ILT+165(___crtTerminateProcess):
004110aa e9cd220000      jmp     Exceptions1!_crtTerminateProcess (0041337c)
Exceptions1!ILT+170(___report_securityfailure):
004110af e99c0d0000      jmp     Exceptions1!__report_securityfailure (00411e50)
Exceptions1!ILT+175(___atonexitinit):
004110b4 e9271f0000      jmp     Exceptions1!__atonexitinit (00412fe0)
Exceptions1!ILT+180(__RTC_UninitUse):
004110b9 e962180000      jmp     Exceptions1!_RTC_UninitUse (00412920)
Exceptions1!ILT+185(___report_securityfailureEx):
004110be e99d0e0000      jmp     Exceptions1!__report_securityfailureEx (00411f60)
Exceptions1!ILT+190(__RTC_Shutdown):
004110c3 e9d8060000      jmp     Exceptions1!_RTC_Shutdown (004117a0)
```
我们看汇编代码，看看具体SHE的实现，因为我是debug版本的程序，所以由上面的跳转
```
0:000> u 4113c0 .
004113c0 55              push    ebp
004113c1 8bec            mov     ebp,esp
004113c3 81eccc000000    sub     esp,0CCh
004113c9 53              push    ebx
004113ca 56              push    esi
004113cb 57              push    edi
004113cc 8dbd34ffffff    lea     edi,[ebp-0CCh]
004113d2 b933000000      mov     ecx,33h
004113d7 b8cccccccc      mov     eax,0CCCCCCCCh
004113dc f3ab            rep stos dword ptr es:[edi]
004113de 8bf4            mov     esi,esp
004113e0 6858584100      push    offset Exceptions1!`string' (00415858)
004113e5 ff1514914100    call    dword ptr [Exceptions1!_imp__printf (00419114)]
004113eb 83c404          add     esp,4
004113ee 3bf4            cmp     esi,esp
```
这就是我们自己加的SHE处理函数
