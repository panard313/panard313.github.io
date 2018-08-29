---
layout: article
title: 广播(Broadcast)相关流程分析
key: 20180828
tags:
  - ActivityManagerService
  - ams
  - activity start
lang: zh-Hans
---

# 广播(Broadcast)相关流程分析


## 一、基础知识
广播（Broadcast）是一种Android组件间的通信方式。

从本质上来看，广播信息的载体是intent。在这种通信机制下，发送intent的对象就是广播发送方，接收intent的对象就是广播接收者。
在Android中，为广播接收者定义了一个单独的组件：BroadcastReceiver。

### 1 BroadcastReceiver的注册类型
在监听广播前，要将BroadcastReceiver注册到系统中。

BroadcastReceiver总体上可以分为两种注册类型：静态注册和动态注册。

#### 静态注册
静态注册是指：通过在AndroidManifest.xml中声明receiver标签，来定义BroadcastReceiver。

PKMS初始化时，通过解析Application的AndroidManifest.xml，就能得到所有静态注册的BroadcastReceiver信息。

当广播发往这种方式注册的BroadcastReceiver时，若该BroadcastReceiver对应的进程没有启动，
AMS需要先启动对应的进程，然后利用Java反射机制构造出BroadcastReceiver，然后才能开始后续的处理。

在AndroidManifest.xml中定义BroadcastReceiver时，对应的标签类似于如下形式：
```xml
<receiver android:enabled=["true" | "false"]
    android:exported=["true" | "false"]
    android:icon="drawable resource"
    android:label="string resource"
    android:name="string"
    android:permission="string"
    android:process="string" >
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```
其中：
- android:enabled表示此broadcastReceiver是否可用，默认值为true。

- android:exported表示此broadcastReceiver能否接收其它App的发出的广播；
如果标签中定义了intent-filter字段，则此值默认值为true，否则为false。

- android:name表示BroadcastReceiver的类名。

- android:permission用于指定广播发送方应该具有的权限；
即具有对应权限的发送者，才能通过AMS将广播发送给此broadcastReceiver；

- android:process表示broadcastReceiver运行的进程；
BroadcastReceiver默认运行在当前app的进程中，也可以通过此字段指定其运行于其它独立的进程。

- intent-filter用于指定该broadcastReceiver接收广播的类型。

#### 动态注册
与静态注册不同，动态注册是指：

应用程序在运行过程中，调用Context的registerReceiver函数注册BroadcastReceiver；

当应用程序不再需要监听广播时，则需要调用unregisterReceiver函数进行反注册。

动态注册BroadcastReceiver时，需要指定对应的IntentFilter。
IntentFilter用于描述该BroadcastReceiver期待接受的广播类型。
同时，动态注册时也可以指定该BroadcastReceiver要求的权限。

### 2 广播的种类

从广播的发送特点来看，可以将Android中定义的广播分为以下几类：

#### 普通广播

普通广播由发送方调用sendBroadcast及相关重载函数发送。

AMS转发这种类型的广播时，根据BroadcastReceiver的注册方式，进行不同的处理流程。
- 对于动态注册的广播，理论上可以认为，AMS将在同一时刻，向所有监听此广播的BroadcastReceiver发送消息，因此整体的消息传递的效率比较高。
- 对于静态注册的广播，AMS将按照有序广播的方式，向BroadcastReceiver发送消息。

#### 有序广播

有序广播由发送方调用sendOrderedBroadcast及相关重载函数发送，是一种串行的广播发送方式。

处理这种类型的广播时，AMS会按照优先级，将广播依次分发给BroadcastReceiver。

AMS收到上一个BroadcastReceiver处理完毕的消息后，才会将广播发送给下一个BroadcastReceiver。

其中，任意一个BroadcastReceiver，都可以中止后续的广播分发流程。

同时，上一个BroadcastReceiver可以将额外的信息添加到广播中。

前文已经指出，当普通广播发送给静态注册的BroadcastReceiver时，AMS实际上是按照有序广播的方式来进行发送，这么做的原因是：

静态注册的BroadcastReceiver仅申明在AndroidManifest.xml中，因此其所在的进程可能并没有启动。

当AMS向这些BroadcastReceiver发送消息时，可能必须先启动对应的进程。
如果同时向静态注册的BroadcastReceiver发送广播，那么可能需要在一段时间内同时创建出大量的进程，
这将对系统造成极大的负担。

若以有序广播的方式来发送，那么系统可以依次创建进程，
同时，每次收到上一个广播处理完毕的消息后，都可以尝试清理掉无用的进程。
这样即可以避免突发创建大量的进程，又可以及时回收一些系统资源。

因此从Android的设计方式可以看出，从接收广播的效率来讲：
- 排在第一的是，接收普通广播的动态注册的BroadcastReceiver，
- 其次是，接收有序广播的动态注册的BroadcastReceiver；
- 最后是，静态注册的BroadcastReceiver，此时接收的广播是否有序，已经不重要了。

实际上，源码就是按这种顺序来处理的，我们后面将进行深入分析。

#### 粘性广播
粘性广播由发送方调用sendStickyBroadcast及相关重载函数发送。

需要注意的是，目前这种广播已经被附上Deprecated标签了，不再建议使用。

Android设计这种广播的初衷是：

正常情况下，假设发送方发送了一个普通广播A。
在广播A发送完毕后，系统中新注册了一个BroadcastReceiver，此时这个BroadcastReceiver是无法收到广播A的。

但是，如果发送方发送的是一个粘性广播B，那么系统将负责存储该粘性广播。
于是，即使BroadcastReceiver在广播B发送完后才注册到系统，这个BroadcastReceiver也会立即收到AMS转发的广播B。

粘性广播和有序广播等概念实际上不是冲突的。
粘性仅仅强调系统将会保存这个广播，它的其它处理过程与上文一致。

### 3 适时地限制广播发送范围
从上文可知，对于定义了intent-filter的BroadcastReceiver而言，其exported属性默认为true。
这种设计是符合预期的，毕竟我们使用广播的目的，就是使得属于不同应用、不同进程的组件能够彼此通信。

但在某些场景下，这种设计也可能带来一些安全隐患：
- 1.其它App可能会针对性的发出与当前App intent-filter相匹配的广播，导致当前App不断接收到广播并处理；
- 2.其它App可以注册与当前App一致的intent-filter用于接收广播，获取广播具体信息。

因此，在必要的时候，需要适时地限制广播发送范围，例如：
- 1.对于同一App内部发送和接收广播，将exported属性设置成false；
- 2.在广播发送和接收时，都增加上相应的permission；
- 3.发送广播时，指定BroadcastReceiver对应的包名。

通过以上三种方式，可以缩小广播的发送和接收范围，提高效率和安全性。

实际上，在framework/support/v4/java/android/support/v4/content目录下，Android定义了LocalBroadcastManager类，用于发送和接收仅在同一个App应用内传递的广播。

这个类仅针对动态注册的BroadcastReceiver有效，具体的使用方式与普通的全局广播类似，
只是注册/反注册BroadcastReceiver和发送广播时，将Context变成了LocalBroadcastManager。

举例如下：
```java
//registerReceiver
localBroadcastManager = LocalBroadcastManager.getInstance(this);
localBroadcastManager.registerReceiver(mBroadcastReceiver, intentFilter);
..........

//unregisterReceiver
localBroadcastManager.unregisterReceiver(mBroadcastReceiver);
..........

//send broadcast
Intent intent = new Intent();
intent.setAction("Test");
localBroadcastManager.sendBroadcast(intent);
...........
```
以上是Android中关于广播的基础知识，接下来进入到实际的源码分析。

## 二、registerReceiver流程分析
现在我们看一下广播相关的源码，先从BroadcastReceiver的动态注册流程开始分析。

### 1 ContextImpl中的registerReceiver

动态注册BroadcastReceiver，将使用Context中定义的registerReceiver函数，具体的实现定义于ContextImpl中：
```java
//这是最常用的接口
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
    return registerReceiver(receiver, filter, null, null);
}

//broadcastPermission与静态注册中的permission标签对应，用于对广播发送方的权限进行限制
//只有拥有对应权限的发送方，发送的广播才能被此receiver接收

//不指定scheduler时，receiver收到广播后，将在主线程调用onReceive函数
//指定scheduler后，onReceive函数由scheduler对应线程处理
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
        String broadcastPermission, Handler scheduler) {

    //ContextImpl是Context家族中实际工作的对象，getOuterContext得到的是ContextImpl对外的代理
    //一般为Application、Activity、Service等
    return registerReceiverInternal(receiver, getUserId(),
            filter, broadcastPermission, scheduler, getOuterContext());
}
```
与上述代码类似，不论调用Context中的哪个接口注册BroadcastReceiver，最终流程均会进入到registerReceiverInternal函数。

