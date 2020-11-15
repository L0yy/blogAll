---
title: Lazagne导出密码
tags: [安全工具]
index_img: https://gitee.com//cve/BlogImg/raw/master/typora/image-20200317181302873.png
banner_img: https://gitee.com//cve/BlogImg/raw/master/typora/image-20200317181302873.png
---



### 前言

**LaZagne project 是一款用于检索大量存储在本地计算机上密码的开源应用程序。每款软件他们保存密码的方法或许不尽相同（明文，API，算法，数据库等等），开发这款工具的目的是为了寻找计算机中最常用软件的密码**

### 工具源码介绍

项目来源 https://github.com/AlessandroZ/LaZagne

增加了**360浏览器**的模块

目前已经支持提取密码的软件列表



|                                      | Windows                                                      | Linux                                                        | Mac                    |
| :----------------------------------: | :----------------------------------------------------------- | ------------------------------------------------------------ | ---------------------- |
|               Browsers               | 7Star<br> Amigo<br> BlackHawk<br> Brave<br> Centbrowser<br> Chedot<br> Chrome Canary<br> Chromium<br> Coccoc<br> Comodo Dragon<br> Comodo IceDragon<br> Cyberfox<br> Elements Browser<br> Epic Privacy Browser<br> Firefox<br> Google Chrome<br> Icecat<br> K-Meleon<br> Kometa<br> Opera<br> Orbitum<br> Sputnik<br> Torch<br> Uran<br> Vivaldi<br> 360Chrom<br> | Chrome<br> Firefox<br> Opera                                 | Chrome<br> Firefox     |
|                Chats                 | Pidgin<br> Psi<br> Skype                                     | Pidgin<br> Psi                                               |                        |
|              Databases               | DBVisualizer<br> Postgresql<br> Robomongo<br> Squirrel<br> SQLdevelopper | DBVisualizer<br> Squirrel<br> SQLdevelopper                  |                        |
|                Games                 | GalconFusion<br> Kalypsomedia<br> RogueTale<br> Turba        |                                                              |                        |
|                 Git                  | Git for Windows                                              |                                                              |                        |
|                Mails                 | Outlook<br> Thunderbird                                      | Clawsmail<br> Thunderbird                                    |                        |
|                Maven                 | Maven Apache<br>                                             |                                                              |                        |
|          Dumps from memory           | Keepass<br> Mimikatz method                                  | System Password                                              |                        |
|              Multimedia              | EyeCON<br>                                                   |                                                              |                        |
|                 PHP                  | Composer<br>                                                 |                                                              |                        |
|                 SVN                  | Tortoise                                                     |                                                              |                        |
|               Sysadmin               | Apache Directory Studio<br> CoreFTP<br> CyberDuck<br> FileZilla<br> FileZilla Server<br> FTPNavigator<br> OpenSSH<br> OpenVPN<br> KeePass Configuration Files (KeePass1, KeePass2)<br> PuttyCM<br>RDPManager<br> VNC<br> WinSCP<br> Windows Subsystem for Linux | Apache Directory Studio<br> AWS<br>  Docker<br> Environnement variable<br> FileZilla<br> gFTP<br> History files<br> Shares <br> SSH private keys <br> KeePass Configuration Files (KeePassX, KeePass2) <br> Grub |                        |
|                 Wifi                 | Wireless Network                                             | Network Manager<br> WPA Supplicant                           |                        |
| Internal mechanism passwords storage | Autologon<br> MSCache<br> Credential Files<br> Credman <br> DPAPI Hash <br> Hashdump (LM/NT)<br> LSA secret<br> Vault Files | GNOME Keyring<br> Kwallet<br> Hashdump                       | Keychains<br> Hashdump |



### 获取浏览器原理

浏览器在用户输入密码登录某个网站后，会有提示询问你是否保存密码，方便下次登录

![image-20200317133729443](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200317133729443.png)

这也存在安全问题，如果有人获得了执行shell的权限，读取浏览器中密码文件，完全可以通过撞库拿到更多的密码。

下面就360浏览器介绍怎么提取密码 

密码存储目录：

`%LOCALAPPDATA%\360Chrome\Chrome\User Data\Default\Login Data`

浏览器中使用数据库的方式保存账号，密码和对应的网站

通过sql管理工具打开，这里使用SQLiteStudio打开

![image-20200317135718468](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200317135718468.png)



在WIndows上

​	浏览器借助Windows内置的`CryptProtectData`函数对密码进行加密。现在，虽然这是使用三重DES算法并创建特定于用户的密钥来加密数据，但是只要您登录到与加密该数据的用户相同的帐户，就可以将其解密。功能有一个对应的API，与之相反。`CryptUnprotectData`，它解密数据。显然，这在尝试解密存储的密码时将非常有用。



> #### Mac/Linux Implementation
>
> Encryption Scheme: AES-128 CBC with a constant salt and constant iterations. The decryption key is a PBKDF2 key generated with the following:
>
> - salt is b'saltysalt'
> - key length is 16
> - iv is 16 bytes of space b' ' * 16
> - on Mac OSX:
>   - password is in keychain under Chrome Safe Storage
>     - I use the excellent keyring package to get the password
>     - You could also use bash: security find-generic-password -w -s "Chrome Safe Storage"
>   - number of iterations is 1003
> - on Linux:
>   - password is peanuts
>   - number of iterations is 1



接下来使用调用`CryptUnprotectData`进行解密就行了，网上代码也很多

**360浏览器和Google chrom保存密码的方式是一样的**

比如，python提取chrom密码

https://github.com/priyankchheda/chrome_password_grabber/blob/master/chrome.py



### 使用开发

该工程使用纯py编写，流程很容易看懂，看到现在还没支持360浏览器，但是用户数也挺多的，所以尝试增加以下这个模块。

**使用前请认真阅读ReadMe**

安装必要的库，我这里环境是**Python 2.7.13**

`pip install -r requirements.txt`

下面的每个文件夹都是不同软件的相关模块

![image-20200317180124260](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200317180124260.png)



要修改浏览器模块的数据，就要修改相应模块

因为有很多浏览器保存密码的方式都是形同的，这里作者分了4类 分别是**chromium_based** **ie**  **mozilla** **ucbrowser** 

大多数浏览器都是**chromium_based**格式存储密码，360浏览器也是

所以只需要增加一项配置文件就行了

https://github.com/AlessandroZ/LaZagne/blob/master/Windows/lazagne/softwares/browsers/chromium_based.py#L216

在里面新加一句

`(u'360ces', u'{LOCALAPPDATA}\\360Chrome\\Chrome\\User Data'),`

测试如下

![image-20200317181302873](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200317181302873.png)



不同环境库肯定不同，这里将他打包发布

可以使用py2exe或pyinstaller

由于py2exe不支持python2.7了，所以这里使用pyinstaller安装

`pip install pyinstaller`

也很简单，单文件模式输出就行

`pyinstaller -F  laZagne.py`

![image-20200317182426210](https://gitee.com//cve/BlogImg/raw/master/typora/image-20200317182426210.png)



最后拷贝dist下的成品exe就行了，但是因为用了import *的方式，所以很多无关的代码也写入了程序，这里暂时不能减少体积，如果要改，需要将每个py文件导入的模块细化，改为from _ import XXX 的格式，调用方式也需要修改。

