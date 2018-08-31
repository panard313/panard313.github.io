---
layout: article
title: 进程管理 三
key: 20180829
tags:
  - ActivityManagerService
  - ams
  - process
  - oom
lang: zh-Hans
---

# 进程管理相关流程分析 三 computeOomAdjLocked

这一篇博客，我们来分析一下AMS进程管理流程中，负责计算进程oom_adj值的computeOomAdjLocked函数。

从难度上来讲，computeOomAdjLocked函数比updateOomAdjLocked函数简单，因为它的职责更明确和单一。
然而，由于Android定义的oom_adj种类庞杂，使得这个函数的分支很多，细节显得极其的繁琐。
因此从功利的角度来看，大家知道这个函数的用途和大概脉络即可。

不过对于一个框架工程师而言，阅读源码的耐心可能比写代码的能力更重要，因此我们还是耐着性子将代码看完。
“RTFSC”，毕竟大神是这么告诉我们的。

computeOomAdjLocked函数的代码很长， 因此在这篇博客中，我们还分段进行研究，然后试着进行总结。

## 一、computeOomAdjLocked Part-I
```java
private final int computeOomAdjLocked(ProcessRecord app, int cachedAdj, ProcessRecord TOP_APP,
        boolean doingAll, long now) {
    //之前的博客中提到过，updateOomAdjLocked函数每次更新oom_adj时，都会分配一个序号
    //此处就是根据序号判断是否已经处理过命令
    if (mAdjSeq == app.adjSeq) {
        // This adjustment has already been computed.
        return app.curRawAdj;
    }

    //ProcessRecord对应的ActivityThread不存在了
    //修改其中的一些变量，此时的oom_adj为CACHED_APP_MAX_ADJ，
    //其意义我们在前一篇博客中已经提到过
    if (app.thread == null) {
        app.adjSeq = mAdjSeq;
        app.curSchedGroup = ProcessList.SCHED_GROUP_BACKGROUND;
        app.curProcState = ActivityManager.PROCESS_STATE_CACHED_EMPTY;
        return (app.curAdj=app.curRawAdj=ProcessList.CACHED_APP_MAX_ADJ);
    }

    //初始化一些变量
    //这些变量的具体用途，在篇博客中我们不关注
    //大家只用留意一下ProcessRecord的schedGroup、procState和oom_adj即可
    app.adjTypeCode = ActivityManager.RunningAppProcessInfo.REASON_UNKNOWN;
    app.adjSource = null;
    app.adjTarget = null;
    app.empty = false;
    app.cached = false;

    final int activitiesSize = app.activities.size();

    //这个判断没啥意义，ProcessRecord中只有初始化时为maxAdj赋值
    //maxAdj取值为UNKNOWN_ADJ，即最大的1001
    if (app.maxAdj <= ProcessList.FOREGROUND_APP_ADJ) {
        //这部分代码就是修改app的curSchedGroup，并将oom_adj设置为maxAdj
        //实际过程中，应该是不会执行的的
        ......................
    }

    //保存当前TOP Activity的状态
    final int PROCESS_STATE_CUR_TOP = mTopProcessState;
    ......................
}
```

以上代码就是computeOomAdjLocked函数的第一部分。

从代码不难看出，这部分内容的主要目的是：

- 1、根据参数及进程的状态，决定是否需要进行后续的计算；
- 2、初始化一些变量。

## 二、computeOomAdjLocked Part-II