现在，我们跟进一下registerReceiverInternal函数：
```java
private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
        IntentFilter filter, String broadcastPermission,
        Handler scheduler, Context context) {
    IIntentReceiver rd = null;

    if (receiver != null) {
        //mPackageInfo的类型为LoadedApk
        if (mPackageInfo != null && context != null) {
            if (scheduler == null) {
                //未设置scheduler时，将使用主线程的handler处理
                //这就是默认情况下，Receiver的onReceive函数在主线程被调用的原因
                scheduler = mMainThread.getHandler();
            }

            //getReceiverDispatcher函数的内部，利用BroadcastReceiver构造出ReceiverDispatcher
            //返回ReceiverDispatcher中的IIntentReceiver对象
            //IIntentReceiver是注册到AMS中，供AMS回调的Binder通信接口
            rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
        } else {
            //这段代码写的我瞬间懵逼了。。。scheduler == null对应的判断，明显可以移出if-else结构的
            //不符合大google的逼格
            if (scheduler == null) {
                scheduler = mMainThread.getHandler();
            }
            //主动创建
            rd = new LoadedApk.ReceiverDispatcher(
                    receiver, context, scheduler, null, true).getIIntentReceiver();
        }
    }

    try {
        //将Receiver注册到AMS
        final Intent intent = ActivityManagerNative.getDefault().registerReceiver(
                mMainThread.getApplicationThread(), mBasePackageName,
                rd, filter, broadcastPermission, userId);
        .................
    } catch (RemoteException e) {
        ...........
    }
}
```

以上代码中主要的工作是：

创建BroadcastReceiver对应的ReceiverDispatcher，得到ReceiverDispatcher内部的IIntentReceiver对象，
然后利用Binder通信将该对象连同BroadcastReceiver的其它信息注册到AMS中。

IIntentReceiver对应的继承关系如下图所示：

![图1](/images/android-n-ams/broadcast-1.jpg)

结合上图，我们需要知道的是：
- 1、BroadcastReceiver收到的广播实际上是AMS发送的，因此BroadcastReceiver与AMS之间必须进行Binder通信。

从类图可以看出，BroadcastReceiver及其内部成员并没有直接继承Binder类。

负责与AMS通信的，实际上是定义于LoadedApk中的InnerReceiver类，该类继承IIntentReceiver.Stub，作为Binder通信的服务端。
从上文的代码可以看出，注册BroadcastReceiver时，会将该对象注册到AMS中供其回调。

当InnerReceiver收到AMS的通知后，将会调用ReceiverDispatcher进行处理。
由于ReceiverDispatcher持有了实际的BroadcastReceiver，于是最终将广播递交给BroadcastReceiver的onReceive函数进行处理。

以上的设计思路实际上是：

将业务接口（处理广播的BroadcastReceiver类）与通信接口（InnerReceiver类）分离，由统一的管理类（ReceiverDispatcher）进行衔接。
这基本是Java层使用Binder通信的标配，AIDL本身也是这种思路。

- 2、BroadcastRecevier中定义了一个PendingResult类，该类用于异步处理广播消息。

从上文的代码我们知道了，如果没有明确指定BroadcastReceiver对应的handler，那么其onReceive函数将在主线程中被调用。
因此当onReceive中需要执行较耗时的操作时，会阻塞主线程，影响用户体验。
为了解决这种问题，可以在onReceive函数中采用异步的方式处理广播消息。

一提到异步的方式处理广播消息，大家可能会想到在BroadcastReceiver的onReceive中单独创建线程来处理广播消息。
然而，这样做存在一些问题。

例如：

对于一个静态注册的BroadcastReceiver 对象，对应的进程初始时可能并没有启动。
当AMS向这个BroadcastReceiver对象发送广播时，才会先启动对应的进程。
一旦BroadcastReceiver对象处理完广播，i并将返回结果通知给AMS后，AMS就有可能清理掉对应进程。
因此，若在onReceive中创建线程处理问题，那么onReceive函数就可能在线程完成工作前返回，导致AMS提前销毁进程。
此时，线程也会消亡，使得工作并没有有效完成。

为了解决这个问题，就可以用到BroadcastReceiver提供的PendingResult。

具体的做法是：

先调用BroadcastReceiver的goAsync函数得到一个PendingResult对象，
然后将该对象放到工作线程中去释放。
这样onReceive函数就可以立即返回而不至于阻塞主线程。
同时，Android系统将保证BroadcastReceiver对应进程的生命周期，
直到工作线程处理完广播消息后，调用PendingResult的finish函数为止。

其中原理是：

正常情况下，BroadcastReceiver在onReceive函数结束后，
判断PendingResult不为null，才会将处理完毕的信息通知给AMS。

一旦调用BroadcastReceiver的goAsync函数，就会将BroadcastReceiver中的PendingResult置为null，
因此即使onReceive函数返回，也不会将信息通知给AMS。
AMS也就不会处理BroadcastReceiver对应的进程。

待工作线程调用PendingResult的finish函数时，才会将处理完毕的信息通知给AMS。

代码示例如下：
```java
private class MyBroadcastReceiver extends BroadcastReceiver {
    ..................
    public void onReceive(final Context context, final Intent intent) {
        //得到PendingResult
        final PendingResult result = goAsync();

        //放到异步线程中执行
        AsyncHandler.post(new Runnable() {
            @Override
            public void run() {
                handleIntent(context, intent);//可进行一些耗时操作
                result.finish();
            }
        });
    }
}

final class AsyncHandler {
    private static final HandlerThread sHandlerThread = new HandlerThread("AsyncHandler");
    private static final Handler sHandler;

    static {
        sHandlerThread.start();
        sHandler = new Handler(sHandlerThread.getLooper());
    }

    public static void post(Runnable r) {
        sHandler.post(r);
    }

    private AsyncHandler() {}
}
```

需要注意的是：

在onReceive函数中执行异步操作，主要目的是避免一些操作阻塞了主线程，
但整个操作仍然需要保证在10s内返回结果，尤其是处理有序广播和静态广播时。
毕竟AMS必须要收到返回结果后，才能向下一个BroadcastReceiver发送广播。

### 2 AMS中的registerReceiver
AMS的registerReceiver函数被调用时，将会保存BroadcastReceiver对应的Binder通信端，我们一起来看看这部分的代码：
```java
public Intent registerReceiver(IApplicationThread caller, String callerPackage,
        IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
    ...............
    //保存系统内已有的sticky广播
    ArrayList<Intent> stickyIntents = null;
    ...............
    synchronized(this) {
        if (caller != null) {
            //根据IApplicationThread得到对应的进程信息
            callerApp = getRecordForAppLocked(caller);

            if (callerApp == null) {
                //抛出异常，即不允许未登记的进程注册BroadcastReceiver
                ...............
            }

            if (callerApp.info.uid != Process.SYSTEM_UID &&
                    !callerApp.pkgList.containsKey(callerPackage) &&
                    !"android".equals(callerPackage)) {
                //抛出异常，即注册进程必须携带Package名称
                ..................
            }
        } else {
            .............
        }
        ......................
        //为了得到IntentFilter中定义的Action信息，先取出其Iterator
        Iterator<String> actions = filter.actionsIterator();
        if (actions == null) {
            ArrayList<String> noAction = new ArrayList<String>(1);
            noAction.add(null);
            actions = noAction.iterator();
        }

        // Collect stickies of users
        int[] userIds = { UserHandle.USER_ALL, UserHandle.getUserId(callingUid) };
        while (actions.hasNext()) {
            //依次比较BroadcastReceiver关注的Action与Stick广播是否一致
            String action = actions.next();
            for (int id : userIds) {
                ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(id);
                if (stickies != null) {

                    //Sticky广播中，有BroadcastReceiver关注的
                    //可能有多个Intent，对应的Action相似，在此先做一个初步筛选
                    ArrayList<Intent> intents = stickies.get(action);
                    if (intents != null) {
                        if (stickyIntents == null) {
                            stickyIntents = new ArrayList<Intent>();
                        }

                        //将这些广播保存到stickyIntents中
                        stickyIntents.addAll(intents);
                    }
                }
            }
        }
    }

    ArrayList<Intent> allSticky = null;

    //stickyIntents中保存的是action匹配的
    if (stickyIntents != null) {
        //用于解析intent的type
        final ContentResolver resolver = mContext.getContentResolver();

        // Look for any matching sticky broadcasts...
        for (int i = 0, N = stickyIntents.size(); i < N; i++) {
            Intent intent = stickyIntents.get(i);

            //此时进一步判断Intent与BroadcastReceiver的IntentFilter是否匹配
            if (filter.match(resolver, intent, true, TAG) >= 0) {
                if (allSticky == null) {
                    allSticky = new ArrayList<Intent>();
                }
                allSticky.add(intent);
            }
        }
    }

    // The first sticky in the list is returned directly back to the client.
    Intent sticky = allSticky != null ? allSticky.get(0) : null;
    ................
    synchronized (this) {
        ..............
        //一个Receiver可能监听多个广播，多个广播对应的BroadcastFilter组成ReceiverList
        ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
        if (rl == null) {
            //首次注册，rl为null，进入此分支
            //新建BroadcastReceiver对应的ReceiverList
            rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                    userId, receiver);
            if (rl.app != null) {
                rl.app.receivers.add(rl);
            } else {
                try {
                    //rl监听receiver所在进程的死亡消息
                    receiver.asBinder().linkToDeath(rl, 0);
                } catch (RemoteException e) {
                    return sticky;
                }
                rl.linkedToDeath = true;
            }
            //保存到AMS上
            mRegisteredReceivers.put(receiver.asBinder(), rl);
        } ..........
        ............

        //创建当前IntentFilter对应的BroadcastFilter
        //AMS收到广播后，将根据BroadcastFilter决定是否将广播递交给对应的BroadcastReceiver
        //一个BroadcastReceiver可以对应多个IntentFilter
        //这些IntentFilter对应的BroadcastFilter共用一个ReceiverList
        BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                permission, callingUid, userId);
        rl.add(bf);
        ................
        mReceiverResolver.addFilter(bf);

        // Enqueue broadcasts for all existing stickies that match
        // this filter.
        //allSticky不为空，说明有粘性广播需要发送给刚注册的BroadcastReceiver
        if (allSticky != null) {
            ArrayList receivers = new ArrayList();
            //receivers记录bf
            receivers.add(bf);

            final int stickyCount = allSticky.size();
            for (int i = 0; i < stickyCount; i++) {
                Intent intent = allSticky.get(i);
                //根据intent的flag （FLAG_RECEIVER_FOREGROUND）决定是使用gBroadcastQueue还是BgBroadcastQueue
                BroadcastQueue queue = broadcastQueueForIntent(intent);

                //创建广播对应的BroadcastRecord
                //BroadcastRecord中包含了Intent，即知道要发送什么广播；
                //同时其中含有receivers，内部包含bf，即知道需要将广播发送给哪个BroadcastReceiver
                BroadcastRecord r = new BroadcastRecord(queue, intent, null,
                        null, -1, -1, null, null, AppOpsManager.OP_NONE, null, receivers,
                        null, 0, null, null, false, true, true, -1);

                //加入到BroadcastQueue中，存入的是并发广播对应的队列
                queue.enqueueParallelBroadcastLocked(r);

                //将发送BROADCAST_INTENT_MSG，触发AMS发送广播
                queue.scheduleBroadcastsLocked();
            }
        }
    }
}
```
至此，AMS中registerReceiver的流程结束。

