---
layout:     post                    # 使用的布局（不需要改）
title:     Hook之Activity的启动拦截    # 标题 
subtitle:   应用内拦截以及系统全局拦截         #副标题
date:       2018-06-03               # 时间
author:     Jack                      # 作者
header-img: img/post-bg-fantasy03.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Hook
    - Activity
    
---


# Hook之Activity的启动拦截，应用内拦截以及系统全局拦截

---
为了更容易的理解，需要掌握JAVA的反射，动态代理技术，以及Activity的启动流程。

## 1、寻找Hook点的原则
1、Hook点，一般是分析源码来得到，而一般的Hook点都是静态变量或者是单例方法。

2、构造一个需要拦截的代理对象，需要的条件是代理的对象必须实现一个接口，其次就是需要获取到原始对象实例

## 2、寻找Hook点
通常点击一个Button就开始Activity跳转了，这中间发生了什么，我们如何Hook,来实现Activity启动的拦截呢？

```java
public void start(View view) {
    Intent intent = new Intent(this, OtherActivity.class);
    startActivity(intent);
}
```

源码

```java
@Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }

@Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }

@Override
    public void startActivityForResult(
            String who, Intent intent, int requestCode, @Nullable Bundle options) {
        Uri referrer = onProvideReferrer();
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, who,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, who, requestCode,
                ar.getResultCode(), ar.getResultData());
        }
        cancelInputsAndStartExitTransition(options);
    }
```


我们的目的是要拦截startActivity方法，跟踪源码，发现启动Activity是由Instrumentation类的execStartActivity做到的。其实这个类相当于启动Activity的中间者，启动Activity中间都是由它来操作的

```java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        ....
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            

    //通过ActivityManagerNative.getDefault()获取一个对象，开始启动新的Activity
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
                    
         //检查activity是否启动成功            
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

对于ActivityManagerNative这个东东，熟悉Activity/Service启动过程的都不陌生

`public abstract class ActivityManagerNative extends Binder implements IActivityManager`

继承了Binder，实现了一个IActivityManager接口，这就是为了远程服务通信做准备的"Stub"类，一个完整的AID L有两部分，一个是个跟服务端通信的Stub,一个是跟客户端通信的Proxy。ActivityManagerNative就是Stub，最终由ActivityManagerService启动的。这是android框架典型的跨进程通信。

```java
static public IActivityManager getDefault() {
    return gDefault.get();
}
```

ActivityManagerNative.getDefault()获取的是一个IActivityManager对象，由IActivityManager去启动Activity，IActivityManager的实现类是ActivityManagerService，ActivityManagerService是在另外一个进程之中，所有Activity 启动是一个跨进程的通信的过程，所以真正启动Activity的是通过远端服务ActivityManagerService来启动的。

```java
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        IBinder b = ServiceManager.getService("activity");
        if (false) {
            Log.v("ActivityManager", "default service binder = " + b);
        }
        IActivityManager am = asInterface(b);
        if (false) {
            Log.v("ActivityManager", "default service = " + am);
        }
        return am;
    }
```

其实gDefalut借助Singleton实现的单例模式，而在内部可以看到先从ServiceManager中获取到AMS远端服务的Binder对象，然后使用asInterface方法转化成本地化对象，我们目的是拦截startActivity,所以改变IActivityManager对象可以做到这个一点，这里gDefault又是静态的，根据Hook原则，这是一个比较好的Hook点。

## 3、Hook掉startActivity的两种方案

### 方案一：
我们先实现一个小需求，启动Activity的时候打印一条日志，写一个工具类HookUtil。

```java
public class HookUtil {

private Class<?> proxyActivity;

private Context context;

public HookUtil(Class<?> proxyActivity, Context context) {
    this.proxyActivity = proxyActivity;
    this.context = context;
}

public void hookAms() {
    
    //一路反射，直到拿到IActivityManager的对象
    try {
        Class<?> ActivityManagerNativeClss = Class.forName("android.app.ActivityManagerNative");
        Field defaultFiled = ActivityManagerNativeClss.getDeclaredField("gDefault");
        defaultFiled.setAccessible(true);
        Object defaultValue = defaultFiled.get(null);
        //反射SingleTon
        Class<?> SingletonClass = Class.forName("android.util.Singleton");
        Field mInstance = SingletonClass.getDeclaredField("mInstance");
        mInstance.setAccessible(true);
        //到这里已经拿到ActivityManager对象
        Object iActivityManagerObject = mInstance.get(defaultValue);
     
        //开始动态代理，用代理对象替换掉真实的ActivityManager，瞒天过海
        Class<?> IActivityManagerIntercept = Class.forName("android.app.IActivityManager");
    
        AmsInvocationHandler handler = new AmsInvocationHandler(iActivityManagerObject);

        Object proxy = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), new Class<?>[]{IActivityManagerIntercept}, handler);