在第二部分，computeOomAdjLocked开始干“正事儿”了：
```java
.................
// Determine the importance of the process, starting with most
// important to least, and assign an appropriate OOM adjustment.
// 上面的这段注释为整个computeOomAdjLocked函数“代言”

int adj;
int schedGroup;
int procState;
boolean foregroundActivities = false;
BroadcastQueue queue;

//若进程包含正在前台显示的Activity
if (app == TOP_APP) {
    // The last app on the list is the foreground app.
    adj = ProcessList.FOREGROUND_APP_ADJ;

    //单独的一种schedGroup
    schedGroup = ProcessList.SCHED_GROUP_TOP_APP;
    app.adjType = "top-activity";

    //当前处理的是包含前台Activity的进程时，才会将该值置为true
    foregroundActivities = true;
    procState = PROCESS_STATE_CUR_TOP;
} else if (app.instrumentationClass != null) {
    //处理正在进行测试的进程

    // Don't want to kill running instrumentation.
    adj = ProcessList.FOREGROUND_APP_ADJ;
    schedGroup = ProcessList.SCHED_GROUP_DEFAULT;

    app.adjType = "instrumentation";
    procState = ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE;
} else if ((queue = isReceivingBroadcast(app)) != null) {
    //处理正在处理广播的进程

    // An app that is currently receiving a broadcast also
    // counts as being in the foreground for OOM killer purposes.
    // It's placed in a sched group based on the nature of the
    // broadcast as reflected by which queue it's active in.
    adj = ProcessList.FOREGROUND_APP_ADJ;

    //根据处理广播的Queue，决定调度策略
    schedGroup = (queue == mFgBroadcastQueue)
            ? ProcessList.SCHED_GROUP_DEFAULT : ProcessList.SCHED_GROUP_BACKGROUND;

    app.adjType = "broadcast";
    procState = ActivityManager.PROCESS_STATE_RECEIVER;
} else if (app.executingServices.size() > 0) {
    //处理Service正在运行的进程

    // An app that is currently executing a service callback also
    // counts as being in the foreground.
    adj = ProcessList.FOREGROUND_APP_ADJ;

    schedGroup = app.execServicesFg ?
            ProcessList.SCHED_GROUP_DEFAULT : ProcessList.SCHED_GROUP_BACKGROUND;

    procState = ActivityManager.PROCESS_STATE_SERVICE;
} else {
    //其它进程，在后续过程中再进一步处理
    // As far as we know the process is empty.  We may change our mind later.
    schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;

    // At this point we don't actually know the adjustment.  Use the cached adj
    // value that the caller wants us to.
    // 先将adj临时赋值为cachedAdj，即参数传入的UNKNOW_ADJ
    adj = cachedAdj;
    procState = ActivityManager.PROCESS_STATE_CACHED_EMPTY;

    app.cached = true;
    app.empty = true;
    app.adjType = "cch-empty";
}
..................
```

以上代码可以看作是computeOomAdjLocked的第二部分，从这部分代码可以看出：

- 1、包含前台Activity的进程、运行测试类的进程、处理广播的进程及包含正在运行服务的进程，

其oom_adj均被赋值为FOREGROUND_APP_ADJ，即从LMK的角度来看，它们的重要性是一致的。
但这些进程的procState不同，于是从AMS主动回收内存的角度来看，它们的重要性不同。

此外，这些进程的schedGroup不同。
之前的博客分析过，Process.java中提供了接口，可以调用Linux提供的接口函数设置schedGroup，使得进程具有不同的调度策略。
从获取CPU资源的能力来看，SCHED_GROUP_TOP_APP应该强于SCHED_GROUP_DEFAULT，
最后才轮到SCHED_GROUP_BACKGROUND。

- 2、对于其它种类的进程，这部分代码先将它们的oom_adj设置为UNKNOW_ADJ，

proc_state置为PROCESS_STATE_CACHED_EMPTY，在后续流程中再作进一步处理。

## 三、computeOomAdjLocked Part-III

这一部分代码主要处理包含Activity，但是Activity不在前台的进程。

