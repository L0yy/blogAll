---
title: H-WORM变种远控分析
date: 2019-09-04 19:11:22
index_img: https://dc.snscz.com/s2/img/1200/2019/04/01/14/14004_4c746d4df0.jpg
banner_img: https://dc.snscz.com/s2/img/1200/2019/04/01/14/14004_4c746d4df0.jpg
tags:
    - Rat H-worm
categories: 样本详细分析
---



## 基本信息

|FileName| FileType|MD5|Size|
|--|--|--|--|
| 4gdrwceq60b7dbl.sct| rat  |69B7D326575C5616D82645960B3D081A|403845 bytes|

## 简介
该样本语言类型为 VBS和JS编写，在一定程序上能躲避安全软件的查杀，通过不同混淆更容易达到免杀的效果。


## 流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916190802115.png)
## 详细分析
这个样本母体总共会释放3个脚本
### 脚本1
首先我们看看`4gdrwceq60b7dbl.sct`，这个样本的母体，是一个sct格式的文件这是Visual FoxPro的表单配置文件
我们用`Sublimi Text`来打开它，记事本也是可以的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916155338357.png)
利用组数来存关键字符串，通过字符串的拼接来合成完整路径
会尝试删除`%APPDATA%\\taskmgr.js`  这个文件
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916155609819.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916160236624.png)
文件中有很长一段密文，这里我改名为了`EncodeList`，通过`System.IO.MemoryStream.WriteByte()`方法把数据读入数据流中。
最重要的是使用`ADODB.Stream`类方法把这个数据流保存到了被上面删除掉的`%APPDATA%\\taskmgr.js`文件中
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916155832544.png)
最后使用`WScript.Shell.Run`方法运行`taskmgr.js`文件

我这里使Python给他拿出来，命名为`taskmgr.js`
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916161513776.png)
### 脚本2
下面是`taskmgr.js` ，这个脚本其实还是一个解密脚本，也是 使用`AdoDB.Stream`处理数据流，然后使用`monKeyKing`中动态生成的eval方法去执行数据，来达到免杀
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916162014581.png)
通过一个`Microsoft.XmlDom`的创建一个`tmp`对象，然后把这个对象赋予`eval`方法
最后就可以用`this.p`去访问到`Microsoft.XmlDom.tmp`，来达到命令执行
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916162041325.png)
当然执行的命令也肯定是被加过密的，下面就是解密数据部分，这里就简单的把base64后的结果，然后把字符"A"替换成"!-%"
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916163628179.png)

最后使用上面构造的`eval`去执行这段解密的代码
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019091616391031.png)

### 脚本3 核心代码
这里开始分析脚本2中解密出来的代码

根据这host，port和代码量，可以判断出这个应该就是这个样本的实现核心的地方了。
这里其实这是一个远控加感染U盘传播
#### U盘传播
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916165932567.png)
加入开机启动注册表项，来达到持久化攻击，这基本已是木马的共性
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916170039587.png)

在下面的try块中，有一个install函数会被循环调用，这个函数是用来检测感染移动存储设备的
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916170300961.png)
install() 会去判断当前系统磁盘是否有U盘或其他移动存储介质，有的话就执行感染操作
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916170657111.png)
枚举移动盘文件
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916171010711.png)
如果文件不是快捷方式，那就将权限改为**隐藏**和**系统权限**，然后给他们创建一个同名`.lnk`的快捷方式
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916171254264.png)
快捷方式被修改为了特殊构造的恶意代码，构造过程如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916171656803.png)
这样每个文件都会被隐藏，然后生成一个被感染的快捷方式去诱惑其他人点击，只要一点击，就会被感染。

#### 远控模块

这个远控一共有`26`个命令模块，分别是
|命令  |含义  |
|--|--|
|disconnect|   断开连接 |
|reboot|   重启脚本 |
|shutdown| 关闭脚本 |
|excecute| 执行命令  |
|install-sdk|  安装SDK |
|get-pass|搜集密码  |
|get-pass-offline|  获取浏览器密码|
|update|    更新|
|uninstall| 卸载|
|up-n-exec| 请求下载执行文件|
|bring-log| 上传日志|
|down-n-exec|  下载执行文件 |
|filemanager|   文件管理|
|rdp|  启动RDP |
|keylogger| 启动kleylogger|
|offline-keylogger| 启动离线版kerlogger|
|browse-logs|   上传日志|
|cmd-shell| 执行cmd命令|
|get-processes| 枚举进程|
|disable-uac|   关闭UAC|
|check-eligible|    检测权限|
|force-eligible| 暴力提权   |
|elevate|  普通提权 |
|if-elevate|   检测权限 |
|kill-process| 结束进程 |
|sleep| 挂起进程|

这个程序会释放一个脚本到%appdata%下去，命名为`aFCnKVCdfY.js`
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916164502275.png)

### 脚本4
aFCnKVCdfY.js这个和脚本三是一样的，除了域名变了，其他都没变化。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190916174051698.png)



## 样本溯源
**C&C**
unknownsoft.duckdns.org:7744
globalization.duckdns.org:50071


## 查杀方案
删除自启动表项
HKEY_CURRENT_USER\\software\\microsoft\\windows\\currentversion\\run\\scriptName
HKEY_LOCAL_MACHINE\\software\\microsoft\\windows\\currentversion\\run\\scriptName

删除
%appdata%\\aFCnKVCdfY.js
%temp%\\aFCnKVCdfY.js
%APPDATA%\\taskmgr.js 

## 总结
谨防不明来路的邮件，网页 文件要谨慎，要对未知文件表示怀疑，切勿怀着试一试的态度打开

