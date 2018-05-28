# Activity start
## launcher.java
```java
    public final class Launcher extends Activity {
        public void onClick(View v) {
            startActivitySafely(intent, tag);
        }
    }

    void startActivitySafely(Intent intent, Object tag) {
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        try {
            startActivity(intent);
            /*action = "android.intent.action.Main"，category="android.intent.category.LAUNCHER", cmp="shy.luo.activity/.MainActivity", 表示它要启动的Activity为shy.luo.activity.MainActivity。Intent.FLAG_ACTIVITY_NEW_TASK表示要在一个新的Task中启动这个Activity*/
        }
    }
```
## frameworks/base/core/java/android/app/Activity.java
```java
    public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks {

    ......

        @Override
        public void startActivity(Intent intent) {
            startActivityForResult(intent, -1);
        }

        ......

        public void startActivityForResult(Intent intent, int requestCode) {
            if (mParent == null) {
                //这里的mInstrumentation是Activity类的成员变量，它的类型是Intrumentation，定义在frameworks/base/core/java/android/app/Instrumentation.java文件中，它用来监控应用程序和系统的交互。
                Instrumentation.ActivityResult ar =
                    mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    //mMainThread是Launcher的ActivityThread,通过getApplicationThread可获得ApplicationThread, which is a binder object
                    intent, requestCode);
                ......
            } else {
                ......
            }
        }

    }
```
## frameworks/base/core/java/android/app/Instrumentation.java
```java
    public class Instrumentation {

        ......

        public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode) {
            IApplicationThread whoThread = (IApplicationThread) contextThread;
            if (mActivityMonitors != null) {
                ......
            }
            try {
                int result = ActivityManagerNative.getDefault() //ActivityManagerProxy
                    .startActivity(whoThread, intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),  //intent MIME type
                    null, 0, token, target != null ? target.mEmbeddedID : null,
                    requestCode, false, false);
                ......
            } catch (RemoteException e) {
            }
            return null;
        }

        ......

    }
```
## frameworks/base/core/java/android/app/ActivityManagerNative.java
```java
    class ActivityManagerProxy implements IActivityManager
    {

        ......

        public int startActivity(IApplicationThread caller, Intent intent,
                String resolvedType /*null*/, Uri[] grantedUriPermissions /*null*/, int grantedMode /*0*/,
                IBinder resultTo, String resultWho /*null*/,
                int requestCode, boolean onlyIfNeeded,
                boolean debug) throws RemoteException {
            Parcel data = Parcel.obtain();
            Parcel reply = Parcel.obtain();
            data.writeInterfaceToken(IActivityManager.descriptor);
            data.writeStrongBinder(caller != null ? caller.asBinder() : null);
            intent.writeToParcel(data, 0);
            data.writeString(resolvedType);
            data.writeTypedArray(grantedUriPermissions, 0);
            data.writeInt(grantedMode);
            data.writeStrongBinder(resultTo);
            data.writeString(resultWho);
            data.writeInt(requestCode);
            data.writeInt(onlyIfNeeded ? 1 : 0);
            data.writeInt(debug ? 1 : 0);
            mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
            reply.readException();
            int result = reply.readInt();
            reply.recycle();
            data.recycle();
            return result;
        }

        ......

    }
```
## through binder:frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
```java
    public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

        ......

        public final int startActivity(IApplicationThread caller,
                Intent intent, String resolvedType, Uri[] grantedUriPermissions,
                int grantedMode, IBinder resultTo,
                String resultWho, int requestCode, boolean onlyIfNeeded,
                boolean debug) {
            return mMainStack.startActivityMayWait(caller, intent, resolvedType,
                grantedUriPermissions, grantedMode, resultTo, resultWho,
                requestCode, onlyIfNeeded, debug, null, null);
            //这里只是简单地将操作转发给成员变量mMainStack的startActivityMayWait函数，这里的mMainStack的类型为ActivityStack。
        }


        ......

    }
```