注意到这些进程包括之前提到的正在处理广播、服务或测试的进程，以及oom_adj暂时为UNKNOW_ADJ的进程。
不过只有UNKNOW_ADJ对应的进程，才有可能进行实际的更新。
```java
..................
// Examine all activities if not already foreground.
if (!foregroundActivities && activitiesSize > 0) {
    //之前分析updateOomAdjLocked的第一部分时，简单提到过rankTaskLayersIfNeeded函数
    //该函数会更新包含Activity的Task的rankLayer
    //按照显示层次从上到下，rankLayer逐渐增加，对应的最大值就是VISIBLE_APP_LAYER_MAX
    int minLayer = ProcessList.VISIBLE_APP_LAYER_MAX;

    //依次轮询进程中的Activity
    for (int j = 0; j < activitiesSize; j++) {
        final ActivityRecord r = app.activities.get(j);
        ...................
        //如果进程包含可见Activity，即该进程是个可见进程
        if (r.visible) {
            // App has a visible activity; only upgrade adjustment.
            if (adj > ProcessList.VISIBLE_APP_ADJ) {
                //adj大于VISIBLE_APP_ADJ时，才更新对应的adj
                //之前提到的正在处理广播、服务或测试的进程，adj为FOREGROUND，是小于VISIBLE_APP_ADJ
                //因此不会在此更新
                adj = ProcessList.VISIBLE_APP_ADJ;
                app.adjType = "visible";
            }

            if (procState > PROCESS_STATE_CUR_TOP) {
                //与oom_adj类似，在条件满足时，更新procState
                procState = PROCESS_STATE_CUR_TOP;
            }

            //正在处理广播、服务或测试的进程，如果它们的调度策略为BACKGROUND
            //但又包含了可见Activity时，调度策略变更为DEFAULT
            schedGroup = ProcessList.SCHED_GROUP_DEFAULT;

            app.cached = false;
            app.empty = false;
            foregroundActivities = true;

            if (r.task != null && minLayer > 0) {
                 final int layer = r.task.mLayerRank;
                 if (layer >= 0 && minLayer > layer) {
                     //更新ranklayer
                     minLayer = layer;
                 }
            }

            //发现可见Activity时，直接可以结束循环
            break;
        } else if (r.state == ActivityState.PAUSING || r.state == ActivityState.PAUSED) {
            //如果进程包含处于PAUSING或PAUSED状态的Activity时

            //将其oom_adj调整为“用户可察觉”的的等级，这个等级还是很高的
            if (adj > ProcessList.PERCEPTIBLE_APP_ADJ) {
                adj = ProcessList.PERCEPTIBLE_APP_ADJ;
                app.adjType = "pausing";
            }
            if (procState > PROCESS_STATE_CUR_TOP) {
                procState = PROCESS_STATE_CUR_TOP;
            }
            schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
            app.cached = false;
            app.empty = false;
            foregroundActivities = true;

            //注意并不会break
        } else if (r.state == ActivityState.STOPPING) {
            //包含处于Stopping状态Activity的进程，其oom_adj也被置为PERCEPTIBLE_APP_ADJ

            if (adj > ProcessList.PERCEPTIBLE_APP_ADJ) {
                adj = ProcessList.PERCEPTIBLE_APP_ADJ;
                app.adjType = "stopping";
            }
            ................
            // 这种进程将被看作潜在的cached或empty进程
            if (!r.finishing) {
                if (procState > ActivityManager.PROCESS_STATE_LAST_ACTIVITY) {
                    procState = ActivityManager.PROCESS_STATE_LAST_ACTIVITY;
                }
            }
            app.cached = false;
            app.empty = false;
            foregroundActivities = true;
        } else {
            //只是含有cached-activity的进程，仅调整procState
            if (procState > ActivityManager.PROCESS_STATE_CACHED_ACTIVITY) {
                procState = ActivityManager.PROCESS_STATE_CACHED_ACTIVITY;
                app.adjType = "cch-act";
            }
        }

        if (adj == ProcessList.VISIBLE_APP_ADJ) {
            //不同可见进程的oom_adj有一定的差异，处在下层的oom_adj越大
            //即越老的Activity所在进程，重要性越低
            adj += minLayer;
        }
    }
}
..................
```

从上面的代码可以看出，computeOomAdjLocked的第三部分处理包含Activity的进程时，
进程最终的oom_adj将由其中最要的Activity决定。
即进程中存在可见Activity时，进程的oom_adj就为VISIBLE_APP_ADJ；
否则，若进程中存在处于PAUSING、PAUSED或STOPPING状态的Activity时，进程的oom_adj就为PERCEPTIBLE_APP_ADJ；
其余的进程仍是UNKNOW_ADJ。

## 四、computeOomAdjLocked Part-IV

