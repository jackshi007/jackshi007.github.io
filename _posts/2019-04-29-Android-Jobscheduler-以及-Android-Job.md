---
layout:     post                    # 使用的布局（不需要改）
title:     Android Jobscheduler 以及 Android-Job    # 标题 
subtitle:  Jobscheluder的使用以及开源方案Android-Job  #副标题
date:       2019-04-29              # 时间
author:     Jack                      # 作者
header-img: img/post-bg-notre-dame.jpg  #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android
    - 保活
    - Jobscheduler



---

# Android Jobscheduler 以及 Android-Job

## 1.前言

> **Android 5.0系统以前**，在处理一些特定情况下的任务，或者是为了应用的保活，我们通常是**使用了Service**常驻后台来满足我们的需求。当达到某个条件时触发该Service来进行相应任务的处理。或者仅仅是为了我们自己的应用不被系统回收销毁。这样做在满足了自己应用的需求的同时也**消耗了部分硬件性能**。对用户的体验上，和Android系统环境上都有不利的影响。
>
> **Android 5.0系统以后**，Google为了优化Android系统，**提高使用流畅度以及延长电池续航**，加入了在应用后台/锁屏时，系统会回收应用，同时**自动销毁应用拉起的Service**的机制。同时为了满足在特定条件下需要执行某些任务的需求，google在全新一代操作系统上，采取了Job *（jobservice & JobInfo）*的方式，即每个需要后台的业务处理为一个job，通过系统管理job，来提高资源的利用率，从而提高性能，节省电源。这样又能满足APP开发商的要求，又能满足系统性能的要求。**Jobscheduler由此应运而生。**

## **2.Jobscheduler特性以及应用场景**

### 2.1 特性：

1、支持在一个任务上**组合多个条件**

2、**内置条件**：设备待机、设备充电和连接网络

3、**支持持续的job**，这意味着设备**重启后**，之前被**中断的job可以继续执行**

4、支持**设置job的最后执行期限**

5、根据你的配置，可以**设置job在后台运行还是在主线程中运行**

### 2.2 应用场景：

 1、应用具有可以推迟的**非面向用户的工作**（定期数据库数据更新）

 2、应用具有当**插入设备时希望优先执行的工作**（充电时才希望执行的工作备份数据)

 3、需要**访问网络**或 **Wi-Fi 连接**时需要进行的任务(如向服务器拉取内置数据)

 4、希望作为一个**批次定期**运行的许多**任务（s）**

### 2.3 特征：

 1、Job Scheduler**只有在Api21或以上的系统支持**。

 2、Job Scheduler是将**多个任务打包**在一个场景下执行。

 3、在系统重启以后，任务会依然保留在Job Scheduler当中，因此不需要监听系统启动状态重复设定。

 4、如果在一定期限内还没有满足特定执行所需情况，Job Scheduler会将这些任务加入队列，并且随后会进行执行。

使用Job Scheduler，应用需要做的事情就是判断哪些任务是不紧急的，可以交给Job Scheduler来处理，Job Scheduler集中处理收到的任务，选择合适的时间，合适的网络，再一起进行执行。**把时效性不强的工作丢给它做**。

## 3.Jobscheduler初识

通俗的来讲，Jobscheduler也是通过Jobservice服务和JobInfo搭配来完成我们特定情况下需要完成的各种工作的一个新组件。和传统的service相比较，（从上边也可以了解到）它具有更优秀的表现，不仅对系统环境的友好，同时在特定的预置条件下进行我们期望的工作也节省了手机的电量，节约了系统资源。下边先了解下Jobservice和JobInfo。之后在完成搭配使用的用例以及应用衍生。

> Jobscheduler的使用流程大概分为以下四个部分：
>
> - 派生JobService 子类，定义需要执行的任务（UI线程）

- 从Context 中获取JobScheduler 实例（相当于管理器）
- 构建JobInfo 实例，指定 JobService任务实现类及其执行条件
- 通过JobScheduler 实例加入到任务队列

