---
title: Android Studio动态调试Smali 和 插桩
tags: [Android]
index_img: https://gitee.com//cve/BlogImg/raw/master/typora/remote-debugging.png
banner_img: https://gitee.com//cve/BlogImg/raw/master/typora/remote-debugging.png
date: 2020-3-30
---

## 前言

无源码调试Smali，很有必要

JVM：夜神模拟器 V6.6.0.5101

开发环境：Android Studio 3.6.1

文中的目标APK命名为：`Nothing.apk`



Android Killer V1.3.1.0（下面简称AK）

夜神模拟器或真机

演示APK：https://gitee.com/L0yy/android_series/raw/master/%E4%BE%8B%E5%AD%90/crackme1.apk



## 安装插件

https://bitbucket.org/JesusFreke/smali/downloads/

**这个包在页面下方**

![image-20200330172401980](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200330172401980.png)

安装位置在设置的插件中，如图

![image-20200330172515100](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200330172515100.png)

## 反编译APK

一般release版本都是不支持调试的，先解包

`apktool d Nothing.apk`

修改`AndroidManifest.xml` 中的调试选项

在`application`中添加`android:debuggable="true"`

![image-20200330172920337](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200330181158966.png)

修改后回包

`apktool b . -o Nothing_path.apk`

这里的apk还没有签名，如何签名参考上一篇文章

[http://l0yy.gitee.io/cray/2019/12/14/android%E9%80%86%E5%90%91%E4%B8%80/](http://l0yy.gitee.io/cray/2019/12/14/android逆向一/)

-----------

**后面的操作都是建立在这个`Nothing_path.apk`上的**

----------

## 安装Nothing_path.apk

**法1：直接把apk拖进去自动安装**

**法2：使用adb**

先链接到模拟器,再安装

```shell
adb connect 127.0.0.1:62001			//夜神模拟器固定链接端口62001
adb install Nothing_path.apk
```



为了方便后期调试，这里将要调试的程序以调试模式开启，让它等待调试。

标准格式 

`adb shell am start -D -n packagename/packagename.MainActivity`

我的命令如下

`adb shell am start -D -n com.cray.nothing/com.cray.nothing.MainActivity`

![image-20200330181158966](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200330175318198.png)



## 配置调试器 AS

**打开apktool 解包生成的目录**

![image-20200330175318198](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200330172920337.png)

打开后在AS目录如下 

![image-20200330175900669](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200330175900669.png)

接下来每一步都要做

1. 设置源根目录

   右键Smali，选择make Drectory As -> sourses root 

   有多少个Smali文件夹就设置多少

   但是也有将整个项目都设为Soourses root（未尝试）

2. 设置SDK

   右键项目，open modlue setting 然后选择Project 选择一个本机的SDK

   ![image-20200330180417381](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200330180417381.png)

3. 配置调试器选项

   基本配置都好了，配置下调试选项就行

   选择 Run -->Edit Configurations，增加一个Remote调试的调试选项

   注意红色表注的就行，其他默认

   ![image-20200330180811567](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200330181702261.png)



### 连接模拟器调试

打开cmd

查询模拟器中所有的进程

`adb shell ps`

找到packName的PID

这里的PID 是`3336`

![image-20200330181315755](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200330180811567.png)

最后做一个端口转发

`adb forward tcp:5005 jdwp:3336`



回到AS中，在入口处下个断点

然后调试，成功如下图

![image-20200330181702261](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200330181315755.png)



ohhhhhhh :star:

