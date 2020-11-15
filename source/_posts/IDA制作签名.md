---
title: IDA制作签名
index_img: https://w.wallhaven.cc/full/96/wallhaven-96z68x.jpg
banner_img: https://w.wallhaven.cc/full/96/wallhaven-96z68x.jpg
date: 2019-08-18 19:11:22
tags:
    - IDA
categories: 逆向
---


我这里使用的是IDA6.8的制作签名包
如果我们知道是什么语言 的，但是没有签名包，就自己做一个

### 手工方式：

	1.pcf.exe source.lib XXXX.pcf
	2.sigmake.exe XXXX.pcf YYYY.sig
	这里一般会生成一个XXXX.exc的文件，打开它，删除以；开始的几行，保存退出
	3.再次执行sigmake.exe XXXX.pcf YYYY.sig，就会看到有YYYY.sig的签名文件了

然后把这个文件拷贝到IDA目录下的sig文件夹下面
接着重启IDA，拖入自己的项目，按shift + F5 打开签名文件管理器，右键添加就可以了

### 脚本
网上有脚本，但是运行不了，我小改了一下，效果还不错 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190808133028934.gif)

如果要制作单个sig的话，可以直接使用脚本中的lib2sig.bat 
用法：
	`lib2sig.bat vclibxxx.lib`
直接生成一个sig文件


链接: https://pan.baidu.com/s/16vuDXs298KBNT7LzRJnwLw 提取码: f2by 