这个函数看起来很长，但主要工作还是比较清晰的，一共可以分为两个部分：

- 1、将BroadcastReceiver对应的信息保存到AMS中，同时为当前使用的IntentFilter生成对应的BroadcastFilter。
当AMS发送广播时，将根据广播和BroadcastFilter的匹配情况，决定是否为BroadcastReceiver发送消息。

- 2、单独对粘性广播进行处理。
由于粘性广播有立即发送的特点，因此当有新的BroadcastReceiver注册到AMS后，
AMS需要判断系统中保存的粘性广播中，是否有与该BroadcastReceiver匹配的。
若有与新注册BroadcastReceiver匹配的粘性广播，那么AMS为这些广播构造BroadcastRecord，
并将其加入到发送队列中，同时发送消息触发AMS的广播发送流程。

上述代码的整个过程，还是比较好理解的，唯一麻烦一点的就是涉及到了一些数据结构，需要稍微整理一下：

![图2](/images/android-n-ams/broadcast-2.jpg)

在AMS中，BroadcastReceiver的过滤条件由BroadcastFilter表示，如上图所示，该类继承自IntentFilter。

由于一个BroadcastReceiver可以设置多个过滤条件，故AMS使用ReceiverList来记录一个BroadcastReceiver对应的所有BroadcastFilter。
同时，BroadcastFilter中持有对ReceiverList的引用，用于记录自己属于哪个ReceiverList；
ReceiverList中也保存着对IIntentReceiver的引用，用于记录自己对应于哪个BroadcastReceiver。

AMS中利用mRegisterdReceivers这个HashMap，来保存广播对应的ReceiverList，其中的键值就是BroadcastReceiver对应的IIntentReceiver。
同时，AMS中的mReceiverResolver用于保存所有动态注册BroadcastReceiver对应的BroadcastFilter。注意此处的IntentResolver是一个模板类，并不是一个Map类型的数据结构。

这部分流程比较简单，大致如下图所示：

![图3](/images/android-n-ams/broadcast-3.jpg)

## 三、sendBroadcast流程分析
分析完广播接收方注册BroadcastReceiver的流程，现在我们来看看广播发送方sendBroadcast的流程。

与registerReceiver一样，Context.java中定义了多个sendBroadcast的接口，但是殊途同归，这些接口最终的流程基本一致。
因此，我们以比较常见的接口入手，看看整个代码的逻辑。
```java
public void sendBroadcast(Intent intent) {
    ............
    String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
    try {
        //StrictMode下，对一些Action需要进行检查
        intent.prepareToLeaveProcess(this);

        //调用AMS中的接口
        ActivityManagerNative.getDefault().broadcastIntent(
                mMainThread.getApplicationThread(), intent, resolvedType, null,
                Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                getUserId());
    } catch(RemoteException e) {
        ..............
    }
}
```
ContextImpl中的sendBroadcast函数比较简单，进行一些必要的检查后，直接调用AMS中接口。
我们跟进流程，看看AMS中的broadcastIntent函数：
```java
public final int broadcastIntent(IApplicationThread caller,
        Intent intent, String resolvedType, IIntentReceiver resultTo,
        int resultCode, String resultData, Bundle resultExtras,
        String[] requiredPermissions, int appOp, Bundle bOptions,
        boolean serialized, boolean sticky, int userId) {
    ..............
    synchronized(this) {
        //检查Broadcast中Intent携带的信息是否有问题
        //例如：Intent中不能携带文件描述符（避免安全隐患）
        //同时检查Intent的Flag
        intent = verifyBroadcastLocked(intent);

        final ProcessRecord callerApp = getRecordForAppLocked(caller);
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();
        final long origId = Binder.clearCallingIdentity();

        int res = broadcastIntentLocked(callerApp,
                callerApp != null ? callerApp.info.packageName : null,
                intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                requiredPermissions, appOp, bOptions, serialized, sticky,
                callingPid, callingUid, userId);
        Binder.restoreCallingIdentity(origId);
        return res;
    }
}
```

broadcastIntent中对Broadcast对应的Intent进行一些检查后，调用broadcastIntentLocked进行实际的处理。
broadcastIntentLocked函数非常长，大概600行左右吧…….我们只能分段看看它的主要思路。

### 1 broadcastIntentLocked函数Part I
```java
final int broadcastIntentLocked(.....) {
    intent = new Intent(intent);

    // By default broadcasts do not go to stopped apps.
    // Android系统对所有app的运行状态进行了跟踪
    // 当应用第一次安装但未使用过，或从程序管理器被强行关闭后，将处于停止状态

    // 在发送广播时，不管是什么广播类型，系统默认增加了FLAG_EXCLUDE_STOPPED_PACKAGES的flag
    // 使得对于所有BroadcastReceiver而言，如果其所在进程的处于停止状态，该BroadcastReceiver将无法接收到广播
    intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);

    //大量条件检查，例如：
    //有些Action只能由系统来发送；
    //有些Action需要发送方申明了对应权限
    ...................

    //某些Action，例如Package增加或移除、时区或时间改变、清除DNS、代理变化等，需要AMS来处理
    //AMS需要根据Action，进行对应的操作
    ..................
}
```

这部分的代码较多，主要是针对具体Action的操作，在分析整体流程时，没有必要深究。
当需要研究对应广播引发的操作时，可以再有针对性的研究。

简单地讲，代码主要工作其实只有两个：
- 1、根据广播对应Intent中的信息，判断发送方是否有发送该广播的权限；
- 2、针对一些特殊的广播，AMS需要进行一些操作。

