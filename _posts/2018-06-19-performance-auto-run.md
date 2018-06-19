---
layout: article 
title: Android性能优化 --- 自启动管理
key: 201806013
tags:
  - 性能优化
  - 自启动
lang: zh-Hans
---


# Android性能优化 ---（6）自启动管理
-----from: [link](https://blog.csdn.net/zhangbijun1230/article/details/79590098)

## 自启动管理简介

Android手机上安装的很多应用都会自启动，占用资源越来越多，造成系统卡顿等现象。良好的自启动管理方案管理后台自启动和开机自启动，这样就可以节约内存、优化系统流畅性等。

## 自启动管理流程分析
自启动管理的实现贯穿了应用APK（AutoRun.apk）以及framework的ActivityManagerService等。实现流程比较复杂，下面分阶段地介绍整个流程。

## 初始化
手机开机后，会发送开机广播：android.intent.action.BOOT_COMPLETED，凡是注册该广播的应用都可以接收到开机广播。（这里设计的AutoRun.apk会有一个为之提供服务apk，取名为AutoRunCore.apk）服务AutoRunCore.apk注册开机广播，接收到开机广播后，发送自定义广播（android.intent.action.security.BOOT_COMPLETED）给AutoRun.apk。

AutoRun.apk的自定义类BootBroadcastReceiver接收到AutoRunCore.apk发过来的广播后，开启线程设置禁止自启动列表和允许自启动列表；从R.array.security_boot_run_applist数组中获取允许自启动列表，定义为白名单，白名单中保存的均是非系统应用；从R.array.security_boot_forbidrun_applist数组中获取禁止自启动的列表，定义为黑名单，黑名单中保存的均是系统应用。最终调用系统接口将禁止自启动的应用（包括黑名单中的系统应用、不在白名单中的非系统应用）全部写到/data/system/forbidden_autorun_packages.xml文件中。

关键代码如下：

### 1、注册广播接收器
```xml
    <receiver
        android:name="com.android.BootBroadcastReceiver"
        android:exported="true"
        android:priority="1000"
        android:process="@string/process_security" >
        <intent-filter>
            <action android:name="android.intent.action.BOOT_COMPLETED" />
            <action android:name="android.intent.action.security.BOOT_COMPLETED" />
        </intent-filter>
    </receiver>
```

### 2、接收到广播后处理方法

```java
        @Override
        public void onReceive(final Context context, Intent intent) {
            action = intent.getAction();
            if((action != null)&&(action.equals("android.intent.action.security.BOOT_COMPLETED"))){
                action = ACTION_BOOT_COMPLETED;
            }
            ......
                    if(BOOT_AUTO_RUN && ACTION_BOOT_COMPLETED.equals(action)) {
                        Log.d("forbid auto run");
                        new Thread(new Runnable() {
                            @Override
                            public void run() {
                                // TODO Auto-generated method stub
                                SharedPreferences preferences = context.getSharedPreferences(
                                        "forbidrun_appList", Context.MODE_PRIVATE);
                                boolean isInit = preferences.getBoolean("initflag", false);
                                long lastModif = preferences.getLong("fileModif", 0);
                                long fileModif = 0;
                                File config = new File("data/system/seccenter/"
                                        + APPAUTORUN_CONFIG_FILE);//APPAUTORUN_CONFIG_FILE = "seccenter_appautorun_applist.xml"
                                pm = context.getPackageManager();
                                List<String> autorun_appList = new ArrayList<String>();
                                if (config.exists()) {
                                    fileModif = config.lastModified();
                                }
                                if (!isInit || (fileModif > lastModif)) {
                                    if (config.exists()) {
                                        try {
                                            autorun_appList = <span style="color:#009900;">parseXML</span>(config);
                                        } catch (Exception e) {
                                            // TODO Auto-generated catch block
                                            e.printStackTrace();
                                        }
                                    } else {
                                        autorun_appList = Arrays.asList(context.getResources()
                                                .getStringArray(R.array.security_boot_run_applist));
                                    }

                                    List<String> forbidrun_appList = Arrays.asList(context.getResources()
                                            .getStringArray(R.array.security_boot_forbidrun_applist));
                                    List<ApplicationInfo> allAppInfo = pm.getInstalledApplications(0);
                                    for (ApplicationInfo appInfo : allAppInfo) {
                                        if (!Util.<span style="color:#009900;">isSystemApp</span>(appInfo)
                                                && !autorun_appList.contains(appInfo.packageName)) {
                                            SystemApiUtil.<span style="color:#009900;">fobidAutoRun</span>(context,appInfo.packageName, true);
                                        } else if (Util.isSystemApp(appInfo)
                                                && forbidrun_appList.contains(appInfo.packageName)) {
                                            SystemApiUtil.fobidAutoRun(context,appInfo.packageName, true);
                                        }
                                    }
                                    SharedPreferences preference = context.getSharedPreferences("forbidrun_appList",
                                                    Context.MODE_PRIVATE);
                                    SharedPreferences.Editor editor = preference.edit();
                                    editor.putBoolean("initflag", true);
                                    editor.putLong("fileModif", fileModif);
                                    editor.commit();
                                }
                            }
                        }, "btReForbidRun").start();
    }
```

在上面的处理方法中使用的几个封装的方法，下面逐一看下。先看parseXML()方法，

```java
    public List<String> parseXML(File xmlFile) throws Exception {
        List<String> appList = new ArrayList<String>();
        DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
        DocumentBuilder db = dbf.newDocumentBuilder();

        Document doc = db.parse(xmlFile);
        NodeList nodeList = doc.getElementsByTagName("appname");
        Node fatherNode = nodeList.item(0);

        NodeList childNodes = fatherNode.getChildNodes();
        for (int j = 0; j < childNodes.getLength(); j++) {
            Node childNode = childNodes.item(j);
            if (childNode instanceof Element) {
                appList.add(childNode.getFirstChild().getNodeValue());
            }
        }
        return appList;
    }
```

接着看下Util.isSystemApp()方法，

```java
    public static boolean isSystemApp(ApplicationInfo appInfo) {
        boolean flag = false;

        if (appInfo != null
                && ((appInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0 || (appInfo.flags & ApplicationInfo.FLAG_UPDATED_SYSTEM_APP) != 0)) {
            flag = true;
        }
        return flag;
    }
```

再接着看下forbitAutorun()方法，这里是通过反射方式调用ActivityManager中的方法。

```java
    private static Method mSetStopAutoStart = null;
    private static Method mgetStopAutoStart = null;

    ......

    public static void fobidAutoRun(Context context, String pkg, boolean isFobid) {

            if (isForceStopAutoStartMethodExist(context)) {
                try {
                    ActivityManager am = (ActivityManager) context
                            .getSystemService(Context.ACTIVITY_SERVICE);
                    mSetStopAutoStart.invoke(am, pkg,
                            isFobid);
                } catch (IllegalArgumentException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                }
            }
        }
        public static boolean isForceStopAutoStartMethodExist(Context context) {
            synchronized (SystemApiUtil.class) {
                if (mSetStopAutoStart == null) {
                    try {
                        ActivityManager am = (ActivityManager) context
                                .getSystemService(Context.ACTIVITY_SERVICE);
                            mSetStopAutoStart = am.getClass().getMethod(
                                    "setForbiddenAutorunPackages", String.class,
                                    boolean.class);

                        mgetStopAutoStart = am.getClass()
                                .getMethod("getForbiddenAutorunPackages");
                    } catch (SecurityException e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    } catch (NoSuchMethodException e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    }
                }
                if (mSetStopAutoStart == null || mgetStopAutoStart == null) {
                    return false;
                } else {
                    return true;
                }
            }
        }
```

总结：在AutoRun接收到广播后，根据数组security_boot_run_applist（允许自启动白名单）和数组security_boot_forbidrun_applist（禁止自启动黑名单）设置好了禁止自启动的xml文件（forbidden_autorun_packages.xml）。

流程图如下：

## AutoRun界面初始化
AutoRun在启动之前就已经设置好了黑白名单，界面初始化过程就是界面加载的过程。

- （1）、AutoRun通过调用ActivityManagerService封装的getForbiddenAutoRunPackages()方法获取禁止自启动的应用列表。该方法从/data/system/forbidden_autorun_packages.xml文件中读取应用，保存到mPackagesForbiddenAutoRun全局变量中，同时也把应用包含的服务加入mServicesForbiddenAutoRun中。

- （2）、security_autorun_notdisplayed_whitelist 数组表示允许自启动不显示白名单，即不显示在自启动管理界面上的应用白名单。

如果这个白名单中的应用包含getForbiddenAutoRunPackages方法返回的数组中的应用，该应用取消禁止自启动（允许自启动），并且不会显示在自启动管理界面。

取消禁止自启动的处理方法就是通过ActivityManagerService封装的setForbiddenAutoRunPackages()方法实现的。

注意：

- 1、第三方应用默认禁止自启动，会设置一个自启动的白名单（R.array.security_boot_run_applist），白名单里的应用就会允许自启动。

- 2、系统应用默认允许自启动，会设置一个禁止自启动的黑名单(R.array.security_boot_forbidrun_applist)，黑名单里的应用会被禁止自启动。

具体代码如下：

```java
    public void onResume() {
        super.onResume();

        checkedItemCount = 0;//设置选中的个数
        refreshList();
    }


    private void refreshList() {
        ......
        new Thread(new InitViewRunable()).start();
    }


    class InitViewRunable implements Runnable {

        @Override
        public void run() {

            initData();
            Message.obtain(mHandler, MSG_INIT_DATA_COMPLETED).sendToTarget();

        }
    }


    private synchronized void initData() {
        applicationList.clear();//清空列表
        forbidAutoRunList = speedLogic.<span style="color:#009900;">getForbidAutoRunList</span>(true);//获取禁止自启动应用列表

        SharedPreferences preferences = mContext.getSharedPreferences(
                "forbidrun_appList", Context.MODE_PRIVATE);
        boolean isInit = preferences.getBoolean("isInit", false);
        // remove the package from the forbidAutoRunList whith need to autorun
        if (!isInit) {//第一次初始化
                if (isAdded() && forbidAutoRunList != null) {
                    List<String> autorun_whiteList = Arrays.asList(mContext
                            .getResources().getStringArray(
                                R.array.<span style="color:#009900;">security_autorun_applist</span>));//从数组中读取允许自启动的列表
                    for (String string : autorun_whiteList) {//security_autorun_applist数组中包含禁止自启动的应用，设置为允许自启动
                        if (forbidAutoRunList.contains(string)) {
                            forbidAutoRunList.remove(string);
                            SystemApiUtil.fobidAutoRun(mContext, string, false);//更新禁止自启动数据，重新设置下xml文件
                        }
                    }
                }
            SharedPreferences.Editor editor = preferences.edit();
            editor.putBoolean("isInit", true);
            editor.commit();
        }
        canAutoRunMap = speedLogic.<span style="color:#009900;">getCanAutoRunList</span>(false);//获得允许自启动应用列表，返回已封装的对象
        if (canAutoRunMap.size() != 0) {
            Map<String, Boolean> tmpAutoRunMap = new HashMap<String, Boolean>();
            tmpAutoRunMap.putAll(canAutoRunMap);
            Iterator iter = tmpAutoRunMap.entrySet().iterator();//将canAutoRunMap保存到临时Map对象中，并迭代

            ApplicationInfo appInfo;
            while (iter.hasNext()) {
                Map.Entry entry = (Map.Entry) iter.next();
                String pkgName = (String) entry.getKey();//取出包名
                ListBean bean = new ListBean();
                bean.setPkgName(pkgName);//设置ListBean对象的包名
                try {
                    appInfo = pm.getApplicationInfo(pkgName, 0);//根据包名获取对应的ApplicationInfo对象

                    String packageName = appInfo.packageName;
                    Bitmap icon = Util.<span style="color:#009900;">getIcon</span>(mContext, packageName, null);//获取图标
                    if (isAdded()) {//设置ListBean对象的icon
                        if (null == icon) {
                            bean.setListIcon(appInfo.loadIcon(pm));
                        } else {
                            bean.setListIcon(new BitmapDrawable(icon));
                        }
                    }

                    bean.setListTitle((String) appInfo.loadLabel(pm));
                    bean.setBoot((Boolean) entry.getValue());//设置是否禁止自启动
                    if (forbidAutoRunList != null
                            && forbidAutoRunList.contains(pkgName)) {
                        bean.setChecked(true);//如果包含在禁止自启动名单里，设置勾选状态
                    }
                    if (type == SYSTEM && Util.isSystemApp(appInfo)) {
                        if (bean.isChecked) {
                            checkedItemCount++;
                        }
                        applicationList.add(bean);
                        Log.d("forbidAutoRunList of SystemApp" + pkgName);
                    } else if (type == PERSONAL && !Util.isSystemApp(appInfo)) {
                        if (bean.isChecked) {
                            checkedItemCount++;
                        }
                        applicationList.add(bean);
                        Log.d("forbidAutoRunList of PersonalApp" + pkgName);
                    }

                } catch (NameNotFoundException e) {
                    e.printStackTrace();
                }
            }
        }
    }
```

initData()完毕后，发送消息更新界面。在initData()方法中用到了几个封装的方法，分别看下。

```java
    public List<String> getForbidAutoRunList(boolean forceFresh) {
        if (forbidAutoRunList == null || forceFresh) {
            forbidAutoRunList = SystemApiUtil.getForbiddenAutorunPackages(mContext);//调用系统接口
            if (forbidAutoRunList != null) {
                List<String> autorun_whiteList = Arrays.asList(mContext.getResources()
                        .getStringArray(R.array.security_autorun_notdisplayed_whitelist));
                for (String string : autorun_whiteList) {//如果从xml文件中获取的禁止自启动列表中，包含允许自启动不显示数组中的应用，则设置该应用允许自启动
                    if (forbidAutoRunList.contains(string)) {
                        Log.d("set autorun packageName:" + string);
                        SystemApiUtil.fobidAutoRun(mContext, string, false);//设置允许自启动，并自动更新xml文件中的应用
                    }
                }
            }
        }
        return forbidAutoRunList;
    }
```

getAutoRunList()方法获取允许自启动列表。

```java
    public Map<String, Boolean> getCanAutoRunList(boolean forceFresh) {
        SharedPreferences pref = mContext.getSharedPreferences(PRE_NAME, Context.MODE_PRIVATE);
        Editor editor = pref.edit();
        boolean hasApkInstalled = pref.getBoolean("hasApkInstalled", false);
        if (autoRunMap.size() == 0 || forceFresh || hasApkInstalled) {

            editor.putBoolean("hasApkInstalled",false);
            editor.commit();
            autoRunMap = new HashMap<String, Boolean>();
            List<String> forbidList = getForbidAutoRunList(forceFresh);//获取最新状态的禁止自启动列表

            List<ApplicationInfo> allAppInfo = null;
            try {
                allAppInfo = mPm.getInstalledApplications(0);//获取安装的第三方应用
            } catch (Exception e) {
                e.printStackTrace();
            }

            if (allAppInfo != null) {
                for (ApplicationInfo appInfo : allAppInfo) {
                    boolean isContains = false;
                    if (forbidList != null
                            && forbidList.contains(appInfo.packageName)) {
                        // That is in forbidden list
                        isContains = true;//如果在禁止自启动列表中，设置isContains为true。
                    }
                    if (<span style="color:#009900;">isExcludeForAutoBoot</span>(appInfo, isContains)) {//过滤掉launcher、白名单(security_autorun_notdisplayed_whitelist)中的应用。
                        continue;
                    }
                    if (<span style="color:#009900;">isPackageHasReceivers</span>(appInfo.packageName)) {//判断是否注册了广播接收器
                        Boolean isBoot = false;
                        if (<span style="color:#009900;">isPackageHasReceiverPermission</span>(appInfo.packageName,
                                Manifest.permission.RECEIVE_BOOT_COMPLETED)) {
                            isBoot = true;//注册了开机广播
                        }
                        autoRunMap.put(appInfo.packageName, isBoot);
                    }
                }
            }
        }
        return autoRunMap;
    }
```

我们再看下isExcludeForAutoBoot()方法，该方法排除掉launcher应用，排除掉白名单（security_autorun_notdisplayed_whitelist）中的应用。

```java
    private boolean isExcludeForAutoBoot(ApplicationInfo appInfo, Boolean isContains) {

        if (appInfo == null) {
            return true;
        }

        if (Util.isSystemApp(appInfo)) {//系统应用
            List<String> launcherPkgs = SpeedUtil.<span style="color:#009900;">getLauncherPkgs</span>(mContext);//获取launcher应用

            // not contain launcher icon ,not list it
            if (!launcherPkgs.contains(appInfo.packageName)) {
                return true;
            }//过滤掉launcher应用。
        }

        // judge whether contain in white list,if it is ,do not list it
        if (<span style="color:#009900;">isInAutoBootWhiteList</span>(appInfo)) {//这里再过滤一次是否在白名单中
            if (isContains) {//如果设置了禁止自启动，则设置为允许自启动
                Log.d("i donot think will come in.");
                SystemApiUtil.fobidAutoRun(mContext, appInfo.packageName, false);
            }
            return true;
        } else {
            return false;
        }
    }
```

再看下isPackageHasReceivers()方法，判断是否注册了广播接收器。

```java
    private boolean isPackageHasReceivers(String packageName) {

        if (mPm == null) {
            mPm = mContext.getPackageManager();
        }
        boolean hasReceivers = true;
        try {
            PackageInfo packinfo = mPm.getPackageInfo(packageName, PackageManager.GET_RECEIVERS);
            if (packinfo.receivers == null || packinfo.receivers.length <= 0) {
                hasReceivers = false;
            }
        } catch (NameNotFoundException e) {
            hasReceivers = false;
            e.printStackTrace();
        }

        return hasReceivers;
    }
```

看下封装的isPackageHasReceiverPermission()方法，判断是否有接收开机广播的权限。
```java
    public boolean isPackageHasReceiverPermission(String packageName, String permissionName) {

        boolean isReceiverFunctionEnable = true;
        if (mPm == null) {
            mPm = mContext.getPackageManager();
        }
        if (permissionName != null
                && PackageManager.PERMISSION_GRANTED != mPm.checkPermission(permissionName,
                        packageName)) {
            isReceiverFunctionEnable = false;
        }
        return isReceiverFunctionEnable;
    }
```

流程图如下：


## AutoRun处理机制
### 1、禁止自启动

禁止自启动：调用系统接口ActivityManagerService类封装的setForbiddenAutorunPackages（final StringpackageName, boolean bAdd），此时bAdd参数为true，将指定的包名添加到禁止自启动列表mPackagesForbiddenAutoRun中，将应用包含的服务添加到禁止自启动服务列表mServicesForbiddenAutoRun中，最后将禁止自启动列表中的数据重新更新到/data/system/forbidden_autorun_packages.xml文件中。

具体代码如下：

```java
    /**
     * @hide
     */
    public boolean setForbiddenAutorunPackages(final String packageName,
            boolean bAdd) {
        boolean bResult = false;

        if (Feature.FEATURE_FORBID_APP_AUTORUN) {
           // Slog.v(TAG, "ctl packagename=" + packageName + "bAdd=" + bAdd);
            synchronized (this) {
                if ((null != packageName) && (null != mPackagesForbiddenAutoRun)
                      && (null != mServicesForbiddenAutoRun)) {

                    final PackageManager pm = mContext.getPackageManager();
                    PackageInfo pi = null;

                    try {
                        pi = pm.getPackageInfo(packageName,
                        PackageManager.GET_SERVICES);
                    } catch (NameNotFoundException e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    }

                    int temp_size = mPackagesForbiddenAutoRun.size();
                    if (bAdd) {
                        if (mPackagesForbiddenAutoRun.indexOfKey(packageName.hashCode())<0) {
                            mPackagesForbiddenAutoRun.append(packageName.hashCode(),packageName);
                            bResult = temp_size!=mPackagesForbiddenAutoRun.size();
                            if (pi != null && null != pi.services) {
                                for (ServiceInfo service : pi.services) {
                                  if (0>mServicesForbiddenAutoRun.indexOfKey(service.processName.hashCode()))  {
                                      Slog.i(TAG, "ADD forbid autorun service: "+ service.processName);
                                      mServicesForbiddenAutoRun.append(service.processName.hashCode(), service.processName);
                                  }
                                }
                            }
                        }
                    } else {
                        mPackagesForbiddenAutoRun.delete(packageName.hashCode());
                        bResult = mPackagesForbiddenAutoRun.size()!=temp_size;
                            if (pi != null && pi.services != null) {
                                for (ServiceInfo service : pi.services) {
                                mServicesForbiddenAutoRun.delete(service.processName.hashCode());
                            }
                        }
                    }
                }

                writeFilterPackages();
            }
        }

        return bResult;
    }

    void writeFilterPackages(){

        if ( mFileFilter == null) {
            return ;
        }

        if (mFileFilter.exists()) {
            mFileFilter.delete();
        }

        try {
            FileOutputStream fstr = new FileOutputStream(mFileFilter);
            BufferedOutputStream str = new BufferedOutputStream(fstr);

            //XmlSerializer serializer = XmlUtils.serializerInstance();
            XmlSerializer serializer = new FastXmlSerializer();
            serializer.setOutput(str, "utf-8");
            serializer.startDocument(null, true);
            //serializer.setFeature("http://xmlpull.org/v1/doc/features.html#indent-output", true);

            serializer.startTag(null, "packages");

            int numPackages = mPackagesForbiddenAutoRun.size();

            //Slog.i(TAG, " forbid autorun numPackages := "+ numPackages);

            for( int i = 0; i < numPackages; ++i){
                serializer.startTag(null, "package");
                String apkPackageName = mPackagesForbiddenAutoRun.valueAt(i);
                //Slog.i(TAG, "forbid autorun package: "+apkPackageName);
                serializer.attribute(null, "name", apkPackageName);
                serializer.endTag(null, "package");
            }

            serializer.endTag(null, "packages");

            serializer.endDocument();

            str.flush();
            FileUtils.sync(fstr);
            str.close();

            FileUtils.setPermissions(mFileFilter.toString(),
                    FileUtils.S_IRUSR|FileUtils.S_IWUSR
                    |FileUtils.S_IRGRP|FileUtils.S_IWGRP
                    |FileUtils.S_IROTH,
                    -1, -1);

        } catch(java.io.IOException e) {
            Slog.w(TAG, "Unable to write broadcast filter, current changes will be lost at reboot", e);
        }
    }


    public List<String> getForbiddenAutorunPackages( ) {

        if (Feature.FEATURE_FORBID_APP_AUTORUN) {
            // Slog.v(TAG, "ctl packagename YLGetForbiddenAutorunPackages" );
            synchronized (this) {
                ArrayList<String> res = new ArrayList<String>();
                for (int i = 0; i < mPackagesForbiddenAutoRun.size(); i++) {
                    res.add(mPackagesForbiddenAutoRun.valueAt(i));
                }
                return res;
            }
        }
    }
```

流程图如下：


### 2、允许自启动

允许自启动：调用系统接口ActivityManagerService类封装的setForbiddenAutorunPackages( final String packageName, boolean bAdd)方法，此时flag参数为false，将设置的应用的包名从全局变量mPackagesForbiddenAutoRun中移除，包括该应用包含的服务从全局变量mServicesForbiddenAutoRun中移除，最后将mPackagesForbiddenAutoRun列表中的数据重新更新到/data/system/forbidden_autorun_packages.xml文件。

流程图如下：


### 3、动态安装

当一个应用安装时，AppInstallReceiver接收到android.intent.action.PACKAGE_ADDED广播，如果该应用不在security_autorun_displayed_whitelist（允许自启动显示白名单）及security_autorun_notdisplayed_whitelist（允许自启动不显示白名单）中，并且该应用又是第三方应用，则将该应用加入禁止自启动列表。

### 4、AMS实现机制
#### 1.开机自启动

##### （1）、开机启动过程中，会调用到ActivityManagerService的systemReady()方法，在该方法中读取/data/system/forbidden_autorun_packages.xml文件中的数据，并将其保存到全局数组变量mPackagesForbiddenAutoRun中。

具体代码如下：
```java
    if (Feature.FEATURE_FORBID_APP_AUTORUN){
            mFileFilter = new File(PATH_PACKAGES_FILTER);
            if(!mFileFilter.exists()){
                 try {
                     mFileFilter.createNewFile();
                    } catch (IOException e) {
                     Slog.d(TAG, "Failed to creat black list file!!!");
                    }
             }
            readFilterPackages();
     }
```

readFilterPackages()方法读取禁止自启动列表，完成mPackagesForbiddenAutoRun、mServicesForbiddenAutoRun数组的初始化。

```java
    private static final String PATH_PACKAGES_FILTER = "/data/system/forbidden_autorun_packages.xml";
    private boolean readFilterPackages(){
        FileInputStream str = null;

        if ( mFileFilter == null) {
            return false;
        }

        if (!mFileFilter.exists()) {
            Log.d(TAG, PATH_PACKAGES_FILTER + "does not exist" );
            return false;
        }

        try {
            str = new FileInputStream(mFileFilter);
            XmlPullParser parser = Xml.newPullParser();
            parser.setInput(str, null);
            int type;
            while ((type=parser.next()) != XmlPullParser.START_TAG
                       && type != XmlPullParser.END_DOCUMENT) {
                ;
            }

            if (type != XmlPullParser.START_TAG) {
                Log.d(TAG, "No start tag found in in" + PATH_PACKAGES_FILTER  );
                return false;
            }

            int outerDepth = parser.getDepth();
            while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
                   && (type != XmlPullParser.END_TAG
                           || parser.getDepth() > outerDepth)) {
                if (type == XmlPullParser.END_TAG
                        || type == XmlPullParser.TEXT) {
                    continue;
                }
                String tagName = parser.getName();
                if (tagName.equals("package")) {
                   String delPackageName = parser.getAttributeValue(null,"name");

                   final PackageManager pm = mContext.getPackageManager();
                   PackageInfo pi = null;

                   try {
                       pi = pm.getPackageInfo(delPackageName, PackageManager.GET_SERVICES);
                   } catch (NameNotFoundException e) {
                       // TODO Auto-generated catch block
                       e.printStackTrace();
                   }

                   Slog.i(TAG, "forbid autorun pakacge: "+ delPackageName);
                   mPackagesForbiddenAutoRun.append(delPackageName.hashCode(),delPackageName);

                   if (pi != null && null != pi.services) {
                       for (ServiceInfo service : pi.services) {
                           if (0>mServicesForbiddenAutoRun.indexOfKey(service.processName.hashCode())) {
                              Slog.i(TAG, "forbid autorun service: "+ service.processName);
                              mServicesForbiddenAutoRun.append(service.processName.hashCode(),service.processName);
                           }
                       }
                   }

                }else {
                    Slog.w(TAG, "Unknown element under <packages>: "
                          + parser.getName());
                    XmlUtils.skipCurrentTag(parser);
                }
            }
            str.close();
        } catch(XmlPullParserException e) {
            Slog.e(TAG, "Error reading "+ PATH_PACKAGES_FILTER, e);

        } catch(java.io.IOException e) {
            Slog.e(TAG, "Error reading "+ PATH_PACKAGES_FILTER, e);
        }
        return true;
    }
```

#### （2）、系统在启动过程中会拉起一些重要的应用，而大多数应用是在启动完成之后拉起的。这里解释下mProcessesOnHold，这是一个数组列表，保存ProcessRecord对象，表示暂时挂起的进程列表，这些进程因尝试在系统启动（systemReady）完成之前启动，而被暂时挂起，当系统启动完成之后，会启动该列表中的进程。

我们看下源码解释：

```c++
    /**
     * List of records for processes that someone had tried to start before the
     * system was ready.  We don't start them at that point, but ensure they
     * are started by the time booting is complete.
     */
    final ArrayList<ProcessRecord> mProcessesOnHold = new ArrayList<ProcessRecord>();
```

这部分在启动过程中不能拉起的应用就暂存在mProcessesOnHold列表中。因此，自启动管理在系统启动完成之前将mProcessesOnHold列表中包含的mPackagesForbiddenAutoRun中的应用排除，这样在系统启动完成后，就不会启动mPackagesForbiddenAutoRun中的应用。

在startProcessLocked方法中会调用Process.start方法开启线程，我们要做的就是从mProcessesOnHold中排除禁止自启动的应用，这样就实现开机禁止自启动了。

```java
    // If the system is not ready yet, then hold off on starting this
    // process until it is.
    if (!mProcessesReady
            && !isAllowedWhileBooting(info)
            && !allowWhileBooting
            && !isInBlackList(info)) {
        if (!mProcessesOnHold.contains(app)) {
            mProcessesOnHold.add(app);
        }
        if (DEBUG_PROCESSES) Slog.v(TAG_PROCESSES,
                "System not ready, putting on hold: " + app);
        checkTime(startTime, "startProcess: returning with proc on hold");
        return app;
    }


    public boolean isInBlackList(final ApplicationInfo info)
    {
        boolean ret = (info.flags & ApplicationInfo.FLAG_PERSISTENT) != ApplicationInfo.FLAG_PERSISTENT//revert for debugging crash
                       || (info.flags & ApplicationInfo.FLAG_SYSTEM) != ApplicationInfo.FLAG_SYSTEM;

        String packageName = info.packageName;
            ret = ((mPackagesForbiddenAutoRun.indexOfKey(packageName.hashCode())>=0 ||
                        mServicesForbiddenAutoRun.indexOfKey(packageName.hashCode())>=0) &&
                        (mPackagesHasWidgetRun.indexOfKey(packageName.hashCode())<0));
            if(ret) Slog.v(TAG, "check isInBlackList:" + packageName + " ret=" + ret);
            return ret;
    }
```

### 2、应用后台自启动
自启动管理同样包含一个应用死掉后后台重新启动的过程。Service运行在前后台都可以，即表示可以运行在当前进程也可以运行在其他进程中，Service可以为多个App共享使用，是通过binder机制来实现的。当我kill掉一个带有服务的进程（没有调用stopService()），过一会该应用会自动重启。下面是代码的调用顺序，自下往上看一下：

    com.android.server.am.ActiveServices.bringDownServiceLocked(ActiveServices.java)
    com.android.server.am.ActiveServices.killServicesLocked(ActiveServices.java)
    com.android.server.am.ActivityManagerService.cleanUpApplicationRecordLocked(ActivityManagerService.java)
    com.android.server.am.ActivityManagerService.handleAppDiedLocked(ActivityManagerService.java)
    com.android.server.am.ActivityManagerService.appDiedLocked(ActivityManagerService.java)
    com.android.server.am.ActivityManagerService$AppDeathRecipient.binderDied(ActivityManagerService.java)

ActivityManagerService.handleAppDiedLocked方法中对于在黑名单中的应用调用PackageManagerService.setPackageStoppedState方法，将应用对应的Package设置为stop状态（如果一个应用正在灭亡，或已经灭亡，就要将其对应的package置于stop状态）。

如果服务重启，则在ActiveServices.killServicesLocked方法中调用bringDownServiceLocked方法，不允许重启，直接挂掉。


具体代码如下：
```java
    /**
     * Main function for removing an existing process from the activity manager
     * as a result of that process going away.  Clears out all connections
     * to the process.哈哈
     */
    private final void handleAppDiedLocked(ProcessRecord app,
            boolean restarting, boolean allowRestart) {
        int pid = app.pid;
        boolean kept = cleanUpApplicationRecordLocked(app, restarting, allowRestart, -1);
        if (!kept && !restarting) {
            removeLruProcessLocked(app);
            if (pid > 0) {
                ProcessList.remove(pid);
            }
        }

        if (mProfileProc == app) {
            clearProfilerLocked();
        }

        if (Feature.FEATURE_FORBID_APP_AUTORUN ) {
            if (app != null && app.info != null &&
                isInBlackList(app.info)) {
                IPackageManager pm = AppGlobals.getPackageManager();
                try {
                    pm.setPackageStoppedState(app.info.packageName, true, UserHandle.getUserId(app.uid));
                    Slog.i(TAG, "forbid restart this app that is contained in forbidden list: "  + app.processName + " and remove its alarms & jobs.");
                    mContext.sendBroadcastAsUser(new Intent(ACTION_APP_KILL).putExtra(Intent.EXTRA_PACKAGES, new String[]{ app.info.packageName }),
                          UserHandle.ALL, "android.permission.DEVICE_POWER");
                } catch (RemoteException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                } catch (IllegalArgumentException e) {
                    Slog.w(TAG, "Failed trying to unstop package "
                            + app.info.packageName + ": " + e);
                    e.printStackTrace();
                }
            }
        }

     ......
    }


    /* optimize memory */
    else if (Feature.FEATURE_FORBID_APP_AUTORUN &&
            app != null && app.info != null && !isCTSMode() &&
            mAm.isInBlackList(app.info)) {
        Slog.w(TAG, "Service crashed " + sr.crashCount
                + " times, stopping: " + sr + "forbid restart this app that is contained in forbidden list: " + app.processName);
        EventLog.writeEvent(EventLogTags.AM_DESTROY_SERVICE ,
                sr.crashCount, sr.shortName, app.pid,  app.processName);
        bringDownServiceLocked(sr);
    }


    private class QuickBootReceiver extends BroadcastReceiver {//AlarmManagerService中注册该广播接收器
        static final String ACTION_APP_KILL = "org.codeaurora.quickboot.appkilled";

        public QuickBootReceiver() {
            IntentFilter filter = new IntentFilter();
            filter.addAction(ACTION_APP_KILL);
            getContext().registerReceiver(this, filter,
                    "android.permission.DEVICE_POWER", null);
        }

        @Override
        public void onReceive(Context context, Intent intent) {
            if (Feature.FEATURE_FORBID_APP_AUTORUN) {
                Message msg = new Message();
                msg.what = ScheduleHandler.REMOVE_DEAD_APP_ALARM_REVEIVE;
                msg.obj = intent;
                mScheduleHandler.sendMessage(msg);
                return  ;
            }

            String action = intent.getAction();
            String pkgList[] = null;
            if (ACTION_APP_KILL.equals(action)) {
                pkgList = intent.getStringArrayExtra(Intent.EXTRA_PACKAGES);
                if (pkgList != null && (pkgList.length > 0)) {
                    for (String pkg : pkgList) {
                        removeLocked(pkg);
                        for (int i=mBroadcastStats.size()-1; i>=0; i--) {
                            ArrayMap<String, BroadcastStats> uidStats = mBroadcastStats.valueAt(i);
                            if (uidStats.remove(pkg) != null) {
                                if (uidStats.size() <= 0) {
                                    mBroadcastStats.removeAt(i);
                                }
                            }
                        }
                    }
                }
            }
        }
    }


    case REMOVE_DEAD_APP_ALARM_REVEIVE:
        synchronized (mLock) {
            removeDeadAppAlarmReceiveLocked((Intent)msg.obj);
        }
        break;


    void removeDeadAppAlarmReceiveLocked(Intent intent) {
        Slog.d(TAG, "Receive for remove dead app alarm: " + intent.getAction());

        String pkgList[] = null;
        pkgList = intent.getStringArrayExtra(Intent.EXTRA_PACKAGES);
        if (pkgList != null && (pkgList.length > 0)) {
            for (String pkg : pkgList) {
                removeLocked(pkg);
                for (int i=mBroadcastStats.size()-1; i>=0; i--) {
                    ArrayMap<String, BroadcastStats> uidStats = mBroadcastStats.valueAt(i);
                    if (uidStats.remove(pkg) != null) {
                        if (uidStats.size() <= 0) {
                            mBroadcastStats.removeAt(i);
                        }
                    }
                }
            }
        }
    }
```

有些应用进程起来后，会在native层产生一个守护进程，应用进程挂掉后，native层进程又会把应用层进程拉起来，因此在ActivityManagerS.forceStopPackage方法中调用SystemProperties.set("ctl.start","cleandaemon");执行清理脚本。

### 3、广播启动场景
在Android系统中，系统发出一个广播，如果应用注册了这个广播，那么都会收到这个广播。自启动管理正是采用这种广播机制来实现禁止自启动的。系统在发送广播的时候，通过判断这个广播接收者是否在禁止自启动列表中，并且该应用没有被拉起来，就不给该应用发送广播。

具体处理方法在processNextBroadcast方法中处理
```java
    /* For do not send broadcast to the APP in blacklist that not running */
    if (Feature.FEATURE_FORBID_APP_AUTORUN &&
            info != null && info.activityInfo != null &&
            info.activityInfo.packageName!="com.yulong.android.dualmmsetting"&&
            mService.isInBlackList(info.activityInfo.applicationInfo)) {
        Slog.v(TAG, "Skipping delivery of static ["
              + mQueueName + "] " + r + " for forbiden auto run");
        r.receiver = null;
        r.curFilter = null;
        r.state = BroadcastRecord.IDLE;
        scheduleBroadcastsLocked();
        return;
    }
```

### 4、AMS垃圾回收机制

打开程序或者有程序进入后台时都会执行updateOomAdjLocked()函数。可以在该方法中进行处理。


```java
    if (Feature.FEATURE_FORBID_APP_AUTORUN) {
        if (isInBlackList(app.info)) {
            app.kill("Mem less than " + ProcessList.MIN_MEM_LEVEL/(1024*1024) + "M, kill background!!", true);

        }
    }
```