        //现在替换掉这个对象
        mInstance.set(defaultValue, proxy);

    } catch (Exception e) {
        e.printStackTrace();
    }
}
```



```java
private class AmsInvocationHandler implements InvocationHandler {

    private Object iActivityManagerObject;

    private AmsInvocationHandler(Object iActivityManagerObject) {
        this.iActivityManagerObject = iActivityManagerObject;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        Log.i("HookUtil", method.getName());
        //我要在这里搞点事情
        if ("startActivity".contains(method.getName())) {
            Log.e("HookUtil","Activity已经开始启动");
            Log.e("HookUtil","小弟到此一游！！！");
        }
        return method.invoke(iActivityManagerObject, args);
    }
}

}

public class MyApplication extends Application {

@Override
public void onCreate() {
    super.onCreate();
    HookUtil hookUtil=new HookUtil(SecondActivity.class, this);
    hookUtil.hookAms()；
}

}
```



### 方案二：

我们可以去hook掉Activity类内变量 mInstrumentation 将它替换成我们自己的Instrumentation
通过手写静态代理类，替换掉原始的方法

```java
public class EvilInstrumentation extends Instrumentation {

private static final String TAG = "EvilInstrumentation";

// ActivityThread中原始的对象, 保存起来
Instrumentation mBase;

public EvilInstrumentation(Instrumentation base) {
    mBase = base;
}

public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {

        // Hook之前, XXX到此一游!
        Log.e(TAG, "\n执行了startActivity, 参数如下: \n" + "who = [" + who + "], " +
            "\ncontextThread = [" + contextThread + "], \ntoken = [" + token + "], " +
            "\ntarget = [" + target + "], \nintent = [" + intent +
            "], \nrequestCode = [" + requestCode + "], \noptions = [" + options + "]");

        Intent intent1 = new Intent(who,BrowserActivity.class);
        intent1.setData(intent.getData());
        // 开始调用原始的方法, 调不调用随你,但是不调用的话, 所有的startActivity都失效了.
        // 由于这个方法是隐藏的,因此需要使用反射调用;首先找到这个方法
        try {

            //这里通过反射找到原始Instrumentation类的execStartActivity方法
            Method execStartActivity = Instrumentation.class.getDeclaredMethod(
                    "execStartActivity",
                    Context.class, IBinder.class, IBinder.class, Activity.class,
                    Intent.class, int.class, Bundle.class);
            execStartActivity.setAccessible(true);

            //执行原始Instrumentation的execStartActivity方法 再次之前 you can do whatever you want ...
            if (intent.getAction().equals(Intent.ACTION_VIEW)) {
                return (ActivityResult) execStartActivity.invoke(mBase, who,
                        contextThread, token, target, intent1, requestCode, options);
            }else {
                return (ActivityResult) execStartActivity.invoke(mBase, who,
                        contextThread, token, target, intent, requestCode, options);
            }
        } catch (Exception e) {
            // 某该死的rom修改了  需要手动适配
            throw new RuntimeException("do not support!!! pls adapt it");
        }
    }

}


```

这里我们将判断intent.getAction().equals(Intent.ACTION_VIEW)，如果相等则不跳转系统浏览器而是跳转到我们自己写的BrowerActivity中取访问网络。

有了代理对象 下面使用反射直接进行替换

```java
public static void replaceInstrumentation(Activity activity){
            Class<?> k = Activity.class;
        try {
            //通过Activity.class 拿到 mInstrumentation字段
            Field field = k.getDeclaredField("mInstrumentation");
            field.setAccessible(true);
            //根据activity内mInstrumentation字段 获取Instrumentation对象
            Instrumentation instrumentation = (Instrumentation)field.get(activity);
            //创建代理对象
            Instrumentation instrumentationProxy = new EvilInstrumentation(instrumentation);
            //进行替换
            field.set(activity,instrumentationProxy);
        } catch (IllegalAccessException e){
            e.printStackTrace();
        }catch (NoSuchFieldException e){
            e.printStackTrace();
        }

}
```

在应用的onCreate()方法中调用replaceInstrumentation（this）即可.

## 4、无需注册，启动Activity
如下，TargetActivity没有在清单文件中注册，怎么去启动TargetActivity？

  

```java
 public void start(View view) {
        Intent intent = new Intent(this, TargetActivity.class);
        startActivity(intent);
    }
