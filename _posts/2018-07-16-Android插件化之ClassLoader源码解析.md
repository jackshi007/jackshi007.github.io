---
layout:     post                    # 使用的布局（不需要改）
title:     Android插件化之ClassLoader源码解析    # 标题 
subtitle:  ClassLoader源码解析       #副标题
date:       2018-07-20               # 时间
author:     Jack                      # 作者
header-img: img/post-bg-fantasy02.jpg  #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
    - 插件化
    - 源码解析 
---

# Android插件化之ClassLoader源码解析

> Java代码会编译成.Class文件，程序运行在虚拟机上时，虚拟机需要把需要的Class加载进来才能创建实例对象并工作，而完成这一个加载工作的角色就是ClassLoader。
>
> Android的Dalvik/ART虚拟机如同标准JAVA的JVM虚拟机一样，在运行程序时首先需要将对应的类加载到内存中。

### **ClassLoader实例有多个**

动态加载的基础是ClassLoader，从名字也可以看出，ClassLoader就是专门用来处理类加载工作的，所以这货也叫类加载器，而且一个运行中的APP 不仅只有一个类加载器。

其实，在Android系统启动的时候会创建一个Boot类型的ClassLoader实例，用于加载一些系统Framework层级需要的类，我们的Android应用里也需要用到一些系统的类，所以APP启动的时候也会把这个Boot类型的ClassLoader传进来。

此外，APP也有自己的类，这些类保存在APK的dex文件里面，所以APP启动的时候，也会创建一个自己的ClassLoader实例，用于加载自己dex文件中的类。

一个是BootClassLoader（系统启动的时候创建的），另一个是PathClassLoader（应用启动时创建的）

### **创建自己ClassLoader实例** 

动态加载外部的dex文件的时候，我们也可以使用自己创建的ClassLoader实例来加载dex里面的Class，不过ClassLoader的创建方式有点特殊，我们先看看它的构造方法

```java
/*      * constructor for the BootClassLoader which needs parent to be null.      */
ClassLoader(ClassLoader parentLoader, boolean nullAllowed) {
    if (parentLoader == null && !nullAllowed) {
        throw new NullPointerException("parentLoader == null && !nullAllowed");
    }
    parent = parentLoader;
}
```

创建一个ClassLoader实例的时候，需要使用一个现有的ClassLoader实例作为新创建的实例的Parent。这样一来，一个Android应用，甚至整个Android系统里所有的ClassLoader实例都会被一棵树关联起来，这也是ClassLoader的 **双亲代理模型（Parent-Delegation Model）**的特点。

### **ClassLoader双亲代理模型加载类的特点和作用** 

JVM中ClassLoader通过defineClass方法加载jar里面的Class，而Android中这个方法被弃用了。

```java
@Deprecated
protected final Class<?> defineClass(byte[] classRep, int offset, int length) throws ClassFormatError {
    throw new UnsupportedOperationException("can't load this type of class file");
}
```

取而代之的是loadClass方法

```java
public Class<?> loadClass(String className) throws ClassNotFoundException {
       return loadClass(className, false);
   }
   protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
       Class<?> clazz = findLoadedClass(className);//查看类是否以前被加载个
        if (clazz == null) {
           ClassNotFoundException suppressed = null;
           try {
               clazz = parent.loadClass(className, false);//先调用父类加载器去加载，可有效加载相关系统类（这个是Android插件化的一个重要机制）
           } catch (ClassNotFoundException e) {
               suppressed = e;
           }
            if (clazz == null) {
               try {
                   clazz = findClass(className);//父类加载没有找到类 就调用自己的findClass机制
               } catch (ClassNotFoundException e) {
                   e.addSuppressed(suppressed);
                   throw e;
               }
           }
       }
        return clazz;
   }
```

从源码中我们也可以看出，loadClass方法在加载一个类的实例的时候 
1) 会先查询当前ClassLoader实例是否加载过此类，有就返回；

2) 如果没有。查询Parent是否已经加载过此类，如果已经加载过，就直接返回Parent加载的类；