## frameworks/base/services/java/com/android/server/am/ActivityStack.java
```java
    public class ActivityStack {

        ......

        final int startActivityMayWait(IApplicationThread caller,
                Intent intent, String resolvedType, Uri[] grantedUriPermissions,
                int grantedMode, IBinder resultTo,
                String resultWho, int requestCode, boolean onlyIfNeeded,
                boolean debug, WaitResult outResult, Configuration config) {

            ......

            boolean componentSpecified = intent.getComponent() != null;

            // Don't modify the client's object!
            intent = new Intent(intent);

            // Collect information about the target of the Intent.
            ActivityInfo aInfo;
            try {
                ResolveInfo rInfo =
                    AppGlobals.getPackageManager().resolveIntent(
                    intent, resolvedType,
                    PackageManager.MATCH_DEFAULT_ONLY
                    | ActivityManagerService.STOCK_PM_FLAGS);
                aInfo = rInfo != null ? rInfo.activityInfo : null;
            } catch (RemoteException e) {
                ......
            }

            if (aInfo != null) {
                // Store the found target back into the intent, because now that
                // we have it we never want to do this again.  For example, if the
                // user navigates back to this point in the history, we should
                // always restart the exact same activity.
                intent.setComponent(new ComponentName(
                    aInfo.applicationInfo.packageName, aInfo.name));
                    //aInfo.applicationInfo.packageName的值为"shy.luo.activity"，aInfo.name的值为"shy.luo.activity.MainActivity"，这是在这个实例的配置文件AndroidManifest.xml里面配置的。
                ......
            }

            synchronized (mService) {
                int callingPid;
                int callingUid;
                if (caller == null) {
                    ......
                } else {
                    callingPid = callingUid = -1;
                }

                mConfigWillChange = config != null
                    && mService.mConfiguration.diff(config) != 0;

                ......

                if (mMainStack && aInfo != null &&
                    (aInfo.applicationInfo.flags&ApplicationInfo.FLAG_CANT_SAVE_STATE) != 0) {

                        ......

                }

                int res = startActivityLocked(caller, intent, resolvedType,
                    grantedUriPermissions, grantedMode, aInfo,
                    resultTo, resultWho, requestCode, callingPid, callingUid,
                    onlyIfNeeded, componentSpecified);

                if (mConfigWillChange && mMainStack) {
                    ......
                }

                ......

                if (outResult != null) {
                    ......
                }

                return res;
            }

        }

        ......

        final int startActivityLocked(IApplicationThread caller,
                Intent intent, String resolvedType,
                Uri[] grantedUriPermissions,
                int grantedMode, ActivityInfo aInfo, IBinder resultTo,
                    String resultWho, int requestCode,
                int callingPid, int callingUid, boolean onlyIfNeeded,
                boolean componentSpecified) {
                int err = START_SUCCESS;

            ProcessRecord callerApp = null;
            if (caller != null) {
                callerApp = mService.getRecordForAppLocked(caller);
                if (callerApp != null) {
                    callingPid = callerApp.pid;
                    callingUid = callerApp.info.uid;
                } else {
                    ......
                }
            }

            ......

            ActivityRecord sourceRecord = null;
            ActivityRecord resultRecord = null;
            if (resultTo != null) {
                int index = indexOfTokenLocked(resultTo);

                ......

                if (index >= 0) {
                    sourceRecord = (ActivityRecord)mHistory.get(index);
                    if (requestCode >= 0 && !sourceRecord.finishing) {
                        ......
                    }
                }
            }

            int launchFlags = intent.getFlags();

            if ((launchFlags&Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0
                && sourceRecord != null) {
                ......
            }

            if (err == START_SUCCESS && intent.getComponent() == null) {
                ......
            }

            if (err == START_SUCCESS && aInfo == null) {
                ......
            }

            if (err != START_SUCCESS) {
                ......
            }

            ......

            ActivityRecord r = new ActivityRecord(mService, this, callerApp, callingUid,
                intent, resolvedType, aInfo, mService.mConfiguration,
                resultRecord, resultWho, requestCode, componentSpecified);

            ......

            return startActivityUncheckedLocked(r, sourceRecord,
                grantedUriPermissions, grantedMode, onlyIfNeeded, true);
        }


        final int startActivityUncheckedLocked(ActivityRecord r,
            ActivityRecord sourceRecord, Uri[] grantedUriPermissions,
            int grantedMode, boolean onlyIfNeeded, boolean doResume) {
            final Intent intent = r.intent;
            final int callingUid = r.launchedFromUid;

            int launchFlags = intent.getFlags();

            // We'll invoke onUserLeaving before onPause only if the launching
            // activity did not explicitly state that this is an automated launch.
            mUserLeaving = (launchFlags&Intent.FLAG_ACTIVITY_NO_USER_ACTION) == 0;

            ......

            ActivityRecord notTop = (launchFlags&Intent.FLAG_ACTIVITY_PREVIOUS_IS_TOP)
                != 0 ? r : null;

            // If the onlyIfNeeded flag is set, then we can do this if the activity
            // being launched is the same as the one making the call...  or, as
            // a special case, if we do not know the caller then we count the
            // current top activity as the caller.
            if (onlyIfNeeded) {
                ......
            }

            if (sourceRecord == null) {
                ......
            } else if (sourceRecord.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
                ......
            } else if (r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE
                || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK) {
                ......
            }

            if (r.resultTo != null && (launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
                ......
            }

            boolean addingToTask = false;
            if (((launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0 &&
                (launchFlags&Intent.FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
                || r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK
                || r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
                    // If bring to front is requested, and no result is requested, and
                    // we can find a task that was started with this same
                    // component, then instead of launching bring that one to the front.
                    if (r.resultTo == null) {
                        // See if there is a task to bring to the front.  If this is
                        // a SINGLE_INSTANCE activity, there can be one and only one
                        // instance of it in the history, and it is always in its own
                        // unique task, so we do a special search.
                        ActivityRecord taskTop = r.launchMode != ActivityInfo.LAUNCH_SINGLE_INSTANCE
                            ? findTaskLocked(intent, r.info)
                            : findActivityLocked(intent, r.info);
                        if (taskTop != null) {
                            ......
                        }
                    }
            }

            ......

            if (r.packageName != null) {
                // If the activity being launched is the same as the one currently
                // at the top, then we need to check if it should only be launched
                // once.
                ActivityRecord top = topRunningNonDelayedActivityLocked(notTop);
                if (top != null && r.resultTo == null) {
                    if (top.realActivity.equals(r.realActivity)) {
                        ......
                    }
                }

            } else {
                ......
            }

            boolean newTask = false;

            // Should this be considered a new task?
            if (r.resultTo == null && !addingToTask
                && (launchFlags&Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
                    // todo: should do better management of integers.
                    mService.mCurTask++;
                    if (mService.mCurTask <= 0) {
                        mService.mCurTask = 1;
                    }
                    r.task = new TaskRecord(mService.mCurTask, r.info, intent,
                        (r.info.flags&ActivityInfo.FLAG_CLEAR_TASK_ON_LAUNCH) != 0);
                    ......
                    newTask = true;
                    if (mMainStack) {
                        mService.addRecentTaskLocked(r.task);
                    }

            } else if (sourceRecord != null) {
                ......
            } else {
                ......
            }

            ......

            startActivityLocked(r, newTask, doResume);
            return START_SUCCESS;
        }


        private final void startActivityLocked(ActivityRecord r, boolean newTask,
                boolean doResume) {
            final int NH = mHistory.size();

            int addPos = -1;

            if (!newTask) {
                ......
            }

            // Place a new activity at top of stack, so it is next to interact
            // with the user.
            if (addPos < 0) {
                addPos = NH;
            }

            // If we are not placing the new activity frontmost, we do not want
            // to deliver the onUserLeaving callback to the actual frontmost
            // activity
            if (addPos < NH) {
                ......
            }

            // Slot the activity into the history stack and proceed
            mHistory.add(addPos, r);
            r.inHistory = true;
            r.frontOfTask = newTask;
            r.task.numActivities++;
            if (NH > 0) {
                // We want to show the starting preview window if we are
                // switching to a new task, or the next activity's process is
                // not currently running.
                ......
            } else {
                // If this is the first activity, don't do any fancy animations,
                // because there is nothing for it to animate on top of.
                ......
            }

            ......

            if (doResume) {
                resumeTopActivityLocked(null);
            }
        }


        final boolean resumeTopActivityLocked(ActivityRecord prev) {
            // Find the first activity that is not finishing.
            ActivityRecord next = topRunningActivityLocked(null);

            // Remember how we'll process this pause/resume situation, and ensure
            // that the state is reset however we wind up proceeding.
            final boolean userLeaving = mUserLeaving;
            mUserLeaving = false;

            if (next == null) {
                ......
            }

            next.delayedResume = false;

            // If the top activity is the resumed one, nothing to do.
            if (mResumedActivity == next && next.state == ActivityState.RESUMED) {
                ......
            }

            // If we are sleeping, and there is no resumed activity, and the top
            // activity is paused, well that is the state we want.
            if ((mService.mSleeping || mService.mShuttingDown)
                && mLastPausedActivity == next && next.state == ActivityState.PAUSED) {
                ......
            }

            ......

            // If we are currently pausing an activity, then don't do anything
            // until that is done.
            if (mPausingActivity != null) {
                ......
            }

            ......

            // We need to start pausing the current activity so the top one
            // can be resumed...
            if (mResumedActivity != null) {
                ......
                startPausingLocked(userLeaving, false);
                //这里没有处于Pausing状态的Activity，即mPausingActivity为null，而且mResumedActivity也不为null，于是就调用startPausingLocked函数把Launcher推入Paused状态去了。
                return true;
            }

            ......
        }


        private final void startPausingLocked(boolean userLeaving, boolean uiSleeping) {
            if (mPausingActivity != null) {
                ......
            }
            ActivityRecord prev = mResumedActivity;
            if (prev == null) {
                ......
            }
            ......
            mResumedActivity = null;
            mPausingActivity = prev;
            mLastPausedActivity = prev;
            prev.state = ActivityState.PAUSING;
            ......

            if (prev.app != null && prev.app.thread != null) {
                ......
                try {
                    ......
                    prev.app.thread.schedulePauseActivity(prev, prev.finishing, userLeaving,
                        prev.configChangeFlags);
                    ......
                } catch (Exception e) {
                    ......
                }
            } else {
                ......
            }
        }
    }
```

