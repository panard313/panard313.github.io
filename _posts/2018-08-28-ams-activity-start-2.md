---
layout: article
title: 启动Activity的过程：二
key: 20180827
tags:
  - ActivityManagerService
  - ams
  - activity start
lang: zh-Hans
---

# 启动Activity的过程：二

在"启动Activity的过程：一"中，我们的流程最终分析到AMS通过zygote启动Activity对应的进程，现在我们看看后续的过程如何进行。

关于zygote启动进程的流程，可以参考Android6.0 SystemServer进程。这篇文章中，分析了zygote如何启动SystemServer进程和普通进程。虽然分析的是Android M的代码，但与Android N的思路基本一致。

## 一、ActivityThread的main函数

通过zygote启动进程时，传入的className为android.app.ActivityThread。
因此，当zygote通过反射调用进程的main函数时，ActivityThread的main函数将被启动：
```java
public static void main(String[] args) {
    ..................
    //开始信息采样
    SamplingProfilerIntegration.start();

    // CloseGuard defaults to true and can be quite spammy.  We
    // disable it here, but selectively enable it later (via
    // StrictMode) on debug builds, but using DropBox, not logs.
    //和调试有关
    CloseGuard.setEnabled(false);

    //得到当前进程的UserEnvironment
    Environment.initForCurrentUser();
    ...............
    // Make sure TrustedCertificateStore looks in the right place for CA certificates
    // 确保进程能够得到CA证书的路径
    final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
    TrustedCertificateStore.setDefaultUserDirectory(configDir);

    Process.setArgV0("<pre-initialized>");

    //准备主线程的Looper
    Looper.prepareMainLooper();

    //创建当前进程的ActivityThread
    ActivityThread thread = new ActivityThread();
    //调用attach函数
    thread.attach(false);

    if (sMainThreadHandler == null) {
        //保存进程对应的主线程Handler
        sMainThreadHandler = thread.getHandler();
    }
    .........
    //进入主线程的消息循环
    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
从上述代码可以看出，ActivityThread的main函数最主要工作是：
- 1、创建出一个Looper，并将主线程加入到消息循环中。
- 2、创建出ActivityThread，并调用其attach函数。

在介绍AMS的启动过程时，我们就提到了ActivityThread。

当时从代码中，我们知道SystemServer进程，为了融入Android体系，调用createSystemContext函数。
在createSystemContext函数中，SystemServer进程创建了自己的ActivityThread，并调用了attach函数。
通过比对代码容易看出，SystemServer进程作为系统进程，attach参数为true，而普通进程传入的参数为false。

现在，我们重新看看ActivityThread的attach函数，这次侧重于普通进程的部分：
```java
private void attach(boolean system) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
        ViewRootImpl.addFirstDrawHandler(new Runnable() {
            @Override
            public void run() {
                //JIT-Just in time
                //JIT技术主要是对多次运行的代码进行编译，当再次调用时使用编译之后的机器码，而不是每次都解释，以节约时间

                //JIT原理：
                //每启动一个应用程序，都会相应地启动一个dalvik虚拟机，启动时会建立JIT线程，一直在后台运行。
                //当某段代码被调用时，虚拟机会判断它是否需要编译成机器码，如果需要，就做一个标记。
                //JIT线程在后台检测该标记，如果发现标记被设定，就把对应代码编译成机器码，并将其机器码地址及相关信息保存起来
                //当进程下次执行到此段代码时，就会直接跳到机器码执行，而不再解释执行，从而提高运行速度

                //这里开启JIT，应该是为了提高android绘制的速度
                ensureJitEnabled();
            }
        });

        //设置在DDMS中看到的进程名为"<pre-initialized>"
        android.ddm.DdmHandleAppName.setAppName("<pre-initialized>", UserHandle.myUserId());

        //设置RuntimeInit的mApplicationObject参数
        RuntimeInit.setApplicationObject(mAppThread.asBinder());

        final IActivityManager mgr = ActivityManagerNative.getDefault();
        try {
            //与AMS通信，调用其attachApplication接口
            mgr.attachApplication(mAppThread);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }

        // Watch for getting close to heap limit.
        // 监控GC操作; 当进程内的一些Activity发生变化，同时内存占用量较大时
        // 通知AMS释放一些Activity
        BinderInternal.addGcWatcher(new Runnable() {
            public void run() {
                if (!mSomeActivitiesChanged) {
                    return;
                }

                Runtime runtime = Runtime.getRuntime();
                long dalvikMax = runtime.maxMemory();
                long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                //判断内存占用量是否过大
                if (dalvikUsed > ((3*dalvikMax)/4)) {
                    ...........
                    mSomeActivitiesChanged = false;
                    try {
                        //通知AMS释放一些Activity，以缓解内存紧张
                        mgr.releaseSomeActivities(mAppThread);
                    } catch (RemoteException e) {
                        throw e.rethrowFromSystemServer();
                    }
                }
            }
        });
    } else {
        .............
    }
    ........
}
```

在分析Activity启动过程的第一部分时，我们提到过AMS创建一个应用进程后，会设置一个超时时间。
如果超过这个时间，应用进程还没有和AMS交互，AMS就认为该进程创建失败。
因此，应用进程启动后，需要尽快和AMS交互。

上述代码中，attachApplication就是应用进程与AMS交互的接口。

## 二、AMS的attachApplication函数

我们进入AMS看看attachApplication相关的流程。
```java
public final void attachApplication(IApplicationThread thread) {
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        final long origId = Binder.clearCallingIdentity();
        //进一步调用attachApplicationLocked函数
        attachApplicationLocked(thread, callingPid);
        Binder.restoreCallingIdentity(origId);
    }
}
```
attachApplicationLocked函数比较长，分段来看一下。

#### 1 Part-I
```java
private final boolean attachApplicationLocked(IApplicationThread thread, int pid) {
    ProcessRecord app;

    //根据pid查找对应的ProcessRecord对象
    if (pid != MY_PID && pid >= 0) {
        synchronized (mPidsSelfLocked) {
            app = mPidsSelfLocked.get(pid);
        }
    } else {
        app = null;
    }

    //如果进程由AMS启动，则它在AMS中一定有对应的ProcessRecord
    //此处app为null，则表示AMS没有该进程的记录，故需要kill掉此异常进程
    if (app == null) {
        ................
        if (pid > 0 && pid != MY_PID) {
            //pid大于0且不是系统进程，则直接kill掉
            //将调用android_util_Process.cpp中的android_os_Process_sendSignalQuiet函数
            //最终通过kill函数杀死进程，kill函数要求pid > 0
            Process.killProcessQuiet(pid);
        } else {
            try {
                //pid < 0时，fork进程失败，因此仅上层完成清理工作即可
                //调用ApplicationThread的scheduleExit函数
                //应用进程将进行一些扫尾工作，例如结束消息循环，然后退出运行
                thread.scheduleExit();
            } catch (Exception e) {
                // Ignore exceptions.
            }
        }
        return false;
    }

    // If this application record is still attached to a previous
    // process, clean it up now.
    // 判断pid对应processRecord的IApplicationThread是否为null
    // AMS创建ProcessRecord后，在attach之前，正常情况下IApplicationThread应该为null

    // 特殊情况下：如果旧应用进程被杀死，底层对应的pid被释放，在通知到达AMS之前（AMS在下面的代码里注册了“讣告”接收对象），
    // 用户又启动了一个新的进程，新进程刚好分配到旧进程的pid时
    // 此处得到的processRecord可能就是旧进程的，于是app.thread可能不为null，因此需要作判断和处理
    if (app.thread != null) {
        handleAppDiedLocked(app, true, true);
    }
    ...................
    final String processName = app.processName;
    try {
        //创建一个“讣告”接收对象，注册到应用进程的ApplicationThread中
        //当应用进程退出时，该对象的binderDied将被调用，这样AMS就能做相应的处理
        //binderDied函数将在另一个线程中被调用，其内部也会调用handleAppDiedLocked函数
        AppDeathRecipient adr = new AppDeathRecipient(
                app, pid, thread);
        thread.asBinder().linkToDeath(adr, 0);
        app.deathRecipient = adr;
    } catch (RemoteException e) {
        app.resetPackageList(mProcessStats);
        startProcessLocked(app, "link fail", processName);
        return false;
    }
    设置app的一些变量，例如调度优先级和oom_adj相关的成员
    .................
    //启动成功，从消息队列中移除PROC_START_TIMEOUT_MSG
    mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
    .................
```

至此，attachApplicationLocked的第一部分介绍完毕。

这部分代码的核心功能比较简单，其实就是：
- 1、判断进程的有效性，同时注册观察者监听进程的死亡信号。
- 2、设置pid对应的ProcessRecord对象的一些成员变量，例如和应用进程交互的IApplicationThread对象、进程调度的优先级等。
- 3、进程注册成功，AMS从消息队列中移除PROC_START_TIMEOUT_MSG。

#### 2 Part-II

现在，我们看看attachApplicationLocked第二部分的代码：
```java
..............
//AMS正常启动后，mProcessesReady就已经变为true了
boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);

