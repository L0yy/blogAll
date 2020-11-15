---
title: Window事件日志清理
tags: [ReadTeam]
index_img: https://cdn.jsdelivr.net/gh/L0yy/tuchuang/typroa/image-20200315155137730.png
banner_img: https://cdn.jsdelivr.net/gh/L0yy/tuchuang/typroa/image-20200315155137730.png

---



### Window事件日志

>Windows系统日志是记录系统中硬件、软件和系统问题的信息，同时还可以监视系统中发生的事件。用户可以通过它来检查错误发生的原因，或者寻找受到攻击时攻击者留下的痕迹。
>
>Windows主要有以下三类日志记录系统事件：应用程序日志、系统日志和安全日志。

### 基本思路



1.遍历进程模块，筛选svchost

2.遍历进程导入模块，是否有wevtsvc.dll模块，如果有，则进程为目标进程

3.再次遍历进程的所有线程，获取每个线程句柄

使用未导出函数`NtQueryInformationThread`获取线程的THREAD_BASIC_INFORMATION结构体信息，读取这个进程的数据ProcessTag，根据这个Tag来找到SC_SERVICE_TAG_QUERY.pBuffer，当这个为eventlog则是日志监控进程

测试效果如下：

将特定线程结束就可以停止记录了。

<img src="https://cdn.jsdelivr.net/gh/L0yy/tuchuang/typroa/1.gif" alt="1" style="zoom:50%;" />



代码参考：https://github.com/QAX-A-Team/EventCleaner

### NtQueryInformationThread

获取到线程信息，这里主要是拿到线程的Tag，线程Tag从线程TEB中拿出

x86 PPEB + 0xF60

x64 PPEB + 0x1720 



参考http://terminus.rewolf.pl/terminus/structures/ntdll/_TEB_x64.html

### _I_QueryTagInformation

这里根据tag来获取线程属性，也就是

![image-20200315155137730](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/typroa/image-20200315155137730.png)



**我修改后的代码**

https://gitee.com/L0yy/log_cleaning