### 2 broadcastIntentLocked函数Part II
```java
..............
// Add to the sticky list if requested.
// 处理粘性广播相关的内容
if (sticky) {
    //检查是否有发送权限
    if (checkPermission(android.Manifest.permission.BROADCAST_STICKY,
            callingPid, callingUid)
            != PackageManager.PERMISSION_GRANTED) {
        ..................
    }

    //粘性广播不能指定接收权限
    if (requiredPermissions != null && requiredPermissions.length > 0) {
        ..............
        return ActivityManager.BROADCAST_STICKY_CANT_HAVE_PERMISSION;
    }

    if (intent.getComponent() != null) {
        //粘性广播不能指定接收方
        ............
    }

    // We use userId directly here, since the "all" target is maintained
    // as a separate set of sticky broadcasts.
    //当粘性广播是针对特定userId时，判断该粘性广播与系统保存的是否冲突
    if (userId != UserHandle.USER_ALL) {

        //取出发送给所有user的粘性广播
        ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(
                UserHandle.USER_ALL);
        if (stickies != null) {
            ArrayList<Intent> list = stickies.get(intent.getAction());
            if (list != null) {
                int N = list.size();
                int i;
                for (i=0; i<N; i++) {
                    //发送给特定user的粘性广播，与发送给所有user的粘性广播，action一致时，发生冲突
                    if (intent.filterEquals(list.get(i))) {
                        //抛出异常
                        .............
                    }
                }
            }
        }

        ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(userId);
        if (stickies == null) {
            stickies = new ArrayMap<>();
            //按userId保存粘性广播
            //即一个user，可能有多个粘性广播
            mStickyBroadcasts.put(userId, stickies);
        }

        //按照Action保存粘性广播
        //即一个Action，可能对应中多个广播
        ArrayList<Intent> list = stickies.get(intent.getAction());
        if (list == null) {
            //list为null时，直接加入
            list = new ArrayList<>();
            stickies.put(intent.getAction(), list);
        }
        final int stickiesCount = list.size();
        int i;
        for (i = 0; i < stickiesCount; i++) {
            //新的粘性广播与之前的重复，则保留新的
            //即当发送多个相同的粘性广播时，系统仅会保留最新的
            if (intent.filterEquals(list.get(i))) {
                // This sticky already exists, replace it.
                list.set(i, new Intent(intent));
                break;
            }
        }
        if (i >= stickiesCount) {
            //不重复时，直接加入
            list.add(new Intent(intent));
        }
    }
}
...............
```
broadcastIntentLocke第二部分主要是处理粘性广播的，整个功能比较清晰，
即判断发送粘性广播的条件是否满足，然后将粘性广播保存起来。

### 3 broadcastIntentLocked函数Part III
```java
.............
// Figure out who all will receive this broadcast.
//receivers主要用于保存匹配当前广播的静态注册的BroadcastReceiver
//若当前广播是有序广播时，还会插入动态注册的BroadcastReceiver
List receivers = null;

//registeredReceivers用于保存匹配当前广播的动态注册的BroadcastReceiver
//BroadcastFilter中有对应的BroadcastReceiver的引用
List<BroadcastFilter> registeredReceivers = null;

// Need to resolve the intent to interested receivers...
// 若设置了FLAG_RECEIVER_REGISTERED_ONLY，那么只有此时完成了注册的BroadcastReceiver才会收到信息
// 简单讲就是，有FLAG_RECEIVER_REGISTERED_ONLY时，不通知静态注册的BroadcastReceiver

// 此处处理未设置该标记的场景
if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
    //利用PKMS的queryIntentReceivers接口，查询满足条件的静态BroadcastReceiver
    receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
}

//广播没有指定特定接收者时
if (intent.getComponent() == null) {
    //这里的要求比较特殊，针对所有user，且从shell发送的广播
    //即处理调试用的广播
    if (userId == UserHandle.USER_ALL && callingUid == Process.SHELL_UID) {
        // Query one target user at a time, excluding shell-restricted users
        for (int i = 0; i < users.length; i++) {
            //user不允许调试时，跳过
            if (mUserController.hasUserRestriction(
                    UserManager.DISALLOW_DEBUGGING_FEATURES, users[i])) {
                continue;
            }

            //得到当前user对应的满足Intent要求的BroadcastReceiver
            //mReceiverResolver中保存的都是动态注册的BroadcastReceiver对应的BroadcastFilter
            List<BroadcastFilter> registeredReceiversForUser =
                    mReceiverResolver.queryIntent(intent,
                            resolvedType, false, users[i]);

            if (registeredReceivers == null) {
                registeredReceivers = registeredReceiversForUser;
            } else if (registeredReceiversForUser != null) {
                registeredReceivers.addAll(registeredReceiversForUser);
            }
        }
    } else {
        //通常的处理流程
        registeredReceivers = mReceiverResolver.queryIntent(intent,
                resolvedType, false, userId);
    }
}

//检查广播中是否有REPLACE_PENDING标签
//如果设置了这个标签，那么新的广播可以替换掉AMS广播队列中，与之匹配的且还未被处理的旧有广播
//这么做的目的是：尽可能的减少重复广播的发送
final boolean replacePending =
        (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;
.....................

//先处理动态注册的BroadcastReceiver对应的广播
int NR = registeredReceivers != null ? registeredReceivers.size() : 0;

//处理无序的广播，即普通广播
if (!ordered && NR > 0) {
    // If we are not serializing this broadcast, then send the
    // registered receivers separately so they don't wait for the
    // components to be launched.
    //根据Intent的Flag决定BroadcastQueue
    final BroadcastQueue queue = broadcastQueueForIntent(intent);

    //构造广播对应的BroadcastRecord
    //BroadcastRecord中包含了该广播的所有接收者
    BroadcastRecord r = new BroadcastRecord(........);
    ................
    //设置了REPLACE_PENDING标签，同时与旧有广播匹配时，才会进行替换
    //若能够替换，replaceParallelBroadcastLocked中就会将新的广播替换到队列中
    final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);

    if (!replaced) {
        //没有替换时，才需要将新的广播加入到队列中
        queue.enqueueParallelBroadcastLocked(r);

        //触发广播发送流程
        queue.scheduleBroadcastsLocked();
    }
    registeredReceivers = null;
    NR = 0;
}
.....................
```

broadcastIntentLocke第三部分的工作主要包括：
- 1、查询与当前广播匹配的静态和动态BroadcastReceiver；
- 2、若当前待发送的广播是无序的，那么为动态注册的BroadcastReceiver，构造该广播对应的BroadcastRecord加入到发送队列中，
并触发广播发送流程。

从这部分代码可以看出，对于无序广播而言，动态注册的BroadcastReceiver接收广播的优先级，高于静态注册的BroadcastReceiver。

### 4 broadcastIntentLocked函数Part IV
```java
.................
// Merge into one list.
int ir = 0;
if (receivers != null) {
    // A special case for PACKAGE_ADDED: do not allow the package
    // being added to see this broadcast.  This prevents them from
    // using this as a back door to get run as soon as they are
    // installed.  Maybe in the future we want to have a special install
    // broadcast or such for apps, but we'd like to deliberately make
    // this decision.
    //处理特殊的Action，例如PACKAGE_ADDED，系统不希望有些应用一安装就能启动
    //APP安装后，PKMS将发送PACKAGE_ADDED广播
    //若没有这个限制，在刚安装的APP内部静态注册监听该消息的BroadcastReceiver，新安装的APP就能直接启动
    String skipPackages[] = null;
    if (Intent.ACTION_PACKAGE_ADDED.equals(intent.getAction())
            || Intent.ACTION_PACKAGE_RESTARTED.equals(intent.getAction())
            || Intent.ACTION_PACKAGE_DATA_CLEARED.equals(intent.getAction())) {
        Uri data = intent.getData();
        if (data != null) {
            String pkgName = data.getSchemeSpecificPart();
            if (pkgName != null) {
                skipPackages = new String[] { pkgName };
            }
        }
    } else if (Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE.equals(intent.getAction())) {
        skipPackages = intent.getStringArrayExtra(Intent.EXTRA_CHANGED_PACKAGE_LIST);
    }

    if (skipPackages != null && (skipPackages.length > 0)) {
        for (String skipPackage : skipPackages) {
            if (skipPackage != null) {
                int NT = receivers.size();
                for (int it=0; it<NT; it++) {
                    ResolveInfo curt = (ResolveInfo)receivers.get(it);
                    if (curt.activityInfo.packageName.equals(skipPackage)) {
                        //将skipPackages对应的BroadcastReceiver移出receivers
                        receivers.remove(it);
                        it--;
                        NT--;
                     }
                }
            }
        }
    }

    int NT = receivers != null ? receivers.size() : 0;
    int it = 0;
    ResolveInfo curt = null;
    BroadcastFilter curr = null;
    //NT对应的是静态BroadcastReceiver的数量
    //NR对应的是动态BroadcastReceiver的数量
    //若发送的是无序广播，此时NR为0
    //若是有序广播，才会进入下面两个while循环

    //下面两个while循环就是将静态注册的BroadcastReceiver和动态注册的BroadcastReceiver
    //按照优先级合并到一起（有序广播才会合并）
    while (it < NT && ir < NR) {
        if (curt == null) {
            curt = (ResolveInfo)receivers.get(it);
        }
        if (curr == null) {
            curr = registeredReceivers.get(ir);
        }

        //动态优先级大于静态时，将动态插入到receivers中
        if (curr.getPriority() >= curt.priority) {
            // Insert this broadcast record into the final list.
            receivers.add(it, curr);
            ir++;
            curr = null;
            it++;
            NT++;
        } else {
            // Skip to the next ResolveInfo in the final list.
            it++;
            curt = null;
        }
    }
}

while (ir < NR) {
    if (receivers == null) {
        receivers = new ArrayList();
    }
    //插入剩下的动态BroadcastReceiver
    receivers.add(registeredReceivers.get(ir));
    ir++;
}

if ((receivers != null && receivers.size() > 0)
        || resultTo != null) {
    BroadcastQueue queue = broadcastQueueForIntent(intent);
    //构造对应的BroadcastRecord
    BroadcastRecord r = new BroadcastRecord(........);
    ............
    //同样判断是否需要替换，这里是Ordered Queue
    boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);
    if (!replaced) {
        //没替换时，就加入Ordered Queue
        queue.enqueueOrderedBroadcastLocked(r);

        //触发发送流程
        queue.scheduleBroadcastsLocked();
    }
} else {
    // There was nobody interested in the broadcast, but we still want to record
    // that it happened.
    if (intent.getComponent() == null && intent.getPackage() == null
            && (intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
        // This was an implicit broadcast... let's record it for posterity.
        //没有处理的静态或有序广播，保存起来
        //感觉保存起来也没什么用啊
        addBroadcastStatLocked(intent.getAction(), callerPackage, 0, 0, 0);
    }
}
...............
```

