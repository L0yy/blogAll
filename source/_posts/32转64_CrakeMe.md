---
title: 32位转64位 easy_CrackMe
tags: [CrakeMe]
index_img: https://gitee.com//cve/BlogImg/raw/master/typora/image-20200602184341253.png
banner_img: https://gitee.com//cve/BlogImg/raw/master/typora/image-20200602184341253.png
date: 2020-5-4
---

## 背景


参考来源：

https://moliam.github.io/2018/11/17/Wow64-at-the-assemly-level.html

思路来源汪师傅

## 主要原理

64位系统中运行32位程序时是使用Wow64来辅助执行的，主要是可以通过控制cs寄存器来改变CPU执行命令标准，当cs=0x23时按照32位可执行性程序执行，当cx=0x33时按照64位标准执行

所以通过调整cs寄存器就可以在32位程序中执行64位的代码

### 难点

常规调试器无法同时调试32位和64位代码

到达64代码无法单步跟踪

详情可参考 https://www.anquanke.com/post/id/171111

开发的时候64位汇编需要完全硬编码后转汇编码写进去，修改的时候基本要完全重写一次汇编

### 逆向过程

无壳无混淆

使用od调试到达`0x401100`跑飞，可以看到是大跳，且修改了段寄存器位`0x33`

这里的目标地址是`0x401090`

![image-20200602184341253](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200602185117668.png)

使用`IDA_64` 找到对应数据，根据前面修改了段寄存器cs为0x33，所以这里箭头所指的代码就因该是64位的代码

![image-20200602184654770](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200602184654770.png)

**请注意，是使用64位的IDA，不是32位的**

按下`alt+s`将代码按照64位反编译



![image-20200602185117668](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200602184341253.png)

然后按**G**跳转到**0x401090**处，按下**x**，将数据按汇编代码识别

![image-20200602185444980](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200602191541585.png)

这就是隐藏在32位中的64位代码，逻辑很简单，两个异或后就能处flag

将输入的flag的前8位 xor 0x1949194820192019 结果要等于 0x2D70283347784C7F

后8位xor 0x1949194820192019  等于 0x6478296510280D20

由于数据在内存中是小端序排列，所以异或后算出的值需要倒置一下

脚本如下

```python
def hex2asc(hexs):
    res = ""
    for i in range(0, len(hexs), 2):
        res += chr(int(hexs[i:i+2], 16))
    return res

v1 = 0x2D70283347784C7F ^ 0x1949194820192019
v2 = 0x6478296510280D20 ^ 0x1949194820192019
frist8 = (str(hex(v1)))[2:]
last8 = (str(hex(v2)))[2:]
res = hex2asc(frist8)
print(res[::-1],end="")

res = hex2asc(last8)
print(res[::-1])

```

测试最终结果

![image-20200602191541585](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200602185444980.png)