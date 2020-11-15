---
title: Android逆向第一个CrakeMe
tags: [Android逆向]
index_img: https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191216120851.png
banner_img: https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191216120851.png
date: 2019-12-14 22:28:01
---


自己写的第一个Android CrakeMe 也是为了练手

随意输入参数为错误，始入正确flag能打印`Success `

![20191214193723.png](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191214193723.png)

配合使用上一篇的流程破解

## 解包

使用`apktool`解包

![20191214193422.png](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191214193422.png)


拿到了Smali文件

## 反编译

这里使用`jadx`发编译

知道了成功会打印`Success`,直接搜索关键字就好了

![20191214194137.png](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191214194137.png)


双击进去后就可以看到关键的代码，可以使用引用查看函数调用情况

这里看到的`JAVA`代码是`Smali`反编译得到的，所以中文都是使用的`Unicode`表示,这也表示以后搜索代码中的关键字需要转化为`Unicode`字符集去搜索

![20191214194305.png](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191214194305.png)


很简单的逻辑就是改跳转，把 `==`改为`!=`这类的相反的比较

我们不能直接修改JAVA代码，要修改JVM虚拟机读的`Smali`语句

这和你要破解windows程序，不能在IDA中直接改F5出来伪代码，要修改汇编一个道理



我们要修改的代码位置如下

![20191214200228.png](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191214200228.png)

找到相应的文件，使用编辑器打开，直接修改Smali语句就好了

![20191214200613.png](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191214200613.png)

这里将`if-eqz`修改为`if-nez`

然后保存退出


## 回编译APK

这里需要将修改后的文件重新编译为APK，才能安装

使用命令

`apktool b [Folder_Paht]`

![20191214201012.png](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191214201012.png)

成功后会在`Folder_Paht/dist`下创建新的打包文件，但是现在不能安装，没有签名文件，需要给这个文件签一个制作者的`sign`

![20191214201139.png](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191214201139.png)

具体命令

`jarsigner -verbose -keystore key.keystore D:\Tools\ApkTools\Hello\dist\Hello.apk key.keystore`


![20191214201718.png](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191214201718.png)




放入模拟器，无论输入什么都会输出Success(除了正确密码...)

![20191214201908.png](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191214201908.png)





以上就是通过修改Smali语句实现的暴力破解


下面详细分析一下Smali语句

源码，注意空格在反编译的Smali语句中也算一行

![20191216101748.png](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191216101748.png)