#### · 1 自定义[Jobservice](https://link.jianshu.com?t=https://developer.android.com/reference/android/app/job/JobService.html)类

JobService是从Service中扩展出来的一个新类，继承Service。但是有两个重要的新接口。

下边使用用例代码来讲解：

```
public class JobSchedulerService extends JobService {

    @Override
    public boolean onStartJob(JobParameters params) {
        // 返回true，表示该工作耗时，同时工作处理完成后需要调用onStopJob销毁（jobFinished）
        // 返回false，任务运行不需要很长时间，到return时已完成任务处理
        return false;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        // 有且仅有onStartJob返回值为true时，才会调用onStopJob来销毁job
        // 返回false来销毁这个工作
        return false;
    }
}  
```

**1、OnStartJob(JobParameters params)**

 当开始一个任务时，onstartjob（jobparameters params） 是必须使用的方法，因为它是系统用来触发已经安排的工作（job）的。
  从上边的用例代码可以看到，该方法返回一个布尔值。不同的返回值对应了不同的处理方式。

 - **如果返回值是false，该系统假定任何任务运行不需要很长时间并且到方法返回时已经完成。**
 -  **如果返回值是true，那么系统假设任务是需要一些时间并且是需要在我们自己应用执行的。**
    当给定的任务完成时需要通过调用`jobFinished(JobParameters params, boolean needsRescheduled)`告知系统，该任务已经处理完成。
    **如果返回值为true，我们需要手动调用jobFinished来停止该任务** 

**2、JobFinished**

**void jobFinished (JobParameters params, boolean needsReschedule)**

*Callback to inform the JobManager you've finished executing. This can be called from any thread, as it will ultimately be run on your application's main thread. When the system receives this message it will release the wakelock being held.*
回调通知已完成执行的JobManager。这可以从任何线程调用，因为它最终将在应用程序的主线程上运行。当系统收到该消息时，它将释放正在保存的唤醒。

 - **JobParameters params**
    -- onStartJob(JobParameters).
    传入的param需要和onStartJob中的param一致
 - **boolean needsReschedule**
    -- True if this job should be rescheduled according to the back-off criteria specified at schedule-time. False otherwise.
    **让系统知道这个任务是否应该在最初的条件下被重复执行**（稍后会介绍这个布尔值的用处）

**3、OnStopJob(JobParameters params)**

当收到取消请求时，`onStopJob(JobParameters params)`是系统用来取消挂起的任务的。
**如果onStartJob(JobParameters params)返回 false**，它根本就**不调用onStopJob(JobParameters params)**。那此时就需要我们手动调用`jobFinished (JobParameters params, boolean needsReschedule)`方法了。

要注意的是，JobService需要在应用程序的主线程上运行。

简单的来讲，我们可以在上面JobSchedulerService类中创建一个Handler或者AsyncTask来处理需要进行的Job。

```
public class JobSchedulerService extends JobService {

    @Override
    public boolean onStartJob(JobParameters params) {
        // 返回true，表示该工作耗时，同时工作处理完成后需要调用jobFinished销毁
        mJobHandler.sendMessage(Message.obtain(mJobHandler, 1, params));
        return true;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        mJobHandler.removeMessages(1);
        return false;
    }
    
    // 创建一个handler来处理对应的job
    private Handler mJobHandler = new Handler(new Handler.Callback() {
        // 在Handler中，需要实现handleMessage(Message msg)方法来处理任务逻辑。
        @Override
        public boolean handleMessage(Message msg) {
            Toast.makeText(getApplicationContext(), "JobService task running", Toast.LENGTH_SHORT).show();
            // 调用jobFinished
            jobFinished((JobParameters) msg.obj, false);
            return true;
        }
    });
} 
```

当然一个异步任务也是可行的：

