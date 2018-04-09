---
layout:     post                    # 使用的布局（不需要改）
title:      [Android] VirtualXposed插件开发   # 标题 
subtitle:   Xposed hook 之入门案例 #副标题
date:       2018-04-09              # 时间
author:     Jack                      # 作者
header-img: img/post-bg-android.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
    - Xposed
    - Hook
    - VirtualApp
    
---


## [Android] Virtual Xposed插件开发：Xposed hook 之入门案例

### 一、什么是Virtual Xposed？
##### Xposed
众所周知Xposed是来自国外[XDA论坛](https://forum.xda-developers.com/)的rovo89开发的一款开源的安卓系统框架。

它是一款特殊的安卓App，其主要功能是提供一个新的应用平台，玩家们安装Xposed框架后，就能够通过Xposed框架搭建起的平台安装更多系统级的应用，实现诸多神奇的功能。

Xposed框架的原理是修改系统文件，替换了/system/bin/app_process可执行文件，在启动Zygote时加载额外的jar文件（/data/data/de.robv.android.xposed.installer/bin/XposedBridge.jar），并执行一些初始化操作(执行XposedBridge的main方法)。然后我们就可以在这个Zygote上下文中进行某些hook操作。

Xposed真正强大的是它可以hook调用的方法.当你反编译修改apk时,你可以在里面插入xposed的命令,于是你就可以在方法调用前后注入自己的代码.

Github开源地址: [https://github.com/rovo89/Xposed](https://github.com/rovo89/Xposed)

由于Xposed最大的弊端在于设备需要root，并且编写插件模块后需要重启手机（当然也有办法可以不用重启），所以有了VirtualApp。

##### VirtualApp
VirtualApp是一个App虚拟化引擎（简称VA）。

VirtualApp在你的App内创建一个虚拟空间（构造了一个虚拟的systemserver），你可以在虚拟空间内任意的安装、启动和卸载APK，这一切都与外部隔离，如同一个沙盒。

运行在VA中的APK无需在外部安装，即VA支持免安装运行APK。

熟悉android系统开机流程的应该知道各services是由system server启动一系列的系统核心服务（AMS,WMS,PMS等等）ViratualApp就是构建了一个虚拟system_process进程，这里面也有一系列的核心服务。

VirtualApp主要技术用到了反射和动态代理来实现的


Github开源地址：[https://github.com/asLody/VirtualApp](https://github.com/asLody/VirtualApp)

##### VirtualXposed
VirtualXposed就是基于VirtualApp和epic 在非ROOT环境下运行Xposed模块的实现（支持5.0~8.1)。

Github开源地址：[https://github.com/android-hacker/VirtualXposed](https://github.com/android-hacker/VirtualXposed)


### 二、编写Xposed插件
##### 1.编写测试app
先编写被劫持的测试app，测试劫持一个app的方法。
代码如下：

    
        textView = findViewById(R.id.text);
        button = findViewById(R.id.button);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                textView.setText("Jack");
            }
        });

点击button后textview显示Jack

##### 2.编写Xposed插件
首先导入Xposed的api库

方法1：
Android Studio的依赖：

    repositories {
        jcenter();
    }
    
    dependencies {
        provided 'de.robv.android.xposed:api:82'
    }

方法2: 下载jar包 

地址: [https://bintray.com/rovo89/de.robv.android.xposed/api](https://bintray.com/rovo89/de.robv.android.xposed/api)

Download有两个jar包: 

>api-82-sources.jar: 包含源码的 

>api-82.jar: 不包含源码


下载api-82.jar，复制到Android Studio的libs目录下，右键Add as library 

修改依赖方式compile files修改为provided fileTree,把 implementation 修改为 provided,原因是Xposed里已有该jar包内容，再次打包进去会冲突。

    dependencies {
        provided fileTree(include: ['*.jar'], dir: 'libs')
        implementation 'com.android.support:appcompat-v7:26.1.0'
        implementation 'com.android.support.constraint:constraint-layout:1.0.2'
        testImplementation 'junit:junit:4.12'
        androidTestImplementation 'com.android.support.test:runner:1.0.1'
        androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.1'
        provided files('libs/api-82.jar')
    }
##### 3.修改AndroidManifest.xml文件
在Application标签里面加三个meta-data

        <!-- 是否是xposed模块，xposed根据这个来判断是否是模块 -->
        <meta-data
            android:name="xposedmodule"
            android:value="true" />
        <!-- 模块描述，显示在xposed模块列表那里第二行 -->
        <meta-data
            android:name="xposeddescription"
            android:value="HookDemo" />
        <!-- 最低xposed版本号(lib文件名可知) -->
        <meta-data
            android:name="xposedminversion"
            android:value="53" />

##### 4.编写hook类
创建一个类，实现IXposedHookLoadPackage接口，重写handleLoadPackage方法，我这里创建了一个HookMain类。

代码如下：

    /**
     * Created by Jack on 2018/4/4.
     */
    
    public class HookMain implements IXposedHookLoadPackage{
        @Override
        public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) throws Throwable {
            XposedBridge.log("load app: " + lpparam.packageName);//显示加载的 app 名称
            if (lpparam.packageName.equals("jack.com.testxposed")){
            XposedBridge.log("开始Hook测试程序");
    
            XposedHelpers.findAndHookMethod(TextView.class, "setText", CharSequence.class, new XC_MethodHook() {
                /**
                * onCreate之前回调
                * @param param  onCreate方法的信息，可以修改
                * @throws Throwable
                */
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                XposedBridge.log("处理setText方法前");
                param.args[0]= "我是被Xposed修改的";
                }
                
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                XposedBridge.log("处理setText方法后");
                }
            });
          }
       }
    }
  
##### 5. 创建xposed_init文件
xposed_init 文件是 Xposed 模块的入口文件，Xposed 就是通过该文件找到对应的函数入口。


AS工程 app目录下右键，新建-folder-assets：
![](http://wx2.sinaimg.cn/mw690/b8fcdcc3gy1fpro2dbufsj20ls0h0dhs.jpg)  

新建 xposed_init,以文本格式打开，输入指定的 Hook 入口：


com.hookdemo.HookMain

##### 6.运行程序
首先将VirtualXposed安装到手机中

然后再VirtualXposed中添加测试应用以及hook模块

运行测试App查看结果

![image](http://wx4.sinaimg.cn/mw690/b5ec746bgy1fq6qsybklbg20gw0u0kjp.gif)




**参考链接：**

[https://blog.csdn.net/niubitianping/article/details/52571438](https://blog.csdn.net/niubitianping/article/details/52571438)