简单来说，broadcastIntentLocked的第四部分工作就是为有序广播和静态广播，构造对应的BroadcastRecord（按优先级顺序），
然后将BroadcastRecord加入到Ordered Queue中，并触发广播发送流程。

至此，整个broadcastIntentLocked函数分析完毕，除去一些条件判断的细节外，整个工作流程如下图所示：

![图4](/images/android-n-ams/broadcast-4.jpg)

- 1、处理粘性广播。

由于粘性广播的特性（BroadcastReceiver注册即可接收），系统必须首先保存粘性广播。

- 2、处理普通动态广播。

普通广播是并发的，系统优先为动态注册的BroadcastReceiver发送广播。
动态广播对应的BroadcastRecord加入到Parallel Queue中。

- 3、处理静态广播和有序广播。

这一步主要是为静态注册的BroadcastReceiver发送广播，对应的BroadcastRecord加入到Ordered Queue中。

此外，需要注意的是：

如果广播是有序的，那么第2步不会为动态注册的BroadcastReceiver发送广播，而是在第3步统一发送。
发送有序广播时，AMS将按照BroadcastReceiver的优先级，依次构造BroadcastRecord加入到Ordered Queue中。

## 四、BroadcastQueue的scheduleBroadcastsLocked函数流程分析

从上面的代码，我们知道广播发送方调用sendBroadcast后，AMS会构造对应的BroadcastRecord加入到BroadcastQueue中，
然后调用BroadcastQueue的scheduleBroadcastsLocked函数。

现在，我们就来看一下scheduleBroadcastsLocked相关的流程。
```java
public void scheduleBroadcastsLocked() {
    ...........
    //避免短时间内重复发送BROADCAST_INTENT_MSG
    if (mBroadcastsScheduled) {
        return;
    }
    mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
    mBroadcastsScheduled = true;
}
```

上面的代码将发送BROADCAST_INTENT_MSG，触发BroadcastQueue调用processNextBroadcast进行处理。
processNextBroadcast函数同样非常的长，大概有500多行。。。。我们还是分段进行分析。

### 1 processNextBroadcast函数Part I
```java
final void processNextBroadcast(boolean fromMsg) {
    synchronized(mService) {
        BroadcastRecord r;
        ..............
        //更新CPU的使用情况
        //处理静态广播时，可能需要拉起对应进程，因此在这里先记录一下CPU情况
        mService.updateCpuStats();

        if (fromMsg) {
            //处理BROADCAST_INTENT_MSG后，将mBroadcastsScheduled置为false
            //scheduleBroadcastsLocked就可以再次被调用了
            mBroadcastsScheduled = false;
        }

        // First, deliver any non-serialized broadcasts right away.
        //先处理“并发”发送的普通广播
        while (mParallelBroadcasts.size() > 0) {
            //依次取出BroadcastRecord
            r = mParallelBroadcasts.remove(0);

            //记录发送的时间
            r.dispatchTime = SystemClock.uptimeMillis();
            r.dispatchClockTime = System.currentTimeMillis();

            final int N = r.receivers.size();
            ................
            for (int i=0; i<N; i++) {
                //mParallelBroadcasts中的每个成员均为BroadcastFilter类型
                Object target = r.receivers.get(i);
                ............
                //为该BroadcastRecord对应的每个Receiver发送广播
                deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false, i);
            }

            //将这里处理过的信息加入到历史记录中
            addBroadcastToHistoryLocked(r);
        }
        ........................
    }
}
```

从上面的代码可以看出，processNextBroadcast函数的第一部分主要是为动态注册的BroadcastReceiver发送普通广播。
发送普通广播的函数为deliverToRegisteredReceiverLocked：
```java
private void deliverToRegisteredReceiverLocked(BroadcastRecord r,
        BroadcastFilter filter, boolean ordered, int index) {
    boolean skip = false;
    //检查广播发送方是否有BroadcastReceiver指定的权限
    if (filter.requiredPermission != null) {
        int perm = mService.checkComponentPermission(filter.requiredPermission,
                r.callingPid, r.callingUid, -1, true);
        if (perm != PackageManager.PERMISSION_GRANTED) {
            ............
            skip = true;
        } else {
            //进一步检查权限的合理性
            final int opCode = AppOpsManager.permissionToOpCode(filter.requiredPermission);
            if (opCode != AppOpsManager.OP_NONE
                    && mService.mAppOpsService.noteOperation(opCode, r.callingUid,
                            r.callerPackage) != AppOpsManager.MODE_ALLOWED) {
                ............
                skip = true;
            }
        }
    }

    //检查BroadcastReceiver是否有Broadcast要求的权限
    if (!skip && r.requiredPermissions != null && r.requiredPermissions.length > 0) {
        for (int i = 0; i < r.requiredPermissions.length; i++) {
            String requiredPermission = r.requiredPermissions[i];
            int perm = mService.checkComponentPermission(requiredPermission,
                    filter.receiverList.pid, filter.receiverList.uid, -1, true);
            if (perm != PackageManager.PERMISSION_GRANTED) {
                .................
                //只要一条权限不满足，就结束
                skip = true;
                break;
            }

            //进一步检查权限的合理性
            int appOp = AppOpsManager.permissionToOpCode(requiredPermission);
            if (appOp != AppOpsManager.OP_NONE && appOp != r.appOp
                    && mService.mAppOpsService.noteOperation(appOp,
                            filter.receiverList.uid, filter.packageName)
                    != AppOpsManager.MODE_ALLOWED) {
                ...........
                skip = true;
                break;
            }
        }
    }

    //这段代码我看的有些懵逼，发送方没要求权限，还检查啥？
    if (!skip && (r.requiredPermissions == null || r.requiredPermissions.length == 0)) {
        int perm = mService.checkComponentPermission(null,
                filter.receiverList.pid, filter.receiverList.uid, -1, true);
        if (perm != PackageManager.PERMISSION_GRANTED) {
            ............
            skip = true;
        }
    }

    //还有一些其它的检查，不再分析代码了。。。。。
    //包括是否允许以background的方式发送、IntentFirewall是否允许广播中的Intent发送
    ...............................

    if (skip) {
        //不满足发送条件的话，标记一下，结束发送
        r.delivery[index] = BroadcastRecord.DELIVERY_SKIPPED;
        return;
    }

    // If permissions need a review before any of the app components can run, we drop
    // the broadcast and if the calling app is in the foreground and the broadcast is
    // explicit we launch the review UI passing it a pending intent to send the skipped
    // broadcast.
    //特殊情况，还需要再次检查权限，中断广播发送
    //再次满足发送条件后，会重新进入到后续的发送流程
    if (Build.PERMISSIONS_REVIEW_REQUIRED) {
        if (!requestStartTargetPermissionsReviewIfNeededLocked(r, filter.packageName,
                filter.owningUserId)) {
            r.delivery[index] = BroadcastRecord.DELIVERY_SKIPPED;
            return;
        }
    }

    //可以发送了，标记一下
    r.delivery[index] = BroadcastRecord.DELIVERY_DELIVERED;

    // If this is not being sent as an ordered broadcast, then we
    // don't want to touch the fields that keep track of the current
    // state of ordered broadcasts.
    if (ordered) {
        //如果发送的是有序广播，则记录一些状态信息等，不涉及广播发送的过程
        .............
    }

    try {
        ..............
        //若BroadcastReceiver对应的进程处于fullBackup状态（备份和恢复），则不发送广播
        if (filter.receiverList.app != null && filter.receiverList.app.inFullBackup) {
            if (ordered) {
                //有序广播必须处理完一个，才能处理下一个，因此这里主动触发一下
                skipReceiverLocked(r);
            }
        } else {
            //执行发送工作
            performReceiveLocked(.............);
        }
        if (ordered) {
            r.state = BroadcastRecord.CALL_DONE_RECEIVE;
        }
    } catch (RemoteException e) {
        .............
    }
}
```

deliverToRegisteredReceiverLocked看起来很长，但大部分内容都是进行权限检查等操作，实际的发送工作交给了performReceiveLocked函数。
广播是一种可以携带数据的跨进程、跨APP通信机制，为了避免安全泄露的问题，Android才在此处使用了大量的篇幅进行权限检查的工作。