## frameworks/base/core/java/android/app/ApplicationThreadNative.java
```java
    class ApplicationThreadProxy implements IApplicationThread {

        ......

        public final void schedulePauseActivity(IBinder token, boolean finished,
        boolean userLeaving, int configChanges) throws RemoteException {
            Parcel data = Parcel.obtain();
            data.writeInterfaceToken(IApplicationThread.descriptor);
            data.writeStrongBinder(token);
            data.writeInt(finished ? 1 : 0);
            data.writeInt(userLeaving ? 1 :0);
            data.writeInt(configChanges);
            mRemote.transact(SCHEDULE_PAUSE_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
            data.recycle();
        }

        ......

    }
```
这个函数通过Binder进程间通信机制进入到ApplicationThread.schedulePauseActivity函数中。

## frameworks/base/core/java/android/app/ActivityThread.java
```java
    public final class ActivityThread {
        ......
        private final class ApplicationThread extends ApplicationThreadNative {
            ......
            public final void schedulePauseActivity(IBinder token, boolean finished,
                    boolean userLeaving, int configChanges) {
                queueOrSendMessage(
                    finished ? H.PAUSE_ACTIVITY_FINISHING : H.PAUSE_ACTIVITY,
                    token,
                    (userLeaving ? 1 : 0),
                    configChanges);
            }
            ......
        }
        ......


        public void handleMessage(Message msg) {
            ......
            switch (msg.what) {
            ......
            case PAUSE_ACTIVITY:
                handlePauseActivity((IBinder)msg.obj, false, msg.arg1 != 0, msg.arg2);
                maybeSnapshot();
                break;
            ......
            }


        private final void handlePauseActivity(IBinder token, boolean finished,
                boolean userLeaving, int configChanges) {

            ActivityClientRecord r = mActivities.get(token);
            if (r != null) {
                //Slog.v(TAG, "userLeaving=" + userLeaving + " handling pause of " + r);
                if (userLeaving) {
                    performUserLeavingActivity(r);
                }

                r.activity.mConfigChangeFlags |= configChanges;
                Bundle state = performPauseActivity(token, finished, true);

                // Make sure any pending writes are now committed.
                QueuedWork.waitToFinish();

                // Tell the activity manager we have paused.
                try {
                    ActivityManagerNative.getDefault().activityPaused(token, state);
                } catch (RemoteException ex) {
                }
            }
            //函数首先将Binder引用token转换成ActivityRecord的远程接口ActivityClientRecord，然后做了三个事情：1. 如果userLeaving为true，则通过调用performUserLeavingActivity函数来调用Activity.onUserLeaveHint通知Activity，用户要离开它了；2. 调用performPauseActivity函数来调用Activity.onPause函数，我们知道，在Activity的生命周期中，当它要让位于其它的Activity时，系统就会调用它的onPause函数；3. 它通知ActivityManagerService，这个Activity已经进入Paused状态了，ActivityManagerService现在可以完成未竟的事情，即启动MainActivity了。
        }
    }
```

