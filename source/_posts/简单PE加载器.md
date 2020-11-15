---
title: 简单PE加载器
tags: [逆向]
index_img: https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191216121346.png
banner_img: https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191216121346.png
date: 2019-12-07 21:28:01
---


思路来源写`Sality`感染型病毒专杀时指令被严重混淆，通过加载PE 修改内存 跑一下解密算法效率是最高的。



很多病毒在运行的时候都会加载另一个主映像文件去执行，而不是创建进程，就很有意思

下面就是如何加载一个PE，再展开，最后修复执行的过程 

该函数主要是为了将文件映射到内存中，保证源程序安全

返回值是未展开文件在内存中的位置

```C
LPBYTE LoadFileToMem(LPCSTR lpFilePath)
{
	//////////////////////////////////////////////////////////////////////////
	////将源文件读到内存中                                                  ///
	//////////////////////////////////////////////////////////////////////////
	DWORD FileSize = 0;
	LPBYTE Buff = NULL;

	HANDLE hFile = CreateFileA(lpFilePath, GENERIC_READ, FILE_SHARE_READ | FILE_SHARE_WRITE, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
	if (hFile == INVALID_HANDLE_VALUE)
	{
		printf("打开文件句柄错误！[%x]", GetLastError());
		return -1;
	}

	FileSize = GetFileSize(hFile, NULL);

	Buff = (LPBYTE)malloc(FileSize);
	if (Buff == NULL)
	{
		printf("空间申请失败![%x]", GetLastError());
		return -1;
	}

	if (!ReadFile(hFile, Buff, FileSize, &FileSize, NULL))
	{
		printf("ReadFile![%x]", GetLastError());
		return -1;
	}
	return Buff;
}
```



接下来按照各个节的对齐粒度展开
返回值是展开后什么都没修复的buff指针

```C
LPBYTE Extension(LPBYTE lpFileBuffer)
{
	//////////////////////////////////////////////////////////////////////////
	////将文件在内存中展开                                                  ///
	//////////////////////////////////////////////////////////////////////////
	int i = 0; 
	PIMAGE_DOS_HEADER pDos = (PIMAGE_DOS_HEADER)lpFileBuffer;
	PIMAGE_NT_HEADERS pNt = (PIMAGE_NT_HEADERS)(lpFileBuffer + pDos->e_lfanew);
	PIMAGE_SECTION_HEADER pSec = (PIMAGE_SECTION_HEADER)((LPBYTE)pNt + sizeof(IMAGE_NT_HEADERS));

	DWORD ImageSize = pNt->OptionalHeader.SizeOfImage;

	//LPBYTE lpMemBuffer = (LPBYTE)malloc(ImageSize);
	LPVOID lpMemBuffer = VirtualAlloc(NULL, ImageSize, MEM_COMMIT, PAGE_EXECUTE_READWRITE);

	VirtualProtect(lpMemBuffer, ImageSize, PAGE_EXECUTE_READWRITE, NULL);//这一句可以不要，上面申请的就是可读可写可执行的空间。

	ZeroMemory(lpMemBuffer, ImageSize);

	//文件头的大小
	DWORD dwSizeOfHeader = pNt->OptionalHeader.SizeOfHeaders;

	//将头部拷贝过去
	CopyMemory(lpMemBuffer, lpFileBuffer, dwSizeOfHeader);


	for (;i < pNt->FileHeader.NumberOfSections;i++)
	{
		if (pSec->VirtualAddress == 0 || pSec->PointerToRawData == 0)
		{
			pSec++;
			continue;
		}
		CopyMemory((LPBYTE)lpMemBuffer + pSec->VirtualAddress, lpFileBuffer + pSec->PointerToRawData, pSec->SizeOfRawData);
		pSec++;
	}

	//已经完全映射，可以把之前的内存释放掉了
	free(lpFileBuffer);
	return lpMemBuffer;
}
```

修复重定位信息

这一步容易出错，核心原理是重定位表中存的是这个程序需要修复的数据，每个数据都是`DWORD`类型的
可以参考如下地址，主要要注意 `pReloca->VirtualAddress`存的是页基质 , `pReloca->SizeOfBlock` 包含了`IMAGE_BASE_RELOCATION` 结构的大小
https://blog.csdn.net/Apollon_krj/article/details/77370452

