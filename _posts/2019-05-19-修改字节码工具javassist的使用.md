---
layout:     post                    # 使用的布局（不需要改）
title:     修改字节码工具javassist的使用    # 标题 
subtitle:  在不重新编译的情况下直接修改Java Class文件中的内容  #副标题
date:       2019-05-19              # 时间
author:     Jack                      # 作者
header-img: img/post-bg-mountain.jpg  #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Java
    - 反编译
    - javassist



---

# 修改字节码工具javassist的使用

## 1.简介

> Javassist是一个提供简单接口操作java字节码的工具。
>它可以让你在一个已经编译好的类中添加新的方法，或者是修改已有的方法。
> 它不要求你对字节码方面具有多么深入的了解，同样的，它也允许你忽略被修改的类本身的细节和结构。

javassist官网：[http://www.javassist.org/](http://www.javassist.org/)

github地址：[https://github.com/jboss-javassist/javassist](https://github.com/jboss-javassist/javassist)

## 2.主要的类

- ClassPool
- CtClass
- CtMethod
- CtField

## 3.主要的方法

- CtClass.addMethod
- CtClass.removeMethod
- CtClass.removeField
- CtClass.writeFile
- CtClass.addField
- CtMethod.insertBefore
- CtMethod.insertAfter
- CtMethod.insertAt
- CtMethod.setBody

## 4.修改Class文件中的方法

```java
package com.lucumt;

public class Test1 {
    public static void main(String[] args) {
        Test1 t1 = new Test1();
        int result = t1.addNumber(3, 5);
        System.out.println("result is: "+result);
    }

    public int addNumber(int a,int b){
        return a+b;
    }
}
```

正常情况下，其输出结果如下
[![未修改方法前的运行结果](https://lucumt.info/blog_img/modify-java-class-file-content-directly/java-method-before-modifing-running-result.png)](https://lucumt.info/blog_img/modify-java-class-file-content-directly/java-method-before-modifing-running-result.png)
若我们想将 *addNumber* 的返回结果从两个数之和变为两个数立方后求和，则可以利用Javassist提供的API通过Java程序来直接修改class文件。

关于如何使用Javassist，请直接参看相应的 **入门教程** ，本文不再详细说明，利用Javassist修改 *addNumber*的Java代码如下：



```java
package com.lucumt;

import java.io.IOException;

import javassist.CannotCompileException;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtField;
import javassist.CtMethod;
import javassist.CtNewMethod;
import javassist.NotFoundException;

public class UpdateMethod {
    
    private static String pathName = "D:\\Java\\xxxxx\\test\\bin";
    private static String className = "com.lucumt.Test1";
    
    public static void main(String[] args) {
        updateMethod();
    }

    public static void updateMethod(){
        try {
            ClassPool cPool = new ClassPool(true);
            //如果该文件引入了其它类，需要利用类似如下方式声明
            //cPool.importPackage("java.util.List");

            //设置class文件的位置
            cPool.insertClassPath(pathName);

            //获取该class对象
            CtClass cClass = cPool.get(className);

            //获取到对应的方法
            CtMethod cMethod = cClass.getDeclaredMethod("addNumber");

            //更改该方法的内部实现
            //需要注意的是对于参数的引用要以$开始，不能直接输入参数名称
            cMethod.setBody("{ return $1*$1*$1+$2*$2*$2; }");

            //替换原有的文件
            cClass.writeFile(pathName);

            System.out.println("=======change finish=========");
            
        } catch (NotFoundException e) {
            e.printStackTrace();
        } catch (CannotCompileException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}

```

运行该代码后重新执行 *Test1* 后的结果如下，从图中可以看出运行结果符合预期
[![修改方法后的运行结果](https://lucumt.info/blog_img/modify-java-class-file-content-directly/java-method-after-modifing-runnning-result.png)](https://lucumt.info/blog_img/modify-java-class-file-content-directly/java-method-after-modifing-runnning-result.png)

关于 *UpdateMethod* 工具类有如下几点说明：

- 如果要修改的class文件中引入了其它类，需要调用 *ClassPool* 中的 *importPackage* 方法引入该类，否则程序会报错
- 修改完后，一定要调用 *CtClass* 中的 *writeFile* 方法覆盖原有的class文件，否则修改不生效
- 在修改方法的过程中若要引用方法参数，不能在修改程序代码中直接写该参数，否则程序会抛出*javassist.CannotCompileException: [source error] no such field:* 异常。在本例中 *addNumber* 的两个参数分别为 *a* 和 *b* ，在修改时不能写成`cMethod.setBody("{ return a*a*a+b*b*b; }")`需要修改为`cMethod.setBody("{ return $1*$1*$1+$2*$2*$2; }")`
- 在Javassist的 **Introspection and customization** 部分有如下一段话
  *The parameters passed to the target method are accessible with $1, $2, … instead of the original parameter names. $1 represents the first parameter, $2 represents the second parameter, and so on. The types of those variables are identical to the parameter types. $0 is equivalent to this. If the method is static, $0 is not available.*
  从中可知，方法中的**参数从 *$1* 开始，若该方法为非 *static* 方法，可以用 *$0* 来表示该方法实例自身，若该方法为 *static* 方法，则 *$0* 不可用**

## 5.在Class文件中增加方法

```java
public static void addMethod(){
        try {
            ClassPool cPool = new ClassPool(true);
            cPool.insertClassPath(pathName);
            CtClass cClass = cPool.get(className);

            CtMethod cMethod = cClass.getDeclaredMethod("addNumber");

            //增加一个新方法
            String methodStr ="public void showParameters(int a,int b){"
                    +"  System.out.println(\"First parameter: \"+a);"
                    +"  System.out.println(\"Second parameter: \"+b);"
                    +"}";
            CtMethod newMethod = CtNewMethod.make(methodStr, cClass);
            cClass.addMethod(newMethod);

            //调用新增的方法
            cMethod.setBody("{ showParameters($1,$2);return $1*$1*$1+$2*$2*$2; }");
            cClass.writeFile(pathName);

        } catch (NotFoundException e) {
            e.printStackTrace();
        } catch (CannotCompileException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

运行该代码后重新执行 *Test1* 后的结果如下，从图中可以看出运行结果符合预期
[![新增方法后的运行结果](https://lucumt.info/blog_img/modify-java-class-file-content-directly/java-add-method-runnning-result.png)](https://lucumt.info/blog_img/modify-java-class-file-content-directly/java-add-method-runnning-result.png)
从上述代码可以看出，利用Javassist增加方法比修改方法更简单，先将要新增的方法内容赋值到字符串，然后分别调用相关类的 *make* 和 *addMethod* 方法即可。



## 6.在Class文件中增加成员变量

```java
public static void addField(){
        try {
            ClassPool cPool = new ClassPool(true);
            cPool.insertClassPath(pathName);
            CtClass cClass = cPool.get(className);

            //增加一个新成员变量
            cClass.addField(CtField.make("private String str;",cClass));

            cClass.writeFile(pathName);

        } catch (NotFoundException e) {
            e.printStackTrace();
        } catch (CannotCompileException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

## 注意事项

- 对类的方法进行修改时，引用方法的参数要用`$1`来代替
- 对类的方法进行修改时，增加的代码好像不能引用原方法中的局部变量，写一样的变量名也不行
- 对类的方法进行修改时，增加的代码的新增类，要写完整的类名，就算原来的类中有import。
- 更高级的使用，见javassist的tutorial



## 参考链接

[javassist官网](https://jboss-javassist.github.io/javassist/tutorial/tutorial.html)

[在不重新编译的情况下直接修改Java Class文件中的内容](https://lucumt.info/post/modify-java-class-file-content-directly)

[修改字节码工具javassist的使用小记](https://blog.fengcl.com/2017/06/17/use-of-javassist)