//generateApplicationProvidersLocked将通过PKMS查询定义在进程中的ContentProvider，并将其保存在AMS的数据结构中
List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;

//这里应该是处理：加载ContentProvider时，启动进程的场景
//checkAppInLaunchingProvidersLocked主要将当前启动进程的ProcessRecord，和AMS中mLaunchingProviders的ProcessRecord进行比较
//当判断出该进程是由于启动ContentProvider而被加载的，那么就发送一个延迟消息（10s）
//通过这里可以看出，当由于加载ContentProvider启动进程时，在进程启动后，ContentProvider在10s内要完成发布
if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
    Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
    msg.obj = app;
    mHandler.sendMessageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
}
...........
try {
    ...........
    //回调进程ApplicationThread的bindApplication接口
    thread.bindApplication(..........);

    //更新进程调度策略
    updateLruProcessLocked(app, false, null);
    ..............
} catch () {
    ............
    app.resetPackageList(mProcessStats);
    app.unlinkDeathRecipient();
    //这里的策略比较激进，当bindApplicaiton失败后，将直接重新启动这个进程
    startProcessLocked(app, "bind fail", processName);
    return false;
}
```
从代码来看，第二阶段最核心工作就是：
调用进程ApplicationThread的bindApplication函数，接下来我们分析一下该函数。

#### 2.1 ApplicationThread的bindApplication函数

从之前的代码，我们知道应用进程由zygote fork得到，然后调用ActivityThread的main函数，进入到Java世界。

但是截至目前，该进程并没有融入到Android的体系中，因此仅能被称为一个Java进程，甚至连进程名也只是“敷衍”地定义为“pre-initialized”。

我们之前分析AMS启动过程时，介绍了SystemServer的createSystemContext函数。
在该函数中，SystemServer在自己的进程中，创建出Android运行环境，才摇身一变成为了Android进程。
同样，此处的bindApplication函数，就是在新进程中创建并初始化对应的Android运行环境。

现在，我们看看bindApplication函数的主要流程：
```java
//参数较多，无需深究，重要的从代码流程中就能知道含义
public final void bindApplication(.......) {
    //按照string-IBinder的方式，保存AMS传递过来的系统service的Binder接口
    //这样进程与系统服务通信时，就不需要先通过SystemServer查询了
    if (services != null) {
        // Setup the service cache in the ServiceManager
        ServiceManager.initServiceCache(services);
    }

    //内部发送H.SET_CORE_SETTINGS消息
    //由handleSetCoreSettings进行处理，主要用于保存新的信息
    setCoreSettings(coreSettings);

    //用AppBindData对象保存参数对应的信息
    AppBindData data = new AppBindData();
    data.processName = processName;
    data.appInfo = appInfo;
    data.providers = providers;
    ..................
    //发送消息
    sendMessage(H.BIND_APPLICATION, data);
}
```
由以上代码可知，ApplicationThread的接口被AMS调用后，会将参数保存到AppBindData对象中，然后发送消息让ActivityThread的主线程处理。
由此可以看出，对应用进程而言，ApplicationThread只是与AMS通信的接口，实际的工作一般还是会交给ActivityThread来完成。

ActivityThread中处理该消息的实际函数为handleBindApplication，我们看看这个函数的内容。

#### 2.2 handleBindApplication函数
```java
private void handleBindApplication(AppBindData data) {
    // Register the UI Thread as a sensitive thread to the runtime.
    VMRuntime.registerSensitiveThread();
    .................
    //初始化性能统计对象
    mProfiler = new Profiler();
    if (data.initProfilerInfo != null) {
        mProfiler.profileFile = data.initProfilerInfo.profileFile;
        mProfiler.profileFd = data.initProfilerInfo.profileFd;
        mProfiler.samplingInterval = data.initProfilerInfo.samplingInterval;
        mProfiler.autoStopProfiler = data.initProfilerInfo.autoStopProfiler;
    }

    // send up app name; do this *before* waiting for debugger
    //重新设置进程名，并修改DDMS中的名称
    Process.setArgV0(data.processName);
    android.ddm.DdmHandleAppName.setAppName(data.processName,
            UserHandle.myUserId());

    if (data.persistent) {
        // Persistent processes on low-memory devices do not get to
        // use hardware accelerated drawing, since this can add too much
        // overhead to the process.

        //在低内存设备上，禁止常驻进程使用硬件加速
        if (!ActivityManager.isHighEndGfx()) {
            ThreadedRenderer.disable(false);
        }
    }

    //启动性能统计
    if (mProfiler.profileFd != null) {
        mProfiler.startProfiling();
    }

    // If the app is Honeycomb MR1 or earlier, switch its AsyncTask
    // implementation to use the pool executor.  Normally, we use the
    // serialized executor as the default. This has to happen in the
    // main thread so the main looper is set right.

    //Java并发使用的ThreadExecutor，没有太深入的去了解
    //比较浅显的来讲，serialized executor应该是保证一个任务执行完毕后，才去执行下一个任务
    //pool executor根据配置的corePoolSize决定初始时，可以并行的数量；
    //当同时提交的任务数量，超过corePoolSize时，任务就会加入到队列中
    if (data.appInfo.targetSdkVersion <= android.os.Build.VERSION_CODES.HONEYCOMB_MR1) {
        AsyncTask.setDefaultExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
    }

    /*
    * Before spawning a new process, reset the time zone to be the system time zone.
    * This needs to be done because the system time zone could have changed after the
    * the spawning of this process. Without doing this this process would have the incorrect
    * system time zone.
    */
    TimeZone.setDefault(null);

    /*
    * Set the LocaleList. This may change once we create the App Context.
    */
    LocaleList.setDefault(data.config.getLocales());

    //更新资源和兼容性相关的配置
    synchronized (mResourcesManager) {
        /*
         * Update the system configuration since its preloaded and might not
         * reflect configuration changes. The configuration object passed
         * in AppBindData can be safely assumed to be up to date
         */
        mResourcesManager.applyConfigurationToResourcesLocked(data.config, data.compatInfo);
        mCurDefaultDisplayDpi = data.config.densityDpi;

        // This calls mResourcesManager so keep it within the synchronized block.
        applyCompatConfiguration(mCurDefaultDisplayDpi);
    }

    //根据传递过来的ApplicationInfo创建一个对应的LoadedApk对象
    data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);

    /**
    * Switch this process to density compatibility mode if needed.
    */
    //如果没有设置屏幕密度，则为Bitmap设置默认的屏幕密度
    if ((data.appInfo.flags&ApplicationInfo.FLAG_SUPPORTS_SCREEN_DENSITIES)
            == 0) {
        mDensityCompatMode = true;
        Bitmap.setDefaultDensity(DisplayMetrics.DENSITY_DEFAULT);
    }
    updateDefaultDensity();

    final boolean is24Hr = "24".equals(mCoreSettings.getString(Settings.System.TIME_12_24));
    DateFormat.set24HourTimePref(is24Hr);
    ....................
    /**
    * For apps targetting Honeycomb or later, we don't allow network usage
    * on the main event loop / UI thread. This is what ultimately throws
    * {@link NetworkOnMainThreadException}.
    */
    //禁止在主线程使用网络操作
    if (data.appInfo.targetSdkVersion >= Build.VERSION_CODES.HONEYCOMB) {
        StrictMode.enableDeathOnNetwork();
    }

    /**
    * For apps targetting N or later, we don't allow file:// Uri exposure.
    * This is what ultimately throws {@link FileUriExposedException}.
    */
    //禁止主线程操作文件
    if (data.appInfo.targetSdkVersion >= Build.VERSION_CODES.N) {
        StrictMode.enableDeathOnFileUriExposure();
    }
    ...............
    /**
    * Initialize the default http proxy in this process for the reasons we set the time zone.
    */
    final IBinder b = ServiceManager.getService(Context.CONNECTIVITY_SERVICE);
    if (b != null) {
        final IConnectivityManager service = IConnectivityManager.Stub.asInterface(b);
        try {
            //设置默认的Http proxy
            final ProxyInfo proxyInfo = service.getProxyForNetwork(null);
            Proxy.setHttpProxySystemProperty(proxyInfo);
        } catch (RemoteException e) {
            .............
        }
    }
    .........................
    //创建出进程读应的Android运行环境
    final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
    .........................
    // Install the Network Security Config Provider. This must happen before the application
    // code is loaded to prevent issues with instances of TLS objects being created before
    // the provider is installed.
    .................
    NetworkSecurityConfigProvider.install(appContext);
    .................
    if ((data.appInfo.flags&ApplicationInfo.FLAG_LARGE_HEAP) != 0) {
        //如果package中声明了FLAG_LARGE_HEAP，则可跳出虚拟机对内存限制
        dalvik.system.VMRuntime.getRuntime().clearGrowthLimit();
    } else {
        // Small heap, clamp to the current growth limit and let the heap release
        // pages after the growth limit to the non growth limit capacity. b/18387825
        dalvik.system.VMRuntime.getRuntime().clampGrowthLimit();
    }

    // Allow disk access during application and provider setup. This could
    // block processing ordered broadcasts, but later processing would
    // probably end up doing the same disk access.
    final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskWrites();

    try {
        // If the app is being launched for full backup or restore, bring it up in
        // a restricted environment with the base application class.
        // 利用LoadedApk的makeApplication函数，通过反射创建出Application
        Application app = data.info.makeApplication(data.restrictedBackupMode, null);
        mInitialApplication = app;

        // don't bring up providers in restricted mode; they may depend on the
        // app's custom Application class
        if (!data.restrictedBackupMode) {
            if (!ArrayUtils.isEmpty(data.providers)) {
                //加载进程对应Package中携带的ContentProvider
                installContentProviders(app, data.providers);

                // For process that contains content providers, we want to
                // ensure that the JIT is enabled "at some point".
                // 通过JIT技术加速
                mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
            }
        }
        ...................
        try {
            //调用Application的onCreate函数，完成一些初始化工作
            mInstrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            .................
        }
    } finally {
        StrictMode.setThreadPolicy(savedPolicy);
    }
}
```
如上文所述，handleBindApplication的目的是让一个Java进程融入到Android体系中。

因此，该函数中的代码主要进行以下工作：
- 1、按照Android的要求，完成对进程基本参数的设置置，包括设置进程名、时区、资源及兼容性配置；
同时也添加了一些限制，例如主线程不能访问网络等。
- 2、创建进程对应的ContextImpl、LoadedApk、Application等对象，同时加载Application中的ContentProvider，并初始化Application。
当完成上述工作后，新建的进程终于加入到了Android体系。

#### 3 Part-III

接下来，我们看看AMS中attachApplicationLocked函数的最后一部分内容：
```java
.....................
// Remove this record from the list of starting applications.
// 进程已经启动，从一些列表中移除对应的记录
mPersistentStartingProcesses.remove(app);
...................
mProcessesOnHold.remove(app);

