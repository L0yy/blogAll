---
title: DDK与WDK WDM的区别
tags: [内核, 驱动学习]
index_img: https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191216120944.png
banner_img: https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191216120944.png
date: 2019-11-18 22:28:01
---


自己总结如下：


![20191216121033.png](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191216121033.png)

![image.png](https://upload.wikimedia.org/wikipedia/commons/thumb/6/6d/Windows_Updated_Family_Tree.png/1920px-Windows_Updated_Family_Tree.png)



转载自：http://blog.sina.com.cn/s/blog_4b9eab320101b6yn.html


1.首先，先从基础的东西说起，开发WINDOWS下的驱动程序，需要一个专门的开发包，如：开发JAVA程序，我们可能需要一个JDK，开发WINDOWS应用程序，我们需要WINDOWS的SDK，现在开发WINDOWS下的驱动程序，我们需要一个DDK/WDK。

2.DDK（Driver Developer Kit）和WDK（Windows Driver Kit）的区别：

这个要说说驱动相关的一些历史:

	1).95/98/ME下，驱动模型为：Vxd，相关资料可以看《编程高手箴言》的前几个章节，里面有很详细的介绍，虽然这个东西已经过时，但大概看看还是会增长见识的。

	2).2000/XP/2003下，Windows采用WDM驱动模型（Windows Driver Model），开发2000/XP/2003的驱动开发包为：DDK。

	WDM驱动无非是微软在NT式驱动之上进行了扩充，过滤驱动也不例外 。

	3).Vista及以后版本，采用了WDF驱动模型（Windows Driver Foudation），对应的开发包：WDK。

其实**WDK可以看做是DDK的升级版本，现在一般的WDK是包含以前DDK相关的功能，现在XP下也可以用WDK开发驱动，WDK能编译出2000-2008的各种驱动**。

3.Vxd驱动文件扩展名为：.vxd。

WDM和WDF驱动文件扩展名为：.sys。

4、WDM 是 Win32设备驱动程序体系结构。

件驱动的驱动程序开发框架，大大降低了开发难度。从现在开始，掌握Windows设备驱动程序的开发人员，由过去的“专业”人士，将变为“普通”大众。

WDF驱动程序包括两个类型，一个是内核级的，称为KMDF（Kernel-Mode Driver Framework），为SYS文件；另一个是用户级的，称为UMDF（User-Mode Driver Framework），为DLL文件。


5、DDK 和WDK

**DDK是基于wdm驱动模型的，而WDK是基于WDF驱动模型的**，wdm驱动模型和wdf驱动模型的最大的区别是：

	1)wdf驱动框架对WDM进行了一次封装，WDF框架就好像C++中的基类一样，且这个基类中的model,IO model ,pnp和电源管理模型;且提供了一些与操作系统相关的处理函数，这些函数好像C++中的虚函数一样，WDF驱动中能够对这些函数进行override；特别是Pnp管理和电源管理！基本上都由WDF框架做了，而WDF的功能驱动几乎不要对它进行特殊的处理；

	2)WDF驱动模型 与WDM驱动模型的另外一个主要区别是：

	WDF 驱动采用队列进行IO处理，而WDM中将所有的IO操作都用默认的队列进行处理，如果要进行IRp同步，必须使用StartIO；

	3)WDF是面向对象的，而WDM是面向过程的，WDF提供对象的封装，如将IRP封装成WDFREQUEST，对象提供方法和Event。

	5）usb设备的读写；

	当应用程序使用ReadFile或WriteFile进行读写时，首先将

	UsbBuildInterruptOrBulkTransferRequest将构建urb请求，然后通过IoCallDriver发送给底层usb 总线驱动；

	对于WDF来说，WdfUsbTargetPipeFormatRequestForRead 格式化读请求，然后使用WdfRequestSend  发送给底层Usb总线驱动；

	对WDM和WDF的usb的读写都可以设置完成例程；