```C
BOOL ReRloc(LPBYTE lpMemBuffer)
{
	//////////////////////////////////////////////////////////////////////////
	////修复重定位表                                                       ///
	////原理：遍历重定位表，计算需要重定位数据的地址：重定位后的地址 = 需要重定位的地址 - 默认加载基址 + 当前加载基址
	//////////////////////////////////////////////////////////////////////////
	PIMAGE_DOS_HEADER pDos = (PIMAGE_DOS_HEADER)lpMemBuffer;
	PIMAGE_NT_HEADERS pNt = (PIMAGE_NT_HEADERS)(lpMemBuffer + pDos->e_lfanew);
	//获得重定位表
	PIMAGE_BASE_RELOCATION pReloca = (PIMAGE_BASE_RELOCATION)(lpMemBuffer + pNt->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC].VirtualAddress);
	
	//如果重定位表为空，上述表达式为pDos+0
	if ((LPBYTE)pReloca == lpMemBuffer)
	{
		printf("没有重定位表！\n");
		return TRUE;
	}

	while (pReloca->VirtualAddress !=0 && pReloca->SizeOfBlock !=0 )
	{
		LPWORD pRelData =  (LPBYTE)pReloca + sizeof(IMAGE_BASE_RELOCATION);
		int nNumRel = (pReloca->SizeOfBlock - sizeof(IMAGE_BASE_RELOCATION)) / sizeof(WORD);
		for (int i = 0; i < nNumRel; i++)
		{
			// 每个WORD由两部分组成。高4位指出了重定位的类型，WINNT.H中的一系列IMAGE_REL_BASED_xxx定义了重定位类型的取值。
			// 低12位是相对于VirtualAddress域的偏移，指出了必须进行重定位的位置。

			if ((WORD)(pRelData[i] & 0xF000) == 0x3000) //这是一个需要修正的地址
			{
				//pReloca->VirtualAddress存的是页基质，(一个页4K，所以是0xFFF，刚好12位)
				LPDWORD pAddress = (LPDWORD)(lpMemBuffer + pReloca->VirtualAddress + (pRelData[i] & 0x0FFF));


				*pAddress = *pAddress - pNt->OptionalHeader.ImageBase + (DWORD)pDos;

				printf("Check!");
				//DWORD dwDelta = (DWORD)pDos - pNt->OptionalHeader.ImageBase;
				//*pAddress += dwDelta;
			}
		}
		pReloca = (LPBYTE)pReloca + pReloca->SizeOfBlock;
	}
	printf("重定位表修复完成！\n");
	return TRUE;
}

```

修复IAT 这一步也是必须的，在很多壳中是对IAT表进行了Hook，了解一下结构

`WinNt.h`中定义的`IMAGE_IMPORT_DESCRIPTOR`结构
```C
typedef struct _IMAGE_IMPORT_DESCRIPTOR {
    union {                                 //注意这是union
        DWORD   Characteristics;            // 0 for terminating null import descriptor
        DWORD   OriginalFirstThunk;         // RVA to original unbound IAT (PIMAGE_THUNK_DATA)
    } DUMMYUNIONNAME;
    DWORD   TimeDateStamp;                  // 0 if not bound,
                                            // -1 if bound, and real date\time stamp
                                            //     in IMAGE_DIRECTORY_ENTRY_BOUND_IMPORT (new BIND)
                                            // O.W. date/time stamp of DLL bound to (Old BIND)

    DWORD   ForwarderChain;                 // -1 if no forwarders
    DWORD   Name;
    DWORD   FirstThunk;                     // RVA to IAT (if bound this IAT has actual addresses)
} IMAGE_IMPORT_DESCRIPTOR;
typedef IMAGE_IMPORT_DESCRIPTOR UNALIGNED *PIMAGE_IMPORT_DESCRIPTOR;
```

`OriginalFirstThunk` 和 `FirstThunk` 都指向一个 `IMAGE_THUNK_DATA32` 结构，该结构是以`0` 结尾

`OriginalFirstThunk` 是一直不会被修改，程序构建好后就固定 INT
`FirstThunk` 在程序加载时动态修改为具体的函数地址，也就是我们常说的IAT

