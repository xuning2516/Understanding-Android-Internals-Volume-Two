<h1>第6章 深入理解ActivityManagerService</h1>
<h2>本章主要内容：</h2>
<p>·  详细分析ActivityManagerService</p>
<h2>本章所涉及的源代码文件名及位置：</h2>
<p>·  SystemServer.java</p>
<p>frameworks/base/services/java/com/android/server/SystemServer.java</p>
<p>·  ActivityManagerService.java</p>
<p>frameworks/base/services/java/com/android/server/am/ActivityManagerService.java</p>
<p>·  ContextImpl.java</p>
<p>frameworks/base/core/java/android/app/ContextImpl.java</p>
<p>·  ActivityThread.java</p>
<p>frameworks/base/core/java/android/app/ActivityThread.java</p>
<p>·  ActivityStack.java</p>
<p>frameworks/base/services/java/com/android/server/am/ActivityStack.java</p>
<p>·  Am.java</p>
<p>frameworks/base/cmds/am/src/com/android/commands/am/Am.java</p>
<p>·  ProcessRecord.java</p>
<p>frameworks/base/services/java/com/android/server/am/ProcessRecord.java</p>
<p>·  ProcessList.java</p>
<p>frameworks/base/services/java/com/android/server/am/ProcessList.java</p>
<p>·  RuntimeInit.java</p>
<p>frameworks/base/core/java/com/android/internal/os/RuntimeInit.java</p>
<h2>6.1  概述</h2>
<p>相信绝大部分读者对本书提到的ActivityManagerService（以后简称AMS）都有所耳闻。AMS是Android中最核心的服务，主要负责系统中四大组件的启动、切换、调度及应用进程的管理和调度等工作，其职责与操作系统中的进程管理和调度模块相类似，因此它在Android中非常重要。</p>
<p>AMS是本书碰到的第一块难啃的骨头<a>[①]</a>，涉及的知识点较多。为了帮助读者更好地理解AMS，本章将带领读者按五条不同的线来分析它。</p>
<p>·  第一条线：同其他服务一样，将分析SystemServer中AMS的调用轨迹。</p>
<p>·  第二条线：以am命令启动一个Activity为例，分析应用进程的创建、Activity的启动，以及它们和AMS之间的交互等知识。</p>
<p>·  第三条线和第四条线：分别以Broadcast和Service为例，分析AMS中Broadcast和Service的相关处理流程。</p>
<p>·  第五条线：以一个Crash的应用进程为出发点，分析AMS如何打理该应用进程的身后事。</p>
<p>除了这五条线外，还将统一分析在这五条线中频繁出现的与AMS中应用进程的调度、内存管理等相关的知识。</p>
<div>
    <p><strong>提示</strong>ContentProvider将放到下一章分析，不过本章将涉及和ContentProvider有关的知识点。</p>
</div>
<p>先来看AMS的家族图谱，如图6-1所示。</p>

![image](images/chapter6/image001.png)

<p>图6-1  AMS家族图谱</p>
<p>由图6-1可知：</p>
<p>·  AMS由ActivityManagerNative（以后简称AMN）类派生，并实现Watchdog.Monitor和BatteryStatsImpl.BatteryCallback接口。而AMN由Binder派生，实现了IActivityManager接口。</p>
<p>·  客户端使用ActivityManager类。由于AMS是系统核心服务，很多API不能开放供客户端使用，所以设计者没有让ActivityManager直接加入AMS家族。在ActivityManager类内部通过调用AMN的getDefault函数得到一个ActivityManagerProxy对象，通过它可与AMS通信。</p>
<p>AMS的简单介绍就到此为止，下面分析AMS。相信不少读者已经磨拳擦掌，跃跃欲试了。</p>
<div>
    <p><strong>提示</strong>读者们最好在桌上放一杯清茶，以保持AMS分析旅途中头脑清醒。</p>
</div>
<h2>6.2  初识ActivityManagerService</h2>
<p>AMS由SystemServer的ServerThread线程创建，提取它的调用轨迹，代码如下：</p>
<p>[--&gt;SystemServer.java::ServerThread的run函数]</p>
<div>
    <p>//①调用main函数，得到一个Context对象</p>
    <p>context =ActivityManagerService.main(factoryTest);</p>
    <p> </p>
    <p>//②setSystemProcess：这样SystemServer进程可加到AMS中，并被它管理</p>
    <p>ActivityManagerService.setSystemProcess();</p>
    <p> </p>
    <p>//③installSystemProviders：将SettingsProvider放到SystemServer进程中来运行</p>
    <p>ActivityManagerService.installSystemProviders();</p>
    <p> </p>
    <p>//④在内部保存WindowManagerService（以后简称WMS）</p>
    <p>ActivityManagerService.self().setWindowManager(wm);</p>
    <p> </p>
    <p>//⑤和WMS交互，弹出“启动进度“对话框</p>
    <p>ActivityManagerNative.getDefault().showBootMessage(</p>
    <p>             context.getResources().getText(</p>
    <p>               //该字符串中文为：“正在启动应用程序”</p>
    <p>               com.android.internal.R.string.android_upgrading_starting_apps),</p>
    <p>              false);</p>
    <p> </p>
    <p>//⑥AMS是系统的核心，只有它准备好后，才能调用其他服务的systemReady</p>
    <p>//注意，有少量服务在AMS systemReady之前就绪，它们不影响此处的分析</p>
    <p>ActivityManagerService.self().systemReady(newRunnable() {</p>
    <p>    publicvoid run() {</p>
    <p>   startSystemUi(contextF);//启动systemUi。如此，状态栏就准备好了</p>
    <p>    if(batteryF != null) batteryF.systemReady();</p>
    <p>    if(networkManagementF != null) networkManagementF.systemReady();</p>
    <p>    ......</p>
    <p>    Watchdog.getInstance().start();//启动Watchdog</p>
    <p>    ......//调用其他服务的systemReady函数</p>
    <p>}</p>
</div>
<p>在以上代码中，一共列出了6个重要调用及这些调用的简单说明，本节将分析除与WindowManagerService（以后简称WMS）交互的4、5外的其余四项调用。</p>
<p>先来分析1处调用。</p>
<h3>6.2.1  ActivityManagerService main分析</h3>
<p>AMS的main函数将返回一个Context类型的对象，该对象在SystemServer中被其他服务大量使用。Context，顾名思义，代表了一种上下文环境（笔者觉得其意义和JNIEnv类似），有了这个环境，我们就可以做很多事情（例如获取该环境中的资源、Java类信息等）。那么AMS的main将返回一个怎样的上下文环境呢？来看以下代码：</p>
<p>[--&gt;ActivityManagerService.java::main]</p>
<div>
    <p> publicstatic final Context main(int factoryTest) {</p>
    <p>    AThreadthr = new AThread();//①创建一个AThread线程对象</p>
    <p>    thr.start();</p>
    <p>    ......//等待thr创建成功</p>
    <p>    ActivityManagerServicem = thr.mService;</p>
    <p>    mSelf =m;</p>
    <p>    //②调用ActivityThread的systemMain函数</p>
    <p>    ActivityThreadat = ActivityThread.systemMain();</p>
    <p>    mSystemThread= at;</p>
    <p> </p>
    <p>    //③得到一个Context对象，注意调用的函数名为getSystemContext，何为System Context</p>
    <p>    Contextcontext = at.getSystemContext();</p>
    <p>    context.setTheme(android.R.style.Theme_Holo);</p>
    <p>    m.mContext= context;</p>
    <p>    m.mFactoryTest= factoryTest;</p>
    <p> </p>
    <p>    //ActivtyStack是AMS中用来管理Activity的启动和调度的核心类，以后再分析它</p>
    <p>    m.mMainStack = new ActivityStack(m, context,true);</p>
    <p>    //调用BSS的publish函数，我们在第5章的BSS知识中介绍过了</p>
    <p>    m.mBatteryStatsService.publish(context);</p>
    <p>    //另外一个service：UsageStatsService。后续再分析该服务</p>
    <p>    m.mUsageStatsService.publish(context);</p>
    <p>    synchronized (thr) {</p>
    <p>           thr.mReady = true;</p>
    <p>           thr.notifyAll();//通知thr线程，本线程工作完成</p>
    <p>     }</p>
    <p> </p>
    <p>    //④调用AMS的startRunning函数</p>
    <p>    m.startRunning(null, null, null, null);</p>
    <p>        </p>
    <p>   returncontext;</p>
    <p>}</p>
</div>
<p>在main函数中，我们又列出了4个关键函数，分别是：</p>
<p>·  创建AThread线程。虽然AMS的main函数由ServerThread线程调用，但是AMS自己的工作并没有放在ServerThread中去做，而是新创建了一个线程，即AThread线程。</p>
<p>·  ActivityThread.systemMain函数。初始化ActivityThread对象。</p>
<p>·  ActivityThread.getSystemContext函数。用于获取一个Context对象，从函数名上看，该Context代表了System的上下文环境。</p>
<p>·  AMS的startRunning函数。</p>
<p>注意，main函数中有一处等待（wait）及一处通知（notifyAll），原因是：</p>
<p>·  main函数首先需要等待AThread所在线程启动并完成一部分工作。</p>
<p>·  AThread完成那一部分工作后，将等待main函数完成后续的工作。</p>
<p>这种双线程互相等待的情况，在Android代码中比较少见，读者只需了解它们的目的即可。下边来分析以上代码中的第一个关键点。</p>
<h4>1.  AThread分析</h4>
<h5>（1） AThread分析</h5>
<p>AThread的代码如下：</p>
<p>[--&gt;ActivityManagerService.java::AThread]</p>
<div>
    <p>static class AThread extends Thread {//AThread从Thread类派生</p>
    <p>   ActivityManagerServicemService;</p>
    <p>   booleanmReady = false;</p>
    <p>   publicAThread() {</p>
    <p>     super("ActivityManager");//线程名就叫“ActivityManager”</p>
    <p>   }</p>
    <p>   publicvoid run() {</p>
    <p>     Looper.prepare();//看来，AThread线程将支持消息循环及处理功能</p>
    <p>     android.os.Process.setThreadPriority(//设置线程优先级</p>
    <p>                   android.os.Process.THREAD_PRIORITY_FOREGROUND);</p>
    <p>     android.os.Process.setCanSelfBackground(false);</p>
    <p>      //创建AMS对象</p>
    <p>     ActivityManagerService m = new ActivityManagerService();</p>
    <p>     synchronized (this) {</p>
    <p>           mService= m;//赋值AThread内部成员变量mService，指向AMS</p>
    <p>          notifyAll();  //通知main函数所在线程</p>
    <p>      }</p>
    <p>     synchronized (this) {</p>
    <p>        while (!mReady) {</p>
    <p>           try{</p>
    <p>                 wait();//等待main函数所在线程的notifyAll</p>
    <p>               }......</p>
    <p>           }</p>
    <p>       }......</p>
    <p>    Looper.loop();//进入消息循环</p>
    <p> }</p>
    <p> }</p>
</div>
<p>从本质上说，AThread是一个支持消息循环及处理的线程，其主要工作就是创建AMS对象，然后通知AMS的main函数。这样看来，main函数等待的就是这个AMS对象。</p>
<h5>（2） AMS的构造函数分析</h5>
<p>AMS的构造函数的代码如下：</p>
<p>[--&gt;ActivityManagerService.java::ActivityManagerService构造]</p>
<div>
    <p>private ActivityManagerService() {</p>
    <p>    FiledataDir = Environment.getDataDirectory();//指向/data/目录</p>
    <p>    FilesystemDir = new File(dataDir, "system");//指向/data/system/目录</p>
    <p>   systemDir.mkdirs();//创建/data/system/目录</p>
    <p> </p>
    <p>    //创建BatteryStatsService（以后简称BSS）和UsageStatsService（以后简称USS）</p>
    <p>   //我们在分析PowerManageService时已经见过BSS了</p>
    <p>   mBatteryStatsService = new BatteryStatsService(new File(</p>
    <p>               systemDir, "batterystats.bin").toString());</p>
    <p>   mBatteryStatsService.getActiveStatistics().readLocked();</p>
    <p>    mBatteryStatsService.getActiveStatistics().writeAsyncLocked();</p>
    <p>   mOnBattery = DEBUG_POWER ? true</p>
    <p>               : mBatteryStatsService.getActiveStatistics().getIsOnBattery();</p>
    <p>   mBatteryStatsService.getActiveStatistics().setCallback(this);</p>
    <p> </p>
    <p>    //创建USS</p>
    <p>    mUsageStatsService= new UsageStatsService(new File(</p>
    <p>               systemDir, "usagestats").toString());</p>
    <p>    //获取OpenGl版本</p>
    <p>   GL_ES_VERSION = SystemProperties.getInt("ro.opengles.version",</p>
    <p>           ConfigurationInfo.GL_ES_VERSION_UNDEFINED);</p>
    <p>     //mConfiguration类型为Configuration，用于描述资源文件的配置属性，例如</p>
    <p>     //字体、语言等。后文再讨论这方面的内容</p>
    <p>    mConfiguration.setToDefaults();</p>
    <p>    mConfiguration.locale = Locale.getDefault();</p>
    <p>     //mProcessStats为ProcessStats类型，用于统计CPU、内存等信息。其内部工作原理就是</p>
    <p>    //读取并解析/proc/stat文件的内容。该文件由内核生成，用于记录kernel及system</p>
    <p>    //一些运行时的统计信息。读者可在Linux系统上通过man proc命令查询详细信息</p>
    <p>    mProcessStats.init();</p>
    <p> </p>
    <p>     //解析/data/system/packages-compat.xml文件，该文件用于存储那些需要考虑屏幕尺寸</p>
    <p>    //的APK的一些信息。读者可参考AndroidManifest.xml中compatible-screens相关说明。</p>
    <p>    //当APK所运行的设备不满足要求时，AMS会根据设置的参数以采用屏幕兼容的方式去运行它</p>
    <p>    mCompatModePackages = new CompatModePackages(this, systemDir);</p>
    <p> </p>
    <p>     Watchdog.getInstance().addMonitor(this);</p>
    <p>     //创建一个新线程，用于定时更新系统信息（和mProcessStats交互）</p>
    <p>    mProcessStatsThread = new Thread("ProcessStats") {...//先略去该段代码}</p>
    <p>    mProcessStatsThread.start();</p>
    <p> }</p>
</div>
<p>AMS的构造函数比想象得要简单些，下面回顾一下它的工作：</p>
<p>·  创建BSS、USS、mProcessStats （ProcessStats类型）、mProcessStatsThread线程，这些都与系统运行状况统计相关。</p>
<p>·  创建/data/system目录，为mCompatModePackages（CompatModePackages类型）和mConfiguration（Configuration类型）等成员变量赋值。</p>
<p>AMS main函数的第一个关键点就分析到此，再来分析它的第二个关键点。</p>
<h4>2.  ActivityThread.systemMain函数分析</h4>
<p>ActivityThread是Android Framework中一个非常重要的类，它代表一个应用进程的主线程（对于应用进程来说，ActivityThread的main函数确实是由该进程的主线程执行），其职责就是调度及执行在该线程中运行的四大组件。</p>
<div>
    <p><strong>注意</strong>应用进程指那些运行APK的进程，它们由Zyote 派生（fork）而来，上面运行了dalvik虚拟机。与应用进程相对的就是系统进程（包括Zygote和SystemServer）。</p>
    <p>另外，读者须将“应用进程和系统进程”与“应用APK和系统APK”的概念区分开来。APK的判别依赖其文件所在位置（如果apk文件在/data/app目录下，则为应用APK）。</p>
</div>
<p>该函数代码如下：</p>
<p>[--&gt;ActivityThread.java::systemMain]</p>
<div>
    <p>public static final ActivityThread systemMain() {</p>
    <p>   HardwareRenderer.disable(true);//禁止硬件渲染加速</p>
    <p>   //创建一个ActivityThread对象，其构造函数非常简单</p>
    <p>  ActivityThread thread = new ActivityThread();</p>
    <p>  thread.attach(true);//调用它的attach函数，注意传递的参数为true</p>
    <p>   returnthread;</p>
    <p> }</p>
</div>
<p>在分析ActivityThread的attach函数之前，先提一个问题供读者思考：前面所说的ActivityThread代表应用进程（其上运行了APK）的主线程，而SystemServer并非一个应用进程，那么为什么此处也需要ActivityThread呢？</p>
<p>·  还记得在PackageManagerService分析中提到的framework-res.apk吗？这个APK除了包含资源文件外，还包含一些Activity（如关机对话框），这些Activity实际上运行在SystemServer进程中<a>[②]</a>。从这个角度看，SystemServer是一个特殊的应用进程。</p>
<p>·  另外，通过ActivityThread可以把Android系统提供的组件之间的交互机制和交互接口（如利用Context提供的API）也拓展到SystemServer中使用。</p>
<div>
    <p><strong>提示</strong>解答这个问题，对于理解SystemServer中各服务的交互方式是尤其重要的。</p>
</div>
<p>下面来看ActivityThread的attach函数。</p>
<h5>（1） attach函数分析</h5>
<p>[--&gt;ActivityThread.java::attach]</p>
<div>
    <p>private void attach(boolean system) {</p>
    <p>    sThreadLocal.set(this);</p>
    <p>    mSystemThread= system;//判断是否为系统进程</p>
    <p>    if(!system) {</p>
    <p>        ......//应用进程的处理流程</p>
    <p>     } else {//系统进程的处理流程，该情况只在SystemServer中处理</p>
    <p>       //设置DDMS时看到的systemserver进程名为system_process</p>
    <p>       android.ddm.DdmHandleAppName.setAppName("system_process");</p>
    <p>       try {</p>
    <p>            //ActivityThread的几员大将出场，见后文的分析</p>
    <p>            mInstrumentation = new Instrumentation();</p>
    <p>            ContextImpl context = new ContextImpl();</p>
    <p>            //初始化context，注意第一个参数值为getSystemContext</p>
    <p>            context.init(getSystemContext().mPackageInfo, null, this);</p>
    <p>            Application app = //利用Instrumentation创建一个Application对象</p>
    <p>                    Instrumentation.newApplication(Application.class,context);</p>
    <p>             //一个进程支持多个Application，mAllApplications用于保存该进程中</p>
    <p>            //的Application对象</p>
    <p>            mAllApplications.add(app);</p>
    <p>             mInitialApplication = app;//设置mInitialApplication</p>
    <p>            app.onCreate();//调用Application的onCreate函数</p>
    <p>           }......//try/catch结束</p>
    <p>      }//if(!system)判断结束</p>
    <p> </p>
    <p>     //注册Configuration变化的回调通知</p>
    <p>     ViewRootImpl.addConfigCallback(newComponentCallbacks2() {</p>
    <p>          publicvoid onConfigurationChanged(Configuration newConfig) {</p>
    <p>            ......//当系统配置发生变化（如语言切换等）时，需要调用该回调</p>
    <p>          }</p>
    <p>           public void onLowMemory() {}</p>
    <p>           public void onTrimMemory(int level) {}</p>
    <p>        });</p>
    <p> }</p>
</div>
<p>attach函数中出现了几个重要成员，其类型分别是Instrumentation类、Application类及Context类，它们的作用如下（为了保证准确，这里先引用Android的官方说明）。</p>
<p>·  Instrumentation：Base class for implementingapplication instrumentation code. When running with instrumentation turned on,this class will be instantiated for you before any of the application code,allowing you to monitor all of the interaction the system has with the application.An Instrumentation implementation is described to the system through anAndroidManifest.xml's &lt;instrumentation&gt; tag.大意是：Instrumentaion是一个工具类。当它被启用时，系统先创建它，再通过它来创建其他组件。另外，系统和组件之间的交互也将通过Instrumentation来传递，这样，Instrumentation就能监测系统和这些组件的交互情况了。在实际使用中，我们可以创建Instrumentation的派生类来进行相应的处理。读者可查询Android中Junit的使用来了解Intrstrumentation的作用。本书不讨论Intrstrumentation方面的内容。</p>
<p>·  Application：Base class for those who need tomaintain global application state. You can provide your own implementation byspecifying its name in your AndroidManifest.xml's &lt;application&gt; tag,which will cause that class to be instantiated for you when the process foryour application/package is created.大意是：Application类保存了一个全局的application状态。Application由AndroidManifest.xml中的&lt;application&gt;标签声明。在实际使用时需定义Application的派生类。</p>
<p>·  Context：Interface to global informationabout an application environment. This is an abstract class whoseimplementation is provided by the Android system. It allows access toapplication-specific resources and classes, as well as up-calls forapplication-level operations such as launching activities, broadcasting andreceiving intents, etc.大意是：Context是一个接口，通过它可以获取并操作Application对应的资源、类，甚至包含于Application中的四大组件。</p>
<div>
    <p><strong>提示</strong>此处的Application是Android中的一个概念，可理解为一种容器，它内部包含四大组件。另外，一个进程可以运行多个Application。</p>
</div>
<p>Context是一个抽象类，而由AMS创建的将是它的子类ContextImpl。如前所述，Context提供了Application的上下文信息，这些信息是如何传递给Context的呢？此问题包括两个方面：</p>
<p>·  Context本身是什么？</p>
<p>·  Context背后所包含的上下文信息又是什么？</p>
<p>下面来关注上边代码中调用的getSystemContext函数。</p>
<h5>（2） getSystemContext函数分析</h5>
<p>[--&gt;ActivityThread.java::getSystemContext]</p>
<div>
    <p>public ContextImpl getSystemContext() {</p>
    <p>  synchronized(this) {</p>
    <p>   if(mSystemContext == null) {//单例模式</p>
    <p>       ContextImplcontext =  ContextImpl.createSystemContext(this);</p>
    <p>       //LoadedApk是2.3引入的一个新类，代表一个加载到系统中的APK</p>
    <p>       LoadedApkinfo = new LoadedApk(this, "android", context, null,</p>
    <p>                       CompatibilityInfo.DEFAULT_COMPATIBILITY_INFO);</p>
    <p>       //初始化该ContextImpl对象 </p>
    <p>      context.init(info, null, this);</p>
    <p>      //初始化资源信息</p>
    <p>      context.getResources().updateConfiguration(</p>
    <p>                        getConfiguration(),getDisplayMetricsLocked(</p>
    <p>                       CompatibilityInfo.DEFAULT_COMPATIBILITY_INFO, false));</p>
    <p>       mSystemContext = context;//保存这个特殊的ContextImpl对象</p>
    <p>      }</p>
    <p>   }</p>
    <p>    returnmSystemContext;</p>
    <p>}</p>
</div>
<p>以上代码无非是先创建一个ContextImpl，然后再将其初始化（调用init函数）。为什么函数名是getSystemContext呢？因为在初始化ContextImp时使用了一个LoadedApk对象。如注释中所说，LoadedApk是Android 2.3引入的一个类，该类用于保存一些和APK相关的信息（如资源文件位置、JNI库位置等）。在getSystemContext函数中初始化ContextImpl的这个LoadedApk所代表的package名为“android”，其实就是framework-res.apk，由于该APK仅供SystemServer使用，所以此处叫getSystemContext。</p>
<p>上面这些类的关系比较复杂，可通过图6-2展示它们之间的关系。</p>

![image](images/chapter6/image002.png)

<p>图6-2  ContextImpl和它的“兄弟”们</p>
<p>由图6-2可知：</p>
<p>·  先来看派生关系， ApplicationContentResolver从ConentResolver派生，它主要用于和ContentProvider打交道。ContextImpl和ContextWrapper均从Context继承，而Application则从ContextWrapper派生。</p>
<p>·  从社会关系角度看，ContextImpl交际面最广。它通过mResources指向Resources，mPackageInfo指向LoadedApk，mMainThread指向ActivityThread，mContentResolver指向ApplicationContentResolver。</p>
<p>·  ActivityThread代表主线程，它通过mInstrumentation指向Instrumentation。另外，它还保存多个Application对象。</p>
<div>
    <p><strong>注意</strong>在函数中有些成员变量的类型为基类类型，而在图6-2中直接指向了实际类型。</p>
</div>
<h5>（3） systemMain函数总结</h5>
<p>调用systemMain函数结束后，我们得到了什么？</p>
<p>·  得到一个ActivityThread对象，它代表应用进程的主线程。</p>
<p>·  得到一个Context对象，它背后所指向的Application环境与framework-res.apk有关。</p>
<p>费了如此大的功夫，systemMain函数的目的到底是什么？一针见血地说：</p>
<p><strong>systemMain函数将为SystemServer进程搭建一个和应用进程一样的Android运行环境。</strong>这句话涉及两个概念。</p>
<p>·  进程：来源于操作系统，是在OS中看到的运行体。我们编写的代码一定要运行在一个进程中。</p>
<p>·  Android运行环境：Android努力构筑了一个自己的运行环境。在这个环境中，进程的概念被模糊化了。组件的运行及它们之间的交互均在该环境中实现。</p>
<p>Android运行环境是构建在进程之上的。有Android开发经验的读者可能会发现，在应用程序中，一般只和Android运行环境交互。基于同样的道理，SystemServer希望它内部的那些Service也通过Android运行环境交互，因此也需为它创建一个运行环境。由于SystemServer的特殊性，此处调用了systemMain函数，而普通的应用进程将在主线程中调用ActivityThread的main函数来创建Android运行环境。</p>
<p>另外，ActivityThread虽然本意是代表进程的主线程，但是作为一个Java类，它的实例到底由什么线程创建，恐怕不是ActivityThread自己能做主的，所以在SystemServer中可以发现，ActivityThread对象由其他线程创建，而在应用进程中，ActivityThread将由主线程来创建。</p>
<h4>3. ActivityThread.getSystemContext函数分析</h4>
<p>该函数在上一节已经见过了。调用该函数后，将得到一个代表系统进程的Context对象。到底什么是Context？先来看如图6-3所示的Context家族图谱。</p>
<div>
    <p><strong>注意</strong>该族谱成员并不完全。另外，Activity、Service和Application所实现的接口也未画出。</p>
</div>

![image](images/chapter6/image003.png)

<p>图6-3  Context家族图谱</p>
<p>由图6-3可知：</p>
<p>·  ContextWrapper比较有意思，其在SDK中的说明为“Proxying implementation ofContext that simply delegates all of its calls to another Context. Can besubclassed to modify behavior without changing the original Context.”大概意思是：ContextWrapper是一个代理类，被代理的对象是另外一个Context。在图6-3中，被代理的类其实是ContextImpl，由ContextWrapper通过mBase成员变量指定。读者可查看ContextWrapper.java，其内部函数功能的实现最终都由mBase完成。这样设计的目的是想把ContextImpl隐藏起来。</p>
<p>·  Application从ContextWrapper派生，并实现了ComponentCallbacks2接口。Application中有一个LoadedApk类型的成员变量mLoadedApk。LoadedApk代表一个APK文件。由于一个AndroidManifest.xml文件只能声明一个Application标签，所以一个Application必然会和一个LoadedApk绑定。</p>
<p>·  Service从ContextWrapper派生，其中Service内部成员变量mApplication指向Application（在AndroidManifest.xml中，Service只能作为Application的子标签，所以在代码中Service必然会和一个Application绑定）。</p>
<p>·  ContextThemeWrapper重载了和Theme（主题）相关的两个函数。这些和界面有关，所以Activity作为Android系统中的UI容器，必然也会从ContextThemeWrapper派生。与Service一样，Activity内部也通过mApplication成员变量指向Application。</p>
<p>对Context的分析先到这里，再来分析第三个关键函数startRunning。</p>
<h4>4.  AMS的startRunning函数分析</h4>
<p>[--&gt;ActivityManagerService.java::startRunning]</p>
<div>
    <p>//注意调用该函数时所传递的4个参数全为null</p>
    <p>public final void startRunning(String pkg, Stringcls, String action,</p>
    <p>                                    String data) {</p>
    <p>  synchronized(this) {</p>
    <p>     if (mStartRunning) return;  //如果已经调用过该函数，则直接返回</p>
    <p> </p>
    <p>    mStartRunning = true;</p>
    <p>     //mTopComponent最终赋值为null</p>
    <p>    mTopComponent = pkg != null &amp;&amp; cls != null</p>
    <p>                   ? new ComponentName(pkg, cls) : null;</p>
    <p>    mTopAction = action != null ? action : Intent.ACTION_MAIN;</p>
    <p>    mTopData = data; //mTopData最终为null</p>
    <p>     if(!mSystemReady) return; //此时mSystemReady为false，所以直接返回</p>
    <p>    }</p>
    <p>   systemReady(null);//这个函数很重要，可惜不在本次startRunning中调用</p>
    <p>}</p>
</div>
<p>startRunning函数很简单，此处不赘述。</p>
<p>至此，ASM 的main函数所涉及的4个知识点已全部分析完。下面回顾一下AMS 的main函数的工作。</p>
<h4>5.  ActivityManagerService的main函数总结</h4>
<p>AMS的main函数的目的有两个：</p>
<p>·  首先也是最容易想到的目的是创建AMS对象。</p>
<p>·  另外一个目的比较隐晦，但是非常重要，那就是创建一个供SystemServer进程使用的Android运行环境。</p>
<p>根据目前所分析的代码，Android运行环境将包括两个成员：ActivityThread和ContextImpl（一般用它的基类Context）。</p>
<p>图6-4展示了在这两个类中定义的一些成员变量，通过它们可看出ActivityThread及ContextImpl的作用。</p>

![image](images/chapter6/image004.png)

<p>图6-4  ActivityThread和ContextImpl的部分成员变量</p>
<p>由图6-4可知：</p>
<p>·  ActivityThread中有一个mLooper成员，它代表一个消息循环。这恐怕是ActivityThread被称做“Thread”的一个直接证据。另外，mServices用于保存Service，Activities用于保存ActivityClientRecord，mAllApplications用于保存Application。关于这些变量的具体作用，以后遇到时再说。</p>
<p>·  对于ContextImpl，其成员变量表明它和资源、APK文件有关。</p>
<p>AMS的main函数先分析到此，至于其创建的Android运行环境将在下节分析中派上用场。</p>
<p>接下来分析AMS的第三个调用函数setSystemProcess。</p>
<h3>6.2.2  AMS的setSystemProcess分析</h3>
<p>AMS的setSystemProcess的代码如下：</p>
<p>[--&gt;ActivityManagerService.java::setSystemProcess]</p>
<div>
    <p>public static void setSystemProcess() {</p>
    <p>  try {</p>
    <p>      ActivityManagerService m = mSelf;</p>
    <p>       //向ServiceManager注册几个服务</p>
    <p>       ServiceManager.addService("activity",m);</p>
    <p>       //用于打印内存信息</p>
    <p>      ServiceManager.addService("meminfo", new MemBinder(m));</p>
    <p> </p>
    <p>       /*</p>
    <p>        Android4.0新增的，用于打印应用进程使用硬件显示加速方面的信息（Applications</p>
    <p>         Graphics Acceleration Info）。读者通过adb shell dumpsys gfxinfo看看具体的</p>
    <p>         输出</p>
    <p>       */</p>
    <p>      ServiceManager.addService("gfxinfo", new GraphicsBinder(m));</p>
    <p> </p>
    <p>        if(MONITOR_CPU_USAGE)//该值默认为true，添加cpuinfo服务</p>
    <p>            ServiceManager.addService("cpuinfo", new CpuBinder(m));</p>
    <p> </p>
    <p>        //向SM注册权限管理服务PermissionController</p>
    <p>       ServiceManager.addService("permission", newPermissionController(m));</p>
    <p> </p>
    <p>      /*    <strong>重要说明：</strong></p>
    <p>        向PackageManagerService查询package名为"android"的ApplicationInfo。</p>
    <p>        注意这句调用：虽然PKMS和AMS同属一个进程，但是二者交互仍然借助Context。</p>
    <p>        其实，此处完全可以直接调用PKMS的函数。为什么要费如此周折呢</p>
    <p>      */</p>
    <p>      ApplicationInfo info = //使用AMS的mContext对象</p>
    <p>               mSelf.mContext.getPackageManager().getApplicationInfo(</p>
    <p>                        "android",STOCK_PM_FLAGS);</p>
    <p> </p>
    <p>          //①调用ActivityThread的installSystemApplicationInfo函数</p>
    <p>            mSystemThread.installSystemApplicationInfo(info);</p>
    <p>           synchronized (mSelf) {</p>
    <p>               //②此处涉及AMS对进程的管理，见下文分析</p>
    <p>               ProcessRecord app = mSelf.newProcessRecordLocked(</p>
    <p>                       mSystemThread.getApplicationThread(), info,</p>
    <p>                        info.processName);//注意，最后一个参数为字符串，值为“system”</p>
    <p>               app.persistent = true;</p>
    <p>               app.pid = MY_PID;</p>
    <p>               app.maxAdj = ProcessList.SYSTEM_ADJ;</p>
    <p>               //③保存该ProcessRecord对象</p>
    <p>               mSelf.mProcessNames.put(app.processName, app.info.uid, app);</p>
    <p>               synchronized (mSelf.mPidsSelfLocked) {</p>
    <p>                   mSelf.mPidsSelfLocked.put(app.pid, app);</p>
    <p>               }</p>
    <p>               //根据系统当前状态，调整进程的调度优先级和OOM_Adj，后续将详细分析该函数</p>
    <p>                mSelf.updateLruProcessLocked(app, true,true);</p>
    <p>           }</p>
    <p>        } ......//抛异常</p>
    <p>    }</p>
</div>
<p>在以上代码中列出了一个重要说明和两个关键点。</p>
<p>·  重要说明：AMS向PKMS查询名为“android”的ApplicationInfo。此处AMS和PKMS的交互是通过Context来完成的，查看这一系列函数调用的代码，最终发现AMS将通过Binder发送请求给PKMS来完成查询功能。AMS和PKMS同属一个进程，它们完全可以不通过Context来交互。此处为何要如此大费周章呢？原因很简单，Android希望SystemServer中的服务也通过Android运行环境来交互。这更多是从设计上来考虑的，比如组件之间交互接口的统一及未来系统的可扩展性。</p>
<p>·  关键点一：ActivityThread的installSystemApplicationInfo函数。</p>
<p>·  关键点二：ProcessRecord类，它和AMS对进程的管理有关。</p>
<p>通过重要说明，相信读者能真正理解AMS的 main函数中第二个隐含目的的作用，故此处不再展开叙述。</p>
<p>现在来看第一个关键点，即ActivityThread的installSystemApplicationInfo函数。</p>
<h4>1.  ActivityThread的installSystemApplicationInfo函数分析</h4>
<p>installSystemApplicationInfo函数的参数为一个ApplicationInfo对象，该对象由AMS通过Context查询PKMS中一个名为“android”的package得来（根据前面介绍的知识，目前只有framework-res.apk声明其package名为“android”）。</p>
<p>再来看installSystemApplicationInfo的代码，如下所示：</p>
<p>[--&gt;ActivityThread.java::installSystemApplicationInfo]</p>
<div>
    <p>public voidinstallSystemApplicationInfo(ApplicationInfo info) {</p>
    <p> synchronized (this) {</p>
    <p>   //返回的ContextImpl对象即之前在AMS的main函数一节中创建的那个对象</p>
    <p>   ContextImpl context = getSystemContext();</p>
    <p>    //又调用init初始化该Context，是不是重复调用init了？</p>
    <p>   context.init(new LoadedApk(this, "android", context, info,</p>
    <p>              CompatibilityInfo.DEFAULT_COMPATIBILITY_INFO), null, this);</p>
    <p>     //创建一个Profiler对象，用于性能统计</p>
    <p>     mProfiler = new Profiler();</p>
    <p>     }</p>
    <p> }</p>
</div>
<p>在以上代码中看到调用context.init的地方，读者可能会有疑惑，getSystemContext函数将返回mSystemContext，而此mSystemContext在AMS的main函数中已经初始化过了，此处为何再次初始化呢？</p>
<p>仔细查看看代码便会发现：</p>
<p>·  第一次执行init时，在LoadedApk构造函数中第四个表示ApplicationInfo的参数为null。</p>
<p>·  第二次执行init时，LoadedApk构造函数的第四个参数不为空，即该参数将真正指向一个实际的ApplicationInfo，该ApplicationInfo来源于framework-res.apk。</p>
<p>基于上面的信息，某些读者可能马上能想到：Context第一次执行init的目的仅仅是为了创建一个Android运行环境，而该Context并没有和实际的ApplicationInfo绑定。而第二次执行init前，先利用Context和PKMS交互得到一个实际的ApplicationInfo，然后再通过init将此Context和ApplicationInfo绑定。</p>
<p>是否觉得前面的疑惑已豁然而解？且慢，此处又抛出了一个更难的问题：</p>
<p>第一次执行init后得到的Context虽然没有绑定ApplicationInfo，不是也能使用吗？此处为何非要和一个ApplicationInfo绑定？</p>
<p>答案很简单，因为framework-res.apk（包括后面将介绍的SettingsProvider.apk）运行在SystemServer中。和其他所有apk一样，它的运行需要一个正确初始化的Android运行环境。</p>
<p>长嘘一口气，这个大难题终于弄明白了！在此即基础上，AMS下一步的工作就就顺理成章了。</p>
<p>由于framework-res.apk是一个APK文件，和其他APK文件一样，它应该运行在一个进程中。而AMS是专门用于进程管理和调度的，所以运行APK的进程应该在AMS中有对应的管理结构。因此AMS下一步工作就是将这个运行环境和一个进程管理结构对应起来并交由AMS统一管理。</p>
<p>AMS中的进程管理结构是ProcessRecord。</p>
<h4>2.  关于ProcessRecord和IApplicationThread的介绍</h4>
<p>分析ProcessRecord之前，先来思考一个问题：</p>
<p>AMS如何与应用进程交互？例如AMS启动一个位于其他进程的Activity，由于该Activity运行在另外一进程中，因此AMS势必要和该进程进行跨进程通信。</p>
<p>答案自然是通过Binder进行通信。为此，Android提供了一个IApplicationThread接口，该接口定义了AMS和应用进程之间的交互函数，如图6-5所示为该接口的家族图谱。</p>

![image](images/chapter6/image005.png)

<p>图6-5  ApplicationThread类</p>
<p>由图6-5可知：</p>
<p>·  ApplicationThreadNative实现了IApplicationThread接口。从该接口定义的函数可知，AMS通过它可以和应用进程进行交互，例如，AMS启动一个Activity的时候会调用该接口的scheduleLaunchActivity函数。</p>
<p>·  ActivityThread通过成员变量mAppThread指向它的内部类ApplicationThread，而ApplicationThread从ApplicationThreadNative派生。</p>
<p>基于以上知识，读者能快速得出下面问题的答案吗？</p>
<p>IApplicationThread的Binder服务端在应用进程中还是在AMS中？</p>
<div>
    <p><strong>提示</strong>如果读者知道Binder系统支持客户端监听服务端的死亡消息，那么这个问题的答案就简单了：服务端自然在应用进程中，因为AMS需要监听应用进程的死亡通知。</p>
</div>
<p>有了IApplicationThread接口，AMS就可以和应用进程交互了。例如，对于下面一个简单的函数：</p>
<p>[--&gt;ActivityThread.java::scheduleStopActivity]</p>
<div>
    <p>public final void scheduleStopActivity(IBindertoken, boolean showWindow,</p>
    <p>                                     intconfigChanges) {</p>
    <p>  queueOrSendMessage(//该函数内部将给一个Handler发送对应的消息</p>
    <p>               showWindow ? H.STOP_ACTIVITY_SHOW : H.STOP_ACTIVITY_HIDE,</p>
    <p>               token, 0, configChanges);</p>
    <p> }</p>
</div>
<p>当AMS想要停止（stop）一个Activity时，会调用对应进程IApplicationThread Binder客户端的scheduleStopActivity函数。该函数服务端实现的就是向ActivityThread所在线程发送一个消息。在应用进程中，ActivityThread运行在主线程中，所以这个消息最终在主线程被处理。</p>
<div>
    <p><strong>提示</strong>Activity的onStop函数也将在主线程中被调用。</p>
</div>
<p>IApplicationThread仅仅是AMS和另外一个进程交互的接口，除此之外，AMS还需要更多的有关该进程的信息。在AMS中，进程的信息都保存在ProcessRecord数据结构中。那么，ProcessRecord是什么呢？先来看setSystemProcess的第二个关键点，即newProcessRecordLocked函数，其代码如下：</p>
<p>[--&gt;ActivityManagerService.java::newProcessRecordLocked]</p>
<div>
    <p>final ProcessRecordnewProcessRecordLocked(IApplicationThread thread,</p>
    <p>                ApplicationInfo info, String customProcess) {</p>
    <p>   Stringproc = customProcess != null ? customProcess : info.processName;</p>
    <p>  BatteryStatsImpl.Uid.Proc ps = null;</p>
    <p>  BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();</p>
    <p>  synchronized (stats) {</p>
    <p>        //BSImpl将为该进程创建一个耗电量统计项</p>
    <p>        ps =stats.getProcessStatsLocked(info.uid, proc);</p>
    <p>   }</p>
    <p>   //创建一个ProcessRecord对象，用于和其他进程通信的thread作为第一个参数</p>
    <p>   returnnew ProcessRecord(ps, thread, info, proc);</p>
    <p> }</p>
</div>
<p>ProcessRecord的成员变量较多，先来看看再其构造函数中都初始化了哪些成员变量。</p>
<p>[--&gt;ProcessRecord.java::ProcessRecord]</p>
<div>
    <p>ProcessRecord(BatteryStatsImpl.Uid.Proc_batteryStats, </p>
    <p>        IApplicationThread_thread,ApplicationInfo _info, String _processName) {</p>
    <p>    batteryStats = _batteryStats; //用于电量统计</p>
    <p>     info =_info; //保存ApplicationInfo</p>
    <p>    processName = _processName; //保存进程名</p>
    <p>      //一个进程能运行多个Package，pkgList用于保存package名</p>
    <p>    pkgList.add(_info.packageName);</p>
    <p>     thread= _thread;//保存IApplicationThread,通过它可以和应用进程交互</p>
    <p> </p>
    <p>     //下面这些xxxAdj成员变量和进程调度优先级及OOM_adj有关。以后再分析它的作用</p>
    <p>     maxAdj= ProcessList.EMPTY_APP_ADJ;</p>
    <p>    hiddenAdj = ProcessList.HIDDEN_APP_MIN_ADJ;</p>
    <p>    curRawAdj = setRawAdj = -100;</p>
    <p>     curAdj= setAdj = -100;</p>
    <p>   //用于控制该进程是否常驻内存（即使被杀掉，系统也会重启它），只有重要的进程才会有此待遇</p>
    <p>    persistent = false; </p>
    <p>     removed= false;</p>
    <p>}</p>
</div>
<p>ProcessRecord除保存和应用进程通信的IApplicationThread对象外，还保存了进程名、不同状态对应的Oom_adj值及一个ApplicationInfo。一个进程虽然可运行多个Application，但是ProcessRecord一般保存该进程中先运行的那个Application的ApplicationInfo。</p>
<p>至此，已经创建了一个ProcessRecord对象，和其他应用进程不同的是，该对象对应的进程为SystemServer。为了彰显其特殊性，AMS为其中的一些成员变量设置了特定的值：</p>
<div>
    <p>   app.persistent = true;//设置该值为true </p>
    <p>   app.pid =MY_PID;//设置pid为SystemServer的进程号</p>
    <p>   app.maxAdj= ProcessList.SYSTEM_ADJ;//设置最大OOM_Adj，系统进程默认值为-16</p>
    <p>   //另外，app的processName被设置成“system”</p>
</div>
<p>这时，一个针对SystemServer的ProcessRecord对象就创建完成了。此后AMS将把它并入自己的势力范围内。</p>
<p>AMS中有两个成员变量用于保存ProcessRecord，一个是mProcessNames，另一个是mPidsSelfLocked，如图6-6所示为这两个成员变量的数据结构示意图。</p>

![image](images/chapter6/image006.png)

<p>图6-6  mPidsSelfLocked和mProcessNames数据结构示意图</p>
<h4>3.  AMS的setSystemProcess总结</h4>
<p>现在来总结回顾setSystemProcess的工作：</p>
<p>·  注册AMS、meminfo、gfxinfo等服务到ServiceManager中。</p>
<p>·  根据PKMS返回的ApplicationInfo初始化Android运行环境，并创建一个代表SystemServer进程的ProcessRecord，从此，SystemServer进程也并入AMS的管理范围内。</p>
<p> </p>
<h3>6.2.3  AMS的installSystemProviders函数分析</h3>
<p>还记得Settings数据库吗？SystemServer中很多Service都需要向它查询配置信息。为此，Android提供了一个SettingsProvider来帮助开发者。该Provider在SettingsProvider.apk中，installSystemProviders就会加载该APK并把SettingsProvider放到SystemServer进程中来运行。</p>
<p>此时的SystemServer已经加载了framework-res.apk，现在又要加载另外一个APK文件，这就是多个APK运行在同一进程的典型案例。另外，通过installSystemProviders函数还能见识ContentProvider的安装过程，下面就来分析它。</p>
<div>
    <p><strong>提示</strong>读者在定制自己的Android系统时，万不可去掉/system/app/SettingsProvider.apk，否则系统将无法正常启动。</p>
</div>
<p>[--&gt;ActivityManagerService.java::installSystemProviders]</p>
<div>
    <p>public static final void installSystemProviders(){</p>
    <p> List&lt;ProviderInfo&gt; providers;</p>
    <p> synchronized (mSelf) {</p>
    <p>    /*</p>
    <p>    从mProcessNames找到进程名为“system”且uid为SYSTEM_UID的ProcessRecord，</p>
    <p>    返回值就是前面在installSystemApplication中创建的那个ProcessRecord，它代表</p>
    <p>    SystemServer进程</p>
    <p>    */</p>
    <p>   ProcessRecord app = mSelf.mProcessNames.get("system",Process.SYSTEM_UID);</p>
    <p> </p>
    <p>    //①关键调用，见下文分析</p>
    <p>    providers= mSelf.generateApplicationProvidersLocked(app);</p>
    <p>    if(providers != null) {</p>
    <p>       ......//将非系统APK（即未设ApplicationInfo.FLAG_SYSTEM标志）提供的Provider</p>
    <p>      //从providers列表中去掉</p>
    <p>        }</p>
    <p>     if(providers != null) {//②为SystemServer进程安装Provider</p>
    <p>         mSystemThread.installSystemProviders(providers);</p>
    <p>     }</p>
    <p>    //监视Settings数据库中Secure表的变化，目前只关注long_press_timeout配置的变化</p>
    <p>    mSelf.mCoreSettingsObserver = new CoreSettingsObserver(mSelf);</p>
    <p> </p>
    <p>     //UsageStatsService的工作，以后再讨论</p>
    <p>    mSelf.mUsageStatsService.monitorPackages();</p>
    <p> }</p>
</div>
<p>在代码中列出了两个关键调用，分别是：</p>
<p>·  调用generateApplicationProvidersLocked函数，该函数返回一个ProviderInfo List。</p>
<p>·  调用ActivityThread的installSystemProviders函数。ActivityThread可以看做是进程的Android运行环境，那么installSystemProviders表示为该进程安装ContentProvider。</p>
<div>
    <p><strong>注意</strong>此处不再区分系统进程还是应用进程。由于只和ActivityThread交互，因此它运行在什么进程无关紧要。</p>
</div>
<p>下面来看第一个关键点generateApplicationProvidersLocked函数。</p>
<h4>1.  AMS的 generateApplicationProvidersLocked函数分析</h4>
<p>[--&gt;ActivityManagerService.java::generateApplicationProvidersLocked]</p>
<div>
    <p>private final List&lt;ProviderInfo&gt; generateApplicationProvidersLocked(</p>
    <p>                              ProcessRecordapp) {</p>
    <p>  List&lt;ProviderInfo&gt; providers = null;</p>
    <p>  try {</p>
    <p>         //①向PKMS查询满足要求的ProviderInfo，最重要的查询条件包括：进程名和进程uid</p>
    <p>         providers = AppGlobals.getPackageManager().</p>
    <p>            queryContentProviders(app.processName, app.info.uid,</p>
    <p>             STOCK_PM_FLAGS | PackageManager.GET_URI_PERMISSION_PATTERNS);</p>
    <p>        } ......</p>
    <p>   if(providers != null) {</p>
    <p>      finalint N = providers.size();</p>
    <p>      for(int i=0; i&lt;N; i++) {</p>
    <p>         //②AMS对ContentProvider的管理，见下文解释</p>
    <p>        ProviderInfo cpi =   (ProviderInfo)providers.get(i);</p>
    <p>        ComponentName comp = new ComponentName(cpi.packageName, cpi.name);</p>
    <p>        ContentProviderRecord cpr = mProvidersByClass.get(comp);</p>
    <p>         if(cpr == null) {</p>
    <p>             cpr = new ContentProviderRecord(cpi, app.info, comp);</p>
    <p>              //ContentProvider在AMS中用ContentProviderRecord来表示</p>
    <p>              mProvidersByClass.put(comp, cpr);//保存到AMS的mProvidersByClass中</p>
    <p>          }</p>
    <p>          //将信息也保存到ProcessRecord中</p>
    <p>         app.pubProviders.put(cpi.name, cpr);</p>
    <p>          //保存PackageName到ProcessRecord中</p>
    <p>         app.addPackage(cpi.applicationInfo.packageName); </p>
    <p>         //对该APK进行dex优化</p>
    <p>         ensurePackageDexOpt(cpi.applicationInfo.packageName);</p>
    <p>         }</p>
    <p>     }</p>
    <p>    returnproviders;</p>
    <p> }</p>
</div>
<p>由以上代码可知：generateApplicationProvidersLocked先从PKMS那里查询满足条件的ProviderInfo信息，而后将它们分别保存到AMS和ProcessRecord中对应的数据结构中。</p>
<p>先看查询函数queryContentProviders。</p>
<h5>（1） PMS中 queryContentProviders函数分析</h5>
<p>[--&gt;PackageManagerService.java::queryContentProviders]</p>
<div>
    <p>public List&lt;ProviderInfo&gt;queryContentProviders(String processName,</p>
    <p>    int uid,int flags) {</p>
    <p>  ArrayList&lt;ProviderInfo&gt; finalList = null;</p>
    <p>  synchronized (mPackages) {</p>
    <p>     //还记得mProvidersByComponent的作用吗？它以ComponentName为key，保存了</p>
    <p>     //PKMS扫描APK得到的PackageParser.Provider信息。读者可参考图4-8</p>
    <p>     finalIterator&lt;PackageParser.Provider&gt; i =</p>
    <p>                      mProvidersByComponent.values().iterator();</p>
    <p>       while(i.hasNext()) {</p>
    <p>          final PackageParser.Provider p = i.next();</p>
    <p>           //下面的if语句将从这些Provider中搜索本例设置的processName为“system”，</p>
    <p>          //uid为SYSTEM_UID，flags为FLAG_SYSTEM的Provider</p>
    <p>          if (p.info.authority != null</p>
    <p>               &amp;&amp; (processName == null</p>
    <p>                   ||(p.info.processName.equals(processName)</p>
    <p>                        &amp;&amp;p.info.applicationInfo.uid == uid))</p>
    <p>               &amp;&amp; mSettings.isEnabledLPr(p.info, flags)</p>
    <p>                &amp;&amp; (!mSafeMode || ( p.info.applicationInfo.flags</p>
    <p>                        &amp;ApplicationInfo.FLAG_SYSTEM) != 0)) {</p>
    <p>                    if (finalList == null) {</p>
    <p>                        finalList = newArrayList&lt;ProviderInfo&gt;(3);</p>
    <p>                   }</p>
    <p>                 //由PackageParser.Provider得到ProviderInfo，并添加到finalList中</p>
    <p>                //关于Provider类及ProviderInfo类，可参考图4-5</p>
    <p>                  finalList.add(PackageParser.generateProviderInfo(p,flags));</p>
    <p>               }</p>
    <p>           }</p>
    <p>        }</p>
    <p> </p>
    <p>   if(finalList != null)</p>
    <p>      //最终结果按provider的initOrder排序，该值用于表示初始化ContentProvider的顺序</p>
    <p>      Collections.sort(finalList, mProviderInitOrderSorter);</p>
    <p>   returnfinalList;//返回最终结果</p>
    <p> }</p>
</div>
<p>queryContentProviders函数很简单，就是从PKMS那里查找满足条件的Provider，然后生成AMS使用的ProviderInfo信息。为何偏偏能找到SettingsProvider呢？来看它的AndroidManifest.xml文件，如图6-7所示。</p>

![image](images/chapter6/image007.png)

<p>图6-7  SettingsProvider的AndroidManifest.xml文件示意</p>
<p>由图6-7可知，SettingsProvider设置了其uid为“android.uid.system”，同时在application中设置了process名为“system”。而在framework-res.apk中也做了相同的设置。所以，现在可以确认SettingsProvider将和framework-res.apk运行在同一个进程，即SystemServer中。</p>
<div>
    <p><strong>提示</strong>从运行效率角度来说，这样做也是合情合理的。因为SystemServer的很多Service都依赖Settings数据库，把它们放在同一个进程中，可以降低由于进程间通信带来的效率损失。</p>
</div>
<h5>（2）  关于ContentProvider的介绍</h5>
<p>前面介绍的从PKMS那里查询到的ProviderInfo还属于公有财产，现在我们要将它与AMS及ProcessRecord联系起来。</p>
<p>·  AMS保存ProviderInfo的原因是它要管理ContentProvider。</p>
<p>·  ProcessRecord保存ProviderInfo的原因是ContentProvider最终要落实到一个进程中。其实也是为了方便AMS管理，例如该进程一旦退出，AMS需要把其中的ContentProvider信息从系统中去除。</p>
<p>AMS及ProcessRecord均使用了一个新的数据结构ContentProviderRecord来管理ContentProvider信息。图6-8展示了ContentProviderRecord相应的数据结构。</p>

![image](images/chapter6/image008.png)

<p>图6-8  ContentProvicerRecord及相应的“管理团队”</p>
<p>由图6-8可知：</p>
<p>·  ContentProviderRecord从ContentProviderHolder派生，内部保存了ProviderInfo、该Provider所驻留的进程ProcessRecord，以及使用该ContentProvider的客户端进程ProcessRecord（即clients成员变量）。</p>
<p>·  AMS的mProviderByClass成员变量及ProcessRecord的pubProviders成员变量均以ComponentName为Key来保存对应的ContentProviderRecord对象。</p>
<p>至此，Provider信息已经保存到AMS及ProcessRecord中了。那么，下一步的工作是什么呢？ </p>
<h4>2.  ActivityThread 的installSystemProviders函数分析</h4>
<p>在AMS和ProcessRecord中都保存了Provider信息，但这些仅仅都是一些信息，并不是ContentProvider，因此下面要创建一个ContentProvider实例（即SettingsProvider对象）。该工作由ActivityThread的installSystemProviders来完成，代码如下：</p>
<p>[--&gt;ActivityThread.java::installSystemProviders]</p>
<div>
    <p>public final void installSystemProviders(List&lt;ProviderInfo&gt;providers) {</p>
    <p>  if(providers != null) </p>
    <p>    //调用installContentProviders，第一个参数真实类型是Application</p>
    <p>   installContentProviders(mInitialApplication, providers);</p>
    <p>}</p>
</div>
<p>installContentProviders这个函数是所有ContentProvider产生的必经之路，其代码如下：</p>
<p>[--&gt;ActivityThread.java::installContentProviders]</p>
<div>
    <p>private void installContentProviders(</p>
    <p>           Context context, List&lt;ProviderInfo&gt; providers) {</p>
    <p> </p>
    <p>   finalArrayList&lt;IActivityManager.ContentProviderHolder&gt; results =</p>
    <p>                   new ArrayList&lt;IActivityManager.ContentProviderHolder&gt;();</p>
    <p> </p>
    <p>  Iterator&lt;ProviderInfo&gt; i = providers.iterator();</p>
    <p>   while(i.hasNext()) {</p>
    <p>    ProviderInfo cpi = i.next();</p>
    <p>     //①调用installProvider函数，得到一个IContentProvider对象</p>
    <p>    IContentProvider cp = installProvider(context, null, cpi, false);</p>
    <p>     if (cp!= null) {</p>
    <p>       IActivityManager.ContentProviderHolder cph =</p>
    <p>                    newIActivityManager.ContentProviderHolder(cpi);</p>
    <p>       cph.provider = cp;</p>
    <p>       //将返回的cp保存到results数组中</p>
    <p>       results.add(cph);</p>
    <p>        synchronized(mProviderMap){</p>
    <p>         //mProviderRefCountMap，；类型为HashMap&lt;IBinder,ProviderRefCount&gt;，</p>
    <p>         //主要通过ProviderRefCount对ContentProvider进行引用计数控制，一旦引用计数</p>
    <p>         //降为零，表示系统中没有地方使用该ContentProvider，要考虑从系统中注销它</p>
    <p>         mProviderRefCountMap.put(cp.asBinder(), new ProviderRefCount(10000));</p>
    <p>         }</p>
    <p>     }</p>
    <p>   }</p>
    <p>    try {</p>
    <p>       //②调用AMS的publishContentProviders注册这些ContentProvider，第一个参数</p>
    <p>       //为ApplicationThread</p>
    <p>       ActivityManagerNative.getDefault().publishContentProviders(</p>
    <p>                      getApplicationThread(), results);</p>
    <p>        } ......</p>
    <p> </p>
    <p>    }</p>
</div>
<p>installContentProviders实际上是标准的ContentProvider安装时调用的程序。安装ContentProvider包括两方面的工作：</p>
<p>·  先在ActivityThread中通过installProvider得到一个ContentProvider实例。</p>
<p>·  向AMS发布这个ContentProvider实例。如此这般，一个APK中声明的ContentProvider才能登上历史舞台，发挥其该有的作用。</p>
<div>
    <p>提示 上述工作其实和Binder Service类似，一个Binder Service也需要先创建，然后注册到ServiceManager中。</p>
</div>
<p>马上来看ActivityThread的installProvider函数。</p>
<h5>（1） ActivityThread的installProvider函数分析</h5>
<p>[--&gt;ActivityThread.java::installProvider]</p>
<div>
    <p>private IContentProvider installProvider(Contextcontext,</p>
    <p>           IContentProvider provider, ProviderInfoinfo, boolean noisy) {</p>
    <p>//注意本例所传的参数：context为mInitialApplication，provider为null,info不为null，</p>
    <p>// noisy为false</p>
    <p> </p>
    <p>  ContentProvider localProvider = null;</p>
    <p>   if(provider == null) {</p>
    <p>      Context c = null;</p>
    <p>      ApplicationInfo ai = info.applicationInfo;</p>
    <p>       /*</p>
    <p>        下面这个if判断的作用就是为该ContentProvider找到对应的Application。</p>
    <p>        在AndroidManifest.xml中，ContentProvider是Application的子标签，所以</p>
    <p>       ContentProvider和Application有一种对应关系。在本例中，传入的context（</p>
    <p>        其实是mInitialApplication）代表的是framework-res.apk，而Provider代表的</p>
    <p>        是SettingsProvider。而SettingsProvider.apk所对应的Application还未创建，</p>
    <p>        所以下面的判断语句最终会进入最后的else分支</p>
    <p>      */</p>
    <p>       if(context.getPackageName().equals(ai.packageName)) {</p>
    <p>          c= context;</p>
    <p>       }else if (mInitialApplication != null &amp;&amp;</p>
    <p>                 mInitialApplication.getPackageName().equals(ai.packageName)){</p>
    <p>          c = mInitialApplication;</p>
    <p>       } else {</p>
    <p>         try{</p>
    <p>             //ai.packageName应该是SettingsProvider.apk的Package，</p>
    <p>             //名为“com.android.providers.settings”</p>
    <p>             //下面将创建一个Context，指向该APK</p>
    <p>              c = context.createPackageContext(ai.packageName,</p>
    <p>                           Context.CONTEXT_INCLUDE_CODE);</p>
    <p>           } </p>
    <p>       }//if(context.getPackageName().equals(ai.packageName))判断结束</p>
    <p> </p>
    <p>       if (c == null)  return null;</p>
    <p>       try {</p>
    <p>         /*</p>
    <p>         为什么一定要找到对应的Context呢？除了ContentProvider和Application的</p>
    <p>         对应关系外，还有一个决定性原因：即只有对应的Context才能加载对应APK的Java字节码，</p>
    <p>         从而可通过反射机制生成ContentProvider实例</p>
    <p>        */</p>
    <p>         finaljava.lang.ClassLoader cl = c.getClassLoader();</p>
    <p>
        <a>         </a>//通过Java反射机制得到真正的ContentProvider，</p>
    <p>         //此处将得到一个SettingsProvider对象</p>
    <p>         localProvider=(ContentProvider)cl.loadClass(info.name).newInstance();</p>
    <p>         //从ContentProvider中取出其mTransport成员（见下文分析）</p>
    <p>         provider =localProvider.getIContentProvider();</p>
    <p>         if (provider == null) return null;</p>
    <p>         //初始化该ContentProvider，内部会调用其onCreate函数</p>
    <p>        localProvider.attachInfo(c, info);</p>
    <p>      }......</p>
    <p>   }//if(provider == null)判断结束</p>
    <p>  synchronized (mProviderMap) {</p>
    <p>      /*</p>
    <p>        ContentProvider必须指明一个和多个authority，在第4章曾经提到过，</p>
    <p>         在URL中host:port的组合表示一个authority。这个单词不太好理解，可简单</p>
    <p>         认为它用于指定ContentProvider的位置（类似网站的域名）</p>
    <p>     */</p>
    <p>      String names[] =PATTERN_SEMICOLON.split(info.authority);</p>
    <p>      for (int i=0; i&lt;names.length; i++) {</p>
    <p>             ProviderClientRecord pr = newProviderClientRecord(names[i], </p>
    <p>                                           provider, localProvider);</p>
    <p>            try {</p>
    <p>                //下面这句对linkToDeath的调用颇让人费解，见下文分析</p>
    <p>                  provider.asBinder().linkToDeath(pr, 0);</p>
    <p>                  mProviderMap.put(names[i], pr);</p>
    <p>           }......</p>
    <p>      }//for循环结束</p>
    <p>     if(localProvider != null) {</p>
    <p>         // mLocalProviders用于存储由本进程创建的ContentProvider信息</p>
    <p>          mLocalProviders.put(provider.asBinder(),</p>
    <p>                   new ProviderClientRecord(null, provider, localProvider));</p>
    <p>              }</p>
    <p>     }//synchronized 结束</p>
    <p>  return provider;</p>
    <p>}</p>
</div>
<p>以上代码不算复杂，但是涉及一些数据结构和一句令人费解的对inkToDeath函数的调用。先来说说那句令人费解的调用。</p>
<p>在本例中，provider变量并非通过函数参数传入，而是在本进程内部创建的。provider在本例中是Bn端（后面分析ContentProvider的getIContentProvider时即可知道），Bn端进程为Bn端设置死亡通知本身就比较奇怪。如果Bn端进程死亡，它设置的死亡通知也无法发送给自己。幸好源代码中有句注释：“Cache the pointer for the remote provider”。意思是如果provider参数是通过installProvider传递过来的（即该Provider代表远端进程的ContentProvider，此时它应为Bp端），那么这种处理是合适的。不管怎样，这仅仅是为了保存pointer，所以也无关宏旨。</p>
<p>至于代码中涉及的数据结构，我们整理为如图6-9所示。</p>

![image](images/chapter6/image009.png)

<p>图6-9  ActivityThread中ContentProvider涉及的数据结构</p>
<p>由图6-9可知：</p>
<p>·  ContentProvider类本身只是一个容器，而跨进程调用的支持是通过内部类Transport实现的。Transport从ContentProviderNative派生，而ContentProvider的成员变量mTransport指向该Transport对象。ContentProvider的getIContentProvider函数即返回mTransport成员变量。</p>
<p>·  ContentProviderNative从Binder派生，并实现了IContentProvider接口。其内部类ContentProviderProxy是供客户端使用的。</p>
<p>·  ProviderClientRecord是ActivityThread提供的用于保存ContentProvider信息的一个数据结构。它的mLocalProvider用于保存ContentProvider对象，mProvider用于保存IContentProvider对象。另外一个成员mName用于保存该ContentProvider的一个authority。注意，ContentProvider可以定义多个authority，就好像一个网站有多个域名一样。</p>
<p>至此，本例中的SettingProvider已经创建完毕，接下来的工作就是把它推向历史舞台——即发布该Provider。</p>
<h5>（2） ASM的 publishContentProviders分析</h5>
<p>publicContentProviders函数用于向AMS注册ContentProviders，其代码如下：</p>
<p>[--&gt;ActivityManagerService.java::publishContentProviders]</p>
<div>
    <p> publicfinal void publishContentProviders(IApplicationThread caller,</p>
    <p>
        <a> </a>                          List&lt;ContentProviderHolder&gt;providers) {</p>
    <p>  ......</p>
    <p>  synchronized(this){</p>
    <p>    //找到调用者所在的ProcessRecord对象</p>
    <p>   finalProcessRecord r = getRecordForAppLocked(caller);</p>
    <p>   ......</p>
    <p>   finallong origId = Binder.clearCallingIdentity();</p>
    <p>   final intN = providers.size();</p>
    <p>   for (inti=0; i&lt;N; i++) {</p>
    <p>      ContentProviderHolder src = providers.get(i);</p>
    <p>       ......</p>
    <p>       //①注意：先从该ProcessRecord中找对应的ContentProviderRecord</p>
    <p>     ContentProviderRecord dst = r.pubProviders.get(src.info.name);</p>
    <p>      if(dst != null) {</p>
    <p>          ComponentName comp = newComponentName(dst.info.packageName,</p>
    <p>                                                         dst.info.name);</p>
    <p>          //以ComponentName为key，保存到mProvidersByClass中</p>
    <p>         mProvidersByClass.put(comp, dst);</p>
    <p>         String names[] = dst.info.authority.split(";");</p>
    <p>         for (int j = 0; j &lt; names.length; j++)</p>
    <p>               mProvidersByName.put(names[j], dst);//以authority为key，保存</p>
    <p>              //mLaunchingProviders用于保存处于启动状态的Provider</p>
    <p>               int NL = mLaunchingProviders.size();</p>
    <p>               int j;</p>
    <p>               for (j=0; j&lt;NL; j++) {</p>
    <p>                 if (mLaunchingProviders.get(j) ==dst) {</p>
    <p>                      mLaunchingProviders.remove(j);</p>
    <p>                      j--;</p>
    <p>                      NL--;</p>
    <p>                  }//</p>
    <p>              }//for (j=0; j&lt;NL; j++)结束</p>
    <p>              synchronized (dst) {</p>
    <p>                  dst.provider = src.provider;</p>
    <p>                  dst.proc = r;</p>
    <p>                  dst.notifyAll();</p>
    <p>              }//synchronized结束</p>
    <p>             updateOomAdjLocked(r);//每发布一个Provider，需要调整对应进程的oom_adj</p>
    <p>            }//for(int j = 0; j &lt; names.length; j++)结束</p>
    <p>        }//for(int i=0; i&lt;N; i++)结束</p>
    <p>        Binder.restoreCallingIdentity(origId);</p>
    <p>    }// synchronized(this)结束</p>
    <p> }</p>
</div>
<p>这里应解释一下publishContentProviders的工作流程：</p>
<p>·  先根据调用者的pid找到对应的ProcessRecord对象。</p>
<p>·  该ProcessRecord的pubProviders中保存了ContentProviderRecord信息。该信息由前面介绍的AMS的generateApplicationProvidersLocked函数根据Package本身的信息生成。此处将判断要发布的ContentProvider是否由该Package声明。</p>
<p>·  如果判断返回成功，则将该ContentProvider及其对应的authority加到mProvidersByName中。注意，AMS中还有一个mProvidersByClass变量，该变量以ContentProvider的ComponentName为key，即系统提供多种方式找到某一个ContentProvider，一种是通过 authority，另一种方式就是指明ComponentName。</p>
<p>·  mLaunchingProviders和最后的notifyAll函数用于通知那些等待ContentProvider所在进程启动的客户端进程。例如，进程A要查询一个数据库，需要通过进程B中的某个ContentProvider 来实施。如果B还未启动，那么AMS就需要先启动B。在这段时间内，A需要等待B启动并注册对应的ContentProvider。B一旦完成注册，就需要告知A退出等待以继续后续的查询工作。</p>
<p>现在，一个SettingsProvider就算正式在系统中挂牌并注册了，此后，和Settings数据库相关的操作均由它来打理。</p>
<h4>3.  ASM的installSystemProviders总结</h4>
<p>AMS的installSystemProviders函数其实就是用于启动SettingsProvider，其中比较复杂的是ContentProvider相关的数据结构，读者可参考图6-9。</p>
<h3>6.2.4  ASM的systemReady分析</h3>
<p>作为核心服务，AMS的systemReady会做什么呢？由于该函数内容较多，我们将它分为三段。首先看第一段的工作。</p>
<h4>1.  systemReady第一阶段的工作</h4>
<p>[--&gt;ActivityManagerService.java::systemReady]</p>
<div>
    <p>public void systemReady(final RunnablegoingCallback) {</p>
    <p>     synchronized(this){</p>
    <p>       ......</p>
    <p>       if(!mDidUpdate) {//判断是否为升级</p>
    <p>           if(mWaitingUpdate)   return; //升级未完成，直接返回</p>
    <p>           //准备PRE_BOOT_COMPLETED广播</p>
    <p>            Intent intent = newIntent(Intent.ACTION_PRE_BOOT_COMPLETED);</p>
    <p>           List&lt;ResolveInfo&gt; ris = null;</p>
    <p>           //向PKMS查询该广播的接收者</p>
    <p>            ris= AppGlobals.getPackageManager().queryIntentReceivers(</p>
    <p>                                intent, null,0);</p>
    <p>            ......//从返回的结果中删除那些非系统APK的广播接收者</p>
    <p>            intent.addFlags(Intent.FLAG_RECEIVER_BOOT_UPGRADE);</p>
    <p>            //读取/data/system/called_pre_boots.dat文件，这里存储了上次启动时候已经</p>
    <p>             //接收并处理PRE_BOOT_COMPLETED广播的组件。鉴于该广播的特殊性，系统希望</p>
    <p>           //该广播仅被这些接收者处理一次</p>
    <p>             ArrayList&lt;ComponentName&gt;lastDoneReceivers =</p>
    <p>                                          readLastDonePreBootReceivers();</p>
    <p>              final ArrayList&lt;ComponentName&gt; doneReceivers=</p>
    <p>                                         newArrayList&lt;ComponentName&gt;();</p>
    <p>             ......//从PKMS返回的接收者中删除那些已经处理过该广播的对象</p>
    <p> </p>
    <p>            for (int i=0; i&lt;ris.size(); i++) {</p>
    <p>                ActivityInfo ai = ris.get(i).activityInfo;</p>
    <p>                 ComponentName comp = newComponentName(ai.packageName, ai.name);</p>
    <p>                doneReceivers.add(comp);</p>
    <p>                intent.setComponent(comp);</p>
    <p>                IIntentReceiver finisher = null;</p>
    <p>                if (i == ris.size()-1) {</p>
    <p>                    //为最后一个广播接收者注册一个回调通知，当该接收者处理完广播后，将调用该</p>
    <p>                   //回调</p>
    <p>                   finisher = new IIntentReceiver.Stub() {</p>
    <p>                       public voidperformReceive(Intent intent, int resultCode,</p>
    <p>                                        Stringdata, Bundle extras, boolean ordered,</p>
    <p>                                        booleansticky) {</p>
    <p>                                 mHandler.post(newRunnable() {</p>
    <p>                                   public void run(){</p>
    <p>                                   synchronized(ActivityManagerService.this) {</p>
    <p>                                       mDidUpdate = true;</p>
    <p>                                    }</p>
    <p>                                 //保存那些处理过该广播的接收者信息</p>
    <p>                                 writeLastDonePreBootReceivers(doneReceivers);</p>
    <p>                                  showBootMessage(mContext.getText(</p>
    <p>                                        R.string.android_upgrading_complete),</p>
    <p>                                        false);</p>
    <p>                                   systemReady(goingCallback);</p>
    <p>                                }//run结束</p>
    <p>                            });// new Runnable结束</p>
    <p>                        }//performReceive结束</p>
    <p>                  };//finisher创建结束</p>
    <p>              }// if (i == ris.size()-1)判断结束</p>
    <p>           //发送广播给指定的接收者</p>
    <p>             broadcastIntentLocked(null, null, intent,null, finisher,</p>
    <p>              0, null, null, null, true, false, MY_PID, Process.SYSTEM_UID);</p>
    <p>            if (finisher != null)   mWaitingUpdate = true;</p>
    <p>         }</p>
    <p>       if(mWaitingUpdate)    return;</p>
    <p>       mDidUpdate= true;</p>
    <p>  }</p>
    <p>            </p>
    <p>   mSystemReady = true;</p>
    <p>    if(!mStartRunning)  return;</p>
    <p>   }//synchronized(this)结束</p>
</div>
<p>由以上代码可知，systemReady第一阶段的工作并不轻松，其主要职责是发送并处理与PRE_BOOT_COMPLETED广播相关的事情。目前代码中还没有接收该广播的地方，不过从代码中的注释中可猜测到，该广播接收者的工作似乎和系统升级有关。</p>
<div>
    <p><strong>建议</strong>如有哪位读者了解与此相关的知识，不妨和大家分享。</p>
</div>
<p>下面来介绍systemReady第二阶段的工作。</p>
<h4>2.  systemReady第二阶段的工作</h4>
<p>[--&gt;ActivityManagerService.java::systemReady]</p>
<div>
    <p>   ArrayList&lt;ProcessRecord&gt;procsToKill = null;</p>
    <p>  synchronized(mPidsSelfLocked) {</p>
    <p>     for(int i=mPidsSelfLocked.size()-1; i&gt;=0; i--) {</p>
    <p>         ProcessRecord proc = mPidsSelfLocked.valueAt(i);</p>
    <p>          //从mPidsSelfLocked中找到那些先于AMS启动的进程，哪些进程有如此能耐，</p>
    <p>         //在AMS还未启动完毕就启动完了呢？对，那些声明了persistent为true的进程有可能</p>
    <p>          if(!isAllowedWhileBooting(proc.info)){</p>
    <p>            if (procsToKill == null) </p>
    <p>                 procsToKill = new ArrayList&lt;ProcessRecord&gt;();</p>
    <p>             procsToKill.add(proc);</p>
    <p>           }</p>
    <p>       }//for结束</p>
    <p>   }// synchronized结束</p>
    <p>        </p>
    <p>   synchronized(this){</p>
    <p>       if(procsToKill != null) {</p>
    <p>          for (int i=procsToKill.size()-1; i&gt;=0; i--) {</p>
    <p>             ProcessRecord proc = procsToKill.get(i);</p>
    <p>             //把这些进程关闭，removeProcessLocked函数比较复杂，以后再分析</p>
    <p>              removeProcessLocked(proc, true, false);</p>
    <p>           }</p>
    <p>        }</p>
    <p>      //至此，系统已经准备完毕</p>
    <p>      mProcessesReady = true;</p>
    <p>   }</p>
    <p>  synchronized(this) {</p>
    <p>     if(mFactoryTest == SystemServer.FACTORY_TEST_LOW_LEVEL) {</p>
    <p>     }//和工厂测试有关，不对此进行讨论</p>
    <p>   }</p>
    <p>   //查询Settings数据，获取一些配置参数</p>
    <p>   retrieveSettings();</p>
</div>
<p>systemReady第二阶段的工作包括：</p>
<p>·  杀死那些竟然在AMS还未启动完毕就先启动的应用进程。注意，这些应用进程一定是APK所在的Java进程，因为只有应用进程才会向AMS注册，而一般Native（例如mediaserver）进程是不会向AMS注册的。</p>
<p>·  从Settings数据库中获取配置信息，目前只取4个配置参数，分别是："debug_app"（设置需要debug的app的名称）、"wait_for_debugger"（如果为1，则等待调试器，否则正常启动debug_app）、"always_finish_activities"（当一个activity不再有地方使用时，是否立即对它执行destroy）、"font_scale"（用于控制字体放大倍数，这是Android 4.0新增的功能）。以上配置项由Settings数据库的System表提供。</p>
<p> </p>
<h4>3.  systemReady第三阶段的工作</h4>
<p>[--&gt;ActivityManagerService.java::systemReady]</p>
<div>
    <p>//调用systemReady传入的参数，它是一个Runnable对象，下节将分析此函数</p>
    <p>if (goingCallback != null) <strong>goingCallback.run</strong>();</p>
    <p>        </p>
    <p> synchronized (this) {</p>
    <p>     if(mFactoryTest != SystemServer.FACTORY_TEST_LOW_LEVEL) {</p>
    <p>        try{</p>
    <p>            //从PKMS中查询那些persistent为1的ApplicationInfo</p>
    <p>            List apps = AppGlobals.getPackageManager().</p>
    <p>                           getPersistentApplications(STOCK_PM_FLAGS);</p>
    <p>            if (apps != null) {</p>
    <p>                int N = apps.size();</p>
    <p>                 int i;</p>
    <p>                 for (i=0; i&lt;N; i++) {</p>
    <p>                   ApplicationInfo info = (ApplicationInfo)apps.get(i);</p>
    <p>                   //由于framework-res.apk已经由系统启动，所以这里需要把它去除</p>
    <p>                   //framework-res.apk的packageName为"android" </p>
    <p>                   if (info != null &amp;&amp; !info.packageName.equals("android"))</p>
    <p>                        addAppLocked(info);//启动该Application所在的进程</p>
    <p>                      }</p>
    <p>                   }</p>
    <p>               }......</p>
    <p>      }</p>
    <p> </p>
    <p>   mBooting= true; //设置mBooting变量为true，其作用后面会介绍</p>
    <p>   try {</p>
    <p>        if(AppGlobals.getPackageManager().hasSystemUidErrors()) {</p>
    <p>            ......//处理那些Uid有错误的Application</p>
    <p>      }......</p>
    <p>       //启动全系统第一个Activity，即Home</p>
    <p>      mMainStack.resumeTopActivityLocked(null);</p>
    <p>    }// synchronized结束</p>
    <p>}</p>
</div>
<p>systemReady第三阶段的工作有3项：</p>
<p>·  调用systemReady设置的回调对象goingCallback的run函数。</p>
<p>·  启动那些声明了persistent的APK。</p>
<p>·  启动桌面。</p>
<p>先看回调对象goingCallback的run函数的工作。</p>
<h5>（1） goingCallback的run函数分析</h5>
<p>[--&gt;SystemServer.java::ServerThread.run]</p>
<div>
    <p>ActivityManagerService.self().systemReady(newRunnable() {</p>
    <p>    publicvoid run() {</p>
    <p>   startSystemUi(contextF);//启动SystemUi</p>
    <p>   //调用其他服务的systemReady函数</p>
    <p>    if(batteryF != null) batteryF.systemReady();</p>
    <p>    if(networkManagementF != null) networkManagementF.systemReady();</p>
    <p>    ......</p>
    <p>    Watchdog.getInstance().start();//启动Watchdog</p>
    <p>    ......//调用其他服务的systemReady函数</p>
    <p> }</p>
</div>
<p>run函数比较简单，执行的工作如下：</p>
<p>·  执行startSystemUi，在该函数内部启动SystemUIService，该Service和状态栏有关。</p>
<p>·  调用一些服务的systemReady函数。</p>
<p>·  启动Watchdog。</p>
<p>startSystemUi的代码如下：</p>
<p>[--&gt;SystemServer.java::startSystemUi]</p>
<div>
    <p>static final void startSystemUi(Context context) {</p>
    <p>     Intentintent = new Intent();</p>
    <p>     intent.setComponent(new ComponentName("com.android.systemui",</p>
    <p>                 "com.android.systemui.SystemUIService"));</p>
    <p>       context.startService(intent);</p>
    <p> }</p>
</div>
<p>SystemUIService由SystemUi.apk提供，它实现了系统的状态栏。</p>
<div>
    <p><strong>注意</strong>在精简ROM时，也不能删除SystemUi.apk。</p>
</div>
<h5>（2）  启动Home界面</h5>
<p>如前所述，resumeTopActivityLocked将启动Home界面，此函数非常重要也比较复杂，故以后再详细分析。我们提取了resumeTopActivityLocked启动Home界面时的相关代码，如下所示：</p>
<p>[--&gt;ActivityStack.java::resumeTopActivityLocked]</p>
<div>
    <p>final booleanresumeTopActivityLocked(ActivityRecord prev) {</p>
    <p>   //找到下一个要启动的Activity</p>
    <p>  ActivityRecord next = topRunningActivityLocked(null);</p>
    <p>   finalboolean userLeaving = mUserLeaving;</p>
    <p>  mUserLeaving = false;</p>
    <p>   if (next== null) {</p>
    <p>       //如果下一个要启动的ActivityRecord为空，则启动Home</p>
    <p>       if(mMainStack) {//全系统就一个ActivityStack，所以mMainStack永远为true</p>
    <p>          //mService指向AMS</p>
    <p>          return mService.startHomeActivityLocked();//mService指向AMS</p>
    <p>        }</p>
    <p>    } </p>
    <p>   ......//以后再详细分析</p>
    <p>}</p>
</div>
<p>下面来看AMS的startHomeActivityLocked函数，代码如下：</p>
<p>[--&gt;ActivityManagerService.java::startHomeActivityLocked]</p>
<div>
    <p>boolean startHomeActivityLocked() {</p>
    <p>   Intentintent = new Intent( mTopAction,</p>
    <p>               mTopData != null ? Uri.parse(mTopData) :null);</p>
    <p>   intent.setComponent(mTopComponent);</p>
    <p>   if(mFactoryTest != SystemServer.FACTORY_TEST_LOW_LEVEL) </p>
    <p>       intent.addCategory(Intent.CATEGORY_HOME);//添加Category为HOME类别</p>
    <p>     </p>
    <p>    //向PKMS查询满足条件的ActivityInfo</p>
    <p>   ActivityInfo aInfo =</p>
    <p>           intent.resolveActivityInfo(mContext.getPackageManager(),</p>
    <p>                                  STOCK_PM_FLAGS);</p>
    <p>    if(aInfo != null) {</p>
    <p>       intent.setComponent(new ComponentName(</p>
    <p>                          aInfo.applicationInfo.packageName,aInfo.name));</p>
    <p>       ProcessRecordapp = getProcessRecordLocked(aInfo.processName,</p>
    <p>                                             aInfo.applicationInfo.uid);</p>
    <p>       //在正常情况下，app应该为null，因为刚开机，Home进程肯定还没启动</p>
    <p>       if(app == null || app.instrumentationClass == null) {</p>
    <p>            intent.setFlags(intent.getFlags()|</p>
    <p>                                 Intent.FLAG_ACTIVITY_NEW_TASK);</p>
    <p>            //启动Home</p>
    <p>            mMainStack.startActivityLocked(null, intent, null, null, 0, aInfo,</p>
    <p>                                  null, null, 0, 0, 0, false, false,null);</p>
    <p>          }</p>
    <p>      }//if(aInfo != null)判断结束</p>
    <p>  return true;</p>
    <p>}</p>
</div>
<p>至此，AMS携诸位Service都启动完毕，Home也靓丽登场，整个系统就准备完毕，只等待用户的检验了。不过在分析逻辑上还有一点没涉及，那会是什么呢？</p>
<h5>（3） 发送ACTION_BOOT_COMPLETED广播</h5>
<p>由前面的代码可知，AMS发送了ACTION_PRE_BOOT_COMPLETED广播，可系统中没有地方处理它。在前面的章节中，还碰到一个ACTION_BOOT_COMPLETED广播，该广播广受欢迎，却不知道它是在哪里发送的。</p>
<p>当Home Activity启动后，ActivityStack的activityIdleInternal函数将被调用，其中有一句代码颇值得注意：</p>
<p>[--&gt;ActivityStack.java::activityIdleInternal]</p>
<div>
    <p>final ActivityRecord activityIdleInternal(IBindertoken, boolean fromTimeout,</p>
    <p>           Configuration config) {</p>
    <p>   booleanbooting = false;</p>
    <p>   ......</p>
    <p>   if(mMainStack) {</p>
    <p>      booting = mService.mBooting; //在systemReady的第三阶段工作中设置该值为true</p>
    <p>      mService.mBooting = false;</p>
    <p>   }</p>
    <p>   ......</p>
    <p>   if(booting)   mService.finishBooting();//调用AMS的finishBooting函数</p>
    <p>}</p>
</div>
<p>[--&gt;ActivityManagerService.java::finishBooting]</p>
<div>
    <p>final void finishBooting() {</p>
    <p>   IntentFilter pkgFilter = new IntentFilter();</p>
    <p>   pkgFilter.addAction(Intent.ACTION_QUERY_PACKAGE_RESTART);</p>
    <p>   pkgFilter.addDataScheme("package");</p>
    <p>   mContext.registerReceiver(new BroadcastReceiver() {</p>
    <p>      public void onReceive(Context context, Intent intent) {</p>
    <p>       ......//处理Package重启的广播</p>
    <p>      }</p>
    <p>    }, pkgFilter);</p>
    <p>        </p>
    <p>  synchronized (this) {</p>
    <p>      finalint NP = mProcessesOnHold.size();</p>
    <p>      if (NP&gt; 0) {</p>
    <p>          ArrayList&lt;ProcessRecord&gt; procs =</p>
    <p>                   new ArrayList&lt;ProcessRecord&gt;(mProcessesOnHold);</p>
    <p>           for(int ip=0; ip&lt;NP; ip++)//启动那些等待启动的进程</p>
    <p>             startProcessLocked(procs.get(ip), "on-hold", null);</p>
    <p>           }</p>
    <p>       if(mFactoryTest != SystemServer.FACTORY_TEST_LOW_LEVEL) {</p>
    <p>          //每15钟检查系统各应用进程使用电量的情况，如果某进程使用WakeLock时间</p>
    <p>          //过长，AMS将关闭该进程</p>
    <p>          Message nmsg = </p>
    <p>                      mHandler.obtainMessage(CHECK_EXCESSIVE_WAKE_LOCKS_MSG);</p>
    <p>          mHandler.sendMessageDelayed(nmsg, POWER_CHECK_DELAY);</p>
    <p>           //设置系统属性sys.boot_completed的值为1</p>
    <p>           SystemProperties.set("sys.boot_completed","1");</p>
    <p>          //发送ACTION_BOOT_COMPLETED广播</p>
    <p>          broadcastIntentLocked(null, null,</p>
    <p>                        newIntent(Intent.ACTION_BOOT_COMPLETED, null),</p>
    <p>                        null, null, 0, null,null,</p>
    <p>                       android.Manifest.permission.RECEIVE_BOOT_COMPLETED,</p>
    <p>                        false, false, MY_PID,Process.SYSTEM_UID);</p>
    <p>           }</p>
    <p>        }</p>
    <p> }</p>
</div>
<p>原来，在Home启动成功后，AMS才发送ACTION_BOOT_COMPLETED广播。</p>
<h4>4.  ASM的 systemReady总结</h4>
<p>systemReady函数完成了系统就绪的必要工作，然后它将启动Home Activity。至此，Android系统就全部启动了。</p>
<h3>6.2.5  初识ActivityManagerService总结</h3>
<p>本节所分析的4个关键函数均较复杂，与之相关的知识点总结如下：</p>
<p>·  AMS的main函数：创建AMS实例，其中最重要的工作是创建Android运行环境，得到一个ActivityThread和一个Context对象。</p>
<p>·  AMS的setSystemProcess函数：该函数注册AMS和meminfo等服务到ServiceManager中。另外，它为SystemServer创建了一个ProcessRecord对象。由于AMS是Java世界的进程管理及调度中心，要做到对Java进程一视同仁，尽管SystemServer贵为系统进程，此时也不得不将其并入AMS的管理范围内。</p>
<p>·  AMS的installSystemProviders：为SystemServer加载SettingsProvider。</p>
<p>·  AMS的systemReady：做系统启动完毕前最后一些扫尾工作。该函数调用完毕后，HomeActivity将呈现在用户面前。</p>
<p>对AMS 调用轨迹分析是我们破解AMS的第一条线，希望读者反复阅读，以真正理解其中涉及的知识点，尤其是和Android运行环境及Context相关的知识。</p>
<h2>6.3  startActivity分析</h2>
<p>本节将重点分析Activity的启动过程，它是五条线中最难分析的一条，只要用心相信读者能啃动这块“硬骨头”。</p>
<h3>6.3.1  从am说起</h3>
<p>am和pm（见4.4.2节）一样，也是一个脚本，它用来和AMS交互，如启动Activity、启动Service、发送广播等。其核心文件在Am.java中，代码如下：</p>
<p>[--&gt;Am.java::main]</p>
<div>
    <p>public static void main(String[] args) {</p>
    <p>   try {</p>
    <p>        (newAm()).run(args);//构造一个Am对象，并调用run函数</p>
    <p>   }......</p>
    <p> }</p>
</div>
<p>am的用法很多，读者可通过adb shell登录手机，然后执行am，这样就能获取它的用法。</p>
<p>利用am启动一个activity的方法如下：</p>
<div>
    <p>am start [-D] [-W] [-P &lt;FILE&gt;][--start-profiler &lt;FILE&gt;] [-S] &lt;INTENT&gt;</p>
</div>
<p>其中：</p>
<p>·  方括号中的参数为可选项，尖括号中的参数为必选项。</p>
<p>·  &lt;INTENT&gt;参数有很多，主要是用于设置Intent的各项参数。</p>
<p>假设已知某个Activity的ComponentName（package名和Activity的Class名），启动这个Activity的相应命令如下：</p>
<div>
    <p>am start -W -n com.dfp.test/.TestActivity</p>
</div>
<p>其中，-W选项表示am将会等目标Activity启动后才返回，-n表示后面的参数用于设置Intent的Component。就本例而言，com.dfp.test为Package名，.TestActivity为该Package下对应的Activity类名，所以将要启动的Activity的全路径名为com.dfp.test.TestActivity。</p>
<p>现在就以上面的命令为例来分析Am的run函数，代码如下：</p>
<p>[--&gt;Am.java::run]</p>
<div>
    <p> privatevoid run(String[] args) throws Exception {</p>
    <p>  mAm =ActivityManagerNative.getDefault();</p>
    <p>  mArgs =args;</p>
    <p>  String op= args[0];</p>
    <p>  mNextArg =1;</p>
    <p>  if (op.equals("start"))  runStart();//用于启动Activity</p>
    <p>  else if ......//处理其他参数</p>
    <p>}</p>
</div>
<p>runStart函数用于处理Activity启动请求，其代码如下：</p>
<p>[--&gt;Am.java::runStart]</p>
<div>
    <p> privatevoid runStart() throws Exception {</p>
    <p>    Intentintent = makeIntent();</p>
    <p> </p>
    <p>    StringmimeType = intent.getType();</p>
    <p>    //获取mimeType，</p>
    <p>    if(mimeType == null &amp;&amp; intent.getData() != null</p>
    <p>         &amp;&amp; "content".equals(intent.getData().getScheme())) {</p>
    <p>        mimeType = mAm.getProviderMimeType(intent.getData());</p>
    <p>    }</p>
    <p> </p>
    <p>    if(mStopOption) {</p>
    <p>     ......//处理-S选项，即先停止对应的Activity，再启动它</p>
    <p>    }</p>
    <p> </p>
    <p>    //FLAG_ACTIVITY_NEW_TASK这个标签很重要</p>
    <p>   intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);</p>
    <p> </p>
    <p>   ParcelFileDescriptor fd = null;</p>
    <p> </p>
    <p>    if(mProfileFile != null) {</p>
    <p>         try{......//处理-P选项，用于性能统计</p>
    <p>            fd = ParcelFileDescriptor.open(......)</p>
    <p>          }......</p>
    <p>    }</p>
    <p> </p>
    <p>   IActivityManager.WaitResult result = null;</p>
    <p>    int res;</p>
    <p>    if(mWaitOption) {//mWaitOption控制是否等待启动结果，如果有-W选项，则该值为true</p>
    <p>               //调用AMS的startActivityAndWait函数</p>
    <p>        result = mAm.startActivityAndWait(null,intent, mimeType,</p>
    <p>                        null, 0, null, null, 0,false, mDebugOption,</p>
    <p>                        mProfileFile, fd,mProfileAutoStop);</p>
    <p>         res= result.result;</p>
    <p>     } ......</p>
    <p>     ......//打印结果</p>
    <p> }</p>
</div>
<p>am最终将调用AMS的startActivityAndWait函数来处理这次启动请求。下面将深入到AMS内部去继续这次旅程。</p>
<div>
    <p><strong>提示</strong>为什么选择从am来分析Activity的启动呢？如果选择从一个Activity来分析如何启动另一个Activity，则将给人一种鸡生蛋、蛋孵鸡的感觉，故此处选择从am入手。除此之外，从am来分析Activity的启动也是Activity启动分析中相对简单的一条路线。</p>
</div>
<h3>6.3.2  AMS的startActivityAndWait函数分析</h3>
<p>startActivityAndWait函数有很多参数，先来认识一下它们。</p>
<p>[--&gt;ActiivtyManagerService.java::startActivityAndWait原型]</p>
<div>
    <p> publicfinal WaitResult startActivityAndWait(</p>
    <p>   /*</p>
    <p>    在绝大多数情况下，一个Acitivity的启动是由一个应用进程发起的，IApplicationThread是</p>
    <p>    应用进程和AMS交互的通道，也可算是调用进程的标示，在本例中，AM并非一个应用进程，所以</p>
    <p>    传递的caller为null</p>
    <p>   */</p>
    <p>  IApplicationThread caller,</p>
    <p>   //Intent及resolvedType，在本例中，resolvedType为null</p>
    <p>   Intentintent, String resolvedType,</p>
    <p>   //下面两个参数和授权有关，读者可参考第3章对CopyboardService分析中介绍的授权知识</p>
    <p>   Uri[] grantedUriPermissions,//在本例中为null</p>
    <p>   intgrantedMode,//在本例中为0</p>
    <p>   IBinder resultTo,//在本例中为null，用于接收startActivityForResult的结果</p>
    <p>   StringresultWho,//在本例中为null</p>
    <p>   //在本例中为0，该值的具体意义由调用者解释。如果该值大于等于0，则AMS内部保存该值，</p>
    <p>   //并通过onActivityResult返回给调用者</p>
    <p>   int requestCode, </p>
    <p>   boolean onlyIfNeeded,//本例为false</p>
    <p>   boolean debug,//是否调试目标进程</p>
    <p>   //下面3个参数和性能统计有关</p>
    <p>   StringprofileFile,</p>
    <p>   ParcelFileDescriptor profileFd, booleanautoStopProfiler)</p>
</div>
<p>关于以上代码中一些参数的具体作用，以后碰到时会再作分析。建议读者先阅读SDK文档中关于Activity类定义的几个函数，如startActivity、startActivityForResult及onActivityResult等。</p>
<p>startActivityAndWait的代码如下：</p>
<p>[--&gt;ActivityManagerService.java::startActivityAndWait]</p>
<div>
    <p> publicfinal WaitResult startActivityAndWait(IApplicationThread caller,</p>
    <p>      Intent intent, String resolvedType, Uri[]grantedUriPermissions,</p>
    <p>      int grantedMode, IBinder resultTo,StringresultWho, int requestCode,</p>
    <p>      booleanonlyIfNeeded, boolean debug,String profileFile,</p>
    <p>      ParcelFileDescriptor profileFd, booleanautoStopProfiler) {</p>
    <p>     //创建WaitResult对象用于存储处理结果</p>
    <p>    WaitResult res = new WaitResult();</p>
    <p>    //mMainStack为ActivityStack类型，调用它的startActivityMayWait函数</p>
    <p>   mMainStack.startActivityMayWait(caller, -1, intent, resolvedType,</p>
    <p>               grantedUriPermissions, grantedMode, resultTo, resultWho,</p>
    <p>               requestCode, onlyIfNeeded, debug, profileFile, profileFd,</p>
    <p>              autoStopProfiler, res, null);//最后一个参数为Configuration，</p>
    <p>    //在本例中为null</p>
    <p>    returnres;</p>
    <p> }</p>
</div>
<p>mMainStack为AMS的成员变量，类型为ActivityStack，该类是Activity调度的核心角色。正式分析它之前，有必要先介绍一下相关的基础知识。</p>
<h4>1.  Task、Back Stack、ActivityStack及Launch mode</h4>
<h5>（1） 关于Task及Back Stack的介绍</h5>
<p>先来看图6-10。</p>

![image](images/chapter6/image010.png)

<p>图6-10  用户想干什么</p>
<p>图6-10列出了用户在Android系统上想干的三件事情，分别用A、B、C表示，将每一件事情称为一个Task。一个Task还可细分为多个子步骤，即Activity。</p>
<div>
    <p><strong>提示</strong>为什么叫Activity？读者可参考Merrian-Webster词典对Activity的解释<a>[③]</a>：“an organizational unit forperforming a specific function”，也就是说，它是一个有组织的单元，用于完成某项指定功能。</p>
</div>
<p>由图6-10可知，A、B两个Task使用了不同的Activity来完成相应的任务。注意，A、B这两个Task的Activity之间没有复用。</p>
<p>再来看C这个Task，它可细分为4个Activity，其中有两个Activity分别使用了A Task的A1、B Task的B2。C Task为什么不新建自己的Activity，反而要用其他Task的呢？这是因为用户想做的事情（即Task）可以完全不同，但是当细分Task为Activity时，就可能出现Activity功能类似的情况。既然A1、B2已能满足要求，为何还要重复“发明轮子”呢？另外，通过重用Activity，也可为用户提供一致的界面和体验。</p>
<p>了解Android设计理念后，我们来看看Android是如何组织Task及它所包含的Activity的。此处有一个简单的例子，如图6-11所示。</p>

![image](images/chapter6/image011.png)

<p>图6-11  Task及Back Stack示例</p>
<p>由图6-11可知：</p>
<p>·  本例中的Task包含4个Activity。用户可单击按钮跳转到下一个Activity。同时，通过返回键可回到上一个Activity。</p>
<p>·  虚线下方是这些Activity的组织方式。Android采用了Stack的方法管理这3个Activity。例如在A1启动A2后，A2入栈成为栈顶成员，A1成为栈底成员，而界面显示的是栈顶成员的内容。当按返回键时，A3出栈，这时候A2成为栈顶，界面显示也相应变成了A2。</p>
<p>以上是一个Task的情况。那么，多个Task又会是何种情况呢？如图6-12所示。</p>

![image](images/chapter6/image012.png)![image](images/chapter6/image012.png)

<p>图6-12  多个Task的情况</p>
<p>由图6-12可知：对多Task的情况来说，系统只支持一个处于前台的Task，即用户当前看到的Activity所属的Task，其余的Task均处于后台，这些后台Task内部的Activity保持顺序不变。用户可以一次将整个Task挪到后台或者置为前台。</p>
<div>
    <p><strong>提示</strong>用过Android手机的读者应该知道，长按Home键，系统会弹出近期Task列表，使用户能快速在多个Task间切换。</p>
</div>
<p>以上内容从抽象角度介绍了什么是Task，以及Android如何分解Task和管理Activity，那么在实际代码中，是如何考虑并设计的呢？</p>
<h5>（2） 关于ActivityStack的介绍</h5>
<p>通过上述分析，我们对Android的设计有了一定了解，那么如何用代码来实现这一设计呢？此处有两点需要考虑：</p>
<p>·  Task内部Activity的组织方式。由图6-11可知，Android通过先入后出的方式来组织Activity。数据结构中的Stack即以这种方式工作。</p>
<p>·  多个Task的组织及管理方式。</p>
<p>Android设计了一个ActivityStack类来负责上述工作，它的组成如图6-13所示。</p>

![image](images/chapter6/image013.png)

<p>图6-13  ActivityStack及相关成员</p>
<p>由图6-13可知：</p>
<p>·  Activity由ActivityRecord表示，Task由TaskRecord表示。ActivityRecord的task成员指向该Activity所在的Task。state变量用于表示该Activity所处的状态（包括INITIALIZING、RESUMED、PAUSED等状态）。</p>
<p>·  ActivityStack用mHistory这个ArrayList保存ActivityRecord。令人大跌眼镜的是，该mHistory保存了系统中所有Task的ActivityRecord，而不是针对某个Task进行保存。</p>
<p>·  ActivityStack的mMainStack成员比较有意思，它代表此ActivityStack是否为主ActivityStack。有主必然有从，但是目前系统中只有一个ActivityStack，并且它的mMainStack为true。从ActivityStack的命名可推测，Android在开发之初也想用ActivityStack来管理单个Task中的ActivityRecord（在ActivityStack.java的注释中说过，该类为“State and management of asingle stack of activities”），但不知何故，在现在的代码实现将所有Task的ActivityRecord都放到mHistory中了，并且依然保留mMainStack。</p>
<p>·  ActivityStack中没有成员用于保存TaskRecord。</p>
<p>由上述内容可知，ActivityStack采用数组的方式保存所有Task的ActivityRecord，并且没有成员保存TaskRecord。这种实现方式有优点亦有缺点：</p>
<p>·  优点是少了TaskRecord一级的管理，直接以ActivityRecord为管理单元。这种做法能降低管理方面的开销。</p>
<p>·  缺点是弱化了Task的概念，结构不够清晰。</p>
<p>下面来看ActivityStack中几个常用的搜索ActivityRecord的函数，代码如下：</p>
<p>[--&gt;ActivityStack.java::topRunningActivityLocked]</p>
<div>
    <p>/* topRunningActivityLocked:</p>
    <p>找到栈中第一个与notTop不同的，并且不处于finishing状态的ActivityRecord。当notTop为</p>
    <p>null时，该函数即返回栈中第一个需要显示的ActivityRecord。提醒读者，栈的出入口只能是栈顶。</p>
    <p>虽然mHistory是一个数组，但是查找均从数组末端开始，所以其行为也粗略符合Stack的定义</p>
    <p>*/</p>
    <p>final ActivityRecordtopRunningActivityLocked(ActivityRecord notTop) {</p>
    <p>    int i =mHistory.size()-1;</p>
    <p>    while (i&gt;= 0) {</p>
    <p>        ActivityRecord r = mHistory.get(i);</p>
    <p>         if (!r.finishing &amp;&amp; r != notTop)  return r;</p>
    <p>         i--;</p>
    <p>     }</p>
    <p>     returnnull;</p>
    <p>  }</p>
</div>
<p>类似的函数还有：</p>
<p>[--&gt;ActivityStack.java::topRunningNonDelayedActivityLocked]</p>
<div>
    <p>/* topRunningNonDelayedActivityLocked</p>
    <p>与topRunningActivityLocked类似，但ActivityRecord要求增加一项，即delayeResume为</p>
    <p>false</p>
    <p>*/</p>
    <p>final ActivityRecordtopRunningNonDelayedActivityLocked(ActivityRecord notTop) {</p>
    <p>     int i =mHistory.size()-1;</p>
    <p>     while(i &gt;= 0) {</p>
    <p>        ActivityRecord r = mHistory.get(i);</p>
    <p>        //delayedResume变量控制是否暂缓resume Activity</p>
    <p>         if (!r.finishing &amp;&amp; !r.delayedResume&amp;&amp; r != notTop)   return r;</p>
    <p>        i--;</p>
    <p>     }</p>
    <p>    returnnull;</p>
    <p>  }</p>
</div>
<p>ActivityStack还提供findActivityLocked函数以根据Intent及ActivityInfo来查找匹配的ActivityRecord，同样，查找也是从mHistory尾端开始，相关代码如下：</p>
<p>[--&gt;ActivityStack.java::findActivityLocked]</p>
<div>
    <p>private ActivityRecord findActivityLocked(Intentintent, ActivityInfo info) {</p>
    <p>  ComponentName cls = intent.getComponent();</p>
    <p>   if(info.targetActivity != null)</p>
    <p>        cls= new ComponentName(info.packageName, info.targetActivity);</p>
    <p>   final intN = mHistory.size();</p>
    <p>   for (inti=(N-1); i&gt;=0; i--) {</p>
    <p>      ActivityRecord r = mHistory.get(i);</p>
    <p>     if (!r.finishing) </p>
    <p>         if (r.intent.getComponent().equals(cls))return r;</p>
    <p>   }</p>
    <p>  return null;</p>
    <p> }</p>
</div>
<p>另一个findTaskLocked函数的返回值是ActivityRecord，其代码如下：</p>
<p>[ActivityStack.java::findTaskLocked]</p>
<div>
    <p>private ActivityRecord findTaskLocked(Intentintent, ActivityInfo info) {</p>
    <p>  ComponentName cls = intent.getComponent();</p>
    <p>   if(info.targetActivity != null)</p>
    <p>        cls= new ComponentName(info.packageName, info.targetActivity);</p>
    <p> </p>
    <p>  TaskRecord cp = null;</p>
    <p> </p>
    <p>   final intN = mHistory.size();</p>
    <p>   for (inti=(N-1); i&gt;=0; i--) {</p>
    <p>       ActivityRecord r = mHistory.get(i);</p>
    <p>       //r.task!=cp，表示不搜索属于同一个Task的ActivityRecord</p>
    <p>        if(!r.finishing &amp;&amp; r.task != cp</p>
    <p>                   &amp;&amp; r.launchMode != ActivityInfo.LAUNCH_SINGLE_INSTANCE) {</p>
    <p>              cp = r.task;</p>
    <p>              //如果Task的affinity相同，则返回这条ActivityRecord</p>
    <p>              if(r.task.affinity != null) {</p>
    <p>               if(r.task.affinity.equals(info.taskAffinity))  return r;</p>
    <p>               }else if (r.task.intent != null </p>
    <p>                        &amp;&amp;r.task.intent.getComponent().equals(cls)) {</p>
    <p>                 //如果Task Intent的ComponentName相同</p>
    <p>                     return r;</p>
    <p>                }else if (r.task.affinityIntent != null </p>
    <p>                        &amp;&amp;r.task.affinityIntent.getComponent().equals(cls)) {</p>
    <p>                   //Task affinityIntent考虑</p>
    <p>                       return r;</p>
    <p>               }//if (r.task.affinity != null)判断结束</p>
    <p>         }//if(!r.finishing &amp;&amp; r.task != cp......)判断结束</p>
    <p>     }//for循环结束</p>
    <p>    returnnull;</p>
    <p> }</p>
</div>
<p>其实，findTaskLocked是根据mHistory中ActivityRecord所属的Task的情况来进行相应的查找工作。</p>
<p>以上这4个函数均是ActivityStack中常用的函数，如果不需要逐项（case by case）地研究AMS，那么读者仅需了解这几个函数的作用即可。 </p>
<h5>（3） 关于Launch Mode的介绍</h5>
<p>Launch Mode用于描述Activity的启动模式，目前一共有4种模式，分别是standard、singleTop、singleTask和singleInstance。初看它们，较难理解，实际上不过是Android玩的一个“小把戏“而已。启动模式就是用于控制Activity和Task关系的。</p>
<p>·  standard：一个Task中可以有多个相同类型的Activity。注意，此处是相同类型的Activity，而不是同一个Activity对象。例如在Task中有A、B、C、D4个Activity，如果再启动A类Activity， Task就会变成A、B、C、D、A。最后一个A和第一个A是同一类型，却并非同一对象。另外，多个Task中也可以有同类型的Activity。</p>
<p>·  singleTop：当某Task中有A、B、C、D4个Activity时，如果D想再启动一个D类型的Activity，那么Task将是什么样子呢？在singleTop模式下，Task中仍然是A、B、C、D，只不过D的onNewIntent函数将被调用以处理这个新Intent，而在standard模式下，则Task将变成A、B、C、D、D，最后的D为新创建的D类型Activity对象。在singleTop这种模式下，只有目标Acitivity当前正好在栈顶时才有效，例如只有处于栈顶的D启动时才有用，如果D启动不处于栈顶的A、B、C等，则无效。</p>
<p>·  singleTask：在这种启动模式下，该Activity只存在一个实例，并且将和一个Task绑定。当需要此Activity时，系统会以onNewIntent方式启动它，而不会新建Task和Activity。注意，该Activity虽只有一个实例，但是在Task中除了它之外，还可以有其他的Activity。</p>
<p>·  singleInstance：它是singleTask的加强版，即一个Task只能有这么一个设置了singleInstance的Activity，不能再有别的Activity。而在singleTask模式中，Task还可以有其他的Activity。</p>
<p>注意，Android建议一般的应用开发者不要轻易使用最后两种启动模式。因为这些模式虽然名意上为Launch Mode，但是它们也会影响Activity出栈的顺序，导致用户按返回键返回时导致不一致的用户体验。</p>
<p>除了启动模式外，Android还有其他一些标志用于控制Activity及Task之间的关系。这里只列举一二，详细信息请参阅SDK文档中Intent的相关说明。</p>
<p>·  FLAG_ACTIVITY_NEW_TASK：将目标Activity放到一个新的Task中。</p>
<p>·  FLAG_ACTIVITY_CLEAR_TASK：当启动一个Activity时，先把和目标Activity有关联的Task“干掉“，然后启动一个新的Task，并把目标Activity放到新的Task中。该标志必须和FLAG_ACTIVITY_NEW_TASK标志一起使用。</p>
<p>·  FLAG_ACTIVITY_CLEAR_TOP：当启动一个不处于栈顶的Activity时候，先把排在它前面的Activity“干掉”。例如Task有A、B、C、D4个Activity，要要启动B，直接把C、D“干掉”，而不是新建一个B。</p>
<div>
    <p><strong>提示</strong>这些启动模式和标志，在笔者看来很像洗扑克牌时的手法，因此可以称之为小把戏。虽是小把戏，但是相关代码的逻辑及分支却异常繁杂，我们应从更高的角度来看待它们。</p>
</div>
<p>介绍完上面的知识后，下面来分析ActivityStack的startActivityMayWait函数。</p>
<h4>2.  ActivityStack的startActivityMayWait函数分析</h4>
<p>startActivityMayWait函数的目标是启动com.dfp.test.TestActivity，假设系统之前没有启动过该Activity，本例最终的结果将是：</p>
<p>·  由于在am中设置了FLAG_ACTIVITY_NEW_TASK标志，因此除了会创建一个新的ActivityRecord外，还会新创建一个TaskRecord。</p>
<p>·  还需要启动一个新的应用进程以加载并运行com.dfp.test.TestActivity的一个实例。</p>
<p>·  如果TestActivity不是Home，还需要停止当前正在显示的Activity。</p>
<p>好了，将这个函数分三部分进行介绍，先来分析第一部分。</p>
<h5>（1） startActivityMayWait分析之一</h5>
<p>[--&gt;ActivityStack.java::startActivityMayWait]</p>
<div>
    <p>final int startActivityMayWait(IApplicationThreadcaller, int callingUid,</p>
    <p>        Intentintent, String resolvedType, Uri[] grantedUriPermissions,</p>
    <p>        intgrantedMode, IBinder resultTo,</p>
    <p>        StringresultWho, int requestCode, boolean onlyIfNeeded,</p>
    <p>        booleandebug, String profileFile, ParcelFileDescriptor profileFd,</p>
    <p>        booleanautoStopProfiler, WaitResult outResult, Configuration config) {</p>
    <p>    ......</p>
    <p>    //在本例中，已经指明了Component，这样可省去为Intent匹配搜索之苦</p>
    <p>    booleancomponentSpecified = intent.getComponent() != null;</p>
    <p> </p>
    <p>    //创建一个新的Intent，防止客户传入的Intent被修改</p>
    <p>    intent =new Intent(intent);</p>
    <p> </p>
    <p>    //查询满足条件的ActivityInfo，在resolveActivity内部和PKMS交互，读者不妨自己</p>
    <p>   //尝试分析该函数</p>
    <p>    ActivityInfoaInfo = resolveActivity(intent, resolvedType, debug,</p>
    <p>                     profileFile, profileFd,autoStopProfiler);</p>
    <p> </p>
    <p>    synchronized(mService) {</p>
    <p>      int callingPid;</p>
    <p>      if (callingUid &gt;= 0) {</p>
    <p>          callingPid= -1;</p>
    <p>      } else if (caller == null) {//本例中，caller为null</p>
    <p>           callingPid= Binder.getCallingPid();//取出调用进程的Pid</p>
    <p>           //取出调用进程的Uid。在本例中，调用进程是am，它由shell启动</p>
    <p>           callingUid= Binder.getCallingUid();</p>
    <p>       } else {</p>
    <p>           callingPid= callingUid = -1;</p>
    <p>       }// if (callingUid &gt;= 0)判断结束</p>
    <p> </p>
    <p>     //在本例中config为null</p>
    <p>     mConfigWillChange= config != null</p>
    <p>                  &amp;&amp; mService.mConfiguration.diff(config) != 0;</p>
    <p>    finallong origId = Binder.clearCallingIdentity();</p>
    <p>    if (mMainStack &amp;&amp; aInfo != null&amp;&amp;  (aInfo.applicationInfo.flags&amp;</p>
    <p>                  ApplicationInfo.FLAG_CANT_SAVE_STATE) != 0){</p>
    <p>        /*</p>
    <p>        AndroidManifest.xml中的Application标签可以声明一个cantSaveState</p>
    <p>        属性，设置了该属性的Application将不享受系统提供的状态保存/恢复功能。</p>
    <p>        当一个Application退到后台时，系统会为它保存状态，当调度其到前台运行时，        将恢复它之前的状态，以保证用户体验的连续性。声明了该属性的Application被称为</p>
    <p>        “heavy weight process”。可惜系统目前不支持该属性，因为PackageParser</p>
    <p>        将不解析该属性。详情请见PackageParser.java parseApplication函数</p>
    <p>       */</p>
    <p>    }</p>
    <p>   ......//待续</p>
</div>
<p>startActivityMayWait第一阶段的工作内容相对较简单：</p>
<p>·  首先需要通过PKMS查找匹配该Intent的ActivityInfo。</p>
<p>·  处理FLAG_CANT_SAVE_STATE的情况，但系统目前不支持此情况。</p>
<p>·  另外，获取调用者的pid和uid。由于本例的caller为null，故所得到的pid和uid均为am所在进程的uid和pid。</p>
<p>下面介绍startActivityMayWait第二阶段的工作。</p>
<h5>（2） startActivityMayWait分析之二</h5>
<p>[--&gt;ActivityStack.java::startActivityMayWait]</p>
<div>
    <p>    //调用此函数启动Activity，将返回值保存到res</p>
    <p>   int res = startActivityLocked(caller, intent,resolvedType,</p>
    <p>            grantedUriPermissions, grantedMode, aInfo,</p>
    <p>            resultTo, resultWho, requestCode, callingPid, callingUid,</p>
    <p>            onlyIfNeeded, componentSpecified, null);</p>
    <p>     //如果configuration发生变化，则调用AMS的updateConfigurationLocked</p>
    <p>    //进行处理。关于这部分内容，读者学完本章后可自行分析</p>
    <p>    if(mConfigWillChange &amp;&amp; mMainStack) {</p>
    <p>            mService.enforceCallingPermission(</p>
    <p>                  android.Manifest.permission.CHANGE_CONFIGURATION,</p>
    <p>                  "updateConfiguration()");</p>
    <p>            mConfigWillChange= false;</p>
    <p>            mService.updateConfigurationLocked(config,null, false);</p>
    <p>   }</p>
</div>
<p>此处，启动Activity的核心函数是startActivityLocked，该函数异常复杂，将用一节专门分析。下面先继续分析startActivityMayWait第三阶段的工作。</p>
<h5>（3） startActivityMayWait分析之三</h5>
<p>[--&gt;ActivityStack.java::startActivityMayWait]</p>
<div>
    <p>    if(outResult != null) {</p>
    <p>      outResult.result = res;//设置启动结果</p>
    <p>       if(res == IActivityManager.START_SUCCESS) {</p>
    <p>          //将该结果加到mWaitingActivityLaunched中保存</p>
    <p>           mWaitingActivityLaunched.add(outResult);</p>
    <p>           do {</p>
    <p>                try {</p>
    <p>                         mService.wait();//等待启动结果</p>
    <p>                       }      </p>
    <p>                } while (!outResult.timeout &amp;&amp; outResult.who == null);</p>
    <p>          }else if (res == IActivityManager.START_TASK_TO_FRONT) {</p>
    <p>              ......//处理START_TASK_TO_FRONT结果，读者可自行分析</p>
    <p>          }</p>
    <p>       }//if(outResult!= null)结束</p>
    <p>           return res;</p>
    <p>     }</p>
    <p> }</p>
</div>
<p>第三阶段的工作就是根据返回值做一些处理，那么res返回成功（即res== IActivityManager.START_SUCCESS的时候）后为何还需要等待呢？</p>
<p>这是因为目标Activity要运行在一个新的应用进程中，就必须等待那个应用进程正常启动并处理相关请求。注意，只有am设置了-W选项，才会进入wait这一状态。</p>
<h3>6.3.3  startActivityLocked分析</h3>
<p>startActivityLocked是startActivityMayWait第二阶段的工作重点，该函数有点长，请读者耐心看代码。</p>
<p>[--&gt;ActivityStack.java::startActivityLocked]</p>
<div>
    <p>final int startActivityLocked(IApplicationThreadcaller,</p>
    <p>           Intent intent, String resolvedType,</p>
    <p>           Uri[] grantedUriPermissions,</p>
    <p>           int grantedMode, ActivityInfo aInfo, IBinder resultTo,</p>
    <p>           String resultWho, int requestCode,</p>
    <p>           int callingPid, int callingUid, boolean onlyIfNeeded,</p>
    <p>             boolean componentSpecified, ActivityRecord[]outActivity) {</p>
    <p> </p>
    <p>   int err = START_SUCCESS;</p>
    <p>  ProcessRecord callerApp = null;</p>
    <p>  //如果caller不为空，则需要从AMS中找到它的ProcessRecord。本例的caller为null</p>
    <p>   if(caller != null) {</p>
    <p>      callerApp = mService.getRecordForAppLocked(caller);</p>
    <p>       //其实就是想得到调用进程的pid和uid</p>
    <p>       if(callerApp != null) {</p>
    <p>           callingPid = callerApp.pid;//一定要保证调用进程的pid和uid正确</p>
    <p>           callingUid = callerApp.info.uid;</p>
    <p>          }else {//如调用进程没有在AMS中注册，则认为其是非法的</p>
    <p>               err = START_PERMISSION_DENIED;</p>
    <p>         }</p>
    <p>     }// if (caller != null)判断结束</p>
    <p> </p>
    <p>   //下面两个变量意义很重要。sourceRecord用于描述启动目标Activity的那个Activity，</p>
    <p> //resultRecord用于描述接收启动结果的Activity，即该Activity的onActivityResult</p>
    <p>  //将被调用以通知启动结果，读者可先阅读SDK中startActivityForResult函数的说明</p>
    <p>   ActivityRecordsourceRecord = null;</p>
    <p>  ActivityRecord resultRecord = null;</p>
    <p>   if(resultTo != null) {</p>
    <p>      //本例resultTo为null， </p>
    <p>   }</p>
    <p>   //获取Intent设置的启动标志，它们是和Launch Mode相类似的“小把戏”，</p>
    <p>  //所以，读者务必理解“关于Launch Mode的介绍”一节的知识点</p>
    <p>   intlaunchFlags = intent.getFlags();</p>
    <p>   if((launchFlags&amp;Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0</p>
    <p>       &amp;&amp; sourceRecord != null) {</p>
    <p>       ......</p>
    <p>       /*</p>
    <p>         前面介绍的Launch Mode和Activity的启动有关，实际上还有一部分标志用于控制</p>
    <p>         Activity启动结果的通知。有关FLAG_ACTIVITY_FORWARD_RESULT的作用，读者可</p>
    <p>         参考SDK中的说明。使用这个标签有个前提，即A必须先存在，正如if中sourceRecord</p>
    <p>         不为null的判断所示。另外，读者自己可尝试编写例子，以测试FLAG_ACTIVITY_FORWARD_</p>
    <p>         RESULT标志的作用</p>
    <p>      */</p>
    <p>    }</p>
    <p>   //检查err值及Intent的情况</p>
    <p>    if (err== START_SUCCESS &amp;&amp; intent.getComponent() == null) </p>
    <p>           err = START_INTENT_NOT_RESOLVED;</p>
    <p>    ......</p>
    <p>    //如果err不为0，则调用sendActivityResultLocked返回错误</p>
    <p>    if (err!= START_SUCCESS) {</p>
    <p>        if(resultRecord != null) {// resultRecord接收启动结果</p>
    <p>          sendActivityResultLocked(-1,esultRecord, resultWho, requestCode,</p>
    <p>            Activity.RESULT_CANCELED, null);</p>
    <p>        }</p>
    <p>       .......</p>
    <p>        returnerr;</p>
    <p>    }</p>
    <p>    //检查权限</p>
    <p>    finalint perm = mService.checkComponentPermission(aInfo.permission,</p>
    <p>            callingPid,callingUid,aInfo.applicationInfo.uid, aInfo.exported);</p>
    <p>    ......//权限检查失败的处理，不必理会</p>
    <p>   if (mMainStack) {</p>
    <p>       //可为AMS设置一个IActivityController类型的监听者，AMS有任何动静都会回调该</p>
    <p>       //监听者。不过谁又有如此本领去监听AMS呢？在进行Monkey测试的时候，Monkey会</p>
    <p>       //设置该回调对象。这样，Monkey就能根据AMS放映的情况进行相应处理了</p>
    <p>        if(mService.mController != null) {</p>
    <p>             boolean abort = false;</p>
    <p>            try {</p>
    <p>                   Intent watchIntent = intent.cloneFilter();</p>
    <p>                   //交给回调对象处理，由它判断是否能继续后面的行程</p>
    <p>                   abort = !mService.mController.activityStarting(watchIntent,</p>
    <p>                           aInfo.applicationInfo.packageName);</p>
    <p>               }......</p>
    <p>            //回调对象决定不启动该Activity。在进行Monkey测试时，可设置黑名单，位于</p>
    <p>            //黑名单中的Activity将不能启动</p>
    <p>             if (abort) {</p>
    <p>                .......//通知resultRecord</p>
    <p>                return START_SUCCESS;</p>
    <p>             }</p>
    <p>           }</p>
    <p>      }// if(mMainStack)判断结束</p>
    <p> </p>
    <p>  //创建一个ActivityRecord对象</p>
    <p>   ActivityRecordr = new ActivityRecord(mService, this, callerApp, callingUid,</p>
    <p>               intent, resolvedType, aInfo, mService.mConfiguration,</p>
    <p>               resultRecord, resultWho, requestCode, componentSpecified);</p>
    <p>   if(outActivity != null)</p>
    <p>        outActivity[0] = r;//保存到输入参数outActivity数组中</p>
    <p>  </p>
    <p>   if(mMainStack) {</p>
    <p>       //mResumedActivity代表当前界面显示的Activity</p>
    <p>       if(mResumedActivity == null </p>
    <p>         || mResumedActivity.info.applicationInfo.uid!= callingUid) {</p>
    <p>           //检查调用进程是否有权限切换Application，相关知识见下文的解释</p>
    <p>           if(!mService.checkAppSwitchAllowedLocked(callingPid, callingUid,</p>
    <p>               "Activity start")) {</p>
    <p>              PendingActivityLaunch pal = new PendingActivityLaunch();</p>
    <p>             //如果调用进程没有权限切换Activity，则只能把这次Activity启动请求保存起来，</p>
    <p>             //后续有机会时再启动它</p>
    <p>              pal.r = r; </p>
    <p>              pal.sourceRecord = sourceRecord;</p>
    <p>              ......</p>
    <p>              //所有Pending的请求均保存到AMS mPendingActivityLaunches变量中</p>
    <p>              mService.mPendingActivityLaunches.add(pal);</p>
    <p>              mDismissKeyguardOnNextActivity = false;</p>
    <p>              return START_SWITCHES_CANCELED;</p>
    <p>           }</p>
    <p>      }//if(mResumedActivity == null...)判断结束</p>
    <p> </p>
    <p>     if (mService.mDidAppSwitch) {//用于控制app switch，见下文解释</p>
    <p>      mService.mAppSwitchesAllowedTime = 0;</p>
    <p>     } else{</p>
    <p>        mService.mDidAppSwitch = true;</p>
    <p>     }</p>
    <p> </p>
    <p>    //启动处于Pending状态的Activity </p>
    <p>    mService.doPendingActivityLaunchesLocked(false);</p>
    <p>   }// if(mMainStack)判断结束</p>
    <p> </p>
    <p>   //调用startActivityUncheckedLocked函数</p>
    <p>   err =startActivityUncheckedLocked(r, sourceRecord,</p>
    <p>            grantedUriPermissions, grantedMode, onlyIfNeeded, true);</p>
    <p>   ......</p>
    <p>   return err;</p>
    <p>}</p>
</div>
<p>startActivityLocked函数的主要工作包括：</p>
<p>·  处理sourceRecord及resultRecord。其中，sourceRecord表示发起本次请求的Activity，resultRecord表示接收处理结果的Activity（启动一个Activity肯定需要它完成某项事情，当目标Activity将事情成后，就需要告知请求者该事情的处理结果）。在一般情况下，sourceRecord和resultRecord应指向同一个Activity。</p>
<p>·  处理app Switch。如果AMS当前禁止app switch，则只能把本次启动请求保存起来，以待允许app switch时再处理。从代码中可知，AMS在处理本次请求前，会先调用doPendingActivityLaunchesLocked函数，在该函数内部将启动之前因系统禁止app switch而保存的Pending请求。</p>
<p>·  调用startActivityUncheckedLocked处理本次Activity启动请求。</p>
<p>先来看app Switch，它虽然是一个小变量，但是意义重大。</p>
<h4>1.  关于resume/stopAppSwitches的介绍</h4>
<p>AMS提供了两个函数，用于暂时（注意，是暂时）禁止App切换。为什么会有这种需求呢？因为当某些重要（例如设置账号等）Activity处于前台（即用户当前所见的Activity）时，不希望系统因用户操作之外的原因而切换Activity（例如恰好此时收到来电信号而弹出来电界面）。</p>
<p>先来看stopAppSwitches，代码如下：</p>
<p>[--&gt;ActivityManagerService.java::stopAppSwitches]</p>
<div>
    <p>public void stopAppSwitches() {</p>
    <p>    ......//检查调用进程是否有STOP_APP_SWITCHES权限</p>
    <p>  synchronized(this) {</p>
    <p>      //设置一个超时时间，过了该时间，AMS可以重新切换App（switch app）了</p>
    <p>     mAppSwitchesAllowedTime = SystemClock.uptimeMillis()</p>
    <p>                   + APP_SWITCH_DELAY_TIME;</p>
    <p>     mDidAppSwitch = false;//设置mDidAppSwitch为false</p>
    <p>     mHandler.removeMessages(DO_PENDING_ACTIVITY_LAUNCHES_MSG);</p>
    <p>     Message msg =//防止应用进程调用了stop却没调用resume，5秒后处理该消息</p>
    <p>           mHandler.obtainMessage(DO_PENDING_ACTIVITY_LAUNCHES_MSG);</p>
    <p>     mHandler.sendMessageDelayed(msg, APP_SWITCH_DELAY_TIME);</p>
    <p>   }</p>
    <p>}</p>
</div>
<p>在以上代码中有两点需要注意：</p>
<p>·  此处控制机制名叫app switch，而不是activity switch。为什么呢？因为如果从受保护的activity中启动另一个activity，那么这个新activity的目的应该是针对同一任务，这次启动就不应该受app switch的制约，反而应该对其大开绿灯。目前，在执行Settings中设置设备策略（DevicePolicy）时就会stopAppSwitch。</p>
<p>·  执行stopAppSwitch后，应用程序应该调resumeAppSwitches以允许app switch，但是为了防止应用程序有意或无意忘记resume app switch，系统设置了一个超时（5秒）消息，过了此超时时间，系统将处理相应的消息，其内部会resume app switch。</p>
<p>再来看resumeAppSwitches函数，代码如下：</p>
<p>[--&gt;ActivityManagerService::resumeAppSwitches]</p>
<div>
    <p> public voidresumeAppSwitches() {</p>
    <p>    ......//检查调用进程是否有STOP_APP_SWITCHES权限</p>
    <p>    synchronized(this) {</p>
    <p>       mAppSwitchesAllowedTime = 0; </p>
    <p>     }</p>
    <p>    //注意，系统并不在此函数内启动那些被阻止的Activity</p>
    <p>}</p>
</div>
<p>在resumeAppSwitches中只设置mAppSwitchesAllowedTime的值为0，它并不处理在stop和resume这段时间内积攒起的Pending请求，那么这些请求是在何时被处理的呢？</p>
<p>·  从前面代码可知，如果在执行resume app switch后，又有新的请求需要处理，则先处理那些pending的请求（调用doPendingActivityLaunchesLocked）。</p>
<p>·  在resumeAppSwitches中并未撤销stopAppSwitches函数中设置的超时消息，所以在处理那条超时消息的过程中，也会处理pending的请求。</p>
<p>在本例中，由于不考虑app switch的情况，那么接下来的工作就是调用startActivityUncheckedLocked函数来处理本次activity的启动请求。此时，我们已经创建了一个ActivityRecord用于保存目标Activity的相关信息。</p>
<h4>2.  startActivityUncheckedLocked函数分析</h4>
<p>startActivityUncheckedLocked函数很长，但是目的比较简单，即为新创建的ActivityRecord找到一个合适的Task。虽然本例最终的结果是创建一个新的Task，但是该函数的处理逻辑却比较复杂。先看第一段分析。</p>
<h5>（1） startActivityUncheckedLocked分析之一</h5>
<p>[--&gt;ActivityStack.java::startActivityUncheckedLocked]</p>
<div>
    <p>final intstartActivityUncheckedLocked(ActivityRecord r,</p>
    <p>   ActivityRecord sourceRecord, Uri[] grantedUriPermissions,</p>
    <p>    intgrantedMode, boolean onlyIfNeeded, boolean doResume) {</p>
    <p>   //在本例中，sourceRecord为null，onlyIfNeeded为false，doResume为true</p>
    <p>   finalIntent intent = r.intent;</p>
    <p>   final intcallingUid = r.launchedFromUid;</p>
    <p>        </p>
    <p>   intlaunchFlags = intent.getFlags();</p>
    <p>   //判断是否需要调用因本次Activity启动而被系统移到后台的当前Activity的</p>
    <p>  //onUserLeaveHint函数。可阅读SDK文档中关于Activity onUserLeaveHint函数的说明</p>
    <p>  mUserLeaving = (launchFlags&amp;Intent.FLAG_ACTIVITY_NO_USER_ACTION) ==0;</p>
    <p> </p>
    <p>   //设置ActivityRecord的delayedResume为true，本例中的doResume为true</p>
    <p>   if (!doResume)   r.delayedResume = true;</p>
    <p> </p>
    <p>    //在本例中，notTop为null</p>
    <p>    ActivityRecord notTop = (launchFlags&amp;Intent.FLAG_ACTIVITY_PREVIOUS_IS_TOP)</p>
    <p>               != 0 ? r : null;</p>
    <p>    if(onlyIfNeeded) {....//在本例中，该变量为false，故略去相关代码</p>
    <p>    }</p>
    <p> </p>
    <p>   //根据sourceRecord的情况进行对应处理，能理解下面这段if/else的判断语句吗</p>
    <p>   if(sourceRecord == null) {</p>
    <p>      //如果请求的发起者为空，则当然需要新建一个Task</p>
    <p>      if((launchFlags&amp;Intent.FLAG_ACTIVITY_NEW_TASK) == 0) </p>
    <p>         launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;</p>
    <p>   }else if (sourceRecord.launchMode ==ActivityInfo.LAUNCH_SINGLE_INSTANCE){</p>
    <p>      //如果sourceRecord单独占一个Instance，则新的Activity必然处于另一个Task中</p>
    <p>     launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;</p>
    <p>   } else if(r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE</p>
    <p>               || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK) {</p>
    <p>      //如果启动模式设置了singleTask或singleInstance，则也要创建新Task</p>
    <p>      launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;</p>
    <p>   }//if(sourceRecord== null)判断结束</p>
    <p> </p>
    <p>   //如果新Activity和接收结果的Activity不在一个Task中，则不能启动新的Activity</p>
    <p>   if(r.resultTo!= null &amp;&amp; (launchFlags&amp;Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {</p>
    <p>      sendActivityResultLocked(-1,r.resultTo, r.resultWho, r.requestCode,</p>
    <p>                                     Activity.RESULT_CANCELED, null);</p>
    <p>      r.resultTo = null;</p>
    <p>   }</p>
</div>
<p>startActivityUncheckedLocked第一阶段的工作还算简单，主要确定是否需要为新的Activity创建一个Task，即是否设置FLAG_ACTIVITY_NEW_TASK标志。</p>
<p>接下来看下一阶段的工作。</p>
<h5>（2） startActivityUncheckedLocked分析之二</h5>
<p>[--&gt;ActivityStack.java::startActivityUncheckedLocked]</p>
<div>
    <p>   booleanaddingToTask = false;</p>
    <p>  TaskRecord reuseTask = null;</p>
    <p>   if(((launchFlags&amp;Intent.FLAG_ACTIVITY_NEW_TASK) != 0 &amp;&amp;</p>
    <p>         (launchFlags&amp;Intent.FLAG_ACTIVITY_MULTIPLE_TASK)== 0)</p>
    <p>               || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK</p>
    <p>               || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {</p>
    <p>          if(r.resultTo == null) {</p>
    <p>              //搜索mHistory，得到一个ActivityRecord</p>
    <p>                ActivityRecord taskTop = r.launchMode !=</p>
    <p>                                   ActivityInfo.LAUNCH_SINGLE_INSTANCE</p>
    <p>                        ?findTaskLocked(intent, r.info)</p>
    <p>                        : findActivityLocked(intent,r.info);</p>
    <p>               if (taskTop != null ){</p>
    <p>                   ......//一堆复杂的逻辑处理，无非就是找到一个合适的Task，然后对应做一些</p>
    <p>                  //处理。此处不讨论这段代码，读者可根据工作中的具体情况进行研究</p>
    <p>               }</p>
    <p>         }//if(r.resultTo == null)判断结束</p>
    <p> }</p>
</div>
<p>在本例中，目标Activity首次登场，所以前面的逻辑处理都没有起作用，建议读者根据具体情况分析该段代码。</p>
<p>下面来看startActivityUncheckLocked第三阶段的工作。</p>
<h5>（3） startActivityUncheckLocked分析之三</h5>
<p>[--&gt;ActivityStack.java::startActivityUncheckLocked]</p>
<div>
    <p>   if(r.packageName != null) {</p>
    <p>        //判断目标Activity是否已经在栈顶，如果是，需要判断是创建一个新的Activity</p>
    <p>        //还是调用onNewIntent（singleTop模式的处理）</p>
    <p>       ActivityRecord top = topRunningNonDelayedActivityLocked(notTop);</p>
    <p>        if (top != null &amp;&amp; r.resultTo == null){</p>
    <p>            ......//不讨论此段代码</p>
    <p>        }//if(top != null...)结束</p>
    <p>    } else {</p>
    <p>         ......//通知错误</p>
    <p>         returnSTART_CLASS_NOT_FOUND;</p>
    <p>   }</p>
    <p>  //在本例中，肯定需要创建一个Task</p>
    <p>   booleannewTask = false;</p>
    <p>   booleankeepCurTransition = false;</p>
    <p>   if(r.resultTo == null &amp;&amp; !addingToTask</p>
    <p>               &amp;&amp; (launchFlags&amp;Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {</p>
    <p>       if(reuseTask == null) {</p>
    <p>          mService.mCurTask++;//AMS中保存了当前Task的数量</p>
    <p>          if (mService.mCurTask &lt;= 0) mService.mCurTask = 1;</p>
    <p>          //为该AactivityRecord设置一个新的TaskRecord</p>
    <p>          r.setTask(new TaskRecord(mService.mCurTask, r.info, intent),</p>
    <p>                                          null,true);</p>
    <p>       }else   r.setTask(reuseTask, reuseTask,true);</p>
    <p> </p>
    <p>      newTask = true;</p>
    <p>      //下面这个函数为Android 4.0新增的，用于处理FLAG_ACTIVITY_TASK_ON_HOME的情况，</p>
    <p>      //请阅读SDK文档对Intent的相关说明</p>
    <p>      moveHomeToFrontFromLaunchLocked(launchFlags);</p>
    <p>   }elseif......//其他处理情况</p>
    <p> </p>
    <p>  //授权控制。在SDK中启动Activity的函数没有授权设置方面的参数。在实际工作中，笔者曾碰</p>
    <p>  //到过一个有趣的情况：在开发的一款定制系统中，用浏览器下载了受DRM保护的图片， </p>
    <p>  //此时要启动Gallery3D来查看该图片，但是由于为DRM目录设置了读写权限，而Gallery3D </p>
    <p>  //并未声明相关权限，结果抛出异常，导致不能浏览该图片。由于startActivity等函数不能设置</p>
    <p>  //授权，最终只能修改Gallery3D并为其添加use-permissions项了</p>
    <p>  if(grantedUriPermissions != null &amp;&amp; callingUid &gt; 0) {</p>
    <p>         for(int i=0; i&lt;grantedUriPermissions.length; i++) {</p>
    <p>            mService.grantUriPermissionLocked(callingUid, r.packageName,</p>
    <p>              grantedUriPermissions[i],grantedMode,</p>
    <p>             r.getUriPermissionsLocked());</p>
    <p>   }</p>
    <p>  mService.grantUriPermissionFromIntentLocked(callingUid, r.packageName,</p>
    <p>               intent, r.getUriPermissionsLocked());</p>
    <p>   //调用startActivityLocked，此时ActivityRecord和TaskRecord均创建完毕</p>
    <p>  startActivityLocked(r, newTask, doResume, keepCurTransition);</p>
    <p>   return START_SUCCESS;</p>
    <p>}//startActivityUncheckLocked函数结束</p>
</div>
<p>startActivityUncheckLocked的第三阶段工作也比较复杂，不过针对本例，它将创建一个新的TaskRecord，并调用startActivityLocked函数进行处理。</p>
<p>下面我们转战startActivityLocked函数。</p>
<h5>（4） startActivityLocked函数分析</h5>
<p>[--&gt;ActivityStack.java::startActivityLocked]</p>
<div>
    <p>private final voidstartActivityLocked(ActivityRecord r, boolean newTask,</p>
    <p>           boolean doResume, boolean keepCurTransition) {</p>
    <p>   final intNH = mHistory.size();</p>
    <p>   intaddPos = -1;</p>
    <p>   if(!newTask){//如果不是新Task，则从mHistory中找到对应的ActivityRecord的位置</p>
    <p>   ......</p>
    <p>   }</p>
    <p>   if(addPos &lt; 0)  addPos = NH; </p>
    <p>   //否则加到mHistory数组的最后</p>
    <p>   mHistory.add(addPos,r);</p>
    <p>   //设置ActivityRecord的inHistory变量为true，表示已经加到mHistory数组中了</p>
    <p>  r.putInHistory();</p>
    <p>  r.frontOfTask = newTask;</p>
    <p>   if (NH&gt; 0) {</p>
    <p>    //判断是否显示Activity切换动画之类的事情，需要与WindowManagerService交互</p>
    <p>  }</p>
    <p>   //最终调用resumeTopActivityLocked</p>
    <p>   if (doResume) resumeTopActivityLocked(null);//重点分析这个函数</p>
    <p> }</p>
</div>
<p>在以上列出的startActivityLocked函数中，略去了一部分逻辑处理，这部分内容和Activity之间的切换动画有关（通过这些动画，使切换过程看起来更加平滑和美观，需和WMS交互）。</p>
<div>
    <p><strong>提示</strong>笔者认为，此处将Activity切换和动画处理这两个逻辑揉到一起并不合适，但是似乎也没有更合适的地方来进行该工作了。读者不妨自行研读一下该段代码以加深体会。</p>
</div>
<h5>（5） startActivityUncheckedLocked总结</h5>
<p>说实话，startActivityUncheckedLocked函数的复杂度超乎笔者的想象，光这些函数名就够让人头疼的。但是针对本例而言，相关逻辑的难度还算适中，毕竟这是Activity启动流程中最简单的情况。可用一句话总结本例中startActivityUncheckedLocked函数的功能：创建ActivityRecord和TaskRecord并将ActivityRecord添加到mHistory末尾，然后调用resumeTopActivityLocked启动它。</p>
<p>下面用一节来分析resumeTopActivityLocked函数。</p>
<h4>3.  resumeTopActivityLocked函数分析</h4>
<p>[--&gt;ActivityStack.java::resumeTopActivityLocked]</p>
<div>
    <p> finalboolean resumeTopActivityLocked(ActivityRecord prev) {</p>
    <p>   //从mHistory中找到第一个需要启动的ActivityRecord</p>
    <p>  ActivityRecord next = topRunningActivityLocked(null);</p>
    <p>   finalboolean userLeaving = mUserLeaving;</p>
    <p>  mUserLeaving = false;</p>
    <p>   if (next== null) {</p>
    <p>      //如果mHistory中没有要启动的Activity，则启动Home</p>
    <p>      if(mMainStack)    returnmService.startHomeActivityLocked();</p>
    <p>   }</p>
    <p>   //在本例中，next将是目标Activity</p>
    <p>   next.delayedResume= false;</p>
    <p>   ......//和WMS交互，略去</p>
    <p>   //将该ActivityRecord从下面几个队列中移除</p>
    <p>   mStoppingActivities.remove(next);</p>
    <p>  mGoingToSleepActivities.remove(next);</p>
    <p>  next.sleeping = false;</p>
    <p>  mWaitingVisibleActivities.remove(next);</p>
    <p>   //如果当前正在中断一个Activity，需先等待那个Activity pause完毕，然后系统会重新</p>
    <p>   //调用resumeTopActivityLocked函数以找到下一个要启动的Activity</p>
    <p>   if(mPausingActivity != null)  return false;</p>
    <p>   /************************<strong>请读者注意</strong>***************************/</p>
    <p>   //①mResumedActivity指向上一次启动的Activity，也就是当前界面显示的这个Activity</p>
    <p>  //在本例中，当前Activity就是Home界面</p>
    <p>   if(mResumedActivity != null) {</p>
    <p>      //先中断 Home。这种情况放到最后进行分析</p>
    <p>      startPausingLocked(userLeaving,false);</p>
    <p>      return true;</p>
    <p>   }</p>
    <p>   //②如果mResumedActivity为空，则一定是系统第一个启动的Activity，读者应能猜测到它就</p>
    <p>   //是Home</p>
    <p>   ......//如果prev不为空，则需要通知WMS进行与Activity切换相关的工作</p>
    <p>   try {</p>
    <p>           //通知PKMS修改该Package stop状态，详细信息参考第4章“readLPw的‘佐料’”</p>
    <p>           //一节的说明</p>
    <p>           AppGlobals.getPackageManager().setPackageStoppedState(</p>
    <p>                   next.packageName, false);</p>
    <p>    }......</p>
    <p>   if(prev!= null){</p>
    <p>   ......//还是和WMS有关，通知它停止绘画</p>
    <p>   }</p>
    <p>  if(next.app != null &amp;&amp; next.app.thread != null) {</p>
    <p>   //如果该ActivityRecord已有对应的进程存在，则只需要重启Activity。就本例而言，</p>
    <p>   //此进程还不存在，所以要先创建一个应用进程</p>
    <p>  }  else {</p>
    <p>           //第一次启动</p>
    <p>           if (!next.hasBeenLaunched) {</p>
    <p>               next.hasBeenLaunched = true;</p>
    <p>           } else {</p>
    <p>               ......//通知WMS显示启动界面</p>
    <p>           }</p>
    <p>       //调用另外一个startSpecificActivityLocked函数</p>
    <p>      startSpecificActivityLocked(next, true, true);</p>
    <p>    }</p>
    <p>    returntrue;</p>
    <p>}</p>
</div>
<p>resumeTopActivityLocked函数中有两个非常重要的关键点：</p>
<p>·  如果mResumedActivity不为空，则需要先暂停（pause）这个Activity。由代码中的注释可知，mResumedActivity代表上一次启动的（即当前正显示的）Activity。现在要启动一个新的Activity，须先停止当前Activity，这部分工作由startPausingLocked函数完成。</p>
<p>·  那么，mResumedActivity什么时候为空呢？当然是在启动全系统第一个Activity时，即启动Home界面的时候。除此之外，该值都不会为空。</p>
<p>先分析第二个关键点，即mResumedActivity为null的情况选择分析此种情况的原因是：如果先分析startPausingLocked，则后续分析会牵扯三个进程，即当前Activity所在进程、AMS所在进程及目标进程，分析的难度相当大。</p>
<p>好了，继续我们的分析。resumeTopActivityLocked最后将调用另外一个startSpecificActivityLocked，该函数将真正创建一个应用进程。</p>
<h5>（1） startSpecificActivityLocked分析</h5>
<p>[--&gt;ActivityStack.java::startSpecificActivityLocked]</p>
<div>
    <p>private final voidstartSpecificActivityLocked(ActivityRecord r,</p>
    <p>           boolean andResume, boolean checkConfig) {</p>
    <p> </p>
    <p>   //从AMS中查询是否已经存在满足要求的进程（根据processName和uid来查找）</p>
    <p>  //在本例中，查询结果应该为null</p>
    <p>  ProcessRecord app = mService.getProcessRecordLocked(r.processName,</p>
    <p>               r.info.applicationInfo.uid);</p>
    <p>   //设置启动时间等信息</p>
    <p>   if(r.launchTime == 0) {</p>
    <p>        r.launchTime = SystemClock.uptimeMillis();</p>
    <p>       if(mInitialStartTime == 0)  mInitialStartTime = r.launchTime;</p>
    <p>   } else if(mInitialStartTime == 0) {</p>
    <p>           mInitialStartTime = SystemClock.uptimeMillis();</p>
    <p>   }</p>
    <p>   //如果该进程存在并已经向AMS注册（例如之前在该进程中启动了其他Activity）</p>
    <p>   if (app!= null &amp;&amp; app.thread != null) {</p>
    <p>       try {</p>
    <p>           app.addPackage(r.info.packageName);</p>
    <p>           //通知该进程中的启动目标Activity</p>
    <p>           realStartActivityLocked(r, app, andResume, checkConfig);</p>
    <p>           return;</p>
    <p>         }......</p>
    <p>    }</p>
    <p>    //如果该进程不存在，则需要调用AMS的startProcessLocked创建一个应用进程</p>
    <p>    mService.startProcessLocked(r.processName, r.info.applicationInfo,</p>
    <p>              true, 0,"activity",r.intent.getComponent(), false);</p>
    <p>}</p>
</div>
<p>来看AMS的startProcessLocked函数，它将创建一个新的应用进程。</p>
<h5>（2） startProcessLocked分析</h5>
<p>[--&gt;ActivityManagerService.java::startProcessLocked]</p>
<div>
    <p>final ProcessRecord startProcessLocked(StringprocessName,</p>
    <p>           ApplicationInfo info, boolean knownToBeDead, int intentFlags,</p>
    <p>           String hostingType, ComponentName hostingName,</p>
    <p>            boolean allowWhileBooting) {</p>
    <p>   //根据processName和uid寻找是否已经存在ProcessRecord</p>
    <p>   ProcessRecordapp = getProcessRecordLocked(processName, info.uid);</p>
    <p>   if (app!= null &amp;&amp; app.pid &gt; 0) {</p>
    <p>       ......//处理相关情况</p>
    <p>   }</p>
    <p> </p>
    <p>    StringhostingNameStr = hostingName != null</p>
    <p>               ? hostingName.flattenToShortString() : null;</p>
    <p>        </p>
    <p>   //①处理FLAG_FROM_BACKGROUND标志，见下文解释</p>
    <p>   if((intentFlags&amp;Intent.FLAG_FROM_BACKGROUND) != 0) {</p>
    <p>       if(mBadProcesses.get(info.processName, info.uid) != null)</p>
    <p>            return null;</p>
    <p>     } else {</p>
    <p>        mProcessCrashTimes.remove(info.processName,info.uid);</p>
    <p>        if(mBadProcesses.get(info.processName, info.uid) != null) {</p>
    <p>            mBadProcesses.remove(info.processName, info.uid);</p>
    <p>            if (app != null)    app.bad =false;</p>
    <p>         }</p>
    <p>   }</p>
    <p>        </p>
    <p>    if (app== null) {</p>
    <p>        //创建一个ProcessRecord，并保存到mProcessNames中。注意，此时还没有创建实际进程</p>
    <p>        app= newProcessRecordLocked(null, info, processName);</p>
    <p>       mProcessNames.put(processName, info.uid, app);</p>
    <p>    }else   app.addPackage(info.packageName);</p>
    <p>       ......</p>
    <p>    //②调用另外一个startProcessLocked函数</p>
    <p>   startProcessLocked(app, hostingType, hostingNameStr);</p>
    <p>     return(app.pid != 0) ? app : null;</p>
    <p> }</p>
</div>
<p>在以上代码中列出两个关键点，其中第一点和FLAG_FROM_BACKGROUND有关，相关知识点如下：</p>
<p>·  FLAG_FROM_BACKGROUND标识发起这次启动的Task属于后台任务。很显然，手机中没有界面供用户操作位于后台Task中的Activity。如果没有设置该标志，那么这次启动请求就是由前台Task因某种原因而触发的（例如用户单击某个按钮）。</p>
<p>·  如果一个应用进程在1分钟内连续崩溃超过2次，则AMS会将其ProcessRecord加入所谓的mBadProcesses中。一个应用崩溃后，系统会弹出一个警告框以提醒用户。但是，如果一个后台Task启动了一个“BadProcess”，然后该Process崩溃，结果弹出一个警告框，那么用户就会觉得很奇怪：“为什么突然弹出一个框？”因此，此处将禁止后台Task启动“Bad Process”。如果用户主动选择启动（例如单击一个按钮），则不能禁止该操作，并且要把应用进程从mBadProcesses中移除，以给它们“重新做人”的机会。当然，要是该应用每次启动时都会崩溃，而且用户不停地去启动，那该用户可能是位测试工作者。</p>
<div>
    <p><strong>提示</strong>这其实是一种安全机制，防止不健全的程序不断启动可能会崩溃的组件，但是这种机制并不限制用户的行为。 </p>
</div>
<p>下面来看第二个关键点，即另一个startProcessLocked函数，其代码如下：</p>
<p>[--&gt;ActivityManagerService.java::startProcessLocked]</p>
<div>
    <p>private final voidstartProcessLocked(ProcessRecord app,</p>
    <p>                            String hostingType, StringhostingNameStr) {</p>
    <p>   if(app.pid &gt; 0 &amp;&amp; app.pid != MY_PID) {</p>
    <p>       synchronized (mPidsSelfLocked) {</p>
    <p>        mPidsSelfLocked.remove(app.pid);</p>
    <p>        mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);</p>
    <p>       }</p>
    <p>      app.pid = 0;</p>
    <p>  }</p>
    <p>   //mProcessesOnHold用于保存那些在系统还没有准备好就提前请求启动的ProcessRecord</p>
    <p>   mProcessesOnHold.remove(app);</p>
    <p>   updateCpuStats(); </p>
    <p> </p>
    <p>  System.arraycopy(mProcDeaths, 0, mProcDeaths, 1, mProcDeaths.length-1);</p>
    <p>   mProcDeaths[0] = 0;</p>
    <p>   </p>
    <p>    try {</p>
    <p>         intuid = app.info.uid;</p>
    <p>        int[] gids = null;</p>
    <p>           try {//从PKMS中查询该进程所属的gid</p>
    <p>               gids = mContext.getPackageManager().getPackageGids(</p>
    <p>                        app.info.packageName);</p>
    <p>           }......</p>
    <p>       ......//工厂测试</p>
    <p> </p>
    <p>      intdebugFlags = 0;</p>
    <p>      if((app.info.flags &amp; ApplicationInfo.FLAG_DEBUGGABLE) != 0) {</p>
    <p>                debugFlags |=Zygote.DEBUG_ENABLE_DEBUGGER;</p>
    <p>              debugFlags |= Zygote.DEBUG_ENABLE_CHECKJNI;</p>
    <p>         }......//设置其他一些debugFlags</p>
    <p> </p>
    <p>     //发送消息给Zygote，它将派生一个子进程，该子进程执行ActivityThread的main函数</p>
    <p>    //注意，我们传递给Zygote的参数并没有包含任何与Activity相关的信息。现在仅仅启动</p>
    <p>    //一个应用进程</p>
    <p>    Process.ProcessStartResult startResult =</p>
    <p>                   Process.start("android.app.ActivityThread",</p>
    <p>                   app.processName, uid, uid, gids, debugFlags,</p>
    <p>                   app.info.targetSdkVersion, null);</p>
    <p>     //电量统计项</p>
    <p>    BatteryStatsImpl bs = app.batteryStats.getBatteryStats();</p>
    <p>    synchronized (bs) {</p>
    <p>          if(bs.isOnBattery()) app.batteryStats.incStartsLocked();</p>
    <p>     }</p>
    <p>      //如果该进程为persisitent，则需要通知Watchdog，实际上processStarted内部只</p>
    <p>      //关心刚才创建的进程是不是com.android.phone</p>
    <p>     if(app.persistent) {</p>
    <p>         Watchdog.getInstance().processStarted(app.processName,</p>
    <p>                                    startResult.pid);</p>
    <p>     }</p>
    <p> </p>
    <p>    app.pid= startResult.pid;</p>
    <p>   app.usingWrapper = startResult.usingWrapper;</p>
    <p>   app.removed = false;</p>
    <p>   synchronized (mPidsSelfLocked) {</p>
    <p>          //以pid为key，将代表该进程的ProcessRecord对象加入到mPidsSelfLocked中保管</p>
    <p>         this.mPidsSelfLocked.put(startResult.pid, app);</p>
    <p>          //发送一个超时消息，如果这个新创建的应用进程10秒内没有和AMS交互，则可断定</p>
    <p>         //该应用进程启动失败</p>
    <p>         Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);</p>
    <p>         msg.obj = app;</p>
    <p>          //正常的超时时间为10秒。不过如果该应用进程通过valgrind加载，则延长到300秒</p>
    <p>        //valgrind是Linux平台上一款检查内存泄露的程序，被加载的应用将在它的环境中工作，</p>
    <p>         //这项工作需耗费较长时间。读者可查询valgrind的用法</p>
    <p>          mHandler.sendMessageDelayed(msg,startResult.usingWrapper</p>
    <p>                        ?PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);</p>
    <p>    }......</p>
    <p> }</p>
</div>
<p>startProcessLocked通过发送消息给Zygote以派生一个应用进程<a>[④]</a>，读者仔细研究所发消息的内容，大概会发现此处并未设置和Activity相关的信息，也就是说，该进程启动后，将完全不知道自己要干什么，怎么办？下面就此进行分析。</p>
<h4>4.  startActivity分析之半程总结</h4>
<p>很抱歉，我们现在还处于startActivity分析之旅的中间点，即使越过了很多险滩恶途，一路走来还是发觉有点艰难。此处用图6-14来记录半程中的各个关键点。</p>

![image](images/chapter6/image014.png)

<p>图6-14  startActivity半程总结</p>
<p>图6-14列出了针对本例的调用顺序，其中对每个函数的大体功能也做了简单描述。</p>
<div>
    <p><strong>注意</strong>图6-14中的调用顺序及功能说明只是针对本例而言的。读者以后可结合具体情况再深入研究其中的内容。</p>
</div>
<h4>5.  应用进程的创建及初始化</h4>
<p>如前所述，应用进程的入口是ActivityThread的main函数，它是在主线程中执行的，其代码如下：</p>
<p>[--&gt;ActivityThread.java::main]</p>
<div>
    <p>public static void main(String[] args) {</p>
    <p>   SamplingProfilerIntegration.start();</p>
    <p>   //和调试及strictMode有关</p>
    <p>  CloseGuard.setEnabled(false);</p>
    <p>   //设置进程名为"&lt;pre-initialized&gt;"</p>
    <p>  Process.setArgV0("&lt;pre-initialized&gt;");</p>
    <p>   //准备主线程消息循环</p>
    <p>  Looper.prepareMainLooper();</p>
    <p>   if(sMainThreadHandler == null) </p>
    <p>      sMainThreadHandler = new Handler();</p>
    <p>   //创建一个ActivityThread对象</p>
    <p>  ActivityThread thread = new ActivityThread();</p>
    <p>   //①调用attach函数，注意其参数值为false</p>
    <p>  thread.attach(false);</p>
    <p>  Looper.loop(); //进入主线程消息循环</p>
    <p>   throw newRuntimeException("Main thread loop unexpectedly exited");</p>
    <p>}</p>
</div>
<p>在main函数内部将创建一个消息循环Loop，接着调用ActivityThread的attach函数，最终将主线程加入消息循环。</p>
<p>我们在分析AMS的setSystemProcess时曾分析过ActivityThread的attach函数，那时传入的参数值为true。现在来看设置其为false的情况：</p>
<p>[--&gt;ActivityThread.java::attach]</p>
<div>
    <p>private void attach(boolean system) {</p>
    <p>  sThreadLocal.set(this);</p>
    <p>  mSystemThread = system;</p>
    <p>   if(!system) {</p>
    <p>      ViewRootImpl.addFirstDrawHandler(new Runnable() {</p>
    <p>          public void run() {</p>
    <p>             ensureJitEnabled();</p>
    <p>           }</p>
    <p>         });</p>
    <p>       //设置在DDMS中看到的本进程的名字为"&lt;pre-initialized&gt;"</p>
    <p>      android.ddm.DdmHandleAppName.setAppName("&lt;pre-initialized&gt;");</p>
    <p>       //设置RuntimeInit的mApplicationObject参数，后续会介绍RuntimeInit类</p>
    <p>      RuntimeInit.setApplicationObject(mAppThread.asBinder());</p>
    <p>       //获取和AMS交互的Binder客户端</p>
    <p>      IActivityManager mgr = ActivityManagerNative.getDefault();</p>
    <p>       try {</p>
    <p>            //①调用AMS的attachApplication，mAppThread为ApplicationThread类型，</p>
    <p>            //它是应用进程和AMS交互的接口</p>
    <p>             mgr.attachApplication(mAppThread);</p>
    <p>         }......</p>
    <p>   } else......// system process的处理</p>
    <p> </p>
    <p>   ViewRootImpl.addConfigCallback(newComponentCallbacks2() </p>
    <p>   {.......//添加回调函数});</p>
    <p>}</p>
</div>
<p>我们知道，AMS创建一个应用进程后，会设置一个超时时间（一般是10秒）。如果超过这个时间，应用进程还没有和AMS交互，则断定该进程创建失败。所以，应用进程启动后，需要尽快和AMS交互，即调用AMS的attachApplication函数。在该函数内部将调用attachApplicationLocked，所以此处直接分析attachApplicationLocked，先看其第一阶段的工作。</p>
<h5>（1） attachApplicationLocked分析之一</h5>
<p>[--&gt;ActivityManagerService.java::attachApplicationLocked]</p>
<div>
    <p>private final booleanattachApplicationLocked(IApplicationThread thread,</p>
    <p>           int pid) {//此pid代表调用进程的pid</p>
    <p>   ProcessRecord app;</p>
    <p>    if (pid != MY_PID &amp;&amp; pid &gt;= 0) {</p>
    <p>        synchronized (mPidsSelfLocked) {</p>
    <p>           app = mPidsSelfLocked.get(pid);//根据pid查找对应的ProcessRecord对象</p>
    <p>        }</p>
    <p>    }else    app = null;</p>
    <p> </p>
    <p>    /*</p>
    <p>    如果该应用进程由AMS启动，则它一定在AMS中有对应的ProcessRecord，读者可回顾前面创建</p>
    <p>    应用进程的代码：AMS先创建了一个ProcessRecord对象，然后才发命令给Zygote。</p>
    <p>    如果此处app为null，表示AMS没有该进程的记录，故需要“杀死”它</p>
    <p>   */</p>
    <p>    if (app== null) {</p>
    <p>       if(pid &gt; 0 &amp;&amp; pid != MY_PID) //如果pid大于零，且不是SystemServer进程，则</p>
    <p>           //Quietly（即不打印任何输出）”杀死”process</p>
    <p>           Process.killProcessQuiet(pid); </p>
    <p>       else{</p>
    <p>           //调用ApplicationThread的scheduleExit函数。应用进程完成处理工作后</p>
    <p>           //将退出运行</p>
    <p>           //为何不像上面一样直接杀死它呢？可查阅linux pid相关的知识并自行解答</p>
    <p>          thread.scheduleExit();</p>
    <p>        }</p>
    <p>      returnfalse;</p>
    <p>   }</p>
    <p>   /*</p>
    <p>     判断app的thread是否为空，如果不为空，则表示该ProcessRecord对象还未和一个</p>
    <p>     应用进程绑定。注意，app是根据pid查找到的，如果旧进程没有被杀死，系统则不会重用</p>
    <p>     该pid。为什么此处会出现ProcessRecord thread不为空的情况呢？见下面代码的注释说明</p>
    <p>   */</p>
    <p>   if(app.thread != null)  handleAppDiedLocked(app, true, true);</p>
    <p>   StringprocessName = app.processName;</p>
    <p>   try {</p>
    <p>          /*</p>
    <p>          创建一个应用进程讣告接收对象。当应用进程退出时，该对象的binderDied将被调</p>
    <p>          用。这样，AMS就能做相应处理。binderDied函数将在另外一个线程中执行，其内部也会</p>
    <p>          调用handleAppDiedLocked。假如用户在binderDied被调用之前又启动一个进程，</p>
    <p>          那么就会出现以上代码中app.thread不为null的情况。这是多线程环境中常出现的</p>
    <p>          情况，不熟悉多线程编程的读者要仔细体会。</p>
    <p>          */</p>
    <p>         AppDeathRecipient adr = new AppDeathRecipient(pp, pid, thread);</p>
    <p>         thread.asBinder().linkToDeath(adr, 0);</p>
    <p>         app.deathRecipient = adr;</p>
    <p>    }......</p>
    <p>   //设置该进程的调度优先级和oom_adj等成员</p>
    <p>   app.thread= thread;</p>
    <p>  app.curAdj = app.setAdj = -100;</p>
    <p>  app.curSchedGroup = Process.THREAD_GROUP_DEFAULT;</p>
    <p>  app.setSchedGroup = Process.THREAD_GROUP_BG_NONINTERACTIVE;</p>
    <p>  app.forcingToForeground = null;</p>
    <p>  app.foregroundServices = false;</p>
    <p>  app.hasShownUi = false;</p>
    <p>  app.debugging = false;</p>
    <p>   //启动成功，从消息队列中撤销PROC_START_TIMEOUT_MSG消息</p>
    <p>  mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);</p>
</div>
<p>attachApplicationLocked第一阶段的工作比较简单：</p>
<p>·  设置代表该应用进程的ProcessRecrod对象的一些成员变量，例如用于和应用进程交互的thread对象、进程调度优先级及oom_adj的值等。</p>
<p>·  从消息队列中撤销PROC_START_TIMEOUT_MSG。</p>
<p>至此，该进程启动成功，但是这一阶段的工作仅针对进程本身（如设置调度优先级，oom_adj等），还没有涉及和Activity启动相关的内容，这部分工作将在第二阶段完成。</p>
<h5>（2） attachApplicationLocked分析之二</h5>
<p>[--&gt;ActivityManagerService.java::attachApplicationLocked]</p>
<div>
    <p>   ......</p>
    <p>   //SystemServer早就启动完毕，所以normalMode为true</p>
    <p>   booleannormalMode = mProcessesReady || isAllowedWhileBooting(app.info);</p>
    <p> </p>
    <p>   /*</p>
    <p>    我们在6.2.3的标题1中分析过generateApplicationProvidersLocked函数，</p>
    <p>    在该函数内部将查询（根据进程名，uid确定）PKMS以获取需运行在该进程中的ContentProvider </p>
    <p>   */</p>
    <p>   Listproviders = normalMode ? generateApplicationProvidersLocked(app) : null;</p>
    <p>   try {</p>
    <p>         int testMode = IApplicationThread.DEBUG_OFF;</p>
    <p>          if(mDebugApp != null &amp;&amp; mDebugApp.equals(processName)) {</p>
    <p>               ......//处理debug选项</p>
    <p>          }</p>
    <p>          ......//处理Profile</p>
    <p> </p>
    <p>          boolean isRestrictedBackupMode = false;</p>
    <p>          ......//</p>
    <p>          //dex化对应的apk包</p>
    <p>          ensurePackageDexOpt(app.instrumentationInfo!= null ?</p>
    <p>                app.instrumentationInfo.packageName : app.info.packageName);</p>
    <p>         //如果设置了Instrumentation类，该类所在的Package也需要dex化</p>
    <p>         if(app.instrumentationClass != null)</p>
    <p>              ensurePackageDexOpt(app.instrumentationClass.getPackageName());</p>
    <p>          </p>
    <p>          ApplicationInfo appInfo =app.instrumentationInfo != null</p>
    <p>                             ? app.instrumentationInfo :app.info;</p>
    <p>         //查询该Application使用的CompatibiliyInfo</p>
    <p>         app.compat =compatibilityInfoForPackageLocked(appInfo);</p>
    <p>         if (profileFd != null) //用于记录性能文件</p>
    <p>               profileFd = profileFd.dup();</p>
    <p> </p>
    <p>           //①通过ApplicationThread和应用进程交互，调用其bindApplication函数</p>
    <p>            thread.bindApplication(processName,appInfo, providers,</p>
    <p>                   app.instrumentationClass, profileFile, profileFd,</p>
    <p>                    profileAutoStop,app.instrumentationArguments,</p>
    <p>                    app.instrumentationWatcher,testMode, </p>
    <p>                   isRestrictedBackupMode || !normalMode, app.persistent,</p>
    <p>                   mConfiguration, app.compat, getCommonServicesLocked(),</p>
    <p>                   mCoreSettingsObserver.getCoreSettingsLocked());</p>
    <p> </p>
    <p>           //updateLruProcessLocked函数以后再作分析</p>
    <p>           updateLruProcessLocked(app,false, true);</p>
    <p>           //记录两个时间</p>
    <p>           app.lastRequestedGc= app.lastLowMemory = SystemClock.uptimeMillis();</p>
    <p>   }......//try结束</p>
    <p> </p>
    <p>..//从mProcessesOnHold和mPersistentStartingProcesses中删除相关信息</p>
    <p>   mPersistentStartingProcesses.remove(app);</p>
    <p>  mProcessesOnHold.remove(app);</p>
</div>
<p>由以上代码可知，第二阶段的工作主要是为调用ApplicationThread的bindApplication做准备，将在后面的章节中分析该函数的具体内容。此处先来看它的原型。</p>
<div>
    <p>/*</p>
    <p>   正如我们在前面分析时提到的，刚创建的这个进程并不知道自己的历史使命是什么，甚至连自己的</p>
    <p>   进程名都不知道，只能设为"&lt;pre-initialized&gt;"。其实，Android应用进程的历史使命是</p>
    <p>   AMS在其启动后才赋予它的，这一点和我们理解的一般意义上的进程不太一样。根据之前的介绍，   Android的组件应该运行在Android运行环境中。从OS角度来说，该运行环境需要和一个进程绑定。</p>
    <p>   所以，创建应用进程这一步只是创建了一个能运行Android运行环境的容器，而我们的工作实际上</p>
    <p>   还远未结束。</p>
    <p>   bindApplication的功能就是创建并初始化位于该进程中的Android运行环境</p>
    <p>*/</p>
    <p>public final void bindApplication(</p>
    <p>       StringprocessName,//进程名，一般是package名</p>
    <p>      ApplicationInfo appInfo,//该进程对应的ApplicationInfo</p>
    <p>       List&lt;ProviderInfo&gt; providers,//在该APackage中声明的Provider信息</p>
    <p>       ComponentName instrumentationName,//和instrumentation有关 </p>
    <p>       //下面3个参数和性能统计有关</p>
    <p>       StringprofileFile, </p>
    <p>      ParcelFileDescriptor profileFd, boolean autoStopProfiler,</p>
    <p>      //这两个和Instrumentation有关，在本例中，这几个参数暂时都没有作用</p>
    <p>      Bundle instrumentationArgs, </p>
    <p>       IInstrumentationWatcherinstrumentationWatcher,</p>
    <p>       intdebugMode,//调试模式</p>
    <p>       boolean isRestrictedBackupMode, </p>
    <p>       boolean persistent,//该进程是否是persist</p>
    <p>       Configuration config,//当前的配置信息，如屏幕大小和语言等</p>
    <p>       CompatibilityInfocompatInfo,//兼容信息</p>
    <p>       //AMS将常用的Service信息传递给应用进程，目前传递的Service信息只有PKMS、</p>
    <p>       //WMS及AlarmManagerService。读者可参看AMS getCommonServicesLocked函数</p>
    <p>       Map&lt;String,IBinder&gt; services,</p>
    <p>       BundlecoreSettings)//核心配置参数，目前仅有“long_press”值</p>
</div>
<p>对bindApplication的原型分析就到此为止，再来看attachApplicationLocked最后一阶段的工作。</p>
<h5>（3） attachApplicationLocked分析之三</h5>
<p>[--&gt;ActivityManagerService.java::attachApplicationLocked]</p>
<div>
    <p>   booleanbadApp = false;</p>
    <p>   booleandidSomething = false;</p>
    <p>   /*</p>
    <p>   至此，应用进程已经准备好了Android运行环境，下面这句调用代码将返回ActivityStack中</p>
    <p>   第一个需要运行的ActivityRecord。由于多线程的原因，难道能保证得到的hr就是我们的目标</p>
    <p>   Activity吗？</p>
    <p>   */</p>
    <p>  ActivityRecord hr = mMainStack.topRunningActivityLocked(null);</p>
    <p>  if (hr !=null &amp;&amp; normalMode) {</p>
    <p>       //需要根据processName和uid等确定该Activity是否运行与目标进程有关</p>
    <p>       if(hr.app == null &amp;&amp; app.info.uid == hr.info.applicationInfo.uid</p>
    <p>            &amp;&amp; processName.equals(hr.processName)) {</p>
    <p>            try {</p>
    <p>               //调用AS的realStartActivityLocked启动该Activity，最后两个参数为true</p>
    <p>                if (mMainStack.realStartActivityLocked(hr, app, true, true)) {</p>
    <p>                      didSomething = true;</p>
    <p>                 }</p>
    <p>            } catch (Exception e) {</p>
    <p>                  badApp = true; //设置badApp为true</p>
    <p>            }</p>
    <p>      } else{</p>
    <p>         //如果hr和目标进程无关，则调用ensureActivitiesVisibleLocked函数处理它</p>
    <p>        mMainStack.ensureActivitiesVisibleLocked(hr, null, processName, 0);</p>
    <p>       }</p>
    <p> }// if (hr!= null &amp;&amp; normalMode)判断结束</p>
    <p>   //mPendingServices存储那些因目标进程还未启动而处于等待状态的ServiceRecord</p>
    <p>   if(!badApp &amp;&amp; mPendingServices.size() &gt; 0) {</p>
    <p>       ServiceRecord sr = null;</p>
    <p>        try{</p>
    <p>           for (int i=0; i&lt;mPendingServices.size(); i++) {</p>
    <p>                sr = mPendingServices.get(i);</p>
    <p>                //和Activity不一样的是，如果Service不属于目标进程，则暂不处理</p>
    <p>                if (app.info.uid != sr.appInfo.uid</p>
    <p>                     ||!processName.equals(sr.processName)) continue;//继续循环</p>
    <p> </p>
    <p>                   //该Service将运行在目标进程中，所以从mPendingService中移除它</p>
    <p>                   mPendingServices.remove(i);</p>
    <p>                   i--;</p>
    <p>                   //处理此service的启动，以后再作分析</p>
    <p>                   realStartServiceLocked(sr, app);</p>
    <p>                   didSomething = true;//设置该值为true</p>
    <p>               }</p>
    <p>           }</p>
    <p>      }......</p>
    <p>   ......//启动等待的BroadcastReceiver</p>
    <p>   ......//启动等待的BackupAgent，相关代码类似Service的启动</p>
    <p>  if(badApp) {</p>
    <p>   //如果以上几个组件启动有错误，则设置badApp为true。此处将调用handleAppDiedLocked</p>
    <p>   //进行处理。该函数我们以后再作分析</p>
    <p>   handleAppDiedLocked(app, false, true);</p>
    <p>    returnfalse;</p>
    <p>  }</p>
    <p>   /*</p>
    <p>   调整进程的oom_adj值。didSomething表示在以上流程中是否启动了Acivity或其他组件。</p>
    <p>   如果启动了任一组件，则didSomething为true。读者以后会知道，这里的启动只是向</p>
    <p>   应用进程发出对应的指令，客户端进程是否成功处理还是未知数。基于这种考虑，所以此处不宜</p>
    <p>   马上调节进程的oom_adj。</p>
    <p>   读者可简单地把oom_adj看做一种优先级。如果一个应用进程没有运行任何组件，那么当内存</p>
    <p>   出现不足时，该进程是最先被系统杀死的。反之，如果一个进程运行的组件越多，那么它就越不易被</p>
    <p>   系统杀死以回收内存。updateOomAdjLocked就是根据该进程中组件的情况对应调节进程的</p>
    <p>   oom_adj值的。</p>
    <p>  */</p>
    <p>   if(!didSomething)  updateOomAdjLocked();</p>
    <p>   returntrue;</p>
    <p> }</p>
</div>
<p>attachApplicationLocked第三阶段的工作就是通知应用进程启动Activity和Service等组件，其中用于启动Activity的函数是ActivityStack realStartActivityLocked。</p>
<p>此处先来分析应用进程的bindApplication，该函数将为应用进程绑定一个Application。</p>
<div>
    <p><strong>提示</strong>还记得AMS中System Context执行的两次init吗？第二次init的功能就是将Context和对应的Application绑定在一起。</p>
</div>
<h5>（4） ApplicationThread的bindApplication分析</h5>
<p>bindApplication在ApplicationThread中的实现，其代码如下：</p>
<p>[--&gt;ActivityThread.java::bindApplication]</p>
<div>
    <p>public final void bindApplication(......) {</p>
    <p> </p>
    <p>   if(services != null)//保存AMS传递过来的系统Service信息</p>
    <p>       ServiceManager.initServiceCache(services);</p>
    <p>    //向主线程消息队列添加SET_CORE_SETTINGS消息</p>
    <p>   setCoreSettings(coreSettings);</p>
    <p>     //创建一个AppBindData对象，其实就是用来存储一些参数</p>
    <p>    AppBindData data = new AppBindData();</p>
    <p>    data.processName = processName;</p>
    <p>    data.appInfo = appInfo;</p>
    <p>    data.providers = providers;</p>
    <p>    data.instrumentationName = instrumentationName;</p>
    <p>    ......//将AMS传过来的参数保存到AppBindData中</p>
    <p>     //向主线程发送H.BIND_APPLICATION消息</p>
    <p>    queueOrSendMessage(H.BIND_APPLICATION, data);</p>
    <p> }</p>
</div>
<p>由以上代码可知，ApplicationThread接收到来自AMS的指令后，均会将指令中的参数封装到一个数据结构中，然后通过发送消息的方式转交给主线程去处理。BIND_APPLICATION最终将由handleBindApplication函数处理。该函数并不复杂，但是其中有些点是值得关注的，这些点主要是初始化应用进程的一些参数。handleBindApplication函数的代码如下：</p>
<p>[--&gt;ActivityThread.java::handleBindApplication]</p>
<div>
    <p>private void handleBindApplication(AppBindDatadata) {</p>
    <p>   mBoundApplication = data;</p>
    <p>   mConfiguration = new Configuration(data.config);</p>
    <p>   mCompatConfiguration = new Configuration(data.config);</p>
    <p>    //初始化性能统计对象</p>
    <p>   mProfiler = new Profiler();</p>
    <p>   mProfiler.profileFile = data.initProfileFile;</p>
    <p>   mProfiler.profileFd = data.initProfileFd;</p>
    <p>   mProfiler.autoStopProfiler = data.initAutoStopProfiler;</p>
    <p> </p>
    <p>    //设置进程名。从此，之前那个默默无名的进程终于有了自己的名字</p>
    <p>   Process.setArgV0(data.processName);</p>
    <p>   android.ddm.DdmHandleAppName.setAppName(data.processName);</p>
    <p> </p>
    <p>    if(data.persistent) {</p>
    <p>       //对于persistent的进程，在低内存设备上，不允许其使用硬件加速显示</p>
    <p>       Display display =</p>
    <p>                 WindowManagerImpl.getDefault().getDefaultDisplay();</p>
    <p>       //当内存大于512MB，或者屏幕尺寸大于1024*600，可以使用硬件加速</p>
    <p>       if(!ActivityManager.isHighEndGfx(display))</p>
    <p>           HardwareRenderer.disable(false);</p>
    <p>     }</p>
    <p>    //启动性能统计</p>
    <p>     if(mProfiler.profileFd != null)   mProfiler.startProfiling();</p>
    <p>   //如果目标SDK版本小于12，则设置AsyncTask使用pool executor，否则使用</p>
    <p>   //serializedexecutor。这些executor涉及Java Concurrent类，对此不熟悉的读者</p>
    <p>   //请自行学习和研究。</p>
    <p>   if(data.appInfo.targetSdkVersion &lt;= 12)</p>
    <p>           AsyncTask.setDefaultExecutor(AsyncTask.THREAD_POOL_EXECUTOR);</p>
    <p> </p>
    <p>    //设置timezone</p>
    <p>   TimeZone.setDefault(null);</p>
    <p>    //设置语言</p>
    <p>    Locale.setDefault(data.config.locale);</p>
    <p>     //设置资源及兼容模式</p>
    <p>    applyConfigurationToResourcesLocked(data.config, data.compatInfo);</p>
    <p>    applyCompatConfiguration();</p>
    <p> </p>
    <p>      //根据传递过来的ApplicationInfo创建一个对应的LoadApk对象</p>
    <p>     data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);</p>
    <p>     //对于系统APK，如果当前系统为userdebug/eng版，则需要记录log信息到dropbox的日志记录</p>
    <p>     if((data.appInfo.flags &amp;</p>
    <p>            (ApplicationInfo.FLAG_SYSTEM |</p>
    <p>             ApplicationInfo.FLAG_UPDATED_SYSTEM_APP)) != 0) {</p>
    <p>           StrictMode.conditionallyEnableDebugLogging();</p>
    <p>    }</p>
    <p>    /*</p>
    <p>    如目标SDK版本大于9，则不允许在主线程使用网络操作（如Socketconnect等），否则抛出</p>
    <p>    NetworkOnMainThreadException，这么做的目的是防止应用程序在主线程中因网络操作执行</p>
    <p>    时间过长而造成用户体验下降。说实话，没有必要进行这种限制，在主线程中是否网络操作</p>
    <p>    是应用的事情。再说，Socket也可作为进程间通信的手段，在这种情况下，网络操作耗时很短。</p>
    <p>    作为系统，不应该设置这种限制。另外，Goolge可以提供一些开发指南或规范来指导开发者，</p>
    <p>    而不应如此蛮横地强加限制。</p>
    <p>    */</p>
    <p>    if (data.appInfo.targetSdkVersion&gt; 9)</p>
    <p>        StrictMode.enableDeathOnNetwork();</p>
    <p> </p>
    <p>    //如果没有设置屏幕密度，则为Bitmap设置默认的屏幕密度</p>
    <p>   if((data.appInfo.flags</p>
    <p>               &amp;ApplicationInfo.FLAG_SUPPORTS_SCREEN_DENSITIES) == 0) </p>
    <p>    Bitmap.setDefaultDensity(DisplayMetrics.DENSITY_DEFAULT);</p>
    <p>   if(data.debugMode != IApplicationThread.DEBUG_OFF){</p>
    <p>    ......//调试模式相关处理</p>
    <p>   }</p>
    <p>   IBinder b= ServiceManager.getService(Context.CONNECTIVITY_SERVICE);</p>
    <p>  IConnectivityManager service =</p>
    <p>                        IConnectivityManager.Stub.asInterface(b);</p>
    <p>  try {</p>
    <p>       //设置Http代理信息</p>
    <p>       ProxyPropertiesproxyProperties = service.getProxy();</p>
    <p>      Proxy.setHttpProxySystemProperty(proxyProperties);</p>
    <p>   } catch(RemoteException e) {}</p>
    <p> </p>
    <p>   if(data.instrumentationName != null){</p>
    <p>        //在正常情况下，此条件不满足</p>
    <p>   } else {</p>
    <p>      //创建Instrumentation对象，在正常情况都再这个条件下执行</p>
    <p>     mInstrumentation = new Instrumentation();</p>
    <p>   }</p>
    <p>   //如果Package中声明了FLAG_LARGE_HEAP，则可跳过虚拟机的内存限制，放心使用内存</p>
    <p>   if((data.appInfo.flags&amp;ApplicationInfo.FLAG_LARGE_HEAP) != 0)</p>
    <p>        dalvik.system.VMRuntime.getRuntime().clearGrowthLimit();</p>
    <p> </p>
    <p>   //创建一个Application，data.info为LoadedApk类型，在其内部会通过Java反射机制</p>
    <p>   //创建一个在该APK AndroidManifest.xml中声明的Application对象</p>
    <p>   Applicationapp = data.info.makeApplication(</p>
    <p>                                     data.restrictedBackupMode, null);</p>
    <p> //mInitialApplication保存该进程中第一个创建的Application</p>
    <p>  mInitialApplication = app;</p>
    <p> </p>
    <p>   //安装本Package中携带的ContentProvider</p>
    <p>   if(!data.restrictedBackupMode){ </p>
    <p>      List&lt;ProviderInfo&gt; providers = data.providers;</p>
    <p>        if(providers != null) {</p>
    <p>               //installContentProviders我们已经分析过了</p>
    <p>               installContentProviders(app, providers);</p>
    <p>               mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);</p>
    <p>           }</p>
    <p>  }</p>
    <p>   //调用Application的onCreate函数，做一些初始工作</p>
    <p>  mInstrumentation.callApplicationOnCreate(app);</p>
    <p>}</p>
</div>
<p>由以上代码可知，bindApplication函数将设置一些初始化参数，其中最重要的有：</p>
<p>·  创建一个Application对象，该对象是本进程中运行的第一个Application。</p>
<p>·  如果该Application有ContentProvider，则应安装它们。</p>
<div>
    <p><strong>提示</strong>从以上代码可知，ContentProvider的创建就在bindApplication函数中，其时机早于其他组件的创建。</p>
</div>
<h5>（5） 应用进程的创建及初始化总结</h5>
<p>本节从应用进程的入口函数main开始，分析了应用进程和AMS之间的两次重要交互，它们分别是：</p>
<p>·  在应用进程启动后，需要尽快调用AMS的attachApplication函数，该函数是这个刚呱呱坠地的应用进程第一次和AMS交互。此时的它还默默“无名”，连一个确定的进程名都没有。不过没关系，attachApplication函数将根据创建该应用进程之前所保存的ProcessRecord为其准备一切“手续”。</p>
<p>·  attachApplication准备好一切后，将调用应用进程的bindApplication函数，在该函数内部将发消息给主线程，最终该消息由handleBindApplication处理。handleBindApplication将为该进程设置进程名，初始化一些策略和参数信息等。另外，它还创建一个Application对象。同时，如果该Application声明了ContentProvider，还需要为该进程安装ContentProvider。</p>
<div>
    <p><strong>提示</strong>这个流程有点类似生孩子，一般生之前需要到医院去登记，生完后又需去注册户口，如此这般，这个孩子才会在社会有合法的身份。</p>
</div>
<h4>6.  ActivityStack realStartActivityLocked分析</h4>
<p>如前所述，AMS调用完bindApplication后，将通过realStartActivityLocked启动Activity。在此之前，要创建完应用进程并初始化Android运行环境（除此之外，连ContentProvider都安装好了）。</p>
<p>[--&gt;ActivityStack.java::realStartActivityLocked]</p>
<div>
    <p>//注意，在本例中该函数的最后两个参数的值都为true</p>
    <p>final booleanrealStartActivityLocked(ActivityRecord r, ProcessRecord app,</p>
    <p>       boolean andResume, boolean checkConfig)   throws RemoteException {</p>
    <p> </p>
    <p>   r.startFreezingScreenLocked(app, 0);</p>
    <p>    mService.mWindowManager.setAppVisibility(r,true);</p>
    <p> </p>
    <p>    if(checkConfig) {</p>
    <p>       ......//处理Config发生变化的情况</p>
    <p>      mService.updateConfigurationLocked(config, r, false);</p>
    <p>    }</p>
    <p> </p>
    <p>    r.app =app;</p>
    <p>   app.waitingToKill = null;</p>
    <p>    //将ActivityRecord加到ProcessRecord的activities中保存</p>
    <p>    int idx= app.activities.indexOf(r);</p>
    <p>    if (idx&lt; 0)   app.activities.add(r);</p>
    <p>    //更新进程的调度优先级等，以后再分析该函数</p>
    <p>    mService.updateLruProcessLocked(app, true, true);</p>
    <p> </p>
    <p>     try {</p>
    <p>        List&lt;ResultInfo&gt; results = null;</p>
    <p>        List&lt;Intent&gt; newIntents = null;</p>
    <p>         if(andResume) {</p>
    <p>               results = r.results;</p>
    <p>               newIntents = r.newIntents;</p>
    <p>           }</p>
    <p>          if(r.isHomeActivity)  mService.mHomeProcess = app;</p>
    <p>        //看看是否有dex对应Package的需要</p>
    <p>         mService.ensurePackageDexOpt(</p>
    <p>                   r.intent.getComponent().getPackageName());</p>
    <p>        r.sleeping = false;</p>
    <p>        r.forceNewConfig = false;</p>
    <p>       ......</p>
    <p>          //①通知应用进程启动Activity</p>
    <p>           app.thread. scheduleLaunchActivity (new Intent(r.intent), r,</p>
    <p>                   System.identityHashCode(r), r.info, mService.mConfiguration,</p>
    <p>                   r.compat, r.icicle, results, newIntents, !andResume,</p>
    <p>                   mService.isNextTransitionForward(), profileFile, profileFd,</p>
    <p>                    profileAutoStop);</p>
    <p>            </p>
    <p>           if ((app.info.flags&amp;ApplicationInfo.FLAG_CANT_SAVE_STATE) != 0) {</p>
    <p>               ......//处理heavy-weight的情况</p>
    <p>               }</p>
    <p>           }</p>
    <p>     } ......//try结束</p>
    <p> </p>
    <p>    r.launchFailed = false;</p>
    <p>     ......</p>
    <p>     if(andResume) {</p>
    <p>         r.state = ActivityState.RESUMED;</p>
    <p>         r.stopped = false;</p>
    <p>        mResumedActivity = r;//设置mResumedActivity为目标Activity</p>
    <p>        r.task.touchActiveTime();</p>
    <p>         //添加该任务到近期任务列表中</p>
    <p>         if(mMainStack)  mService.addRecentTaskLocked(r.task);</p>
    <p>         //②关键函数，见下文分析</p>
    <p>        completeResumeLocked(r);</p>
    <p>         //如果在这些过程中，用户按了Power键，怎么办？</p>
    <p>        checkReadyForSleepLocked();</p>
    <p>        r.icicle = null;</p>
    <p>        r.haveState = false;</p>
    <p>    }......</p>
    <p>    //启动系统设置向导Activity，当系统更新或初次使用时需要进行配置</p>
    <p>    if(mMainStack)  mService.startSetupActivityLocked();</p>
    <p>    returntrue;</p>
    <p> }</p>
</div>
<p>在以上代码中有两个关键函数，分别是：scheduleLaunchActivity和completeResumeLocked。其中，scheduleLaunchActivity用于和应用进程交互，通知它启动目标Activity。而completeResumeLocked将继续AMS的处理流程。先来看第一个关键函数。</p>
<h5>（1） scheduleLaunchActivity函数分析</h5>
<p>[--&gt;ActivityThread.java::scheduleLaunchActivity]</p>
<div>
    <p>public final void scheduleLaunchActivity(Intentintent, IBinder token, int ident,</p>
    <p>     ActivityInfo info, Configuration curConfig,CompatibilityInfo compatInfo,</p>
    <p>     Bundlestate, List&lt;ResultInfo&gt; pendingResults,</p>
    <p>    List&lt;Intent&gt; pendingNewIntents, boolean notResumed, booleanisForward,</p>
    <p>     StringprofileName, ParcelFileDescriptor profileFd, </p>
    <p>     booleanautoStopProfiler) {</p>
    <p>  ActivityClientRecord r = new ActivityClientRecord();</p>
    <p>   ......//保存AMS发送过来的参数信息</p>
    <p>   //向主线程发送消息，该消息的处理在handleLaunchActivity中进行</p>
    <p>  queueOrSendMessage(H.LAUNCH_ACTIVITY, r);</p>
    <p> }</p>
</div>
<p>[--&gt;ActivityThread.java::handleMessage]</p>
<div>
    <p>public void handleMessage(Message msg) {</p>
    <p>  switch(msg.what) {</p>
    <p>       case LAUNCH_ACTIVITY: {</p>
    <p>        ActivityClientRecord r = (ActivityClientRecord)msg.obj;</p>
    <p>        //根据ApplicationInfo得到对应的PackageInfo</p>
    <p>        r.packageInfo = getPackageInfoNoCheck(</p>
    <p>                           r.activityInfo.applicationInfo, r.compatInfo);</p>
    <p>         //调用handleLaunchActivity处理</p>
    <p>        handleLaunchActivity(r, null);</p>
    <p>       }break;</p>
    <p>......</p>
    <p>}</p>
</div>
<p>[--&gt;ActivityThread.java::handleLaunchActivity]</p>
<div>
    <p>private voidhandleLaunchActivity(ActivityClientRecord r, </p>
    <p>                             Intent customIntent){</p>
    <p>   unscheduleGcIdler();</p>
    <p> </p>
    <p>   if (r.profileFd != null) {......//略去}</p>
    <p> </p>
    <p>  handleConfigurationChanged(null, null);</p>
    <p>   /*</p>
    <p>   ①创建Activity，通过Java反射机制创建目标Activity，将在内部完成Activity生命周期</p>
    <p>   的前两步，即调用其onCreate和onStart函数。至此，我们的目标com.dfp.test.TestActivity</p>
    <p>   创建完毕</p>
    <p>   */</p>
    <p>   Activitya = performLaunchActivity(r, customIntent);</p>
    <p>   if (a !=null) {</p>
    <p>     r.createdConfig = new Configuration(mConfiguration);</p>
    <p>      BundleoldState = r.state;</p>
    <p>      //②调用handleResumeActivity，其内部有个关键点，见下文分析</p>
    <p>     handleResumeActivity(r.token, false, r.isForward);</p>
    <p>      if(!r.activity.mFinished &amp;&amp; r.startsNotResumed) {</p>
    <p>         .......//</p>
    <p>.       r.paused = true;</p>
    <p>       }else {</p>
    <p>            //如果启动错误，通知AMS</p>
    <p>           ActivityManagerNative.getDefault()</p>
    <p>                    .finishActivity(r.token,Activity.RESULT_CANCELED, null);</p>
    <p>      }</p>
    <p>  }</p>
</div>
<p>handleLaunchActivity的工作包括：</p>
<p>·  首先调用performLaunchActivity，该在函数内部通过Java反射机制创建目标Activity，然后调用它的onCreate及onStart函数。</p>
<p>·  调用handleResumeActivity，会在其内部调用目标Activity的onResume函数。除此之外，handleResumeActivity还完成了一件很重要的事情，见下面的代码：</p>
<p>[--&gt;ActivityThread.java::handleResumeActivity]</p>
<div>
    <p>final void handleResumeActivity(IBinder token,boolean clearHide, </p>
    <p>                                     booleanisForward) {</p>
    <p> unscheduleGcIdler();</p>
    <p>  //内部调用目标Activity的onResume函数</p>
    <p> ActivityClientRecord r = performResumeActivity(token, clearHide);</p>
    <p> </p>
    <p>  if (r !=null) {</p>
    <p>   finalActivity a = r.activity;</p>
    <p> </p>
    <p>   final int forwardBit = isForward ?</p>
    <p>          WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;</p>
    <p>   ......</p>
    <p>   if(!r.onlyLocalRequest) {</p>
    <p>       //将上面完成onResume的Activity保存到mNewActivities中</p>
    <p>      r.nextIdle = mNewActivities;</p>
    <p>      mNewActivities = r;</p>
    <p>      //①向消息队列中添加一个Idler对象</p>
    <p>     Looper.myQueue().addIdleHandler(new Idler());</p>
    <p>   }</p>
    <p>   r.onlyLocalRequest = false;</p>
    <p>   ......</p>
    <p> }</p>
</div>
<p>根据第2章对MessageQueue的分析，当消息队列中没有其他要处理的消息时，将处理以上代码中通过addIdleHandler添加的Idler对象，也就是说，Idler对象的优先级最低，这是不是说它的工作不重要呢？非也。至少在handleResumeActivity函数中添加的这个Idler并不不简单，其代码如下：</p>
<p>[--&gt;ActivityThread.java::Idler]</p>
<div>
    <p>private class Idler implements MessageQueue.IdleHandler{</p>
    <p>    publicfinal boolean queueIdle() {</p>
    <p>   ActivityClientRecord a = mNewActivities;</p>
    <p>    booleanstopProfiling = false;</p>
    <p>    ......</p>
    <p>   if (a !=null) {</p>
    <p>    mNewActivities = null;</p>
    <p>    IActivityManager am = ActivityManagerNative.getDefault();</p>
    <p>    ActivityClientRecord prev;</p>
    <p>    do {</p>
    <p>       if(a.activity != null &amp;&amp; !a.activity.mFinished) {</p>
    <p>          //调用AMS的activityIdle</p>
    <p>          am.activityIdle(a.token, a.createdConfig, stopProfiling);</p>
    <p>          a.createdConfig = null;</p>
    <p>      }</p>
    <p>     prev =a;</p>
    <p>     a =a.nextIdle;</p>
    <p>    prev.nextIdle = null;</p>
    <p>    } while(a != null); //do循环结束</p>
    <p>  }//if(a!=null)判断结束</p>
    <p>   ......</p>
    <p>   ensureJitEnabled();</p>
    <p>    returnfalse;</p>
    <p>   }// queueIdle函数结束</p>
    <p> }</p>
</div>
<p>由以上代码可知，Idler将为那些已经完成onResume的Activity调用AMS的activityIdle函数。该函数是Activity成功创建并启动的流程中与AMS交互的最后一步。虽然对应用进程来说，Idler处理的优先级最低，但AMS似乎不这么认为，因为它还设置了超时等待，以处理应用进程没有及时调用activityIdle的情况。这个超时等待即由realStartActivityLocked中最后一个关键点completeResumeLocked函数设置。</p>
<h5>（2） completeResumeLocked函数分析</h5>
<p>[--&gt;ActivityStack.java::completeResumeLocked]</p>
<div>
    <p>private final voidcompleteResumeLocked(ActivityRecord next) {</p>
    <p>   next.idle = false;</p>
    <p>   next.results = null;</p>
    <p>   next.newIntents = null;</p>
    <p>   //发送一个超时处理消息，默认为10秒。IDLE_TIMEOUT_MSG就是针对acitivityIdle函数的</p>
    <p>    Messagemsg = mHandler.obtainMessage(IDLE_TIMEOUT_MSG);</p>
    <p>    msg.obj= next;</p>
    <p>   mHandler.sendMessageDelayed(msg, IDLE_TIMEOUT);</p>
    <p>     //通知AMS </p>
    <p>     if(mMainStack)   mService.reportResumedActivityLocked(next);</p>
    <p>     ......//略去其他逻辑的代码</p>
    <p>}</p>
</div>
<p>由以上代码可知，AMS给了应用进程10秒的时间，希望它在10秒内调用activityIdle函数。这个时间不算长，和前面AMS等待应用进程启动的超时时间一样。所以，笔者有些困惑，为什么要把这么重要的操作放到idler中去做。</p>
<p>下面来看activityIdle函数，在其内部将调用ActivityStack activityIdleInternal。</p>
<h5>（3） activityIdleInternal函数分析</h5>
<p>[--&gt;ActivityStack.java::activityIdleInternal]</p>
<div>
    <p>final ActivityRecord activityIdleInternal(IBindertoken, boolean fromTimeout,</p>
    <p>                                      Configuration config) {</p>
    <p>   /*</p>
    <p>   如果应用进程在超时时间内调用了activityIdleInternal函数，则fromTimeout为false</p>
    <p>   否则，一旦超时，在IDLE_TIMEOUT_MSG的消息处理中也会调用该函数，并设置fromTimeout</p>
    <p>   为true</p>
    <p>   */</p>
    <p>   ActivityRecord res = null;</p>
    <p> </p>
    <p>  ArrayList&lt;ActivityRecord&gt; stops = null;</p>
    <p>  ArrayList&lt;ActivityRecord&gt; finishes = null;</p>
    <p>  ArrayList&lt;ActivityRecord&gt; thumbnails = null;</p>
    <p>   int NS =0;</p>
    <p>   int NF =0;</p>
    <p>   int NT =0;</p>
    <p>  IApplicationThread sendThumbnail = null;</p>
    <p>   booleanbooting = false;</p>
    <p>   booleanenableScreen = false;</p>
    <p> </p>
    <p>  synchronized (mService) {</p>
    <p>     //从消息队列中撤销IDLE_TIMEOUT_MSG</p>
    <p>     if(token != null)   mHandler.removeMessages(IDLE_TIMEOUT_MSG, token);</p>
    <p>      int index= indexOfTokenLocked(token);</p>
    <p>      if(index &gt;= 0) {</p>
    <p>        ActivityRecord r = mHistory.get(index);</p>
    <p>         res =r;</p>
    <p>        //注意，只有fromTimeout为true，才会走执行下面的条件语句</p>
    <p>        if(fromTimeout) reportActivityLaunchedLocked(fromTimeout, r, -1, -1);</p>
    <p>        if(config != null)  r.configuration =config;</p>
    <p>        /*</p>
    <p>        mLaunchingActivity是一个WakeLock，它能防止在操作Activity过程中掉电，同时</p>
    <p>        这个WakeLock又不能长时间使用，否则有可能耗费过多电量。所以，系统设置了一个超时</p>
    <p>        处理消息LAUNCH_TIMEOUT_MSG，超时时间为10秒。一旦目标Activity启动成功，</p>
    <p>        就需要需要释放 WakeLock</p>
    <p>        */</p>
    <p>        if(mResumedActivity == r &amp;&amp; mLaunchingActivity.isHeld()) {</p>
    <p>            mHandler.removeMessages(LAUNCH_TIMEOUT_MSG);</p>
    <p>            mLaunchingActivity.release();</p>
    <p>         }</p>
    <p>       r.idle = true;</p>
    <p>       mService.scheduleAppGcsLocked();</p>
    <p>       ......</p>
    <p>       ensureActivitiesVisibleLocked(null, 0);</p>
    <p>     if(mMainStack) {</p>
    <p>         if(!mService.mBooted) {</p>
    <p>            mService.mBooted = true;</p>
    <p>             enableScreen = true;</p>
    <p>          }</p>
    <p>       }//if (mMainStack)判断结束</p>
    <p>   } else if(fromTimeout) {//注意，只有fromTimeout为true，才会走下面的case</p>
    <p>       reportActivityLaunchedLocked(fromTimeout, null, -1, -1);</p>
    <p>  }</p>
    <p>  /*</p>
    <p>   ①processStoppingActivitiesLocked函数返回那些因本次Activity启动而</p>
    <p>   被暂停（paused）的Activity</p>
    <p>  */</p>
    <p>  stops =processStoppingActivitiesLocked(true);</p>
    <p> ......</p>
    <p>   for (i=0;i&lt;NS; i++) {</p>
    <p>      ActivityRecord r = (ActivityRecord)stops.get(i);</p>
    <p>       synchronized (mService) {</p>
    <p>      //如果这些Acitivity 处于finishing状态，则通知它们执行Destroy操作，最终它们</p>
    <p>     //的onDestroy函数会被调用</p>
    <p>       if(r.finishing) finishCurrentActivityLocked(r, FINISH_IMMEDIATELY);</p>
    <p>       else  //否则将通知它们执行stop操作，最终Activity的onStop被调用</p>
    <p>         stopActivityLocked(r); </p>
    <p>       }//synchronized结束</p>
    <p>  }//for循环结束</p>
    <p> </p>
    <p>   ......//处理等待结束的Activities</p>
    <p> </p>
    <p>   //发送ACTION_BOOT_COMPLETED广播</p>
    <p>   if(booting)  mService.finishBooting();</p>
    <p>   ......</p>
    <p>   returnres;</p>
    <p> }</p>
</div>
<p>在activityIdleInternal中有一个非常重要的关键点，即处理那些因为本次Activity启动而被暂停的Activity。有两种情况需考虑：</p>
<p>·  如果被暂停的Activity处于finishing状态（例如Activity在其onStop中调用了finish函数），则调用finishCurrentActivityLocked。</p>
<p>·  否则，要调用stopActivityLocked处理暂停的Activity。</p>
<p>此处涉及除AMS和目标进程外的第三个进程，即被切换到后台的那个进程。不过至此，我们的目标Activity终于正式登上了历史舞台。</p>
<div>
    <p><strong>提示</strong>本例的分析结束了吗？没有。因为am设置了-W选项，所以其实我们还在startActivityAndWait函数中等待结果。ActivityStack中有两个函数能够触发AMS notifyAll，一个是reportActivityLaunchedLocked，另一个是reportActivityVisibleLocked。前面介绍的activityInternal函数只在fromTimeout为true时才会调用reportActivityLaunchedLocked，而本例中fromTimeout为false，如何是好？该问题的解答非常复杂，姑且先一语带过：当Activity显示出来时，其在AMS中对应ActivityRecord对象的windowVisible函数将被调用，其内部会触发reportActivityLaunchedLocked函数，这样我们的startActivityAndWait才能被唤醒。</p>
</div>
<h4>7.  startActivity分析之后半程总结</h4>
<p>总结startActivity后半部分的流程，主要涉及目标进程和AMS的交互，如图6-15所示。</p>

![image](images/chapter6/image015.png)

<p>图6-15  startActivity后半程总结</p>
<p>图6-15中涉及16个重要函数调用，而且这仅是startActivity后半部分的调用流程，可见整个流程有多么复杂！</p>
<h4>8. startPausingLocked函数分析</h4>
<p>现在我们分析图6-14中的startPausingLocked分支。根据前面的介绍，当启动一个新Activity时，系统将先行处理当前的Activity，即调用startPausingLocked函数来暂停当前Activity。</p>
<h5>（1） startPausingLocked分析</h5>
<p>[--&gt;ActivityStack.java::startPausingLocked]</p>
<div>
    <p>private final void startPausingLocked(booleanuserLeaving, boolean uiSleeping) {</p>
    <p>   //mResumedActivity保存当前正显示的Activity，</p>
    <p>  ActivityRecord prev = mResumedActivity;</p>
    <p>  mResumedActivity = null;</p>
    <p> </p>
    <p>   //设置mPausingActivity为当前Activity</p>
    <p>  mPausingActivity = prev;</p>
    <p>  mLastPausedActivity = prev;</p>
    <p> </p>
    <p>  prev.state = ActivityState.PAUSING;//设置状态为PAUSING</p>
    <p>  prev.task.touchActiveTime();</p>
    <p>   ......</p>
    <p>   if(prev.app != null &amp;&amp; prev.app.thread != null) {</p>
    <p>     try {</p>
    <p>        //①调用当前Activity所在进程的schedulePauseActivity函数</p>
    <p>        prev.app.thread.schedulePauseActivity(prev,prev.finishing, </p>
    <p>                                userLeaving,prev.configChangeFlags);</p>
    <p>       if(mMainStack) mService.updateUsageStats(prev, false);</p>
    <p>     } ......//catch分支</p>
    <p>    }......//else分支</p>
    <p> </p>
    <p>   if(!mService.mSleeping &amp;&amp; !mService.mShuttingDown) {</p>
    <p>        //获取WakeLock，以防止在Activity切换过程中掉电</p>
    <p>       mLaunchingActivity.acquire();</p>
    <p>        if(!mHandler.hasMessages(LAUNCH_TIMEOUT_MSG)) {</p>
    <p>            Message msg = mHandler.obtainMessage(LAUNCH_TIMEOUT_MSG);</p>
    <p>            mHandler.sendMessageDelayed(msg, LAUNCH_TIMEOUT);</p>
    <p>        }</p>
    <p>   }</p>
    <p> </p>
    <p>    if(mPausingActivity != null) {</p>
    <p>      //暂停输入事件派发</p>
    <p>      if(!uiSleeping) prev.pauseKeyDispatchingLocked();</p>
    <p>     //设置PAUSE超时，时间为500毫秒，这个时间相对较短</p>
    <p>     Message msg = mHandler.obtainMessage(PAUSE_TIMEOUT_MSG);</p>
    <p>     msg.obj = prev;</p>
    <p>     mHandler.sendMessageDelayed(msg, PAUSE_TIMEOUT);</p>
    <p>    }......//else分支</p>
    <p> }</p>
</div>
<p>startPausingLocked将调用应用进程的schedulePauseActivity函数，并设置500毫秒的超时时间，所以应用进程需尽快完成相关处理。和scheduleLaunchActivity一样，schedulePauseActivity将向ActivityThread主线程发送PAUSE_ACTIVITY消息，最终该消息由handlePauseActivity来处理。</p>
<h5>（2） handlePauseActivity分析</h5>
<p>[--&gt;ActivityThread.java::handlePauseActivity]</p>
<div>
    <p>private void handlePauseActivity(IBinder token,boolean finished,</p>
    <p>                               boolean userLeaving, int configChanges){</p>
    <p>  //当Activity处于finishing状态时，finished参数为true，不过在本例中该值为false</p>
    <p>  ActivityClientRecord r = mActivities.get(token);</p>
    <p>   if (r !=null) {</p>
    <p>      //调用Activity的onUserLeaving函数，</p>
    <p>      if(userLeaving) performUserLeavingActivity(r);</p>
    <p>      r.activity.mConfigChangeFlags |=configChanges;</p>
    <p>      //调用Activity的onPause函数</p>
    <p>     performPauseActivity(token, finished, r.isPreHoneycomb());</p>
    <p>      ......</p>
    <p>      try {</p>
    <p>             //调用AMS的activityPaused函数</p>
    <p>             ActivityManagerNative.getDefault().activityPaused(token);</p>
    <p>         }......</p>
    <p>    }</p>
    <p> }</p>
</div>
<p>[--&gt;ActivityManagerService.java::activityPaused]</p>
<div>
    <p>public final void activityPaused(IBinder token) {</p>
    <p>   ......</p>
    <p>   mMainStack.activityPaused(token, false);</p>
    <p>}</p>
</div>
<p>[--&gt;ActivityStack.java::activityPaused]</p>
<div>
    <p>final void activityPaused(IBinder token, booleantimeout) {</p>
    <p> ActivityRecord r = null;</p>
    <p> </p>
    <p>  synchronized (mService) {</p>
    <p>   int index= indexOfTokenLocked(token);</p>
    <p>   if (index&gt;= 0) {</p>
    <p>        r =mHistory.get(index);</p>
    <p>         //从消息队列中撤销PAUSE_TIMEOUT_MSG消息</p>
    <p>        mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);</p>
    <p>         if(mPausingActivity == r) {</p>
    <p>            r.state = ActivityState.PAUSED;//设置ActivityRecord的状态</p>
    <p>            completePauseLocked();//完成本次Pause操作</p>
    <p>        }......</p>
    <p>   }</p>
    <p> }</p>
</div>
<h5>（3） completePauseLocked分析</h5>
<p>[--&gt;ActivityStack.java::completePauseLocked]</p>
<div>
    <p>private final void completePauseLocked() {</p>
    <p>  ActivityRecord prev = mPausingActivity;</p>
    <p>   if (prev!= null) {</p>
    <p>     if(prev.finishing) {</p>
    <p>          prev = finishCurrentActivityLocked(prev, FINISH_AFTER_VISIBLE);</p>
    <p>      } elseif (prev.app != null) {</p>
    <p>        if(prev.configDestroy) {</p>
    <p>             destroyActivityLocked(prev, true, false);</p>
    <p>         } else {</p>
    <p>          //①将刚才被暂停的Activity保存到mStoppingActivities中</p>
    <p>          mStoppingActivities.add(prev);</p>
    <p>          if(mStoppingActivities.size() &gt; 3) {</p>
    <p>            //如果被暂停的Activity超过3个，则发送IDLE_NOW_MSG消息，该消息最终</p>
    <p>           //由我们前面介绍的activeIdleInternal处理</p>
    <p>             scheduleIdleLocked();</p>
    <p>            } </p>
    <p>         }</p>
    <p>      //设置mPausingActivity为null，这是图6-14②、③分支的分割点</p>
    <p>       mPausingActivity = null;</p>
    <p>    }</p>
    <p>     //②resumeTopActivityLocked将启动目标Activity</p>
    <p>    if(!mService.isSleeping()) resumeTopActivityLocked(prev);</p>
    <p> </p>
    <p>   ......</p>
    <p>}</p>
</div>
<p>就本例而言，以上代码还算简单，最后还是通过resumeTopActivityLocked来启动目标Activity。当然，由于之前已经设置了mPausingActivity为null，所以最终会走到图6-14中③的分支。</p>
<h5>（4） stopActivityLocked分析</h5>
<p>根据前面的介绍，此次目标Activity将走完onCreate、onStart和onResume流程，但是被暂停的Activity才刚走完onPause流程，那么它的onStop什么时候调用呢？</p>
<p>答案就在activityIdelInternal中，它将为mStoppingActivities中的成员调用stopActivityLocked函数。</p>
<p>[--&gt;ActivityStack.java::stopActivityLocked]</p>
<div>
    <p> privatefinal void stopActivityLocked(ActivityRecord r) {</p>
    <p>   if((r.intent.getFlags()&amp;Intent.FLAG_ACTIVITY_NO_HISTORY) != 0</p>
    <p>           || (r.info.flags&amp;ActivityInfo.FLAG_NO_HISTORY) != 0) {</p>
    <p>       if(!r.finishing) {</p>
    <p>           requestFinishActivityLocked(r, Activity.RESULT_CANCELED, null,</p>
    <p>                       "no-history");</p>
    <p>         }</p>
    <p>     } elseif (r.app != null &amp;&amp; r.app.thread != null) {</p>
    <p>         try {</p>
    <p>         r.stopped = false;</p>
    <p>          //设置STOPPING状态，并调用对应的scheduleStopActivity函数</p>
    <p>          r.state = ActivityState.STOPPING; </p>
    <p>          r.app.thread.scheduleStopActivity(r, r.visible,</p>
    <p>                     r.configChangeFlags);</p>
    <p>    }......</p>
    <p>  }</p>
</div>
<p>对应进程的scheduleStopActivity函数将根据visible的情况，向主线程消息循环发送H. STOP_ACTIVITY_HIDE或H. STOP_ACTIVITY_SHOW消息。不论哪种情况，最终都由handleStopActivity来处理。</p>
<p>[--&gt;ActivityThread.java::handleStopActivity]</p>
<div>
    <p>private void handleStopActivity(IBinder token,boolean show, int configChanges) {</p>
    <p> ActivityClientRecord r = mActivities.get(token);</p>
    <p> r.activity.mConfigChangeFlags |= configChanges;</p>
    <p> </p>
    <p>  StopInfoinfo = new StopInfo();</p>
    <p>  //调用Activity的onStop函数</p>
    <p> performStopActivityInner(r, info, show, true);</p>
    <p> ......</p>
    <p>  try { //调用AMS的activityStopped函数</p>
    <p>        ActivityManagerNative.getDefault().activityStopped(</p>
    <p>               r.token, r.state, info.thumbnail, info.description);</p>
    <p>   }</p>
    <p>}</p>
</div>
<p>AMS没有为stop设置超时消息处理。严格来说，还是有超时限制的，只是这个超时处理与activityIdleInternal结合起来了。</p>
<h5>（5） startPausingLocked总结</h5>
<p>总结startPausingLocked流程，如图6-16所示。</p>

![image](images/chapter6/image016.png)

<p>图6-16  startPausingActivity流程总结</p>
<p>图6-16比较简单，读者最好结合代码再把流程走一遍，以加深理解。</p>
<h4>9.  startActivity总结</h4>
<p>Activity的启动就介绍到这里。这一路分析下来，相信读者也和笔者一样觉得此行绝不轻松。先回顾一下此次旅程：</p>
<p>·   行程的起点是am。am是Android中很重要的程序，读者务必要掌握它的用法。我们利用am start命令，发起本次目标Activity的启动请求。</p>
<p>·  接下来进入ActivityManagerService和ActivityStack这两个核心类。对于启动Activity来说，这段行程又可分细分为两个阶段：第一阶段的主要工作就是根据启动模式和启动标志找到或创建ActivityRecord及对应的TaskRecord；第二阶段工作就是处理Activity启动或切换相关的工作。</p>
<p>·  首先讨论了AMS直接创建目标进程并运行Activity的流程，其中涉及目标进程的创建，在目标进程中Android运行环境的初始化，目标Activity的创建以及触发onCreate、onStart及onResume等其生命周期中重要函数调用等相关知识点。</p>
<p>·  接着又讨论了AMS先pause当前Activity，然后再创建目标进程并运行Activity的流程。其中牵扯到两个应用进程和AMS的交互，其难度之大可见一斑。</p>
<p>读者在阅读本节时，务必要区分此旅程中两个阶段工作的重点：其一是找到合适的ActivityRecord和TaskRecord；其二是调度相关进程进行Activity切换。在SDK文档中，介绍最为详细的是第一阶段中系统的处理策略，例如启动模式、启动标志的作用等。第二阶段工作其实是与Android组件调度相关的工作。SDK文档只是针对单个Activity进行生命周期方面的介绍。</p>
<p>坦诚地说，这次旅程略过不少逻辑情况。原因有二，一方面受限于精力和篇幅，另方面是作为调度核心类，和AMS相关的代码及处理逻辑非常复杂，而且其间还夹杂了与WMS的交互逻辑，使复杂度更甚。再者，笔者个人感觉这部分代码绝谈不上高效、严谨和美观，甚至有些丑陋（在分析它们的过程中，远没有研究Audio、Surface时那种畅快淋漓的感觉）。</p>
<p>此处列出几个供读者深入研究的点：</p>
<p>·  各种启动模式、启动标志的处理流程。</p>
<p>·  Configuration发生变化时Activity的处理，以及在Activity中对状态保存及恢复的处理流程。</p>
<p>·  Activity生命周期各个阶段的转换及相关处理。Android 2.3以后新增的与Fragment的生命周期相关的转换及处理。</p>
<div>
    <p><strong>建议</strong>在研究代码前，先仔细阅读SDK文档相关内容，以获取必要的感性认识，否则直接看代码很容易迷失方向。</p>
</div>
<h2>6.4  Broadcast和BroadcastReceiver分析</h2>
<p>Broadcast，汉语意思为“广播”。它是Android平台中的一种通知机制。从广义来说，它是一种进程间通信的手段。有广播，就对应有广播接收者。Android中四大组件之一的BroadcastReceiver即代表广播接收者。目前，系统提供两种方式来声明一个广播接收者。</p>
<p>·  在AndroidManifest.xml中声明&lt;receiver&gt;标签。在应用程序运行时，系统会利用Java反射机制构造一个广播接收者实例。本书将这种广播接收者称为静态注册者或静态接收者。</p>
<p>·  在应用程序运行过程中，可调用Context提供的registerReceiver函数注册一个广播接收者实例。本书将这种广播接收者称为动态注册者或动态接收者。与之相对应，当应用程序不再需要监听广播时（例如当应用程序退到后台时），则要调用unregisterReceiver函数撤销之前注册的BroadcastReceiver实例。</p>
<p>当系统将广播派发给对应的广播接收者时，广播接收者的onReceive函数会被调用。在此函数中，可对该广播进行相应处理。</p>
<p>另外，Android定义了三种不同类型的广播发送方式，它们分别是：</p>
<p>·  普通广播发送方式，由sendBroadcast及相关函数发送。以工作中的场景为例，当程序员们正埋头工作之时，如果有人大喊一声“吃午饭去”，前刻还在专心编码的人即作鸟兽散。这种方式即为普通广播发送方式，所有对“吃午饭”感兴趣的接收者都会响应。</p>
<p>·  串行广播发送方式，即ordered广播，由sendOrdedBroadcast及相关函数发送。在该类型方式下，按接收者的优先级将广播一个个地派发给接收者。只有等这一个接收者处理完毕，系统才将该广播派发给下一个接收者。其中，任意一个接收者都可以中止后续的派发流程。还是以工作中的场景为例：经常有项目经理（PM）深夜组织一帮人跟踪bug的状态。PM看见一个bug，问某程序员，“这个bug你能改吗?”如果得到的答案是“暂时不会”或“暂时没时间”，他会将目光转向下一个神情轻松者，直到找到一个担当者为止。这种方式即为ordered广播发送方式，很明显，它的特点是“一个一个来”。</p>
<p>·  Sticky广播发送方式，由sendStickyBroadcast及相关函数发送。Sticky的意思是“粘”，其背后有一个很重要的考虑。我们举个例子：假设某广播发送者每5秒发送一次携带自己状态信息的广播，此时某个应用进程注册了一个动态接收者来监听该广播，那么该接收者的OnReceive函数何时被调用呢？在正常情况下需要等这一轮的5秒周期结束后才调用（因为发送者在本周期结束后会主动再发一个广播）。而在Sticky模式下，系统将马上派发该广播给刚注册的接收者。注意，这个广播是系统发送的，其中存储的是上一次广播发送者的状态信息。也就是说，在Sticky模式下，广播接收者能立即得到广播发送者的信息，而不用等到这一轮周期结束。其实就是系统会保存Sticky的广播<a>[⑤]</a>，当有新广播接收者来注册时，系统就把Sticky广播发给它。</p>
<p>以上我们对广播及广播接收者做了一些简单介绍，读者也可参考SDK文档中的相关说明来加强理解。</p>
<p>下面将以动态广播接收者为例，分析Android对广播的处理流程。</p>
<h3>6.4.1  registerReceiver流程分析</h3>
<h4>1.  ContextImpl registerReceiver分析</h4>
<p>registerReceiver函数用于注册一个动态广播接收者，该函数在Context.java中声明。根据本章前面对Context家族的介绍（参考图6-3），其功能最终将通过ContextImpl类的registerReceiver函数来完成，可直接去看ContextImpl是如何实现此函数的。在SDK中一共定义了两个同名的registerReceiver函数，其代码如下：</p>
<p>[--&gt;ContextImpl.java::registerReceiver]</p>
<div>
    <p>/*</p>
    <p>  在SDK中输出该函数，这也是最常用的函数。当广播到来时，BroadcastReceiver对象的onReceive</p>
    <p>  函数将在主线程中被调用</p>
    <p>*/</p>
    <p>public Intent registerReceiver(BroadcastReceiverreceiver, IntentFilter filter) {</p>
    <p>    returnregisterReceiver(receiver, filter, null, null);</p>
    <p> }</p>
    <p>/*</p>
    <p>   功能和前面类似，但增加了两个参数，分别是broadcastPermission和scheduler，作用有</p>
    <p>   两个：</p>
    <p>   其一：对广播者的权限增加了控制，只有拥有相应权限的广播者发出的广播才能被此接收者接收</p>
    <p>   其二：BroadcastReceiver对象的onReceiver函数可调度到scheduler所在的线程中执行</p>
    <p>*/</p>
    <p> publicIntent registerReceiver(BroadcastReceiver receiver, </p>
    <p>       IntentFilterfilter, String broadcastPermission, Handler scheduler) {</p>
    <p>/*</p>
    <p> 注意，下面所调用函数的最后一个参数为getOuterContext的返回值。前面曾说过，ContextImpl为Context家族中真正干活的对象，而它对外的代理人可以是Application和Activity等，</p>
    <p>getOuterContext就返回这个对外代理人。一般在Activity中调用registerReceiver函数，故此处getOuterContext返回的对外代理人的类型就是Activity。</p>
    <p>*/</p>
    <p>     returnregisterReceiverInternal(receiver, filter, broadcastPermission,</p>
    <p>               scheduler, getOuterContext());</p>
    <p>}</p>
</div>
<p>殊途同归，最终的功能由registerReceiverInternal来完成，其代码如下：</p>
<p>[--&gt;ContextImpl.java::registerReceiverInternal]</p>
<div>
    <p> privateIntent registerReceiverInternal(BroadcastReceiver receiver,</p>
    <p>      IntentFilter filter, String broadcastPermission, Handler scheduler,</p>
    <p>      Context context) {</p>
    <p>   IIntentReceiver rd = null;</p>
    <p>   if(receiver != null) {</p>
    <p>        //①准备一个IIntentReceiver对象</p>
    <p>        if(mPackageInfo != null &amp;&amp; context != null) {</p>
    <p>          //如果没有设置scheduler，则默认使用主线程的Handler</p>
    <p>          if (scheduler == null)   scheduler= mMainThread.getHandler();</p>
    <p>          //通过getReceiverDispatcher函数得到一个IIntentReceiver类型的对象</p>
    <p>          rd = mPackageInfo.getReceiverDispatcher(</p>
    <p>             receiver, context, scheduler, mMainThread.getInstrumentation(),</p>
    <p>             true);</p>
    <p>        } else {</p>
    <p>          if (scheduler == null)  scheduler= mMainThread.getHandler();</p>
    <p>          //直接创建LoadedApk.ReceiverDispatcher对象</p>
    <p>          rd = new LoadedApk.ReceiverDispatcher(receiver, context, scheduler,</p>
    <p>                   null, true).getIIntentReceiver();</p>
    <p>         }//if (mPackageInfo != null &amp;&amp; context != null)结束</p>
    <p>     }// if(receiver != null)结束</p>
    <p>  try {</p>
    <p>        //②调用AMS的registerReceiver函数</p>
    <p>       return ActivityManagerNative.getDefault().registerReceiver(</p>
    <p>                     mMainThread.getApplicationThread(),mBasePackageName,</p>
    <p>                     rd, filter,broadcastPermission);</p>
    <p>        } ......</p>
    <p> }</p>
</div>
<p>以上代码列出了两个关键点：其一是准备一个IIntentReceiver对象；其二是调用AMS的registerReceiver函数。</p>
<p>先来看IIntentReceiver，它是一个Interface，图6-17列出了和它相关的成员图谱。</p>

![image](images/chapter6/image017.png)

<p>图6-17  IIntentReceiver相关成员示意图</p>
<p>由图6-17可知：</p>
<p>·  BroadcastReceiver内部有一个PendingResult类。该类是Android 2.3以后新增的，用于异步处理广播消息。例如，当BroadcastReceiver收到一个广播时，其onReceive函数将被调用。一般都是在该函数中直接处理该广播。不过，当该广播处理比较耗时，还可采用异步的方式进行处理，即先调用BroadcastReceiver的goAsync函数得到一个PendingResult对象，然后将该对象放到工作线程中去处理，这样onReceive就可以立即返回而不至于耽误太长时间（这一点对于onReceive函数被主线程调用的情况尤为有用）。在工作线程处理完这条广播后，需调用PendingResult的finish函数来完成整个广播的处理流程。</p>
<p>·  广播由AMS发出，而接收及处理工作却在另外一个进程中进行，整个过程一定涉及进程间通信。在图6-17中，虽然在BroadcastReceiver中定义了一个广播接收者，但是它与Binder没有有任何关系，故其并不直接参与进程间通信。与之相反，IIntentReceiver接口则和Binder有密切关系，故可推测广播的接收是由IIntentReceiver接口来完成的。确实，在整个流程中，首先接收到来自AMS的广播的将是该接口的Bn端，即LoadedApk.ReceiverDispather的内部类InnerReceiver。</p>
<p>接收广播的处理将放到本节最后再来分析，下面先来看AMS 的registerReceiver函数。</p>
<h4>2.  AMS的registerReceiver分析</h4>
<p>AMS的registerReceiver函数比较简单，但是由于其中将出现一些新的变量类型和成员，因此接下来按分两部分进行分析。</p>
<h5>（1） registerReceiver分析之一</h5>
<p>registerReceiver的返回值是一个Intent，它指向一个匹配过滤条件（由filter参数指明）的Sticky Intent。即使有多个符合条件的Intent，也只返回一个。</p>
<p>[--&gt;ActivityManagerService.java::registerReceiver]</p>
<div>
    <p>public Intent registerReceiver(IApplicationThreadcaller, String callerPackage,</p>
    <p>           IIntentReceiver receiver, IntentFilter filter, String permission) {</p>
    <p>  synchronized(this) {</p>
    <p>      ProcessRecord callerApp = null;</p>
    <p>      if(caller != null) {</p>
    <p>         callerApp = getRecordForAppLocked(caller);</p>
    <p>         ....... //如果callerApp为null，则抛出异常，即系统不允许未登记照册的进程注册</p>
    <p>         //动态广播接收者</p>
    <p> </p>
    <p>          //检查调用进程是否有callerPackage的信息，如果没有，也抛异常</p>
    <p>          if(callerApp.info.uid != Process.SYSTEM_UID &amp;&amp;</p>
    <p>             !callerApp.pkgList.contains(callerPackage)){</p>
    <p>                </p>
    <p>                throw new SecurityException(......);</p>
    <p>             }</p>
    <p>       }......//if(caller != null)判断结束</p>
    <p> </p>
    <p>     List allSticky = null;</p>
    <p>      //下面这段代码的功能是从系统中所有Sticky Intent中查询匹配IntentFilter的Intent，</p>
    <p>     //匹配的Intent保存在allSticky中</p>
    <p>     Iterator actions = filter.actionsIterator();</p>
    <p>      if(actions != null) {</p>
    <p>        while (actions.hasNext()) {</p>
    <p>              String action = (String)actions.next();</p>
    <p>              allSticky = getStickiesLocked(action, filter, allSticky);</p>
    <p>           }</p>
    <p>       } ......</p>
    <p>     //如果存在sticky的Intent，则选取第一个Intent作为本函数的返回值</p>
    <p>     Intentsticky = allSticky != null ? (Intent)allSticky.get(0) : null;</p>
    <p>     //如果没有设置接收者，则直接返回sticky 的intent</p>
    <p>     if(receiver == null)  return sticky;</p>
    <p> </p>
    <p>    //新的数据类型ReceiverList及mRegisteredReceivers成员变量，见下文的解释</p>
    <p>  //receiver.asBinder将返回IIntentReceiver的Bp端</p>
    <p>    ReceiverList rl</p>
    <p>               = (ReceiverList)mRegisteredReceivers.get(receiver.asBinder());</p>
    <p> </p>
    <p>     //如果是首次调用，则此处rl的值将为null</p>
    <p>     if (rl== null) {</p>
    <p>       rl =new ReceiverList(this, callerApp, Binder.getCallingPid(),</p>
    <p>                     Binder.getCallingUid(),receiver);</p>
    <p>       if (rl.app != null) {</p>
    <p>           rl.app.receivers.add(rl);</p>
    <p>       }else {</p>
    <p>         try {</p>
    <p>                 //监听广播接收者所在进程的死亡消息</p>
    <p>                 receiver.asBinder().linkToDeath(rl, 0);</p>
    <p>             }...... </p>
    <p>          rl.linkedToDeath = true;</p>
    <p>        }// if(rl.app != null)判断结束</p>
    <p> </p>
    <p>           //将rl保存到mRegisterReceivers中</p>
    <p>          mRegisteredReceivers.put(receiver.asBinder(), rl);</p>
    <p>     }</p>
    <p>    //新建一个BroadcastFilter对象</p>
    <p>    BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,</p>
    <p>                                                        permission);</p>
    <p>    rl.add(bf);//将其保存到rl中</p>
    <p>     // mReceiverResolver成员变量，见下文解释</p>
    <p>    mReceiverResolver.addFilter(bf);</p>
</div>
<p>以上代码的流程倒是很简单，不过其中出现的几个成员变量和数据类型却严重阻碍了我们的思维活动。先解决它们，BroadcastFilter及相关成员变量如图6-18所示。</p>

![image](images/chapter6/image018.png)

<p>图6-18  BroadcastFilter及相关成员变量</p>
<p>结合代码，对图6-18中各数据类型和成员变量的作用及关系的解释如下：</p>
<p>·  在AMS中，BroadcastReceiver的过滤条件由BroadcastFilter表示，该类从IntentFilter派生。由于一个BroadcastReceiver可设置多个过滤条件（即多次为同一个BroadcastReceiver对象调用registerReceiver函数以设置不同的过滤条件），故AMS使用ReceiverList（从ArrayList&lt;BroadcastFilter&gt;派生）这种数据类型来表达这种一对多的关系。</p>
<p>·  ReceiverList除了能存储多个BroadcastFilter外，还应该有成员指向某一个具体BroadcastReceiver，否则如何知道到底是哪个BroadcastReceiver设置的过滤条件呢？前面说过，BroadcastReceiver接收广播是通过IIntentReceiver接口进行的，故ReceiverList中有receiver成员变量指向IIntentReceiver。</p>
<p>·  AMS提供mRegisterReceivers用于保存IIntentReceiver和对应ReceiverList的关系。此外，AMS还提供mReceiverResolver变量用于存储所有动态注册的BroadcastReceiver所设置的过滤条件。</p>
<p>清楚这些成员变量和数据类型之间的关系后，接着来分析registerReceiver第二阶段的工作。</p>
<h5>（2） registerReceiver分析之二</h5>
<p>[--&gt;ActivityManagerService.java::registerReceiver]</p>
<div>
    <p>    //如果allSticky不为空，则表示有Sticky的Intent，需要立即调度广播发送</p>
    <p>     if(allSticky != null) {</p>
    <p>        ArrayList receivers = new ArrayList();</p>
    <p>         receivers.add(bf);</p>
    <p>         intN = allSticky.size();</p>
    <p>         for(int i=0; i&lt;N; i++) {</p>
    <p>            Intent intent = (Intent)allSticky.get(i);</p>
    <p>            //为每一个需要发送的广播创建一个BroadcastRecord（暂称之为广播记录）对象</p>
    <p>            BroadcastRecord r = new BroadcastRecord(intent, null,</p>
    <p>                            null, -1, -1, null,receivers, null, 0, null, null,</p>
    <p>                            false, true, true);</p>
    <p>            //如果mParallelBroadcasts当前没有成员，则需要触发AMS发送广播</p>
    <p>            if (mParallelBroadcasts.size() == 0) </p>
    <p>                 scheduleBroadcastsLocked();//向AMS发送BROADCAST_INTENT_MSG消息</p>
    <p>             //所有非ordered广播记录都保存在mParallelBroadcasts中</p>
    <p>             mParallelBroadcasts.add(r);</p>
    <p>          }//for循环结束</p>
    <p>    }//if (allSticky != null)判断结束</p>
    <p>    returnsticky;</p>
    <p>   }//synchronized结束</p>
    <p>}</p>
</div>
<p>这一阶段的工作用一句话就能说清楚：为每一个满足IntentFilter的Sticky的intent创建一个BroadcastRecord对象，并将其保存到mParllelBroadcasts数组中，最后，根据情况调度AMS发送广播。</p>
<p>从上边的描述中可以看出，一旦存在满足条件的Sticky的Intent，系统需要尽快调度广播发送。说到这里，想和读者分享一种在实际工作中碰到的情况。</p>
<p>我们注册了一个BroadcastReceiver，用于接收USB的连接状态。在注册完后，它的onReceiver函数很快就会被调用。当时笔者的一些同事认为是注册操作触发USB模块又发送了一次广播，却又感到有些困惑，USB模块应该根据USB的状态变化去触发广播发送，而不应理会广播接收者的注册操作，这到底是怎么一回事呢？</p>
<p>相信读者现在应该轻松解决这种困惑了吧？对于Sticky的广播，一旦有接收者注册，系统会马上将该广播传递给它们。</p>
<p>和ProcessRecord及ActivityRecord类似，AMS定义了一个BroadcastRecord数据结构，用于存储和广播相关的信息，同时还有两个成员变量，它们作用和关系如图6-19所示。</p>

![image](images/chapter6/image019.png)

<p>图6-19  BroadcastReceiver及相关变量</p>
<p>图6-19比较简单，读者可自行研究。</p>
<p>在代码中，registerReceiver将调用scheduleBroadcastsLocked函数，通知AMS立即着手开展广播发送工作，在其内部就发送BROADCAST_INTENT_MSG消息给AMS。相关的处理工作将放到本节最后再来讨论。</p>
<p> </p>
<h3>6.4.2  sendBroadcast流程分析</h3>
<p>在SDK中同样定义了好几个函数用于发送广播。不过，根据之前的经验，最终和AMS交互的函数可能通过一个接口就能完成。来看最简单的广播发送函数sendBroadcast，其代码如下：</p>
<p>[--&gt;ContextImpl.java::sendBroadcast]</p>
<div>
    <p>public void sendBroadcast(Intent intent) {</p>
    <p>   StringresolvedType = intent.resolveTypeIfNeeded(getContentResolver());</p>
    <p>   try {</p>
    <p>       intent.setAllowFds(false);</p>
    <p>         //调用AMS的brodcastIntent，在SDK中定义的广播发送函数最终都会调用它</p>
    <p>        ActivityManagerNative.getDefault().broadcastIntent(</p>
    <p>               mMainThread.getApplicationThread(), intent, resolvedType, null,</p>
    <p>               Activity.RESULT_OK, null, null, null, false, false);</p>
    <p>        }......</p>
    <p> }</p>
</div>
<p>AMS的broadcastIntent函数的主要工作将交由AMS的broadcastIntentLocked来完成，故此处直接分析broadcastIntentLocked。</p>
<h4>1.  broadcastIntentLocked分析</h4>
<p>我们分阶段来分析broadcastIntentLocked的工作，先来看第一阶段工作。</p>
<h5>（1） broadcastIntentLocked分析之一</h5>
<p>[--&gt;ActivityManagerService.java::broadcastIntentLocked]</p>
<div>
    <p>private final int broadcastIntentLocked(ProcessRecordcallerApp,</p>
    <p>      StringcallerPackage, Intent intent, String resolvedType,</p>
    <p>     IIntentReceiver resultTo, int resultCode, String resultData,</p>
    <p>      Bundlemap, String requiredPermission,</p>
    <p>     boolean ordered, boolean sticky, int callingPid, int callingUid) {</p>
    <p> </p>
    <p>   intent =new Intent(intent);</p>
    <p>   //为Intent增加FLAG_EXCLUDE_STOPPED_PACKAGES标志，表示该广播不会传递给被STOPPED</p>
    <p>   //的Package</p>
    <p>  intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);</p>
    <p>   //处理一些特殊的广播，包括UID_REMOVED,PACKAGE_REMOVED和PACKAGE_ADDED等</p>
    <p>   finalboolean uidRemoved = Intent.ACTION_UID_REMOVED.equals(</p>
    <p>                                                        intent.getAction());</p>
    <p>   ......//处理特殊的广播，主要和PACKAGE相关，例如接收到PACKAGE_REMOVED广播后,AMS</p>
    <p>   //需将该Package的组件从相关成员中删除，相关代码可自行阅读</p>
    <p> </p>
    <p>   //处理TIME_ZONE变化广播</p>
    <p>   if(intent.ACTION_TIMEZONE_CHANGED.equals(intent.getAction()))</p>
    <p>        mHandler.sendEmptyMessage(UPDATE_TIME_ZONE);</p>
    <p>   //处理CLEAR_DNS_CACHE广播</p>
    <p>   if(intent.ACTION_CLEAR_DNS_CACHE.equals(intent.getAction()))</p>
    <p>        mHandler.sendEmptyMessage(CLEAR_DNS_CACHE);</p>
    <p>   //处理PROXY_CHANGE广播</p>
    <p>   if(Proxy.PROXY_CHANGE_ACTION.equals(intent.getAction())) {</p>
    <p>        ProxyProperties proxy = intent.getParcelableExtra("proxy");</p>
    <p>         mHandler.sendMessage(mHandler.obtainMessage(</p>
    <p>                                  UPDATE_HTTP_PROXY,proxy));</p>
    <p>   }</p>
</div>
<p>从以上代码可知，broadcastIntentLocked第一阶段的工作主要是处理一些特殊的广播消息。</p>
<p>下面来看broadcastIntentLocked第二阶段的工作。</p>
<h5>（2） broadcastIntentLocked分析之二</h5>
<p>[--&gt;ActivityManagerService.java::broadcastIntentLocked]</p>
<div>
    <p>   //处理发送sticky广播的情况</p>
    <p>  if(sticky) {</p>
    <p>   ......//检查发送进程是否有BROADCAST_STICKY权限</p>
    <p> </p>
    <p>   if(requiredPermission != null) {</p>
    <p>       //在发送Sticky广播的时候，不能携带权限信息</p>
    <p>        returnBROADCAST_STICKY_CANT_HAVE_PERMISSION;</p>
    <p>     }</p>
    <p>   //在发送Stikcy广播的时候，也不能指定特定的接收对象</p>
    <p>   if(intent.getComponent() != null) ......//抛异常</p>
    <p> </p>
    <p>   //将这个Sticky的Intent保存到mStickyBroadcasts中</p>
    <p>  ArrayList&lt;Intent&gt; list =mStickyBroadcasts.get(intent.getAction());</p>
    <p>   if (list== null) {</p>
    <p>        list= new ArrayList&lt;Intent&gt;();</p>
    <p>        mStickyBroadcasts.put(intent.getAction(), list);</p>
    <p>    }</p>
    <p>    ......//如果list中已经有该intent，则直接用新的intent替换旧的intent</p>
    <p>    //否则将该intent加入到list中保存</p>
    <p>    if (i &gt;= N) list.add(new Intent(intent));</p>
    <p>  }</p>
    <p> </p>
    <p>    //定义两个变量，其中receivers用于保存匹配该Intent的所有广播接收者，包括静态或动态</p>
    <p>    //注册的广播接收者，registeredReceivers用于保存符合该Intent的所有动态注册者</p>
    <p>   Listreceivers = null;</p>
    <p>   List&lt;BroadcastFilter&gt;registeredReceivers = null;</p>
    <p> </p>
    <p>   try {</p>
    <p>       if(intent.getComponent() != null) {</p>
    <p>        ......//如果指定了接收者，则从PKMS中查询它的信息</p>
    <p>       }else {</p>
    <p>          //FLAG_RECEIVER_REGISTERED_ONLY标签表明该广播只能发给动态注册者</p>
    <p>          if ((intent.getFlags()&amp;Intent.FLAG_RECEIVER_REGISTERED_ONLY)== 0) {</p>
    <p>            //如果没有设置前面的标签，则需要查询PKMS，获得那些在AndroidManifest.xml</p>
    <p>           //中声明的广播接收者，即静态广播接收者的信息，并保存到receivers中</p>
    <p>             receivers =</p>
    <p>                   AppGlobals.getPackageManager().queryIntentReceivers(</p>
    <p>                          intent, resolvedType,STOCK_PM_FLAGS);</p>
    <p>            }</p>
    <p>          //再从AMS的mReceiverResolver中查询符合条件的动态广播接收者</p>
    <p>            registeredReceivers =mReceiverResolver.queryIntent(intent,</p>
    <p>                                                resolvedType, false);</p>
    <p>         }</p>
    <p>        }......</p>
    <p> </p>
    <p>  /*</p>
    <p>   判断该广播是否设置了REPLACE_PENDING标签。如果设置了该标签，并且之前的那个Intent还</p>
    <p>   没有被处理，则可以用新的Intent替换旧的Intent。这么做的目的是为了减少不必要的广播发送，</p>
    <p>   但笔者感觉，这个标签的作用并不靠谱，因为只有旧的Intent没被处理的时候，它才能被替换。</p>
    <p>   因为旧Intent被处理的时间不能确定，所以不能保证广播发送者的一番好意能够实现。因此，</p>
    <p>   在发送广播时，千万不要以为设置了该标志就一定能节约不必要的广播发送。</p>
    <p>  */</p>
    <p>   final boolean replacePending =</p>
    <p>        (intent.getFlags()&amp;Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;</p>
    <p> </p>
    <p>  //先处理动态注册的接收者</p>
    <p>   int NR =registeredReceivers != null ? registeredReceivers.size() : 0;</p>
    <p>   //如果此次广播为非串行化发送，并且符合条件的动态注册接收者个数不为零</p>
    <p>   if(!ordered &amp;&amp; NR &gt; 0) {</p>
    <p>      //创建一个BroadcastRecord对象即可，注意，一个BroadcastRecord对象可包括所有的</p>
    <p>     //接收者（可参考图6-19）</p>
    <p>     BroadcastRecord r = new BroadcastRecord(intent, callerApp,</p>
    <p>                   callerPackage, callingPid, callingUid, requiredPermission,</p>
    <p>                   registeredReceivers, resultTo, resultCode, resultData, map,</p>
    <p>                   ordered, sticky, false);</p>
    <p> </p>
    <p>       boolean replaced = false;</p>
    <p>       if (replacePending) {</p>
    <p>       ......//从mParalledBroadcasts中查找是否有旧的Intent，如果有就替代它，并设置</p>
    <p>      //replaced为true</p>
    <p>     }</p>
    <p>       if(!replaced) {//如果没有被替换，则保存到mParallelBroadcasts中</p>
    <p>          mParallelBroadcasts.add(r);</p>
    <p>          scheduleBroadcastsLocked();//调度一次广播发送</p>
    <p>       }</p>
    <p>       //至此，动态注册的广播接收者已处理完毕，设置registeredReceivers为null</p>
    <p>      registeredReceivers = null;//</p>
    <p>       NR =0;</p>
    <p>  }</p>
</div>
<p>broadcastIntentLocked第二阶段的工作有两项：</p>
<p>·  查询满足条件的动态广播接收者及静态广播接收者。</p>
<p>·  当本次广播不为ordered时，需要尽快发送该广播。另外，非ordered的广播都被AMS保存在mParallelBroadcasts中。</p>
<h5>（3） broadcastIntentLocked分析之三</h5>
<p>下面来看broadcastIntentLocked最后一阶段的工作，其代码如下：</p>
<p>[--&gt;ActivityManagerService.java::broadcastIntentLocked]</p>
<div>
    <p>   int ir = 0;</p>
    <p>   if(receivers != null) {</p>
    <p>      StringskipPackages[] = null;</p>
    <p>     ......//处理PACKAGE_ADDED的Intent，系统不希望有些应用程序一安装就启动。</p>
    <p>      //这类程序的工作原理是什么呢？即在该程序内部监听PACKAGE_ADDED广播。如果系统</p>
    <p>     //没有这一招防备，则PKMS安装完程序后所发送的PAKCAGE_ADDED消息将触发该应用的启动</p>
    <p> </p>
    <p>    ......//处理ACTION_EXTERNAL_APPLICATIONS_AVAILABLE广播</p>
    <p> </p>
    <p>   ......//将动态注册的接收者registeredReceivers的元素合并到receivers中去</p>
    <p>   //处理完毕后，所有的接收者（无论动态还是静态注册的）都存储到receivers变量中了</p>
    <p>   if((receivers != null &amp;&amp; receivers.size() &gt; 0) || resultTo != null) {</p>
    <p>         //创建一个BroadcastRecord对象，注意它的receivers中包括所有的接收者</p>
    <p>        BroadcastRecord r = new BroadcastRecord(intent, callerApp,</p>
    <p>              callerPackage, callingPid, callingUid, requiredPermission,</p>
    <p>              receivers, resultTo, resultCode, resultData, map, ordered,</p>
    <p>              sticky, false);</p>
    <p>       booleanreplaced = false;</p>
    <p>       if (replacePending) {</p>
    <p>        ......//替换mOrderedBroadcasts中旧的Intent</p>
    <p>        }//</p>
    <p>     if(!replaced) {//如果没有替换，则添加该条广播记录到mOrderedBroadcasts中</p>
    <p>        mOrderedBroadcasts.add(r);</p>
    <p>        scheduleBroadcastsLocked();//调度AMS进行广播发送工作</p>
    <p>     }</p>
    <p>  }</p>
    <p>  returnBROADCAST_SUCCESS;</p>
    <p>}</p>
</div>
<p>由以上代码可知，AMS将动态注册者和静态注册者都合并到receivers中去了。注意，如果本次广播不是ordered，那么表明动态注册者就已经在第二阶段工作中被处理了。因此在合并时，将不会有动态注册者被加到receivers中。最终所创建的广播记录存储在mOrderedBroadcasts中，也就是说，不管是否串行化发送，静态接收者对应的广播记录都将保存在mOrderedBroadcasts中。为什么不将它们保存在mParallelBroadcasts中呢？</p>
<p>结合代码会发现，保存在mParallelBroadcasts中的BroadcastRecord所包含的都是动态注册的广播接收者信息，这是因为动态接收者所在的进程是已经存在的（如果该进程不存在，如何去注册动态接收者呢？），而静态广播接收者就不能保证它已经和一个进程绑定在一起了（静态广播接收者此时可能还仅仅是在AndroidManifest.xml中声明的一个信息，相应的进程并未创建和启动）。根据前一节所分析的Activity启动流程，AMS还需要进行创建应用进程，初始化Android运行环境等一系列复杂的操作。读到后面内容时会发现，AMS将在一个循环中逐条处理mParallelBroadcasts中的成员（即派发给接收者）。如果将静态广播接收者也保存到mParallelBroadcasts中，会有什么后果呢？假设这些静态接收者所对应的进程全部未创建和启动，那么AMS将在那个循环中创建多个进程！结果让人震惊，系统压力一下就上去了。所以，对于静态注册者，它们对应的广播记录都被放到mOrderedBroadcasts中保存。AMS在处理这类广播信息时，一个进程一个进程地处理，只有处理完一个接收者，才继续下一个接收者。这种做法的好处是，避免了惊群效应的出现，坏处则是延时较长。</p>
<p>下面进入AMS的BROADCAST_INTENT_MSG消息处理函数，看看情况是否如上所说。</p>
<h3>6.4.3  BROADCAST_INTENT_MSG消息处理函数</h3>
<p>BROADCAST_INTENT_MSG消息将触发processNextBroadcast函数，下面分阶段来分析它。</p>
<h4>1.  processNextBroadcast分析之一</h4>
<p>[--&gt;ActivityManagerService.java::processNextBroadcast]</p>
<div>
    <p>private final void processNextBroadcast(booleanfromMsg) {</p>
    <p>   //如果是BROADCAST_INTENT_MSG消息触发该函数，则fromMsg为true</p>
    <p>  synchronized(this) {</p>
    <p>    BroadcastRecord r;</p>
    <p>   updateCpuStats();//更新CPU使用情况</p>
    <p>    if(fromMsg)  mBroadcastsScheduled = false;</p>
    <p> </p>
    <p>    //先处理mParallelBroadcasts中的成员。如前所述，AMS在一个循环中处理它们</p>
    <p>    while(mParallelBroadcasts.size() &gt; 0) {</p>
    <p>         r =mParallelBroadcasts.remove(0);</p>
    <p>         r.dispatchTime =SystemClock.uptimeMillis();</p>
    <p>         r.dispatchClockTime =System.currentTimeMillis();</p>
    <p>        final int N = r.receivers.size();</p>
    <p>         for(int i=0; i&lt;N; i++) {</p>
    <p>           Object target = r.receivers.get(i);</p>
    <p>            //①mParallelBroadcasts中的成员全为BroadcastFilter类型，所以下面的函数</p>
    <p>           //将target直接转换成BroadcastFilter类型。注意，最后一个参数为false</p>
    <p>           deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, </p>
    <p>                                                         false);</p>
    <p>        }</p>
    <p>        //将这条处理过的记录保存到mHistoryBroadcast中，供调试使用</p>
    <p>        addBroadcastToHistoryLocked(r);</p>
    <p>  }</p>
</div>
<p>deliverToRegisteredReceiverLocked函数的功能就是派发广播给接收者，其代码如下：</p>
<p>[--&gt;ActivityManagerService.java::deliverToRegisteredReceiverLocked]</p>
<div>
    <p>private final voiddeliverToRegisteredReceiverLocked(BroadcastRecord r,</p>
    <p>                         BroadcastFilter filter, booleanordered) {</p>
    <p>   booleanskip = false;</p>
    <p>   //检查发送进程是否有filter要求的权限</p>
    <p>   if(filter.requiredPermission != null) {</p>
    <p>       intperm = checkComponentPermission(filter.requiredPermission,</p>
    <p>                   r.callingPid, r.callingUid, -1, true);</p>
    <p>       if(perm != PackageManager.PERMISSION_GRANTED) skip = true;</p>
    <p>  }</p>
    <p>   //检查接收者是否有发送者要求的权限</p>
    <p>   if(r.requiredPermission != null) {</p>
    <p>       intperm = checkComponentPermission(r.requiredPermission,</p>
    <p>               filter.receiverList.pid, filter.receiverList.uid, -1, true);</p>
    <p>       if(perm != PackageManager.PERMISSION_GRANTED) skip = true;</p>
    <p>    }</p>
    <p> </p>
    <p>   if(!skip) {</p>
    <p>     if(ordered) {</p>
    <p>     ......//设置一些状态，成员变量等信息，不涉及广播发送</p>
    <p>      } </p>
    <p>     try {</p>
    <p>         //发送广播</p>
    <p>        performReceiveLocked(filter.receiverList.app,   </p>
    <p>              filter.receiverList.receiver,new Intent(r.intent), r.resultCode,</p>
    <p>              r.resultData, r.resultExtras, r.ordered, r.initialSticky);</p>
    <p> </p>
    <p>         if(ordered) r.state = BroadcastRecord.CALL_DONE_RECEIVE;</p>
    <p>       }......</p>
    <p>     }</p>
    <p>   }</p>
    <p> }</p>
</div>
<p>来看performReceiveLocked函数，其代码如下：</p>
<p>[--&gt;ActivityManagerService.java::performReceiveLocked]</p>
<div>
    <p>static void performReceiveLocked(ProcessRecordapp, IIntentReceiver receiver,</p>
    <p>      Intentintent, int resultCode, String data, Bundle extras,</p>
    <p>     boolean ordered, boolean sticky) throws RemoteException {</p>
    <p>      if(app != null &amp;&amp; app.thread != null) {</p>
    <p>         //如果app及app.thread不为null，则调度scheduleRegisteredReceiver，</p>
    <p>        //注意这个函数名为scheduleRegisteredReceiver，它只针对动态注册的广播接收者</p>
    <p>        app.thread.scheduleRegisteredReceiver(receiver,intent, resultCode,</p>
    <p>                   data, extras, ordered, sticky);</p>
    <p>      } else{</p>
    <p>       //否则调用IIntentReceiver的performReceive函数</p>
    <p>      receiver.performReceive(intent, resultCode, data, extras, </p>
    <p>                                     ordered, sticky);</p>
    <p>   }</p>
    <p> }</p>
</div>
<p>对于动态注册者而言，在大部分情况下会执行if分支，所以应用进程ApplicationThread的scheduleRegisteredReceiver函数将被调用。稍后再分析应用进程的广播处理流程。</p>
<h4>2.  processNextBroadcast分析之二</h4>
<p>至此，processNextBroadcast已经在一个while循环中处理完mParallelBroadcasts的所有成员了，实际上，这种处理方式也会造成惊群效应，但影响相对较少。这是因为对于动态注册者来说，它们所在的应用进程已经创建并初始化成功。此处的广播发送只是调用应用进程的一个函数而已。相比于创建进程，再初始化Android运行环境所需的工作量，调用scheduleRegisteredReceiver的工作就比较轻松了。</p>
<p>来看processNextBroadcast第二阶段的工作，代码如下：</p>
<p>[--&gt;ActivityManagerService.java::processNextBroadcast]</p>
<div>
    <p>   /* </p>
    <p>    现在要处理mOrderedBroadcasts中的成员。如前所述，它要处理一个接一个的接受者，如果</p>
    <p>    接收者所在进程还未启动，则需要等待。mPendingBroadcast变量用于标识因为应用进程还未</p>
    <p>    启动而处于等待状态的BroadcastRecord。</p>
    <p>   */</p>
    <p>  if(mPendingBroadcast != null) {</p>
    <p>     boolean isDead;</p>
    <p>     synchronized (mPidsSelfLocked) {</p>
    <p>        isDead= (mPidsSelfLocked.get(mPendingBroadcast.curApp.pid) == null);</p>
    <p>      }</p>
    <p>      /*<strong>重要说明</strong></p>
    <p>       判断要等待的进程是否为dead进程，如果没有dead进程，则继续等待。仔细思考，此处直接</p>
    <p>       返回会有什么问题。</p>
    <p>       问题不小！假设有两个ordered广播A和B，有两个接收者，AR和BR，并且BR所</p>
    <p>       在进程已经启动并完成初始化Android运行环境。如果processNextBroadcast先处理A，</p>
    <p>       再处理B，那么此处B的处理将因为mPendingBroadcast不为空而被延后。虽然B和A</p>
    <p>       之间没有任何关系（例如完全是两个不同的广播消息），</p>
    <p>       但是事实上，大多数开发人员理解的order是针对单个广播的，例如A有5个接收者，那么对这</p>
    <p>       5个接收者的广播的处理是串行的。通过此处的代码发现，系统竟然串行处理A和B广播，即</p>
    <p>       B广播要待到A的5个接收者都处理完了才能处理。</p>
    <p>      */</p>
    <p>      if(!isDead)   return;</p>
    <p>      else {</p>
    <p>       mPendingBroadcast.state = BroadcastRecord.IDLE;</p>
    <p>       mPendingBroadcast.nextReceiver = mPendingBroadcastRecvIndex;</p>
    <p>       mPendingBroadcast = null;</p>
    <p>     }</p>
    <p>   }</p>
    <p> </p>
    <p>  boolean looped = false;</p>
    <p>  do {</p>
    <p>        // mOrderedBroadcasts处理完毕</p>
    <p>        if (mOrderedBroadcasts.size() == 0) {</p>
    <p>            scheduleAppGcsLocked();</p>
    <p>            if (looped)  updateOomAdjLocked();</p>
    <p>             return;</p>
    <p>        }</p>
    <p>        r =mOrderedBroadcasts.get(0);</p>
    <p>       boolean forceReceive = false;</p>
    <p>        //下面这段代码用于判断此条广播是否处理时间过长</p>
    <p>        //先得到该条广播的所有接收者</p>
    <p>        intnumReceivers = (r.receivers != null) ? r.receivers.size() : 0;</p>
    <p>        if(mProcessesReady &amp;&amp; r.dispatchTime &gt; 0) {</p>
    <p>            long now = SystemClock.uptimeMillis();</p>
    <p>           //如果总耗时超过2倍的接收者个数*每个接收者最长处理时间（10秒），则</p>
    <p>           //强制结束这条广播的处理</p>
    <p>            if ((numReceivers &gt; 0) &amp;&amp;</p>
    <p>                   (now &gt; r.dispatchTime + </p>
    <p>                              (2*BROADCAST_TIMEOUT*numReceivers))){</p>
    <p>               broadcastTimeoutLocked(false);//读者阅读完本节后可自行研究该函数</p>
    <p>              forceReceive = true;</p>
    <p>              r.state = BroadcastRecord.IDLE;</p>
    <p>            }</p>
    <p>      }//if(mProcessesReady...)判断结束</p>
    <p> </p>
    <p>     if (r.state != BroadcastRecord.IDLE)   return;</p>
    <p>     //如果下面这个if条件满足，则表示该条广播要么已经全部被处理，要么被中途取消</p>
    <p>     if(r.receivers == null || r.nextReceiver &gt;= numReceivers</p>
    <p>            || r.resultAbort || forceReceive) {</p>
    <p>         if(r.resultTo != null) {</p>
    <p>          try {</p>
    <p>               //将该广播的处理结果传给设置了resultTo的接收者</p>
    <p>               performReceiveLocked(r.callerApp, r.resultTo,</p>
    <p>                  new Intent(r.intent), r.resultCode,</p>
    <p>                 r.resultData, r.resultExtras, false, false);</p>
    <p>               r.resultTo = null;</p>
    <p>            }......</p>
    <p>       }</p>
    <p>      cancelBroadcastTimeoutLocked();</p>
    <p>       addBroadcastToHistoryLocked(r);</p>
    <p>      mOrderedBroadcasts.remove(0);</p>
    <p>       r = null;</p>
    <p>      looped = true;</p>
    <p>       continue;</p>
    <p>     }</p>
    <p>  } while (r== null);</p>
</div>
<p>processNextBroadcast第二阶段的工作比较简单：</p>
<p>·  首先根据是否处于pending状态（mPendingBroadcast不为null）进行相关操作。读者要认真体会代码中的重要说明。</p>
<p>·  处理超时的广播记录。这个超时时间是2*BROADCAST_TIMEOUT*numReceivers。BROADCAST_TIMEOUT默认为10秒。由于涉及创建进程，初始化Android运行环境等重体力活，故此处超时时间还乘以一个固定倍数2。</p>
<h4>3.  processNextBroadcast分析之三</h4>
<p>来看processNextBroadcast第三阶段的工作，代码如下：</p>
<p>[--&gt;ActivityManagerService.java::processNextBroadcast]</p>
<div>
    <p>  int recIdx= r.nextReceiver++;</p>
    <p> r.receiverTime = SystemClock.uptimeMillis();</p>
    <p>  if (recIdx== 0) {</p>
    <p>     r.dispatchTime = r.receiverTime;//记录本广播第一次处理开始的时间</p>
    <p>      r.dispatchClockTime= System.currentTimeMillis();</p>
    <p>  }</p>
    <p>  //设置广播处理超时时间，BROADCAST_TIMEOUT为10秒</p>
    <p>  if (!mPendingBroadcastTimeoutMessage) {</p>
    <p>        longtimeoutTime = r.receiverTime + BROADCAST_TIMEOUT;</p>
    <p>       setBroadcastTimeoutLocked(timeoutTime);</p>
    <p>  }</p>
    <p>  //取该条广播下一个接收者</p>
    <p>  ObjectnextReceiver = r.receivers.get(recIdx);</p>
    <p>  if(nextReceiver instanceof BroadcastFilter) {</p>
    <p>     //如果是动态接收者，则直接调用deliverToRegisteredReceiverLocked处理</p>
    <p>    BroadcastFilter filter = (BroadcastFilter)nextReceiver;</p>
    <p>    deliverToRegisteredReceiverLocked(r, filter, r.ordered);</p>
    <p>     if(r.receiver == null || !r.ordered) {</p>
    <p>        r.state = BroadcastRecord.IDLE;</p>
    <p>        scheduleBroadcastsLocked();</p>
    <p>       }</p>
    <p>     return;//已经通知一个接收者去处理该广播，需要等它的处理结果，所以此处直接返回</p>
    <p>  }</p>
    <p> //如果是静态接收者，则它的真实类型是ResolveInfo</p>
    <p> ResolveInfo info =  (ResolveInfo)nextReceiver;</p>
    <p>  booleanskip = false;</p>
    <p>  //检查权限</p>
    <p>  int perm =checkComponentPermission(info.activityInfo.permission,</p>
    <p>        r.callingPid, r.callingUid, info.activityInfo.applicationInfo.uid,</p>
    <p>        info.activityInfo.exported);</p>
    <p>  if (perm!= PackageManager.PERMISSION_GRANTED) skip = true;</p>
    <p>  if(info.activityInfo.applicationInfo.uid != Process.SYSTEM_UID &amp;&amp;</p>
    <p>        r.requiredPermission != null) {</p>
    <p>        ......</p>
    <p>       }</p>
    <p>   //设置skip为true</p>
    <p>   if(r.curApp != null &amp;&amp; r.curApp.crashing) skip = true;</p>
    <p> </p>
    <p>   if (skip){</p>
    <p>     r.receiver = null;</p>
    <p>     r.curFilter = null;</p>
    <p>     r.state = BroadcastRecord.IDLE;</p>
    <p>     scheduleBroadcastsLocked();//再调度一次广播处理</p>
    <p>     return;</p>
    <p>   }</p>
    <p> </p>
    <p>    r.state= BroadcastRecord.APP_RECEIVE;</p>
    <p>    StringtargetProcess = info.activityInfo.processName;</p>
    <p>   r.curComponent = new ComponentName(</p>
    <p>             info.activityInfo.applicationInfo.packageName,</p>
    <p>             info.activityInfo.name);</p>
    <p>  r.curReceiver = info.activityInfo;</p>
    <p>   try {</p>
    <p>          //设置Package stopped的状态为false</p>
    <p>          AppGlobals.getPackageManager().setPackageStoppedState(</p>
    <p>                       r.curComponent.getPackageName(),false);</p>
    <p>         } ......</p>
    <p> </p>
    <p>  ProcessRecord app = getProcessRecordLocked(targetProcess,</p>
    <p>                         info.activityInfo.applicationInfo.uid);</p>
    <p>   //如果该接收者对应的进程已经存在</p>
    <p>   if (app!= null &amp;&amp; app.thread != null) {</p>
    <p>    try {</p>
    <p>           app.addPackage(info.activityInfo.packageName);</p>
    <p>           //该函数内部调用应用进程的scheduleReceiver函数，读者可自行分析</p>
    <p>           processCurBroadcastLocked(r, app);</p>
    <p>           return;//已经触发该接收者处理本广播，需要等待处理结果</p>
    <p>        }......</p>
    <p>     }</p>
    <p>   //最糟的情况就是该进程还没有启动</p>
    <p>   if((r.curApp=startProcessLocked(targetProcess,</p>
    <p>          info.activityInfo.applicationInfo, true,......)!= 0)) == null) {</p>
    <p>          ......//进程启动失败的处理</p>
    <p>         return;</p>
    <p>   }</p>
    <p>  mPendingBroadcast = r; //设置mPendingBroadcast</p>
    <p>  mPendingBroadcastRecvIndex = recIdx;</p>
    <p> }</p>
</div>
<p>对processNextBroadcast第三阶段的工作总结如下：</p>
<p>·  如果广播接收者为动态注册对象，则直接调用deliverToRegisteredReceiverLocked处理它。</p>
<p>·  如果广播接收者为静态注册对象，并且该对象对应的进程已经存在，则调用processCurBroadcastLocked处理它。</p>
<p>·  如果广播接收者为静态注册对象，并且该对象对应的进程还不存在，则需要创建该进程。这是最糟糕的情况。</p>
<p>此处，不再讨论新进程创建及Android运行环境初始化相关的逻辑，读者可返回阅读“attachApplicationLocked分析之三”，其中有处理mPendingBroadcast的内容。</p>
<h3>6.4.4   应用进程处理广播分析</h3>
<p>下面来分析当应用进程收到广播后的处理流程，以动态接收者为例。</p>
<h4>1.  ApplicationThreadscheduleRegisteredReceiver函数分析</h4>
<p>如前所述，AMS将通过scheduleRegisteredReceiver函数将广播交给应用进程，该函数代码如下：</p>
<p>[--&gt;ActivityThread.java::scheduleRegisteredReceiver]</p>
<div>
    <p>public voidscheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,</p>
    <p>       intresultCode, String dataStr, Bundle extras, boolean ordered,</p>
    <p>      boolean sticky) throws RemoteException {</p>
    <p>  //又把receiver对象传了回来。还记得注册时传递的一个IIntentReceiver类型</p>
    <p>  //的对象吗？</p>
    <p>   receiver.performReceive(intent,resultCode, dataStr, extras, ordered,</p>
    <p>                               sticky);</p>
    <p> }</p>
</div>
<p>就本例而言，receiver对象的真实类型为LoadedApk.ReceiverDispatcher，来看它的performReceive函数，代码如下：</p>
<p>[--&gt;LoadedApk.java::performReceive]</p>
<div>
    <p>public void performReceive(Intent intent, intresultCode,</p>
    <p>   Stringdata, Bundle extras, boolean ordered, boolean sticky) {</p>
    <p>   //Args是一个runnable对象</p>
    <p>   Args args= new Args(intent, resultCode, data, extras, ordered, sticky);</p>
    <p>      // mActivityThread是一个Handler，还记得SDK提供的两个同名的registerReceiver</p>
    <p>     //函数吗？如果没有传递Handler，则使用主线程的Handler。</p>
    <p>  if(!mActivityThread.post(args)) {</p>
    <p>      if(mRegistered &amp;&amp; ordered) {</p>
    <p>        IActivityManager mgr = ActivityManagerNative.getDefault();</p>
    <p>        args.sendFinished(mgr);</p>
    <p>      }</p>
    <p>   }</p>
    <p>}</p>
</div>
<p>scheduleRegisteredReceiver最终向主线程的Handler投递了一个Args对象，这个对象的run函数将在主线程中被调用。</p>
<h4>2.  Args.run分析</h4>
<p>[--&gt;LoadedApk.java::Args.run]</p>
<div>
    <p>   publicvoid run() {</p>
    <p>     finalBroadcastReceiver receiver = mReceiver;</p>
    <p>     final boolean ordered = mOrdered;</p>
    <p>     finalIActivityManager mgr = ActivityManagerNative.getDefault();</p>
    <p>     finalIntent intent = mCurIntent;</p>
    <p>    mCurIntent = null;</p>
    <p>     ......</p>
    <p>    try {</p>
    <p>          //获取ClassLoader对象，千万注意，此处并没有通过反射机制创建一个广播接收者，</p>
    <p>          //对于动态接收者来说，在注册前就已经创建完毕</p>
    <p>           ClassLoadercl = mReceiver.getClass().getClassLoader();</p>
    <p>          intent.setExtrasClassLoader(cl);</p>
    <p>          setExtrasClassLoader(cl);</p>
    <p>          receiver.setPendingResult(this);//设置pendingResult</p>
    <p>          //调用该动态接收者的onReceive函数</p>
    <p>          receiver.onReceive(mContext, intent);</p>
    <p>    }......</p>
    <p>     //调用finish完成工作</p>
    <p>     if(receiver.getPendingResult() != null) finish();</p>
    <p>    }</p>
</div>
<p>Finish的代码很简单，此处不在赘述，在其内部会通过sendFinished函数调用AMS的finishReceiver函数，以通知AMS。</p>
<h4>3.  AMS的finishReceiver函数分析</h4>
<p>不论ordered还是非orded广播，AMS的finishReceiver函数都会被调用，它的代码如下：</p>
<p>[--&gt;ActivityManagerService.java::finishReceiver]</p>
<div>
    <p>public void finishReceiver(IBinder who, intresultCode, String resultData,</p>
    <p>           Bundle resultExtras, boolean resultAbort) {</p>
    <p>      ......</p>
    <p>    booleandoNext;</p>
    <p>    final long origId =Binder.clearCallingIdentity();</p>
    <p>    synchronized(this){</p>
    <p>        //判断是否还需要继续调度后续的广播发送</p>
    <p>       doNext = finishReceiverLocked(</p>
    <p>             who, resultCode, resultData, resultExtras, resultAbort, true);</p>
    <p>        }</p>
    <p>        if(doNext) { //发起下一次广播发送</p>
    <p>           processNextBroadcast(false);</p>
    <p>        }</p>
    <p>       trimApplications();</p>
    <p>   Binder.restoreCallingIdentity(origId);</p>
    <p>}</p>
</div>
<p>由以上代码可知，finishReceiver将根据情况调度下一次广播发送。</p>
<h3>6.4.5  广播处理总结</h3>
<p>广播处理的流程及相关知识点还算比较简单，可以用图6-20来表示本例的流程。</p>

![image](images/chapter6/image020.png)

<p>图6-20  Broadcast处理流程</p>
<p>在图6-20中，将调用函数所属的实际对象类型标注了出来，其中第11步的MyBroadcastReceiver为本例中所注册的广播接收者。</p>
<div>
    <p><strong>注意</strong>任何广播对于静态注册者来说，都是ordered，而且该order是全局性的，并非只针对该广播的接收者，故从广播发出到静态注册者的onReceive函数被调用中间经历的这段时间相对较长。</p>
</div>
<p> </p>
<h2>6.5  startService之按图索骥</h2>
<p>Service是Android的四大组件之一。和Activity，BroadcastReceiver相比，Service定位于业务层逻辑处理，而Activity定位于前端UI层逻辑处理，BroadcastReceiver定位于通知逻辑的处理。</p>
<p>做为业务服务提供者，Service自有一套规则，先来看有关Service的介绍。</p>
<h3>6.5.1  Service知识介绍</h3>
<p>四大组件之一的Service，其定义非常符合C/S架构中Service的概念，即为Client服务，处理Client的请求。在Android中，目前接触最多的是Binder中的C/S架构。在这种架构中，Client通过调用预先定义好的业务函数向对应的Service发送请求。作为四大组件之一Service，其响应Client的请求方式有两种：</p>
<p>·  Client通过调用startService向Service端发送一个Intent，该Intent携带请求信息。而Service的onStartCommand会接受该Intent，并处理之。该方式是Android平台特有的，借助Intent来传递请求。</p>
<p>·  Client调用bindService函数和一个指定的Service建立Binder关系，即绑定成功后，Client端将得到处理业务逻辑的Binder Bp端。此后Client直接调用Bp端提供的业务函数向Service端发出请求。注意，在这种方式中，Service的onBind函数被调用，如果该Service支持Binder，需返回一个IBinder对象给客户端。</p>
<p>以上介绍的是Service响应客户端请求的两种方式，务必将两者分清楚。此外，这两种方式还影响Service对象的生命周期，简单总结如下：</p>
<p>·  对于以startService方式启动的Service对象，其生命周期一直延续到stopSelf或stopService被调用为止。</p>
<p>·  对于以bindService方式启动的Service对象，其生命周期延续到最后一个客户端调用完unbindService为止。</p>
<div>
    <p><strong>注意</strong>生命周期控制一般都涉及引用计数的使用。如果某Service对象同时支持这两种请求方式，那么当总引用计数减为零时，其生命就走向终点。</p>
</div>
<p>和Service相关的知识还有，当系统内存不足时，系统如何处理Service。如果Service和UI某个部分绑定（例如类似通知栏中Music播放的信息），那么此Service优先级较高（可通过调用startForeground把自己变成一个前台Service），系统不会轻易杀死这些Service来回收内存。</p>
<p>以上这些内容都较简单，阅读SDK文档中Service的相关说明即可了解，具体路径为SDK路径/docs/guide/topics/fundamentals/services.html。</p>
<p> </p>
<h3>6.5.2  startService流程图</h3>
<p>本章不过多介绍和Service相关的知识，原因有二：</p>
<p>·  Service的处理流程和本章重点介绍的Activity的处理流程差不多，并且Service的处理逻辑更简单。能阅读到此处的读者，想必对拿下Service信心满满。</p>
<p>·  “授人以鱼，不如授人以渔”。希望读者在经历过如此大量而又复杂的代码分析考验后，能学会和掌握分析方法，因此本节将以startService为分析对象，把相关的流程图描绘出来，旨在帮读者根据该流程图自行研读与Service相关的处理逻辑。</p>
<p>startService调用轨迹如图6-21和图6-22所示。</p>

![image](images/chapter6/image021.png)

<p>图6-21  startService流程图之一</p>
<p>图6-21列出了和startService相关的调用流程。在这个流程中，可假设Service所对应的进程已经存在。</p>
<p>单独提取图6-21中Service所在进程对H.CREATE_SERVICE等消息的处理流程如图6-22所示。</p>

![image](images/chapter6/image022.png)

<p>图6-22  startService中相关Message的处理流程</p>
<div>
    <p><strong>注意</strong>图6-21和图6-22中也包含了bindService的处理流程。在实际分析时，读者可分开研究bindService和startService的处理流程。</p>
</div>
<h2>6.6  AMS中的进程管理</h2>
<p>前面曾反复提到，Android平台中很少能接触到进程的概念，取而代之的是有明确定义的四大组件。但是作为运行在Linux用户空间内的一个系统或框架，Android不仅不能脱离进程，反而要大力利用Linux OS提供的进程管理机制和手段，更好地为自己服务。作为Android平台中组件运行管理的核心服务，ActivityManagerService当仁不让地接手了这方面的工作。目前，AMS对进程的管理仅涉及两个方面：</p>
<p>·  调节进程的调度优先级和调度策略。</p>
<p>·  调节进程的OOM值。</p>
<p>先来看在Linux OS中这两方面的进程管理和控制手段。</p>
<h3>6.6.1  Linux进程管理介绍<a>[⑥]</a></h3>
<h4>1.  Linux进程调度优先级和调度策略</h4>
<p>调度优先级和调度策略是操作系统中一个很重要的概念。简而言之，它是系统中CPU资源的管理和控制手段。如何理解？此处进行简单介绍。读者可自行阅读操作系统方面的书籍以加深理解。</p>
<p>·  相对于在OS上运行的应用进程个数来说，CPU的资源非常有限。</p>
<p>·  调度优先级是OS分配CPU资源给应用进程时（即调度应用进程运行）需要参考的一个指标。一般而言，优先级高的进程将更有机会得到CPU资源。</p>
<p>调度策略用于描述OS调度模块分配CPU给应用进程所遵循的规则，即当将CPU控制权交给调度模块时，系统如何选择下一个要运行的进程。此处不能仅考虑各进程的调度优先级，因为存在多个进程具有相同调度优先级的情况。在这种情况下，一个需要考虑的重要因素是，每个进程所分配的时间片及它们的使用情况.</p>
<div>
    <p><strong>提示</strong>可简单认为，调度优先级及调度策略二者一起影响了系统分配CPU资源给应用进程。注意，此处的措词为“影响”，而非“控制”。因为对于用户空间可以调用的API来说，如果二者能控制CPU资源分配，那么该系统的安全性会大打折扣。</p>
</div>
<p>Linux提供了两个API用于设置调度优先级及调度策略。先来看设置调度优先级的函数setpriority，其原型如下：</p>
<div>
    <p>int setpriority(int which, int who, int prio);</p>
</div>
<p>其中：</p>
<p>·  which和who参数联合使用。当which为PRIO_PROGRESS时，who代表一个进程；当which为PRIO_PGROUP时，who代表一个进程组；当which为PRIO_USER时，who代表一个uid。</p>
<p>·  第三个参数prio用于设置应用进程的nicer值，可取范围从-20到19。Linux kernel用nicer值来描述进程的调度优先级，该值越大，表明该进程越友（nice），其被调度运行的几率越低。</p>
<p>下面来看设置调度策略的函数sched_setscheduler，其原型如下：</p>
<div>
    <p>int sched_setscheduler(pid_t pid, int policy,conststruct sched_param *param);</p>
</div>
<p>其中：</p>
<p>·  第一个参数为进程id。</p>
<p>·  第二个参数为调度策略。目前Android支持三种调度策略：SCHED_OTHER，标准round-robin分时共享策略(也就是默认的策略)；SCHED_BATCH，针对具有batch风格（批处理）进程的调度策略；SCHED_IDLE，针对优先级非常低的适合在后台运行的进程。除此之外，Linux还支持实时（Real-time）调度策略，包括SCHED_FIFO，先入先出调度策略；SCHED_RR，：round-robin调度策略，也就是循环调度。</p>
<p>·  param参数中最重要的是该结构体中的sched_priority变量。针对Android中的三种非实时调度策略，该值必须为NULL。</p>
<p>以上介绍了调度优先级和调度策略的概念。建议读者做个小实验来测试调动优先级及调动策略的作用，步骤如下：</p>
<p>·  挂载SD卡到PC机并往向其中复制一些媒体文件，然后重新挂载SD卡到手机。该操作就能触发MediaScanner扫描新增的这些媒体文件。</p>
<p>·  利用top命令查看CPU使用率，会发现进程com.process.media（即MediaScanner所在的进程）占用CPU较高（可达70%以上），原因是该进程会扫描媒体文件，该工作将利用较多的CPU资源。</p>
<p>·  此时，如果启动另一个进程，然后做一些操作，会发现MediaScanner所在进程的CPU利用率会降下来（例如下降到30%~40%），这表明系统将CPU资源更多地分给了用户正在操作的这个进程。</p>
<p>出现这种现象的原因是，MediaScannerSerivce的扫描线程将调度优先级设置为11，而默认的调度优先级为0。 相比而言，MediaScannerService优先级真的很高。</p>
<h4>2.  关于Linux进程oom_adj的介绍</h4>
<p>从Linux kernel 2.6.11开始，内核提供了进程的OOM控制机制，目的是当系统出现内存不足（out of memory，简称OOM）的情况时，Kernel可根据进程的oom_adj来选择并杀死一些进程，以回收内存。简而言之，oom_adj可标示Linux进程内存资源的优先级，其可取范围从-16到15，另外有一个特殊值-17用于禁止系统在OOM情况下杀死该进程。和nicer值一样，oom_adj的值越高，那么在OOM情况下，该进程越有可能被杀掉。每个进程的oom_adj初值为0。</p>
<p>Linux没有提供单独的API用于设置进程的oom_adj。目前的做法就是向/proc/进程id/oom_adj文件中写入对应的oom_adj值，通过这种方式就能调节进程的oom_adj了。</p>
<p>另外，有必要简单介绍一下Android为Linux Kernel新增的lowmemorykiller（以后简称LMK）模块的工作方式：LMK的职责是根据当前内存大小去杀死对应oom_adj及以上的进程以回收内存。这里有两个关键参数：为LMK设置不同的内存阈值及oom_adj，它们分别由/sys/module/lowmemorykiller/parameters/minfree和/sys/module/lowmemorykiller/parameters/adj控制。</p>
<div>
    <p><strong>注意</strong>这两个参数的典型设置为：</p>
    <p>minfree，2048,3072,4096,6144,7168,8192 用于描述不同级别的内存阈值，单位为KB。</p>
    <p>adj，0,1,2,4,7,15 用于描述对应内存阈值的oom_adj值。</p>
    <p>表示当剩余内存为2048KB时，LMK将杀死oom_adj大于等于0的进程。</p>
</div>
<p>网络上有一些关于Android手机内存优化的方法，其中一种就利用了LMK的工作原理。</p>
<div>
    <p><strong>提示</strong>lowmemorykiller的代码在kernel/drivers/staging/android/lowmemorykiller.c中，感兴趣的读者可尝试自行阅读。</p>
</div>
<h3>6.6.2  关于Android中的进程管理的介绍</h3>
<p>前面介绍了Linux OS中进程管理（包括调度和OOM控制）方面的API，但AMS是如何利用它们的呢？这就涉及AMS中的进程管理规则了。这里简单介绍相关规则。</p>
<p>Android将应用进程分为五大类，分别为Forground类、Visible类、Service类、Background类及Empty类。这五大类的划分各有规则。</p>
<h4>1.  进程分类</h4>
<h5>（1） Forground类</h5>
<p>该类中的进程重要性最高，属于该类的进程包括下面几种情况：</p>
<p>·  含一个前端Activity（即onResume函数被调用过了，或者说当前正在显示的那个Activity）。</p>
<p>·  含一个Service，并且该Service和一个前端Activity绑定（例如Music应用包括一个前端界面和一个播放Service，当我们一边听歌一边操作Music界面时，该Service即和一个前端Activity绑定）。</p>
<p>·  含一个调用了startForground的Service，或者该进程的Service正在调用其生命周期的函数（onCreate、onStart或onDestroy）。</p>
<p>·  最后一种情况是，该进程中有BroadcastReceiver实例正在执行onReceive函数。</p>
<h5>（2） Visible类</h5>
<p>属于Visible类的进程中没有处于前端的组件，但是用户仍然能看到它们，例如位于一个对话框后的Activity界面。目前该类进程包括两种：</p>
<p>·  该进程包含一个仅onPause被调用的Activity（即它还在前台，只不过部分界面被遮住）。</p>
<p>·  或者包含一个Service，并且该Service和一个Visible（或Forground）的Activity绑定（从字面意义上看，这种情况不太好和Forground进程中第二种情况区分）。</p>
<h5>（3） Service类、Background类及Empty类</h5>
<p>这三类进程都没有可见的部分，具体情况如下。</p>
<p>·  Service进程：该类进程包含一个Service。此Service通过startService启动，并且不属于前面两类进程。这种进程一般在后台默默地干活，例如前面介绍的MediaScannerService。</p>
<p>·  Background进程：该类进程包含当前不可见的Activity（即它们的onStop被调用过）。系统保存这些进程到一个LRU（最近最少使用）列表。当系统需要回收内存时，该列表中那些最近最少使用的进程将被杀死。</p>
<p>·  Empty进程：这类进程中不包含任何组件。为什么会出现这种不包括任何组件的进程呢？其实很简单，假设该进程仅创建了一个Activity，它完成工作后主动调用finish函数销毁（destroy）自己，之后该进程就会成为Empty进程。系统保留Empty进程的原因是当又重新需要它们时（例如用户在别的进程中通过startActivity启动了它们），可以省去fork进程、创建Android运行环境等一系列漫长而艰苦的工作。</p>
<p>通过以上介绍可发现，当某个进程和前端显示有关系时，其重要性相对要高，这或许是体现Google重视用户体验的一个很直接的证据吧。</p>
<div>
    <p><strong>建议</strong>读者可阅读SDK/docs/guide/topics/fundamentals/processes-and-threads.html以获取更为详细的信息。</p>
</div>
<h4>2.  Process类API介绍</h4>
<p>我们先来介绍Android平台中进程调度和OOM控制的API，它们统一被封装在Process.java中，其相关代码如下：</p>
<p>[--&gt;Process.java]</p>
<div>
    <p>//设置线程的调度优先级，Linux kernel并不区分线程和进程，二者对应同一个数据结构Task</p>
    <p>public static final native void setThreadPriority(inttid, int priority)</p>
    <p>           throws IllegalArgumentException, SecurityException;</p>
    <p> </p>
    <p>/*</p>
    <p>   设置线程的Group，实际上就是设置线程的调度策略，目前Android定义了三种Group。</p>
    <p>   由于缺乏相关资料，加之笔者感觉对应的注释也只可意会不可言传，故此处直接照搬了英文</p>
    <p>   注释，敬请读者谅解。</p>
    <p>   <strong>THREAD_GROUP_DEFAULT</strong>：Default thread group - gets a 'normal'share of the CPU</p>
    <p>   <strong>THREAD_GROUP_BG_NONINTERACTIVE</strong>：Background non-interactive thread group.</p>
    <p>   Allthreads in this group are scheduled with a reduced share of the CPU</p>
    <p>   <strong>THREAD_GROUP_FG_BOOST</strong>：Foreground 'boost' thread group - Allthreads in</p>
    <p>   this group are scheduled with an increasedshare of the CPU.</p>
    <p>   目前代码中还没有地方使用THREAD_GROUP_FG_BOOST这种Group</p>
    <p>*/</p>
    <p>public static final native void setThreadGroup(inttid, int group)</p>
    <p>           throws IllegalArgumentException, SecurityException;</p>
    <p>//设置进程的调度策略，包括该进程的所有线程</p>
    <p>public static final native voidsetProcessGroup(int pid, int group)</p>
    <p>           throws IllegalArgumentException, SecurityException;</p>
    <p>//设置线程的调度优先级</p>
    <p>public static final native voidsetThreadPriority(int priority)</p>
    <p>           throws IllegalArgumentException, SecurityException;</p>
    <p>//调整进程的oom_adj值</p>
    <p>public static final native boolean setOomAdj(intpid, int amt);</p>
</div>
<p>Process类还为不同调度优先级定义一些非常直观的名字以避免在代码中直接使用整型，例如为最低的调度优先级19定义了整型变量THREAD_PRIORITY_LOWEST。除此之外，Process还提供了fork子进程等相关的函数。</p>
<div>
    <p><strong>注意</strong>Process.java中的大多数函数是由JNI层实现的，其中Android在调度策略设置这一功能上还有一些特殊的地方，感兴趣的读者不妨阅读system/core/libcutils/sched_policy.c文件。</p>
</div>
<h4>3.  关于ProcessList类和ProcessRecord类的介绍</h4>
<h5>（1） ProcessList类的介绍</h5>
<p>ProcessList类有两个主要功能：</p>
<p>·  定义一些成员变量，这些成员变量描述了不同状态下进程的oom_adj值。</p>
<p>·  在Android 4.0之后，LMK的配置参数由ProcessList综合考虑手机总内存大小和屏幕尺寸后再行设置（在Android 2.3中，LMK的配置参数在init.rc中由init进程设置，并且没有考虑屏幕尺寸的影响）。读者可自行阅读其updateOomLevels函数，此处不再赘述。</p>
<p>本节主要关注ProcessList对oom_adj的定义。虽然前面介绍时将Android进程分为五大类，但是在实际代码中的划分更为细致，考虑得更为周全。</p>
<p>[--&gt;ProcessList.java]</p>
<div>
    <p>class ProcessList {</p>
    <p>    //当一个进程连续发生Crash的间隔小于60秒时，系统认为它是为Bad进程</p>
    <p>    staticfinal int MIN_CRASH_INTERVAL = 60*1000;</p>
    <p>   //下面定义各种状态下进程的oom_adj</p>
    <p> </p>
    <p>   //不可见进程的oom_adj，最大15，最小为7。杀死这些进程一般不会带来用户能够察觉的影响</p>
    <p>    staticfinal int HIDDEN_APP_MAX_ADJ = 15;</p>
    <p>    staticint HIDDEN_APP_MIN_ADJ = 7;</p>
    <p> </p>
    <p>   /*</p>
    <p>   位于B List中Service所在进程的oom_adj。什么样的Service会在BList中呢？其中的</p>
    <p>   解释只能用原文来表达” these are the old anddecrepit services that aren't as</p>
    <p>   shiny andinteresting as the ones in the A list“</p>
    <p>   */</p>
    <p>    staticfinal int SERVICE_B_ADJ = 8;</p>
    <p> </p>
    <p>   /*</p>
    <p>   前一个进程的oom_adj，例如从A进程的Activity跳转到位于B进程的Activity，</p>
    <p>   B进程是当前进程，而A进程就是previous进程了。因为从当前进程A退回B进程是一个很简单</p>
    <p>   却很频繁的操作（例如按back键退回上一个Activity），所以previous进程的oom_adj</p>
    <p>   需要单独控制。这里不能简单按照五大类来划分previous进程，还需要综合考虑</p>
    <p>   */</p>
    <p>   staticfinal int PREVIOUS_APP_ADJ = 7;</p>
    <p> </p>
    <p>   //Home进程的oom_adj为6，用户经常和Home进程交互，故它的oom_adj也需要单独控制</p>
    <p>    staticfinal int HOME_APP_ADJ = 6;</p>
    <p> </p>
    <p>   //Service类中进程的oom_adj为5</p>
    <p>    staticfinal int SERVICE_ADJ = 5;</p>
    <p> </p>
    <p>   //正在执行backup操作的进程，其oom_adj为4</p>
    <p>    staticfinal int BACKUP_APP_ADJ = 4;</p>
    <p> </p>
    <p>   //heavyweight的概念前面曽提到过，该类进程的oom_adj为3</p>
    <p>    staticfinal int HEAVY_WEIGHT_APP_ADJ = 3;</p>
    <p> </p>
    <p>   //Perceptible进程指那些当前并不在前端显示而用户能感觉到它在运行的进程，例如正在</p>
    <p>   //后台播放音乐的Music</p>
    <p>    staticfinal int PERCEPTIBLE_APP_ADJ = 2;</p>
    <p> </p>
    <p>   //Visible类进程，oom_adj为1</p>
    <p>    staticfinal int VISIBLE_APP_ADJ = 1;</p>
    <p> </p>
    <p>   //Forground类进程，oom_adj为0</p>
    <p>    staticfinal int FOREGROUND_APP_ADJ = 0;</p>
    <p> </p>
    <p>   //persistent类进程（即退出后，系统需要重新启动的重要进程），其oom_adj为-12</p>
    <p>    staticfinal int PERSISTENT_PROC_ADJ = -12;</p>
    <p> </p>
    <p>    //核心服务所在进程，oom_adj为-12</p>
    <p>    staticfinal int CORE_SERVER_ADJ = -12;</p>
    <p> </p>
    <p>    //系统进程，其oom_adj为-16</p>
    <p>    staticfinal int SYSTEM_ADJ = -16;</p>
    <p> </p>
    <p>    //内存页大小定义为4KB</p>
    <p>    staticfinal int PAGE_SIZE = 4*1024;</p>
    <p> </p>
    <p>    //系统中能容纳的不可见进程数最少为2，最多为15。在Android 4.0系统中，可通过设置程序来</p>
    <p>    //调整</p>
    <p>    staticfinal int MIN_HIDDEN_APPS = 2;</p>
    <p>    staticfinal int MAX_HIDDEN_APPS = 15;</p>
    <p> </p>
    <p>  //下面两个参数用于调整进程的LRU权重。注意，它们的注释和具体代码中的用法不能统一起来</p>
    <p>  staticfinal long CONTENT_APP_IDLE_OFFSET = 15*1000;</p>
    <p>  staticfinal long EMPTY_APP_IDLE_OFFSET = 120*1000;</p>
    <p> </p>
    <p>  //LMK设置了6个oom_adj阈值</p>
    <p>  privatefinal int[] mOomAdj = new int[] {</p>
    <p>           FOREGROUND_APP_ADJ, VISIBLE_APP_ADJ, PERCEPTIBLE_APP_ADJ,</p>
    <p>           BACKUP_APP_ADJ, HIDDEN_APP_MIN_ADJ, EMPTY_APP_ADJ</p>
    <p>  };</p>
    <p>   //用于内存较低机器（例如小于512MB）中LMK的内存阈值配置</p>
    <p>    privatefinal long[] mOomMinFreeLow = new long[] {</p>
    <p>           8192, 12288, 16384,</p>
    <p>           24576, 28672, 32768</p>
    <p>    };</p>
    <p>   //用于较高内存机器（例如大于1GB）的LMK内存阈值配置</p>
    <p>   privatefinal long[] mOomMinFreeHigh = new long[] {</p>
    <p>           32768, 40960, 49152,</p>
    <p>           57344, 65536, 81920</p>
    <p>    };</p>
</div>
<p>从以上代码中定义的各种ADJ值可知，AMS中的进程管理规则远比想象得要复杂（读者以后见识到具体的代码，更会有这样的体会）。</p>
<div>
    <p><strong>说明</strong>在ProcessList中定义的大部分变量在Android 2.3代码中定义于ActivityManagerService.java中，但这段代码的开发者仅把代码复制了过来，其中的注释并未随着系统升级而更新。</p>
</div>
<h5>（2） ProcessRecord中相关成员变量的介绍</h5>
<p> ProcessRecord定义了较多成员变量用于进程管理。笔者不打算深究其中的细节。这里仅把其中的主要变量及一些注释列举出来。下文会分析到它们的作用。</p>
<p>[--&gt;ProcessRecord.java]</p>
<div>
    <p>//用于LRU列表控制</p>
    <p>long lastActivityTime;   // For managing the LRU list</p>
    <p>long lruWeight;      // Weight for ordering in LRU list</p>
    <p>//和oom_adj有关</p>
    <p>int maxAdj;           // Maximum OOM adjustment for thisprocess</p>
    <p>int hiddenAdj;       // If hidden, this is the adjustment touse</p>
    <p>int curRawAdj;       // Current OOM unlimited adjustment forthis process</p>
    <p>int setRawAdj;       // Last set OOM unlimited adjustment forthis process</p>
    <p>int curAdj;           // Current OOM adjustment for thisprocess</p>
    <p>int setAdj;           // Last set OOM adjustment for thisprocess</p>
    <p>//和调度优先级有关</p>
    <p>int curSchedGroup;   // Currently desired scheduling class</p>
    <p>int setSchedGroup;   // Last set to background scheduling class</p>
    <p>//回收内存级别，见后文解释</p>
    <p>int trimMemoryLevel; // Last selected memorytrimming level</p>
    <p>//判断该进程的状态，主要和其中运行的Activity，Service有关</p>
    <p>boolean keeping;     // Actively running code sodon't kill due to that?</p>
    <p>boolean setIsForeground;    // Running foreground UI when last set?</p>
    <p>boolean foregroundServices; // Running anyservices that are foreground?</p>
    <p>boolean foregroundActivities; // Running anyactivities that are foreground?</p>
    <p>boolean systemNoUi;   // This is a system process, but notcurrently showing UI.</p>
    <p>boolean hasShownUi;  // Has UI been shown in this process since itwas started?</p>
    <p>boolean pendingUiClean;     // Want to clean up resources from showingUI?</p>
    <p>boolean hasAboveClient;     // Bound using BIND_ABOVE_CLIENT, so wantto be lower</p>
    <p>//是否处于系统BadProcess列表</p>
    <p>boolean bad;                // True if disabled in the badprocess list</p>
    <p>//描述该进程因为是否有太多后台组件而被杀死</p>
    <p>boolean killedBackground;   // True when proc has been killed due to toomany bg</p>
    <p>String waitingToKill;       // Process is waiting to be killed whenin the bg; reason</p>
    <p>//序号，每次调节进程优先级或者LRU列表位置时，这些序号都会递增</p>
    <p>int adjSeq;                 // Sequence id for identifyingoom_adj assignment cycles</p>
    <p>int lruSeq;                 // Sequence id for identifyingLRU update cycles</p>
</div>
<p>上面注释中提到了LRU（最近最少使用）一词，它和AMS另外一个用于管理应用进程ProcessRecord的数据结构有关。</p>
<div>
    <p><strong>提示</strong>进程管理和调度一向比较复杂，从ProcessRecord定义的这些变量中可见一斑。需要提醒读者的是，对这部分功能的相关说明非常少，代码读起来会感觉比较晦涩。</p>
</div>
<h3>6.6.3  AMS进程管理函数分析</h3>
<p>在AMS中，和进程管理有关的函数只要有两个，分别是updateLruProcessLocked和updateOomAdjLocked。这两个函数的调用点有多处，本节以attachApplication为切入点，尝试对它们进行分析。</p>
<div>
    <p><strong>注意</strong>AMS一共定义了3个updateOomAdjLocked函数，此处将其归为一类。</p>
</div>
<p>先回顾一下attachApplication函数被调用的情况：AMS新创建一个应用进程，该进程启动后最重要的就是调用AMS的attachApplication。</p>
<div>
    <p><strong>提示</strong>不熟悉的读者可阅读6.3.3节的第5小节。</p>
</div>
<p>其相关代码如下：</p>
<p>[--&gt;ActivityManagerService.java::attachApplicationLocked]</p>
<div>
    <p>//attachApplication主要工作由attachApplicationLocked完成，故直接分析它</p>
    <p>private final booleanattachApplicationLocked(IApplicationThread thread,</p>
    <p>                                                      int pid) {</p>
    <p> ProcessRecord app;</p>
    <p>  //根据之前的介绍的内容，AMS在创建应用进程前已经将对应的ProcessRecord保存到</p>
    <p>  //mPidsSelfLocked中了</p>
    <p>  ...... //其他一些处理</p>
    <p> </p>
    <p>  //初始化ProcessRecord中的一些成员变量</p>
    <p>  app.thread= thread;</p>
    <p>  app.curAdj= app.setAdj = -100;</p>
    <p> app.curSchedGroup = Process.THREAD_GROUP_DEFAULT;</p>
    <p>  app.setSchedGroup= Process.THREAD_GROUP_BG_NONINTERACTIVE;</p>
    <p> app.forcingToForeground = null;</p>
    <p> app.foregroundServices = false;</p>
    <p> app.hasShownUi = false;</p>
    <p>  ......</p>
    <p>  //调用应用进程的bindApplication，以初始化其内部的Android运行环境</p>
    <p>  thread.bindApplication(......);</p>
    <p>  //①调用updateLruProcessLocked函数</p>
    <p> updateLruProcessLocked(app, false, true);</p>
    <p> app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();</p>
    <p>  ......//启动Activity等操作</p>
    <p> </p>
    <p>  //②didSomething为false，则调用updateOomAdjLocked函数</p>
    <p>  if(!didSomething) {</p>
    <p>      updateOomAdjLocked();</p>
    <p> }</p>
</div>
<p>在以上这段代码中有两个重要函数调用，分别是updateLruProcessLocked和updateOomAdjLocked。</p>
<h4>1.  updateLruProcessLocked函数分析</h4>
<p>根据前文所述，我们知道了系统中所有应用进程（同时包括SystemServer）的ProcessRecord信息都保存在mPidsSelfLocked成员中。除此之外，AMS还有一个成员变量mLruProcesses也用于保存ProcessRecord。mLruProcesses的类型虽然是ArrayList，但其内部成员却是按照ProcessRecord的lruWeight大小排序的。在运行过程中，AMS会根据lruWeight的变化调整mLruProcesses成员的位置。</p>
<p>就本例而言，刚连接（attach）上的这个应用进程的ProcessRecord需要通过updateLruProcessLocked函数加入mLruProcesses数组中。来看它的代码，如下所示：</p>
<p>[--&gt;ActivityManagerService.java::updateLruProcessLocked]</p>
<div>
    <p>final void updateLruProcessLocked(ProcessRecordapp,</p>
    <p>           boolean oomAdj, boolean updateActivityTime) {</p>
    <p>  mLruSeq++;//每一次调整LRU列表，系统都会分配一个唯一的编号</p>
    <p>  updateLruProcessInternalLocked(app, oomAdj, updateActivityTime, 0);</p>
    <p> }</p>
</div>
<p>[--&gt;ActivityManagerService.java::updateLruProcessInternalLocked]</p>
<div>
    <p>private final voidupdateLruProcessInternalLocked(ProcessRecord app,</p>
    <p>           boolean oomAdj, boolean updateActivityTime, int bestPos) {</p>
    <p> </p>
    <p>  //获取app在mLruProcesses中的索引位置，对于本例而言，返回值lrui为-1 </p>
    <p>  int lrui =mLruProcesses.indexOf(app);</p>
    <p>  //如果之前有记录，则先从数组中删掉，因为此处需要重新调整位置</p>
    <p>  if (lrui&gt;= 0) mLruProcesses.remove(lrui);</p>
    <p> </p>
    <p>  //获取mLruProcesses中数组索引的最大值，从0开始</p>
    <p>  int i =mLruProcesses.size()-1;</p>
    <p>  intskipTop = 0;</p>
    <p>        </p>
    <p>  app.lruSeq= mLruSeq;  //将系统全局的lru调整编号赋给ProcessRecord的lruSeq</p>
    <p>        </p>
    <p>  //更新lastActivityTime值，其实就是获取一个时间</p>
    <p>  if(updateActivityTime) {</p>
    <p>        app.lastActivityTime =SystemClock.uptimeMillis();</p>
    <p>  }</p>
    <p> </p>
    <p>  if(app.activities.size() &gt; 0) {</p>
    <p>      //如果该app含Activity，则lruWeight为当前时间</p>
    <p>      app.lruWeight = app.lastActivityTime;</p>
    <p>   } else if(app.pubProviders.size() &gt; 0) {</p>
    <p>       /*</p>
    <p>        如果有发布的ContentProvider，则lruWeight要减去一个OFFSET。</p>
    <p>        对此的理解需结合CONTENT_APP_IDLE_OFFSET的定义。读者暂时把它</p>
    <p>        看做一个常数</p>
    <p>        */</p>
    <p>       app.lruWeight = app.lastActivityTime -</p>
    <p>                            ProcessList.CONTENT_APP_IDLE_OFFSET;</p>
    <p>        //设置skipTop。这个变量实际上没有用，放在此处让人很头疼</p>
    <p>        skipTop = ProcessList.MIN_HIDDEN_APPS;</p>
    <p>   } else {</p>
    <p>         app.lruWeight = app.lastActivityTime - </p>
    <p>                              ProcessList.EMPTY_APP_IDLE_OFFSET;</p>
    <p>         skipTop = ProcessList.MIN_HIDDEN_APPS;</p>
    <p>   }</p>
    <p> </p>
    <p>  //从数组最后一个元素开始循环</p>
    <p>  while (i&gt;= 0) {</p>
    <p>     ProcessRecord p = mLruProcesses.get(i);</p>
    <p> </p>
    <p>     //下面这个if语句没有任何意义，因为skipTop除了做自减操作外，不影响其他任何内容</p>
    <p>      if(skipTop &gt; 0 &amp;&amp; p.setAdj &gt;= ProcessList.HIDDEN_APP_MIN_ADJ) {</p>
    <p>          skipTop--;</p>
    <p>       }</p>
    <p>       //将app调整到合适的位置</p>
    <p>       if(p.lruWeight &lt;= app.lruWeight || i &lt; bestPos) {</p>
    <p>           mLruProcesses.add(i+1, app);</p>
    <p>           break;</p>
    <p>       }</p>
    <p>       i--;</p>
    <p>  }</p>
    <p>  //如果没有找到合适的位置，则把app加到队列头</p>
    <p>  if (i &lt;0)   mLruProcesses.add(0, app);</p>
    <p> </p>
    <p>   //如果该将app 绑定到其他service，则要对应调整Service所在进程的LRU</p>
    <p>   if (app.connections.size() &gt; 0) {</p>
    <p>       for(ConnectionRecord cr : app.connections) {</p>
    <p>          if(cr.binding != null &amp;&amp; cr.binding.service != null</p>
    <p>                        &amp;&amp; cr.binding.service.app!= null</p>
    <p>                        &amp;&amp;cr.binding.service.app.lruSeq != mLruSeq) {</p>
    <p>                        updateLruProcessInternalLocked(cr.binding.service.app,</p>
    <p>                               oomAdj,updateActivityTime,i+1);</p>
    <p>            }</p>
    <p>        }</p>
    <p>    }</p>
    <p>  //conProviders也是一种Provider，相关信息下一章再介绍</p>
    <p>    if(app.conProviders.size() &gt; 0) {</p>
    <p>         for(ContentProviderRecord cpr : app.conProviders.keySet()) {</p>
    <p>            ......//对ContentProvider所在进程做类似的调整</p>
    <p> </p>
    <p>           }</p>
    <p>        }</p>
    <p>  //在本例中，oomAdj为false，故updateOomAdjLocked不会被调用</p>
    <p>  if (oomAdj) updateOomAdjLocked(); //以后分析</p>
    <p>}</p>
</div>
<p>从以上代码可知，updateLruProcessLocked的主要工作是根据app的lruWeight值调整它在数组中的位置。lruWeight值越大，其在数组中的位置就越靠后。如果该app和某些Service（仅考虑通过bindService建立关系的那些Service）或ContentProvider有交互关系，那么这些Service或ContentProvider所在的进程也需要调节lruWeight值。</p>
<p>下面介绍第二个重要函数updateOomAdjLocked。</p>
<div>
    <p><strong>提示</strong>在以上代码中， skipTop变量完全没有实际作用，却给为阅读代码带来了很大干扰。</p>
</div>
<p> </p>
<h4>2.  updateOomAdjLocked函数分析</h4>
<h5>（1） updateOomAdjLocked分析之一</h5>
<p>分段来看updateOomAdjLocked函数。</p>
<p>[--&gt;ActivityManagerService.java::updateOomAdjLocked()]</p>
<div>
    <p>final void updateOomAdjLocked() {</p>
    <p>  //在一般情况下，resumedAppLocked返回 mResumedActivity，即当前正处于前台的Activity</p>
    <p>  finalActivityRecord TOP_ACT = resumedAppLocked();</p>
    <p>  //得到前台Activity所属进程的ProcessRecord信息</p>
    <p>  finalProcessRecord TOP_APP = TOP_ACT != null ? TOP_ACT.app : null;</p>
    <p> </p>
    <p>  mAdjSeq++;//oom_adj在进行调节时也会有唯一的序号</p>
    <p> </p>
    <p>  mNewNumServiceProcs= 0;</p>
    <p> </p>
    <p>  /*</p>
    <p>    下面这几句代码的作用如下：</p>
    <p>    1 根据hidden adj划分级别，一共有9个级别（即numSlots值）</p>
    <p>    2 根据mLruProcesses的成员个数计算平均落在各个级别的进程数（即factor值）。但是这里</p>
    <p>      的魔数（magic number）4却令人头疼不已。如有清楚该内容的读者，不妨让分享一下</p>
    <p>      研究结果</p>
    <p>  */ </p>
    <p>  intnumSlots = ProcessList.HIDDEN_APP_MAX_ADJ -</p>
    <p>                                          ProcessList.HIDDEN_APP_MIN_ADJ + 1;</p>
    <p>  int factor= (mLruProcesses.size()-4)/numSlots;</p>
    <p>  if (factor&lt; 1) factor = 1;</p>
    <p> </p>
    <p> </p>
    <p>  int step =0;</p>
    <p>  intnumHidden = 0;</p>
    <p>        </p>
    <p>  int i =mLruProcesses.size();</p>
    <p>  intcurHiddenAdj = ProcessList.HIDDEN_APP_MIN_ADJ;</p>
    <p> //从mLruProcesses数组末端开始循环</p>
    <p>  while (i&gt; 0) {</p>
    <p>     i--;</p>
    <p>     ProcessRecordapp = mLruProcesses.get(i);</p>
    <p> </p>
    <p>     //①调用另外一个updateOomAdjLocked函数</p>
    <p>     updateOomAdjLocked(app,curHiddenAdj, TOP_APP, true);</p>
    <p> </p>
    <p>     // updateOomAdjLocked函数会更新app的curAdj</p>
    <p>    if(curHiddenAdj &lt; ProcessList.HIDDEN_APP_MAX_ADJ</p>
    <p>                              &amp;&amp;app.curAdj == curHiddenAdj) {</p>
    <p>       /*</p>
    <p>       这段代码的目的其实很简单。即当某个adj级别的ProcessRecord处理个数超过均值后，</p>
    <p>       就跳到下一级别进行处理。注意，这段代码的结果会影响updateOomAdjLocked的第二个参数</p>
    <p>      */</p>
    <p>           step++;</p>
    <p>           if(step &gt;= factor) {</p>
    <p>               step = 0;</p>
    <p>               curHiddenAdj++;</p>
    <p>           }</p>
    <p>       }// if(curHiddenAdj &lt; ProcessList.HIDDEN_APP_MAX_ADJ...)判断结束</p>
    <p> </p>
    <p>     //app.killedBackground初值为false</p>
    <p>     if(!app.killedBackground) {</p>
    <p>          if (app.curAdj &gt;= ProcessList.HIDDEN_APP_MIN_ADJ) {</p>
    <p>                numHidden++;</p>
    <p>              //mProcessLimit初始值为ProcessList.MAX（值为15），</p>
    <p>             //可通过setProcessLimit函数对其进行修改</p>
    <p>                if (numHidden &gt; mProcessLimit) {</p>
    <p>                      app.killedBackground =true;</p>
    <p>                      //如果后台进程个数超过限制，则会杀死对应的后台进程</p>
    <p>                     Process.killProcessQuiet(app.pid);</p>
    <p>                } </p>
    <p>            }</p>
    <p>      }//if(!app.killedBackground)判断结束</p>
    <p>  }//while循环结束</p>
</div>
<p>updateOomAdjLocked第一阶段的工作看起来很简单，但是其中也包含一些较难理解的内容。</p>
<p>·  处理hidden adj，划分9个级别。</p>
<p>·  根据mLruProcesses中进程个数计算每个级别平均会存在多少进程。在这个计算过程中出现了一个魔数4令人极度费解。</p>
<p>·  然后利用一个循环从mLruProcesses末端开始对每个进程执行另一个updateOomAdjLocked函数。关于这个函数的内容，我们放到下一节再讨论。</p>
<p>·  判断处于Hidden状态的进程数是否超过限制，如果超过限制，则会杀死一些进程。</p>
<p>接着来看updateOomAdjLocked下一阶段的工作。</p>
<h5>（2） updateOomAdjLocked分析之二</h5>
<p>[--&gt;ActivityManagerService.java::updateOomAdjLocked]</p>
<div>
    <p> mNumServiceProcs = mNewNumServiceProcs;</p>
    <p> //numHidden表示处于hidden状态的进程个数</p>
    <p>  //当Hidden进程个数小于7时候（15/2的整型值）,执行if分支</p>
    <p>  if(numHidden &lt;= (ProcessList.MAX_HIDDEN_APPS/2)) {</p>
    <p>       ......</p>
    <p>     /*</p>
    <p>      我们不讨论这段缺乏文档及使用魔数的代码，但这里有个知识点要注意：</p>
    <p>      该知识点和Android 4.0新增接口ComponentCallbacks2有关，主要是通知应用进程进行</p>
    <p>      内存清理，ComponentCallbacks2接口定了一个函数onTrimMemory(int level)，</p>
    <p>      而四大组件除BroadcastReceiver外，均实现了该接口。系统定义了4个level以通知进程</p>
    <p>      做对应处理： </p>
    <p>      <strong>TRIM_MEMORY_UI_HIDDEN</strong>，提示进程当前不处于前台，故可释放一些UI资源</p>
    <p>      <strong>TRIM_MEMORY_BACKGROUND</strong>，表明该进程已加入LRU列表，此时进程可以对一些简单的资源</p>
    <p>      进行清理</p>
    <p>      <strong>TRIM_MEMORY_MODERATE</strong>，提示进程可以释放一些资源，这样其他进程的日子会好过些。</p>
    <p>      即所谓的“我为人人，人人为我”</p>
    <p>      <strong>TRIM_MEMORY_COMPLETE</strong>，该进程需尽可能释放一些资源，否则当内存不足时，它可能被杀死</p>
    <p>      */</p>
    <p>   } else {//假设hidden进程数超过7，</p>
    <p>      finalint N = mLruProcesses.size();</p>
    <p>      for(i=0; i&lt;N; i++) {</p>
    <p>       ProcessRecord app = mLruProcesses.get(i);</p>
    <p>        if ((app.curAdj &gt;ProcessList.VISIBLE_APP_ADJ || app.systemNoUi)</p>
    <p>              &amp;&amp; app.pendingUiClean) {</p>
    <p>            if (app.trimMemoryLevel &lt; </p>
    <p>                        ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN</p>
    <p>                        &amp;&amp; app.thread!= null) {</p>
    <p>                   try {//调用应用进程ApplicationThread的scheduleTrimMemory函数</p>
    <p>                        app.thread.scheduleTrimMemory(</p>
    <p>                                    ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN);</p>
    <p>                      }......</p>
    <p>                }// if (app.trimMemoryLevel...)判断结束</p>
    <p>              app.trimMemoryLevel = </p>
    <p>                             ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN;</p>
    <p>                   app.pendingUiClean = false;</p>
    <p>         }else {</p>
    <p>              app.trimMemoryLevel = 0;</p>
    <p>          }</p>
    <p>       }//for循环结束</p>
    <p>        }//else结束</p>
    <p>   //Android4.0中设置有一个开发人员选项，其中有一项用于控制是否销毁后台的Activity。</p>
    <p>   //读者可自行研究destroyActivitiesLocked函数</p>
    <p>   if (mAlwaysFinishActivities)</p>
    <p>       mMainStack.destroyActivitiesLocked(null, false,"always-finish");</p>
    <p> }</p>
</div>
<p>通过上述代码，可获得两个信息：</p>
<p>·  Android 4.0增加了新的接口类ComponentCallbacks2，其中只定义了一个函数onTrimMemory。从以上描述中可知，它主要通知应用进程进行一定的内存释放。</p>
<p>·  Android 4.0 Settings新增了一个开放人员选项，通过它可控制AMS对后台Activity的操作。</p>
<p>这里和读者探讨一下ComponentCallbacks2接口的意义。此接口的目的是通知应用程序根据情况做一些内存释放，但笔者觉得，这种设计方案的优劣尚有待考证，主要是出于以下下几种考虑：</p>
<p>第一，不是所有应用程序都会实现该函数。原因有很多，主要原因是，该接口只是SDK 14才有的，之前的版本没有这个接口。另外，应用程序都会尽可能抢占资源（在不超过允许范围内）以保证运行速度，不应该考虑其他程序的事情。</p>
<p>第二个重要原因是无法区分在不同的level下到底要释放什么样的内存。代码中的注释也是含糊其辞。到底什么样的资源可以在TRIM_MEMORY_BACKGROUND级别下释放，什么样的资源不可以在TRIM_MEMORY_BACKGROUND级别下释放？</p>
<p>既然系统加了这些接口，读者不妨参考源码中的使用案例来开发自己的程序。</p>
<div>
    <p><strong>建议</strong>真诚希望Google能给出一个明确的文档，说明这几个函数该怎么使用。</p>
</div>
<p>接下来分析在以上代码中出现的针对每个ProcessRecord都调用的updateOomAdjLocked函数。</p>
<h4>3.  第二个updateOomAdjLocked分析</h4>
<p>[--&gt;ActivityManagerService.java::updateOomAdjLocked]</p>
<div>
    <p>private final boolean updateOomAdjLocked( ProcessRecordapp, int hiddenAdj,</p>
    <p>           ProcessRecord TOP_APP, boolean doingAll) {</p>
    <p>  //设置该app的hiddenAdj</p>
    <p> app.hiddenAdj = hiddenAdj;</p>
    <p> </p>
    <p>  if(app.thread == null)  return false;</p>
    <p> </p>
    <p>  finalboolean wasKeeping = app.keeping;</p>
    <p>  booleansuccess = true;</p>
    <p>   //下面这个函数的调用极其关键。从名字上看，它会计算该进程的oom_adj及调度策略</p>
    <p>  computeOomAdjLocked(app, hiddenAdj, TOP_APP, false, doingAll);</p>
    <p> </p>
    <p>  if(app.curRawAdj != app.setRawAdj) {</p>
    <p>      if (wasKeeping &amp;&amp; !app.keeping) {</p>
    <p>           .....//统计电量</p>
    <p>          app.lastCpuTime = app.curCpuTime;</p>
    <p>        }</p>
    <p> </p>
    <p>     app.setRawAdj = app.curRawAdj;</p>
    <p>  }</p>
    <p>  //如果新旧oom_adj不同，则重新设置该进程的oom_adj</p>
    <p>  if(app.curAdj != app.setAdj) {</p>
    <p>       if(Process.setOomAdj(app.pid, app.curAdj)) //设置该进程的oom_adj</p>
    <p>            app.setAdj = app.curAdj;</p>
    <p>       .....</p>
    <p>   }</p>
    <p>  //如果新旧调度策略不同，则需重新设置该进程的调度策略</p>
    <p>  if(app.setSchedGroup != app.curSchedGroup) {</p>
    <p>      app.setSchedGroup = app.curSchedGroup;</p>
    <p>        //waitingToKill是一个字符串，用于描述杀掉该进程的原因</p>
    <p>       if (app.waitingToKill != null &amp;&amp;</p>
    <p>            app.setSchedGroup == Process.THREAD_GROUP_BG_NONINTERACTIVE) {</p>
    <p>             Process.killProcessQuiet(app.pid);//</p>
    <p>            success = false;</p>
    <p>        } else{</p>
    <p>          if (true) {//强制执行if分支</p>
    <p>                long oldId = Binder.clearCallingIdentity();</p>
    <p>                   try {//设置进程调度策略</p>
    <p>                       Process.setProcessGroup(app.pid, app.curSchedGroup);</p>
    <p>                   }......</p>
    <p>               } ......</p>
    <p>           }</p>
    <p>        }</p>
    <p>   returnsuccess;</p>
    <p> }</p>
</div>
<p>上面的代码还算简单，主要完成两项工作：</p>
<p>·  调用computeOomAdjLocked计算获得某个进程的oom_adj和调度策略。</p>
<p>·  调整进程的调度策略和oom_adj。</p>
<div>
    <p><strong>建议</strong>思考一个问题：为何AMS只设置进程的调度策略，而不设置进程的调度优先级？</p>
</div>
<p>看来AMS调度算法的核心就在computeOomAdjLocked中。</p>
<h4>4.  computeOomAdjLocked分析</h4>
<p>这段代码较长，其核心思想是综合考虑各种情况以计算进程的oom_adj和调度策略。建议读者阅读代码时聚焦到AMS关注的几个因素上。computeOomAdjLocked的代码如下：</p>
<p>[--&gt;ActivityManagerService.java::computeOomAdjLocked]</p>
<div>
    <p>private final intcomputeOomAdjLocked(ProcessRecord app, int hiddenAdj,</p>
    <p>           ProcessRecord TOP_APP, boolean recursed, boolean doingAll) {</p>
    <p>  ......</p>
    <p>  app.adjTypeCode= ActivityManager.RunningAppProcessInfo.REASON_UNKNOWN;</p>
    <p> app.adjSource = null;</p>
    <p> app.adjTarget = null;</p>
    <p>  app.empty= false;</p>
    <p>  app.hidden= false;</p>
    <p> </p>
    <p>  //该应用进程包含Activity的个数</p>
    <p>  final intactivitiesSize = app.activities.size();</p>
    <p> </p>
    <p>  //如果maxAdj小于FOREGROUND_APP_ADJ，基本上没什么工作可以做了。这类进程优先级相当高</p>
    <p>  if(app.maxAdj &lt;= ProcessList.FOREGROUND_APP_ADJ) {</p>
    <p>       ......//读者可自行阅读这块代码</p>
    <p>      return(app.curAdj=app.maxAdj);</p>
    <p>   } </p>
    <p> </p>
    <p>  finalboolean hadForegroundActivities = app.foregroundActivities;</p>
    <p> </p>
    <p> app.foregroundActivities = false;</p>
    <p> app.keeping = false;</p>
    <p> app.systemNoUi = false;</p>
    <p> </p>
    <p>  int adj;</p>
    <p>  intschedGroup;</p>
    <p>  //如果app为前台Activity所在的那个应用进程</p>
    <p>  if (app ==TOP_APP) {</p>
    <p>       adj =ProcessList.FOREGROUND_APP_ADJ;</p>
    <p>      schedGroup = Process.THREAD_GROUP_DEFAULT;</p>
    <p>       app.adjType= "top-activity";</p>
    <p>      app.foregroundActivities = true;</p>
    <p>    } else if (app.instrumentationClass != null){</p>
    <p>             ......//略过instrumentationClass不为null的情况</p>
    <p>    } else if (app.curReceiver != null ||</p>
    <p>         (mPendingBroadcast != null &amp;&amp; mPendingBroadcast.curApp == app)){</p>
    <p>         //此情况对应正在执行onReceive函数的广播接收者所在进程，它的优先级也很高</p>
    <p>         adj = ProcessList.FOREGROUND_APP_ADJ;</p>
    <p>         schedGroup = Process.THREAD_GROUP_DEFAULT;</p>
    <p>         app.adjType = "broadcast";</p>
    <p>   } else if(app.executingServices.size() &gt; 0) {</p>
    <p>        //正在执行Service生命周期函数的进程</p>
    <p>        adj= ProcessList.FOREGROUND_APP_ADJ;</p>
    <p>       schedGroup = Process.THREAD_GROUP_DEFAULT;</p>
    <p>       app.adjType = "exec-service";</p>
    <p>    } elseif (activitiesSize &gt; 0) {</p>
    <p>         adj = hiddenAdj;</p>
    <p>        schedGroup =Process.THREAD_GROUP_BG_NONINTERACTIVE;</p>
    <p>        app.hidden = true;</p>
    <p>        app.adjType = "bg-activities";</p>
    <p>    } else {//不含任何组件的进程，即所谓的Empty进程</p>
    <p>         adj= hiddenAdj;</p>
    <p>        schedGroup = Process.THREAD_GROUP_BG_NONINTERACTIVE;</p>
    <p>         app.hidden= true;</p>
    <p>         app.empty = true;</p>
    <p>        app.adjType = "bg-empty";</p>
    <p>   }</p>
    <p>   //下面几段代码将根据情况重新调整前面计算处理的adj和schedGroup，我们以后面的</p>
    <p>   //mHomeProcess判断为例</p>
    <p>   if(!app.foregroundActivities &amp;&amp; activitiesSize &gt; 0) {</p>
    <p>         //对无前台Activity所在进程的处理</p>
    <p>   }</p>
    <p> </p>
    <p>   if (adj&gt; ProcessList.PERCEPTIBLE_APP_ADJ) {</p>
    <p>        .......</p>
    <p>    }</p>
    <p>   //如果前面计算出来的adj大于HOME_APP_ADJ，并且该进程又是Home进程，则需要重新调整</p>
    <p>   if (adj&gt; ProcessList.HOME_APP_ADJ &amp;&amp; app == mHomeProcess) {</p>
    <p>        //重新调整adj和schedGroupde的值</p>
    <p>        adj = ProcessList.HOME_APP_ADJ;</p>
    <p>       schedGroup = Process.THREAD_GROUP_BG_NONINTERACTIVE;</p>
    <p>       app.hidden = false;</p>
    <p>       app.adjType = "home";//描述调节adj的原因</p>
    <p>   }</p>
    <p>   </p>
    <p>   if (adj&gt; ProcessList.PREVIOUS_APP_ADJ &amp;&amp; app == mPreviousProcess</p>
    <p>           &amp;&amp; app.activities.size() &gt; 0) {</p>
    <p>          ......</p>
    <p>   }</p>
    <p> </p>
    <p>  app.adjSeq = mAdjSeq;</p>
    <p>  app.curRawAdj = adj;</p>
    <p> </p>
    <p>    ......</p>
    <p>   //下面这几段代码处理那些进程中含有Service，ContentProvider组件情况下的adj调节</p>
    <p>   if(app.services.size() != 0 &amp;&amp; (adj &gt; ProcessList.FOREGROUND_APP_ADJ</p>
    <p>          || schedGroup == Process.THREAD_GROUP_BG_NONINTERACTIVE)) {</p>
    <p>   }</p>
    <p> </p>
    <p>   if(s.connections.size() &gt; 0 &amp;&amp; (adj &gt;ProcessList.FOREGROUND_APP_ADJ</p>
    <p>             || schedGroup == Process.THREAD_GROUP_BG_NONINTERACTIVE)) {</p>
    <p>   }</p>
    <p> </p>
    <p>   if(app.pubProviders.size() != 0 &amp;&amp; (adj &gt;ProcessList.FOREGROUND_APP_ADJ</p>
    <p>              || schedGroup == Process.THREAD_GROUP_BG_NONINTERACTIVE)) {</p>
    <p>           ......</p>
    <p>   }</p>
    <p> </p>
    <p>   //终于计算完毕</p>
    <p>  app.curRawAdj = adj;</p>
    <p>        </p>
    <p>   if (adj&gt; app.maxAdj) {</p>
    <p>        adj= app.maxAdj;</p>
    <p>        if(app.maxAdj &lt;= ProcessList.PERCEPTIBLE_APP_ADJ) </p>
    <p>           schedGroup = Process.THREAD_GROUP_DEFAULT;</p>
    <p>    }</p>
    <p> </p>
    <p>    if (adj&lt; ProcessList.HIDDEN_APP_MIN_ADJ) </p>
    <p>        app.keeping = true;</p>
    <p>     ......</p>
    <p>   app.curAdj = adj;</p>
    <p>   app.curSchedGroup = schedGroup;</p>
    <p>   ......</p>
    <p>   returnapp.curRawAdj;</p>
    <p>}</p>
</div>
<p>computeOomAdjLocked的工作比较琐碎，实际上也谈不上什么算法，仅仅是简单地根据各种情况来设置几个值。随着系统的改进和完善，这部分代码变动的可能性比较大。</p>
<h4>5. updateOomAdjLocked调用点统计</h4>
<p>updateOomAdjLocked调用点很多，这里给出其中一个updateOomAdjLocked函数的调用点统计，如图6-23所示。</p>

![image](images/chapter6/image023.png)

<p>图6-23  updateOomAdjLocked函数的调用点统计图</p>
<p>注意，图6-23统计的是updateOomAdjLocked(ProcessRecord)函数的调用点。从该图可知，此函数被调用的地方较多，这也说明AMS非常关注应用进程的状况。</p>
<div>
    <p><strong>提示</strong>笔者觉得，AMS中这部分代码不是特别高效，不知各位读者是否有同感,？</p>
</div>
<h3>6.6.4  AMS进程管理总结</h3>
<p>本节首先向读者介绍了Linux平台中和进程调度、OOM管理方面的API，然后介绍了AMS如何利用这些API完成Android平台中进程管理方面的工作，从中可以发现，AMS设置的检查点比较密集，也就是经常会进行进程调度方面的操作。</p>
<h2>6.7  App的 Crash处理</h2>
<p>在Android平台中，应用进程fork出来后会为虚拟机设置一个未截获异常处理器，即在程序运行时，如果有任何一个线程抛出了未被截获的异常，那么该异常最终会抛给未截获异常处理器去处理。设置未截获异常处理器的代码如下：</p>
<p>[--&gt;RuntimeInit.java::commonInit]</p>
<div>
    <p>private static final void commonInit() {</p>
    <p>   //调用完毕后，该应用中所有线程抛出的未处理异常都会由UncaughtHandler来处理</p>
    <p>   Thread.setDefaultUncaughtExceptionHandler(newUncaughtHandler());</p>
    <p>  ......</p>
    <p>}</p>
</div>
<p>应用程序有问题是再平常不过的事情了，不过，当抛出的异常没有被截获时，系统又会做什么处理呢？来看UncaughtHandler的代码。</p>
<h3>6.7.1  应用进程的Crash处理</h3>
<p>[--&gt;RuntimeInit.java::UncaughtHandler]</p>
<div>
    <p> privatestatic class UncaughtHandler implements </p>
    <p>                                           Thread.UncaughtExceptionHandler{</p>
    <p>   publicvoid uncaughtException(Thread t, Throwable e) {</p>
    <p>    try {</p>
    <p>          if (mCrashing) return;</p>
    <p>           mCrashing = true;</p>
    <p>         //调用AMS的handleApplicationCrash函数。第一个参数mApplicationObject其实</p>
    <p>        //就是前面经常见到的ApplicationThread对象</p>
    <p>          ActivityManagerNative.getDefault().handleApplicationCrash(</p>
    <p>               mApplicationObject, new  ApplicationErrorReport.CrashInfo(e));</p>
    <p>     } ......</p>
    <p>    finally{</p>
    <p>           //这里有一句注释很有意思：Try everything to make surethis process goes </p>
    <p>          //away.从下面这两句调用来看，该应用进程确实想方设法要离开Java世界</p>
    <p>          Process.killProcess(Process.myPid());//把自己杀死</p>
    <p>          System.exit(10);//如果上面那句话不成功，再尝试exit方法</p>
    <p>     }</p>
    <p>   }</p>
    <p>}</p>
</div>
<h3>6.7.2  AMS的handleApplicationCrash分析</h3>
<p>AMS handleApplicationCrash函数的代码如下：</p>
<p>[--&gt;ActivityManagerService.java::handleApplicationCrash]</p>
<div>
    <p>public void handleApplicationCrash(IBinder app, </p>
    <p>                             ApplicationErrorReport.CrashInfocrashInfo) {</p>
    <p>  //找到对应的ProcessRecord信息，后面那个字符串“Crash”用于打印输出</p>
    <p> ProcessRecord r = findAppProcess(app, "Crash");</p>
    <p>  finalString processName = app == null ? "system_server"</p>
    <p>                        : (r == null ? "unknown": r.processName);</p>
    <p>   //添加crash信息到dropbox中</p>
    <p> ddErrorToDropBox("crash", r, processName, null, null, null,null, null,</p>
    <p>                         crashInfo);</p>
    <p>  //调用crashApplication函数</p>
    <p> crashApplication(r, crashInfo);</p>
    <p>}</p>
</div>
<p>[--&gt;ActivityManagerService.java::crashApplication]</p>
<div>
    <p>private void crashApplication(ProcessRecord r,</p>
    <p>                   ApplicationErrorReport.CrashInfocrashInfo) {</p>
    <p>   longtimeMillis = System.currentTimeMillis();</p>
    <p>   //从应用进程传递过来的crashInfo中获取相关信息</p>
    <p>   StringshortMsg = crashInfo.exceptionClassName;</p>
    <p>   StringlongMsg = crashInfo.exceptionMessage;</p>
    <p>   StringstackTrace = crashInfo.stackTrace;</p>
    <p>   if(shortMsg != null &amp;&amp; longMsg != null) {</p>
    <p>      longMsg = shortMsg + ": " + longMsg;</p>
    <p>     } elseif (shortMsg != null) {</p>
    <p>      longMsg = shortMsg;</p>
    <p>   }</p>
    <p> </p>
    <p>  AppErrorResult result = new AppErrorResult();</p>
    <p>  synchronized (this) {</p>
    <p>      if(mController != null) {</p>
    <p>         //通知监视者。目前仅MonkeyTest中会为AMS设置监听者。测试过程中可设定是否一检测</p>
    <p>         //到App Crash即停止测试。另外，Monkey测试也会将App Crash信息保存起来</p>
    <p>        //供开发人员分析</p>
    <p>       }</p>
    <p> </p>
    <p>      final long origId =Binder.clearCallingIdentity();</p>
    <p> </p>
    <p>      if (r!= null &amp;&amp; r.instrumentationClass != null) {</p>
    <p>       ......// instrumentationClass不为空的情况</p>
    <p>      }</p>
    <p>      //①调用makeAppCrashingLocked处理，如果返回false，则整个处理流程完毕</p>
    <p>     if (r == null || !makeAppCrashingLocked(r,shortMsg, longMsg, </p>
    <p>                        stackTrace)) {</p>
    <p>         ......</p>
    <p>       return;</p>
    <p>      } </p>
    <p>     Message msg = Message.obtain();</p>
    <p>     msg.what = SHOW_ERROR_MSG;</p>
    <p>     HashMap data = new HashMap();</p>
    <p>     data.put("result", result);</p>
    <p>     data.put("app", r);</p>
    <p>     msg.obj = data;</p>
    <p>    //发送SHOW_ERROR_MSG消息给mHandler，默认处理是弹出一个对话框，提示用户某进程   //崩溃（Crash），用户可以选择“退出”或 “退出并报告”</p>
    <p>     mHandler.sendMessage(msg);</p>
    <p>      ......</p>
    <p>   }//synchronized(this)结束</p>
    <p> </p>
    <p>   //下面这个get函数是阻塞的，直到用户处理了对话框为止。注意，此处涉及两个线程：</p>
    <p> //handleApplicationCrash函数是在Binder调用线程中处理的，而对话框则是在mHandler所</p>
    <p>  //在线程中处理的</p>
    <p>   int res =result.get();</p>
    <p>   IntentappErrorIntent = null;</p>
    <p>  synchronized (this) {</p>
    <p>     if (r!= null) </p>
    <p>        mProcessCrashTimes.put(r.info.processName, r.info.uid,</p>
    <p>                       SystemClock.uptimeMillis());</p>
    <p> </p>
    <p>   if (res== AppErrorDialog.FORCE_QUIT_AND_REPORT) </p>
    <p>      //createAppErrorIntentLocked返回一个Intent，该Intent的Action是</p>
    <p>      //"android.intent.action.APP_ERROR"，指定接收者是app. errorReportReceiver</p>
    <p>      //成员，该成员变量在关键点makeAppCrashingLocked中被设置</p>
    <p>       appErrorIntent =createAppErrorIntentLocked(r, timeMillis,</p>
    <p>                       crashInfo);</p>
    <p>   }//synchronized(this)结束</p>
    <p> </p>
    <p>   if(appErrorIntent != null) {</p>
    <p>       try {//启动能处理APP_ERROR的应用进程，目前的源码中还没有地方处理它</p>
    <p>             mContext.startActivity(appErrorIntent);</p>
    <p>       } ......</p>
    <p>    }</p>
    <p> }</p>
</div>
<p>以上代码中还有一个关键函数makeAppCrashingLocked，其代码如下：</p>
<p>[--&gt;ActivityManagerService.java::makeAppCrashingLocked]</p>
<div>
    <p>private booleanmakeAppCrashingLocked(ProcessRecord app,</p>
    <p>           String shortMsg, String longMsg, String stackTrace) {</p>
    <p>  app.crashing = true;</p>
    <p>   //生成一个错误报告，存放在crashingReport变量中</p>
    <p>  app.crashingReport = generateProcessError(app,</p>
    <p>               ActivityManager.ProcessErrorStateInfo.CRASHED, null, shortMsg,</p>
    <p>               longMsg, stackTrace);</p>
    <p>   /* </p>
    <p>    在上边代码中，我们知道系统会通过APP_ERROR Intent启动一个Activity去处理错误报告，</p>
    <p>    实际上在此之前，系统需要为它找到指定的接收者（如果有）。代码在startAppProblemLocked</p>
    <p>    中，此处简单描述该函数的处理过程如下：</p>
    <p>    1 先查询Settings Secure表中“send_action_app_error”是否使能，如果没有<span>使能</span>，则</p>
    <p>      不能设置指定接收者</p>
    <p>    2 通过PKMS查询安装此Crash应用的应用是否能接收APP_ERROR Intent。必须注意</p>
    <p>     安装此应用的应用（例如通过“安卓市场”安装了该Crash应用）。如果没有，则转下一步处理</p>
    <p>    3 查询系统属性“ro.error.receiver.system.apps”是否指定了APP_ERROR处理者，如果</p>
    <p>      没有，则转下一步处理</p>
    <p>    4 查询系统属性“ro.error.receiver.default”是否指定了默认的处理者</p>
    <p>    5 处理者的信息保存在app的errorReportReceiver变量中</p>
    <p>    另外，如果该Crash应用正好是串行广播发送处理中的一员，则需要结束它的处理流程以</p>
    <p>    保证后续广播处理能正常执行。读者可参考skipCurrentReceiverLocked函数</p>
    <p>  */</p>
    <p>  startAppProblemLocked(app);</p>
    <p>  app.stopFreezingAllLocked();</p>
    <p>   //调用handleAppCrashLocked做进一步处理。读者可自行阅读</p>
    <p>   returnhandleAppCrashLocked(app);</p>
    <p> }</p>
</div>
<p>当App的Crash处理完后，事情并未就此结束，因为该应用进程退出后，之前AMS为它设置的讣告接收对象将被唤醒。接下来介绍AppDeathRecipientbinderDied的处理流程。</p>
<h3>6.7.3  AppDeathRecipient binderDied分析</h3>
<h4>1.  binderDied函数分析</h4>
<p>[--&gt;ActvityManagerService.java::AppDeathRecipientbinderDied]</p>
<div>
    <p>public void binderDied() {</p>
    <p>  //注意，该函数也是通过Binder线程调用的，所以此处要加锁</p>
    <p> synchronized(ActivityManagerService.this) {</p>
    <p>       appDiedLocked(mApp, mPid, mAppThread);</p>
    <p>   }</p>
    <p> }</p>
</div>
<p>最终的处理函数是appDiedLocked，其中所传递的3个参数保存了对应死亡进程的信息。来看appDiedLocked的代码：：</p>
<p>[--&gt;ActvityManagerService.java::appDiedLocked]</p>
<div>
    <p>final void appDiedLocked(ProcessRecord app, intpid,</p>
    <p>                                                IApplicationThread thread) {</p>
    <p>   ......</p>
    <p>   if(app.pid == pid &amp;&amp; app.thread != null &amp;&amp;</p>
    <p>          app.thread.asBinder() == thread.asBinder()) {</p>
    <p>   //当内存低到水位线时，LMK模块也会杀死进程。对AMS来说，需要区分进程死亡是LMK导致的</p>
    <p>   //还是其他原因导致的。App instrumentationClass一般都为空，故此处doLowMem为true</p>
    <p>    booleandoLowMem = app.instrumentationClass == null;</p>
    <p> </p>
    <p>    //①下面这个函数非常重要</p>
    <p>   handleAppDiedLocked(app, false, true);</p>
    <p> </p>
    <p>    if(doLowMem) {</p>
    <p>      boolean haveBg = false;</p>
    <p>      //如果系统中还存在oom_adj大于HIDDEN_APP_MIN_ADJ的进程，就表明不是LMK模块因</p>
    <p>      //内存不够而导致进程死亡的</p>
    <p>       for(int i=mLruProcesses.size()-1; i&gt;=0; i--) {</p>
    <p>          ProcessRecord rec = mLruProcesses.get(i);</p>
    <p>          if (rec.thread != null &amp;&amp; rec.setAdj &gt;= </p>
    <p>                                           ProcessList.HIDDEN_APP_MIN_ADJ){</p>
    <p>                haveBg = true;//还有后台进程，故可确定系统内存尚未吃紧</p>
    <p>                break;</p>
    <p>            }</p>
    <p>        }//for循环结束</p>
    <p>      //如果没有后台进程，表明系统内存已吃紧</p>
    <p>      if(!haveBg) {</p>
    <p>         long now = SystemClock.uptimeMillis();</p>
    <p>          for(int i=mLruProcesses.size()-1; i&gt;=0; i--) {</p>
    <p>               .....//将这些进程按一定规则加到mProcessesToGc中，尽量保证</p>
    <p>              //heavy/important/visible/foreground的进程位于mProcessesToGc数组</p>
    <p>              //的前端</p>
    <p>          }//for循环结束</p>
    <p>          /*</p>
    <p>           发送GC_BACKGROUND_PROCESSES_MSG消息给mHandler，该消息的处理过程就是：</p>
    <p>           调用应用进程的scheduleLowMemory或processInBackground函数。其中，</p>
    <p>           scheduleLowMemory将触发onLowMemory回调被调用，而processInBackground将</p>
    <p>           触发应用进程进行一次垃圾回收</p>
    <p>           读者可自行阅读该消息的处理函数performAppGcsIfAppropriateLocked</p>
    <p>          */</p>
    <p>         scheduleAppGcsLocked();</p>
    <p>       }// if(!haveBg)判断结束</p>
    <p>    }//if(doLowMem)判断结束</p>
    <p> }</p>
</div>
<p>以上代码中有一个关键函数handleAppDiedLocked，下面来看它的处理过程。</p>
<h4>2.  handleAppDiedLocked函数分析</h4>
<p>[--&gt;ActivityManagerService.java::handleAppDiedLocked]</p>
<div>
    <p>private final voidhandleAppDiedLocked(ProcessRecord app,</p>
    <p>           boolean restarting, boolean allowRestart) {</p>
    <p>   //①在本例中，传入的参数为restarting=false, allowRestart=true</p>
    <p>  cleanUpApplicationRecordLocked(app, restarting, allowRestart, -1);</p>
    <p>   if(!restarting) {</p>
    <p>       mLruProcesses.remove(app);</p>
    <p>   }</p>
    <p>   ......//下面还有一部分代码处理和Activity相关的收尾工作，读者可自行阅读</p>
    <p>}</p>
</div>
<p>重点看上边代码中的cleanUpApplicationRecordLocked函数，该函数的主要功能就是处理Service、ContentProvider及BroadcastReceiver相关的收尾工作。先来看Service方面的工作。</p>
<h5>（1） cleanUpApplicationRecordLocked之处理Service </h5>
<p>[--&gt;ActivityManagerService.java::cleanUpApplicationRecordLocked]</p>
<div>
    <p> privatefinal void cleanUpApplicationRecordLocked(ProcessRecord app,</p>
    <p>       boolean restarting, boolean allowRestart, int index) {</p>
    <p>   if (index&gt;= 0) mLruProcesses.remove(index);</p>
    <p> </p>
    <p>   mProcessesToGc.remove(app);</p>
    <p>   //如果该Crash进程有对应打开的对话框，则关闭它们，这些对话框包括crash、anr和wait等</p>
    <p>   if(app.crashDialog != null) {</p>
    <p>       app.crashDialog.dismiss();</p>
    <p>       app.crashDialog = null;</p>
    <p>   }</p>
    <p>   ......//处理anrDialog、waitDialog</p>
    <p> </p>
    <p>   //清理app的一些参数</p>
    <p>  app.crashing = false;</p>
    <p>  app.notResponding = false;</p>
    <p>   ......</p>
    <p>   //处理该进程中所驻留的Service或它和别的进程中的Service建立的Connection关系</p>
    <p>   //该函数是AMS Service处理流程中很重要的一环，读者要仔细阅读</p>
    <p>   killServicesLocked(app,allowRestart);</p>
</div>
<p>cleanUpApplicationRecordLocked函数首先处理几个对话框（dialog），然后调用killServicesLocked函数做相关处理。作为Service流程的一部分，读者需要深入研究。 </p>
<h5>（2） cleanUpApplicationRecordLocked之处理ContentProvider</h5>
<p>再来看cleanUpApplicationRecordLocked下一阶段的工作，主要和ContentProvider有关。</p>
<p>[--&gt;ActivityManagerService.java::cleanUpApplicationRecordLocked]</p>
<div>
    <p>   booleanrestart = false;</p>
    <p> </p>
    <p>   int NL = mLaunchingProviders.size();</p>
    <p>        </p>
    <p>   if(!app.pubProviders.isEmpty()) {</p>
    <p>       //得到该进程中发布的ContentProvider信息</p>
    <p>       Iterator&lt;ContentProviderRecord&gt; it =</p>
    <p>                          app.pubProviders.values().iterator();</p>
    <p>       while(it.hasNext()) {</p>
    <p>         ContentProviderRecord cpr = it.next();</p>
    <p>          cpr.provider = null;</p>
    <p>         cpr.proc = null;</p>
    <p>         int i = 0;</p>
    <p>          if(!app.bad &amp;&amp; allowRestart) {</p>
    <p>              for (; i&lt;NL; i++) {</p>
    <p>                 /*</p>
    <p>                   如果有客户端进程在等待这个已经死亡的ContentProvider,则系统会</p>
    <p>                   尝试重新启动它，即设置restart变量为true</p>
    <p>                */</p>
    <p>                 if (mLaunchingProviders.get(i) == cpr) {</p>
    <p>                       restart = true;</p>
    <p>                        break;</p>
    <p>                  } </p>
    <p>              }//for循环结束</p>
    <p>          } else  i = NL;</p>
    <p> </p>
    <p>          if(i &gt;= NL) {</p>
    <p>             /*</p>
    <p>              如果没有客户端进程等待这个ContentProvider，则调用下面这个函数处理它，我们</p>
    <p>              在卷I的第10章曾提过一个问题，即ContentProvider进程被杀死</p>
    <p>              后，系统该如何处理那些使用了该ContentProvider的客户端进程。例如，Music和</p>
    <p>              MediaProvider之间有交互，如果杀死了MediaProvider，Music会怎样呢？</p>
    <p>              答案是系统会杀死Music，证据就在removeDyingProviderLocked函数</p>
    <p>              中，读者可自行阅读其内部处理流程</p>
    <p>             */</p>
    <p>             removeDyingProviderLocked(app, cpr);</p>
    <p>             NL = mLaunchingProviders.size();</p>
    <p>           }</p>
    <p>      }// while(it.hasNext())循环结束</p>
    <p>    app.pubProviders.clear();</p>
    <p>   }</p>
    <p>   //下面这个函数的功能是检查本进程中的ContentProvider是否存在于</p>
    <p>  // mLaunchingProviders中，如果存在，则表明有客户端在等待，故需考虑是否重启本进程或者</p>
    <p>  //杀死客户端（当死亡进程变成bad process的时，需要杀死客户端）</p>
    <p>   if(checkAppInLaunchingProvidersLocked(app, false)) restart = true;</p>
    <p>   ......</p>
</div>
<p>从以上的描述中可知，ContentProvider所在进程和其客户端进程实际上有着非常紧密而隐晦（之所以说其隐晦，是因为SDK中没有任何说明）的关系。在目前软件开发追求模块间尽量保持松耦合关系的大趋势下，Android中的ContentProvider和其客户端这种紧耦合的设计思路似乎不够明智。不过，这种设计是否是不得已而为之呢？读者不妨探讨一下，如果有更合适的解决方案，期待能一起分享。</p>
<h5>（3） cleanUpApplicationRecordLocked之处理BroadcastReceiver</h5>
<p>[--&gt;ActivityManagerService.java::cleanUpApplicationRecordLocked]</p>
<div>
    <p> </p>
    <p>  skipCurrentReceiverLocked(app);</p>
    <p>    //从AMS中去除接收者</p>
    <p>   if(app.receivers.size() &gt; 0) {</p>
    <p>      Iterator&lt;ReceiverList&gt; it = app.receivers.iterator();</p>
    <p>       while(it.hasNext()) {</p>
    <p>          removeReceiverLocked(it.next());</p>
    <p>        }</p>
    <p>       app.receivers.clear();</p>
    <p>   }</p>
    <p>        </p>
    <p>   if(mBackupTarget != null &amp;&amp; app.pid == mBackupTarget.app.pid) {</p>
    <p>         //处理Backup信息</p>
    <p>    }</p>
    <p> </p>
    <p>    mHandler.obtainMessage(DISPATCH_PROCESS_DIED, app.pid, </p>
    <p>                                   app.info.uid,null).sendToTarget();</p>
    <p>    //注意该变量名为restarting，前面设置为restart.</p>
    <p>    if(restarting) return;</p>
    <p> </p>
    <p> </p>
    <p>    if(!app.persistent) {</p>
    <p>        mProcessNames.remove(app.processName, app.info.uid);</p>
    <p>         if(mHeavyWeightProcess == app) {</p>
    <p>             ......//处理HeavyWeightProcess</p>
    <p>           }</p>
    <p>      } elseif (!app.removed) {</p>
    <p>         if(mPersistentStartingProcesses.indexOf(app) &lt; 0) {</p>
    <p>              mPersistentStartingProcesses.add(app);</p>
    <p>               restart = true;</p>
    <p>          }</p>
    <p>      }</p>
    <p>    mProcessesOnHold.remove(app);</p>
    <p>     if (app== mHomeProcess)  mHomeProcess = null;</p>
    <p>        </p>
    <p>     </p>
    <p>     if(restart) {//如果需要重启，则调用startProcessLocked处理它</p>
    <p>          mProcessNames.put(app.processName, app.info.uid, app);</p>
    <p>         startProcessLocked(app, "restart", app.processName);</p>
    <p>     } elseif (app.pid &gt; 0 &amp;&amp; app.pid != MY_PID) {</p>
    <p>        synchronized (mPidsSelfLocked) {</p>
    <p>             mPidsSelfLocked.remove(app.pid);</p>
    <p>             mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);</p>
    <p>          }</p>
    <p>         app.setPid(0);</p>
    <p>     }</p>
    <p> }</p>
</div>
<p>在这段代码中，除了处理BrodcastReceiver方面的工作外，还包括其他方面的收尾工作。最后，如果要重启该应用，则需调用startProcessLocked函数进行处理。这部分代码不再详述，读者可自行阅读。</p>
<h3>6.7.4  App的Crash处理总结</h3>
<p>分析完整个处理流程，有些读者或许会咂舌。应用进程的诞生是一件很麻烦的事情，没想到应用进程的善后工作居然也很费事，希望各个应用进程能活得更稳健点儿。</p>
<p>图6-24展示了应用进程进行Crash处理的流程。</p>

![image](images/chapter6/image024.png)

<p>图6-24  应用进程的Crash处理流程</p>
<p> </p>
<h2>6.7  本章学习指导</h2>
<p>本章内容较为复杂，即使用了这么长的篇幅来讲解AMS，依然只能覆盖其中一部分内容。读者在阅读本章时，一定要注意文中的分析脉络，以搞清楚流程为主旨。以下是本章的思路总结：</p>
<p>·  首先搞清楚AMS初创时期的一系列流程，这对理解Android运行环境和系统启动流程等很有帮助。</p>
<p>·  搞清楚一个最简单的情形下Activity启动所历经的磨难。这部分流程最复杂，建议读者在搞清书中所阐述内容的前提下结合具体问题进行分析。</p>
<p>·  BroadcastReceiver的处理流程相对简单。读者务必理解AMS发送广播的处理流程，这对实际工作非常有帮助。例如最近在处理一个广播时，发现静态注册的广播接收者收到广播的时间较长，研究了AMS广播发送的流程后，将其改成了动态注册，结果响应速度就快了很多。</p>
<p>·  关于Service的处理流程希望读者根据流程图自行分析和研究。</p>
<p>·  AMS中的进程管理这一部分内容最难看懂。此处有多方面的原因，笔者觉得和缺乏相关说明有重要关系。建议读者只了解AMS进程管理的大概面貌即可。另外，建议读者不要试图通过修改这部分代码来优化Android的运行效率。进程调度规则向来比较复杂，只有在大量实验的基础上才能得到一个合适的模型。</p>
<p>·  AMS在处理应用进程的Crash及死亡的工作上也是不遗余力的。这部分工作相对比较简单，相信读者能轻松掌握。</p>
<p>由于精力和篇幅的原因，AMS中还有很多精彩的内容未能涉及，建议读者在本章学习基础上，根据具体情况继续深入研究。</p>
<h2>6.8  本章小结</h2>
<p>本章对AMS进行有针对性的分析：</p>
<p>·  先分析了AMS的创建及初始化过程。</p>
<p>·  以启动一个Activity为例，分析了Activity的启动，应用进程的创建等一系列比较复杂的处理流程。</p>
<p>·  介绍了广播接收者及广播发送的处理流程。Service的处理流程读者可根据流程图并结合代码自行开展研究。</p>
<p>·  还介绍了AMS中的进程管理及相关的知识。重点是在AMS中对LRU、oom_adj及scheduleGroup的调整。</p>
<p>·  最后介绍了应用进程Crash及死亡后，AMS的处理流程。</p>
<p> </p>
<div>
    <br />
    <hr />
    <div>
        <p><a>[①]</a> AMS较复杂，其中一个原因是其功能较多，另一个不可忽视的原因是它的代码质量及结构并不出彩。</p>
    </div>
    <div>
        <p><a>[②]</a> 实际上，SettingsProvider.apk也运行于SystemServer进程中。</p>
    </div>
    <div>
        <p><a>[③]</a> http://www.merriam-webster.com/dictionary/activity中第六条解释</p>
    </div>
    <div>
        <p><a>[④]</a> 关于zygote的工作原理，请读者阅读卷I第4章“深入理解Zygote”。</p>
    </div>
    <div>
        <p><a>[⑤]</a> 由于系统内部保存了Stikcy广播，当一个恶意程序频繁发送这种广播时，是否会把内存都耗光呢？</p>
    </div>
    <div>
        <p><a>[⑥]</a> 更为详细的知识请阅读<a>http://blog.csdn.net/innost/article/details/6940136</a>。</p>
    </div>
</div>
<div>
    <p>版权声明：本文为博主原创文章，未经博主允许不得转载。</p>
</div>