```
public class JobSchedulerService extends JobService {

    private JobParameters mJobParameters

    @Override
    public boolean onStartJob(JobParameters params) {
        // 返回true，表示该工作耗时，同时工作处理完成后需要调用jobFinished销毁
        mJobParameters = params;
        mTask.execute();
        return true;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        return false;
    }

    private AsyncTask<Void, Void, Void> mTask = new AsyncTask<Void, Void, Void>() {

        @Override
        protected Void doInBackground(Void... params) {
            // TODO Auto-generated method stub
            return null;
        }

        @Override
        protected void onPostExecute(Void result) {
            // TODO Auto-generated method stub
            Toast.makeText(wenfengService.this, "finish job", 1000).show();
            jobFinished(mJobParameters, true);
            super.onPostExecute(result);
        }
    } 
}
```

当任务完成时，需要调用`jobFinished(JobParameters params, boolean needsRescheduled)`让系统知道完成了哪项任务，它可以开始排队接下来的操作。**如果不这样做，工作将只运行一次，应用程序将不被允许执行额外的工作**。

代码片段中，因为handler中的操作可能比`onStartJob(JobParameters params)`方法它可能需要更长的时间来完成。通过设置true返回值，让程序了解将手动调用`jobFinished(JobParameters params, boolean needsRescheduled)`方法标记完成任务。

同时从上边的用例也可发现`jobFinished(JobParameters params, boolean needsRescheduled)`的**布尔值**是false，它让系统知道是否需要重复执行。同时这个布尔值是非常有用，可以**帮助我们解决如何处理由于其他问题（如一个失败的网络电话）而导致任务无法完成的情况**。设置为true我们就可以*重复*(参见下面的描述)

> 任务失败的情况有很多，例如下载失败了，例如下载过程wifi断掉了。
>  例如如果下载过程中，wifi断掉了，JobService会回调onStopJob函数，这时只需要把函数的返回值设置为true就可以了。当wifi重新连接后，JobService会重新回调onStartJob函数。
>  而如果下载失败了，例如上面的例子中的mJobHandler执行失败，怎么办呢？我们只需要在Handler的handleMessage中执行jobFinished(mJobParameters, true)，这里的true代表任务要在wifi条件重新满足情况下重新调度。

**4、 绑定服务：**

在简单的完成以上发送toast的Java代码之后，同Service一样，需要在AndroidManifest.xml中添加一个Service节点让应用拥有绑定和使用这个JobService的权限。

```
<service android:name="pkgName.JobSchedulerService"
    android:permission="android.permission.BIND_JOB_SERVICE" />
```

------

#### · 2 创建JobScheduler对象

在完成JobSchedulerService的构建以及绑定Service节点之后，接下来进行的是如何与JobScheduler API交互。

**1、创建一个JobScheduler**

在Activity中我们通过`getSystemService(Context.JOB_SCHEDULER_SERVICE)`实例化一个mJobScheduler的JobScheduler对象。

```java
JobScheduler mJobScheduler = (JobScheduler) getSystemService(Context.JOB_SCHEDULER_SERVICE);
```

创建定时任务时，可以使用**JobInfo.Builder**来构建一个**JobInfo**对象，然后传递给JobService。

```java
JobInfo.Builder builder = new JobInfo.Builder(jobId, 
new ComponentName(getPackageName(), JobSchedulerService.class.getName()));
```

**JobInfo.Builder**

> *JobInfo.Builder(int **jobId**, ComponentName **jobService**)*
>  *Initialize a new Builder to construct a JobInfo.*
>
> JobInfo.Builder接收两个参数
>
> -  **jobId** ： 要运行的任务的标识符
> -  **jobService** ： Service组件的类名。

下面简要的介绍部分**builder**中的方法：