3) 如果继承路线上的ClassLoader都没有加载，才由Child执行类的加载工作；

这样做有个明显的特点，如果一个类被位于树根的ClassLoader加载过，那么在以后整个系统的生命周期内，这个类永远不会被重新加载。

### **DexClassLoader.java源码：**

```java
public class DexClassLoader extends BaseDexClassLoader {
    /**
     * Creates a {@code DexClassLoader} that finds interpreted and native
     * code.  Interpreted classes are found in a set of DEX files contained
     * in Jar or APK files.
     *
     * <p>The path lists are separated using the character specified by the
     * {@code path.separator} system property, which defaults to {@code :}.
     *
     * @param dexPath the list of jar/apk files containing classes and
     *     resources, delimited by {@code File.pathSeparator}, which
     *     defaults to {@code ":"} on Android
     * @param optimizedDirectory directory where optimized dex files
     *     should be written; must not be {@code null}
     * @param libraryPath the list of directories containing native
     *     libraries, delimited by {@code File.pathSeparator}; may be
    *     {@code null}
     * @param parent the parent class loader
     */
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
}
```

> DexClassLoader构造方法参数说明：
>
> dexPath:apk/dex/jar文件路径
>
> optimizedDirectory:文件解压路径（这个路径下保存的是.dex文件不是.class）
>
> libraryPath:加载时所需的so库
>
> parent:父加载器（这个比较重要与Android加载class的机制有关）

从DexClassLoader源码可以知道 DexClassLoader直接实现的是BaseDexClassLoader ，所以需要我们进一步去查看BaseDexClassLoader 的源码

