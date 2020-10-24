---
layout: article 
title: 深入理解Android卷二 第四章 深入理解PackageManagerService
key: 20180605
tags:
  - 深入理解 二
  - PackageManagerService
  - pms
lang: zh-Hans
---

# 第4章  深入理解PackageManagerService
**本章主要内容：**

详细分析PackageManagerService

本章所涉及的源代码文件名及位置：
-  SystemServer.java

frameworks/base/services/java/com/android/server/SystemServer.java

-   IPackageManager.aidl

frameworks/base/core/android/java/content/pm/IPackageManager.aidl

-  PackageManagerService.java

frameworks/base/services/java/com/android/server/pm/PackageManagerService.java

-  Settings.java

frameworks/base/services/java/com/android/server/pm/Settings.java

-  SystemUI的AndroidManifest.xml

frameworks/base/package/systemui/AndroidManifest.xml

-  PackageParser.java

frameworks/base/core/java/android/content/pm/PackageParser.java

-  commandline.c

system/core/adb/commandline.c

-  installd.c

frameworks/base/cmds/installd/installd.c

-  commands.c

frameworks/base/cmds/installd/commands.c

-  pm脚本文件

frameworks/base/cmds/pm/pm

-  Pm.java

frameworks/base/cmds/pm/src/com/android/commands/pm/Pm.java

-  DefaultContainerService.java

frameworks/base/packages/defaultcontainerservice/src/com/android/defaultcontainerservice/DefaultContainerService.java

-  UserManager.java

frameworks/base/services/java/com/android/server/pm/UserManager.java

-  UserInfo.java

frameworks/base/core/android/java/content/pm/UserInfo.java

## 4.1  概述
PackageManagerService是本书分析的第一个核心服务，也是Android系统中最常用的服务之一。它负责系统中Package的管理，应用程序的安装、卸载、信息查询等。图4-1展示了PackageManagerService及客户端的类家族。



![图4-1  PackageManagerService及客户端类家族](/images/understand2/4-1.png)

由图4-1可知：

-  IPackageManager接口类中定义了服务端和客户端通信的业务函数，还定义了内部类Stub，该类从Binder派生并实现了IPackageManager接口。

-  PackageManagerService继承自IPackageManager.Stub类，由于Stub类从Binder派生，因此PackageManagerService将作为服务端参与Binder通信。

-  Stub类中定义了一个内部类Proxy，该类有一个IBinder类型（实际类型为BinderProxy）的成员变量mRemote，根据第2章介绍的Binder系统的知识，mRemote用于和服务端PackageManagerService通信。

-  IPackageManager接口类中定义了许多业务函数，但是出于安全等方面的考虑，Android对外（即SDK）提供的只是一个子集，该子集被封装在抽象类PackageManager中。客户端一般通过Context的getPackageManager函数返回一个类型为PackageManager的对象，该对象的实际类型是PackageManager的子类ApplicationPackageManager。这种基于接口编程的方式，虽然极大降低了模块之间的耦合性，却给代码分析带来了不小的麻烦。

-  ApplicationPackageManager类继承自PackageManager类。它并没有直接参与Binder通信，而是通过mPM成员变量指向一个IPackageManager.Stub.Proxy类型的对象。

提示读者在源码中可能找不到IPackageManager.java文件。该文件在编译过程中是经aidl工具处理IPackageManager.aidl后得到，最终的文件位置在Android源码/out/target

/common/obj/JAVA_LIBRARIES/framework_intermediates/src/core/java/android/content/pm/目录中。如果读者没有整体编译过源码，也可使用aidl工具单独处理IPackageManager.aidl。

aidl工具生成的结果文件有着相似的代码结构。读者不妨看看下面这个笔者通过编译生成的IPackageManager.java文件。注意，aidl工具生成的结果文件没有格式缩进，所以看起来惨不忍睹，读者可用Eclipse中的源文件格式化命令处理它。

[-->IPackageManager.java]
```java
public interface IPackageManager extendsandroid.os.IInterface {

    //定义内部类Stub，派生自Binder，实现IPackageManager接口

    publicstatic abstract class Stub extends android.os.Binder

        implements  android.content.pm.IPackageManager {

            privatestatic final java.lang.String DESCRIPTOR =

                "android.content.pm.IPackageManager";

            publicStub() {

                this.attachInterface(this,DESCRIPTOR);

            }

            ......

                //定义Stub的内部类Proxy，实现IPackageManager接口

                privatestatic class Proxy implements

                android.content.pm.IPackageManager{

                    //通过mRemote变量和服务端交互

                    private android.os.IBinder mRemote;

                    Proxy(android.os.IBinderremote) {

                        mRemote = remote;

                    }

                    ......

                }

            ......

        }
```

接下来分析PackageManagerService，为书写方便起见，以后将其简称为PKMS。

## 4.2  初识PackageManagerService
PKMS作为系统的核心服务，由SystemServer创建，相关代码如下：

[-->SystemServer.java]
```java
......//ServerThread的run函数

/*
   4.0新增的一个功能，即设备加密（encrypting the device）,该功能由
   系统属性vold.decrypt指定。这部分功能比较复杂，本书暂不讨论。
   该功能对PKMS的影响就是通过onlyCore实现的，该变量用于判断是否只扫描系统库
   （包括APK和Jar包）
 */

StringcryptState = SystemProperties.get("vold.decrypt");

booleanonlyCore = false;

//ENCRYPTING_STATE的值为"trigger_restart_min_framework"

if(ENCRYPTING_STATE.equals(cryptState)) {

    ......

        onlyCore = true;

} else if(ENCRYPTED_STATE.equals(cryptState)) {

    ......//ENCRYPTED_STATE的值为"1"

        onlyCore = true;

}

//①调用PKMS的main函数，第二个参数用于判断是否为工厂测试，我们不讨论的这种情况，

//假定onlyCore的值为false

pm =PackageManagerService.main(context,

        factoryTest !=SystemServer.FACTORY_TEST_OFF,onlyCore);

booleanfirstBoot = false;

try {

    //判断本次是否为初次启动。当Zygote或SystemServer退出时，init会再次启动

    //它们，所以这里的FirstBoot是指开机后的第一次启动

    firstBoot = pm.isFirstBoot();

}

......

try {

    //②做dex优化。dex是Android上针对Java字节码的一种优化技术，可提高运行效率

    pm.performBootDexOpt();

}

......

try {

    pm.systemReady();//③通知系统进入就绪状态

}

......

}//run函数结束
```

以上代码中共有4个关键调用，分别是：

-  PKMS的main函数。这个函数是PKMS的核心，稍后会重点分析它。

-  isFirstBoot、performBootDexOpt和systemReady。这3个函数比较简单。学完本章后，读者可完全自行分析它们，故这里不再赘述。

首先分析PKMS的main函数，它是核心函数，此处单独用一节进行分析。

## 4.3  PKMS的main函数分析
PKMS的main函数代码如下：

[-->PackageManagerService.java]
```java
public static final IPackageManager main(Contextcontext, boolean factoryTest,

        boolean onlyCore) {

    //调用PKMS的构造函数，factoryTest和onlyCore的值均为false

    PackageManagerService m = new PackageManagerService(context,

            factoryTest, onlyCore);

    //向ServiceManager注册PKMS

    ServiceManager.addService("package", m);

    return m;

}
 ```

main函数很简单，只有短短几行代码，执行时间却较长，主要原因是PKMS在其构造函数中做了很多“重体力活”，这也是Android启动速度慢的主要原因之一。在分析该函数前，先简单介绍一下PKMS构造函数的功能。

PKMS构造函数的主要功能是，扫描Android系统中几个目标文件夹中的APK，从而建立合适的数据结构以管理诸如Package信息、四大组件信息、权限信息等各种信息。抽象地看，PKMS像一个加工厂，它解析实际的物理文件（APK文件）以生成符合自己要求的产品。例如，PKMS将解析APK包中的AndroidManifest.xml，并根据其中声明的Activity标签来创建与此对应的对象并加以保管。

PKMS的工作流程相对简单，复杂的是其中用于保存各种信息的数据结构和它们之间的关系，以及影响最终结果的策略控制（例如前面代码中的onlyCore变量，用于判断是否只扫描系统目录）。曾经阅读过PKMS的读者可能会发现，代码中大量不同的数据结构以及它们之间的关系会令人大为头疼。所以，本章除了分析PKMS的工作流程外，也将关注重要的数据结构及它们的作用。

PKMS构造函数的工作流程大体可分三个阶段：

-  扫描目标文件夹之前的准备工作。

-  扫描目标文件夹。

-  扫描之后的工作。

该函数涉及到的知识点较多，代码段也较长，因此我们将通过分段讨论的方法，集中解决相关的重点问题。

4.3.1  构造函数分析之前期准备工作
下面开始分析构造函数第一阶段的工作，先看如下所示的代码。

[-->PackageManagerService.java::构造函数]
```java
public PackageManagerService(Context context,boolean factoryTest,

        booleanonlyCore) {

    ......

        if(mSdkVersion <= 0) {

            /*

               mSdkVersion是PKMS的成员变量，定义的时候进行赋值，其值取自系统属性

               “ro.build.version.sdk”，即编译的SDK版本。如果没有定义，则APK

               就无法知道自己运行在Android哪个版本上

             */

            Slog.w(TAG, "**** ro.build.version.sdk not set!");//打印一句警告

        }

    mContext = context;

    mFactoryTest= factoryTest;//假定为false，即运行在非工厂模式下

    mOnlyCore = onlyCore;//假定为false，即运行在普通模式下

    //如果此系统是eng版，则扫描Package后，不对package做dex优化

    mNoDexOpt ="eng".equals(SystemProperties.get("ro.build.type"));

    //mMetrics用于存储与显示屏相关的一些属性，例如屏幕的宽/高尺寸，分辨率等信息

    mMetrics = new DisplayMetrics();

    //Settings是一个非常重要的类，该类用于存储系统运行过程中的一些设置，

    //下面进行详细分析        mSettings = new Settings();

    //①addSharedUserLPw是什么？马上来分析

    mSettings.addSharedUserLPw("android.uid.system",

            Process.SYSTEM_UID, ApplicationInfo.FLAG_SYSTEM);

    mSettings.addSharedUserLPw("android.uid.phone",

            MULTIPLE_APPLICATION_UIDS  //该变量的默认值是true

            ? RADIO_UID :FIRST_APPLICATION_UID,

            ApplicationInfo.FLAG_SYSTEM);

    mSettings.addSharedUserLPw("android.uid.log",

            MULTIPLE_APPLICATION_UIDS

            ? LOG_UID :FIRST_APPLICATION_UID,

            ApplicationInfo.FLAG_SYSTEM);

    mSettings.addSharedUserLPw("android.uid.nfc",

            MULTIPLE_APPLICATION_UIDS

            ? NFC_UID :FIRST_APPLICATION_UID,

            ApplicationInfo.FLAG_SYSTEM);

    ......//第一段结束
```
 

刚进入构造函数，就会遇到第一个较为复杂的数据结构Setting及它的addSharedUserLPw函数。Setting的作用是管理Android系统运行过程中的一些设置信息。到底是哪些信息呢？来看下面的分析。

#### 1.  初识Settings
先分析addSharedUserLPw函数。此处截取该函数的调用代码，如下所示：

mSettings.addSharedUserLPw("android.uid.system",//字符串

              Process.SYSTEM_UID, //系统进程使用的用户id，值为1000

              ApplicationInfo.FLAG_SYSTEM//标志系统Package

);

以此处的函数调用为例，我们为addSharedUserLPw传递了3个参数：

第一个是字符串“android.uid.system“；第二个是SYSTEM_UID，其值为1000；第三个是FLAG_SYSTEM标志，用于标识系统Package。

在进入对addSharedUserLPw函数的分析前，先介绍一下SYSTEM_UID 及相关知识。

##### （1） Android系统中UID/GID介绍
UID为用户ID的缩写，GID为用户组ID的缩写，这两个概念均与Linux系统中进程的权限管理有关。一般说来，每一个进程都会有一个对应的UID（即表示该进程属于哪个user，不同user有不同权限）。一个进程也可分属不同的用户组（每个用户组都有对应的权限）。

提示Linux的UID/GID还可细分为几种类型，此处我们仅考虑普适意义的UID/GID。

如上所述，UID/GID和进程的权限有关。在Android平台中，系统定义的UID/GID在Process.java文件中，如下所示：

[-->Process.java]
```java
//系统进程使用的UID/GID，值为1000

publicstatic final int SYSTEM_UID = 1000;

//Phone进程使用的UID/GID，值为1001

publicstatic final int PHONE_UID = 1001;

//shell进程使用的UID/GID，值为2000

publicstatic final int SHELL_UID = 2000;

//使用LOG的进程所在的组的UID/GID为1007

publicstatic final int LOG_UID = 1007;

//供WIF相关进程使用的UID/GID为1010

publicstatic final int WIFI_UID = 1010;

//mediaserver进程使用的UID/GID为1013

publicstatic final int MEDIA_UID = 1013;

//设置能读写SD卡的进程的GID为1015

publicstatic final int SDCARD_RW_GID = 1015;

//NFC相关的进程的UID/GID为1025

publicstatic final int NFC_UID = 1025;

//有权限读写内部存储的进程的GID为1023

publicstatic final int MEDIA_RW_GID = 1023;

//第一个应用Package的起始UID为10000

publicstatic final int FIRST_APPLICATION_UID = 10000;

//系统所支持的最大的应用Package的UID为99999

publicstatic final int LAST_APPLICATION_UID = 99999;

//和蓝牙相关的进程的GID为2000

publicstatic final int BLUETOOTH_GID = 2000;
```

对不同的UID/GID授予不同的权限，接下来就介绍和权限设置相关的代码。

提示读者可用adb shell（将什么？）登录到自己的手机，然后用busybox提供的ps命令查看进程的UID。

下面分析addSharedUserLPw函数，代码如下：

[-->Settings.java]
```java
SharedUserSetting addSharedUserLPw(String name,int uid, int pkgFlags) {

    /*

       注意这里的参数：name为字符串”android.uid.system”,uid为1000，pkgFlags为

       ApplicationInfo.FLAG_SYSETM(以后简写为FLAG_SYSTEM)

     */

    //mSharedUsers是一个HashMap，key为字符串，值为SharedUserSetting对象

    SharedUserSetting s = mSharedUsers.get(name);

    if(s != null) {

        if (s.userId == uid) {

            return s;

        }......

        return null;

    }

    //创建一个新的SharedUserSettings对象，并设置的userId为uid，

    //SharedUserSettings是什么？有什么作用？

    s =new SharedUserSetting(name, pkgFlags);

    s.userId = uid;

    if(addUserIdLPw(uid, s, name)) {

        mSharedUsers.put(name, s);//将name与s键值对添加到mSharedUsers中保存

        return s;

    }

    return null;

}
```

从以上代码可知，Settings中有一个mSharedUsers成员，该成员存储的是字符串与SharedUserSetting键值对，也就是说以字符串为key得到对应的SharedUserSetting对象。

那么SharedUserSettings是什么？它的目的是什么？来看一个例子。

##### （2） SharedUserSetting分析
该例子来源于SystemUI的AndroidManifest.xml，如下所示：

[-->SystemUI的AndroidManifest.xml]
```xml
<manifestxmlns:android="http://schemas.android.com/apk/res/android"

       package="com.android.systemui"

       coreApp="true"

        android:sharedUserId="android.uid.system"

       android:process="system">

 ......
 ```

在xml中，声明了一个名为android:sharedUserId的属性，其值为“android.uid.system”。sharedUserId看起来和UID有关，确实如此，它有两个作用：

-  两个或多个声明了同一种sharedUserIds的APK可共享彼此的数据，并且可运行在同一进程中。

-  更重要的是，通过声明特定的sharedUserId，该APK所在进程将被赋予指定的UID。例如，本例中的SystemUI声明了system的uid，运行SystemUI的进程就可享有system用户所对应的权限（实际上就是将该进程的uid设置为system的uid）了。

提示除了在AndroidManifest.xml中声明sharedUserId外，APK在编译时还必须使用对应的证书进行签名。例如本例的SystemUI，在其Android.mk中需要额外声明LOCAL_CERTIFICATE := platform，如此，才可获得指定的UID。