## frameworks/base/core/java/android/app/ActivityManagerNative.java
```java
    class ActivityManagerProxy implements IActivityManager
    {
        ......

        public void activityPaused(IBinder token, Bundle state) throws RemoteException
        {
            Parcel data = Parcel.obtain();
            Parcel reply = Parcel.obtain();
            data.writeInterfaceToken(IActivityManager.descriptor);
            data.writeStrongBinder(token);
            data.writeBundle(state);
            mRemote.transact(ACTIVITY_PAUSED_TRANSACTION, data, reply, 0);
            reply.readException();
            data.recycle();
            reply.recycle();
        }

        ......

    }
```

## frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
```java
    public final class ActivityManagerService extends ActivityManagerNative
                implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
        ......

        public final void activityPaused(IBinder token, Bundle icicle) {

            ......

            final long origId = Binder.clearCallingIdentity();
            mMainStack.activityPaused(token, icicle, false);

            ......
        }

        ......

    }
```

## frameworks/base/services/java/com/android/server/am/ActivityStack.java
```java
    public class ActivityStack {

        ......

        final void activityPaused(IBinder token, Bundle icicle, boolean timeout) {

            ......

            ActivityRecord r = null;

            synchronized (mService) {
                int index = indexOfTokenLocked(token);
                if (index >= 0) {
                    r = (ActivityRecord)mHistory.get(index);
                    if (!timeout) {
                        r.icicle = icicle;
                        r.haveState = true;
                    }
                    mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
                    if (mPausingActivity == r) {
                        r.state = ActivityState.PAUSED;
                        completePauseLocked();
                    } else {
                        ......
                    }
                }
            }
        }

        ......


    private final void completePauseLocked() {
            ActivityRecord prev = mPausingActivity;

            ......

            if (prev != null) {

                ......

                mPausingActivity = null;  //把mPausingActivity变量清空
            }

            if (!mService.mSleeping && !mService.mShuttingDown) {
                resumeTopActivityLocked(prev);
            } else {
                ......
            }

            ......
        }


    final boolean resumeTopActivityLocked(ActivityRecord prev) {
            ......

            // Find the first activity that is not finishing.
            ActivityRecord next = topRunningActivityLocked(null);

            // Remember how we'll process this pause/resume situation, and ensure
            // that the state is reset however we wind up proceeding.
            final boolean userLeaving = mUserLeaving;
            mUserLeaving = false;

            ......

            next.delayedResume = false;

            // If the top activity is the resumed one, nothing to do.
            if (mResumedActivity == next && next.state == ActivityState.RESUMED) {
                ......
                return false;
            }

            // If we are sleeping, and there is no resumed activity, and the top
            // activity is paused, well that is the state we want.
            if ((mService.mSleeping || mService.mShuttingDown)
                && mLastPausedActivity == next && next.state == ActivityState.PAUSED) {
                ......
                return false;
            }

            .......


            // We need to start pausing the current activity so the top one
            // can be resumed...
            if (mResumedActivity != null) {
                ......
                return true;
            }

            ......


            if (next.app != null && next.app.thread != null) {
                ......

            } else {
                ......
                startSpecificActivityLocked(next, true, true);
            }

            return true;
        }

        /*当前在堆栈顶端的Activity为我们即将要启动的MainActivity，这里通过调用topRunningActivityLocked将它取回来，保存在next变量中。之前最后一个Resumed状态的Activity，即Launcher，到了这里已经处于Paused状态了，因此，mResumedActivity为null。最后一个处于Paused状态的Activity为Launcher，因此，这里的mLastPausedActivity就为Launcher。前面我们为MainActivity创建了ActivityRecord后，它的app域一直保持为null。有了这些信息后，上面这段代码就容易理解了，它最终调用startSpecificActivityLocked进行下一步操作。*/


        private final void startSpecificActivityLocked(ActivityRecord r,
                boolean andResume, boolean checkConfig) {
            // Is this activity's application already running?
            ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid);
                //取回来的app为null。在Activity应用程序中的AndroidManifest.xml配置文件中，我们没有指定Application标签的process属性，系统就会默认使用package的名称，这里就是"shy.luo.activity"了。每一个应用程序都有自己的uid，因此，这里uid + process的组合就可以为每一个应用程序创建一个ProcessRecord。当然，我们可以配置两个应用程序具有相同的uid和package，或者在AndroidManifest.xml配置文件的application标签或者activity标签中显式指定相同的process属性值，这样，不同的应用程序也可以在同一个进程中启动。

            ......

            if (app != null && app.thread != null) {
                try {
                    realStartActivityLocked(r, app, andResume, checkConfig);
                    return;
                } catch (RemoteException e) {
                    ......
                }
            }

            mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false);
        }

    }
```

## frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
```java
    public final class ActivityManagerService extends ActivityManagerNative
            implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

        ......

        final ProcessRecord startProcessLocked(String processName,
                ApplicationInfo info, boolean knownToBeDead, int intentFlags,
                String hostingType, ComponentName hostingName, boolean allowWhileBooting) {

            ProcessRecord app = getProcessRecordLocked(processName, info.uid);

            ......

            String hostingNameStr = hostingName != null
                ? hostingName.flattenToShortString() : null;

            ......

            if (app == null) {
                app = new ProcessRecordLocked(null, info, processName);
                mProcessNames.put(processName, info.uid, app);
            } else {
                // If this is a new package in the process, add the package to the list
                app.addPackage(info.packageName);
            }

            ......

            startProcessLocked(app, hostingType, hostingNameStr);
            return (app.pid != 0) ? app : null;
        }

        ......

    private final void startProcessLocked(ProcessRecord app,
                    String hostingType, String hostingNameStr) {

            ......

            try {
                int uid = app.info.uid;
                int[] gids = null;
                try {
                    gids = mContext.getPackageManager().getPackageGids(
                        app.info.packageName);
                } catch (PackageManager.NameNotFoundException e) {
                    ......
                }

                ......

                int debugFlags = 0;

                ......

                int pid = Process.start("android.app.ActivityThread",
                    mSimpleProcessManagement ? app.processName : null, uid, uid,
                    gids, debugFlags, null);

                ......

            } catch (RuntimeException e) {

                ......

            }
        }

    }
```