```

这个思路可以是这样，上面已经拦截了启动Activity流程，在invoke中我们可以得到启动参数intent信息，那么就在这里，我们可以自己构造一个假的Activity信息的intent，这个Intent启动的Activity是在清单文件中注册的，当真正启动的时候（ActivityManagerService校验清单文件之后），用真实的Intent把代理的Intent在调换过来，然后启动即可。

首先获取真实启动参数intent信息

```java
@Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if ("startActivity".contains(method.getName())) {
                //换掉
                Intent intent = null;
                int index = 0;
                for (int i = 0; i < args.length; i++) {
                    Object arg = args[i];
                    if (arg instanceof Intent) {
                        //说明找到了startActivity的Intent参数
                        intent = (Intent) args[i];
                        //这个意图是不能被启动的，因为Acitivity没有在清单文件中注册
                        index = i;
                    }
                }

           //伪造一个代理的Intent，代理Intent启动的是proxyActivity
            Intent proxyIntent = new Intent();
            ComponentName componentName = new ComponentName(context, proxyActivity);
            proxyIntent.setComponent(componentName);
            proxyIntent.putExtra("oldIntent", intent);
            args[index] = proxyIntent;
        }

        return method.invoke(iActivityManagerObject, args);
    }
```

有了上面的两个步骤,这个代理的Intent是可以通过ActivityManagerService检验的，因为我在清单文件中注册过

```java
  <activity android:name=".ProxyActivity" />
