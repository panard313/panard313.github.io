# Android后台优化
文章译自：https://developer.android.com/topic/performance/background-optimization.html#connectivity-action

后台进行需要更高的内存和电量。例如，一个隐性的广播也许会唤起多个监听它的后台进程，尽管这些进程没错做什么，这也会对设备性能和用户体验有一个实质性的影响。

为了缓解这一问题，Android 7.0(API level 24) 使用以下限制:

- 对于 Android 7.0 (API level 24)的app，如果在manifest文件中注册了receive.不再接收 CONNECTIVITY_ACTION 广。通过Context.registerReceiver()注册的BroadcastReceiver，运行时的app任可以在主线程监听CONNECTIVITY_CHANGE广播。
- App不能发送或者接收 ACTION_NEW_PICTURE 和 ACTION_NEW_VIDEO 广播.这个优化会影响所有app,不仅仅只针对 Android 7.0 (API level 24).
- 如果你的app使用了这些intent，你应该尽量避免使用，以便你的设备可以正确的运行 Android 7.0。Android framework提供了一些办法来缓解这种隐性广播的需求。例如，当遇到指定的条件时，比如，一个非计费的网络连接，JobScheduler和GcmNetworkManager提供了健全的机制来处理网络操作。你也可以使用JobScheduler对内容提供商的变化做出反应。JobInfo对象封装了JobScheduler参数用于调度job。当job的条件满足时，系统将会在JobService上执行job。

在本文中，我们将学习如何使用可选择的方法，例如JobScheduler,使得你的app适应这些新的限制。

## 关于CONNECTIVITY_ACTION的限制

&emsp;&emsp;对于Android 7.0(API level 24)的app,如果在manifest文件中注册了监听 CONNECTIVITY_ACTION 的receiver，也是不会收到 CONNECTIVITY_ACTION 广播的，这依赖于这种广播不会开启。对于需要监听网络变化或者执行批量网络活动的app，这将会产生一个问题。在Android framwork中围绕这个限制已经有一些解决办法。选择一个正确的方法取决于你希望你的应用程序完成什么功能。 
注意，在程序中使用 Context.registerReceiver()注册的 BroadcastReceiver，在程序运行时仍可以接收这些广播。

1. **Scheduling Network Jobs on Unmetered Connections**

当使用JobInfo.Builder类来构建JobInfo对象时，使用setRequiredNetWorkType()方法并将JobInfo.NETWORK_TYPE_UNMETERED 作为参数传递给该方法。下面的示例代码展示当设备连接到一个非计费的网络并在充电时启动一个service。
```java
public static final int MY_BACKGROUND_JOB = 0;
...
public static void scheduleJob(Context context) {
  JobScheduler js =
      (JobScheduler) context.getSystemService(Context.JOB_SCHEDULER_SERVICE);
  JobInfo job = new JobInfo.Builder(
    MY_BACKGROUND_JOB,
    new ComponentName(context, MyJobService.class))
      .setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED)
      .setRequiresCharging(true)
      .build();
  js.schedule(job);
}
//权限android:permission="android.permission.BIND_JOB_SERVICE"  
```
当你的job条件满足时，你的app将会接收一个回调运行JobService.class类中的onStartJob()方法。

使用GMSCore 服务，Android 5.0 (API level 21) 或者更低版本的应用程序，可以使用GcmNetworkManager并指定Task.NETWORK_STATE_UNMETERED.

2. **Monitoring Network Connectivity While the App is Running**

运行中的app仍可以接受 CONNECTIVITY_CHANGE。 ConnectivityManager API提供了更加健全的方法，当指定的网络条件满足时，会发生一个回调。

NetworkRequest对象依据NetworkCapabilities定义回调参数。你可以使用NetworkRequest.Builder类创建NetworkRequest对象。使用registerNetworkCallback()把NetworkRequest对象传递给系统。当网络条件满足时，app会接收一个回调 执ConnectivityManager.NetworkCallback 中的onAvailable()方法。

app会持续接受回调，直到app退出或者调用 unregisterNetworkCallback().方法。

## 关于NEW_PICTURE 和NEW_VIDEO的限制