```java
/*
* Copyright (C) 2011 The Android Open Source Project
*
* Licensed under the Apache License, Version 2.0 (the "License");
* you may not use this file except in compliance with the License.
* You may obtain a copy of the License at
*
*      http://www.apache.org/licenses/LICENSE-2.0
*
* Unless required by applicable law or agreed to in writing, software
* distributed under the License is distributed on an "AS IS" BASIS,
* WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
* See the License for the specific language governing permissions and
* limitations under the License.
*/
package dalvik.system;
import java.io.File;
import java.io.IOException;
import java.net.MalformedURLException;
import java.net.URL;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.Enumeration;
import java.util.List;
import java.util.zip.ZipFile;
import libcore.io.ErrnoException;
import libcore.io.IoUtils;
import libcore.io.Libcore;
import libcore.io.StructStat;
import static libcore.io.OsConstants.*;
/**
* A pair of lists of entries, associated with a {@code ClassLoader}.
* One of the lists is a dex/resource path &mdash; typically referred
* to as a "class path" &mdash; list, and the other names directories
* containing native code libraries. Class path entries may be any of:
* a {@code .jar} or {@code .zip} file containing an optional
* top-level {@code classes.dex} file as well as arbitrary resources,
* or a plain {@code .dex} file (with no possibility of associated
* resources).
*
* <p>This class also contains methods to use these lists to look up
* classes and resources.</p>
*/
/*package*/ final class DexPathList {
   private static final String DEX_SUFFIX = ".dex";
   private static final String JAR_SUFFIX = ".jar";
   private static final String ZIP_SUFFIX = ".zip";
   private static final String APK_SUFFIX = ".apk";
   /** class definition context */
   private final ClassLoader definingContext;
   /**
    * List of dex/resource (class path) elements.
    * Should be called pathElements, but the Facebook app uses reflection
    * to modify 'dexElements' (http://b/7726934).
    */
   private final Element[] dexElements;
   /** List of native library directories. */
   private final File[] nativeLibraryDirectories;
   /**
    * Exceptions thrown during creation of the dexElements list.
    */
   private final IOException[] dexElementsSuppressedExceptions;
  /**
    * Constructs an instance.
    *
    * @param definingContext the context in which any as-yet unresolved
    * classes should be defined
    * @param dexPath list of dex/resource path elements, separated by
    * {@code File.pathSeparator}
    * @param libraryPath list of native library directory path elements,
    * separated by {@code File.pathSeparator}
    * @param optimizedDirectory directory where optimized {@code .dex} files
    * should be found and written to, or {@code null} to use the default
    * system directory for same
    */
   public DexPathList(ClassLoader definingContext, String dexPath,
           String libraryPath, File optimizedDirectory) {
       if (definingContext == null) {
           throw new NullPointerException("definingContext == null");
       }
      if (dexPath == null) {
           throw new NullPointerException("dexPath == null");
       }
      if (optimizedDirectory != null) {
           if (!optimizedDirectory.exists())  {
               throw new IllegalArgumentException(
                       "optimizedDirectory doesn't exist: "
                       + optimizedDirectory);
           }
          if (!(optimizedDirectory.canRead()
                           && optimizedDirectory.canWrite())) {
               throw new IllegalArgumentException(
                       "optimizedDirectory not readable/writable: "
                       + optimizedDirectory);
           }
       }
       this.definingContext = definingContext;
       ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
       this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                          suppressedExceptions);
       if (suppressedExceptions.size() > 0) {
           this.dexElementsSuppressedExceptions =
               suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
       } else {
           dexElementsSuppressedExceptions = null;
       }
       this.nativeLibraryDirectories = splitLibraryPath(libraryPath);
   }
   @Override public String toString() {
       return "DexPathList[" + Arrays.toString(dexElements) +
           ",nativeLibraryDirectories=" + Arrays.toString(nativeLibraryDirectories) + "]";
   }
   /**
    * For BaseDexClassLoader.getLdLibraryPath.
    */
   public File[] getNativeLibraryDirectories() {
       return nativeLibraryDirectories;
   }
  /**
    * Splits the given dex path string into elements using the path
    * separator, pruning out any elements that do not refer to existing
    * and readable files. (That is, directories are not included in the
    * result.)
    */
   private static ArrayList<File> splitDexPath(String path) {
       return splitPaths(path, null, false);
   }
   /**
    * Splits the given library directory path string into elements
    * using the path separator ({@code File.pathSeparator}, which
    * defaults to {@code ":"} on Android, appending on the elements
    * from the system library path, and pruning out any elements that
    * do not refer to existing and readable directories.
    */
   private static File[] splitLibraryPath(String path) {
       // Native libraries may exist in both the system and
       // application library paths, and we use this search order:
       //
       //   1. this class loader's library path for application libraries
       //   2. the VM's library path from the system property for system libraries
       //
       // This order was reversed prior to Gingerbread; see http://b/2933456.
       ArrayList<File> result = splitPaths(path, System.getProperty("java.library.path"), true);
       return result.toArray(new File[result.size()]);
   }
   /**
    * Splits the given path strings into file elements using the path
    * separator, combining the results and filtering out elements
    * that don't exist, aren't readable, or aren't either a regular
    * file or a directory (as specified). Either string may be empty
    * or {@code null}, in which case it is ignored. If both strings
    * are empty or {@code null}, or all elements get pruned out, then
    * this returns a zero-element list.
    */
   private static ArrayList<File> splitPaths(String path1, String path2,
           boolean wantDirectories) {
       ArrayList<File> result = new ArrayList<File>();
       splitAndAdd(path1, wantDirectories, result);
       splitAndAdd(path2, wantDirectories, result);
       return result;
   }
   /**
    * Helper for {@link #splitPaths}, which does the actual splitting
    * and filtering and adding to a result.
    */
   private static void splitAndAdd(String searchPath, boolean directoriesOnly,
           ArrayList<File> resultList) {
       if (searchPath == null) {
           return;
       }
       for (String path : searchPath.split(":")) {
           try {
               StructStat sb = Libcore.os.stat(path);
               if (!directoriesOnly || S_ISDIR(sb.st_mode)) {
                   resultList.add(new File(path));
               }
           } catch (ErrnoException ignored) {
           }
       }
   }
  /**
    * Makes an array of dex/resource path elements, one per element of
    * the given array.
    */
   private static Element[] makeDexElements(ArrayList<File> files, File optimizedDirectory,
                                            ArrayList<IOException> suppressedExceptions) {
       ArrayList<Element> elements = new ArrayList<Element>();
       /*
        * Open all files and load the (direct or contained) dex files
        * up front.
        */
       for (File file : files) {
           File zip = null;
           DexFile dex = null;
           String name = file.getName();
           if (name.endsWith(DEX_SUFFIX)) {
               // Raw dex file (not inside a zip/jar).
               try {
                   dex = loadDexFile(file, optimizedDirectory);
               } catch (IOException ex) {
                   System.logE("Unable to load dex file: " + file, ex);
               }
           } else if (name.endsWith(APK_SUFFIX) || name.endsWith(JAR_SUFFIX)
                   || name.endsWith(ZIP_SUFFIX)) {
               zip = file;
             try {
                   dex = loadDexFile(file, optimizedDirectory);
               } catch (IOException suppressed) {
                   /*
                    * IOException might get thrown "legitimately" by the DexFile constructor if the
                    * zip file turns out to be resource-only (that is, no classes.dex file in it).
                    * Let dex == null and hang on to the exception to add to the tea-leaves for
                    * when findClass returns null.
                    */
                   suppressedExceptions.add(suppressed);
               }
           } else if (file.isDirectory()) {
               // We support directories for looking up resources.
               // This is only useful for running libcore tests.
               elements.add(new Element(file, true, null, null));
           } else {
               System.logW("Unknown file type for: " + file);
           }
          if ((zip != null) || (dex != null)) {
               elements.add(new Element(file, false, zip, dex));
           }
       }
      return elements.toArray(new Element[elements.size()]);
   }
  /**
    * Constructs a {@code DexFile} instance, as appropriate depending
    * on whether {@code optimizedDirectory} is {@code null}.
    */
   private static DexFile loadDexFile(File file, File optimizedDirectory)
           throws IOException {
       if (optimizedDirectory == null) {
           return new DexFile(file);
       } else {
           String optimizedPath = optimizedPathFor(file, optimizedDirectory);
           return DexFile.loadDex(file.getPath(), optimizedPath, 0);
       }
   }
   /**
    * Converts a dex/jar file path and an output directory to an
    * output file path for an associated optimized dex file.
    */
   private static String optimizedPathFor(File path,
           File optimizedDirectory) {
       /*
        * Get the filename component of the path, and replace the
        * suffix with ".dex" if that's not already the suffix.
        *
        * We don't want to use ".odex", because the build system uses
        * that for files that are paired with resource-only jar
        * files. If the VM can assume that there's no classes.dex in
        * the matching jar, it doesn't need to open the jar to check
        * for updated dependencies, providing a slight performance
        * boost at startup. The use of ".dex" here matches the use on
        * files in /data/dalvik-cache.
        */
       String fileName = path.getName();
       if (!fileName.endsWith(DEX_SUFFIX)) {
           int lastDot = fileName.lastIndexOf(".");
           if (lastDot < 0) {
               fileName += DEX_SUFFIX;
           } else {
               StringBuilder sb = new StringBuilder(lastDot + 4);
               sb.append(fileName, 0, lastDot);
               sb.append(DEX_SUFFIX);
               fileName = sb.toString();
           }
       }
       File result = new File(optimizedDirectory, fileName);
       return result.getPath();
   }
   /**
    * Finds the named class in one of the dex files pointed at by
    * this instance. This will find the one in the earliest listed
    * path element. If the class is found but has not yet been
    * defined, then this method will define it in the defining
    * context that this instance was constructed with.
    *
    * @param name of class to find
    * @param suppressed exceptions encountered whilst finding the class
    * @return the named class or {@code null} if the class is not
    * found in any of the dex files
    */
   public Class findClass(String name, List<Throwable> suppressed) {
       for (Element element : dexElements) {
           DexFile dex = element.dexFile;
          if (dex != null) {
               Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
               if (clazz != null) {
                   return clazz;
               }
           }
       }
       if (dexElementsSuppressedExceptions != null) {
           suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
       }
       return null;
   }
   /**
    * Finds the named resource in one of the zip/jar files pointed at
    * by this instance. This will find the one in the earliest listed
    * path element.
    *
    * @return a URL to the named resource or {@code null} if the
    * resource is not found in any of the zip/jar files
    */
   public URL findResource(String name) {
       for (Element element : dexElements) {
           URL url = element.findResource(name);
           if (url != null) {
               return url;
           }
       }
      return null;
   }
  /**
    * Finds all the resources with the given name, returning an
    * enumeration of them. If there are no resources with the given
    * name, then this method returns an empty enumeration.
    */
   public Enumeration<URL> findResources(String name) {
       ArrayList<URL> result = new ArrayList<URL>();
       for (Element element : dexElements) {
           URL url = element.findResource(name);
           if (url != null) {
               result.add(url);
           }
       }
       return Collections.enumeration(result);
   }
   /**
    * Finds the named native code library on any of the library
    * directories pointed at by this instance. This will find the
    * one in the earliest listed directory, ignoring any that are not
    * readable regular files.
    *
    * @return the complete path to the library or {@code null} if no
    * library was found
    */
   public String findLibrary(String libraryName) {
       String fileName = System.mapLibraryName(libraryName);
       for (File directory : nativeLibraryDirectories) {
           String path = new File(directory, fileName).getPath();
           if (IoUtils.canOpenReadOnly(path)) {
               return path;
           }
       }
       return null;
   }
   /**
    * Element of the dex/resource file path
    */
   /*package*/ static class Element {
       private final File file;
       private final boolean isDirectory;
       private final File zip;
       private final DexFile dexFile;
       private ZipFile zipFile;
       private boolean initialized;
       public Element(File file, boolean isDirectory, File zip, DexFile dexFile) {
           this.file = file;
           this.isDirectory = isDirectory;
           this.zip = zip;
           this.dexFile = dexFile;
       }
        @Override public String toString() {
           if (isDirectory) {
               return "directory \"" + file + "\"";
           } else if (zip != null) {
               return "zip file \"" + zip + "\"";
           } else {
               return "dex file \"" + dexFile + "\"";
           }
       }
        public synchronized void maybeInit() {
           if (initialized) {
               return;
           }
            initialized = true;
            if (isDirectory || zip == null) {
               return;
           }
            try {
               zipFile = new ZipFile(zip);
           } catch (IOException ioe) {
               /*
                * Note: ZipException (a subclass of IOException)
                * might get thrown by the ZipFile constructor
                * (e.g. if the file isn't actually a zip/jar
                * file).
                */
               System.logE("Unable to open zip file: " + file, ioe);
               zipFile = null;
           }
       }
        public URL findResource(String name) {
           maybeInit();
            // We support directories so we can run tests and/or legacy code
           // that uses Class.getResource.
           if (isDirectory) {
               File resourceFile = new File(file, name);
               if (resourceFile.exists()) {
                   try {
                       return resourceFile.toURI().toURL();
                   } catch (MalformedURLException ex) {
                       throw new RuntimeException(ex);
                   }
               }
           }
            if (zipFile == null || zipFile.getEntry(name) == null) {
               /*
                * Either this element has no zip/jar file (first
                * clause), or the zip/jar file doesn't have an entry
                * for the given name (second clause).
                */
               return null;
           }
            try {
               /*
                * File.toURL() is compliant with RFC 1738 in
                * always creating absolute path names. If we
                * construct the URL by concatenating strings, we
                * might end up with illegal URLs for relative
                * names.
                */
               return new URL("jar:" + file.toURL() + "!/" + name);
           } catch (MalformedURLException ex) {
               throw new RuntimeException(ex);
           }
       }
   }
}
```