```

为了不启动ProxyActivity，现在我们需要找一个合适的时机，把真实的Intent换过了来，启动我们真正想启动的Activity。看过Activity的启动流程的朋友，我们都知道这个过程是由Handler发送消息来实现的，可是通过Handler处理消息的代码来看，消息的分发处理是有顺序的，下面是Handler处理消息的代码:

```java
public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```

handler处理消息的时候，首先去检查是否实现了callback接口，如果有实现的话，那么会直接执行接口方法，然后才是handleMessage方法，最后才是执行重写的handleMessage方法，我们一般大部分时候都是重写了handleMessage方法,而ActivityThread主线程用的正是重写的方法，这种方法的优先级是最低的，我们完全可以实现接口来替换掉系统Handler的处理过程。

```java
public void hookSystemHandler() {
        try {
            Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
            Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
            currentActivityThreadMethod.setAccessible(true);
            //获取主线程对象
            Object activityThread = currentActivityThreadMethod.invoke(null);
            //获取mH字段
            Field mH = activityThreadClass.getDeclaredField("mH");
            mH.setAccessible(true);
            //获取Handler
            Handler handler = (Handler) mH.get(activityThread);
            //获取原始的mCallBack字段
            Field mCallBack = Handler.class.getDeclaredField("mCallback");
            mCallBack.setAccessible(true);
            //这里设置了我们自己实现了接口的CallBack对象
            mCallBack.set(handler, new ActivityThreadHandlerCallback(handler)) ;

    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

自定义Callback类

```java
private class ActivityThreadHandlerCallback implements Handler.Callback {
        

    private Handler handler;

    private ActivityThreadHandlerCallback(Handler handler) {
        this.handler = handler;
    }

    @Override
    public boolean handleMessage(Message msg) {
        Log.i("HookAmsUtil", "handleMessage");
        //替换之前的Intent
        if (msg.what ==100) {
            Log.i("HookAmsUtil","lauchActivity");
            handleLauchActivity(msg);
        }

        handler.handleMessage(msg);
        return true;
    }

    private void handleLauchActivity(Message msg) {
        Object obj = msg.obj;//ActivityClientRecord
        try{
            Field intentField = obj.getClass().getDeclaredField("intent");
            intentField.setAccessible(true);
            Intent proxyInent = (Intent) intentField.get(obj);
            Intent realIntent = proxyInent.getParcelableExtra("oldIntent");
            if (realIntent != null) {
                proxyInent.setComponent(realIntent.getComponent());
            }
        }catch (Exception e){
            Log.i("HookAmsUtil","lauchActivity falied");
        }

    }
}
```

最后在application中注入

```java
public class MyApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        //这个ProxyActivity在清单文件中注册过，以后所有的Activitiy都可以用ProxyActivity无需声明，绕过监测
        HookAmsUtil hookAmsUtil = new HookAmsUtil(ProxyActivity.class, this);
        hookAmsUtil.hookSystemHandler();
        hookAmsUtil.hookAms();
    }
}
```




## 5、关于Android中的Hook技术其实可以分为两种：

1、第一种是获取root权限，利用进程注入技术，修改指定函数指针，达到拦截效果，这种方式可以拦截系统所有的服务。对系统所有应用有效果。（例如Xposed框架）

2、无需root权限，利用反射机制和动态代理技术，达到拦截效果，这种方式只能对本应用有效果。


## 6、系统全局拦截

这里我们使用Xposed框架进行演示，如果对Xposed框架陌生的请看我的这篇博客
[VirtualXposed插件开发](http://jackzhang.info/2018/04/09/VirtualXposed/)

代码如下:

```java
public class HookStartActivity implements IXposedHookLoadPackage {
    private Intent hookIntent;

@Override
public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) throws Throwable {
    //XposedBridge.log("load app: " + lpparam.packageName);//显示加载的 app 名称

    if (lpparam.packageName.equals("com.jack.sftest")) {
        XposedBridge.log("开始Hook测试程序");

        XposedHelpers.findAndHookMethod(Activity.class, "startActivity", Intent.class, Bundle.class, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                XposedBridge.log("beforeHookedMethod");

                String activityName = param.thisObject.getClass().getName();
                Log.d("Xposed", "Started activity: " + activityName);
                Intent intent = (Intent) param.args[0];
                Log.d("Xposed", "原始Intent数据: " + intent.toString());
                if (intent.getAction() == Intent.ACTION_VIEW) {
                    intent.setAction(Intent.ACTION_MAIN);
                    intent.setClassName("com.jack.sftest", "com.jack.sftest.BrowserActivity");
                    Log.d("Xposed", "替换后Intent数据: " + intent.toString());
                }
            }

            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                XposedBridge.log("afterHookedMethod");

            }
        });
    }
}

}
```

这里主要Hook包名为com.jack.sftest应用的Activity，当然你也
可以不加包名则系统所有Activity都将会Hook到
这里仍然是将intent.getAction() == Intent.ACTION_VIEW的Intent替换为目标应用的BrowserActivity

当然别忘了在xposed_init中修改Xposed 模块的入口文件为

     jack.com.hookdemo.HookStartActivity


## 7、技术用途

到这里我们就实现了Android中无需声明Activity就可以启动的效果，那么这个有什么用呢？

1、现在很多应用有时候会集成微信和支付宝支付功能，但是这时候就需要在AndroidManifest.xml中声明一些Activity，而恶心的是，有些市场在审核个人开发者提交的app的时候，如果有支付功能是不能审核通过的，这个应该也是为了防止恶意扣费吧，那么对于个人开发者就是没辙了？想想路子还是有的：

1》可以先上一个没有支付功能的，先到市场再说，然后在自己的app中自升级带有支付功能的即可，完全绕过市场审核了，但是这种方式是需要自升级工作。

2》采用插件化开发，把支付功能SDK做成动态加载，这样市场在扫描包的时候是找不到指定支付api就可以的，同时还得把AndroidManifest.xml中的支付Activity给隐藏起来躲避检测，那么如何隐藏就用到了这里的技术了，咋们可以自定义一个代理假的Activity，然后通过这种方式启动真正的支付Activity即可。

2、上面也提到了，在插件化开发中处理Activity的生命周期问题，也是可以采用这种方式去做处理的。





## 参考链接

[Android插件化系列第（一）篇---Hook技术之Activity的启动过程拦截](https://www.jianshu.com/p/69bfbda302df)

[Android系统篇之----Hook系统的AMS服务实现应用启动的拦截功能](https://blog.csdn.net/jiangwei0910410003/article/details/52550147)

[Android插件化之startActivity hook的几种姿势](https://www.jianshu.com/p/5c6ff86331c8)