---
title: Android逆向原理 一
tags: [Android逆向]
index_img: https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191216120724.png
banner_img: https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191216120724.png
date: 2019-12-14 21:28:01
---


## 一套流程概述

使用`jadx`反编译后找到修改的文件，通过`apktool`反编`apk`文件后，在文件中找到对应的`smail文件`，修改后使用`apktool`回编，然后再用`jarsigner` 签名生成的apk

## 开始

apk本身可以使用压缩软件打开，打开后的目录

![20191214115813.png](https://cdn.jsdelivr.net/gh/L0yy/tuchuang/lmg/20191214115813.png)

这样打开的文件结构肯定是查看不完整的，很多二进制文件也不能解析


## 反编译APK


**apktool:逆向apk工具集**

**jadx：用于从Android Dex和Apk文件生成Java源代码的命令行和GUI工具**

**AndroidKiller：和jadx一样，还可以直接修改smail语句后回编APK以及添加签名**


使用apk反编译工具`apktool`,这里不推荐用`AndroidKiller`了

虽然他的反编很方便，但是连我的`Android studio`的Hello world程序都反编译不了，表示劝退好吧，而且也存在很多Bug

这里不得不安利一个超级好用的软件`jadx`，具体优点如下，使用方法也比较简单，去混淆的设置是真的好

[jadx介绍](https://segmentfault.com/a/1190000012180752)

### 第一步jadx反编译

我用jadx的目的是找到需要修改的具体文本，以及修改思路


### 修改后的值

apktool 反编出smali
使用命令

`apktool d [apk_path]`

就可以在apk目录下创建一个文件夹

对apk的所有修改都是对这个文件夹中内容进行修改

### 重新回编译

apktool 回编成apk

使用命令

`apktool b [apk_folder_path]`

**将会在这个文件夹下生成一个dist夹，其中就有回编的apk文件**

apktool回编的时候经常出现文件夹占用，可以使用
`apktool empty-framework-dir --force`  清空一下历史文件夹

### 文件重签名

注意上面生成的文件还不能安装，没有签名信息，安装会失败

如果是第一次使用签名，需要先生成一个,`keytool`一般的`JAVA`环境都自带


```
格式
keytool -genkeypair -keystore 密钥库名 -alias 密钥别名 -validity 天数 -keyalg RSA

参数:
    -genkeypair  生成一条密钥对(由私钥和公钥组成)
    -keystore    密钥库名字以及存储位置(默认当前目录)
    -alias       密钥对的别名(密钥库可以存在多个密钥对,用于区分不同密钥对)
    -validity    密钥对的有效期(单位: 天)
    -keyalg      生成密钥对的算法(常用RSA/DSA,DSA只用于签名,默认采用DSA)
    -delete      删除一条密钥
    
提示: 可重复使用此条命令,在同一密钥库中创建多条密钥对

例如:     
    在debug.keystore中新增一对密钥,别名是release
    keytool -genkeypair -keystore debug.keystore -alias release -validity 30000
```

生成好后就可以使用这个签名文件进行签名了


```
jarsigner -keystore 密钥库名 xxx.apk 密钥别名

参数:
    -digestalg  摘要算法
    -sigalg     签名算法

例如:
    用JDK7及以上jarsigner签名,不支持Android 4.2 以下
    jarsigner -keystore debug.keystore MyApp.apk release
    
    用JDK7及以上jarsigner签名,兼容Android 4.2 以下            
    jarsigner -keystore debug.keystore -digestalg SHA1 -sigalg SHA1withRSA MyApp.apk androiddebugkey
```




签名就OK，剩下安装测试就行了





参考

https://www.jianshu.com/p/53078d03c9bf

https://blog.csdn.net/dreamer2020/article/details/52761606

https://ibotpeaches.github.io/Apktool/