computeOomAdjLocked的第四部分主要用于处理一些特殊的进程。
```java
...............
if (adj > ProcessList.PERCEPTIBLE_APP_ADJ
        || procState > ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE) {
    //进程包含前台服务或被强制在前台运行时
    //oom_adj被调整为PERCEPTIBLE_APP_ADJ，只是procState略有不同
    if (app.foregroundServices) {
        // The user is aware of this app, so make it visible.
        adj = ProcessList.PERCEPTIBLE_APP_ADJ;
        procState = ActivityManager.PROCESS_STATE_FOREGROUND_SERVICE;
        app.cached = false;
        app.adjType = "fg-service";
        schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
    } else if (app.forcingToForeground != null) {
        // The user is aware of this app, so make it visible.
        adj = ProcessList.PERCEPTIBLE_APP_ADJ;
        procState = ActivityManager.PROCESS_STATE_IMPORTANT_FOREGROUND;
        app.cached = false;
        app.adjType = "force-fg";
        app.adjSource = app.forcingToForeground;
        schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
    }
}

//AMS的HeavyWeight进程单独处理
if (app == mHeavyWeightProcess) {
    if (adj > ProcessList.HEAVY_WEIGHT_APP_ADJ) {
        // We don't want to kill the current heavy-weight process.
        adj = ProcessList.HEAVY_WEIGHT_APP_ADJ;
        schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;

        app.cached = false;
        app.adjType = "heavy";
    }

    if (procState > ActivityManager.PROCESS_STATE_HEAVY_WEIGHT) {
        procState = ActivityManager.PROCESS_STATE_HEAVY_WEIGHT;
    }
}

//home进程特殊处理
if (app == mHomeProcess) {
    if (adj > ProcessList.HOME_APP_ADJ) {
        // This process is hosting what we currently consider to be the
        // home app, so we don't want to let it go into the background.
        adj = ProcessList.HOME_APP_ADJ;
        schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;

        app.cached = false;
        app.adjType = "home";
    }
    if (procState > ActivityManager.PROCESS_STATE_HOME) {
        procState = ActivityManager.PROCESS_STATE_HOME;
    }
}

//前台进程之前的一个进程
if (app == mPreviousProcess && app.activities.size() > 0) {
    if (adj > ProcessList.PREVIOUS_APP_ADJ) {
        // This was the previous process that showed UI to the user.
        // We want to try to keep it around more aggressively, to give
        // a good experience around switching between two apps.
        adj = ProcessList.PREVIOUS_APP_ADJ;
        schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;
        app.cached = false;
        app.adjType = "previous";
    }
    if (procState > ActivityManager.PROCESS_STATE_LAST_ACTIVITY) {
        procState = ActivityManager.PROCESS_STATE_LAST_ACTIVITY;
    }
}

// By default, we use the computed adjustment.  It may be changed if
// there are applications dependent on our services or providers, but
// this gives us a baseline and makes sure we don't get into an
// infinite recursion.
app.adjSeq = mAdjSeq;
app.curRawAdj = adj;
app.hasStartedServices = false;

//处理正在进行backup工作的进程
if (mBackupTarget != null && app == mBackupTarget.app) {
    // If possible we want to avoid killing apps while they're being backed up
    if (adj > ProcessList.BACKUP_APP_ADJ) {
        ..............
        adj = ProcessList.BACKUP_APP_ADJ;
        if (procState > ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND) {
            procState = ActivityManager.PROCESS_STATE_IMPORTANT_BACKGROUND;
        }
        app.adjType = "backup";
        app.cached = false;
    }
    if (procState > ActivityManager.PROCESS_STATE_BACKUP) {
        procState = ActivityManager.PROCESS_STATE_BACKUP;
    }
}
..................
```

至此，我们应该可以看出computeOomAdjLocked处理一个进程时，按照重要性由高到底的顺序，
逐步判断该进程是否满足对应的条件。
尽管计算一个进程的oom_adj时，会经过上述所有的判断，但当一个进程已经满足重要性较高的条件时，
后续的判断实际上不会更改它已经获得的oom_adj。

上面四部分的逻辑基本上如下图所示：

![image-1](/images/android-n-ams/process-3-1.jpg)


后续的处理逻辑，仍然满足上述规则。
只是在考虑含有Service和Provider的进程时，整体流程显得极其复杂，
融入上图的成本太高，因此就不再画图了。

## 五、computeOomAdjLocked Part-V

computeOomAdjLocked的第五部分，主要是处理包含服务的进程。
这一部分代码写的比较繁琐，复杂度应该超过了前四部分的和，
因此我们进一步分段说明。

### 1、Unbounded Service的处理