对于Android 7.0 (API level 24)，app不能发送或者接收 ACTION_NEW_PICTURE 和 ACTION_NEW_VIDEO 广播。当一些app必须被唤醒来处理新的图片和视频时，这种限制有利于缓解对性能和用户体验的影响。Android 7.0中扩展了JobInfo和JobParameters，提供一种可选择的解决方案。

1. **New JobInfo methods**

为了在content URI 改变时触发Job, Android 7.0 (API level 24) 扩展了JobInfo API ，下面是这些方法：

- JobInfo.TriggerContentUri() 
Encapsulates parameters required to trigger a job on content URI changes.

- JobInfo.Builder.addTriggerContentUri() 
传递TriggerContentUri对象给JobInfo。一个contentobserver监听封装的content URI。如果有多个TriggercontentUri对象与一个job关联系，统将会提供一个回调。

- 如果任何给定URI的继承者改变，增加 TriggerContentUri.FLAG_NOTIFY_FOR_DESCENDANTS标志，就会触发job,这种标志相当于给registerContentObserver()传递notifyForDescendants参数。

注意：TriggerContentUri不能和setPeriodic()以及setPersisted()方法一起使用。为了持续监听content的变化，在app的JobService完成最近的回调之前，启动一个新的JobInfo。

下面是示例代码，当系统报出一个MEDIA_RUI变化时触发一个job。
```java
public static final int MY_BACKGROUND_JOB = 0;
...
public static void scheduleJob(Context context) {
  JobScheduler js =
          (JobScheduler) context.getSystemService(Context.JOB_SCHEDULER_SERVICE);
  JobInfo.Builder builder = new JobInfo.Builder(
          MY_BACKGROUND_JOB,
          new ComponentName(context, MediaContentJob.class));
  builder.addTriggerContentUri(
          new JobInfo.TriggerContentUri(MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
          JobInfo.TriggerContentUri.FLAG_NOTIFY_FOR_DESCENDANTS));
  js.schedule(builder.build());
}
```
当系统报出一个指定的content URI改变，你的app会接受一个回调并且JobParameters对象会被传递给MediaContentJob.class中的onStartJob()方法

2. **New JobParameter Methods**

Android 7.0 (API level 24)也扩展了 JobParameters， 它允许你的app可以接受有用的信息， 这些信息是关于哪些authority和URI触发了job。

- Uri[] getTriggeredContentUris() 
返回触发job的URI数组。如果没有URI触发job这个方法将会返回null(例如job由于截止期限或者其他原因触发)，或者改变的URI超过50个也会返回null。
- String[] getTriggeredContentAuthorities() 
这个方法返回一个触发了job的content authority的字符串数字。如果返回的数组不是null,可以使用getTriggeredContentUris()来检索URI的改变的详细信息。
下面的代码重写了 JobService.onStartJob()方法并记录了触发job的authority和URI。
```java
@Override
public boolean onStartJob(JobParameters params) {
  StringBuilder sb = new StringBuilder();
  sb.append("Media content has changed:\n");
  if (params.getTriggeredContentAuthorities() != null) {
      sb.append("Authorities: ");
      boolean first = true;
      for (String auth :
          params.getTriggeredContentAuthorities()) {
          if (first) {
              first = false;
          } else {
             sb.append(", ");
          }
           sb.append(auth);
      }
      if (params.getTriggeredContentUris() != null) {
          for (Uri uri : params.getTriggeredContentUris()) {
              sb.append("\n");
              sb.append(uri);
          }
      }
  } else {
      sb.append("(No content)");
  }
  Log.i(TAG, sb.toString());
  return true;
}
```
# 未来的优化

&emsp;&emsp;优化你的app在低内存设备或者低内存条件下运行,可以改善性能和用户体验。删除掉后台的service以及静态隐性注册广播接收者能够使你的app在这些设备上更好的运行。Android 7.0 (API level 24) 正在逐渐减少这些问题。推荐优化你的app完全不要使用这些后台进程。

Android 7.0 (API level 24) 有一些额外的 Android Debug Bridge (ADB) 命令，你可以使用这些命令在不使用后台进程情况下测试app的行为。 
模拟隐性的广播和后台服务不可用，使用下面的命令：

    $ adb shell cmd appops set <package_name> RUN_IN_BACKGROUND ignore

为了使隐性的广播和后台服务再次可用，使用下面的命令：

    $ adb shell cmd appops set <package_name> RUN_IN_BACKGROUND allow