```C
typedef struct _IMAGE_THUNK_DATA32 {
    union {
        DWORD ForwarderString;      // PBYTE 
        DWORD Function;             // PDWORD
        DWORD Ordinal;
        DWORD AddressOfData;        // PIMAGE_IMPORT_BY_NAME
    } u1;
} IMAGE_THUNK_DATA32;
typedef IMAGE_THUNK_DATA32 * PIMAGE_THUNK_DATA32;
```

根据Ordinal的值，判断是按序号导入还是按名称导入，如果是按名称导入则需要去`AddressOfData`指向的`IMAGE_IMPORT_BY_NAME`结构中去拿到导入函数名
```C
typedef struct _IMAGE_IMPORT_BY_NAME {
    WORD    Hint;
    CHAR   Name[1];                 //保存具体导入函数的名称
} IMAGE_IMPORT_BY_NAME, *PIMAGE_IMPORT_BY_NAME;
```

如果是序号导入就根据`Ordinal`的`低16`位决定

https://www.cnblogs.com/night-ride-depart/p/5776107.html


```C
BOOL InitIAT(LPBYTE lpMemBuffer)
{
	//////////////////////////////////////////////////////////////////////////
	////修复IAT                                                            
	//////////////////////////////////////////////////////////////////////////
	PIMAGE_DOS_HEADER pDos = (PIMAGE_DOS_HEADER)lpMemBuffer;
	PIMAGE_NT_HEADERS pNt = (PIMAGE_NT_HEADERS)(lpMemBuffer + pDos->e_lfanew);
	PIMAGE_IMPORT_DESCRIPTOR pImportTalbe = (PIMAGE_IMPORT_DESCRIPTOR)(lpMemBuffer + pNt->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].VirtualAddress);
	LPCSTR szDllname = NULL;
	PIMAGE_THUNK_DATA lpOrgNameArry = NULL;
	PIMAGE_THUNK_DATA lpFirNameArry = NULL;
	PIMAGE_IMPORT_BY_NAME lpImportByNameTable = NULL;
	HMODULE hMou;
	FARPROC Funaddr;
	int i = 0;

	while (pImportTalbe->OriginalFirstThunk)
	{
		szDllname = lpMemBuffer + pImportTalbe->Name;
		hMou = GetModuleHandleA(szDllname);
		if (hMou == NULL)
		{
			hMou = LoadLibraryA(szDllname);
			if (hMou == NULL)
			{
				printf("加载%s失败！[%x]\n ", szDllname, GetLastError());
				return FALSE;
			}
		}

		//dll加载成功，开始导入需要的函数
		lpOrgNameArry = (PIMAGE_THUNK_DATA)(lpMemBuffer + pImportTalbe->OriginalFirstThunk);

		lpFirNameArry = (PIMAGE_THUNK_DATA)(lpMemBuffer + pImportTalbe->FirstThunk);

		i = 0;

		while (lpOrgNameArry[i].u1.AddressOfData)
		{
			lpImportByNameTable = (PIMAGE_IMPORT_BY_NAME)(lpMemBuffer + lpOrgNameArry[i].u1.AddressOfData);

			if (lpOrgNameArry[i].u1.Ordinal & 0x80000000 == 1)
			{
				//序号导入
				Funaddr = GetProcAddress(hMou, (LPSTR)(lpOrgNameArry[i].u1.Ordinal & 0xFFFF));
			}
			else
			{
				//名称导入
				Funaddr = GetProcAddress(hMou, lpImportByNameTable->Name);
			}

			lpFirNameArry[i].u1.Function = Funaddr;
			i++;
		}
		pImportTalbe++;
	}
	return TRUE;
}
```

最后就是修复`ImageBase`

```C
FARPROC InitEnv(LPBYTE lpMemBuffer)
{
	//////////////////////////////////////////////////////////////////////////
	////修改ImageBase，返回入口点                                           ///
	//////////////////////////////////////////////////////////////////////////
	PIMAGE_DOS_HEADER pDos = (PIMAGE_DOS_HEADER)lpMemBuffer;
	PIMAGE_NT_HEADERS pNt = (PIMAGE_NT_HEADERS)(lpMemBuffer + pDos->e_lfanew);
	pNt->OptionalHeader.ImageBase = lpMemBuffer;
	
	return lpMemBuffer + pNt->OptionalHeader.AddressOfEntryPoint;
}
```

返回这个被加载程序的入口地址，直接调用就好



吃水不忘挖井人 参考来源
https://bbs.pediy.com/thread-249133.htm