- [addTriggerContentUri(JobInfo.TriggerContentUri uri)]：添加一个TriggerContentUri，该Uri将利用ContentObserver来监控一个Content Uri，当且仅当其发生变化时将触发任务的执行。为了持续监控content的变化，你需要在最近的任务触发后再调度一个新的任务（**需要注意的是触发器URI不能与setPeriodic(long)或setPersisted(boolean)组合使用。要持续监控内容更改，需要在完成JobService处理最近的更改之前，调度新的JobInfo，观察相同的URI。因为设置此属性与定期或持久化Job不兼容，这样做会在调用build()时抛出IllegalArgumentException异常。**)
- `setBackoffCriteria(long initialBackoffMillis, int backoffPolicy)`：**特殊：**设置回退/重试的策略，详细的可以参阅Google API。
   类似网络原理中的冲突退避，当一个任务的调度失败时需要重试，所采取的策略。第一个参数时第一次尝试重试的等待间隔，单位为毫秒，预设的参数有：`DEFAULT_INITIAL_BACKOFF_MILLIS 30000` 、`MAX_BACKOFF_DELAY_MILLIS 18000000`。第二个参数是对应的退避策略，预设的参数有：`BACKOFF_POLICY_EXPONENTIAL` 二进制退避。等待间隔呈指数增长 `BACKOFF_POLICY_LINEAR`。
