---
title: Powershell基础
tags: [Shell]
index_img: https://cdn.jsdelivr.net/gh/L0yy/tuchuang/Img/20200302103202.png
banner_img: https://cdn.jsdelivr.net/gh/L0yy/tuchuang/Img/20200302103202.png
---





### Powershell基础

参考微软[官方文档]( https://docs.microsoft.com/zh-cn/powershell/scripting/how-to-use-docs?view=powershell-7 )

几个特性：

> 1. .NET Core可跨平台，powershell6可在Mac Linux平台使用，大气
> 2. Win10自带组建，功能强大



#### 了解Powershell

1. 输出基于对象
   - PowerShell cmdlet 旨在处理对象
   - 在大多数情况下，可以使用标准 PowerShell 对象语法直接访问数据的各部分
2. 命令系列是可扩展的
   - 可以自己实现函数
   - 支持批处理文件的脚本
3. PowerShell 处理控制台输入和显示
   - 在cmdlet 后使用 **-？**可显示关于此命令的帮助
4. PowerShell 使用某些 C# 语法

#### 了解powershell的名称

 	1. Cmdlet 使用谓词-名词的名称来减少命令记忆
     - [PowerShell 批准的谓词]( https://docs.microsoft.com/zh-cn/powershell/scripting/developer/cmdlet/approved-verbs-for-windows-powershell-commands?view=powershell-7 )
 	2. Cmdlet 使用标准参数
     - 参数主要通过**-？**获取，还有通用参数如 WhatIf、Confirm、Verbose、Debug、Warn、ErrorAction、ErrorVariable、OutVariable 和 OutBuffer
	3. 建议的参数名称
    -   加上方便人理解： Force 、Exclude 、Include 、PassThru 、Path 和 CaseSensitive 

#### 使用熟悉的命令名称

​	**1. Powershell 支持常见的Unix 和cmd.exe 的命令**

​	

|       |         |       |       |
| :---- | :------ | :---- | :---- |
| cat   | dir     | mount | rm    |
| cd    | echo    | move  | rmdir |
| chdir | erase   | popd  | sleep |
| clear | h       | ps    | sort  |
| cls   | history | pushd | tee   |
| copy  | kill    | pwd   | type  |
| del   | lp      | r     | write |
| diff  | ls      | ren   |       |

2. 别名相关操作

    ` Get-Command -Noun Alias`

#### 获取详细的帮助信息

1. 获取有关 cmdlet 的帮助
   - 常规帮助 **Get-Help cmdlet**、**man cmdlet** 、**cmdlet   -?** 
   - 获取到参数的帮助  **Get-Help cmdlet -Full** 

#### 使用变量存储对象

1. 使用**$**表示变量
2. Get-Member $env:获取变量的信息
    - 可以创建设置键值对，例：` $env:LIB_PATH='/usr/local/lib' `

####  了解 PowerShell 管道

​	主要是方法和属性的使用，下面举个例子

​	` Get-Location ` 获取一个pathInfo

​	`Get-Location | Get-Member`获取这个类的方法和属性

​	想要使用必须先实例化，传给一个变量

​	` $myLocal = Get-Location`实例化，这个`$myLocal`拥有`Get-Location`的全部方法和属性

​	比如`$myloc.Drive.Used `获取所在驱动内存使用大小，通过`Get-Member`获取详情



#### 术语表

powershell的常用术语

建议中英对照看，翻译的有些不易理解

[中文]( https://docs.microsoft.com/zh-cn/powershell/scripting/learn/windows-powershell-glossary?view=powershell-7 )

[英文](https://docs.microsoft.com/en-us/powershell/scripting/learn/windows-powershell-glossary?view=powershell-77)