通过以上介绍，读者能知道如何组织一种数据结构来包括上面的内容。此处有三个关键点需注意：

-  XML中sharedUserId属性指定了一个字符串，它是UID的字符串描述，故对应数据结构中也应该有这样一个字符串，这样就把代码和XML中的属性联系起来了。

-  在Linux系统中，真正的UID是一个整数，所以该数据结构中必然有一个整型变量。

-  多个Package可声明同一个sharedUserId，因此该数据结构必然会保存那些声明了相同sharedUserId的Package的某些信息。

了解了上面三个关键点，再来看Android是如何设计相应数据结构的，如图4-2所示。



![图4-2  SharedUserSetting类的关系图](/images/understand2/4-2.png)

由图4-2可知：

-  Settings类定义了一个mSharedUsers成员，它是一个HashMap，以字符串（如“android.uid.system”）为Key，对应的Value是一个SharedUserSettings对象。

-  SharedUserSetting派生自GrantedPermissions类，从GrantedPermissions类的命名可知，它和权限有关。SharedUserSetting定义了一个成员变量packages，类型为HashSet，用于保存声明了相同sharedUserId的Package的权限设置信息。

-  每个Package有自己的权限设置。权限的概念由PackageSetting类表达。该类继承自PackagesettingBase，而PackageSettingBase又继承自GrantedPermissions。

-  Settings中还有两个成员，一个是mUserIds，另一个是mOtherUserIds，这两位成员的类型分别是ArrayList和SparseArray。其目的是以UID为索引，得到对应的SharedUserSettings对象。在一般情况下，以索引获取数组元素的速度，比以key获取HashMap中元素的速度要快很多。

提示 根据以上对mUserIds和mOtherUserIds的描述，可知这是典型的以空间换时间的做法。

下边来分析addUserIdLPw函数，它的功能就是将SharedUserSettings对象保存到对应的数组中，代码如下：

[-->Settings.java]
```java
private boolean addUserIdLPw(int uid, Object obj, Objectname) {

    //uid不能超出限制。Android对UID进行了分类，应用APK所在进程的UID从10000开始，

    //而系统APK所在进程小于10000

    if(uid >= PackageManagerService.FIRST_APPLICATION_UID +

            PackageManagerService.MAX_APPLICATION_UIDS){

        return false;

    }



    if(uid >= PackageManagerService.FIRST_APPLICATION_UID) {

        int N = mUserIds.size();

        //计算索引，其值是uid和FIRST_APPLICATION_UID的差

        final int index = uid - PackageManagerService.FIRST_APPLICATION_UID;

        while (index >= N) {

            mUserIds.add(null);

            N++;

        }

        ......//判断该索引位置的内容是否为空，为空才保存

            mUserIds.set(index, obj);//mUserIds保存应用Package的UID

    }else {

        ......

            mOtherUserIds.put(uid, obj);//系统Package的UID由mOtherUserIds保存

    }

    return true;

}
```

至此，对Settings的分析就告一段落了。在这次“行程”中，我们重点分析了UID/GID以及SharedUserId方面的知识，并见识好几个重要的数据结构。希望读者通过SystemUI的实例能够理解这些数据结构存在的目的。

#### 2.  XML文件扫描
下面继续分析PKMS的构造函数，代码如下：

[-->PackageMangerService.java::构造函数]
```java
......//接前一段

String separateProcesses = //该值和调试有关。一般不设置该属性

SystemProperties.get("debug.separate_processes");

if(separateProcesses != null && separateProcesses.length() > 0) {

    ......

}else {

    mDefParseFlags = 0;

    mSeparateProcesses = null;

}

//创建一个Installer对象，该对象和Native进程installd交互，以后分析installd

//时再来讨论它的作用

mInstaller = new Installer();

WindowManager wm =  //得到一个WindowManager对象

(WindowManager)context.getSystemService(Context.WINDOW_SERVICE);



Display d = wm.getDefaultDisplay();

d.getMetrics(mMetrics); //获取当前设备的显示屏信息

synchronized (mInstallLock) {

    synchronized (mPackages) {

        //创建一个ThreadHandler对象，实际就是创建一个带消息循环处理的线程，该线程

        //的工作是：程序的和卸载等。以后分析程序安装时会和它亲密接触

        mHandlerThread.start();

        //以ThreadHandler线程的消息循环(Looper对象)为参数创建一个PackageHandler，

        //可知该Handler的handleMessage函数将运行在此线程上

        mHandler = new PackageHandler(mHandlerThread.getLooper());

        File dataDir = Environment.getDataDirectory();

        // mAppDataDir指向/data/data目录

        mAppDataDir = new File(dataDir, "data");

        // mUserAppDataDir指向/data/user目录

        mUserAppDataDir = new File(dataDir, "user");

        // mDrmAppPrivateInstallDir指向/data/app-private目录

        mDrmAppPrivateInstallDir = new File(dataDir, "app-private");

        /*

           创建一个UserManager对象，目前没有什么作用，但其前途将不可限量。

           根据Google的设想，未来手机将支持多个User，每个User将安装自己的应用，

           该功能为Andorid智能手机推向企业用户打下坚实基础

         */

        mUserManager = new UserManager(mInstaller, mUserAppDataDir);

        //①从文件中读权限

        readPermissions();

        //②readLPw分析

        mRestoredSettings = mSettings.readLPw();

        long startTime = SystemClock.uptimeMillis();
```

以上代码中创建了几个对象，此处暂可不去理会它们。另外，以上代码中还调用了两个函数，分别是readPermission和Setttings的readLPw，它们有什么作用呢？下面就展开分析。

####（1） readPermissions函数分析
先来分析readPermissions函数，从其函数名可猜测到它和权限有关，代码如下：

[-->PackageManagerService.java]
```java
void readPermissions() {

    // 指向/system/etc/permission目录，该目录中存储了和设备相关的一些权限信息

    FilelibraryDir = new File(Environment.getRootDirectory(),

            "etc/permissions");

    ......

        for(File f : libraryDir.listFiles()) {

            //先处理该目录下的非platform.xml文件

            if (f.getPath().endsWith("etc/permissions/platform.xml")) {

                continue;

            }

            ......//调用readPermissionFromXml解析此XML文件

                readPermissionsFromXml(f);

        }

    finalFile permFile = new File(Environment.getRootDirectory(),

            "etc/permissions/platform.xml");

    //解析platform.xml文件，看来该文件优先级最高

    readPermissionsFromXml(permFile);

}
```

悬着的心终于放了下来！readPermissions函数不就是调用readPermissionFromXml函数解析/system/etc/permissions目录下的文件吗？这些文件似乎都是XML文件。该目录下都有哪些XML文件呢？这些XML文件中有些什么内容呢？来看一个实际的例子，如图4-3所示。



![图4-3  /system/etc/permissions目录下的内容](/images/understand2/4-3.png)

图4-3中列出的是本人G7手机上/system/etc/permissions目录下的内容。在上面的代码中，虽然最后才解析platform.xml文件， 不过此处先分析此文件其内容如下所示：

[-->platform.xml]
```xml
<permissions>

   <!--建立权限名与gid的映射关系。如下面声明的BLUTOOTH_ADMIN权限，它对应的用户组是

    net_bt_admin。注意，该文件中的permission标签只对那些需要通过读写设备（蓝牙/camera）

     /创建socket等进程划分了gid。因为这些权限涉及和Linux内核交互，所以需要在底层

     权限（由不同的用户组界定）和Android层权限（由不同的字符串界定）之间建立映射关系

  -->

 <permission name="android.permission.BLUETOOTH_ADMIN" >

       <group gid="net_bt_admin" />

 </permission>

 <permission name="android.permission.BLUETOOTH" >

       <group gid="net_bt" />

  </permission>

  ......

   <!--

     赋予对应uid相应的权限。如果下面一行表示uid为shell，那么就赋予

       它SEND_SMS的权限，其实就是把它加到对应的用户组中-->

   <assign-permission name="android.permission.SEND_SMS"uid="shell" />

   <assign-permission name="android.permission.CALL_PHONE"uid="shell" />

   <assign-permission name="android.permission.READ_CONTACTS"uid="shell" />

   <assign-permission name="android.permission.WRITE_CONTACTS"uid="shell" />

    <assign-permissionname="android.permission.READ_CALENDAR" uid="shell" />

......

    <!-- 系统提供的Java库，应用程序运行时候必须要链接这些库，该工作由系统自动完成 -->

    <libraryname="android.test.runner"

           file="/system/frameworks/android.test.runner.jar" />

    <library name="javax.obex"

           file="/system/frameworks/javax.obex.jar"/>

</permissions>
```

platform.xml文件中主要使用了如下4个标签：

-  permission和group用于建立Linux层gid和Android层pemission之间的映射关系。

-  assign-permission用于向指定的uid赋予相应的权限。这个权限由Android定义，用字符串表示。

-  library用于指定系统库。当应用程序运行时，系统会自动为这些进程加载这些库。

了解了platform.xml后，再看其他的XML文件，这里以handheld-core-hardware.xml为例进行介绍，其内容如下：

[-->handheld-core-hardware.xml]
```xml
<permissions>

   <feature name="android.hardware.camera" />

   <feature name="android.hardware.location" />

   <feature name="android.hardware.location.network" />

   <feature name="android.hardware.sensor.compass" />

   <feature name="android.hardware.sensor.accelerometer" />

   <feature name="android.hardware.bluetooth" />

   <feature name="android.hardware.touchscreen" />

   <feature name="android.hardware.microphone" />

   <feature name="android.hardware.screen.portrait" />

   <feature name="android.hardware.screen.landscape" />

   </permissions>
```

这个XML文件包含了许多feature标签。根据该文件中的注释，这些feature用来描述一个手持终端（包括手机、平板电脑等）应该支持的硬件特性，例如支持camera、支持蓝牙等。

注意对于不同的硬件特性，还需要包含其他的xml文件。例如，要支持前置摄像头，还需要包含android.hardware.camera.front.xml文件。这些文件内容大体一样，都通过feature标签表明自己的硬件特性。相关说明可参考handheld-core-hardware.xml中的注释。

有读者可能会好奇，真实设备上/system/etc/permission目录中的文件是从哪里的呢？

答案是，在编译阶段由不同硬件平台根据自己的配置信息复制相关文件到目标目录中得来的。

这里给出一个例子，如图4-4所示。



![图4-4  /system/etc/permission目录中文件的来源](/images/understand2/4-4.png)

由图4-4可知，当编译的设备目标为htc-passion时，就会将Android源码目录/frameworks/base/data/etc/下某些和该目标设备硬件特性匹配的XML文件复制到最终输出目录/system/etc/permissions下。编译完成后，将生成system镜像。把该镜像文件烧到手机中，就成了目标设备使用的情况了。