boolean badApp = false;
boolean didSomething = false;

// See if the top visible activity is waiting to run in this process...
if (normalMode) {
    try {
        //启动Activity
        if (mStackSupervisor.attachApplicationLocked(app)) {
            didSomething = true;
        }
    } catch (Exception e) {
        Slog.wtf(TAG, "Exception thrown launching activities in " + app, e);
        badApp = true;
    }
}

// Find any services that should be running in this process...
if (!badApp) {
    try {
        //启动因目标进程还未启动，而处于等待状态的service
        didSomething |= mServices.attachApplicationLocked(app, processName);
    } catch (Exception e) {
        Slog.wtf(TAG, "Exception thrown starting services in " + app, e);
        badApp = true;
    }
}

// Check if a next-broadcast receiver is in this process...
if (!badApp && isPendingBroadcastProcessLocked(pid)) {
    try {
        //发送因目标进程还未启动，而处于等待状态的Broadcast
        didSomething |= sendPendingBroadcastsLocked(app);
    } catch (Exception e) {
        // If the app died trying to launch the receiver we declare it 'bad'
        Slog.wtf(TAG, "Exception thrown dispatching broadcasts in " + app, e);
        badApp = true;
    }
}

// Check whether the next backup agent is in this process...
if (!badApp && mBackupTarget != null && mBackupTarget.appInfo.uid == app.uid) {
    ...............
    try {
        //启动backup Agent
        thread.scheduleCreateBackupAgent(mBackupTarget.appInfo,
                compatibilityInfoForPackageLocked(mBackupTarget.appInfo),
                mBackupTarget.backupMode);
        } catch (Exception e) {
            Slog.wtf(TAG, "Exception thrown creating backup agent in " + app, e);
            badApp = true;
        }
    }
}