在这一部分的最后，我们跟进一下performReceiveLocked函数：
```java
void performReceiveLocked(.........) {
    // Send the intent to the receiver asynchronously using one-way binder calls.
    if (app != null) {
        if (app.thread != null) {
            // If we have an app thread, do the call through that so it is
            // correctly ordered with other one-way calls.
            try {
                //通过Binder通信，将广播信息发往BroadcastReceiver处在的进程
                app.thread.scheduleRegisteredReceiver(.......);
            } catch (RemoteException ex) {
                // Failed to call into the process. It's either dying or wedged. Kill it gently.
                .......
                app.scheduleCrash("can't deliver broadcast");
            }
            throw ex;
        } else {
            // Application has died. Receiver doesn't exist.
            throw new RemoteException("app.thread must not be null");
        }
    } else {
        //直接通过Binder通信，将消息发往BroadcastReceiver的IIntentReceiver接口
        receiver.performReceive(intent, resultCode, data, extras, ordered,
                sticky, sendingUser);
    }
}
```

对于动态注册的BroadcastReceiver而言，发送广播的流程应该进入到上述代码的If分支。
进程的ApplicationThread调用scheduleRegisteredReceiver函数的流程，我们放在后面单独分析。

这一部分整体的流程如下图所示：

![图5](/images/android-n-ams/broadcast-5.jpg)

### 2 processNextBroadcast函数Part II
至此，processNextBroadcast函数已经发送完所有的动态普通广播，我们看看该函数后续的流程：
```java
..........
// Now take care of the next serialized one...

// If we are waiting for a process to come up to handle the next
// broadcast, then do nothing at this point.  Just in case, we
// check that the process we're waiting for still exists.
// 对于有序或静态广播而言，需要依次向每个BroadcastReceiver发送广播，前一个处理完毕后才能发送下一个广播
// 如果BroadcastReceiver对应的进程还未启动，则需要等待
// mPendingBroadcast就是用于保存，因为对应进程还未启动，而处于等待状态的BroadcastRecord
if (mPendingBroadcast != null) {
    ................
    boolean isDead;
    synchronized (mService.mPidsSelfLocked) {
        //通过AMS得到对应进程的信息
        //BroadRecord对应多个BroadcastReceiver，即对应多个进程
        //此处curApp保存当前正在等待的进程
        ProcessRecord proc = mService.mPidsSelfLocked.get(mPendingBroadcast.curApp.pid);
        isDead = proc == null || proc.crashing;
    }
    if (!isDead) {
        // It's still alive, so keep waiting
        // 注意到有序和静态广播必须依次处理
        // 因此，若前一个BroadcastRecord对应的某个进程启动较慢，不仅会影响该BroadcastRecord中后续进程接收广播
        // 还会影响到后续所有BroadcastRecord对应进程接收广播
        return;
    } else {
        ............
        mPendingBroadcast.state = BroadcastRecord.IDLE;
        mPendingBroadcast.nextReceiver = mPendingBroadcastRecvIndex;
        mPendingBroadcast = null;
    }
}

boolean looped = false;

//处理之前发送BroadcastRecord的状态
do {
    if (mOrderedBroadcasts.size() == 0) {
        // No more broadcasts pending, so all done!
        mService.scheduleAppGcsLocked();

        //当至少一个BroadcastRecord被处理完毕后，looped才会被置变为true
        if (looped) {
            // If we had finished the last ordered broadcast, then
            // make sure all processes have correct oom and sched
            // adjustments.
            // 因为静态广播和有序广播，可能拉起进程
            // 因此这些广播处理完毕后，AMS需要释放掉一些不需要的进程
            mService.updateOomAdjLocked();
        }
        return;
    }

    //每次均处理当前的第一个
    r = mOrderedBroadcasts.get(0);
    boolean forceReceive = false;

    // Ensure that even if something goes awry with the timeout
    // detection, we catch "hung" broadcasts here, discard them,
    // and continue to make progress.
    //
    // This is only done if the system is ready so that PRE_BOOT_COMPLETED
    // receivers don't get executed with timeouts. They're intended for
    // one time heavy lifting after system upgrades and can take
    // significant amounts of time.
    // 这里用于判断此广播是否处理超时
    // 仅在系统启动完毕后，才进行该判断，因为PRE_BOOT_COMPLETED广播可能由于系统升级需要等待较长时间
    int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
    if (mService.mProcessesReady && r.dispatchTime > 0) {
        long now = SystemClock.uptimeMillis();
        if ((numReceivers > 0) &&
                //单个广播处理的超时时间，定义为2 * 每个接收者处理的最长时间（10s）* 接收者的数量
                (now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers))) {
            ...............
            //如果超时，则强制结束这条广播的处理
            broadcastTimeoutLocked(false); // forcibly finish this broadcast
            forceReceive = true;
            r.state = BroadcastRecord.IDLE;
        }
    }

    if (r.state != BroadcastRecord.IDLE) {
        //BroadcastRecord还未处理或处理完毕后，状态为IDLE态
        ............
        return;
    }

    //如果广播处理完毕，或中途被取消
    if (r.receivers == null || r.nextReceiver >= numReceivers
            || r.resultAbort || forceReceive) {
        // No more receivers for this broadcast!  Send the final
        // result if requested...
        if (r.resultTo != null) {
            try {
                ..........
                //广播处理完毕，将该广播的处理结果发送给resultTo对应BroadcastReceiver
                performReceiveLocked(r.callerApp, r.resultTo,
                        new Intent(r.intent), r.resultCode,
                        r.resultData, r.resultExtras, false, false, r.userId);
            } catch (RemoteException e) {
                .............
            }
        }
        .................
        cancelBroadcastTimeoutLocked();
        addBroadcastToHistoryLocked(r);

        //作一下记录
        if (r.intent.getComponent() == null && r.intent.getPackage() == null
                 && (r.intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
            // This was an implicit broadcast... let's record it for posterity.
            mService.addBroadcastStatLocked(r.intent.getAction(), r.callerPackage,
                    r.manifestCount, r.manifestSkipCount, r.finishTime-r.dispatchTime);
        }
        mOrderedBroadcasts.remove(0);
        //一个BroadcastRecord处理完毕后，将r置为null
        r = null;

        //如果一个广播处理完毕，说明可能拉起过进程，于是looped置为true
        looped = true;
        continue;
    }
//r == null时，可以处理下一个BroadcastRecord
//r != null, 继续处理当前BroadcastRecord的下一个BroadReceiver
} while (r == null);
....................
```

这一部分代码，乍一看有点摸不着头脑的感觉，其实主要做了两个工作：
#### 1、判断是否有PendingBroadcast。

当存在PendingBroadcast，且当前正在等待启动的进程并没有死亡，那么不能处理下一个BroadcastRecord，必须等待PendingBroadcast处理完毕。

#### 2、处理mOrderedBroadcasts中的BroadcastRecord

由于有序广播和静态广播，必须一个接一个的处理。
因此每发送完一个广播后，均会重新调用processNextBroadcast函数。

在发送新的BroadcastRecord时，需要先处理旧有BroadcastRecord的状态。

于是，这段代码后续部分，主要进行了以下操作：
- 若所有BroadcastRecord均处理完毕，利用AMS释放掉无用进程；
- 更新超时BroadcastRecord的状态，同时越过此BroadcastRecord；
- 当一个BroadcastRecord处理完毕后，将结果发送给指定BroadcastReceiver（指定了接收者，才进行此操作），
同时将该BroadcastRecord从mOrderedBroadcasts中移除。

这一系列动作的最终目的，就是选出下一个处理的BroadcastRecord，
然后就可以开始向该BroadcastRecord中下一个BroadcastReceiver发送广播了。

这一部分整体的判断逻辑大致上如下图所示：

大图链接
![图6](/images/android-n-ams/broadcast-6.jpg)

### 3 processNextBroadcast函数Part III
```java
..............
// Get the next receiver...
// 开始处理当前BroadcastRecord的下一个BroadcastReceiver
int recIdx = r.nextReceiver++;

// Keep track of when this receiver started, and make sure there
// is a timeout message pending to kill it if need be.
// 记录单个广播的起始时间
r.receiverTime = SystemClock.uptimeMillis();
if (recIdx == 0) {
    //记录整个BroadcastRecord的起始时间
    r.dispatchTime = r.receiverTime;
    r.dispatchClockTime = System.currentTimeMillis();
    ................
}

if (! mPendingBroadcastTimeoutMessage) {
    long timeoutTime = r.receiverTime + mTimeoutPeriod;
    ....................
    //设置广播处理的超时时间为10s
    setBroadcastTimeoutLocked(timeoutTime);
}

final BroadcastOptions brOptions = r.options;
//取出下一个广播接收者
final Object nextReceiver = r.receivers.get(recIdx);

if (nextReceiver instanceof BroadcastFilter) {
    // Simple case: this is a registered receiver who gets
    // a direct call.
    //动态注册的BroadcastReceiver，即处理的是有序广播
    BroadcastFilter filter = (BroadcastFilter)nextReceiver;
    ............
    //与处理普通广播一样，调用deliverToRegisteredReceiverLocked
    //将广播发送给BroadcastReceiver对应进程的ApplicationThread
    deliverToRegisteredReceiverLocked(r, filter, r.ordered, recIdx);

    //从代码流程来看，deliverToRegisteredReceiverLocked发送广播出现问题时，r.receiver才可能是null
    //代码运行到这个位置，加入到Ordered Queue的Broadcast一定是order的
    if (r.receiver == null || !r.ordered) {
        // The receiver has already finished, so schedule to
        // process the next one.
        ...............
        //因此这里应该是有序广播发送错误，因此重新发送BROADCAST_INTENT_MSG，触发下一次发送广播的流程
        r.state = BroadcastRecord.IDLE;
        scheduleBroadcastsLocked();
    } else {
        //处理特殊的选项
        //向DeviceIdleController发送消息，赋予白名单内的应用，
        //在Doze模式的激活窗口中，额外的可以访问网络和持锁的时间
        if (brOptions != null && brOptions.getTemporaryAppWhitelistDuration() > 0) {
            scheduleTempWhitelistLocked(filter.owningUid,
                    brOptions.getTemporaryAppWhitelistDuration(), r);
        }
    }

    //对于有序广播而言，已经通知了一个BroadcastReceiver，需要等待处理结果，因此返回
    return;
}
.......................
```