主要看这一句

```java
this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
```

BaseDexClassLoader的findClass方法，实际实现的是DexPathList的findClass方法

### DexPathList

DexPathList的构造方法

```java
public DexPathList(ClassLoader definingContext, String dexPath,
           String libraryPath, File optimizedDirectory) {//
       if (definingContext == null) {
           throw new NullPointerException("definingContext == null");
       }
       if (dexPath == null) {
           throw new NullPointerException("dexPath == null");
       }
       if (optimizedDirectory != null) {
           if (!optimizedDirectory.exists())  {
               throw new IllegalArgumentException(
                       "optimizedDirectory doesn't exist: "
                       + optimizedDirectory);
           }
           if (!(optimizedDirectory.canRead()
                           && optimizedDirectory.canWrite())) {
               throw new IllegalArgumentException(
                       "optimizedDirectory not readable/writable: "
                       + optimizedDirectory);
           }
       }
       this.definingContext = definingContext;
       ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
       this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                          suppressedExceptions);//解压APK/jar添加dex到dexElements数组中去
       if (suppressedExceptions.size() > 0) {
           this.dexElementsSuppressedExceptions =
               suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
       } else {
           dexElementsSuppressedExceptions = null;
       }
       this.nativeLibraryDirectories = splitLibraryPath(libraryPath);
   }
```

