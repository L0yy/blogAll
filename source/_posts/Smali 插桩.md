---
title: Smali插桩
tags: [Android]
index_img: https://gitee.com//cve/BlogImg/raw/master/typora/image-20200331171804535.png
banner_img: https://gitee.com//cve/BlogImg/raw/master/typora/image-20200331171804535.png
date: 2020-3-31
---

## 前言

大型程序不好直接调试，一个API 有可能多次调用，直接下断也是有印象，下面介绍下在smali中插入log代码达到调试的作用

smali源码可以随意加正规代码，就像汇编一样，只是可能修改后不能反编译成java代码

### 工具介绍

Android Killer V1.3.1.0（下面简称AK）

夜神模拟器或真机

演示APK：https://gitee.com/L0yy/android_series/raw/master/%E4%BE%8B%E5%AD%90/crackme1.apk

## 过程

**这里已经默认你安装过这个APK，知道他的一些行为**

打开AK，加载目标APK，等待反编译完成

打开项目的`Mainactivity.smali`的反编译代码

![image-20200331134623301](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200331134623301.png)

逻辑还是很清晰的，这里设置了按钮的`OnClickListener`当点击时就会执行相关函数，也就是这里的`onClick`

> 在反编译后的smali代码中，需要寻找equal判断的位置，在其进行输出getkey()返回值时插桩，由于功能唯一，代码位置很容易找到，在`com/mzheng/crackme1/MainActivity$1`文件中；
>
> BTW：为什么是MainActivity$1.smali而不是MainActivity.smali呢？
>
> 因为主要的判断逻辑是在OnClickListener这个类里，而这个类是MainActivity的一个内部类，同时我们在实现的时候也没有给这个类声明具体的名字，所以这个类用$1表示。实际环境会有很多`$+id`的文件，大多如此，多功能情况下还是搜索特殊字符或对怀疑位置进行插桩来判断实际功能位置；因为继承呢
>
> 引用自：https://www.bodkin.ren/index.php/archives/560/



这里主要讲方法

#### 法1：用Ak直接插入Log

我们目的是修改`com/mzheng/crackme1/MainActivity$1`这个文件，达到把Getkey的返回值给打印出来



![image-20200331142750849](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200331142750849.png)



在这个位置**右键->插入代码->log**



AK会帮你插入这个方法，直接调用就行

![image-20200331143003496](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200331171804535.png)



把V0改为你要打印的数据，**一定记得要保存你的smali代码** AK编译的时候不会帮你保存，要你自己保存

然后编译安装，注意箭头表注的地方

![image-20200331171526913](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200331143003496.png)



最后打开日志，稍微过滤下就可以得到`Tag：AndroidKiller-string` 的消息

![image-20200331171804535](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200331171526913.png)



如果使用adb命令如下

`adb logcat -s AndroidKiller-string`





#### 法2 手动添加Log类

原理和法1一样，只是这个稍微麻烦一点

参考SewellDinG的方法

https://github.com/SewellDinG/smaliArmory

### 遇到的几个坑



1. AK编译慢，老出错

   网上给的方案，替换 **AndroidKiller_v1.3.1\bin\apktool\apktool** 下的**ShakaApktool.jar** 吾爱盘中有

2. AK找不到设备

   现场时使用环境变量中的adb连接一下

   adb connect 127.0.0.1:62001

   点击刷新后，还是没有的话就用**AndroidKiller_v1.3.1\bin\apktool**中的adb去连，因为AK用的自己的adb工具

3. AK回编不了

   AK的apktool有很久没更新了，导致某些版本不能回编

   但是这些都可以通过用高版本的apktool 手动敲命令替换，所以都不是问题

## 总结

smali中可以插入任意代码，只要不影响程序运行，AK只是一个辅助工具，基本功能都是再apktool上实现的，所以它解决不了的还是老老实实敲代码吧！