```Smali
.class public Lcom/Crat/changeme/MainActivity;#类名
.super Landroidx/appcompat/app/AppCompatActivity;#父类
.source "MainActivity.java"#对应的JAVA源代码文件名


# direct methods
.method public constructor <init>()V
    .locals 0

    .line 11
    #调用父类的AppCompatActivity初始化函数，已经识别为init
    invoke-direct {p0}, Landroidx/appcompat/app/AppCompatActivity;-><init>()V                    

    #无返回值
    return-void
.end method


# virtual methods
.method public Pushed(Landroid/view/View;)V
    .locals 8
    .param p1, "view"    # Landroid/view/View;

    .line 21
    # v0 = 0x7f070042
    const v0, 0x7f070042

    #调用MainActivity.findViewById(int v0)  返回值是 View 类型
    invoke-virtual {p0, v0}, Lcom/Crat/changeme/MainActivity;->findViewById(I)Landroid/view/View;

    #将返回值(对象)赋给v0
    move-result-object v0

    #强制类型转换，将v0转换为TextView类型
    check-cast v0, Landroid/widget/TextView;

    .line 22
    #声明变量 TextView Name  来保存v0寄存器的值
    .local v0, "Name":Landroid/widget/TextView;

    const v1, 0x7f0700ab

    invoke-virtual {p0, v1}, Lcom/Crat/changeme/MainActivity;->findViewById(I)Landroid/view/View;

    move-result-object v1

    check-cast v1, Landroid/widget/TextView;

    .line 23
    .local v1, "Text2":Landroid/widget/TextView;
    const v2, 0x7f070054

    invoke-virtual {p0, v2}, Lcom/Crat/changeme/MainActivity;->findViewById(I)Landroid/view/View;

    move-result-object v2

    check-cast v2, Landroid/widget/TextView;

    .line 24
    .local v2, "Edit2":Landroid/widget/TextView;
    #下面一句翻译为java语句是 v2.getText() 函数返回类型为char类型的序列
    #CharSequence和String 详情请看：https://blog.csdn.net/iblade/article/details/78111223
    invoke-virtual {v2}, Landroid/widget/TextView;->getText()Ljava/lang/CharSequence;

    move-result-object v3
    #给上面的CharSequence转换为String类型
    invoke-interface {v3}, Ljava/lang/CharSequence;->toString()Ljava/lang/String;

    move-result-object v3

    .line 25
    #将获得的String字符赋值给InputKey局部变量中
    .local v3, "InputKey":Ljava/lang/String;
    const-string v4, ""

    .line 27
    .local v4, "TmpFlag":Ljava/lang/String;
 
    # 下面两句表示 int i = 0
    const/4 v5, 0x0
    .local v5, "i":I

    :goto_0
    #获取到输入字符串的长度
    invoke-virtual {v3}, Ljava/lang/String;->length()I
    #将长度返回给v6寄存器保存
    move-result v6

    #如果v5>=v6 就跳转
    if-ge v5, v6, :cond_0

    .line 29
    new-instance v6, Ljava/lang/StringBuilder;

    invoke-direct {v6}, Ljava/lang/StringBuilder;-><init>()V

    invoke-virtual {v6, v4}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    invoke-virtual {v3, v5}, Ljava/lang/String;->charAt(I)C

    move-result v7
    #xor-int/lit8 v7, v7, 0xa	v7(前) = v7(后) ^  0xa
    xor-int/lit8 v7, v7, 0xa

    int-to-char v7, v7

    invoke-virtual {v6, v7}, Ljava/lang/StringBuilder;->append(C)Ljava/lang/StringBuilder;

    invoke-virtual {v6}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    move-result-object v4

    .line 27
    #add-int/lit8 v5, v5, 0x1  v5(前) = v5(后) + 1
    add-int/lit8 v5, v5, 0x1

    goto :goto_0

    .line 32
    .end local v5    # "i":I


    :cond_0
    #Sting v5 = "lfkmqBoffe*}exfnw"
    const-string v5, "lfkmqBoffe*}exfnw"

    #v4.equals(v5) 返回值是boolean
    invoke-virtual {v4, v5}, Ljava/lang/String;->equals(Ljava/lang/Object;)Z

    move-result v5

    #v5不等于0跳转
    if-nez v5, :cond_1

    .line 34
    const-string v5, "\u606d\u559c\u6b63\u786e\uff01"

    invoke-virtual {v0, v5}, Landroid/widget/TextView;->setText(Ljava/lang/CharSequence;)V

    .line 35
    const-string v5, "Success!"

    invoke-virtual {v1, v5}, Landroid/widget/TextView;->setText(Ljava/lang/CharSequence;)V

    goto :goto_1

    .line 39
    :cond_1
    const-string v5, "\u9519\u8bef\uff0c\u8bf7\u91cd\u8bd5\uff01"

    invoke-virtual {v0, v5}, Landroid/widget/TextView;->setText(Ljava/lang/CharSequence;)V

    .line 40
    invoke-virtual {v1, v4}, Landroid/widget/TextView;->setText(Ljava/lang/CharSequence;)V

    .line 42
    :goto_1
    return-void
.end method

.method protected onCreate(Landroid/os/Bundle;)V
    .locals 1
    .param p1, "savedInstanceState"    # Landroid/os/Bundle;

    .line 15
    invoke-super {p0, p1}, Landroidx/appcompat/app/AppCompatActivity;->onCreate(Landroid/os/Bundle;)V

    .line 16
    #这里给到的是资源布局文件的编号，去找到这个值对应的资源文件就行
    const v0, 0x7f0a001c
    #setContentView(R.layout.activity_main);
    invoke-virtual {p0, v0}, Lcom/Crat/changeme/MainActivity;->setContentView(I)V

    .line 17
    return-void
.end method

```