if (badApp) {
    //如果以上组件启动出错，则需要杀死进程并移除记录
    app.kill("error during init", true);
    handleAppDiedLocked(app, false, true);
    return false;
}

//如果以上没有启动任何组件，那么didSomething为false
if (!didSomething) {
    //调整进程的oom_adj值， oom_adj相当于一种优先级
    //如果应用进程没有运行任何组件，那么当内存出现不足时，该进程是最先被系统“杀死”
    //反之，进程中运行的组件越多，则越不容易被“杀死”
    updateOomAdjLocked();
}

return true;
......
```
这段代码比较好理解，主要功能就是启动新建进程中运行的Activity、Service等组件。

这里我们主要关注一下Activity的启动过程，即下面这段代码：
```java
.............
if (normalMode) {
    try {
        if (mStackSupervisor.attachApplicationLocked(app)) {
            didSomething = true;
        }
    } catch (Exception e) {
        ..........
        badApp = true;
    }
}
...............
```
我们跟进定义于ActivityStackSupervisor中的attachApplicationLocked函数：
```java
boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
    final String processName = app.processName;
    boolean didSomething = false;

    //ActivityStackSupervisor维护着终端中所有Activity和Task之间的关系
    //此处通过轮寻，找出前台栈顶端的待启动Activity
    for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
        ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
        for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
            final ActivityStack stack = stacks.get(stackNdx);
            if (!isFocusedStack(stack)) {
                continue;
            }

            ActivityRecord hr = stack.topRunningActivityLocked();
            if (hr != null) {
                //前台待启动的Activity与当前新建的进程一致时，启动这个Activity
                if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                        && processName.equals(hr.processName)) {
                    try {
                        //realStartActivityLocked进行实际的启动工作
                        if (realStartActivityLocked(hr, app, true, true)) {
                            didSomething = true;
                        }
                    } catch (RemoteException e) {
                        .............
                    }
                }
            }
        }
    }
    .................
    return didSomething;
}
```
通过这段代码，我们终于明白了启动Activity的过程：一中的流程，为什么要花大力气先将待启动的Activity的Task移动到前台，并且要将该Activity移动到栈顶。

毕竟，在创建新进程时，无法将待启动Activity的信息一并传递给新进程（进程刚创建时，并没有加入到Android体系），因此新进程被创建后，无法知道需要创建的Activity。

于是，上面的代码就规定：如果前台栈顶Activity对应的进程信息，与新启动的进程相互吻合时，该进程就需要启动该Activity。

## 三、realStartActivityLocked函数

接下来，我们看看ActivityStatckSupervisor中的realStartActivityLocked函数：
```java
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
        boolean andResume, boolean checkConfig) throws RemoteException {

    if (!allPausedActivitiesComplete()) {
        // While there are activities pausing we skipping starting any new activities until
        // pauses are complete. NOTE: that we also do this for activities that are starting in
        // the paused state because they will first be resumed then paused on the client side.
        ...........
        return false;
    }

    if (andResume) {
        //为显示做准备
        r.startFreezingScreenLocked(app, 0);
        mWindowManager.setAppVisibility(r.appToken, true);
        ..........
    }

    // Have the window manager re-evaluate the orientation of
    // the screen based on the new activity order.  Note that
    // as a result of this, it can call back into the activity
    // manager with a new orientation.  We don't care about that,
    // because the activity is not currently running so we are
    // just restarting it anyway.
    if (checkConfig) {
        Configuration config = mWindowManager.updateOrientationFromAppTokens(
                mService.mConfiguration,
                r.mayFreezeScreenLocked(app) ? r.appToken : null);
        //AMS更新绘制相关的配置信息
        mService.updateConfigurationLocked(config, r, false);
    }

    r.app = app;
    app.waitingToKill = null;
    r.launchCount++;
    ...............
    //将待启动Activity对应ActivityRecord加入到进程中保存
    int idx = app.activities.indexOf(r);
    if (idx < 0) {
        app.activities.add(r);
    }

    //更新优先级
    mService.updateLruProcessLocked(app, true, null);
    mService.updateOomAdjLocked();
    ...............
    final ActivityStack stack = task.stack;
    try {
        ...............
        List<ResultInfo> results = null;
        List<ReferrerIntent> newIntents = null;
        if (andResume) {
            results = r.results;
            newIntents = r.newIntents;
        }
        ...............
        if (andResume) {
            app.hasShownUi = true;
            app.pendingUiClean = true;
        }
        app.forceProcessStateUpTo(mService.mTopProcessState);
        //通知应用进程启动Activity
        app.thread.scheduleLaunchActivity(............);

        if ((app.info.privateFlags&ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0) {
            //处理heavy-weight进程的情况
            .........
        }
    } catch (RemoteException e) {
        if (r.launchFailed) {
            // This is the second time we failed -- finish activity
            // and give up.
            ................
            //从代码来看，第二次启动失败，才会将ActivityRecord中的launchFailed置为true
            mService.appDiedLocked(app);
            stack.requestFinishActivityLocked(r.appToken, Activity.RESULT_CANCELED, null,
                    "2nd-crash", false);
            return false;
        }

        // This is the first time we failed -- restart process and
        // retry.
        app.activities.remove(r);
        throw e;
    }

    r.launchFailed = false;

    if (andResume) {
        // As part of the process of launching, ActivityThread also performs
        // a resume.
        // Activity进入resumeState后，更新相应的状态
        stack.minimalResumeActivityLocked(r);
    } else {
        ................
    }

    // Launch the new version setup screen if needed.  We do this -after-
    // launching the initial activity (that is, home), so that it can have
    // a chance to initialize itself while in the background, making the
    // switch back to it faster and look better.
    if (isFocusedStack(stack)) {
        //启动系统设置向导对应的Activity，当系统更新或初次使用时需要配置
        mService.startSetupActivityLocked();
    }
    ................
    return true;
}
```

从上面的代码可以看出，realStartActivityLocked函数主要工作包括：
- 1、进一步配置ActivityRecord和ProcessRecord；
- 2、调用scheduleLaunchActivity，通知应用进程启动Activity；
- 3、在Activity启动后，AMS调用minimalResumeActivityLocked更新相应的状态。

这里我们只需要进一步看看scheduleLaunchActivity和minimalResumeActivityLocked这两个函数的流程。

### 1、scheduleLaunchActivity

scheduleLaunchActivity定义于ApplicationThread中：
```java
public final void scheduleLaunchActivity(........) {
    //更新进程的状态
    updateProcessState(procState, false);

    ActivityClientRecord r = new ActivityClientRecord();

    //保存AMS传递过来的信息
    r.token = token;
    r.ident = ident;
    r.intent = intent;
    ...........
    //更新本地配置
    updatePendingConfiguration(curConfig);

    //发送消息，即ApplicationThread的工作其实就是与AMS通信
    //实际的处理，还是交给进程主线程的代表ActivityThread处理
    sendMessage(H.LAUNCH_ACTIVITY, r);
}
```
ActivityThread中的handler处理消息的代码如下：
```java
public void handleMessage(Message msg) {
    ...............
    switch (msg.what) {
        case LAUNCH_ACTIVITY: {
            ..............
            final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

            //利用ApplicationInfo等信息得到对应的LoadedApk，保存到ActivityClientRecord
            r.packageInfo = getPackageInfoNoCheck(
                    r.activityInfo.applicationInfo, r.compatInfo);
            //调用handleLaunchActivity
            handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
            ..............
        } break;
        ...........
    }
    .............
}
```
我们跟进一下handleLaunchActivity：
```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    // 如果Activity从前台移动到后台，则有可能准备进行Gc操作
    // 现在Activity重新启动，就需要取消Gc操作了
    // 对于新建Activity而言，此处无实际动作
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    //Activity进行性能统计
    if (r.profilerInfo != null) {
        mProfiler.setProfiler(r.profilerInfo);
        mProfiler.startProfiling();
    }

    // Make sure we are running with the most recent config.
    // 保证Activity以最新的配置启动，即保证Activity符合最新语言、主题、分辨率等的要求
    handleConfigurationChanged(null, null);
    ............
    // Initialize before creating the activity
    WindowManagerGlobal.initialize();

    //1、创建Activity
    Activity a = performLaunchActivity(r, customIntent);
    if (a != null) {
        r.createdConfig = new Configuration(mConfiguration);
        ............
        //2、调用activity的onResume
        handleResumeActivity(.......);

        if (!r.activity.mFinished && r.startsNotResumed) {
            // The activity manager actually wants this one to start out paused, because it
            // needs to be visible but isn't in the foreground. We accomplish this by going
            // through the normal startup (because activities expect to go through onResume()
            // the first time they run, before their window is displayed), and then pausing it.
            // 处理可见但非前台的Activity，这种Activity在启动后将进入到pause状态
            performPauseActivityIfNeeded(r, reason);
        }
        ...................
    } else {
        try {
            //如果启动错误，则通知AMS
            ActivityManagerNative.getDefault()
                    .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                            Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
        } catch (RemoteException ex) {
            ...............
        }
    }
}
```
从上述代码可以看出，handleLaunchActivity的工作主要包括：
- 1、调用performLaunchActivity创建出Activity；
- 2、调用handleResumeActivity，完成调用目标Activity的onResume接口等工作；
- 3、对于可见但非前台的Activity，还需要调用performPauseActivityIfNeeded函数，调用Activity的onPause接口。

我们主要看一下performLaunchActivity和handleResumeActivity函数。

#### 1.1 performLaunchActivity
```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ............
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
        //反射创建Activity
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        ...............
    } catch (Exception e) {
        ...........
    }

    try {
        ..............
        if (activity != null) {
            ..........
            //设置Activity的主要变量，例如mMainThread、mUiThread等
            //mMainThread的类型为ActivityThread；mUiThread的类型为Thread
            //二者实际的工作线程是同一个
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window);
            ...........
            //进行主题的设置
            activity.mStartedActivity = false;
            int theme = r.activityInfo.getThemeResource();
            if (theme != 0) {
                activity.setTheme(theme);
            }

            //调用Activity的onCreate函数
            activity.mCalled = false;
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            .............
            r.activity = activity;
            r.stopped = true;
            if (!r.activity.mFinished) {
                //调用Activity的onStart函数
                activity.performStart();
                r.stopped = false;
            }

            //调用Activity的onRestoreInstanceState函数
            if (!r.activity.mFinished) {
                if (r.isPersistable()) {
                    if (r.state != null || r.persistentState != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                r.persistentState);
                    }
                } else if (r.state != null) {
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                }
            }

            //调用Activity的onPostCreate接口
            if (!r.activity.mFinished) {
                activity.mCalled = false;
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnPostCreate(activity, r.state,
                            r.persistentState);
                } else {
                    mInstrumentation.callActivityOnPostCreate(activity, r.state);
                }
                .............
            }
            r.paused = true;

            //Activity保存到进程中
            mActivities.put(r.token, r);
        }
    } catch (SuperNotCalledException e) {
        .........
    } catch (Exception e) {
        ..........
    }

    return activity;
}
```
performLaunchActivity的功能比较直观，就是利用反射创建出目标Activity，然后设置Activity的内部变量，最后依次调用Activity生命周期中的接口，主要包括onCreate、onStart等。

#### 1.2 handleResumeActivity

接下来，看看handleResumeActivity的工作：
```java
final void handleResumeActivity(.......) {
    ActivityClientRecord r = mActivities.get(token);
    ..............
    //主要是调用目标Activity的onResume函数
    r = performResumeActivity(token, clearHide, reason);

    if (r != null) {
        final Activity a = r.activity;
        //为绘制做准备工作
        ..............
        if (!r.activity.mFinished && willBeVisible
                 && r.activity.mDecor != null && !r.hideForNow) {
            //进行绘制相关的工作
            .............
        }

        if (!r.onlyLocalRequest) {
            //可以看出ActivityThread用Stack的方式，保存完成onResume的Activity

            //r为刚刚完成onResume的新Activity，mNewActivities保存已经完成onResume的Activity
            //新的Activity的nextIdle指向旧的
            r.nextIdle = mNewActivities;

            //然后将新建的Activity保存到mNewActivities中
            mNewActivities = r;
            ...........
            //向ActivityThread的MessageQueue中增加一个IdleHandler
            Looper.myQueue().addIdleHandler(new Idler());
        }
        r.onlyLocalRequest = false;

        // Tell the activity manager we have resumed.
        if (reallyResume) {
            try {
                //通知AMS Activity进入Resume状态
                //AMS会修改对应的存储信息
                ActivityManagerNative.getDefault().activityResumed(token);
            } catch (RemoteException ex) {
                ............
            }
        }
    } else {
        ...........
    }
}
```
从上面的代码可以看出，handleResumeActivity函数将会调用Activity的onResume接口，并进行绘制相关的操作，
然后向Activity的MessageQueue中增加一个IdleHandler，最终通知AMS更新Activity的状态。

在之前的博客Android7.0 MessageQueue 中，我们提到过当MessagQueue的next函数被调用时，如果队列中没有message需要处理，那么将调用添加到MessageQueue的IdleHandler的queueIdle接口。

handleResumeActivity函数向ActivityThread增加了一个IdleHandler，我们看看它的作用：
```java
private class Idler implements MessageQueue.IdleHandler {
    public final boolean queueIdle() {
        ActivityClientRecord a = mNewActivities;
        ..................
        if (a != null) {
            mNewActivities = null;
            IActivityManager am = ActivityManagerNative.getDefault();
            ActivityClientRecord prev;
            do {
                ............
                if (a.activity != null && !a.activity.mFinished) {
                    try {
                        //调用AMS的activityIdle函数
                        am.activityIdle(a.token, a.createdConfig, stopProfiling);
                        a.createdConfig = null;
                    } catch (RemoteException ex) {
                        ..............
                    }

                    //do-while循环将处理所有的已经完成onResume的Activity
                    prev = a;
                    a = a.nextIdle;
                    prev.nextIdle = null;
                }
            }while (a != null);
            ..........
        }
        .........
        //返回值为false，于是执行一次后，会被移除
        return false;
    }
}
```
从上面的代码可以看出，当ActivityThread空闲下来后，将为所有完成onResume的Activity调用AMS的activityIdle函数。
该函数是Activity成功创建并启动流程中的最后一步，我们稍后再分析。

至此，scheduleLaunchActivity函数分析完毕，我们看看realStartActivityLocked函数的下一个关键点minimalResumeActivityLocked。

### 2、minimalResumeActivityLocked

minimalResumeActivityLocked定义于ActivityStack中。ActivityStack是Activity所在Task中，保存ActivityRecord的数据结构。
我们看看minimalResumeActivityLocked的代码：
```java
void minimalResumeActivityLocked(ActivityRecord r) {
    r.state = ActivityState.RESUMED;
    .............
    mResumedActivity = r;
    r.task.touchActiveTime();

    //mRecentTasks保存近期被调用的Task
    mRecentTasks.addLocked(r.task);

    //更新一些状态
    completeResumeLocked(r);

    //判断AMS当前维护的Activity对应的进程是否可以sleep
    //若可以sleep，将调用对应ApplicationThread的scheduleSleeping函数
    mStackSupervisor.checkReadyForSleepLocked();
    setLaunchTime(r);
    ............
}
```
上述代码中主要的工作由completeResumeLocked来完成：
```java
/**
* Once we know that we have asked an application to put an activity in
* the resumed state (either by launching it or explicitly telling it),
* this function updates the rest of our state to match that fact.
*/
private void completeResumeLocked(ActivityRecord next) {
    //更新ActivityRecord的变量
    next.visible = true;
    next.idle = false;
    ............

    if (next.isHomeActivity()) {
        //对Home Activity的特殊处理
        ..........
    }

    if (next.nowVisible) {
        // We won't get a call to reportActivityVisibleLocked() so dismiss lockscreen now.
        //如果已经visible将进行通知
        mStackSupervisor.reportActivityVisibleLocked(next);
        mStackSupervisor.notifyActivityDrawnForKeyguard();
    }

    // schedule an idle timeout in case the app doesn't do it for us.
    //发送一个延迟消息IDLE_TIMEOUT_MSG，延迟时间为10s
    //这里就是要求Activity启动后，10s内要调用AMS的activityIdle函数

    //不过从代码来看，即使Activity 10s内没有调用activityIdle函数，
    //ActivityStackSupervisor也会自己调用activityIdleInternalLocked函数
    mStackSupervisor.scheduleIdleTimeoutLocked(next);

    mStackSupervisor.reportResumedActivityLocked(next);
    ......................
}
```
从这部分代码，可以看出minimalResumeActivityLocked函数整体上就是负责更新一些状态，不用过于在意它的细节。

## 四、activityIdle函数

现在，我们看一下Activity启动的最后一步，即调用AMS的activityIdle函数：
```java
public final void activityIdle(IBinder token, Configuration config, boolean stopProfiling) {
    ................
    synchronized (this) {
        ActivityStack stack = ActivityRecord.getStackLocked(token);
        if (stack != null) {
            //主要调用ActivityStackSupervisor的activityIdleInternalLocked函数
            ActivityRecord r =
                    mStackSupervisor.activityIdleInternalLocked(token, false, config);
            .................
        }
    }
    ..............
}
```
跟进activityIdleInternalLocked函数：
```java
//ActivityThread在超时时间内，调用activityIdle时，fromTimeout的值为false
//如果超时后，由ActivityStackSupervisor主动调用该函数，fromTimeout的值为true
final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
        Configuration config) {
    ................
    ActivityRecord r = ActivityRecord.forTokenLocked(token);
    if (r != null) {
        ....................
        mHandler.removeMessages(IDLE_TIMEOUT_MSG, r);
        ....................
        if (fromTimeout) {
            //超时时，将通知Activity完成启动
            reportActivityLaunchedLocked(fromTimeout, r, -1, -1);
        }

        if (config != null) {
            r.configuration = config;
        }

        r.idle = true;
        ................
    }

    //当所有Resumed Activity都处于空闲状态时
    if (allResumedActivitiesIdle()) {
        if (r != null) {
            mService.scheduleAppGcsLocked();
        }

        //mLaunchingActivity是一个WakeLock，用于防止在操作Activity的过程中休眠
        //由于该WakeLock不能长时间使用，因此系统设置了一个超时消息LAUNCH_TIMEOUT_MSG
        //处理该消息时，将释放该WakeLock
        //此处，当所有Activity均空闲时，说明事件处理完毕，此时需要释放该WakeLock
        if (mLaunchingActivity.isHeld()) {
            mHandler.removeMessages(LAUNCH_TIMEOUT_MSG);
            ...............
            mLaunchingActivity.release();
        }
        ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
    }

    // Atomically retrieve all of the other things to do.
    //processStoppingActivitiesLocked返回那些因本次Activity启动而被暂停(paused)的Activity
    final ArrayList<ActivityRecord> stops = processStoppingActivitiesLocked(true);
    NS = stops != null ? stops.size() : 0;

    if ((NF = mFinishingActivities.size()) > 0) {
        //finishes保存等待结束的Activity
        finishes = new ArrayList<>(mFinishingActivities);
        mFinishingActivities.clear();
    }

    .................
    // Stop any activities that are scheduled to do so but have been
    // waiting for the next one to start.
    for (int i = 0; i < NS; i++) {
        r = stops.get(i);
        final ActivityStack stack = r.task.stack;
        if (stack != null) {
            if (r.finishing) {
                //如果被暂停的Activity已经处于了finishing状态，则通知它们执行Destroy操作，即Activity的onDestroy函数将被调用
                stack.finishCurrentActivityLocked(r, ActivityStack.FINISH_IMMEDIATELY, false);
            } else {
                //否则通知它们执行stop操作，即Activity的onStop将被调用
                stack.stopActivityLocked(r);
            }
        }
    }

    // Finish any activities that are scheduled to do so but have been
    // waiting for the next one to start.
    // 处理等待结束的Activity，从代码来看最终也是调用其onDestroy接口
    for (int i = 0; i < NF; i++) {
        r = finishes.get(i);
        final ActivityStack stack = r.task.stack;
        if (stack != null) {
            activityRemoved |= stack.destroyActivityLocked(r, true, "finish-idle");
        }
    }
    ................
    //AMS清理没有任何组建的无用进程
    mService.trimApplications();

    if (activityRemoved) {
        //如果等待结束的Activity被处理，保证前台栈顶Activity处于Resumed状态
        resumeFocusedStackTopActivityLocked();
    }

    return r;
}
```
至此，启动Activity最后一部分的主要工作介绍完毕。

从代码来看，这一部分最主要的工作是：

进行一些扫尾工作，即根据后台Activity是否处于finishing状态，分别调用其onStop和onDestroy接口。
同时，AMS也会清理一些无实际工作的进程。

最后需要补充的是：

我们利用am start -W启动Activity时，需要等待返回结果，详细代码可以参考启动Activity的过程：一。

在ActivityStarter的startActivityMayWait函数中：
```java
final int startActivityMayWait(.........) {
    .............
    //outResult != null，说明启动命令发送成功
    if (outResult != null) {
        outResult.result = res;
        if (res == ActivityManager.START_SUCCESS) {
            mSupervisor.mWaitingActivityLaunched.add(outResult);
            do {
                try {
                    //等待Task移动到前台
                    mService.wait();
                } catch (InterruptedException e) {
                }
            } while (outResult.result != START_TASK_TO_FRONT
                    && !outResult.timeout && outResult.who == null);
            if (outResult.result == START_TASK_TO_FRONT) {
                res = START_TASK_TO_FRONT;
            }
        }

        if (res == START_TASK_TO_FRONT) {
            ActivityRecord r = stack.topRunningActivityLocked();
            if (r.nowVisible && r.state == RESUMED) {
                outResult.timeout = false;
                outResult.who = new ComponentName(r.info.packageName, r.info.name);
                outResult.totalTime = 0;
                outResult.thisTime = 0;
            } else {
                outResult.thisTime = SystemClock.uptimeMillis();
                mSupervisor.mWaitingActivityVisible.add(outResult);
                do {
                    try {
                        //等待Activity可见
                        mService.wait();
                    } catch (InterruptedException e) {
                    }
                } while (!outResult.timeout && outResult.who == null);
             }
        }
    }
    ..................
}
```
上面代码中，startActivityMayWait等待的信号均由ActivityStackSupervisor触发。

在启动Activity的过程：一中简单提到过，在startActivityLocked函数的最后，调用了postStartActivityUncheckedProcessing函数进行通知：
```java
void postStartActivityUncheckedProcessing(......) {
    .............
    // We're waiting for an activity launch to finish, but that activity simply
    // brought another activity to front. Let startActivityMayWait() know about
    // this, so it waits for the new activity to become visible instead.
    if (result == START_TASK_TO_FRONT && !mSupervisor.mWaitingActivityLaunched.isEmpty()) {
        mSupervisor.reportTaskToFrontNoLaunch(mStartActivity);
    }
    ............
}
```
我们看ActivityStackSupervisor的reportTaskToFrontNoLaunch函数：
```java
void reportTaskToFrontNoLaunch(ActivityRecord r) {
    boolean changed = false;
    for (int i = mWaitingActivityLaunched.size() - 1; i >= 0; i--) {
        WaitResult w = mWaitingActivityLaunched.remove(i);
        //who表示componentName，Activity未可见前为null
        if (w.who == null) {
            changed = true;
            // Set result to START_TASK_TO_FRONT so that startActivityMayWait() knows that
            // the starting activity ends up moving another activity to front, and it should
            // wait for this new activity to become visible instead.
            // Do not modify other fields.
            w.result = START_TASK_TO_FRONT;
        }
    }
    if (changed) {
        //notifyAll通知ActivityStarter待启动Activity对应的Task移动到了前台
        mService.notifyAll();
    }
}
```
同样，当Activity可见时，WindowManagerService将回调ActivityRecord中的windowsVisible函数：
```java
public void windowsVisible() {
    synchronized (mService) {
        ActivityRecord r = tokenToActivityRecordLocked(this);
        if (r != null) {
            r.windowsVisibleLocked();
        }
    }
}

