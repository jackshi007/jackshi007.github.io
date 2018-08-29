---
layout:     post                    # 使用的布局（不需要改）
title:     JNI 中 C\C++ 反射调用Java方法    # 标题 
subtitle:  JNI 中 C\C++ 反射调用Java方法的基本使用       #副标题
date:       2018-07-03               # 时间
author:     Jack                      # 作者
header-img: img/post-bg-fantasy03.jpg  #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
    - JNI
    - 反射 
---

# JNI 中 C\C++ 反射调用Java方法

> 反射一般分3个步骤： 
>
> 1.加载class（字节码），获取class的对象。
>
> 2.获取对应的方法或属性。 
>
> 3.修改属性，或执行方法。 

关于java的反射可以去另一篇文章查看→[传送门](http://jackzhang.info/2018/05/15/JAVA%E5%8F%8D%E5%B0%84%E6%9C%BA%E5%88%B6/)

本篇文章主要记录一下在JNI中用C/C++去反射调用java的方法。

> 关于env，在C和C++语言中调用方式是不同的，如下，本例使用的C++语言
>
> 在C中：
> (*env)->方法名(env,参数列表)
> 在C++中：
> env->方法名(参数列表)

C代码反射调用Java方法步骤：
①获取字节码对象

```c
jclass FindClass(const char* name)
```

②通过字节码对象找到方法对象

```c
jmethodID GetMethodID(jclass clazz, const char* name, const char* sig)
```

③通过字节码文件创建一个object对象（该方法可选，方法中已经传递一个object，如果需要调用的方法与本地方法不在同一个文件夹则需要新创建

```c
jobject AllocObject(jclass clazz)
```

如果需要反射调用的java方法与本地方法不在同一个类中，需要创建该方法，但是如果是这样，并且需要更新 UI操作，例如打印一个Toast 会报空指针异常，因为这时候调用的方法只是一个方法，没有actiivty的生命周期。（没有上下文环境就在创建方法的时候在构造方法中接收一个。 ）

④通过对象调用方法，可以调用空参数方法，也可以调用有参数方法，并且将参数通过调用的方法传入

```c
void CallVoidMethod(jobject obj, jmethodID methodID, ...)
```

 

 下面我们来看一下代码

```java
package com.android.sohook;

import android.content.Context;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.TextView;
import android.widget.Toast;

public class MainActivity extends AppCompatActivity implements View.OnClickListener{

    private Context context;
    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        context = this;
        // Example of a call to a native method
        TextView tv = (TextView) findViewById(R.id.sample_text);
        tv.setText(stringFromJNI());
        findViewById(R.id.bt_javaInt).setOnClickListener(this);
        findViewById(R.id.bt_javanull).setOnClickListener(this);
        findViewById(R.id.bt_javaString).setOnClickListener(this);
        findViewById(R.id.bt_static).setOnClickListener(this);
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()){
            case R.id.bt_javanull:
                callbackmethod();
                break;
            case R.id.bt_javaInt:
                callbackIntmethod();
                break;
            case R.id.bt_javaString:
                callbackStringmethod();
                break;
            case R.id.bt_static:
                callStaticmethod();
                break;
        }
    }

    /**
     * A native method that is implemented by the 'native-lib' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();
    //C++调用java空方法
    public native void callbackmethod();
    //C++调用java中的带两个int参数的方法
    public native int callbackIntmethod();
    //C++调用java中参数为string的方法
    public native void callbackStringmethod();
    //C++调用java中静态方法
    public native void callStaticmethod();

    public void helloFromJava(){
        Toast.makeText(context, "C++调用了java的空方法",Toast.LENGTH_SHORT ).show();}
    //C调用java中的带两个int参数的方法
    public int add(int x,int y) {
        return x+y;}
    //C调用java中参数为string的方法
    public void printString(String s){
        Toast.makeText(context, s, Toast.LENGTH_SHORT).show();}
    //C调用java中静态方法
    public static void staticmethod(String s){
        Log.w("MainActivity",s+"=======静态方法");}

}
```

 

下面来编写natvie-lib.cpp

```c++
#include <jni.h>
#include <string>
#include <android/log.h>
#define LOG_TAG "Native-lib"
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, __VA_ARGS__)

extern "C" {

JNIEXPORT jstring

JNICALL
Java_com_android_sohook_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}

JNIEXPORT void JNICALL
Java_com_android_sohook_MainActivity_callbackmethod(JNIEnv *env, jobject object) {
    jclass jclazz = env->FindClass("com/android/sohook/MainActivity");
    jmethodID methodID = env->GetMethodID(jclazz, "helloFromJava", "()V");
    env->CallVoidMethod(object, methodID);
}

JNIEXPORT jint JNICALL
Java_com_android_sohook_MainActivity_callbackIntmethod(
        JNIEnv *env,
        jobject object) {
    jclass jclazz = env->FindClass("com/android/sohook/MainActivity");
    jmethodID methodID = env->GetMethodID(jclazz, "add", "(II)I");
    int result = env->CallIntMethod(object, methodID, 2, 3);
    LOGD("RESLUT = %d", result);
    return result;
}

JNIEXPORT void JNICALL
Java_com_android_sohook_MainActivity_callbackStringmethod(
        JNIEnv *env,
        jobject object) {
    jclass jclazz = env->FindClass("com/android/sohook/MainActivity");
    jmethodID methodID = env->GetMethodID(jclazz, "printString", "(Ljava/lang/String;)V");
    env->CallVoidMethod(object, methodID, env->NewStringUTF("I'm from JNI C++"));
}

JNIEXPORT void JNICALL
Java_com_android_sohook_MainActivity_callStaticmethod(
        JNIEnv *env,
        jobject object) {
    jclass jclazz = env->FindClass("com/android/sohook/MainActivity");
    jmethodID methodID = env->GetStaticMethodID(jclazz, "staticmethod", "(Ljava/lang/String;)V");
    jstring str = env->NewStringUTF("C++调用java");
    env->CallStaticVoidMethod(jclazz, methodID, str);
}
}
```

 获取方法签名的方法是进入工程目的 ..../build/classes/debug 进入控制台， 输入命令 javap -s 要获取方法的路径（例如本例 javap -s com.android.sohook.MainActivity） 

![](http://wx1.sinaimg.cn/mw690/b5ec746bgy1fuaconkerzj20w40otn16.jpg)

> JNI类型映射

| **Java** **类型** | **本地类型**  | **描述**                                 |
| ----------------- | ------------- | ---------------------------------------- |
| boolean           | jboolean      | C/C++8位整型                             |
| byte              | jbyte         | C/C++带符号的8位整型                     |
| char              | jchar         | C/C++无符号的16位整型                    |
| short             | jshort        | C/C++带符号的16位整型                    |
| int               | jint          | C/C++带符号的32位整型                    |
| long              | jlong         | C/C++带符号的64位整型e                   |
| float             | jfloat        | C/C++32位浮点型                          |
| double            | jdouble       | C/C++64位浮点型                          |
| Object            | jobject       | 任何Java对象，或者没有对应java类型的对象 |
| Class             | jclass        | Class对象                                |
| String            | jstring       | 字符串对象                               |
| Object[]          | jobjectArray  | 任何对象的数组                           |
| boolean[]         | jbooleanArray | 布尔型数组                               |
| byte[]            | jbyteArray    | 比特型数组                               |
| char[]            | jcharArray    | 字符型数组                               |
| short[]           | jshortArray   | 短整型数组                               |
| int[]             | jintArray     | 整型数组                                 |
| long[]            | jlongArray    | 长整型数组                               |
| float[]           | jfloatArray   | 浮点型数组                               |
| double[]          | jdoubleArray  | 双浮点型数组                             |

> JNI数组存取函数

| **函数**                | **Java** **数组类型** | **本地类型** |
| ----------------------- | --------------------- | ------------ |
| GetBooleanArrayElements | jbooleanArray         | jboolean     |
| GetByteArrayElements    | jbyteArray            | jbyte        |
| GetCharArrayElements    | jcharArray            | jchar        |
| GetShortArrayElements   | jshortArray           | jshort       |
| GetIntArrayElements     | jintArray             | jint         |
| GetLongArrayElements    | jlongArray            | jlong        |
| GetFloatArrayElements   | jfloatArray           | jfloat       |
| GetDoubleArrayElements  | jdoubleArray          | jdouble      |

> 域和方法的函数

| **函数**          | 描述                   |
| ----------------- | ---------------------- |
| GetFieldID        | 得到一个实例的域的ID   |
| GetStaticFieldID  | 得到一个静态的域的ID   |
| GetMethodID       | 得到一个实例的方法的ID |
| GetStaticMethodID | 得到一个静态方法的ID   |

> 确定域和方法签名的符号
>

| **Java** **类型** | **签名符号**                                  |
| ----------------- | --------------------------------------------- |
| boolean           | Z                                             |
| byte              | B                                             |
| char              | C                                             |
| short             | S                                             |
| int               | I                                             |
| long              | L                                             |
| float             | F                                             |
| double            | D                                             |
| void              | V                                             |
| objects对象       | Lfully-qualified-class-name;L类名             |
| Arrays数组        | [array-type [数组类型                         |
| methods方法       | (argument-types)return-type(参数类型)返回类型 |

```java
externalNativeBuild {   
 cmake {       
 cppFlags ""       
 // Clang是一个C语言、Objective-C、C++语言的轻量级编译器。        
arguments "-DANDROID_TOOLCHAIN=clang"        
// 生成.so库的目标平台        
abiFilters "armeabi-v7a" , "armeabi" ,"x86"    
        }
}
```



最后附上运行结果：

![](http://wx2.sinaimg.cn/mw690/b5ec746bgy1fuacot0r7aj21401z40vi.jpg)