---
title: udis86和capstone基本使用
tags: [汇编]
index_img: https://gitee.com//cve/BlogImg/raw/master/typora/timg (1).jpg
banner_img: https://gitee.com//cve/BlogImg/raw/master/typora/timg (1).jpg
---

### udis86

第一次使用很麻烦，（自己太菜

lib库要自己去生成，源文件缺少头文件，未解决。使用加号生成的lib库

要导入

![image-20200310173936136](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/typroa/image-20200310173936136.png)



需要手动导入头文件`udis86.h`

lib库文件`libudis86.lib` 

测试代码如下

```c++
#include <stdio.h>
#include <udis86.h>
#include <windows.h>
#pragma comment(lib, "libudis86.lib") 

struct FileInfo
{
	LPVOID pAsmCode;
	int AsmLen;
};

FileInfo LoadFileToMem(const char* FilePath)
{
	FileInfo sAsm;
	FILE* fp;
	fopen_s(&fp, FilePath, "rb");
	fseek(fp, 0, SEEK_END);
	sAsm.AsmLen = ftell(fp);
	fseek(fp, 0, SEEK_SET);
	sAsm.pAsmCode = new char[sAsm.AsmLen+1];
	ZeroMemory(sAsm.pAsmCode, sAsm.AsmLen);
	fread_s(sAsm.pAsmCode, sAsm.AsmLen, 1, sAsm.AsmLen, fp);
	fclose(fp);
	return sAsm;
}

int main()
{
	FileInfo sAsm = LoadFileToMem("C:\\Users\\Cray\\Desktop\\dEMO\\Shellcode_64.bin");

	ud_t ud_obj;
	ud_init(&ud_obj);
	//ud_set_input_file(&ud_obj, stdin);	//从数据流中读入二进制数据
	ud_set_input_buffer(&ud_obj, (PBYTE)sAsm.pAsmCode, sAsm.AsmLen);	//内存中读入
	ud_set_mode(&ud_obj, 64);	//输出32位汇编
	ud_set_syntax(&ud_obj, UD_SYN_INTEL); //INTEL 汇编语法

	while (ud_disassemble(&ud_obj)) {
		printf("\t%s\n", ud_insn_asm(&ud_obj));
	}
	delete[] sAsm.pAsmCode;
	return 0;
}
```





**disasm.exe 生成的二进制格式，用来加载shellcode也是相当好用的**



```shell
D:\encodeDecodeOops\udis>disasm.exe  -b 32 -f c Shellcode_32.bin

#define SHELLCODE_32_SIZE 307
char SHELLCODE_32[] = {
  /* 0000 */ "\x31\xc9"                     /* xor ecx, ecx                    */
  /* 0002 */ "\x64\x8b\x41\x30"             /* mov eax, [fs:ecx+0x30]          */
  /* 0006 */ "\x8b\x40\x0c"                 /* mov eax, [eax+0xc]              */
  /* 0009 */ "\x8b\x70\x14"                 /* mov esi, [eax+0x14]             */
  /* 000C */ "\xad"                         /* lodsd                           */
  /* 000D */ "\x96"                         /* xchg esi, eax                   */
  /* 000E */ "\xad"                         /* lodsd                           */
  /* 000F */ "\x8b\x58\x10"                 /* mov ebx, [eax+0x10]             */
  /* 0012 */ "\x8b\x53\x3c"                 /* mov edx, [ebx+0x3c]             */
  /* 0015 */ "\x01\xda"                     /* add edx, ebx                    */
  /* 0017 */ "\x8b\x52\x78"                 /* mov edx, [edx+0x78]             */
  /* 001A */ "\x01\xda"                     /* add edx, ebx                    */
  /* 001C */ "\x8b\x72\x20"                 /* mov esi, [edx+0x20]             */
  /* 001F */ "\x01\xde"                     /* add esi, ebx                    */
  /* 0021 */ "\x31\xc9"                     /* xor ecx, ecx                    */
  /* 0023 */ "\x41"                         /* inc ecx                         */
  /* 0024 */ "\xad"                         /* lodsd                           */
  /* 0025 */ "\x01\xd8"                     /* add eax, ebx                    */
  /* 0027 */ "\x81\x38\x47\x65\x74\x50"     /* cmp dword [eax], 0x50746547     */
  /* 002D */ "\x75\xf4"                     /* jnz 0x23                        */
  /* 002F */ "\x81\x78\x04\x72\x6f\x63\x41" /* cmp dword [eax+0x4], 0x41636f72 */
  /* 0036 */ "\x75\xeb"                     /* jnz 0x23                        */
  /* 0038 */ "\x81\x78\x08\x64\x64\x72\x65" /* cmp dword [eax+0x8], 0x65726464 */
  /* 003F */ "\x75\xe2"                     /* jnz 0x23                        */
  /* 0041 */ "\x8b\x72\x24"                 /* mov esi, [edx+0x24]             */
  /* 0044 */ "\x01\xde"                     /* add esi, ebx                    */
  /* 0046 */ "\x66\x8b\x0c\x4e"             /* mov cx, [esi+ecx*2]             */
  	...
```

### capstone 

也是需要自己导入lib库

例子如下，网上也有教程



```c++
#include <stdio.h>
#include <udis86.h>
#include <windows.h>
#pragma comment(lib, "libudis86.lib") 

struct FileInfo
{
	LPVOID pAsmCode;
	int AsmLen;
};

FileInfo LoadFileToMem(const char* FilePath)
{
	FileInfo sAsm;
	FILE* fp;
	fopen_s(&fp, FilePath, "rb");
	fseek(fp, 0, SEEK_END);
	sAsm.AsmLen = ftell(fp);
	fseek(fp, 0, SEEK_SET);
	sAsm.pAsmCode = new char[sAsm.AsmLen+1];
	ZeroMemory(sAsm.pAsmCode, sAsm.AsmLen);
	fread_s(sAsm.pAsmCode, sAsm.AsmLen, 1, sAsm.AsmLen, fp);
	fclose(fp);
	return sAsm;
}

int main()
{
	FileInfo sAsm = LoadFileToMem("C:\\Users\\Cray\\Desktop\\dEMO\\Shellcode_64.bin");

	ud_t ud_obj;

	ud_init(&ud_obj);
	//ud_set_input_file(&ud_obj, stdin);	//从数据流中读入二进制数据
	ud_set_input_buffer(&ud_obj, (PBYTE)sAsm.pAsmCode, sAsm.AsmLen);	//内存中读入
	ud_set_mode(&ud_obj, 64);	//输出32位汇编
	ud_set_syntax(&ud_obj, UD_SYN_INTEL); //INTEL 汇编语法

	while (ud_disassemble(&ud_obj)) {
		printf("\t%s\n", ud_insn_asm(&ud_obj));
	}

	delete[] sAsm.pAsmCode;
	return 0;
}
```