processNextBroadcast的第3部分，主要是处理有序广播发往动态BroadcastReceiver的场景。
从代码可以看出，有序广播具体的方法流程与普通广播一致，均是调用deliverToRegisteredReceiverLocked函数。
唯一不同的是，有序广播发往一个BroadcastReceiver后，必须等待处理结果，才能进行下一次发送过程。

### 4 processNextBroadcast函数 Part IV
```java
...................
// Hard case: need to instantiate the receiver, possibly
// starting its application process to host it.
// 开始处理静态广播

ResolveInfo info = (ResolveInfo)nextReceiver;

//得到静态广播对应的组件名
ComponentName component = new ComponentName(
        info.activityInfo.applicationInfo.packageName,
        info.activityInfo.name);

boolean skip = false;

//以下与deliverToRegisteredReceiverLocked中类似，进行发送广播前的检查工作

//判断发送方和接收方要求的权限，是否互相满足
//判断Intent是否满足AMS的IntentFirewall要求
...................

boolean isSingleton = false;
try {
    //判断BroadcastReceiver是否是单例的
    isSingleton = mService.isSingleton(info.activityInfo.processName,
            info.activityInfo.applicationInfo,
            info.activityInfo.name, info.activityInfo.flags);
} catch (SecurityException e) {
    .................
}

//BroadcastReceiver要求SINGLE_USER
//那么必须申明INTERACT_ACROSS_USERS权限
if ((info.activityInfo.flags&ActivityInfo.FLAG_SINGLE_USER) != 0) {
    if (ActivityManager.checkUidPermission(
            android.Manifest.permission.INTERACT_ACROSS_USERS,
            info.activityInfo.applicationInfo.uid)
                    != PackageManager.PERMISSION_GRANTED) {
        ..............
        skip = true;
    }
}

if (!skip) {
    r.manifestCount++;
} else {
    r.manifestSkipCount++;
}

if (r.curApp != null && r.curApp.crashing) {
    .........
    skip = true;
}

if (!skip) {
    boolean isAvailable = false;
    try {
        isAvailable = AppGlobals.getPackageManager().isPackageAvailable(
                info.activityInfo.packageName,
                UserHandle.getUserId(info.activityInfo.applicationInfo.uid));
    } catch (Exception e) {
        ..........
    }
    if (!isAvailable) {
        .........
        skip = true;
    }
}

//判断BroadcastReceiver对应进程是否允许后台启动
//不允许也会skip
..............

if (skip) {
    ..................
    r.delivery[recIdx] = BroadcastRecord.DELIVERY_SKIPPED;
    r.receiver = null;
    r.curFilter = null;
    r.state = BroadcastRecord.IDLE;
    //跳过该广播，发送下一个广播
    scheduleBroadcastsLocked();
    return;
}

r.delivery[recIdx] = BroadcastRecord.DELIVERY_DELIVERED;
r.state = BroadcastRecord.APP_RECEIVE;
r.curComponent = component;
r.curReceiver = info.activityInfo;
................
if (brOptions != null && brOptions.getTemporaryAppWhitelistDuration() > 0) {
    //与处理有序普通广播一样，在此处理特殊的选项
    scheduleTempWhitelistLocked(receiverUid,
            brOptions.getTemporaryAppWhitelistDuration(), r);
}

// Broadcast is being executed, its package can't be stopped.
try {
    //发送静态广播前，修改BroadcastReceiver对应的Package状态
    AppGlobals.getPackageManager().setPackageStoppedState(
            r.curComponent.getPackageName(), false, UserHandle.getUserId(r.callingUid));
} catch (RemoteException e) {
} catch (IllegalArgumentException e) {
    ...............
}

// Is this receiver's application already running?
if (app != null && app.thread != null) {
    try {
        app.addPackage(info.activityInfo.packageName,
                info.activityInfo.applicationInfo.versionCode, mService.mProcessStats);
        //BroadcastReceiver对应进程启动时，调用ApplicationThread的scheduleReceiver
        processCurBroadcastLocked(r, app);

        //等待结果，故return
        return;
    } catch (RemoteException e) {
        //这可能是对应进程死亡，可以重新拉起进程发送
        ..........
    } catch (RuntimeException e) {
        .........
        //发送失败，结束本次发送
        finishReceiverLocked(r, r.resultCode, r.resultData,
        r.resultExtras, r.resultAbort, false);

        //继续发送后续的广播
        scheduleBroadcastsLocked();
        // We need to reset the state if we failed to start the receiver.
        r.state = BroadcastRecord.IDLE;
        return;
    }
}
.............
//启动进程处理广播
if ((r.curApp=mService.startProcessLocked(targetProcess,
        info.activityInfo.applicationInfo, true,
        r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
        "broadcast", r.curComponent,
        (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
                == null) {
    //创建进程失败
    ...............
    //发送失败，结束本次发送
    finishReceiverLocked(r, r.resultCode, r.resultData,
    r.resultExtras, r.resultAbort, false);

    //继续发送后续的广播
    scheduleBroadcastsLocked();
    r.state = BroadcastRecord.IDLE;
    return;
}

//进程启动成功时，mPendingBroadcast保存当前的BroadcastRecord，及待发送广播的下标
//当进程启动后，将根据这两个值处理广播
mPendingBroadcast = r;
mPendingBroadcastRecvIndex = recIdx;
.............
```

processNextBroadcast第四部分主要是处理静态广播，除了检查是否满足发送条件外，主要进行了以下工作：
- 1、若BroadcastReceiver对应的进程已经启动，那么将直接调用进程对应ApplicationThread的scheduleReceiver发送广播；
- 2、若BroadcastReceiver对应的进程没有启动，那么AMS将启动对应的进程。

当对应的进程启动，同时完成Android环境的创建后，AMS在attachApplicationLocked函数中重新处理等待发送的广播，对应代码如下：
```java
...............
// Check if a next-broadcast receiver is in this process...
if (!badApp && isPendingBroadcastProcessLocked(pid)) {
    try {
        //发送因目标进程还未启动，而处于等待状态的广播
        //sendPendingBroadcastsLocked将调用BroadcastQueue中的sendPendingBroadcastsLocked函数
        //sendPendingBroadcastsLocked最后仍会调用进程对应ApplicationThread的scheduleReceiver函数
        didSomething |= sendPendingBroadcastsLocked(app);
    } catch (Exception e) {
        // If the app died trying to launch the receiver we declare it 'bad'
        Slog.wtf(TAG, "Exception thrown dispatching broadcasts in " + app, e);
        badApp = true;
    }
}
.............
```
关于进程启动的过程，可以参考启动Activity的过程：二。

这一部分整体的流程大致如下图所示：

![图7](/images/android-n-ams/broadcast-7.jpg)

## 五、应用进程处理广播分析
最后，我们看看应用进程收到广播处理请求后的流程。

### 1、动态广播的处理流程
AMS处理动态广播时，最后通过Binder通信调用的是进程对应ApplicationThread的scheduleRegisteredReceiver接口：
```java
 public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
        int resultCode, String dataStr, Bundle extras, boolean ordered,
        boolean sticky, int sendingUser, int processState) throws RemoteException {
    ..............
    //调用的是LoadedApk中ReceiverDispatcher的内部类InnerReceiver的接口
    receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
            sticky, sendingUser);
}
```