void windowsVisibleLocked() {
    mStackSupervisor.reportActivityVisibleLocked(this);
    ...........
}
```

跟进一下ActivityStackSupervisor的reportActivityVisibleLocked函数：
```java
void reportActivityVisibleLocked(ActivityRecord r) {
    sendWaitingVisibleReportLocked(r);
}

void sendWaitingVisibleReportLocked(ActivityRecord r) {
    boolean changed = false;
    for (int i = mWaitingActivityVisible.size()-1; i >= 0; i--) {
        WaitResult w = mWaitingActivityVisible.get(i);
        if (w.who == null) {
            changed = true;
            w.timeout = false;
            if (r != null) {
                //构造出ComponentName等信息
                w.who = new ComponentName(r.info.packageName, r.info.name);
            }
            w.totalTime = SystemClock.uptimeMillis() - w.thisTime;
            w.thisTime = w.totalTime;
        }
    }
    if (changed) {
        //notifyAll通知ActivityStarter待启动Activity可见
        mService.notifyAll();
    }
}
```
## 五、总结


![图1](/images/android-n-ams/start-2-1.jpg)

至此，AMS启动Activity的后一部分基本介绍完毕。
这一部分的代码整体流程如上图所示，我们省略一些细节，例如绘制等。

从整个逻辑来看，这部分流程比较清晰，主要思想如下：
- 1、启动进程，创建自己的ActivityThread和ApplicationThread。
- 2、进程与AMS通信，其实就是完成一个注册过程（将ApplicationThread作为Binder通信接口交给AMS保存），AMS毕竟要统一管理系统中所有的应用进程。
- 3、AMS通知进程创建自己的Android运行环境，加入到Android体系。
- 4、AMS通知进程可以启动进程中的组件了，注意四大组件的顺序以此是ContentProvider、Activity、Service、BroadcastReceiver（参考上述代码，可以容易看出这个结论，本文我们仅分析了Activity的启动过程）。
- 5、Activity启动完成后，通知AMS更新相关的状态，并进行一些扫尾工作。

整体而言，所有的实际工作都是由进程自己完成的，AMS仅起到一个管理的作用。