当进程中包含Unbounded Service时，进程的oom_adj先按照Unbounded Service的处理方式进行调整。
```java
.................
//依次处理进程中的每一个Service
for (int is = app.services.size()-1;
    is >= 0 && (adj > ProcessList.FOREGROUND_APP_ADJ
            || schedGroup == ProcessList.SCHED_GROUP_BACKGROUND
            || procState > ActivityManager.PROCESS_STATE_TOP);
    is--) {
    ServiceRecord s = app.services.valueAt(is);

    //Service被已Unbounded Service的方式启动过
    if (s.startRequested) {
        app.hasStartedServices = true;

        //调整procState
        if (procState > ActivityManager.PROCESS_STATE_SERVICE) {
            procState = ActivityManager.PROCESS_STATE_SERVICE;
        }

        if (app.hasShownUi && app != mHomeProcess) {
            // If this process has shown some UI, let it immediately
            // go to the LRU list because it may be pretty heavy with
            // UI stuff.  We'll tag it with a label just to help
            // debug and understand what is going on.
            // 仅有含有服务且显示过UI的进程，由于其占用内存可能较多，因此需要尽早回收
            // 故此处不调整其oom_adj
            if (adj > ProcessList.SERVICE_ADJ) {
                app.adjType = "cch-started-ui-services";
            }
        } else {
            if (now < (s.lastActivity + ActiveServices.MAX_SERVICE_INACTIVITY)) {
                //MAX_SERVICE_INACTIVITY为activity启动service后，系统最多保留Service的时间

                // This service has seen some activity within
                // recent memory, so we will keep its process ahead
                // of the background processes.

                //此时进程的oom_adj就可以被调整为后台服务对应的SERVICE_ADJ
                //adj大于500的进程均会受此判断的影响
                if (adj > ProcessList.SERVICE_ADJ) {
                    adj = ProcessList.SERVICE_ADJ;
                    app.adjType = "started-services";
                    app.cached = false;
                }
            }

            //处理Service存在超时的情况，可见超时时也不会调整oom_adj
            // If we have let the service slide into the background
            // state, still have some text describing what it is doing
            // even though the service no longer has an impact.
            if (adj > ProcessList.SERVICE_ADJ) {
                app.adjType = "cch-started-services";
            }
        }
    }
.................
```

从上面的代码可以看出，当进程中含有Unbounded Service时，
如果进程之前没有启动过UI，且Unbounded Service存活的时间没有超时，
进程的oom_ad才能被调整为SERVICE_ADJ；否则进程的oom_adj仍然是UNKNOW_ADJ或其它大于500的值。

### 2、Bounded Service的处理

这部分代码紧接着上述流程。
即进程将先按照Unbounded Service的方式调整oom_adj，
然后再按照Bounded Service的方式进一步调整。

当然，若Service仅为Unbounded Service或Bounded Service中的一种时，
computeOomAdjLocked函数的第五部分，只会按照一种方式调整oom_adj。

