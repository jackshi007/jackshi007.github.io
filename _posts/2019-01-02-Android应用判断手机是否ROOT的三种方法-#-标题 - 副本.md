---
layout:     post                    # 使用的布局（不需要改）
title:     Android应用判断手机是否ROOT的三种方法   # 标题 
subtitle:       如何查看手机是否ROOT #副标题
date:       2019-01-02              # 时间
author:     Jack                      # 作者
header-img: img/post-bg-tibet.jpg  #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android 
    - ROOT
---

# Android应用判断手机是否ROOT的三种方法

### 1.查看系统的Build Tags:

```java
private static boolean isRoot1() {
        String str = Build.TAGS;
        return str != null && str.contains("test-keys");
    }
```

### 2.查看system/app/下是否有Superuser

```java
private static boolean isRoot2() {
        return new File("/system/app/Superuser.apk").exists();
    }
```

### 3.查看系统各目录下是否有su文件

```java
private static boolean isRoot3() {
    for (String file : new String[]{"/sbin/su", "/system/bin/su", "/system/xbin/su", "/data/local/xbin/su", "/data/local/bin/su", "/system/sd/xbin/su", "/system/bin/failsafe/su", "/data/local/su"}) {
        if (new File(file).exists()) {
            return true;
        }
    }
    return false;
}
```