## frameworks/base/core/java/android/app/ActivityThread.java
```java
    public final class ActivityThread {

        ......

        private final void attach(boolean system) {
            ......

            mSystemThread = system;
            if (!system) {

                ......

                IActivityManager mgr = ActivityManagerNative.getDefault();
                try {
                    mgr.attachApplication(mAppThread);
                } catch (RemoteException ex) {
                }
            } else {

                ......

            }
        }

        ......

        public static final void main(String[] args) {

            .......

            ActivityThread thread = new ActivityThread();
            thread.attach(false);

            ......

            Looper.loop();

            .......

            thread.detach();

            ......
        }
    }
```

## frameworks/base/core/java/android/app/ActivityManagerNative.java
```java
    class ActivityManagerProxy implements IActivityManager
    {
        ......

        public void attachApplication(IApplicationThread app) throws RemoteException
        {
            Parcel data = Parcel.obtain();
            Parcel reply = Parcel.obtain();
            data.writeInterfaceToken(IActivityManager.descriptor);
            data.writeStrongBinder(app.asBinder());
            mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
            reply.readException();
            data.recycle();
            reply.recycle();
        }

        ......

    }
```

## frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
```java
    public final class ActivityManagerService extends ActivityManagerNative
            implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

        ......

        public final void attachApplication(IApplicationThread thread) {
            synchronized (this) {
                int callingPid = Binder.getCallingPid();
                final long origId = Binder.clearCallingIdentity();
                attachApplicationLocked(thread, callingPid);
                Binder.restoreCallingIdentity(origId);
            }
        }

        ......



    private final boolean attachApplicationLocked(IApplicationThread thread,
                int pid) {
            // Find the application record that is being attached...  either via
            // the pid if we are running in multiple processes, or just pull the
            // next app record if we are emulating process with anonymous threads.
            ProcessRecord app;
            if (pid != MY_PID && pid >= 0) {
                synchronized (mPidsSelfLocked) {
                    app = mPidsSelfLocked.get(pid);
                }
            } else if (mStartingProcesses.size() > 0) {
                ......
            } else {
                ......
            }

            if (app == null) {
                ......
                return false;
            }

            ......

            String processName = app.processName;
            try {
                thread.asBinder().linkToDeath(new AppDeathRecipient(
                    app, pid, thread), 0);
            } catch (RemoteException e) {
                ......
                return false;
            }

            ......

            app.thread = thread;
            app.curAdj = app.setAdj = -100;
            app.curSchedGroup = Process.THREAD_GROUP_DEFAULT;
            app.setSchedGroup = Process.THREAD_GROUP_BG_NONINTERACTIVE;
            app.forcingToForeground = null;
            app.foregroundServices = false;
            app.debugging = false;

            ......

            boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);

            ......

            boolean badApp = false;
            boolean didSomething = false;

            // See if the top visible activity is waiting to run in this process...
            ActivityRecord hr = mMainStack.topRunningActivityLocked(null);
            if (hr != null && normalMode) {
                if (hr.app == null && app.info.uid == hr.info.applicationInfo.uid
                    && processName.equals(hr.processName)) {
                        try {
                            if (mMainStack.realStartActivityLocked(hr, app, true, true)) {
                                didSomething = true;
                            }
                        } catch (Exception e) {
                            ......
                        }
                } else {
                    ......
                }
            }

            ......

            return true;
        }
    }
```