Bounded Service的处理方式，远比Unbounded Service复杂，依赖于客户端的oom_adj和绑定服务时使用的flag。
```java
....................
    //如果该Service还被客户端Bounded，即是Bounded Service时
    for (int conni = s.connections.size()-1;
            conni >= 0 && (adj > ProcessList.FOREGROUND_APP_ADJ
                    || schedGroup == ProcessList.SCHED_GROUP_BACKGROUND
                    || procState > ActivityManager.PROCESS_STATE_TOP);
            conni--) {
        ArrayList<ConnectionRecord> clist = s.connections.valueAt(conni);

        //客户端可以通过一个Connection以不同的参数绑定Service
        //因此，一个Service可以对应多个Connection，一个Connection又对应多个ConnectionRecord
        //这里依次处理每一个ConnectionRecord
        for (int i = 0;
                i < clist.size() && (adj > ProcessList.FOREGROUND_APP_ADJ
                        || schedGroup == ProcessList.SCHED_GROUP_BACKGROUND
                        || procState > ActivityManager.PROCESS_STATE_TOP);
                i++) {
            ConnectionRecord cr = clist.get(i);

            if (cr.binding.client == app) {
                // Binding to ourself is not interesting.
                continue;
            }

            //当BIND_WAIVE_PRIORITY为1时，客户端就不会影响服务端
            //if中的流程就可以略去；否则，客户端就会影响服务端
            if ((cr.flags&Context.BIND_WAIVE_PRIORITY) == 0) {
                ProcessRecord client = cr.binding.client;

                //计算出客户端进程的oom_adj
                //由此可看出Android oom_adj的计算多么麻烦
                //要是客户端进程中，又有个服务进程被绑定，那么将再计算其客户端进程的oom_adj？！
                int clientAdj = computeOomAdjLocked(client, cachedAdj,
                        TOP_APP, doingAll, now);

                int clientProcState = client.curProcState;
                if (clientProcState >= ActivityManager.PROCESS_STATE_CACHED_ACTIVITY) {
                    // If the other app is cached for any reason, for purposes here
                    // we are going to consider it empty.  The specific cached state
                    // doesn't propagate except under certain conditions.
                    clientProcState = ActivityManager.PROCESS_STATE_CACHED_EMPTY;
                }

                String adjType = null;

                //BIND_ALLOW_OOM_MANAGEMENT置为1时，先按照通常的处理方式，调整服务端进程的adjType
                if ((cr.flags&Context.BIND_ALLOW_OOM_MANAGEMENT) != 0) {
                    //与前面分析Unbounded Service基本一致，若进程显示过UI或Service超时
                    //会将clientAdj修改为当前进程的adj，即不需要考虑客户端进程了
                    if (app.hasShownUi && app != mHomeProcess) {
                        if (adj > clientAdj) {
                            adjType = "cch-bound-ui-services";
                        }
                        app.cached = false;
                        clientAdj = adj;
                        clientProcState = procState;
                    } else {
                        if (now >= (s.lastActivity
                                + ActiveServices.MAX_SERVICE_INACTIVITY)) {
                            if (adj > clientAdj) {
                                adjType = "cch-bound-services";
                            }
                            clientAdj = adj;
                        }
                    }
                }

                //根据情况，按照clientAdj调整当前进程的adj
                if (adj > clientAdj) {
                    // If this process has recently shown UI, and
                    // the process that is binding to it is less
                    // important than being visible, then we don't
                    // care about the binding as much as we care
                    // about letting this process get into the LRU
                    // list to be killed and restarted if needed for
                    // memory.
                    // 上面的注释很清楚
                    if (app.hasShownUi && app != mHomeProcess
                            && clientAdj > ProcessList.PERCEPTIBLE_APP_ADJ) {
                        adjType = "cch-bound-ui-services";
                    } else {
                        //以下的流程表明，client和flag将同时影响Service进程的adj

                        if ((cr.flags&(Context.BIND_ABOVE_CLIENT
                                |Context.BIND_IMPORTANT)) != 0) {
                            //从这里再次可以看出，Service重要性小于等于Client
                            adj = clientAdj >= ProcessList.PERSISTENT_SERVICE_ADJ
                                    ? clientAdj : ProcessList.PERSISTENT_SERVICE_ADJ;

                        //BIND_NOT_VISIBLE表示不将服务端当作visible进程看待
                        //于是，即使客户端的adj小于PERCEPTIBLE_APP_ADJ，service也只能取到PERCEPTIBLE_APP_ADJ
                        } else if ((cr.flags&Context.BIND_NOT_VISIBLE) != 0
                                && clientAdj < ProcessList.PERCEPTIBLE_APP_ADJ
                                && adj > ProcessList.PERCEPTIBLE_APP_ADJ) {
                            adj = ProcessList.PERCEPTIBLE_APP_ADJ;
                        } else if (clientAdj >= ProcessList.PERCEPTIBLE_APP_ADJ) {
                            adj = clientAdj;
                        } else {
                            if (adj > ProcessList.VISIBLE_APP_ADJ) {
                                adj = Math.max(clientAdj, ProcessList.VISIBLE_APP_ADJ);
                            }
                        }

                        if (!client.cached) {
                            app.cached = false;
                        }
                        adjType = "service";
                    }
                }

                if ((cr.flags&Context.BIND_NOT_FOREGROUND) == 0) {
                    //进一步更具client调整当前进程的procState、schedGroup等
                    ...................
                } else {
                    ...................
                }
                .................
                if (procState > clientProcState) {
                    procState = clientProcState;
                }
                //其它参数的赋值
                .................
            }

            if ((cr.flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                app.treatLikeActivity = true;
            }

            //取出ConnectionRecord所在的Activity
            final ActivityRecord a = cr.activity;

            //BIND_ADJUST_WITH_ACTIVITY值为1时，表示服务端可以根据客户端Activity的oom_adj作出相应的调整
            if ((cr.flags&Context.BIND_ADJUST_WITH_ACTIVITY) != 0) {
                if (a != null && adj > ProcessList.FOREGROUND_APP_ADJ &&
                        (a.visible || a.state == ActivityState.RESUMED ||
                                a.state == ActivityState.PAUSING)) {
                //BIND_ADJUST_WITH_ACTIVITY置为1，且绑定的activity可见或在前台时，
                //Service进程的oom_adj可以变为FOREGROUND_APP_ADJ
                adj = ProcessList.FOREGROUND_APP_ADJ;

                //BIND_NOT_FOREGROUND为0时，才准许调整Service进程的调度优先级
                if ((cr.flags&Context.BIND_NOT_FOREGROUND) == 0) {
                    if ((cr.flags&Context.BIND_IMPORTANT) != 0) {
                        schedGroup = ProcessList.SCHED_GROUP_TOP_APP;
                    } else {
                        schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
                    }
                }

                //改变其它参数
                app.cached = false;
                app.adjType = "service";
                app.adjTypeCode = ActivityManager.RunningAppProcessInfo
                        .REASON_SERVICE_IN_USE;
                app.adjSource = a;
                app.adjSourceProcState = procState;
                app.adjTarget = s.name;
            }
        }
    }
}
....................
```