- `setExtras(PersistableBundle extras)`：设置可选附件。这是持久的，所以只允许原始类型。
- `setMinimumLatency(long minLatencyMillis)`： 设置任务的最小延迟时间(毫秒），相当于post delay。
- `setOverrideDeadline(long maxExecutionDelayMillis)`： 设置任务最大的延迟时间 。**
- `setPeriodic(long time)`：设置任务运行的周期（每X毫秒，运行一次）。
- `setPeriodic(long intervalMillis, long flexMillis)`：设置在Job周期末的一个flex长度的窗口，任务都有可能被执行 **require API LEVEL 24**
- **setPersisted(boolean isPersisted): 设备重启之后任务是否继续执行**。
- `setRequiredNetworkType(int networkType)`： 只在满足指定的网络条件时才会被执行。**默认条件是JobInfo.NETWORK_TYPE_NONE**，这意味着不管是否有网络这个任务都会被执行。另外三个可选类型，**JobInfo.NETWORK_TYPE_ANY**，它表明需要任意一种网络才使得任务可以执行。**JobInfo.NETWORK_TYPE_UNMETERED**，设备不是蜂窝网络( 比如在WIFI连接时 )时任务才会被执行，**JobInfo.NETWORK_TYPE_NOT_ROAMING**,它表示非漫游网络状态。
- `setRequiresCharging(boolean requiresCharging)`: *是否在充电时执行*。这个也并非只是插入充电器，而且还要在电池处于健康状态的情况下才会触发，一般来说是手机电量>15%
- `setRequiresDeviceIdle(boolean requiresDeviceIdle)`：*是否在空闲时执行*
- `setTransientExtras(Bundle extras)`：设置可选的临时附加功能。
- `setTriggerContentMaxDelay(long durationMs)`：设置从第一次检测到内容更改到Job之前允许的最大总延迟（以毫秒为单位）。说人话就是**设置从content变化到任务被执行，中间的最大延迟**。  *同样的**require API 24***
- `setTriggerContentUpdateDelay(long durationMs)`：设置从content变化到任务被执行中间的延迟。如果在延迟期间content发生了变化，延迟会重新计算
- `setExtras(PersistableBundle extra)`：Bundle

**要注意的是： 1、***设置延迟时间* `setMinimumLatency(long minLatencyMillis)`**和***设置最终期限时间* `setOverrideDeadline(long maxExecutionDelayMillis)`**的两个方法不能同时与setPeriodic(long time)同时设置，也就是说，在设置延迟和最终期限时间时是不能设置重复周期时间的。还有在具体开发过程中需要注意各个方法的API兼容情况。**

2、`setRequiredNetworkType(int networkType)`, `setRequiresCharging(boolean requireCharging)` 和 `setRequiresDeviceIdle(boolean requireIdle)`这几个方法可能会使得任务无法执行，除非调用`setOverrideDeadline(long time)`设置了最大延迟时间，使得任务在为满足条件的情况下也会被执行。

构建一个JobInfo对象设置预置的条件，然后通过如下所示的代码将它发送到的JobScheduler中。

开启一个JobScheduler任务：

```
mJobScheduler.schedule(JobInfo job)
```

在schedule时，**会返回一个int类型的值来标记这次任务是否执行成功**，如果返回小于0的错误码，这表示该次任务执行失败，反之则成功(**成功会返回该任务的id，这里可以使用这个id来判断哪些任务成功了**)。所以在返回值小于0的时候就需要我们手动去处理一些事情了。

```
if(mJobScheduler.schedule(JobInfo job) < 0){
    // do something when schedule goes wrong
}
```

最后如果需要**停止一个任务**，就通过JobScheduler中，`cancel(int jobId)`来实现(所以之前在Builer中的指定id又有了重要作用)；如果想**取消所有的任务**，可以调用JobScheduler对象的`cancelAll()`来实现。

### 四、Jobscheduler使用

在上边初略的讲解了job scheduler的一些使用方法，下边通过一个用例来加深一下理解。

**创建我们Job依附的Activity:**

```java
public class SchedulerAcitvity extends Activity {  

    private static final String TAG = "SchedulerAcitvity";  

    public static final String MESSENGER_INTENT_KEY = TAG + ".MESSENGER_INTENT_KEY";
    public static final String WORK_DURATION_KEY = TAG + ".WORK_DURATION_KEY";
    public static final int MSG_JOB_START = 0;
    public static final int MSG_JOB_STOP = 1;
    public static final int MSG_ONJOB_START = 2;
    public static final int MSG_ONJOB_STOP = 3;

    private int mJobId = 0;// 执行的JobId

    ComponentName mServieComponent;// 这就是我们的jobservice组件了
    private IncomingMessageHandler mHandler;// 用于来自服务的传入消息的处理程序。
    
    // UI
    private EditText mEt_Delay;// 设置delay时间
    private EditText mEt_Deadline;// 设置最长的截止时间
    private EditText mEt_DurationTime;// setPeriodic周期
    private RadioButton mRb_WiFiConnectivity;// 设置builder中的是否有WiFi连接
    private RadioButton mRb_AnyConnectivity;// 设置builder中的是否有网络即可
    private CheckBox mCb_RequiresCharging;// 设置builder中的是否需要充电
    private CheckBox mCb_RequiresIdle;// 设置builder中的是否设备空闲
    private Button mBtn_StartJob;// 点击开始任务的按钮
    private Button mBtn_StopAllJob;// 点击结束所有任务的按钮
    
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.durian_main);

        mHandler = new IncomingMessageHandler(this);
        mServieComponent = new ComponentName(this, MyJobService.class);// 获取到我们自己的jobservice，同时启动该service

        // 设置UI
        mEt_Delay = (EditText) findViewById(R.id.delay_time);
        mEt_DurationTime = (EditText) findViewById(R.id.duration_time);
        mEt_Deadline = (EditText) findViewById(R.id.deadline_time);
        mRb_WiFiConnectivity = (RadioButton) findViewById(R.id.checkbox_unmetered);
        mRb_AnyConnectivity = (RadioButton) findViewById(R.id.checkbox_any);
        mCb_RequiresCharging = (CheckBox) findViewById(R.id.checkbox_charging);
        mCb_RequiresIdle = (CheckBox) findViewById(R.id.checkbox_idle);
        mBtn_StartJob = (Button)findViewById(R.id.button_start_job);
        mBtn_StopAllJob = (Button)findViewById(R.id.button_start_job);

        mBtn_StartJob.setOnClickListener(new View.OnClickListener() {  
            @Override  
            public void onClick(View v) {  
                scheduleJob();
                }  
        }); 

        mBtn_StopAllJob.setOnClickListener(new View.OnClickListener() {  
            @Override  
            public void onClick(View v) {  
                cancelAllJobs();
                }  
        });
    }
    
    @Override
    protected void onStart() {
        super.onStart();
        // 启动服务并提供一种与此类通信的方法。
        Intent startServiceIntent = new Intent(this, MyJobService.class);
        Messenger messengerIncoming = new Messenger(mHandler);
        startServiceIntent.putExtra(MESSENGER_INTENT_KEY, messengerIncoming);
        startService(startServiceIntent);
    }
    
    @Override
    protected void onStop() {
        // 服务可以是“开始”和/或“绑定”。 在这种情况下，它由此Activity“启动”
        // 和“绑定”到JobScheduler（也被JobScheduler称为“Scheduled”）。
        // 对stopService（）的调用不会阻止处理预定作业。
        // 然而，调用stopService（）失败将使它一直存活。
        stopService(new Intent(this, MyJobService.class));
        super.onStop();
    }
    
    // 当用户单击SCHEDULE JOB时执行。
    public void scheduleJob() {
        //开始配置JobInfo
        JobInfo.Builder builder = new JobInfo.Builder(mJobId++, mServiceComponent);

        //设置任务的延迟执行时间(单位是毫秒)
        String delay = mEt_Delay.getText().toString();
        if (!TextUtils.isEmpty(delay)) {
            builder.setMinimumLatency(Long.valueOf(delay) * 1000);
        }
        //设置任务最晚的延迟时间。如果到了规定的时间时其他条件还未满足，你的任务也会被启动。
        String deadline = mEt_Deadline.getText().toString();
        if (!TextUtils.isEmpty(deadline)) {
            builder.setOverrideDeadline(Long.valueOf(deadline) * 1000);
        }
        boolean requiresUnmetered = mRb_WiFiConnectivity.isChecked();
        boolean requiresAnyConnectivity = mRb_AnyConnectivity.isChecked();

        //让你这个任务只有在满足指定的网络条件时才会被执行
        if (requiresUnmetered) {
            builder.setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED);
        } else if (requiresAnyConnectivity) {
            builder.setRequiredNetworkType(JobInfo.NETWORK_TYPE_ANY);
        }

        //你的任务只有当用户没有在使用该设备且有一段时间没有使用时才会启动该任务。
        builder.setRequiresDeviceIdle(mCb_RequiresIdle.isChecked());
        //告诉你的应用，只有当设备在充电时这个任务才会被执行。
        builder.setRequiresCharging(mCb_RequiresCharging.isChecked());

        // Extras, work duration.
        PersistableBundle extras = new PersistableBundle();
        String workDuration = mEt_DurationTime.getText().toString();
        if (TextUtils.isEmpty(workDuration)) {
            workDuration = "1";
        }
        extras.putLong(WORK_DURATION_KEY, Long.valueOf(workDuration) * 1000);

        builder.setExtras(extras);

        // Schedule job
        Log.d(TAG, "Scheduling job");
        JobScheduler mJobScheduler = (JobScheduler) getSystemService(Context.JOB_SCHEDULER_SERVICE);
        // 这里就将开始在service里边处理我们配置好的job
        mJobScheduler.schedule(builder.build());

        //mJobScheduler.schedule(builder.build())会返回一个int类型的数据
        //如果schedule方法失败了，它会返回一个小于0的错误码。否则它会返回我们在JobInfo.Builder中定义的标识id。
    }

    // 当用户点击取消所有时执行
    public void cancelAllJobs() {
        JobScheduler mJobScheduler = (JobScheduler) getSystemService(Context.JOB_SCHEDULER_SERVICE);
        mJobScheduler.cancelAll();
        Toast.makeText(MainActivity.this, R.string.all_jobs_cancelled, Toast.LENGTH_SHORT).show();
    }

    /**
    * {@link Handler}允许您发送与线程相关联的消息。
    * {@link Messenger}使用此处理程序从{@link MyJobService}进行通信。
    * 它也用于使开始和停止视图在短时间内闪烁。
    */
    private static class IncomingMessageHandler extends Handler {

        // 使用弱引用防止内存泄露
        private WeakReference<SchedulerAcitvity> mActivity;

        IncomingMessageHandler(SchedulerAcitvity activity) {
            super(/* default looper */);
            this.mActivity = new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            SchedulerAcitvity mSchedulerAcitvity = mActivity.get();
            if (mSchedulerAcitvity == null) {
                // 活动不再可用，退出。
                return;
            }
            
            // 获取到两个View，用于之后根据Job运行状态显示不同的运行状态（颜色变化）
            View showStartView = mSchedulerAcitvity.findViewById(R.id.onstart_textview);
            View showStopView = mSchedulerAcitvity.findViewById(R.id.onstop_textview);

            Message m;
            switch (msg.what) {
                 // 当作业登录到应用程序时，从服务接收回调。 打开指示灯（上方View闪烁）并发送一条消息，在一秒钟后将其关闭。
                 case MSG_JOB_START:
                    // Start received, turn on the indicator and show text.
                    // 开始接收，打开指示灯（上方View闪烁）并显示文字。
                    showStartView.setBackgroundColor(getColor(R.color.start_received));
                    updateParamsTextView(msg.obj, "started");

                    // Send message to turn it off after a second.
                    // 发送消息，一秒钟后关闭它。
                    m = Message.obtain(this, MSG_ONJOB_START);
                    sendMessageDelayed(m, 1000L);
                    break;

                // 当先前执行在应用程序中的作业必须停止执行时，
                // 从服务接收回调。 打开指示灯并发送一条消息，
                // 在两秒钟后将其关闭。
                case MSG_JOB_STOP:
                    // Stop received, turn on the indicator and show text.
                    // 停止接收，打开指示灯并显示文本。
                    showStopView.setBackgroundColor(getColor(R.color.stop_received));
                    updateParamsTextView(msg.obj, "stopped");

                    // Send message to turn it off after a second.
                    // 发送消息，一秒钟后关闭它。
                    m = obtainMessage(MSG_ONJOB_STOP);
                    sendMessageDelayed(m, 2000L);
                    break;
                case MSG_ONJOB_START:
                    showStartView.setBackgroundColor(getColor(R.color.none_received));
                    updateParamsTextView(null, "job had started");
                    break;
                case MSG_ONJOB_STOP:
                    showStopView.setBackgroundColor(getColor(R.color.none_received));
                    updateParamsTextView(null, "job had stoped");
                    break;
            }
        } 

        // 更新UI显示
        // @param jobId jobId
        // @param action 消息
        private void updateParamsTextView(@Nullable Object jobId, String action) {
            TextView paramsTextView = (TextView) mActivity.get().findViewById(R.id.task_params);
            if (jobId == null) {
                paramsTextView.setText("");
                return;
            }
            String jobIdText = String.valueOf(jobId);
            paramsTextView.setText(String.format("Job ID %s %s", jobIdText, action));
        }

        private int getColor(@ColorRes int color) {
            return mActivity.get().getResources().getColor(color);
        }
    }
}  
```

在activity中我们使用一个按钮来开启JobScheduler，同时简单的配置了相应Builder的配置。在点击button的时候，此项scheduler就开始schedule(我们配置的jobInfo)。那当我们的条件匹配我们配置的JobInfo的时候会开始怎样的处理呢？这里就需要我们在service里边做作业了。

**满足配置条件时候启动任务的Service：**

```java
public class MyJobService extends JobService {

    private static final String TAG = MyJobService.class.getSimpleName();

    private Messenger mActivityMessenger;

    @Override
    public void onCreate() {
        super.onCreate();
        Log.i(TAG, "Service created");
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.i(TAG, "Service destroyed");
    }

    // 当应用程序的MainActivity被创建时，它启动这个服务。
    // 这是为了使活动和此服务可以来回通信。 请参见“setUiCallback（）”
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        mActivityMessenger = intent.getParcelableExtra(MESSENGER_INTENT_KEY);
        return START_NOT_STICKY;
    }

    @Override
    public boolean onStartJob(final JobParameters params) {
        // The work that this service "does" is simply wait for a certain duration and finish
        // the job (on another thread).

        // 该服务做的工作只是等待一定的持续时间并完成作业（在另一个线程上）。
        sendMessage(MSG_JOB_START, params.getJobId());
        // 当然这里可以处理其他的一些任务
        // TODO something else
        
        // 获取在activity里边设置的每个任务的周期，其实可以使用setPeriodic()
        long duration = params.getExtras().getLong(WORK_DURATION_KEY);

        // 使用一个handler处理程序来延迟jobFinished（）的执行。
        Handler handler = new Handler();
        handler.postDelayed(new Runnable() {
            @Override
            public void run() {
                sendMessage(MSG_JOB_STOP, params.getJobId());
                jobFinished(params, false);
            }
        }, duration);
        Log.i(TAG, "on start job: " + params.getJobId());

        // 返回true，很多工作都会执行这个地方，我们手动结束这个任务
        return true;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        // 停止跟踪这些作业参数，因为我们已经完成工作。
        sendMessage(MSG_JOB_STOP, params.getJobId());
        Log.i(TAG, "on stop job: " + params.getJobId());

        // 返回false来销毁这个工作
        return false;
    }

    private void sendMessage(int messageID, @Nullable Object params) {
        // 如果此服务由JobScheduler启动，则没有回调Messenger。
        // 它仅在MainActivity在Intent中使用回调函数调用startService()时存在。
        if (mActivityMessenger == null) {
            Log.d(TAG, "Service is bound, not started. There's no callback to send a message to.");
            return;
        }

        Message m = Message.obtain();
        m.what = messageID;
        m.obj = params;
        try {
            mActivityMessenger.send(m);
        } catch (RemoteException e) {
            Log.e(TAG, "Error passing service object back to activity.");
        }
    }
}
```

虽然在service里边只是简单地进行了一个我们设置的耗时操作，但是通过以上的例子应该很容易理解JobScheduler的使用了。在某些条件下（充电，网络连接【可以指定特定的状态】，设备空闲）JobScheduler可以更优秀的完成我们的触发型任务。

### 五、应用：

似乎都很热衷于应用的保活，很多地方都是将jobScheduler应用于Service杀不死，进一步拉应用的状态
 [使用JobScheduler进行开机自启动](https://www.jianshu.com/p/e0a06d5abf98)
 [Android服务保活-JobScheduler拉活](https://link.jianshu.com?t=http://www.aoaoyi.com/archives/456.html)
 [Android进程保活的一般套路](https://www.jianshu.com/p/1da4541b70ad)

JobScheduler省电：
 [Android L 的 JobScheduler API 是怎么让设备省电的](https://link.jianshu.com?t=https://www.zhihu.com/question/24360587)

除了JobScheduler ,还有其他一些类似的APIs去帮助安排你的工作计划，它们包括：
 [AlarmManager](https://link.jianshu.com?t=https://developer.android.com/reference/android/app/AlarmManager.html)
 [Firebase JobDispatcher](https://link.jianshu.com?t=https://github.com/firebase/firebase-jobdispatcher-android#user-content-firebase-jobdispatcher-)
 [GCM NETwork Manager](https://link.jianshu.com?t=https://developers.google.com/cloud-messaging/network-manager)
 [SyncAdapter](https://link.jianshu.com?t=https://developer.android.com/reference/android/content/AbstractThreadedSyncAdapter.html)
 [Additional Facilities](https://link.jianshu.com?t=https://developer.android.com/reference/android/content/AbstractThreadedSyncAdapter.html)



## 参考链接

[Android Jobscheduler使用](https://www.jianshu.com/p/9fb882cae239)

