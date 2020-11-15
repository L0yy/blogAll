---
title: Powershell 官方例子
tags: [Shell]
index_img: https://cdn.jsdelivr.net/gh/L0yy/tuchuang/Img/20200302232152.png
banner_img: https://cdn.jsdelivr.net/gh/L0yy/tuchuang/Img/20200302232152.png

---



### powershell 官方例子说明

​	全部例子来源自[示例脚本]( https://docs.microsoft.com/zh-cn/powershell/scripting/samples/sample-scripts-for-administration?view=powershell-7 )

### 使用对象

#### 查看对象结构 (Get-Member)

>  `Get-Process | Get-Member | Out-Host -Paging`

获取进程列表，再获取他们各自的成员信息，最后按页输出

>  `Get-Process | Get-Member -MemberType Properties` 
>
> MemberType 的允许值有 AliasProperty、CodeProperty、Property、NoteProperty、ScriptProperty、Properties、PropertySet、Method、CodeMethod、ScriptMethod、Methods、ParameterizedProperty、MemberSet 以及 All。 



#### 选择对象部件 (Select-Object)

> `Get-CimInstance -Class Win32_LogicalDisk | Select-Object -Property Name,FreeSpace`

利用WMI win32-logicaldisk类来获取服务器的磁盘空间使用率的信息，然后选择性的打印出 Name和FreeSpace两个项目

但是FreeSpace是 uint64格式的

```powershell
Get-CimInstance -Class Win32_LogicalDisk |
  Select-Object -Property Name, @{
    label='FreeSpace Cray'
    expression={($_.FreeSpace/1GB).ToString('F2')}
  }
```



第一行是获取基本数据，第二行是选择对象的属性，分别是Name和一个自定义的

 `$_` 来指代管道中的当前对象 

`ToString('F2')` 表示保留两位小数

#### 从管道中删除对象 (Where-Object)

```powershell
Get-CimInstance -Class Win32_SystemDriver |
  Where-Object {$_.State -eq "Running"} |
    Where-Object {$_.StartMode -eq "Manual"} |
      Format-Table -Property Name,DisplayName 
```
`Where-Object` 作用就是筛选


|  比较运算符  |            含义            |      示例（返回 True）       |
| :----------: | :------------------------: | :--------------------------: |
|     -eq      |            等于            |           1 -eq 1            |
|     -ne      |           不等于           |           1 -ne 2            |
|     -lt      |            小于            |           1 -lt 2            |
|     -le      |         小于或等于         |           1 -le 2            |
|     -gt      |            大于            |           2 -gt 1            |
|     -ge      |         大于或等于         |           2 -ge 1            |
|    -like     |  相似（文本的通配符比较）  |  "file.doc" -like "f*.do?"   |
|   -notlike   | 不相似（文本的通配符比较） | "file.doc" -notlike "p*.doc" |
|  -contains   |            包含            |      1,2,3 -contains 1       |
| -notcontains |           不包含           |     1,2,3 -notcontains 4     |

```powershell
Get-CimInstance -Class Win32_SystemDriver |
  Where-Object {($_.State -eq 'Running') -and ($_.StartMode -eq 'Manual')} |
    Format-Table -Property Name,DisplayName
```

如果筛选条件有多个，可以使用逻辑运算符

`Format-Table` 可以格式化最后的输出格式

| Logical and；如果两侧都为 True，则返回 True | -and | (1 -eq 1) -and (2 -eq 2) |
| :-----------------------------------------: | :--: | :----------------------: |
| Logical or；如果某一侧为 True，则返回 True  | -or  | (1 -eq 1) -or (1 -eq 2)  |
|       Logical not；反转 True 和 False       | -not |      -not (1 -eq 2)      |
|       Logical not；反转 True 和 False       |  !   |        !(1 -eq 2)        |

#### 对对象进行排序( Sort-Object )

简单使用

```powershell
Get-ChildItem |
  Sort-Object -Property CreationTime -Descending  |
  #Select-Object -Property Name,CreationTime
  Format-Table -Property Name,CreationTime
```

运用哈希排序的复杂使用

```powershell
Get-ChildItem |
  Sort-Object -Property @{ Expression = { $_.LastWriteTime - $_.CreationTime }; Descending = $true } |
  Format-Table -Property LastWriteTime, CreationTime
```

对目录中的文件，按最后写日期-创建日期大小进行降序排列



####  创建 .NET 和 COM 对象 (New-Object)

创建某些COM对象

```powershell
New-Object -ComObject WScript.Shell
New-Object -ComObject WScript.Network
New-Object -ComObject Scripting.Dictionary
New-Object -ComObject Scripting.FileSystemObject
```

一个使用**WScript.Shell** 创建桌面快捷方式并执行

```powershell
$WshShell = New-Object -ComObject WScript.Shell
$calclink = $WshShell.CreateShortcut("C:\Users\Cray\Desktop\calc.lnk")
$calclink.TargetPath = "C:\WINDOWS\system32\calc.exe"
$calclink.Save()
$WshShell.run("C:\Users\Cray\Desktop\calc.lnk")
```

#### 使用静态类和方法

静态类不能使用New-Object来创建

无需创建即可使用

使用时用**[ ]**调用，通常使用静态类的静态方法

```powershell
[System.Environment] | Get-Member  #此类的详细信息
[System.Environment] | Get-Member -Static #查看静态成员
```

调用方式使用  **::**

```powershell
[System.Environment]::OSVersion 
```

还有很多静态类，比如 math

```powershell
PS> [System.Math]::Sqrt(9)
3
PS> [System.Math]::Pow(2,3)
8
PS> [System.Math]::Floor(3.3)
3
PS> [System.Math]::Floor(-3.3)
-4
PS> [System.Math]::Ceiling(3.3)
4
PS> [System.Math]::Ceiling(-3.3)
-3
PS> [System.Math]::Max(2,7)
7
PS> [System.Math]::Min(2,7)
2
PS> [System.Math]::Truncate(9.3)
9
PS> [System.Math]::Truncate(-9.3)
-9
```



#### 获取 WMI 对象 (Get-CimInstance)

> Windows Management Instrumentation (WMI) 是 Windows 系统管理的核心技术，因为它以统一的方式公开大量信息。 由于 WMI 可实现的效果，用于访问 WMI 对象的 PowerShell cmdlet `Get-CimInstance` 是进行实际工作最有用的对象之一 

列出本机上可用的WMI类列表
```powershell
Get-CimClass -Namespace root/CIMV2 |
  Where-Object CimClassName -like Win32* |
    Select-Object CimClassName
```

使用WMI获取系统信息  Namespace 默认为 `root/CIMV2` 

`Get-CimInstance -Class Win32_OperatingSystem`

他有很多属性，都可以查看。

例如

```powershell
 Get-CimInstance -Class Win32_OperatingSystem | 
 select -Property Free*, @{
 label = "TotalVirtualMemorySize" 
 expression = {($_.TotalVirtualMemorySize/1MB).ToString("F0")}}  | Format-Table 
```

或者下面这样都是可以的

```powershell
Get-CimInstance Win32_OperatingSystem |
Format-List Total*Memory*, Free*
```

### 管理计算机

该节有大量关于计算机的例子，建议直接看[官方文档](https://docs.microsoft.com/zh-cn/powershell/scripting/samples/collecting-information-about-computers?view=powershell-7)

> 若要完整显示具有极长名称的临时服务的名称，可能需要使用具有 AutoSize 和 Wrap 参数的 `Format-Table`，用于优化列宽并允许较长名称换行而不是被截断 