参数说明：
参数一：BaseDexClassLoader本身
参数二：apk/dex/jar路径
参数三：系统的一些文件库
参数四：dex文件路径（参数二apk或jar解压输出文件库路径）

通过makeDexelements(...)方法我们获取了一个 Element[] dexElements数组。保存了dex文件的相关信息



我们上面说到：DexClassLoader加载类实际调用的是DexPathList的findClass进行加载的。我们再来看下DexPathList的findClass方法

```java
public Class findClass(String name, List<Throwable> suppressed) {
       for (Element element : dexElements) {//这里的代码比较重要 有一种插件化的思想就是把修改过的dex进行插队 
           DexFile dex = element.dexFile;
           if (dex != null) {
               Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
               if (clazz != null) {
                   return clazz;
               }
           }
       }
       if (dexElementsSuppressedExceptions != null) {
           suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
       }
       return null;
   }
```

**重要思想：dex插队处理，比如A.dex中包含了一个a.class文件。我们通过插入一个B.dex在A.dex文件之前，**
**findClass方法在遍历DEXeLEMENTS文件会先在B.dex中查找，如果找到了a.class文件就会直接返回该a.class，如果没有找到才去B.dex中查找**

类在dex中的加载 是执行的DexFile类的loadClassBinaryName方法

```java
    /**
     * See {@link #loadClass(String, ClassLoader)}.
     *
     * This takes a "binary" class name to better match ClassLoader semantics.
     *
     * @hide
     */
    public Class loadClassBinaryName(String name, ClassLoader loader, List<Throwable> suppressed) {
        return defineClass(name, loader, mCookie, suppressed);
    }

    private static Class defineClass(String name, ClassLoader loader, int cookie,
                                     List<Throwable> suppressed) {
        Class result = null;
        try {
            result = defineClassNative(name, loader, cookie);
        } catch (NoClassDefFoundError e) {
            if (suppressed != null) {
                suppressed.add(e);
            }
        } catch (ClassNotFoundException e) {
            if (suppressed != null) {
                suppressed.add(e);
            }
        }
        return result;
    }

    private static native Class defineClassNative(String name, ClassLoader loader, int cookie)
        throws ClassNotFoundException, NoClassDefFoundError;
```

```java
loadClassBinaryName(String name, ClassLoader loader, List<Throwable> suppressed) 方法返回调用了
defineClass(String name, ClassLoader loader, int cookie,List<Throwable> suppressed) 而该方法调用了
private static native Class defineClassNative(String name, ClassLoader loader, int cookie)
        throws ClassNotFoundException, NoClassDefFoundError; 此方法为natvie方法
```

### 总结

```
第一步：执行ClassLoader的loadClass(String className)方法
第二步：执行BaseDexClassLoader的findClass(String classNmae)方法
第三步：执行DexPathList的findClass（String className）方法
第四步：执行DexFile的loadClassBinaryName方法
第五步：执行DexFile的defindClass方法
第六步：C或C++的defineClassNative方法
```











