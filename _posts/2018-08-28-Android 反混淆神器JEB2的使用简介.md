---
layout:     post                    # 使用的布局（不需要改）
title:     Android 反混淆神器JEB2的使用简介   # 标题 
subtitle:   反混淆神器JEB2      #副标题
date:       2018-08-28              # 时间
author:     Jack                      # 作者
header-img: img/post-bg-sunrise.jpg  #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 反混淆
    - JEB2
    - Android 
---

# Android 反混淆神器JEB2的使用简介

## 背景

我们在逆向Android应用的时候会使用 **jd-GUI**、**jadx**、**Luyten**等工具进行代码审查，但市面上大部分的应用是混淆过的，简单点的混淆会使类名，方法名变成诸如 **a,b,c**之类的。更有甚者会自定义混淆规则将类名，方法名混淆成藏文，蒙古文等“天书”（后面我会写一篇自定义混淆规则的文章）。如何反混淆则是今天所要介绍的

## JEB2工具的基本使用

> 首先下载jeb2.2.5破解版
>
> 下载link：
> 链接：http://pan.baidu.com/s/1bJdWse 密码：ncr3	

Jeb支持Windows，Linux，Macos 系统我这里用的Windows系统所以点击jeb_wincon.bat，然后在JEB2中打开需要逆向的apk

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fx9x3yqh5zj20su0kp0vm.jpg)

双击Bytecode打开smali代码

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fx9x3ypxg2j215j0k5tar.jpg)

点击Bytecode/Hierarchy窗口即可查看包名树状图 树状图排列: 包名->类名

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fx9x3yxtouj20uq0gdwfx.jpg)

双击包名下的类名即可查看smali代码 

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fx9x3yp5vuj20r10dngmz.jpg)

想把smali代码转换成java代码 很简单 只需右键Q即可  

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fx9x3yu9vqj20tb0jvac4.jpg)

双击方法即可跳转到方法的定义 点击方法按x键可以查看方法的调用 

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fx9x3yqb2hj216m0h5wf6.jpg)

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fx9x3yoq6nj210g0d80tn.jpg)

双击Manifest即可查看AndroidManifest.xml

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fx9x3yw0khj20yi0b8tak.jpg)
以上就是JEB2的基本用法，接下来重点来了

## JEB2反混淆脚本

### 反混淆脚本思路

许多APK开发商为了在崩溃时保存源文件类名、行号等信息会在APK混淆时添加以下规则保留源文件信息.

（注意：若APK没有保留这些源信息时则无法反混淆）

```
-keepattributes SourceFile,LineNumberTable
```

这样我们在看Smali时就能在(JEB中称为)字段中看到原始类名信息.如下图所示:

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fx9xipizlrj20ce0283yp.jpg)

JEB2.2.x是默认不显示这些调试信息的可以根据以下步骤在设置中打开:Edit -> Options -> Engines -> 修改ShowDebugDirectives的值为true



这样我们就可以在JEB2中根据每个类的原始信息进行批量重命名达到反混淆的效果。

> 脚本下载地址：
>
> https://github.com/S3cuRiTy-Er1C/JebScripts

加载脚本之前先需要在 jeb225/scripts/目录下安装Jypthon环境，具体步骤如下：

> Setting up Jython:
> - Download a stand-alone Jython package from http://www.jython.org/downloads.html
>   We recommend either version 2.5 (fastest) or version 2.7 (latest)
> - Drop the downloaded 'jython-standalone-???.jar' file in the scripts/
>   sub-directory located in your JEB installation directory
> - Make sure that the client property '.ScriptsFolder' refers to that directory
>   (it is the case by default; use 'Edit/Options, Advanced...' to verify this)

我这边下载的是  **jython-standalone-2.5.4-rc1**

完成后打开JEB2 -> File -> Scripts -> Run Scripts -> 选择从上面下载的 JEB2DeobscureClass.py脚本



## 效果

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fx9xtt39xqj20bp0eggmc.jpg)

这样看起来是不是清楚多了？

O(∩_∩)O哈哈~