以上就是计算含有Service的进程的oom_adj的全部过程。
从代码来看当进程仅含有Unbounded Service时，整个计算过程比较单纯，只要进程没有显示过UI，且Service的存在没有超时时，
进程的oom_adj就被调整为SERVICE_ADJ。
当进程含有的是Bounded Service时，整个计算的复杂度就飙升了，
它将考虑到Bound时使用的flag及客户端的情况，综合调整进程的oom_adj。

不过正因为Bounded Service的处理流程依赖于大量的flag，而这些flag基本很少用到，
因此个人怀疑这些代码都是些实验性质的代码。手机真正运行时，使用的频率可能并不高。

从另一个角度来看，这么设计似乎也是合理的。
当一个进程中的Service被许多客户端需求时，确实应该给这个进程机会，提高自己的重要性。
不知道如此细粒度的处理，Google是如何进行测试，并得到有效结论的？虽不明，但觉厉啊。

## 六、computeOomAdjLocked Part-VI
computeOomAdjLocked的第六部分主要是处理含有ContentProvider的进程。
由于ContentProvider也有客户端，因此同样需要根据客户端进程调整当前进程的oom_adj。
```java
....................
//依次处理进程中的ContentProvider
for (int provi = app.pubProviders.size()-1;
                provi >= 0 && (adj > ProcessList.FOREGROUND_APP_ADJ
                        || schedGroup == ProcessList.SCHED_GROUP_BACKGROUND
                        || procState > ActivityManager.PROCESS_STATE_TOP);
                provi--) {
    ContentProviderRecord cpr = app.pubProviders.valueAt(provi);

    //依次处理ContentProvider的客户端
    for (int i = cpr.connections.size()-1;
            i >= 0 && (adj > ProcessList.FOREGROUND_APP_ADJ
                    || schedGroup == ProcessList.SCHED_GROUP_BACKGROUND
                    || procState > ActivityManager.PROCESS_STATE_TOP);
            i--) {
        ContentProviderConnection conn = cpr.connections.get(i);

        ProcessRecord client = conn.client;
        if (client == app) {
            // Being our own client is not interesting.
            continue;
        }

        //计算客户端的oom_adj
        int clientAdj = computeOomAdjLocked(client, cachedAdj, TOP_APP, doingAll, now);
        int clientProcState = client.curProcState;
        if (clientProcState >= ActivityManager.PROCESS_STATE_CACHED_ACTIVITY) {
            // If the other app is cached for any reason, for purposes here
            // we are going to consider it empty.
            clientProcState = ActivityManager.PROCESS_STATE_CACHED_EMPTY;
        }

        //与Unbounded Service的处理基本类似
        if (adj > clientAdj) {
            if (app.hasShownUi && app != mHomeProcess
                    && clientAdj > ProcessList.PERCEPTIBLE_APP_ADJ) {
                app.adjType = "cch-ui-provider";
            } else {
                //根据clientAdj，调整当前进程的adj
                adj = clientAdj > ProcessList.FOREGROUND_APP_ADJ
                        ? clientAdj : ProcessList.FOREGROUND_APP_ADJ;
                        app.adjType = "provider";
            }

            //调整其它变量
            app.cached &= client.cached;
            app.adjTypeCode = ActivityManager.RunningAppProcessInfo
                    .REASON_PROVIDER_IN_USE;
            app.adjSource = client;
            app.adjSourceProcState = clientProcState;
            app.adjTarget = cpr.name;
        }

        //进一步调整调度策略和procState
        ....................

        //特殊情况的处理
        // If the provider has external (non-framework) process
        // dependencies, ensure that its adjustment is at least
        // FOREGROUND_APP_ADJ.
        if (cpr.hasExternalProcessHandles()) {
            if (adj > ProcessList.FOREGROUND_APP_ADJ) {
                adj = ProcessList.FOREGROUND_APP_ADJ;
                schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
                app.cached = false;
                app.adjType = "provider";
                app.adjTarget = cpr.name;
            }
            if (procState > ActivityManager.PROCESS_STATE_IMPORTANT_FOREGROUND) {
                procState = ActivityManager.PROCESS_STATE_IMPORTANT_FOREGROUND;
            }
        }
    }
}

//如果进程之前运行过ContentProvider，同时ContentProvider的存活时间没有超时
//那么进程的adj可以变为PREVIOUS_APP_ADJ
if (app.lastProviderTime > 0 && (app.lastProviderTime+CONTENT_PROVIDER_RETAIN_TIME) > now) {
    if (adj > ProcessList.PREVIOUS_APP_ADJ) {
        adj = ProcessList.PREVIOUS_APP_ADJ;
        schedGroup = ProcessList.SCHED_GROUP_BACKGROUND;

        app.cached = false;
        app.adjType = "provider";
    }
    if (procState > ActivityManager.PROCESS_STATE_LAST_ACTIVITY) {
        procState = ActivityManager.PROCESS_STATE_LAST_ACTIVITY;
    }
}
....................
```

