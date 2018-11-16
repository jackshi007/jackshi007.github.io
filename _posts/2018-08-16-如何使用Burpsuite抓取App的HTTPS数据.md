---
layout:     post                    # 使用的布局（不需要改）
title:     如何使用Burpsuite抓取App的HTTPS数据    # 标题 
subtitle:  抓取App的HTTPS数据       #副标题
date:       2018-08-16               # 时间
author:     Jack                      # 作者
header-img: img/post-bg-tree.jpg  #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Burpsuite
    - 抓包工具
    - HTTPS 
---

# 如何使用Burpsuite抓取App的HTTPS数据



## 概述

**Burp Suite**是一个用于测试Web应用程序安全性的图形工具。该工具使用[Java](https://en.wikipedia.org/wiki/Java_(programming_language))编写，由PortSwigger Web Security开发。

## 工具

- **Proxy** - 它作为Web代理服务器运行，并且作为浏览器和目标Web服务器之间的中间人。这允许拦截，检查和修改在两个方向上传递的原始业务。
- **Scanner** - 一种Web应用程序安全扫描程序，用于执行Web应用程序的自动漏洞扫描。
- **Intruder** - 此工具可以对Web应用程序执行自动攻击。该工具提供了可配置的[算法](https://en.wikipedia.org/wiki/Algorithm)，可以生成恶意HTTP请求。入侵者工具可以测试和检测[SQL注入](https://en.wikipedia.org/wiki/SQL_Injection)，[跨站点脚本](https://en.wikipedia.org/wiki/Cross_Site_Scripting)，参数操作和易受暴力攻击的漏洞。
- **Spider** - 用于自动抓取Web应用程序的工具。它可以与手动映射技术结合使用，以加快映射应用程序内容和功能的过程。
- **Repeater** - 一种可用于手动测试应用程序的简单工具。它可用于修改对服务器的请求，重新发送它们并观察结果。
- **Decoder** - 用于将编码数据转换为规范形式，或将原始数据转换为各种编码和散列形式的工具。它能够使用启发式技术智能地识别多种编码格式。
- **Comparer** - 用于在任意两项数据之间执行比较（差异）的工具。
- **扩展器** - 允许安全测试人员加载Burp扩展，使用安全测试人员自己或第三方代码扩展Burp的功能[（BAppStore）](https://portswigger.net/bappstore/)
- **Sequencer** - 用于分析数据项样本中随机性质的工具。它可用于测试应用程序的会话令牌或其他意图不可预测的重要数据项，例如反CSRF令牌，密码重置令牌等。

## 使用

### Proxy模块（代理模块）

1.Proxy->Intercept，Intercept in on(默认开启，表示拦截请求)，点击后变成Intercept id off(关闭拦截)。

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fwygxf3aqbj20ew066wes.jpg)

2.Proxy->Options->Proxy Listeners，检查代理设置，设置成所有IP都能访问，点击127.0.0.1:8080，Edit，选择All interfaces，点击OK；

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fwygxf4dcoj20oo0ev0ty.jpg)

3.Proxy->HTTP history，点击Filter栏进行过滤设置。

Filter by Search term(过滤查询搜索)，在输入框中输入(Host: \w+.xxxxx.com)，勾选Regex。【表示通过正则匹配host为*.xxxx.com的】
 Filter by file extension(过滤文件扩展名)，勾选Hide。【表示过滤js，css，png等相关文件】

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fwygxf3m4yj20ok0by75k.jpg)

### 安装证书抓取HTTPS

在电脑端使用Firefox浏览器访问设置的代理ip:端口，下载burpsuite证书，比如我上面的ip为192.168.1.105，端口为8080，就访问<<http://192.168.1.105:8080/>>然后去下载证书

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fwygxf3h2ij21gq04a3yx.jpg)

点击CA certificate下载burpsuite的证书，保存证书文件

**将下载后的证书重命名为xxx.cer（注意后缀改为cer）由于我们是要在Android手机上安装此证书所以需要将证书后缀改为cer，否则无法抓取HTTPS的数据。**

**将修改过的xxx.cer证书push到手机上，然后在Setting -> Security -> Install from storage 中选择xxx.cer安装到手机上**

### 在手机上配置代理服务器

首先确保安装Burpsuite服务器的网络与手机Wifi连接在同一局域网内。

进入Setting-> Wi-Fi ->长按当前连接的wifi，选择修改网络，勾选显示高级选项，然后选择代理设置为手动代理服务器主机名字填电脑ip，端口填你刚刚设置的端口（默认为8080）。然后确定，就设置成功了

![](https://ws1.sinaimg.cn/large/b5ec746bgy1fwygxf578yj20f00qo40e.jpg)

设置完成后就可以在BurpSuite上查看Https的网络数据了