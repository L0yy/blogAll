---
title: 基于C#的PEloder
tags: [C#,RedTeam]
index_img: https://gitee.com//cve/BlogImg/raw/master/typora/wallhaven-g8k8yq.jpg
banner_img: https://gitee.com//cve/BlogImg/raw/master/typora/wallhaven-g8k8yq.jpg
date: 2020-3-25
---

使用C#的pe加载器可以很方便的生成和修改，代码可变性较高，可在被目标机中编译生成

思路来源：Casey Smith，GhostPack  和 3student

###  C#PE加载

Casey Smith提供的方案有两种加载方式

1.`public PELoader(string filePath)`

2.`public PELoader(byte[] fileBytes)`

方法1 可以直接加载运行一个本地的可执行程序

方法2 则是可以直接执行内存中的可运行程序的流数据

代码见：

https://gitee.com/L0yy/Cshap/blob/master/PEloader.cs

### 改良

经过测试，上述代码只能加载64位的PE。

通过3student的改良，将64位版本中的Opheader结构修改位32位的



拷贝一份学习

https://gitee.com/L0yy/Cshap/blob/master/SharpPELoaderGenerater.cs

其他代码整合见

https://gitee.com/L0yy/Cshap