从代码来看，处理含有ContentProvider的进程时，相对比较简单。
基本上与处理含有Unbounded Service的进程一致，只是最后增加了一些特殊情况的处理。

## 七、computeOomAdjLocked Part-VII
现在我们来看看computeOomAdjLocked函数的最后一部分。
```java
//根据进程信息，进一步调整procState
...................

//对Service进程做一些特殊处理
if (adj == ProcessList.SERVICE_ADJ) {
    if (doingAll) {
        //每次updateOomAdj时，将mNewNumAServiceProcs置为0
        //然后LRU list中，从后往前数，前1/3的service进程就是AService
        //其余的就是bService
        //mNumServiceProcs为上一次update时，service进程的数量
        app.serviceb = mNewNumAServiceProcs > (mNumServiceProcs/3);

        //记录这一次update后，service进程的数量
        //update完毕后，该值将赋给mNumServiceProcs
        mNewNumServiceProcs++;
        .............
        if (!app.serviceb) {
            // This service isn't far enough down on the LRU list to
            // normally be a B service, but if we are low on RAM and it
            // is large we want to force it down since we would prefer to
            // keep launcher over it.
            // 如果不是bService，但内存回收等级过高，也被视为bService
            if (mLastMemoryLevel > ProcessStats.ADJ_MEM_FACTOR_NORMAL
                    && app.lastPss >= mProcessList.getCachedRestoreThresholdKb()) {
                app.serviceHighRam = true;
                app.serviceb = true;
                ................
            } else {
                //LRU中后1/3的Service，都是AService
                mNewNumAServiceProcs++;
                ............
            }
        } else {
            app.serviceHighRam = false;
        }
    }
    //将bService的oom_adj调整为SERVICE_B_ADJ
    if (app.serviceb) {
        adj = ProcessList.SERVICE_B_ADJ;
    }
}

//计算完毕
app.curRawAdj = adj;

.............
//if基本没有用，maxAdj已经是最大的UNKNOW_ADJ
if (adj > app.maxAdj) {
    adj = app.maxAdj;
    if (app.maxAdj <= ProcessList.PERCEPTIBLE_APP_ADJ) {
        schedGroup = ProcessList.SCHED_GROUP_DEFAULT;
    }
}

//最后做一些记录和调整
.............

return app.curRawAdj;
```

从上面的代码可以看出，computeOomAdjLocked 的最后一部分主要是针对Service进程作一些处理。
LRU表从后往前，比较重要的Service进程是AService，不太重要的就是bService。

同时，判断的依据与前一次记录的Service进程总数有关。
即若前一次Service进程数量大，本次Service进程数变少了，那么本次AService的比例将变大；
同样若前一次Service进程数量小，本次Service进程数量变多了，那么本次BService的比例将变大。
这也算是一种动态自适应吧。

## 八、总结

computeOomAdjLocked函数基本上就介绍到这里了。
其实该函数的原理还是依据进程中运行的组件以及进程的种类，来计算相应的oom_adj。
基本过程还是比较容易看懂，就是处理Bounded Service时，需要同时考虑客户端和绑定时使用的flag，
使得整个代码显得很繁琐。