注意4.0源码中并没有htc相关的文件，这是笔者从2.3源码中移植过去的。读者可参考笔者一篇关于如何移植4.0到G7的博文，[地址为](http://blog.csdn.net/innost/article/details/6977167)。

了解了与XML相关的知识后，再来分析readPermissionFromXml函数。相信聪明的读者已经知道它的作用了，就是将XML文件中的标签以及它们之间的关系转换成代码中的相应数据结构，代码如下：

[-->PackageManagerService.java]
```java
private void readPermissionsFromXml(File permFile){

    FileReader permReader = null;

    try{

        permReader = new FileReader(permFile);

    } ......

    try{

        XmlPullParser parser = Xml.newPullParser();

        parser.setInput(permReader);

        XmlUtils.beginDocument(parser, "permissions");

        while (true) {

            ......

                String name = parser.getName();

            //解析group标签，前面介绍的XML文件中没有单独使用该标签的地方

            if ("group".equals(name)) {

                String gidStr = parser.getAttributeValue(null, "gid");

                if (gidStr != null) {

                    int gid =Integer.parseInt(gidStr);

                    //转换XML中的gid字符串为整型，并保存到mGlobalGids中

                    mGlobalGids =appendInt(mGlobalGids, gid);

                } ......

            } else if ("permission".equals(name)) {//解析permission标签

                String perm = parser.getAttributeValue(null, "name");

                ......

                    perm = perm.intern();

                //调用readPermission处理

                readPermission(parser, perm);

                //下面解析的是assign-permission标签

            } else if("assign-permission".equals(name)) {

                String perm = parser.getAttributeValue(null, "name");

                ......

                    String uidStr = parser.getAttributeValue(null, "uid");

                ......

                    //如果是assign-permission，则取出uid字符串，然后获得Linux平台上

                    //的整型uid值

                    int uid = Process.getUidForName(uidStr);

                ......

                    perm = perm.intern();

                //和assign相关的信息保存在mSystemPermissions中

                HashSet<String> perms = mSystemPermissions.get(uid);

                if (perms == null) {

                    perms = newHashSet<String>();

                    mSystemPermissions.put(uid, perms);

                }

                perms.add(perm);......

            } else if ("library".equals(name)) {//解析library标签

                String lname = parser.getAttributeValue(null, "name");

                String lfile = parser.getAttributeValue(null, "file");

                if (lname == null) {

                    ......

                } else if (lfile == null) {

                    ......

                } else {

                    //将XML中的name和library属性值存储到mSharedLibraries中

                    mSharedLibraries.put(lname,lfile);

                } ......

            } else if ("feature".equals(name)) {//解析feature标签

                String fname = parser.getAttributeValue(null, "name");

                ......{

                    //在XML中定义的feature由FeatureInfo表达

                    FeatureInfo fi = newFeatureInfo();

                    fi.name = fname;

                    //存储feature名和对应的FeatureInfo到mAvailableFeatures中

                    mAvailableFeatures.put(fname, fi);

                }......

            } ......

        } ......

    }
```

readPermissions函数果然将XML中的标签转换成对应的数据结构。总结相关的数据结构，如图4-4所示，此处借用了UML类图。在每个类图中，首行是数据结构名，第二行是数据结构的类型，第三行是注释。



![图4-4  通过readPermissions函数建立的数据结构及其关系](/images/understand2/4-4-4.png)

这里必须再次强调：图4-4中各种数据结构的目的是为了保存XML中各种标签及它们之间的关系。在分析过程中，最重要的是理解各种标签的作用，而不是它们所使用的数据结构。

##### （2） readLPw的“佐料”
readLPw函数的功能也是解析文件，不过这些文件的内容却是在PKMS正常启动后生成的。这里仅介绍作为readLPw“佐料”的文件的信息。文件的具体位置在Settings构造函数中指明，其代码如下：

[-->Settings.java]
```java
Settings() {

    FiledataDir = Environment.getDataDirectory();

    FilesystemDir = new File(dataDir, "system");//指向/data/system目录

    systemDir.mkdirs();//创建该目录

    ......

        /*
           一共有5个文件，packages.xml和packages-backup.xml为一组，用于描述系统中
           所安装的Package的信息，其中backup是临时文件。PKMS先把数据写到backup中，
           信息都写成功后再改名成非backup的文件。其目的是防止在写文件过程中出错，导致信息丢失。
           packages-stopped.xml和packages-stopped-backup.xml为一组，用于描述系统中
           强制停止运行的pakcage的信息，backup也是临时文件。如果此处存在该临时文件，表明
           此前系统因为某种原因中断了正常流程
           packages.list列出当前系统中应用级（即UID大于10000）Package的信息
         */

        mSettingsFilename = new File(systemDir, "packages.xml");
    mBackupSettingsFilename = new File(systemDir,"packages-backup.xml");
    mPackageListFilename = new File(systemDir, "packages.list");
    mStoppedPackagesFilename = new File(systemDir,"packages-stopped.xml");
    mBackupStoppedPackagesFilename = new File(systemDir,
            "packages-stopped-backup.xml");
}
```

上面5个文件共分为三组，这里简单介绍一下这些文件的来历（不考虑临时的backup文件）。

-  packages.xml： PKMS扫描完目标文件夹后会创建该文件。当系统进行程序安装、卸载和更新等操作时，均会更新该文件。该文件保存了系统中与package相关的一些信息。

-  packages.list：描述系统中存在的所有非系统自带的APK的信息。当这些程序有变动时，PKMS就会更新该文件。

-  packages-stopped.xml：从系统自带的设置程序中进入应用程序页面，然后在选择强制停止（ForceStop）某个应用时，系统会将该应用的相关信息记录到此文件中。也就是该文件保存系统中被用户强制停止的Package的信息。

readLPw的函数功能就是解析其中的XML文件的内容，然后建立并更新对应的数据结构，例如停止的package重启之后依然是stopped状态。

提示读者看完本章后，可自行分析该函数。在此之前，建议读者不必关注该函数。

#### 3.  第一阶段工作总结
在继续征程前，先总结一下PKMS构造函数在第一阶段的工作，千言万语汇成一句话：扫描并解析XML文件，将其中的信息保存到特定的数据结构中。

第一阶段扫描的XML文件与权限及上一次扫描得到的Package信息有关，它为PKMS下一阶段的工作提供了重要的参考信息。

### 4.3.2  构造函数分析之扫描Package
PKMS构造函数第二阶段的工作就是扫描系统中的APK了。由于需要逐个扫描文件，因此手机上装的程序越多，PKMS的工作量越大，系统启动速度也就越慢。

#### 1.  系统库的dex优化
接着对PKMS构造函数进行分析，代码如下：

[-->PackageManagerService.java]
```java
......

mRestoredSettings= mSettings.readLPw();//接第一段的结尾

longstartTime = SystemClock.uptimeMillis();//记录扫描开始的时间

//定义扫描参数

intscanMode = SCAN_MONITOR | SCAN_NO_PATHS | SCAN_DEFER_DEX;

if(mNoDexOpt) {

    scanMode|= SCAN_NO_DEX; //在控制扫描过程中是否对APK文件进行dex优化

}

finalHashSet<String> libFiles = new HashSet<String>();

// mFrameworkDir指向/system/frameworks目录

mFrameworkDir = newFile(Environment.getRootDirectory(),"framework");

// mDalvikCacheDir指向/data/dalvik-cache目录

mDalvikCacheDir= new File(dataDir, "dalvik-cache");

booleandidDexOpt = false;

/*

   获取Java启动类库的路径，在init.rc文件中通过BOOTCLASSPATH环境变量输出，该值如下

   /system/framework/core.jar:/system/frameworks/core-junit.jar:

   /system/frameworks/bouncycastle.jar:/system/frameworks/ext.jar:

   /system/frameworks/framework.jar:/system/frameworks/android.policy.jar:

   /system/frameworks/services.jar:/system/frameworks/apache-xml.jar:

   /system/frameworks/filterfw.jar

   该变量指明了framework所有核心库及文件位置

 */

StringbootClassPath = System.getProperty("java.boot.class.path");

if(bootClassPath != null) {

    String[] paths = splitString(bootClassPath, ':');

    for(int i=0; i<paths.length; i++) {

        try{  //判断该jar包是否需要重新做dex优化

            if (dalvik.system.DexFile.isDexOptNeeded(paths[i])) {

                /*

                   将该jar包文件路径保存到libFiles中，然后通过mInstall对象发送

                   命令给installd，让其对该jar包进行dex优化

                 */

                libFiles.add(paths[i]);

                mInstaller.dexopt(paths[i], Process.SYSTEM_UID, true);

                didDexOpt = true;

            }

        } ......

    }

} ......

/*

   读者还记得mSharedLibrarires的作用吗？它保存的是platform.xml中声明的系统库的信息。

   这里也要判断系统库是否需要做dex优化。处理方式同上

 */

if (mSharedLibraries.size() > 0) {

    ......

}

//将framework-res.apk添加到libFiles中。framework-res.apk定义了系统常用的

//资源，还有几个重要的Activity，如长按Power键后弹出的选择框

libFiles.add(mFrameworkDir.getPath() + "/framework-res.apk");

//列举/system/frameworks目录中的文件

String[] frameworkFiles = mFrameworkDir.list();

if(frameworkFiles != null) {

    ......//判断该目录下的apk或jar文件是否需要做dex优化。处理方式同上

}

/*
   上面代码对系统库（BOOTCLASSPATH指定，或 platform.xml定义，或
   /system/frameworks目录下的jar包与apk文件）进行一次仔细检查，该优化的一定要优化。
   如果发现期间对任何一个文件进行了优化，则设置didDexOpt为true
 */

if (didDexOpt) {

    String[] files = mDalvikCacheDir.list();

    if (files != null) {

        /*
           如果前面对任意一个系统库重新做过dex优化，就需要删除cache文件。原因和
           dalvik虚拟机的运行机制有关。本书暂不探讨dex及cache文件的作用。
           从删除cache文件这个操作来看，这些cache文件应该使用了dex优化后的系统库
           所以当系统库重新做dex优化后，就需要删除旧的cache文件。可简单理解为缓存失效
         */

        for (int i=0; i<files.length; i++) {
            String fn = files[i];
            if(fn.startsWith("data@app@")
                    ||fn.startsWith("data@app-private@")) {
                (newFile(mDalvikCacheDir, fn)).delete();
            }
```

#### 2.  扫描系统Package
清空cache文件后，PKMS终于进入重点段了。接下来看PKMS第二阶段工作的核心内容，即扫描Package，相关代码如下：

[-->PackageManagerService.java]
```java
//创建文件夹监控对象，监视/system/frameworks目录。利用了Linux平台的inotify机制

mFrameworkInstallObserver = new AppDirObserver(

        mFrameworkDir.getPath(),OBSERVER_EVENTS, true);

mFrameworkInstallObserver.startWatching();

/*

   调用scanDirLI函数扫描/system/frameworks目录，这个函数很重要，稍后会再分析。

   注意，在第三个参数中设置了SCAN_NO_DEX标志，因为该目录下的package在前面的流程

   中已经过判断并根据需要做过dex优化了

 */

scanDirLI(mFrameworkDir, PackageParser.PARSE_IS_SYSTEM

        | PackageParser.PARSE_IS_SYSTEM_DIR,scanMode | SCAN_NO_DEX, 0);

//创建文件夹监控对象，监视/system/app目录

mSystemAppDir = new File(Environment.getRootDirectory(),"app");

mSystemInstallObserver = new AppDirObserver(

        mSystemAppDir.getPath(), OBSERVER_EVENTS, true);

mSystemInstallObserver.startWatching();

//扫描/system/app下的package

scanDirLI(mSystemAppDir, PackageParser.PARSE_IS_SYSTEM

        | PackageParser.PARSE_IS_SYSTEM_DIR, scanMode, 0);

//监视并扫描/vendor/app目录

mVendorAppDir = new File("/vendor/app");

mVendorInstallObserver = new AppDirObserver(

        mVendorAppDir.getPath(), OBSERVER_EVENTS, true);

mVendorInstallObserver.startWatching();

//扫描/vendor/app下的package

scanDirLI(mVendorAppDir, PackageParser.PARSE_IS_SYSTEM

        | PackageParser.PARSE_IS_SYSTEM_DIR, scanMode, 0);

//和installd交互。以后单独分析installd

mInstaller.moveFiles();
```

由以上代码可知，PKMS将扫描以下几个目录。

-  /system/frameworks：该目录中的文件都是系统库，例如framework.jar、services.jar、framework-res.apk。不过scanDirLI只扫描APK文件，所以framework-res.apk是该目录中唯一“受宠”的文件。

-  /system/app：该目录下全是默认的系统应用，例如Browser.apk、SettingsProvider.apk等。

-  /vendor/app：该目录中的文件由厂商提供，即厂商特定的APK文件，不过目前市面上的厂商都把自己的应用放在/system/app目录下。

注意本书把这三个目录称为系统Package目录，以区分后面的非系统Package目录。

PKMS调用scanDirLI函数进行扫描，下面来分析此函数。

##### （1） scanDirLI函数分析
scanDirLI函数的代码如下：

[-->PackageManagerService.java]
```java
private void scanDirLI(File dir, int flags, intscanMode, long currentTime) {

    String[] files = dir.list();//列举该目录下的文件

    ......

        inti;

    for(i=0; i<files.length; i++) {
        File file = new File(dir, files[i]);
        if (!isPackageFilename(files[i])) {
            continue; //根据文件名后缀，判断是否为APK 文件。这里只扫描APK 文件
        }

        /*
           调用scanPackageLI函数扫描一个特定的文件，返回值是PackageParser的内部类
           Package，该类的实例代表一个APK文件，所以它就是和APK文件对应的数据结构
         */

        PackageParser.Package pkg = scanPackageLI(file,
                flags|PackageParser.PARSE_MUST_BE_APK, scanMode, currentTime);

        if (pkg == null && (flags &PackageParser.PARSE_IS_SYSTEM) == 0 &&
                mLastScanError ==PackageManager.INSTALL_FAILED_INVALID_APK) {
            //注意此处flags的作用，只有非系统Package扫描失败，才会删除该文件
            file.delete();
        }
    }
}
```

接着来分析scanPackageLI函数。PKMS中有两个同名的scanPackageLI函数，后面会一一见到。先来看第一个也是最先碰到的scanPackageLI函数。

##### （2） 初会scanPackageLI函数
首次相遇的scanPackageLI函数的代码如下：

[-->PackageManagerService.java]
```java
private PackageParser.Package scanPackageLI(FilescanFile, int parseFlags,
        int scanMode, long currentTime)
{

    mLastScanError = PackageManager.INSTALL_SUCCEEDED;

    StringscanPath = scanFile.getPath();

    parseFlags |= mDefParseFlags;//默认的扫描标志，正常情况下为0

    //创建一个PackageParser对象

    PackageParser pp = new PackageParser(scanPath);

    pp.setSeparateProcesses(mSeparateProcesses);// mSeparateProcesses为空

    pp.setOnlyCoreApps(mOnlyCore);// mOnlyCore为false

    /*
       调用PackageParser的parsePackage函数解析APK文件。注意，这里把代表屏幕
       信息的mMetrics对象也传了进去
     */

    finalPackageParser.Package pkg = pp.parsePackage(scanFile,
            scanPath, mMetrics, parseFlags);

    ......

        PackageSetting ps = null;
    PackageSetting updatedPkg;

    ......

        /*
           这里略去一大段代码，主要是关于Package升级方面的工作。读者可能会比较好奇：既然是
           升级，一定有新旧之分，如果这里刚解析后得到的Package信息是新，那么旧Package
           的信息从何得来？还记得”readLPw的‘佐料’”这一小节提到的package.xml文件吗？此
           文件中存储的就是上一次扫描得到的Package信息。对比这两次的信息就知道是否需要做
           升级了。这部分代码比较繁琐，但不影响我们正常分析。感兴趣的读者可自行研究
         */

        //收集签名信息，这部分内容涉及signature，本书暂不拟讨论[①]。
        if (!collectCertificatesLI(pp, ps, pkg,scanFile, parseFlags))

            returnnull;

    //判断是否需要设置PARSE_FORWARD_LOCK标志，这个标志针对资源文件和Class文件
    //不在同一个目录的情况。目前只有/vendor/app目录下的扫描会使用该标志。这里不讨论
    //这种情况。

    if (ps != null &&!ps.codePath.equals(ps.resourcePath))
        parseFlags|= PackageParser.PARSE_FORWARD_LOCK;

    String codePath = null;
    String resPath = null;

    if((parseFlags & PackageParser.PARSE_FORWARD_LOCK) != 0) {
        ......//这里不考虑PARSE_FORWARD_LOCK的情况。
    }else {
        resPath = pkg.mScanPath;
    }

    codePath = pkg.mScanPath;//mScanPath指向该APK文件所在位置
    //设置文件路径信息，codePath和resPath都指向APK文件所在位置
    setApplicationInfoPaths(pkg, codePath, resPath);
    //调用第二个scanPackageLI函数
    return scanPackageLI(pkg, parseFlags, scanMode | SCAN_UPDATE_SIGNATURE,
            currentTime);
}
```

scanPackageLI函数首先调用PackageParser对APK文件进行解析。根据前面的介绍可知，PackageParser完成了从物理文件到对应数据结构的转换。下面来分析这个PackageParser。

##### （3） PackageParser分析
PackageParser主要负责APK文件的解析，即解析APK文件中的AndroidManifest.xml代码如下：

[-->PackageParser.java]
```java
publicPackage parsePackage(File sourceFile, String destCodePath,
        DisplayMetrics metrics, int flags) {

    mParseError = PackageManager.INSTALL_SUCCEEDED;
    mArchiveSourcePath = sourceFile.getPath();

    ......//检查是否为APK文件

        XmlResourceParser parser = null;
    AssetManager assmgr = null;
    Resources res = null;
    boolean assetError = true;

    try{
        assmgr = new AssetManager();
        int cookie = assmgr.addAssetPath(mArchiveSourcePath);
        if (cookie != 0) {
            res = new Resources(assmgr, metrics, null);
            assmgr.setConfiguration(0, 0, null, 0, 0, 0, 0, 0, 0, 0, 0, 0,    0, 0, 0, 0,Build.VERSION.RESOURCES_SDK_INT);

            /*
               获得一个XML资源解析对象，该对象解析的是APK中的AndroidManifest.xml文件。
               以后再讨论AssetManager、Resource及相关的知识
             */

            parser = assmgr.openXmlResourceParser(cookie,
                    ANDROID_MANIFEST_FILENAME);

            assetError = false;
        } ......//出错处理

        String[] errorText = new String[1];
        Package pkg = null;
        Exception errorException = null;

        try {
            //调用另外一个parsePackage函数
            pkg = parsePackage(res, parser, flags, errorText);
        } ......

        ......//错误处理

            parser.close();
        assmgr.close();

        //保存文件路径，都指向APK文件所在的路径
        pkg.mPath = destCodePath;
        pkg.mScanPath = mArchiveSourcePath;
        pkg.mSignatures = null;

        return pkg;
    }
```

以上代码中调用了另一个同名的PackageParser函数，此函数内容较长，但功能单一，就是解析AndroidManifest.xml中的各种标签，这里只提取其中相关的代码：

[-->PackageParser.java]
```java
private Package parsePackage(
        Resources res, XmlResourceParser parser, int flags, String[] outError)

throws XmlPullParserException, IOException {

    AttributeSet attrs = parser;



    mParseInstrumentationArgs = null;

    mParseActivityArgs = null;

    mParseServiceArgs= null;

    mParseProviderArgs = null;

    //得到Package的名字，其实就是得到AndroidManifest.xml中package属性的值，

    //每个APK都必须定义该属性

    String pkgName = parsePackageName(parser, attrs, flags, outError);

    ......

        inttype;

    ......

        //以pkgName名字为参数，创建一个Package对象。后面的工作就是解析XML并填充

        //该Package信息

        finalPackage pkg = new Package(pkgName);

    boolean foundApp = false;

    ......//下面开始解析该文件中的标签，由于这段代码功能简单，所以这里仅列举相关函数

        while(如果解析未完成){

            ......

                StringtagName = parser.getName(); //得到标签名

            if(tagName.equals("application")){

                ......//解析application标签

                    parseApplication(pkg,res, parser, attrs, flags, outError);

            } elseif (tagName.equals("permission-group")) {

                ......//解析permission-group标签

                    parsePermissionGroup(pkg, res, parser, attrs, outError);

            } elseif (tagName.equals("permission")) {

                ......//解析permission标签

                    parsePermission(pkg, res, parser, attrs, outError);

            } else if(tagName.equals("uses-permission")){

                //从XML文件中获取uses-permission标签的属性

                sa= res.obtainAttributes(attrs,

                        com.android.internal.R.styleable.AndroidManifestUsesPermission);

                //取出属性值，也就是对应的权限使用声明

                String name = sa.getNonResourceString(com.android.internal.

                        R.styleable.AndroidManifestUsesPermission_name);

                //添加到Package的requestedPermissions数组

                if(name != null && !pkg.requestedPermissions.contains(name)) {

                    pkg.requestedPermissions.add(name.intern());

                }

            }elseif (tagName.equals("uses-configuration")){

                /*

                   该标签用于指明本package对硬件的一些设置参数，目前主要针对输入设备（触摸屏、键盘

                   等）。游戏类的应用可能对此有特殊要求。

                 */

                ConfigurationInfocPref = new ConfigurationInfo();

                ......//解析该标签所支持的各种属性

                    pkg.configPreferences.add(cPref);//保存到Package的configPreferences数组
            }
            ......//对其他标签解析和处理
        }
```

上面代码展示了AndroidManifest.xml解析的流程，其中比较重要的函数是parserApplication，它用于解析application标签及其子标签（Android的四大组件在application标签中已声明）。

图4-5表示了PackageParser及其内部重要成员的信息。



![图4-5  PackageParser大家族](/images/understand2/4-5.png)

由图4-5可知：

-  PackageParser定了相当多的内部类，这些内部类的作用就是保存对应的信息。解析AndroidManifest.xml文件得到的信息由Package保存。从该类的成员变量可看出，和Android四大组件相关的信息分别由activites、receivers、providers、services保存。由于一个APK可声明多个组件，因此activites和receivers等均声明为ArrayList。

-  以PackageParser.Activity为例，它从Component<ActivityIntentInfo>派生。Component是一个模板类，元素类型是ActivityIntentInfo，此类的顶层基类是IntentFilter。PackageParser.Activity内部有一个ActivityInfo类型的成员变量，该变量保存的就是四大组件中Activity的信息。细心的读者可能会有疑问，为什么不直接使用ActivityInfo，而是通过IntentFilter构造出一个使用模板的复杂类型PackageParser.Activity呢？原来，Package除了保存信息外，还需要支持Intent匹配查询。例如，设置Intent的Action为某个特定值，然后查找匹配该Intent的Activity。由于ActivityIntentInfo是从IntentFilter派生的，因此它它能判断自己是否满足该Intent的要求，如果满足，则返回对应的ActivityInfo。在后续章节会详细讨论根据Intent查询特定Activity的工作流程。

-  PackageParser定了一个轻量级的数据结构PackageLite，该类仅存储Package的一些简单信息。我们在介绍Package安装的时候，会遇到PackageLite。

注意读者需要了解Java泛型类的相关知识。

##### （4） 与scanPackageLI再相遇
在PackageParser扫描完一个APK后，此时系统已经根据该APK中AndroidManifest.xm，创建了一个完整的Package对象，下一步就是将该Package加入到系统中。此时调用的函数就是另外一个scanPackageLI，其代码如下：

[-->PackageManagerService.java::scanPackageLI函数]
```java
private PackageParser.PackagescanPackageLI(PackageParser.Package pkg,

        int parseFlags, int scanMode, long currentTime) {

    FilescanFile = new File(pkg.mScanPath);

    ......

        mScanningPath = scanFile;

    //设置package对象中applicationInfo的flags标签，用于标示该Package为系统

    //Package

    if((parseFlags&PackageParser.PARSE_IS_SYSTEM) != 0) {

        pkg.applicationInfo.flags |= ApplicationInfo.FLAG_SYSTEM;

    }

    //①下面这句if判断极为重要，见下面的解释

    if(pkg.packageName.equals("android")) {

        synchronized (mPackages) {

            if (mAndroidApplication != null) {

                ......

                    mPlatformPackage = pkg;

                pkg.mVersionCode = mSdkVersion;

                mAndroidApplication = pkg.applicationInfo;

                mResolveActivity.applicationInfo = mAndroidApplication;

                mResolveActivity.name = ResolverActivity.class.getName();

                mResolveActivity.packageName = mAndroidApplication.packageName;

                mResolveActivity.processName = mAndroidApplication.processName;

                mResolveActivity.launchMode = ActivityInfo.LAUNCH_MULTIPLE;

                mResolveActivity.flags = ActivityInfo.FLAG_EXCLUDE_FROM_RECENTS;

                mResolveActivity.theme =

                    com.android.internal.R.style.Theme_Holo_Dialog_Alert;

                mResolveActivity.exported = true;

                mResolveActivity.enabled = true;

                //mResoveInfo的activityInfo成员指向mResolveActivity

                mResolveInfo.activityInfo = mResolveActivity;

                mResolveInfo.priority = 0;

                mResolveInfo.preferredOrder = 0;

                mResolveInfo.match = 0;

                mResolveComponentName = new ComponentName(

                        mAndroidApplication.packageName, mResolveActivity.name);

            }

        }
```

刚进入scanPackageLI函数，我们就发现了一个极为重要的内容，即单独判断并处理packageName为“android”的Package。和该Package对应的APK是framework-res.apk，有图为证，如图4-6所示为该APK的AndroidManifest.xml中的相关内容。



![图4-6  framework-res.apk的AndroidManifest.xml](/images/understand2/4-6.png)

实际上，framework-res.apk还包含了以下几个常用的Activity。

-  ChooserActivity：当多个Activity符合某个Intent的时候，系统会弹出此Activity，由用户选择合适的应用来处理。

-  RingtonePickerActivity：铃声选择Activity。

-  ShutdownActivity：关机前弹出的选择对话框。

由前述知识可知，该Package和系统息息相关，因此它得到了PKMS的特别青睐，主要体现在以下几点。

-  mPlatformPackage成员用于保存该Package信息。

-  mAndroidApplication用于保存此Package中的ApplicationInfo。

-  mResolveActivity指向用于表示ChooserActivity信息的ActivityInfo。

-  mResolveInfo为ResolveInfo类型，它用于存储系统解析Intent（经IntentFilter的过滤）后得到的结果信息，例如满足某个Intent的Activity的信息。由前面的代码可知，mResolveInfo的activityInfo其实指向的就是mResolveActivity。

注意在从PKMS中查询满足某个Intent的Activity时，返回的就是ResolveInfo，再根据ResolveInfo的信息得到具体的Activity。

此处保存这些信息，主要是为了提高运行过程中的效率。Goolge工程师可能觉得ChooserActivity使用的地方比较多，所以这里单独保存了此Activity的信息。

好，继续对scanPackageLI函数的分析。

[-->PackageManagerService::scanPackageLI函数]
```java
......//mPackages用于保存系统内的所有Package，以packageName为key

if(mPackages.containsKey(pkg.packageName)

        || mSharedLibraries.containsKey(pkg.packageName)) {

    return null;

}

File destCodeFile = newFile(pkg.applicationInfo.sourceDir);

FiledestResourceFile = new File(pkg.applicationInfo.publicSourceDir);

SharedUserSettingsuid = null;//代表该Package的SharedUserSetting对象

PackageSetting pkgSetting = null;//代表该Package的PackageSetting对象

synchronized(mPackages) {

    ......//此段代码大约有300行左右，主要做了以下几方面工作

        /*

           ①如果该Packge声明了” uses-librarie”话，那么系统要判断该library是否

           在mSharedLibraries中

           ②如果package声明了SharedUser，则需要处理SharedUserSettings相关内容，

           由Settings的getSharedUserLPw函数处理

           ③处理pkgSetting，通过调用Settings的getPackageLPw函数完成

           ④调用verifySignaturesLP函数，检查该Package的signature

         */

}

finallong scanFileTime = scanFile.lastModified();

finalboolean forceDex = (scanMode&SCAN_FORCE_DEX) != 0;

//确定运行该package的进程的进程名，一般用packageName作为进程名

pkg.applicationInfo.processName = fixProcessName(

        pkg.applicationInfo.packageName,

        pkg.applicationInfo.processName,

        pkg.applicationInfo.uid);

if(mPlatformPackage == pkg) {

    dataPath = new File (Environment.getDataDirectory(),"system");

    pkg.applicationInfo.dataDir = dataPath.getPath();

}else {

    /*

       getDataPathForPackage函数返回该package的目录

       一般是/data/data/packageName/

     */

    dataPath = getDataPathForPackage(pkg.packageName, 0);

    if(dataPath.exists()){

        ......//如果该目录已经存在，则要处理uid的问题

    } else {

        ......//向installd发送install命令，实际上就是在/data/data下

            //建立packageName目录。后续将分析installd相关知识

            int ret = mInstaller.install(pkgName, pkg.applicationInfo.uid,

                    pkg.applicationInfo.uid);

        //为系统所有user安装此程序

        mUserManager.installPackageForAllUsers(pkgName,

                pkg.applicationInfo.uid);



        if (dataPath.exists()) {

            pkg.applicationInfo.dataDir = dataPath.getPath();

        } ......



        if (pkg.applicationInfo.nativeLibraryDir == null &&

                pkg.applicationInfo.dataDir!= null) {

            ......//为该Package确定native library所在目录

                //一般是/data/data/packagename/lib

        }

    }

    //如果该APK包含了native动态库，则需要将它们从APK文件中解压并复制到对应目录中

    if(pkg.applicationInfo.nativeLibraryDir != null) {

        try {

            final File nativeLibraryDir = new

                File(pkg.applicationInfo.nativeLibraryDir);

            final String dataPathString = dataPath.getCanonicalPath();

            //从2.3开始，系统package的native库统一放在/system/lib下。所以

            //系统不会提取系统Package目录下APK包中的native库

            if (isSystemApp(pkg) && !isUpdatedSystemApp(pkg)) {

                NativeLibraryHelper.removeNativeBinariesFromDirLI(

                        nativeLibraryDir)){

                        } else if (nativeLibraryDir.getParentFile().getCanonicalPath()

                                .equals(dataPathString)) {

                            boolean isSymLink;

                            try {

                                isSymLink = S_ISLNK(Libcore.os.lstat(

                                            nativeLibraryDir.getPath()).st_mode);

                            } ......//判断是否为链接，如果是，需要删除该链接

                            if (isSymLink) {

                                mInstaller.unlinkNativeLibraryDirectory(dataPathString);

                            }

                            //在lib下建立和CPU类型对应的目录，例如ARM平台的是arm/，MIPS平台的是mips/

                            NativeLibraryHelper.copyNativeBinariesIfNeededLI(scanFile,

                                    nativeLibraryDir);

                        } else {

                            mInstaller.linkNativeLibraryDirectory(dataPathString,

                                    pkg.applicationInfo.nativeLibraryDir);

                        }

            } ......

        }

        pkg.mScanPath= path;

        if((scanMode&SCAN_NO_DEX) == 0) {

            ......//对该APK做dex优化

                performDexOptLI(pkg,forceDex, (scanMode&SCAN_DEFER_DEX);

                        }

                        //如果该APK已经存在，要先杀掉运行该APK的进程

                        if((parseFlags & PackageManager.INSTALL_REPLACE_EXISTING) != 0) {

                        killApplication(pkg.applicationInfo.packageName,

                            pkg.applicationInfo.uid);

                        }

                        ......

                        /*

                           在此之前，四大组件信息都属于Package的私有财产，现在需要把它们登记注册到PKMS内部的

                           财产管理对象中。这样，PKMS就可对外提供统一的组件信息，而不必拘泥于具体的Package

                         */

                synchronized(mPackages) {

                    if ((scanMode&SCAN_MONITOR) != 0) {

                        mAppDirs.put(pkg.mPath, pkg);

                    }

                    mSettings.insertPackageSettingLPw(pkgSetting, pkg);

                    mPackages.put(pkg.applicationInfo.packageName,pkg);

                    //处理该Package中的Provider信息

                    int N =pkg.providers.size();

                    int i;

                    for (i=0;i<N; i++) {

                        PackageParser.Providerp = pkg.providers.get(i);

                        p.info.processName=fixProcessName(

                                pkg.applicationInfo.processName,

                                p.info.processName, pkg.applicationInfo.uid);

                        //mProvidersByComponent提供基于ComponentName的Provider信息查询

                        mProvidersByComponent.put(new ComponentName(

                                    p.info.packageName,p.info.name), p);

                        ......

                    }

                    //处理该Package中的Service信息

                    N =pkg.services.size();

                    r = null;

                    for (i=0;i<N; i++) {

                        PackageParser.Service s =pkg.services.get(i);

                        mServices.addService(s);

                    }

                    //处理该Package中的BroadcastReceiver信息

                    N =pkg.receivers.size();

                    r = null;

                    for (i=0;i<N; i++) {

                        PackageParser.Activity a =pkg.receivers.get(i);

                        mReceivers.addActivity(a,"receiver");

                        ......

                    }

                    //处理该Package中的Activity信息

                    N = pkg.activities.size();

                    r =null;

                    for (i=0; i<N; i++) {

                        PackageParser.Activity a =pkg.activities.get(i);

                        mActivities.addActivity(a,"activity");//后续将详细分析该调用

                    }

                    //处理该Package中的PermissionGroups信息

                    N = pkg.permissionGroups.size();

                    ......//permissionGroups处理

                        N =pkg.permissions.size();

                    ......//permissions处理

                        N =pkg.instrumentation.size();

                    ......//instrumentation处理

                        if(pkg.protectedBroadcasts != null) {

                            N = pkg.protectedBroadcasts.size();

                            for(i=0; i<N; i++) {

                                mProtectedBroadcasts.add(pkg.protectedBroadcasts.get(i));

                            }

                        }

                    ......//Package的私有财产终于完成了公有化改造

                        return pkg;

                }
```

到此这个长达800行的代码就分析完了，下面总结一下Package扫描的流程。

##### （5） scanDirLI函数总结
scanDirLI用于对指定目录下的APK文件进行扫描，如图4-7所示为该函数的调用流程。



![图4-7  scanDirLI工作流程总结](/images/understand2/4-7.png)

图4-7比较简单，相关知识无须赘述。读者在自行分析代码时，只要注意区分这两个同名scanPackageLI函数即可。

扫描完APK文件后，Package的私有财产就充公了。PKMS提供了好几个重要数据结构来保存这些财产，这些数据结构的相关信息如图4-8所示。



![图4-8  PKMS中重要的数据结构](/images/understand2/4-8.png)

图4-8借用UML的类图来表示PKMS中重要的数据结构。每个类图的第一行为成员变量名，第二行为数据类型，第三行为注释说明。

#### 3.  扫描非系统Package
非系统Package就是指那些不存储在系统目录下的APK文件，这部分代码如下：

[-->PackageManagerService.java::构造函数第三部分]
```java
if (!mOnlyCore) {//mOnlyCore用于控制是否扫描非系统Package

    Iterator<PackageSetting> psit =  

        mSettings.mPackages.values().iterator();

    while (psit.hasNext()) {

        ......//删除系统package中那些不存在的APK

    }

    mAppInstallDir = new File(dataDir,"app");

    ......//删除安装不成功的文件及临时文件

        if (!mOnlyCore) {

            //在普通模式下，还需要扫描/data/app以及/data/app_private目录 

            mAppInstallObserver = new AppDirObserver(

                    mAppInstallDir.getPath(), OBSERVER_EVENTS, false);

            mAppInstallObserver.startWatching();

            scanDirLI(mAppInstallDir, 0, scanMode, 0);

            mDrmAppInstallObserver = newAppDirObserver(

                    mDrmAppPrivateInstallDir.getPath(), OBSERVER_EVENTS, false);

            mDrmAppInstallObserver.startWatching();

            scanDirLI(mDrmAppPrivateInstallDir,           

                    PackageParser.PARSE_FORWARD_LOCK,scanMode,0);

        } else {

            mAppInstallObserver = null;

            mDrmAppInstallObserver = null;

        }
```

结合前述代码，这里总结几个存放APK文件的目录。

-  系统Package目录包括：/system/frameworks、/system/app和/vendor/app。

-  非系统Package目录包括：/data/app、/data/app-private。

#### **4.  第二阶段工作总结**

PKMS构造函数第二阶段的工作任务非常繁重，要创建比较多的对象，所以它是一个耗时耗内存的操作。在工作中，我们一直想优化该流程以加快启动速度，例如延时扫描不重要的APK，或者保存Package信息到文件中，然后在启动时从文件中恢复这些信息以减少APK文件读取并解析XML的工作量。但是一直没有一个比较完满的解决方案，原因有很多。比如APK之间有着比较微妙的依赖关系，因此到底延时扫描哪些APK，尚不能确定。另外，笔者感到比较疑惑的一个问题是：对于多核CPU架构，PKMS可以启动多个线程以扫描不同的目录，但是目前代码中还没有寻找到相关的蛛丝马迹。难道此处真的就不能优化了吗？读者如果有更好的解决方案，不妨和大家分享一下。

### 4.3.3  构造函数分析之扫尾工作
下面分析PKMS第三阶段的工作，这部分任务比较简单，就是将第二阶段收集的信息再集中整理一次，比如将有些信息保存到文件中，相关代码如下：

[-->PackageManagerService.java::构造函数]
```java
......

mSettings.mInternalSdkPlatform= mSdkVersion;

//汇总并更新和Permission相关的信息

updatePermissionsLPw(null, null, true,

        regrantPermissions,regrantPermissions);

//将信息写到package.xml、package.list及package-stopped.xml文件中

mSettings.writeLPr();

Runtime.getRuntime().gc();

mRequiredVerifierPackage= getRequiredVerifierLPr();

......//PKMS构造函数返回

}
```

读者可自行研究以上代码中涉及的几个函数，这里不再赘述。

### 4.3.4  PKMS构造函数总结
从流程角度看，PKMS构造函数的功能还算清晰，无非是扫描XML或APK文件，但是其中涉及的数据结构及它们之间的关系却较为复杂。这里有一些建议供读者参考：

-  理解PKMS构造函数工作的三个阶段及其各阶段的工作职责。

-  了解PKMS第二阶段工作中解析APK文件的几个关键步骤，可参考图4-7。

-  了解重点数据结构的名字和大体功能。

如果对PKMS的分析到此为止，则未免有些太小视它了。下面将分析几个重量级的知识点，期望能带领读者全方位认识PKMS。

## 4.4  APK Installation分析
本节将分析APK的安装及相关处理流程，它可能比读者想象得要复杂。

马上开始我们的行程，故事从adb install开始。

### 4.4.1  adb install分析
adb install有多个参数，这里仅考虑最简单的，如adb installframeworktest.apk。adb是一个命令，install是它的参数。此处直接跳到处理install参数的代码：

[-->commandline.c]
```c
int adb_commandline(int argc, char **argv){

   //...... 

if(!strcmp(argv[0], "install")) {

       ......//调用install_app函数处理

       return install_app(ttype, serial, argc, argv);

}

......

}
```

install_app函数也在commandline.c中定义，代码如下：

[-->commandline.c]
```c
int install_app(transport_type transport, char*serial, int argc, char** argv)

{

    //要安装的APK现在还在Host机器上，要先把APK复制到手机中。

   //这里需要设置复制目标的目录，如果安装在内部存储中，则目标目录为/data/local/tmp；

   //如果安装在SD卡上，则目标目录为/sdcard/tmp。

    staticconst char *const DATA_DEST = "/data/local/tmp/%s";

    staticconst char *const SD_DEST = "/sdcard/tmp/%s";

    constchar* where = DATA_DEST;

    charapk_dest[PATH_MAX];

    charverification_dest[PATH_MAX];

    char*apk_file;

    char*verification_file = NULL;

    intfile_arg = -1;

    int err;

    int i;

    for (i =1; i < argc; i++) {

        if(*argv[i] != '-') {

           file_arg = i;

           break;

        }else if (!strcmp(argv[i], "-i")) {

            i++;

        }else if (!strcmp(argv[i], "-s")) {

           where = SD_DEST; //-s参数指明该APK安装到SD卡上

        }

    }

    ......

    apk_file= argv[file_arg];

    ......

    //获取目标文件的全路径，如果安装在内部存储中，则目标全路径为/data/local/tmp/安装包名，

    //调用do_sync_push将此APK传送到手机的目标路径

    err =do_sync_push(apk_file, apk_dest, 1 /* verify APK */);

    ...... //①4.0新增了一个安装包Verification功能，相关知识稍后分析

    //②执行pm命令，这个函数很有意思

    pm_command(transport,serial, argc, argv);

......

  cleanup_apk:

    //③在手机中执行shell rm 命令，删除刚才传送过去的目标APK文件。为什么要删除呢

   delete_file(transport, serial, apk_dest);

    returnerr;

}
```

以上代码中共有三个关键点，分别是：

-  4.0新增了APK安装过程中的Verification的功能。其实就是在安装时，把相关信息发送给指定的Verification程序（另外一个APK），由它对要安装的APK进行检查（Verify）。这部分内容在后面分析APK 安装时会介绍。目前，标准代码中还没有从事Verification工作的APK。

-  调用pm_command进行安装，这是一个比较有意思的函数，稍后对其进行分析。

-  安装完后，执行shell rm删除刚才传送给手机的APK文件。为什么会删除呢？因为PKMS在安装过程中会将该APK复制一份到/data/app目录下，所以/data/local/tmp下的对应文件就可以删除了。这部分代码在后面也能见到。

先来分析pm_command命令。为什么说它很有意思呢？

### 4.4.2  pm分析
pm_command代码如下：

[-->commandline.c]
```c
static int pm_command(transport_type transport,char* serial,

                      int argc, char** argv)

{

    charbuf[4096];

    snprintf(buf,sizeof(buf), "shell:pm");

  ......//准备参数

  //发送"shell:pm install 参数"给手机端的adbd

   send_shellcommand(transport, serial, buf);

    return0;

}
```

手机端的adbd在收到客户端发来的shellpm命令时会启动一个shell，然后在其中执行pm。pm是什么？为什么可以在shell下执行？

提示读者可以通过adb shell登录到自己手机，然后执行pm，看看会发现什么。

pm实际上是一个脚本，其内容如下：

[-->pm]

```shell
# Script to start "pm" on the device,which has a very rudimentary

# shell.

#

base=/system

export CLASSPATH=$base/frameworks/pm.jar

exec app_process $base/bincom.android.commands.pm.Pm "$@"
```

在编译system.image时，Android.mk中会将该脚本复制到system/bin目录下。从pm脚本的内容来看，它就是通过app_process执行pm.jar包的main函数。在卷I第4章分析Zygote时，已经介绍了app_process是一个Native进程，它通过创建虚拟机启动了Zygote，从而转变为一个Java进程。实际上，app_process还可以通过类似的方法（即先创建Dalvik虚拟机，然后执行某个类的main函数）来转变成其他Java程序。

注意Android系统中常用的monkeytest、pm、am等（这些都是脚本文件）都是以这种方式启动的，所以严格地说，app_process才是Android Java进程的老祖宗。

下面来分析pm.java，app_process执行的就是它定义的main函数，它相当于Java进程的入口函数，其代码如下：

[-->pm.java]
```java
public static void main(String[] args) {

    newPm().run(args);//创建一个Pm对象，并执行它的run函数

}

//直接分析run函数

public void run(String[] args) {

    boolean validCommand = false;

    ......

        //获取PKMS的binder客户端

        mPm= IPackageManager.Stub.asInterface(

                ServiceManager.getService("package"));

    ......

        mArgs = args;

    String op = args[0];

    mNextArg = 1;

    ......//处理其他命令，这里仅考虑install的处理

        if("install".equals(op)) {

            runInstall();

            return;

        }

    ......

}
```

接下来分析pm.java的runInstall函数，代码如下：

[-->pm.java]
```java
private void runInstall() {

    intinstallFlags = 0;

    String installerPackageName = null;



    String opt;

    while ((opt=nextOption()) != null) {

        if (opt.equals("-l")) {

            installFlags |= PackageManager.INSTALL_FORWARD_LOCK;

        } else if (opt.equals("-r")) {

            installFlags |= PackageManager.INSTALL_REPLACE_EXISTING;

        } else if (opt.equals("-i")) {

            installerPackageName = nextOptionData();

            ...... //参数解析

        } ......

    }



    final Uri apkURI;

    final Uri verificationURI;

    final String apkFilePath = nextArg();

    System.err.println("/tpkg: " + apkFilePath);

    if(apkFilePath != null) {

        apkURI = Uri.fromFile(new File(apkFilePath));

    }......

    //获取Verification Package的文件位置

    final String verificationFilePath = nextArg();

    if(verificationFilePath != null) {

        verificationURI = Uri.fromFile(new File(verificationFilePath));

    }else {

        verificationURI = null;

    }

    //创建PackageInstallObserver，用于接收PKMS的安装结果

    PackageInstallObserver obs = new PackageInstallObserver();

    try{

        //①调用PKMS的installPackageWithVerification完成安装

        mPm.installPackageWithVerification(apkURI, obs,

                installFlags,installerPackageName,

                verificationURI,null);

        synchronized (obs) {

            while(!obs.finished) {

                try{

                    obs.wait();//等待安装结果

                } ......

            }

            if(obs.result == PackageManager.INSTALL_SUCCEEDED) {

                System.out.println("Success");//安装成功，打印Success

            }......//安装失败，打印失败原因

        } ......

    }
```

Pm解析参数后，最终通过PKMS的Binder客户端调用installPackageWithVerification以完成后续的安装工作，所以，下面进入PKMS看看安装到底是怎么一回事。

 

### 4.4.3  installPackageWithVerification函数分析
installPackageWithVerification的代码如下：

[-->PackageManagerService.java::installPackageWithVerification函数]
```java
public void installPackageWithVerification(UripackageURI,

        IPackageInstallObserverobserver,

        int flags, String installerPackageName, Uri verificationURI,

        ManifestDigest manifestDigest) {

    //检查客户端进程是否具有安装Package的权限。在本例中，该客户端进程是shell

    mContext.enforceCallingOrSelfPermission(

            android.Manifest.permission.INSTALL_PACKAGES,null);

    final int uid = Binder.getCallingUid();

    final int filteredFlags;

    if(uid == Process.SHELL_UID || uid == 0) {

        ......//如果通过shell pm的方式安装，则增加INSTALL_FROM_ADB标志

            filteredFlags = flags | PackageManager.INSTALL_FROM_ADB;

    }else {

        filteredFlags = flags & ~PackageManager.INSTALL_FROM_ADB;

    }

    //创建一个Message，code为INIT_COPY，将该消息发送给之前在PKMS构造函数中

    //创建的mHandler对象，将在另外一个工作线程中处理此消息

    final Message msg = mHandler.obtainMessage(INIT_COPY);

    //创建一个InstallParams，其基类是HandlerParams

    msg.obj = new InstallParams(packageURI, observer,

            filteredFlags,installerPackageName,

            verificationURI,manifestDigest);

    mHandler.sendMessage(msg);

}

```

installPackageWithVerification函数倒是蛮清闲，简简单单创建几个对象，然后发送INIT_COPY消息给mHandler，就甩手退出了。根据之前在PKMS构造函数中介绍的知识可知，mHandler被绑定到另外一个工作线程（借助ThreadHandler对象的Looper）中，所以该INIT_COPY消息也将在那个工作线程中进行处理。我们马上转战到那。

#### 1.  INIT_COPY处理
INIT_COPY只是安装流程的第一步。先来看相关代码：

[-->PackageManagerService.java::handleMesssage]
```java
public void handleMessage(Message msg) {

    try {

        doHandleMessage(msg);//调用doHandleMessage函数

    } ......

}

voiddoHandleMessage(Message msg) {

    switch(msg.what) {

caseINIT_COPY: {

                   //①这里记录的是params的基类类型HandlerParams，实际类型为InstallParams

                   HandlerParams params = (HandlerParams) msg.obj;

                   //idx为当前等待处理的安装请求的个数

                   intidx = mPendingInstalls.size();

                   if(!mBound) {

                       /*

                          很多读者可能想不到，APK的安装居然需要使用另外一个APK提供的服务，该服务就是

                          DefaultContainerService，由DefaultCotainerService.apk提供，

                          下面的connectToService函数将调用bindService来启动该服务

                        */

                       if(!connectToService()) {

                           return;

                       }else {//如果已经连上，则以idx为索引，将params保存到mPendingInstalls中

                           mPendingInstalls.add(idx, params);

                       }

                   } else {

                       mPendingInstalls.add(idx, params);

                       if(idx == 0) {

                           //如果安装请求队列之前的状态为空，则表明要启动安装

                           mHandler.sendEmptyMessage(MCS_BOUND);

                       }

                   }

                   break;

               }

               ......//后续再分析
```

这里假设之前已经成功启动了DefaultContainerService（以后简称DCS），并且idx为零，所以这是PKMS首次处理安装请求，也就是说，下一个将要处理的是MCS_BOUND消息。

注意connectToService在调用bindService时会传递一个DefaultContainerConnection类型的对象，以接收服务启动的结果。当该服务成功启动后，此对象的onServiceConnected被调用，其内部也将发送MCS_BOUND消息给mHandler。

#### 2.  MCS_BOUND处理
现在，安装请求的状态从INIT_COPY变成MCS_BOUND了，此时的处理流程时怎样的呢？依然在doHandleMessage函数中，直接从对应的case开始，代码如下：

[-->PackageManagerService.java::]

```java
......//接doHandleMesage中的switch/case

        case MCS_BOUND: {

                            if(msg.obj != null) {

                                mContainerService= (IMediaContainerService) msg.obj;

                            }

                            if(mContainerService == null) {

                                ......//如果没法启动该service，则不能安装程序

                                    mPendingInstalls.clear();

                            } else if(mPendingInstalls.size() > 0) {

                                HandlerParamsparams = mPendingInstalls.get(0);

                                if(params != null) {

                                    //调用params对象的startCopy函数，该函数由基类HandlerParams定义

                                    if(params.startCopy()) {

                                        ......

                                            if(mPendingInstalls.size() > 0) {

                                                mPendingInstalls.remove(0);//删除队列头

                                            }

                                        if (mPendingInstalls.size() == 0) {

                                            if (mBound) {

                                                ......//如果安装请求都处理完了，则需要和Service断绝联系,

                                                    //通过发送MSC_UNB消息处理断交请求。读者可自行研究此情况的处理流程

                                                    removeMessages(MCS_UNBIND);

                                                Message ubmsg = obtainMessage(MCS_UNBIND);

                                                sendMessageDelayed(ubmsg, 10000);

                                            }

                                        }else {

                                            //如果还有未处理的请求，则继续发送MCS_BOUND消息。

                                            //为什么不通过一个循环来处理所有请求呢

                                            mHandler.sendEmptyMessage(MCS_BOUND);

                                        }

                                    }

                                } ......

                                break;
```

MCS_BOUND的处理还算简单，就是调用HandlerParams的startCopy函数。在深入分析前，应先认识一下HandlerParams及相关的对象。

#####（1） HandlerParams和InstallArgs介绍
除了HandlerParams家族外，这里提前请出另外一个家族InstallArgs及其成员，如图4-8所示。



![图4-8  HandlerParams及InstallArgs家族成员](/images/understand2/4-8-8.png)

由图4-8可知：

-  HandlerParams和InstallArgs均为抽象类。

-  HandlerParams有三个子类，分别是InstallParams、MoveParams和MeasureParams。其中，InstallParams用于处理APK的安装，MoveParams用于处理某个已安装APK的搬家请求（例如从内部存储移动到SD卡上），MeasureParams用于查询某个已安装的APK占据存储空间的大小（例如在设置程序中得到的某个APK使用的缓存文件的大小）。

-  对于InstallParams来说，它还有两个伴儿，即InstallArgs的派生类FileInstallArgs和SdInstallArgs。其中，FileInstallArgs针对的是安装在内部存储的APK，而SdInstallArgs针对的是那些安装在SD卡上的APK。

本节将讨论用于内部存储安装的FileInstallArgs。

提示读者可以在介绍完MountService后，结合本章知识点，自行研究SdInstallArgs的处理流程。

在前面MCS_BOUND的处理中，首先调用InstallParams的startCopy函数，该函数由其基类HandlerParams实现，代码如下：

[-->PackageManagerService.java::HandlerParams.startCopy函数]
```java
final boolean startCopy() {

    booleanres;

    try {

        //MAX_RETIRES目前为4，表示尝试4次安装，如果还不成功，则认为安装失败

        if(++mRetries > MAX_RETRIES) {

            mHandler.sendEmptyMessage(MCS_GIVE_UP);

            handleServiceError();

            return false;

        } else {

            handleStartCopy();//①调用派生类的handleStartCopy函数

            res= true;

        }

    } ......

    handleReturnCode();//②调用派生类的handleReturnCode，返回处理结果

    returnres;

}
```

在上述代码中，基类的startCopy将调用子类实现的handleStartCopy和handleReturnCode函数。下面来看InstallParams是如何实现这两个函数的。

##### （2） InstallParams分析
先来看派生类InstallParams的handleStartCopy函数，代码如下：

[-->PackageManagerService::InstallParams.handleStartCopy]
```java
public void handleStartCopy() throwsRemoteException {

    int ret= PackageManager.INSTALL_SUCCEEDED;

    finalboolean fwdLocked = //本书不考虑fwdLocked的情况

        (flags &PackageManager.INSTALL_FORWARD_LOCK) != 0;

    //根据adb install的参数，判断安装位置

    finalboolean onSd = (flags & PackageManager.INSTALL_EXTERNAL) != 0;

    finalboolean onInt = (flags & PackageManager.INSTALL_INTERNAL) != 0;

    PackageInfoLite pkgLite = null;

    if(onInt && onSd) {

        //APK不能同时安装在内部存储和SD卡上

        ret =PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;

    } elseif (fwdLocked && onSd) {

        //fwdLocked的应用不能安装在SD卡上

        ret =PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;

    } else {

        finallong lowThreshold;

        //获取DeviceStorageMonitorService的binder客户端

        finalDeviceStorageMonitorService dsm =                           

            (DeviceStorageMonitorService) ServiceManager.getService(

                    DeviceStorageMonitorService.SERVICE);

        if(dsm == null) {

            lowThreshold = 0L;

        }else {

            //从DSMS查询内部空间最小余量，默认是总空间的10%

            lowThreshold = dsm.getMemoryLowThreshold();

        }

        try {

            //授权DefContainerService URI读权限

            mContext.grantUriPermission(DEFAULT_CONTAINER_PACKAGE,

                    packageURI,Intent.FLAG_GRANT_READ_URI_PERMISSION);

            //①调用DCS的getMinimalPackageInfo函数，得到一个PackageLite对象

            pkgLite =mContainerService.getMinimalPackageInfo(packageURI,

                    flags,lowThreshold);

        }finally ......//撤销URI授权

        //PacakgeLite的recommendedInstallLocation成员变量保存该APK推荐的安装路径

        int loc =pkgLite.recommendedInstallLocation;

        if (loc== PackageHelper.RECOMMEND_FAILED_INVALID_LOCATION) {

            ret= PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;

        } else if......{

        } else {

            //②根据DCS返回的安装路径，还需要调用installLocationPolicy进行检查

            loc =installLocationPolicy(pkgLite, flags);

            if(!onSd && !onInt) {

                if(loc == PackageHelper.RECOMMEND_INSTALL_EXTERNAL) {

                    flags |= PackageManager.INSTALL_EXTERNAL;

                    flags &=~PackageManager.INSTALL_INTERNAL;

                } ......//处理安装位置为内部存储的情况

            }

        }

    }

    //③创建一个安装参数对象，对于安装位置为内部存储的情况，args的真实类型为FileInstallArgs

    finalInstallArgs args = createInstallArgs(this);

    mArgs =args;

    if (ret== PackageManager.INSTALL_SUCCEEDED) {

        final int requiredUid = mRequiredVerifierPackage == null ? -1

            :getPackageUid(mRequiredVerifierPackage);

        if(requiredUid != -1 && isVerificationEnabled()) {

            ......//④待会再讨论verification的处理

        }else {

            //⑤调用args的copyApk函数

            ret= args.copyApk(mContainerService, true);

        }

    }

    mRet =ret;//确定返回值

}
```

在以上代码中，一共列出了五个关键点，总结如下：

-  调用DCS的getMinimalPackageInfo函数，将得到一个PackageLite对象，该对象是一个轻量级的用于描述APK的结构（相比PackageParser.Package来说）。在这段代码逻辑中，主要想取得其recommendedInstallLocation的值。此值表示该APK推荐的安装路径。

-  调用installLocationPolicy检查推荐的安装路径。例如系统Package不允许安装在SD卡上。

-  createInstallArgs将根据安装位置创建不同的InstallArgs。如果是内部存储，则返回FileInstallArgs，否则为SdInstallArgs。

-  在正式安装前，应先对该APK进行必要的检查。这部分代码后续再介绍。

-  调用InstallArgs的copyApk。对本例来说，将调用FileInstallArgs的copyApk函数。

下面围绕这五个基本关键点展开分析，其中installLocationPolicy和createInstallArgs比较简单，读者可自行研究。

#### 3.  handleStartCopy分析
##### （1） DefaultContainerService分析
首先分析DCS的getMinimalPackageInfo函数，其代码如下：

[-->DefaultContainerService.java::getMinimalPackageInfo函数]
```java
public PackageInfoLite getMinimalPackageInfo(finalUri fileUri, int flags, longthreshold) {

    //注意该函数的参数：fileUri指向该APK的文件路径（此时还在/data/local/tmp下）

    PackageInfoLite ret = new PackageInfoLite();

    ......

        Stringscheme = fileUri.getScheme();

    ......

        StringarchiveFilePath = fileUri.getPath();

    DisplayMetrics metrics = new DisplayMetrics();

    metrics.setToDefaults();

    //调用PackageParser的parsePackageLite解析该APK文件

    PackageParser.PackageLite pkg =

        PackageParser.parsePackageLite(archiveFilePath,0);

    if (pkg== null) {//解析失败

        ......//设置错误值

            returnret;

    }

    ret.packageName = pkg.packageName;

    ret.installLocation = pkg.installLocation;

    ret.verifiers = pkg.verifiers;

    //调用recommendAppInstallLocation，取得一个合理的安装位置

    ret.recommendedInstallLocation =

        recommendAppInstallLocation(pkg.installLocation,archiveFilePath, flags, threshold);

    returnret;

}
```

APK可在AndroidManifest.xml中声明一个安装位置，不过DCS除了解析该位置外，还需要做进一步检查，这个工作由recommendAppInstallLocation函数完成，代码如下：

[-->DefaultContainerService.java::recommendAppInstallLocation函数]
```java
private int recommendAppInstallLocation(intinstallLocation,
        StringarchiveFilePath, int flags,long threshold) {

    int prefer;
    booleancheckBoth = false;
check_inner: {
                 if((flags & PackageManager.INSTALL_FORWARD_LOCK) != 0) {
                     prefer = PREFER_INTERNAL;
                     break check_inner; //根据FOWRAD_LOCK的情况，只能安装在内部存储
                 } elseif ((flags & PackageManager.INSTALL_INTERNAL) != 0) {

                     prefer = PREFER_INTERNAL;

                     break check_inner;

                 }

                 ......//检查各种情况

             } else if(installLocation == PackageInfo.INSTALL_LOCATION_AUTO) {

                 prefer= PREFER_INTERNAL;//一般设定的位置为AUTO，默认是内部空间

                 checkBoth = true; //设置checkBoth为true

                 breakcheck_inner;

             }

             //查询settings数据库中的secure表，获取用户设置的安装路径

             intinstallPreference =
                 Settings.System.getInt(getApplicationContext()

                         .getContentResolver(),

                         Settings.Secure.DEFAULT_INSTALL_LOCATION,

                         PackageHelper.APP_INSTALL_AUTO);

             if(installPreference == PackageHelper.APP_INSTALL_INTERNAL) {

                 prefer = PREFER_INTERNAL;

                 break check_inner;

             } else if(installPreference == PackageHelper.APP_INSTALL_EXTERNAL) {

                 prefer= PREFER_EXTERNAL;

                 breakcheck_inner;

             }

             prefer =PREFER_INTERNAL;

}

//判断外部存储空间是否为模拟的，这部分内容我们以后再介绍

finalboolean emulated = Environment.isExternalStorageEmulated();

final FileapkFile = new File(archiveFilePath);

booleanfitsOnInternal = false;

if(checkBoth || prefer == PREFER_INTERNAL) {

    try {//检查内部存储空间是否足够大

        fitsOnInternal = isUnderInternalThreshold(apkFile, threshold);

    } ......

}

booleanfitsOnSd = false;

if(!emulated && (checkBoth || prefer == PREFER_EXTERNAL)) {

    try{ //检查外部存储空间是否足够大

        fitsOnSd = isUnderExternalThreshold(apkFile);

    } ......

}

if (prefer== PREFER_INTERNAL) {

    if(fitsOnInternal) {//返回推荐安装路径为内部空间

        return PackageHelper.RECOMMEND_INSTALL_INTERNAL;

    }

} elseif (!emulated && prefer == PREFER_EXTERNAL) {

    if(fitsOnSd) {//返回推荐安装路径为外部空间

        returnPackageHelper.RECOMMEND_INSTALL_EXTERNAL;

    }

}



if(checkBoth) {

    if(fitsOnInternal) {//如果内部存储满足条件，先返回内部空间

        return PackageHelper.RECOMMEND_INSTALL_INTERNAL;

    }else if (!emulated && fitsOnSd) {

        return PackageHelper.RECOMMEND_INSTALL_EXTERNAL;

    }

}

...... //到此，前几个条件都不满足，此处将根据情况返回一个明确的错误值

returnPackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE;

}

}
```

DCS的getMinimalPackageInfo函数为了得到一个推荐的安装路径做了不少工作，其中，各种安装策略交叉影响。这里总结一下相关的知识点：

-  APK在AndroidManifest.xml中设置的安装点默认为AUTO，在具体对应时倾向内部空间。

-  用户在Settings数据库中设置的安装位置。

-  检查外部存储或内部存储是否有足够空间。

##### （2） InstallArgs的copyApk函数分析
至此，我们已经得到了一个合适的安装位置（先略过Verification这一步）。下一步工作就由copyApk来完成。根据函数名可知该函数将完成APK文件的复制工作，此中会有蹊跷吗？来看下面的代码。

[-->PackageManagerService.java::InstallArgs.copyApk函数]
```java
int copyApk(IMediaContainerService imcs, booleantemp) throws RemoteException {

    if (temp){

        /*

           本例中temp参数为true，createCopyFile将在/data/app下创建一个临时文件。

           临时文件名为vmdl-随机数.tmp。为什么会用这样的文件名呢？

           因为PKMS通过Linux的inotify机制监控了/data/app,目录，如果新复制生成的文件名后缀

           为apk，将触发PKMS扫描。为了防止发生这种情况，这里复制生成的文件才有了

           如此奇怪的名字

         */

        createCopyFile();

    }

    FilecodeFile = new File(codeFileName);

    ......

        ParcelFileDescriptor out = null;

    try {

        out =ParcelFileDescriptor.open(codeFile,

                ParcelFileDescriptor.MODE_READ_WRITE);

    }......

    int ret= PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;

    try {

        mContext.grantUriPermission(DEFAULT_CONTAINER_PACKAGE,

                packageURI,Intent.FLAG_GRANT_READ_URI_PERMISSION);

        //调用DCS的copyResource，该函数将执行复制操作，最终结果是/data/local/tmp

        //下的APK文件被复制到/data/app下，文件名也被换成vmdl-随机数.tmp

        ret= imcs.copyResource(packageURI, out);

    }finally {

        ......//关闭out，撤销URI授权

    }

    returnret;

}
 ```

关于临时文件，这里提供一个示例，如图4-9所示。



![图4-9  createCopyFile生成的临时文件](/images/understand2/4-9.jpg)

由图4-9可知：/data/app下有两个文件，第一个是正常的APK文件，第二个是createCopyFile生成的临时文件。

#### 4.  handleReturnCode分析
在HandlerParams的startCopy函数中，handleStartCopy执行完之后，将调用handleReturnCode开展后续工作，代码如下：

[-->PackageManagerService.java::InstallParams.HandleParams]
```java
void handleReturnCode() {

    if(mArgs != null) {

        //调用processPendingInstall函数，mArgs指向之前创建的FileInstallArgs对象

        processPendingInstall(mArgs, mRet);

    }

}
```

[-->PackageManagerService.java::]
```java
private void processPendingInstall(finalInstallArgs args,

        final intcurrentStatus) {

    //向mHandler中抛一个Runnable对象

    mHandler.post(new Runnable() {

            publicvoid run() {

            mHandler.removeCallbacks(this);

            //创建一个PackageInstalledInfo对象，

            PackageInstalledInfo res = new PackageInstalledInfo();

            res.returnCode = currentStatus;

            res.uid = -1;

            res.pkg = null;

            res.removedInfo = new PackageRemovedInfo();

            if(res.returnCode == PackageManager.INSTALL_SUCCEEDED) {

            //①调用FileInstallArgs的doPreInstall

                args.doPreInstall(res.returnCode);

                synchronized (mInstallLock) {

                    //②调用installPackageLI进行安装

                    installPackageLI(args, true, res);

                }

                //③调用FileInstallArgs的doPostInstall

                args.doPostInstall(res.returnCode);

            }

            final boolean update = res.removedInfo.removedPackage != null;

            boolean doRestore = (!update&& res.pkg != null &&

                    res.pkg.applicationInfo.backupAgentName!= null);

            int token;//计算一个ID号

            if(mNextInstallToken < 0) mNextInstallToken = 1;

            token = mNextInstallToken++;

            //创建一个PostInstallData对象

            PostInstallData data = new PostInstallData(args, res);

            //保存到mRunningInstalls结构中，以token为key

            mRunningInstalls.put(token, data);

            if (res.returnCode ==PackageManager.INSTALL_SUCCEEDED && doRestore)

            {

                ......//备份恢复的情况暂时不考虑

            }

            if(!doRestore) {

                //④抛一个POST_INSTALL消息给mHandler进行处理

                Message msg = mHandler.obtainMessage(POST_INSTALL, token, 0);

                mHandler.sendMessage(msg);

            }

            }

    });

}
```

由上面代码可知，handleReturnCode主要做了4件事情：

-  调用InstallArgs的doPreInstall函数，在本例中是FileInstallArgs的doPreInstall函数。

-  调用PKMS的installPackageLI函数进行APK安装，该函数内部将调用InstallArgs的doRename对临时文件进行改名。另外，还需要扫描此APK文件。此过程和之前介绍的“扫描系统Package”一节的内容类似。至此，该APK中的私有财产就全部被登记到PKMS内部进行保存了。

-  调用InstallArgs的doPostInstall函数，在本例中是FileInstallArgs的doPostInstall函数。

-  此时，该APK已经安装完成（不论失败还是成功），继续向mHandler抛送一个POST_INSTALL消息，该消息携带一个token，通过它可从mRunningInstalls数组中取得一个PostInstallData对象。

提示对于FileInstallArgs来说，其doPreInstall和doPostInstall都比较简单，读者可自行阅读相关代码。另外，读者也可自行研究PKMS的installPackageLI函数。

这里介绍一下FileInstallArgs的doRename函数，它的功能是将临时文件改名，最终的文件的名称一般为“包名-数字.apk”。其中，数字是一个index，从1开始。读者可参考图4-9中/data/app目录下第一个文件的文件名。

#### 5.  POST_INSTALL处理
现在需要处理POST_INSTALL消息，因为adb install还等着安装结果呢。相关代码如下：

[-->PackageManagerService.java::doHandleMessage函数]
```java
......//接前面的switch/case

case POST_INSTALL: {

                       PostInstallData data = mRunningInstalls.get(msg.arg1);

                       mRunningInstalls.delete(msg.arg1);

                       booleandeleteOld = false;



                       if (data!= null) {

                           InstallArgs args = data.args;

                           PackageInstalledInfo res = data.res;

                           if(res.returnCode == PackageManager.INSTALL_SUCCEEDED) {

                               res.removedInfo.sendBroadcast(false, true);

                               Bundle extras = new Bundle(1);

                               extras.putInt(Intent.EXTRA_UID, res.uid);

                               final boolean update = res.removedInfo.removedPackage != null;

                               if (update) {

                                   extras.putBoolean(Intent.EXTRA_REPLACING, true);

                               }

                               //发送PACKAGE_ADDED广播

                               sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED,

                                       res.pkg.applicationInfo.packageName,extras, null, null);

                               if (update) {

                                   /*

                                      如果是APK升级，那么发送PACKAGE_REPLACE和MY_PACKAGE_REPLACED广播。

                                      二者不同之处在于PACKAGE_REPLACE将携带一个extra信息

                                    */

                               }

                               Runtime.getRuntime().gc();

                               if(deleteOld) {

                                   synchronized (mInstallLock) {

                                       //调用FileInstallArgs的doPostDeleteLI进行资源清理

                                       res.removedInfo.args.doPostDeleteLI(true);

                                   }

                               }

                               if(args.observer != null) {

                                   try {

                                       // 向pm通知安装的结果

                                       args.observer.packageInstalled(res.name, res.returnCode);

                                   } ......

                               } break;
``` 

### 4.4.4  APK 安装流程总结
没想到APK的安装流程竟然如此复杂，其目的无非是让APK中的私人财产公有化。相比之下，在PKMS构造函数中进行公有化改造就非常简单。另外，如果考虑安装到SD卡的处理流程，那么APK的安装将会更加复杂。

这里要总结APK安装过程中的几个重要步骤，如图4-10所示。



![图4-10  APK安装流程](/images/understand2/4-10.png)

图4-10中列出以下内容：

-  安装APK到内部存储空间这一工作流程涉及的主要对象包括：PKMS、DCS、InstallParams和FileInstallArgs。

-  此工作流程中每个对象涉及到的关键函数。

-  对象之间的调用通过虚线表达，调用顺序通过①②③等标明。

4.4.5  Verification介绍
Verification功能的出现将打乱图4-10的工作流程，所以这部分内容要放在最后来介绍。其代码在InstallParams的handleStartCopy中，如下所示：

[-->PackageManagerService.java::InstallParams.handleStartCopy函数]
```java
......//此处已经获得了合适的安装位置

finalInstallArgs args = createInstallArgs(this);

mArgs =args;

if (ret == PackageManager.INSTALL_SUCCEEDED) {

    final int requiredUid =mRequiredVerifierPackage == null ? -1

        :getPackageUid(mRequiredVerifierPackage);

    if (requiredUid != -1 &&isVerificationEnabled()) {

        //创建一个Intent，用于查找满足条件的广播接收者

        finalIntent verification = new

            Intent(Intent.ACTION_PACKAGE_NEEDS_VERIFICATION);

        verification.setDataAndType(packageURI, PACKAGE_MIME_TYPE);

        verification.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);

        //查找满足Intent条件的广播接收者

        finalList<ResolveInfo> receivers = queryIntentReceivers(

                verification,null,PackageManager.GET_DISABLED_COMPONENTS);

        // verificationId为当前等待Verification的安装包个数

        finalint verificationId = mPendingVerificationToken++;

        //设置Intent的参数，例如要校验的包名

        verification.putExtra(PackageManager.EXTRA_VERIFICATION_ID,

                VerificationId);

        verification.putExtra(

                PackageManager.EXTRA_VERIFICATION_INSTALLER_PACKAGE,

                installerPackageName);

        verification.putExtra(

                PackageManager.EXTRA_VERIFICATION_INSTALL_FLAGS,flags);

        if(verificationURI != null) {

            verification.putExtra(PackageManager.EXTRA_VERIFICATION_URI,

                    verificationURI);

        }

        finalPackageVerificationState verificationState = new

            PackageVerificationState(requiredUid,args);

        //将上面创建的PackageVerificationState保存到mPendingVerification中

        mPendingVerification.append(verificationId, verificationState);

        //筛选符合条件的广播接收者

        finalList<ComponentName> sufficientVerifiers =

            matchVerifiers(pkgLite,receivers,verificationState);

        if (sufficientVerifiers != null) {

            finalint N = sufficientVerifiers.size();

            ......

                for(int i = 0; i < N; i++) {

                    finalComponentName verifierComponent = sufficientVerifiers.get(i);

                    final Intent sufficientIntent = newIntent(verification);

                    sufficientIntent.setComponent(verifierComponent);

                    //向校验包发送广播

                    mContext.sendBroadcast(sufficientIntent);

                }

        }

    }

    //除此之外，如果在执行adb install的时候指定了校验包，则需要向其单独发送校验广播

    finalComponentName requiredVerifierComponent =

        matchComponentForVerifier(mRequiredVerifierPackage,

                receivers);

    if (ret == PackageManager.INSTALL_SUCCEEDED

            &&mRequiredVerifierPackage != null) {

        verification.setComponent(requiredVerifierComponent);

        mContext.sendOrderedBroadcast(verification,

                android.Manifest.permission.PACKAGE_VERIFICATION_AGENT,

                new BroadcastReceiver() {

                //调用sendOrderdBroadcast，并传递一个BroadcastReceiver，该对象将在

                //广播发送的最后被调用。读者可参考sendOrderdBroadcast的文档说明

                public void onReceive(Context context, Intent intent) {

                final Message msg =mHandler.obtainMessage(

                    CHECK_PENDING_VERIFICATION);

                msg.arg1 = verificationId;

                //设置一个超时执行时间，该值来自Settings数据库的secure表，默认为60秒

                mHandler.sendMessageDelayed(msg, getVerificationTimeout());

                }

                },null, 0, null, null);

        mArgs = null;

    }

}......//不用做Verification的流程
```

PKMS的Verification工作其实就是收集安装包的信息，然后向对应的校验者发送广播。但遗憾的是，当前Android中还没有能处理Verification的组件。

另外，该组件处理完Verification后，需要调用PKMS的verifyPendingInstall函数，以通知校验结果。

## 4.5  queryIntentActivities分析
PKMS除了负责Android系统中Package的安装、升级、卸载外，还有一项很重要的职责，就是对外提供统一的信息查询功能，其中包括查询系统中匹配某Intent的Activities、BroadCastReceivers或Services等。本节将以查询匹配某Intent的Activities为例，介绍PKMS在这方便提供的服务。

正式分析queryIntentActivities之前，先来认识一下Intent及IntentFilter。

### 4.5.1  Intent及IntentFilter介绍
#### 1.  Intent介绍
Intent中文是“意图”的意思，它是Android系统中一个很重要的概念，其基本思想来源于对日常生活及行为的高度抽象。我们结合用人单位招聘的例子介绍Intent背后的思想。

-  假设某用人单位现需招聘人员完成某项工作。该单位首先其需求发给猎头公司。

-  猎头公司从其内部的信息库中查找合适的人选。猎头公司除了考虑用人单位的需求外，还需要考虑求职者本身的要求，例如有些求职者对工作地点、加班等有要求。

-  二者匹配后，就会得到满足要求的求职者。之后用人单位将工作交给满足条件的人员来完成。

在现实生活中，用人单位还需和求职者进行一系列其他交互工作，例如面试、签订合同之类。但是从完成工作的角度来看，只要把工作任务交给满足要求的求职者去做即可，中间的系列行为和工作任务本身没有太大关系。因此，Android并未将这部分内容抽象化。

意图，是一个非常抽象的概念，在编码设计中，如何将它实例化呢？Android系统明确指定的一个Intent可由两方面属性来衡量。

-  主要属性：包括Action和Data。其中Action用于表示该Intent所表达的动作意图、Data用于表示该Action所操作的数据。

-  次要属性：包括Category、Type、Component和Extras。其中Category表示类别，Type表示数据的MIME类型，Component可用于指定特定的Intent响应者（例如指定广播接收者为某Package的某个BroadcastReceiver），Extras用于承载其他的信息。

如果Intent是一份用工需求表，那么上述信息就是该表的全部可填项。在实际使用中，可根据需要填写该表的内容。

当这份需求表传给猎头公司后，猎头公司就根据该表所填写的内容，进一步对Intent进行分类。

-  Explicit Intents：这类Intent明确指明了要找哪些人。在代码中通过setComponent或setClass来锁定目标对象。处理这种Intent，工作就很轻松了。

-  Implicit Intents：这一类Intents只标明了工作内容，而没有指定具体人名。对于这类意图，猎头公司不得不做一系列复杂的工作才能找到满足用人单位需求的人才。

Intent就先介绍到这里。下面来看在这次招聘过程中求职者填写的信息。

##### 2.  IntentFilter介绍
求职方需要填写IntentFilter来表达自己的诉求。Andorid规定了3项内容.

-  Action：求职方支持的Intent动作（和Intent中的Action对应）。

-  Category：求职方支持的Intent种类（和Intent的Category对应）。

-  Data：求职方支持的Intent 数据（和Intent的Data对应，包括URI和MIME类型）。

至此，猎头公司既有了需求，又有了求职者的信息，马上要做的工作就是匹配查询。在Android中，该工作被称为Intent Resolution。由于现在及未来人才都是最宝贵的资源，因此猎头公司在做匹配工作时，将以Intent Filter列出的3项内容为参考标准，具体步骤如下：

-  首先匹配IntentFilter的Action，如果Intent设置的Action不满足IntentFilter的Action，则匹配失败。如果IntentFilter未设定Action，则匹配成功。

-  然后检查IntentFilter的Category，匹配方法同Action的匹配，唯一有些例外的是Category为CATEGORY_DEFAULT的情况。

-  最后检查Data。Data的匹配过程比较繁琐，因为它和IntentFilter设置的Data内容有关，见接下来的介绍。

IntentFilter中的Data可以包括两个内容。

-  URI：完整格式为“scheme://host:port/path”，包含4个部分，scheme、host、port和path。其中host和port合起来标示URI authority，用于指明服务器的网络地址（IP加端口号）。由于URI最多可包含,4个部分，因此要根据情况相应部分做匹配检查。

-  Date type：指定数据的MIME类型

要特别注意的是，URI中也可以携带数据的类型信息，所以在匹配过程中，还需要考虑URI中指定的数据类型。

提示关于具体的匹配流程，请读者务必阅读SDK docs/guide/topics/intents/intents-filters.html中的说明。

### 4.5.2  Activity信息的管理
前面在介绍PKMS扫描APK时提到，PKMS将解析得到的Package私有的Activity信息加入到自己的数据结构mActivities中保存。先来回顾一下代码：

[-->PacakgeManagerService.java::scanPackageLI函数]
```java
......//此时APK文件已经解析完成

N =pkg.activities.size();//取出该APK中包含的Activities信息

r =null;

for (i=0; i<N; i++) {

    PackageParser.Activity a = pkg.activities.get(i);

    a.info.processName = fixProcessName(pkg.applicationInfo.processName,

            a.info.processName,pkg.applicationInfo.uid);

    mActivities.addActivity(a,"activity");//①加到mActivities中保存

}
```

上面的代码中有两个比较重要的数据结构，如图4-11所示。



![图4-11  相关数据结构示意图](/images/understand2/4-11.png)

结合代码，由图4-11可知：

-  mActivities为ActivityIntentResolver类型，是PKMS的成员变量，用于保存系统中所有与Activity相关的信息。此数据结构内部有一个mActivities变量，它以ComponetName为Key，保存PackageParser.Activity对象

-  从APK中解析得到的所有和Activity相关的信息（包括在XML中声明的IntentFilter标签）都由PacakgeParser.Activity来保存。

前面代码中调用addActivity函数完成了私有信息的公有化。addActivity函数的代码如下：

[-->PacakgeManagerService.java::ActivityIntentResolver.addActivity]
```java
public final voidaddActivity(PackageParser.Activity a, String type) {

    finalboolean systemApp = isSystemApp(a.info.applicationInfo);

    //将Component和Activity保存到mActivities中

    mActivities.put(a.getComponentName(), a);

    finalint NI = a.intents.size();

    for(int j=0; j<NI; j++) {

        //ActivityIntentInfo存储的就是XML中声明的IntentFilter信息

        PackageParser.ActivityIntentInfo intent = a.intents.get(j);

        if(!systemApp && intent.getPriority() > 0 &&"activity".equals(type)) {

            //非系统APK的priority必须为0。后续分析中将介绍priority的作用

            intent.setPriority(0);

        }

        addFilter(intent);//接下来将分析这个函数

    }

}
 ```

下面来分析addFilter函数，这里涉及较多的复杂数据结构，代码如下：

[-->IntentResolver.java::IntentResolver.addFilter]
```java
public void addFilter(F f) {

    ......

        mFilters.add(f);//mFilters保存所有IntentFilter信息

    //除此之外，为了加快匹配工作的速度，还需要分类保存IntentFilter信息

    //下边register_xxx函数的最后一个参数用于打印信息

    intnumS = register_intent_filter(f, f.schemesIterator(),

            mSchemeToFilter,"      Scheme: ");

    intnumT = register_mime_types(f, "     Type: ");

    if(numS == 0 && numT == 0) {

        register_intent_filter(f, f.actionsIterator(),

                mActionToFilter,"      Action: ");

    }

    if(numT != 0) {

        register_intent_filter(f, f.actionsIterator(),

                mTypedActionToFilter, "     TypedAction: ");

    }

}
 ```

正如代码注释中所说，为了加快匹配工作的速度，这里使用了泛型编程并定义了较多的成员变量。下面总结一下这些变量的作用（注意，除mFilters为HashSet<F>类型外，其他成员变量的类型都是HashMap<String, ArrayList<F>>，其中F为模板参数）。

-  mSchemeToFilter：用于保存URI中与schema相关的IntentFilter信息。

-  mActionToFilter：用于保存仅设置Action条件的IntentFilter信息。

-  mTypedActionToFilter：用于保存既设置了Action又设置了Data的MIME类型的IntentFilter信息。

-  mFilters：用于保存所有IntentFilter信息

-  mWildTypeToFilter：用于保存设置了Data类型类似“image/*”的IntentFilter，但是设置MIME类型类似“Image/jpeg”的不算在此类。

-  mTypeToFilter：除了包含mWildTypeToFilter外，还包含那些指明了Data类型为确定参数的IntentFilter信息，例如“image/*”和”image/jpeg“等都包含在mTypeToFilter中。

-  mBaseTypeToFilter：包含MIME中Base 类型的IntentFilter信息，但不包括Sub type为“*”的IntentFilter。

不妨举个例子来说明这些变量的用法。

假设，在XML中声明一个IntentFilter，代码如下：

<intent-filter android:label="test">

   <actionandroid:name="android.intent.action.VIEW" />

   dataandroid:mimeType="audio/*" android:scheme="http"

 </intent-filter>

那么：

-  在mTypedActionToFilter中能够以“android.intent.action.VIEW”为key找到该IntentFilter。

-  在mWildTypeToFilter和mTypeToFilter中能够以“audio”为key找到该IntentFilter。

-  在mSchemeToFilter中能够以”http“为key找到该IntentFilter。

下面来分析Intent匹配查询工作。

### 4.5.2  Intent 匹配查询分析
#### 1.  客户端查询
客户端通过ApplicationPackageManager输出的queryIntentActivities函数向PKMS发起一次查询请求，代码如下：

[-->ApplicationPackageManager.java::queryIntentActivities]
```java
public List<ResolveInfo>queryIntentActivities(Intent intent, int flags) {

try {

return mPM.queryIntentActivities(

intent,//下面这句话很重要

intent.resolveTypeIfNeeded(mContext.getContentResolver()),

flags);

}......

}
```

如果Intent的Data包含一个URI，那么就需要查询该URI的提供者（即ContentProvider）以取得该数据的数据类型。读者可自行阅读resolveTypeIfNeeded函数的代码。

另外，flags参数目前有3个可选值，分别是MATCH_DEFAULT_ONLY、GET_INTENT_FILTERS和GET_RESOLVED_FILTER。详细信息读者可查询SDK相关文档。

下面来看PKMS对匹配查询的处理。

#### 2.  queryIntentActivities分析
该函数代码如下：

[-->PacakgeManagerService.java::queryIntentActivities]
```java
public List<ResolveInfo>queryIntentActivities(Intent intent,

String resolvedType, int flags) {

final ComponentName comp = intent.getComponent();

if(comp != null) {

//Explicit的Intents，直接根据component得到对应的ActivityInfo

final List<ResolveInfo> list = new ArrayList<ResolveInfo>(1);

final ActivityInfo ai = getActivityInfo(comp, flags);

if (ai != null) {

    final ResolveInfo ri = new ResolveInfo();

    //ResovlerInfo的activityInfo指向查询得到的ActivityInfo

    ri.activityInfo = ai;

    list.add(ri);

}

return list;

}



synchronized (mPackages) {

    final String pkgName = intent.getPackage();

    if (pkgName == null) {

        //Implicit Intents，我们重点分析此中情况

        return mActivities.queryIntent(intent, resolvedType, flags);

    }

    //Intent指明了PackageName，比Explicit Intents情况差一点

    final PackageParser.Package pkg = mPackages.get(pkgName);

    if (pkg != null) {

        //其实是从该Package包含的Activities中进行匹配查询

        return mActivities.queryIntentForPackage(intent, resolvedType,

                flags, pkg.activities);

    }

    return new ArrayList<ResolveInfo>();

}

}
```

上边代码分三种情况：

-  如果Intent指明了Component，则直接查询该Component对应的ActivityInfo。

-  如果Intent指明了Package名，则根据Package名找到该Package，然后再从该Package包含的Activities中进行匹配查询。

-  如果上面条件都不满足，则需要在全系统范围内进行匹配查询，这就是queryIntent的工作。

queryIntent函数的代码如下：
```java
public List<ResolveInfo> queryIntent(Intentintent, String resolvedType, intflags) {

    mFlags =flags;

    //调用基类的queryIntent函数

    returnsuper.queryIntent(intent, resolvedType,

            (flags&PackageManager.MATCH_DEFAULT_ONLY) != 0);

}
 ```

[-->IntentResolver.java::queryIntent]
```java
public List<R> queryIntent(Intent intent,String resolvedType,
        booleandefaultOnly) {

    Stringscheme = intent.getScheme();

    ArrayList<R> finalList = new ArrayList<R>();

    //最多有四轮匹配工作要做

    ArrayList<F> firstTypeCut = null;

    ArrayList<F> secondTypeCut = null;

    ArrayList<F> thirdTypeCut = null;

    ArrayList<F> schemeCut = null;

    //下面将设置各轮校验者

    if(resolvedType != null) {

        intslashpos = resolvedType.indexOf('/');

        if(slashpos > 0) {

            final String baseType = resolvedType.substring(0, slashpos);

            if (!baseType.equals("*")) {

                if (resolvedType.length() != slashpos+2

                        || resolvedType.charAt(slashpos+1) != '*') {

                    firstTypeCut =mTypeToFilter.get(resolvedType);

                    secondTypeCut =mWildTypeToFilter.get(baseType);

                }......//略去一部分内容

            }

        }



        if(scheme != null) {

            schemeCut = mSchemeToFilter.get(scheme);

        }

        if(resolvedType == null && scheme == null && intent.getAction()!= null)

        {

            //看来action的filter优先级最低

            firstTypeCut = mActionToFilter.get(intent.getAction());

        }

        //FastImmutableArraySet是一种特殊的数据结构，用于保存该Intent中携带的

        //Category相关的信息。

        FastImmutableArraySet<String>categories = getFastIntentCategories(intent);

        if(firstTypeCut != null) {

            //匹配查询，第一轮过关斩将

            buildResolveList(intent, categories, debug, defaultOnly,

                    resolvedType, scheme, firstTypeCut,finalList);

        }

        if(secondTypeCut != null) {

            buildResolveList(intent, categories, debug, defaultOnly,

                    resolvedType, scheme, secondTypeCut, finalList);

        }

        if(thirdTypeCut != null) {

            buildResolveList(intent, categories, debug, defaultOnly,

                    resolvedType, scheme, thirdTypeCut, finalList);

        }

        if(schemeCut != null) {

            //生成符合schemeCut条件的finalList

            buildResolveList(intent, categories, debug, defaultOnly,

                    resolvedType, scheme, schemeCut, finalList);

        }

        //将匹配结果按Priority的大小排序

        sortResults(finalList);

        returnfinalList;

    }
```

在以上代码中设置了最多四轮匹配关卡，然后逐一执行匹配工作。具体的匹配代码由buildResolveList完成，无非是一项查找工作而已。此处就不再深究细节了，建议读者在研究代码时以目的为导向，不宜深究其中的数据结构。

### 4.5.3 queryIntentActivities总结
本节分析了queryIntentActivities函数的实现，其功能很简单，就是进行Intent匹配查询。一路走来，相信读者也感觉到旅程并不轻松，主要原因是涉及的数据结构较多，让人有些头晕。这里，再次建议读者不要在数据结构上花太多时间，最好结合SDK中的文档说明来分析相关代码。

## 4.6  installd及UserManager介绍
### 4.6.1  installd介绍
在前面对PKMS构造函数分析时介绍过一个Installer类型的对象mInstaller，它通过socket和后台服务installd交互，以完成一些重要操作。这里先回顾一下PKMS中mInstaller的调用方法：

```java
mInstaller = new Installer();//创建一个Installer对象

//对某个APK文件进行dexopt优化

mInstaller.dexopt(paths[i], Process.SYSTEM_UID,true);

//扫描完系统Package后，调用moveFiles函数

mInstaller.moveFiles();

//当存储空间不足时，调用该函数清理存储空间

mInstaller.freeCache(freeStorageSize);
```

Installer的种种行为都和其背后的installd有关。下面来分析installd。

#### 1.  installd概貌
installd是一个native进程，代码非常简单，其功能就是启动一个socket，然后处理来自Installer的命令，其代码如下：

[-->installd.c]
```c
int main(const int argc, const char *argv[]) {

    charbuf[BUFFER_MAX];

    structsockaddr addr;

   socklen_t alen;

    intlsocket, s, count;

    //

  if (初始化全局变量,如果失败则退出) {

    initialize_globals();

    initialize_directories();

        ......

    }

    ......

    lsocket= android_get_control_socket(SOCKET_PATH);

  

   listen(lsocket, 5);

   fcntl(lsocket, F_SETFD, FD_CLOEXEC);

 

    for (;;){

        alen= sizeof(addr);

        s =accept(lsocket, &addr, &alen);

       fcntl(s, F_SETFD, FD_CLOEXEC);

        for(;;) {

           unsigned short count;

           readx(s, &count, sizeof(count));

           //执行installer发出的命令，具体解释见下文

            execute(s, buf);

        }

       close(s);

    }

 

    return0;

}
```

installd支持的命令及参数信息都保存在数据结构cmds中，代码如下：

[-->installd.c]
```c
struct cmdinfo cmds[] = {//第二个变量是参数个数，第三个参数是命令响应函数

    {"ping",                 0,do_ping },

    {"install",              3,do_install },

    {"dexopt",               3,do_dexopt },

    {"movedex",              2,do_move_dex },

    {"rmdex",                1,do_rm_dex },

    {"remove",               2,do_remove },

    {"rename",               2, do_rename },

    {"freecache",            1,do_free_cache },

    {"rmcache",              1,do_rm_cache },

    {"protect",              2,do_protect },

    {"getsize",              4,do_get_size },

    {"rmuserdata",           2,do_rm_user_data },

    {"movefiles",            0,do_movefiles },

    {"linklib",              2,do_linklib },

    {"unlinklib",            1,do_unlinklib },

    {"mkuserdata",           3,do_mk_user_data },

    {"rmuser",               1,do_rm_user },

};
```

下面来分析相关的几个命令。

#### 2.  dexOpt命令分析
PKMS在需要对一个APK或jar包做dex优化时，会发送dexopt命令给installd，相应的处理函数为do_dexopt，代码如下：

[-->installd.c]
```c
static int do_dexopt(char **arg, charreply[REPLY_MAX])
{
     returndexopt(arg[0], atoi(arg[1]), atoi(arg[2]));
}
```
[-->commands.c]
```c
int dexopt(const char *apk_path, uid_t uid, intis_public)
{

    structutimbuf ut;

    structstat apk_stat, dex_stat;

    chardex_path[PKG_PATH_MAX];

    chardexopt_flags[PROPERTY_VALUE_MAX];

    char*end;

    int res,zip_fd=-1, odex_fd=-1;

    ......

    //取出系统级的dexopt_flags参数

   property_get("dalvik.vm.dexopt-flags", dexopt_flags,"");

 

   strcpy(dex_path, apk_path);

    end =strrchr(dex_path, '.');

    if (end!= NULL) {

       strcpy(end, ".odex");

        if(stat(dex_path, &dex_stat) == 0) {

           return 0;

        }

    }

    //得到一个字符串，用于描述dex文件名，位于/data/dalvik-cache/下

    if(create_cache_path(dex_path, apk_path)) {

       return -1;

    }

 

   memset(&apk_stat, 0, sizeof(apk_stat));

   stat(apk_path, &apk_stat);

 

    zip_fd =open(apk_path, O_RDONLY, 0);

    ......

   unlink(dex_path);

    odex_fd= open(dex_path, O_RDWR | O_CREAT | O_EXCL, 0644);

    ......

    pid_tpid;

    pid =fork();

    if (pid== 0) {

        ......//uid设置

        //创建一个新进程，然后对exec dexopt进程进行dex优化

        run_dexopt(zip_fd,odex_fd, apk_path, dexopt_flags);

       exit(67);  

} else {

        //installd将等待dexopt完成优化工作

        res= wait_dexopt(pid, apk_path);

        ......

    }

    ......//资源清理

    return-1;

}
```

让人大跌眼镜的是，dex优化工作竟然由installd委派给dexopt进程来实现。dex优化后会生成一个dex文件，一般位于/data/dalvik-cache/目录中。这里给出一个示例，如图4-12所示。



![图4-12  dex文件示例](/images/understand2/4-12.png)

提示 dexopt进程由android源码/dalvik/dexopt/OptMain.cpp定义。感兴趣的读者可深入研究dex优化的工作原理。

#### 3.  movefiles命令分析
PKMS扫描完系统Package后，将发送该命令给installd，相应处理函数的代码如下：

[-->installd.c]
```c
static int do_movefiles(char **arg, charreply[REPLY_MAX])

{

    returnmovefiles();

}

[-->commands.c]

int movefiles()

{

    DIR *d;

    int dfd,subfd;

    structdirent *de;

    structstat s;

    charbuf[PKG_PATH_MAX+1];

    intbufp, bufe, bufi, readlen;

 

    charsrcpkg[PKG_NAME_MAX];

    chardstpkg[PKG_NAME_MAX];

    charsrcpath[PKG_PATH_MAX];

    chardstpath[PKG_PATH_MAX];

    intdstuid=-1, dstgid=-1;

    inthasspace;

    //打开/system/etc/updatecmds/目录

    d =opendir(UPDATE_COMMANDS_DIR_PREFIX);

    if (d ==NULL) {

        gotodone;

    }

    dfd =dirfd(d);

       while((de = readdir(d))) {

        ......//解析该目录下的文件，然后执行对应操作

   }

   closedir(d);

done:

    return0;

}
```

先来看/system/etc/updatecmds/目录下到底是什么文件，这里给出一个示例，如图4-13所示。



![图4-13  movefiles示例](/images/understand2/4-13.png)

以图4-13中最后两行为例，movefiles将把com.google.android.gsf下的databases目录转移到com.andorid.providers.im下。从文件中的注释可知，movefiles的功能和系统升级有关。

#### 4.  doFreeCache
第3章介绍了DeviceStorageMonitorService，当系统空间不够时，DSMS会调用PKMS的freeStorageAndNotify函数进行空间清理。该工作真正的实施者是installd，相应的处理命令为do_free_cache，其代码如下：

[-->installd.c]
```c
static int do_free_cache(char **arg, charreply[REPLY_MAX])

{

    returnfree_cache((int64_t)atoll(arg[0]));

}

[-->commands.c]

int free_cache(int64_t free_size)

{

    constchar *name;

    int dfd,subfd;

    DIR *d;

    structdirent *de;

    int64_tavail;

 

    avail =disk_free();//获取当前系统的剩余空间大小

    if(avail < 0) return -1;

    if(avail >= free_size) return 0;

    d =opendir(android_data_dir.path);//打开/data/目录

    dfd =dirfd(d);

    while((de = readdir(d))) {

        if (de->d_type != DT_DIR) continue;

        name= de->d_name;

       ......//略过.和..文件

       subfd = openat(dfd, name, O_RDONLY | O_DIRECTORY);

        //删除/data及各级子目录中的cache文件夹

       delete_dir_contents_fd(subfd, "cache");

       close(subfd);

       ......//如果剩余空间恢复正常，则返回

    }

   closedir(d);

    return-1;//清理空间后，仍然不满足要求

}
```

installd的介绍就到此为止，这部分内容比较简单，读者完全可自行深入研究。

### 4.6.2  UserManager介绍
UserManager是Andorid 4.0新增的一个功能，其作用是管理手机上的不同用户。这一点和PC上的Windows系统比较相似，例如，在Windows上安装程序时，都会提示是安装给本人使用还是安装给系统所有用户使用。非常遗憾的是，在目前的Andorid版本中，该功能尚未完全实现，在SDK中也没有相关说明。不过从现有代码中，也能发现一些蛛丝马迹。

提示 小米手机的访客模式和UserManager比较相似。

#### 1.  UserManager构造函数分析
在PKMS中，创建UserManager调用的代码如下：

//mUserAppDataDir指向/data/app。该目录中包含的是非系统APK文件

mUserManager = new UserManager(mInstaller,mUserAppDataDir);

[-->UserManager.java]
```java
public UserManager(Installer installer, FilebaseUserPath) {

    this(Environment.getDataDirectory(), baseUserPath);

    mInstaller = installer;

}

UserManager(File dataDir, File baseUserPath) {

    //mUsersDir指向/data/system/users目录

    mUsersDir = new File(dataDir, USER_INFO_DIR);

    mUsersDir.mkdirs();//创建该目录

    mBaseUserPath = baseUserPath;

    FileUtils.setPermissions(mUsersDir.toString(),

            FileUtils.S_IRWXU|FileUtils.S_IRWXG

            |FileUtils.S_IROTH|FileUtils.S_IXOTH,

            -1, -1);

    //mUserListFile指向/data/system/user/userlist.xml

    mUserListFile = new File(mUsersDir, USER_LIST_FILENAME);

    readUserList();//解析userlist.xml文件

}
```

此处不深入readUserList代码了，只介绍其内部工作流程。

-  userlist.xml保存每个用户的id。

-  readUserList到/data/system/users下解析id.xml，将最终得到的信息保存在UserInfo对象中。

原来用户信息由UserInfo表达，下面是UserInfo的定义。

[-->UserInfo]
```java
public class UserInfo implements Parcelable {

    //主用户，全系统只能有一个这样的用户

    publicstatic final int FLAG_PRIMARY = 0x00000001;



    //管理员，可以创建、删除其他用户信息

    publicstatic final int FLAG_ADMIN   =0x00000002;

    //访客用户

    publicstatic final int FLAG_GUEST   =0x00000004;

    publicint id; //id

    publicString name;//用户名

    publicint flags; //属性标志

    ......//其他函数

}
```

UserInfo信息比较简单，笔者觉得UserManager的功能暂时还不能企业用户的需求。感兴趣的读者不妨关注Android未来版本在此方面的变化。

#### 2.  installPackageForAllUsers分析
PKMS在扫描非系统APK的时候，每扫描完一个APK都会调用installPackageForAllUsers，调用代码如下：

mUserManager.installPackageForAllUsers(pkgName,pkg.applicationInfo.uid);

[-->UserManager.java::installPackageForAllUsers]
```java
public void installPackageForAllUsers(StringpackageName, int uid) {

    for (intuserId : mUserIds) {

        if(userId == 0)

            continue;

        //向installd发送命令，其中getUid将组合userId和uid为一个整型值

        //installd将在/data/对应user/目录下创建相应的package子目录

        mInstaller.createUserData(packageName, PackageManager.getUid(userId,uid),

                userId);

    }

}
```
 

## 4.7  本章学习指导
PKMS是本书分析的第一个重要核心服务，其中的代码量，关联的知识点，涉及的数据结构都比较多。这里提出一些学习建议供读者参考。

-  从工作流程上看，PKMS包含几条重要的主线。一条是PKMS自身启动时构造函数的工作流程，另外几条和APK安装、卸载相关。每一条主线的难度都比较大，读者可结合日常工作的需求进行单独研究，例如研究如何加快构造函数的执行时间等。

-  从数据结构上看，PKMS涉及非常多的数据类型。如果对每个数据结构进行孤立分析，很容易陷入不可自拔的状态。笔者建议不妨跳出各种数据结构的具体形态，只从目的及功能角度去考虑。这里需要读者仔细查看前面的重要数据结构及说明示意图。

另外，由于篇幅所限，本章还有一些内容并没有涉及，需要读者在学习本章内容的基础上自行研究。这些内容包括：

-  APK安装在SD卡，以及APK从内部存储转移到SD卡的流程。

-  和Package相关的内容，例如签名管理、dex优化等。

## 4.8  本章小结
本章对PackageManagerService进行了较深入的分析。首先分析了PKMS创建时构造函数的工作流程；接着以APK安装为例，较详细地讲解了这个复杂的处理流程；然后又介绍了PKMS另外一项功能，即根据Intent查找匹配的Activities；最后，介绍了与installd和UserManager有关的知识。


[①] Signature和Android安全机制有关。本系列书后续拟考虑编写有关Android安全方面的专题卷。
