---
title: MASH和内联MASH
date: 2019-08-22 19:11:22

index_img: https://w.wallhaven.cc/full/2k/wallhaven-2kmoxm.jpg
banner_img: https://w.wallhaven.cc/full/2k/wallhaven-2kmoxm.jpg

tags:
    - shellcode
    - 内联汇编
    - mash
categories: 编程
---


开始前一定先了解下汇编的种类`https://blog.csdn.net/ye1223/article/details/79060434`

我这里只简述我学习的**MASH汇编**和**内联的MASH格式**的汇编

### mash 汇编如下
下面程序的目的是遍历Kernel32模块的导出函数和基值，在shellcode中可以用到


```mash
.386
.model flat,stdcall
option casemap:none

; 包含printf函数所在的头文件和库文件
include msvcrt.inc ; 微软vc运行时库的头文件,
		   ; 一般包含的时c语言的各个头文件
includelib msvcrt.lib ; 包含头文件所对应的库文件

assume fs:nothing

.const ; 全局常量
g_formtStr db "%-40s      ",0
g_formtInt db "%d ",0ah,0
g_formtHex db "%08X ",0ah,0

.code


GetA proc
	LOCAL @addOfFunc;
	LOCAL @AddOfName;
	LOCAL @AddOfNaOrd;
	LOCAL @Sum;

	mov eax,fs:[48];fs表示当前线程的teb结构，eax为PEB的地址
	mov eax,[eax+12];获取这个进程的导入dll
	mov eax,[eax+28];获取PEB_LDR_DATA结构v 
	mov eax,[eax];获取第一个结构的值
	mov eax,[eax];获取第一个结构的值
	
	
	
	mov ebx,[eax+8h];ebx=dll基质
	mov eax,[ebx+3ch]
	mov eax,[eax+ebx+78h]
	add eax,ebx
	
	mov ecx,[eax+14h];//Sumfunc
	mov edx,[eax+1ch];//AddressOfFunctions
	mov esi,[eax+20h];//AddressOfNames
	mov edi,[eax+24h];//AddressOfNameOrdinals
	add edx,ebx;
	add esi,ebx;
	add edi,ebx;//edi 已经是第一个符号的地址（2字节）
	mov @addOfFunc,edx;
	mov @AddOfName,esi;
	mov @AddOfNaOrd,edi;
	;mov ecx,10;
	mov @Sum,ecx;
	
	xor ecx,ecx;清空计数器
	
LL1:
	
	
	push ecx;因为printf会影响ecx,eax,edx的值，所以只要把这个push到栈中临时保存
	
	mov eax,[esi+ecx*4];esi指向的是AddressOfNames的RVA表地址，每一个RVA都是一个DWORD 所以要*4
	add eax,ebx;ebx是这个dll的基值
	
 	push eax ; offset 伪指令能够取到一个标识符的地址
	push offset g_formtStr;输出格式
	call crt__cprintf ; 调用函数
	add esp,8;//打印Func名字 
	
	pop ecx;把偏移pop出来使用
	
	mov eax,@AddOfNaOrd;取符号表的基值
	add eax,ecx;
	add eax,ecx;这里的两个add是因为每个符号表只占一个WORD，（等于eax+ecx*2)
	mov eax,[eax];拿出这个偏移的符号值
	and eax,0FFFFh;应为eax是DWORD 而我们只要内存中的低四位，所以这样取
	push ecx;还是因为prinf会破坏ecx的值
	
 	;push eax ; offset 伪指令能够取到一个标识符的地址
	;push offset g_formtInt
	;call crt__cprintf ; 调用函数
	;add esp,8;//打印名字 
	
	mov edx,@addOfFunc;
	add edx,eax;
	add edx,eax;
	add edx,eax;
	add edx,eax;这里也和上面一样，相当于edx+eax*4,因为函数地址=BaseAddressOfFunctions+对应符号表值
	mov edx,[edx];取AddressOfFunctions指向的地址
	add edx,ebx;这个地址是RVA要加上dll的BASE
	
	;call edx;

	push edx ; offset 伪指令能够取到一个标识符的地址
	push offset g_formtHex;a
	call crt__cprintf ; 调用函数
	add esp,8;//打印名字 

	pop ecx
	add ecx ,1;
	cmp ecx,@Sum
	jne LL1
	ret
GetA endp

main:
	call GetA;
	ret 
end main
end 
```

### 内联汇编
使用这种汇编可以提高我们生产效率,完全可以写成shellcode，任何函数都能自己实现，自己调用，不需要静态导入函数，全部动态自己调用导入函数的API
	
内联函数的硬编码确实是个麻烦事，找到了如下的解决方案，在代码段写数据，然后跳过他

```c
#include "windows.h"

FARPROC LoadLibA(char *szModlePath,char* funcName)
{
	/*
	__IN szModlePath 要导入的dll地址
	__IN funcName 要查询这个dll里面的具体函数名
	__return 返回值是一个远call，保存的是这个funcName的地址
	*/
	__asm
	{
		mov eax,FS:[30h];
		mov eax,[eax+0Ch];
		mov eax,[eax+1Ch];//这个是第一个ldr_data结构指向第一个模块
		mov eax,[eax];//拿到第一个模块的门三地址  C:\Windows\system32\KERNELBASE.dll
		mov eax,[eax]//kernel32.dll
		mov ebx,[eax+08h];//GetDllBase = ebx

		// 		现在进去dll内存操作
		mov eax,[ebx+3Ch];
		mov eax,[eax+ebx+78h];
		add eax,ebx;
		mov edi,[eax+1Ch];
		add edi,ebx;     //edx = AddressOfFunctions这张表的基值(已经指向第一个无名函数了)
		//查表，LoadLibraryW在kernel32中符号位为0x341    LoadLibraryA = 0x33e
		//mov esi,341H;
		mov esi,33EH; //LoadLibraryW
		sub esi,1h;//可以不用管
		mov eax,[edi+esi*4];
		add eax,ebx;//LoadLibraryW的地址
		push szModlePath
		call eax	//eax = 获取dll的起始地址
		

		mov esi,246h;//GetProcAddress
		sub esi,1;
		mov ecx,[edi+esi*4];
		add ecx,ebx;

		jmp L1
szFuncName: _EMIT 'A'
			_EMIT 'd'
			_EMIT 'd'
			_EMIT 0x00
			//硬编码字符串在代码段
L1:
		//push offset szFuncName;
		push funcName;
		push eax;
		call ecx;//获取到funcName的地址，返回到eax中
		//call eax;//执行这个func
	}  
	 //函数返回都是eax方式返回
}

int main()
{
	FARPROC Addr =LoadLibA("user32.dll","MessageBoxA");

	__asm
	{
		push 0;
		push 0;
		push 0;
		push 0;
		call eax;
	}
	return 0;
}


```