我们看看InnerReceiver的performReceive函数：
```java
public void performReceive(Intent intent, int resultCode, String data,
        Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
    final LoadedApk.ReceiverDispatcher rd;
    if (intent == null) {
        ............
        rd = null;
    } else {
        rd = mDispatcher.get();
    }
    ..........
    if (rd != null) {
        //调用ReceiverDispatcher的performReceive函数
        rd.performReceive(intent, resultCode, data, extras,
                ordered, sticky, sendingUser);
    } else {
        // The activity manager dispatched a broadcast to a registered
        // receiver in this process, but before it could be delivered the
        // receiver was unregistered.  Acknowledge the broadcast on its
        // behalf so that the system's broadcast sequence can continue.
        // 处理特殊情况，在收到广播前，BroadcastReceiver已经unregister
        IActivityManager mgr = ActivityManagerNative.getDefault();
        try {
            if (extras != null) {
                extras.setAllowFds(false);
            }
            //通知AMS结果
            mgr.finishReceiver(this, resultCode, data, extras, false, intent.getFlags());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
}
```
我们跟进一下ReceiverDispatcher的performReceive函数：
```java
public void performReceive(Intent intent, int resultCode, String data,
        Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
    //构造一个Args对象（runnable对象）
    final Args args = new Args(intent, resultCode, data, extras, ordered,
            sticky, sendingUser);
    ..............
    //mActivityThread是一个Handler对象
    //若注册BroadcastReceiver时没有指定，则是ActivityThread主线程对应的Handler
    if (intent == null || !mActivityThread.post(args)) {
        //出现问题时，若处理的是有序广播，则需要通知AMS
        if (mRegistered && ordered) {
            IActivityManager mgr = ActivityManagerNative.getDefault();
            .............
            args.sendFinished(mgr);
        }
    }
}
```
当Args对象被Handler对应线程处理时，对应的run函数将被调用：
```java
public void run() {
    final BroadcastReceiver receiver = mReceiver;
    final boolean ordered = mOrdered;
    ...............
    final IActivityManager mgr = ActivityManagerNative.getDefault();
    final Intent intent = mCurIntent;
    ..............
    mCurIntent = null;
    mDispatched = true;
    ...........
    try {
        //此处并没有反射创建BroadcastReceiver
        ClassLoader cl =  mReceiver.getClass().getClassLoader();
        intent.setExtrasClassLoader(cl);
        intent.prepareToEnterProcess();
        setExtrasClassLoader(cl);

        //receiver设置pendingResult
        receiver.setPendingResult(this);
        receiver.onReceive(mContext, intent);
    } catch (Exception e) {
        if (mRegistered && ordered) {
            ................
            sendFinished(mgr);
        }
        ...........
    }

    //当调用BroadcastReceiver的goAsync时，会将pendingResult置为null
    //于是不会结束发送流程，直到调用pendingResult的finish函数
    if (receiver.getPendingResult() != null) {
        //Args继承PendingResult，此处也是调用PendingResult的finish函数
        finish();
    }
}
```
当BroadcastReceiver调用onReceive函数处理完广播后，调用PendingResult的finish函数结束处理流程：
```java
/**
* Finish the broadcast.  The current result will be sent and the
* next broadcast will proceed.
*/
public final void finish() {
    //对于动态广播而言，type为TYPE_REGISTERED或TYPE_UNREGISTERED
    if (mType == TYPE_COMPONENT) {
        ................
    //有序广播才返回结果
    } else if (mOrderedHint && mType != TYPE_UNREGISTERED){
        ................
        final IActivityManager mgr = ActivityManagerNative.getDefault();
        //Binder通信，调用AMS的finishReceiver函数
        sendFinished(mgr);
    }
}
```
我们最后看看AMS的finishReceiver函数：
```java
public void finishReceiver(IBinder who, int resultCode, String resultData,
        Bundle resultExtras, boolean resultAbort, int flags) {
    ............
    final long origId = Binder.clearCallingIdentity();
    try {
        boolean doNext = false;
        BroadcastRecord r;

        synchronized(this) {
            BroadcastQueue queue = (flags & Intent.FLAG_RECEIVER_FOREGROUND) != 0
                    ? mFgBroadcastQueue : mBgBroadcastQueue;
            r = queue.getMatchingOrderedReceiver(who);
            if (r != null) {
                //判断是否还需要发送后续的广播
                doNext = r.queue.finishReceiverLocked(r, resultCode,
                        resultData, resultExtras, resultAbort, true);
            }
        }

        if (doNext) {
            //处理下一个广播
            r.queue.processNextBroadcast(false);
        }
        //释放不必要的进程
        trimApplications();
    } finally {
        Binder.restoreCallingIdentity(origId);
    }
}
```
这里写图片描述

![图8](/images/android-n-ams/broadcast-8.jpg)

### 2、静态广播的处理流程
AMS处理静态广播时，最后通过Binder通信调用的是进程对应ApplicationThread的scheduleReceiver接口：
```java
public final void scheduleReceiver(Intent intent, ActivityInfo info,
        CompatibilityInfo compatInfo, int resultCode, String data, Bundle extras,
        boolean sync, int sendingUser, int processState) {
    ..............
    ReceiverData r = new ReceiverData(intent, resultCode, data, extras,
            sync, false, mAppThread.asBinder(), sendingUser);
    r.info = info;
    r.compatInfo = compatInfo;
    //发送消息触发ActivityThread调用handleReceiver函数
    sendMessage(H.RECEIVER, r);
}
```
跟进ActivityThread的handleReceiver函数：
```java
//构造ReceiverData进行处理，ReceiverData同样继承PendingResult，其Type为TYPE_COMPONENT
private void handleReceiver(ReceiverData data) {
    ............
    String component = data.intent.getComponent().getClassName();

    LoadedApk packageInfo = getPackageInfoNoCheck(
            data.info.applicationInfo, data.compatInfo);

    IActivityManager mgr = ActivityManagerNative.getDefault();

    BroadcastReceiver receiver;

    try {
        java.lang.ClassLoader cl = packageInfo.getClassLoader();
        data.intent.setExtrasClassLoader(cl);
        data.intent.prepareToEnterProcess();
        data.setExtrasClassLoader(cl);
        //对于静态广播而言，启动进程后就调用scheduleReceiver函数处理广播
        //BroadcastReceiver的实例还没有创建，因此需要在此进行反射初始化
        receiver = (BroadcastReceiver)cl.loadClass(component).newInstance();
    } catch (Exception e) {
        ...........
    }

    try {
        Application app = packageInfo.makeApplication(false, mInstrumentation);
        ..............
        ContextImpl context = (ContextImpl)app.getBaseContext();
        sCurrentBroadcastIntent.set(data.intent);
        receiver.setPendingResult(data);

        //调用BroadcastReceiver的onReceive进行处理
        receiver.onReceive(context.getReceiverRestrictedContext(),
                data.intent);
    } catch (Exception e) {
        ..............
        //静态广播出问题，需要通知AMS
        data.sendFinished(mgr);
        ..............
    } finally {
        //处理完毕，将进程的静态变量置为null
        sCurrentBroadcastIntent.set(null);
    }

    if (receiver.getPendingResult() != null) {
        //再次调用PendingResult的finish函数，通知AMS
        //不过此次的Type为TYPE_COMPONENT
        data.finish();
    }
}
```
看看这次PendingResult的finish函数流程：
```java
public final void finish() {
    if (mType == TYPE_COMPONENT) {
        final IActivityManager mgr = ActivityManagerNative.getDefault();
        //静态广播一定是由主线程处理事件
        //若主线程的QueuedWork中有事情还未处理完，则必须让事情做完后，才通知结果
        //保证AMS不会将进程kill掉
        if (QueuedWork.hasPendingWork()) {
            // If this is a broadcast component, we need to make sure any
            // queued work is complete before telling AM we are done, so
            // we don't have our process killed before that.  We now know
            // there is pending work; put another piece of work at the end
            // of the list to finish the broadcast, so we don't block this
            // thread (which may be the main thread) to have it finished.

            //主线的QueuedWork是单线程依次处理任务的
            QueuedWork.singleThreadExecutor().execute( new Runnable() {
                @Override public void run() {
                    ............
                    //通知AMS
                    sendFinished(mgr);
                }
            });
        } else {
            ............
            //无等待处理的事件，直接通知AMS处理结果
            sendFinished(mgr);
        }
    } else (mOrderedHint && mType != TYPE_UNREGISTERED){
        .................
    }
```

进程处理静态广播时，主要流程与处理动态广播时一致。
主要的差别就是：进程需要反射创建出BroadcastReceiver，同时广播处理完毕后，一定要向AMS返回结果。

## 六、总结
至此，Android中广播相关的流程分析完毕。

从实现原理看上，Android中的广播机制使用了观察者模式，基于消息的发布/订阅模型。
广播发送者和广播接收者分别属于观察者模式中的消息发布和订阅两端，AMS属于中间的处理中心。

整体来看，流程可粗略概括如下：

- 1.广播接收者BroadcastReceiver，通过Binder通信向AMS进行注册；

- 2.广播发送者通过Binder通信向AMS发送广播；

- 3.AMS收到广播后，查找与之匹配的BroadcastReceiver，然后将广播发送到BroadcastReceiver对应进程的消息队列中；

- 4.BroadcastReceiver对应进程的处理该消息时，将回调BroadcastReceiver中的onReceive()方法；

- 5.广播处理完毕后，BroadcastReceiver对应进程按需将执行结果通知给AMS，以便触发下一次广播发送。

对于不同的广播类型，以及不同的BroadcastReceiver注册方式，具体实现上会有不同，但整体的通信流程大致相似。
