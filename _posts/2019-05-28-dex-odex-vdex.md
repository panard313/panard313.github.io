---
layout: article
title: apk dex vdex odex art 区别
key: 20190529
tags:
  - dex
  - cx
lang: zh-Hans
---

# apk dex vdex odex art 区别

## apk

- APK(Android package)：android安装包，由aapt（Android Assert Packaging Tool）把AndroidManifest.xml、资源文件、dex（二进制字节码）文件组合而成。将apk文件修改扩展名为rar，然后解压可已看到目录如下：
apk

- METE-INF:存放应用签名证书等信息

- res:存放资源文件

- AndroidManifest.xml：应用配置文件

- classes.dex:应用程序二进制字节码文件

- resources.arsc：二进制资源文件

## dex

dex（Dalvik VM Excutors）：Dalvik虚拟机执行程序，执行前需要优化

image

## vdex

android O 新增的格式包，dex代码 直接转化的 可执行二进制码 文件：

- 1.第一次开机就会生成在/system/app/<packagename>/oat/ 下；

- 2.在系统运行过程中，虚拟机将其 从 “/system/app” 下 copy 到 “/data/davilk-cache/” 下

## odex

在Android N 之前，Dalvik虚拟机执行程序dex文件前，系统会对dex文件做优化，生成可执行文件odex，保存到data/dalvik-cache目录，最后把apk文件中的dex文件删除。

优点：

- 1. 减少了启动时间（省去了系统第一次启动应用时从apk文件中读取dex文件，并对dex文件做优化的过程）和对RAM的占用（apk文件中的dex如果不删除，同一个应用就会存在两个dex文件：apk中和data/dalvik-cache目录下）。

- 2.防止第三方用户反编译系统的软件（odex文件是跟随系统环境变化的，改变环境会无法运行；而apk文件中又不包含dex文件，无法独立运行）。

在Android O 之后，odex 是从vdex 这个文件中提取了部分模块生成的一个新的可执行二进制码文件 ， odex 从vdex 中提取后，vdex 的大小就减少了。

- 1.第一次开机就会生成在/system/app/<packagename>/oat/ 下

- 2.在系统运行过程中，虚拟机将其 从 “/system/app” 下 copy 到 “/data/davilk-cache/” 下

- 3.odex + vdex = apk 的全部源码 （vdex 并不是独立于odex 的文件 odex + vdex 才代表一个apk ）

## art

odex 进行优化 生成的 可执行二进制码 文件，主要是apk 启动的常用函数相关地址的记录，方便寻址相关； 通常会在data/dalvik-cache/保存常用的jar包的相关地址记录。

- 1.第一次开机不会生成在/system/app/<packagename>/oat/ 下，以后也不会；

- 2.odex 文件在运行时，虚拟机会计算函数调用频率，进行函数地址的修改；

- 3.最后在/data/davilk-cache/ 由虚拟机生成；

- 4.生成art 文件后，/system/app 下的odex 和 vdex 会无效，即使你删除，apk也会正常运行

- 5.push 一个新的apk file 覆盖之前/system/app 下apk file ，会触发PKMS 扫描时下发force_dex flag ，强行生成新的vdex 文件 ，覆盖之前的vdex 文件，由于某种机制，这个新vdex 文件会copy到/data/dalvik-cache/下，于是art 文件也变化了。

## oat

ART虚拟机使用的是oat文件，oat文件是一种Android私有ELF文件格式，它不仅包含有从DEX文件翻译而来的本地机器指令，还包含有原来的DEX文件内容。APK在安装的过程中，会通过dex2oat工具生成一个OAT文件。对于apk来说，oat文件实际上就是对odex文件的包装，即oat=odex，而对于一些framework中的一些jar包，会生成相应的oat尾缀的文件，如system@framework@boot-telephony-common.oat。

## QA

    Android 5.0开始，默认已经使用ART，弃用Dalvik了，app会在安装时被编译成OAT文件，（ART上运行的格式）ODEX还有什么用呢？ Google权威的回答：

Dex file compilation uses a tool called dex2oat and takes more time than dexopt. The increase in time varies, but 2-3x increases in compile time are not unusual. For example, apps that typically take a second to install using dexopt might take 2-3 seconds.

DEX转换成OAT的这个过程是5.0以上系统用户在安装程序或是刷入ROM、增量更新后首次启动时必然执行的。 

按照Google的说法，相比做过ODEX优化，未做过优化的DEX转换成OAT要花费更长的时间，比如2-3倍。 

比如安装一个odex优化过的程序假设需要1秒钟，未做过优化的程序就需要2~3秒。

由此可见，虽然dalvik被弃用了，但ODEX优化在Android 5.0系统以上依旧起着作用。 

ODEX优化事实上是由一个叫做WITH_DEXPREOPT的参数控制的，开启该参数后，会对APK、JAR以及内核镜像进行优化。

其中，针对APK和JAR的最直观的优化体现就是，程序的dex被转换成odex。

    Android8.0引入的vdex？

https://www.zhihu.com/question/275955357/answer/383865933



# self

### odex

dex2oat 生成，由 --multi-image选项控制 p451

核心库一般生成art， app不一定生成art

### oat

一种定制化的elf文件， 实际上是以elf格式封装了oat信息, 将oat的信息存储在.rodata和.text section中。 而oat信息又以oat格式来组织. p494

p544 compile总结， dex2oat中dex字结码， 编译

p544 oat和art文件格式介绍

p553 art文件可以看成是直接保存的对象

p554 oatdump

p566 提前创建的object

p599 JIT



## odex优化的地方

### 1.首次开机或者升级

在SystemServer.java 中有mPackageManagerService.updatePackagesIfNeeded() 
这里先列举流程，具体步骤有空再贴上 
updatePackagesIfNeeded->performDexOptUpgrade->performDexOptTraced->performDexOptInternal->performDexOptInternalWithDependenciesLI->PackageDexOptimizer.performDexOpt->performDexOptLI->dexOptPath->Installer.dexopt->InstalldNativeService.dexopt->dexopt.dexopt

### 2.安装应用：

在PKMS.installPackageLI函数中有： 
```java
mPackageDexOptimizer.performDexOpt(pkg, pkg.usesLibraryFiles, 
    null /* instructionSets /, false / checkProfiles */, 
    getCompilerFilterForReason(REASON_INSTALL), 
    getOrCreateCompilerPackageStats(pkg), 
    mDexManager.isUsedByOtherApps(pkg.packageName));
```

### 3.IPackageManager.aidl提供了performDexOpt方法

在PKMS中有实现的地方，但是没找到调用的地方

### 4.IPackageManager.aidl提供了performDexOptMode方法

在PKMS中有实现的地方，在PackageManagerShellCommand中会被调用，应该是提供给shell命令调用

### 5.OTA升级后：

在SystemServer.java 中有OtaDexoptService.main(mSystemContext, mPackageManagerService);
```java
public static OtaDexoptService main(Context context,
        PackageManagerService packageManagerService) {
    OtaDexoptService ota = new OtaDexoptService(context, packageManagerService);
    ServiceManager.addService("otadexopt", ota);

    // Now it's time to check whether we need to move any A/B artifacts.
    ota.moveAbArtifacts(packageManagerService.mInstaller);

    return ota;
}

private void moveAbArtifacts(Installer installer) {
    if (mDexoptCommands != null) {
        throw new IllegalStateException("Should not be ota-dexopting when trying to move.");
    }
    //如果不是升级上来的，就return掉
    if (!mPackageManagerService.isUpgrade()) {
        Slog.d(TAG, "No upgrade, skipping A/B artifacts check.");
        return;
    }

        installer.moveAb(path, dexCodeInstructionSet, oatDir);
}
```

moveAbArtifacts函数的逻辑： 
- 1.判断是否升级 
- 2.判断扫描过的package是否有code，没有则跳过 
- 3.判断package的code路径是否为空，为空则跳过 
- 4.如果package的code在system或者vendor目录下，跳过 
- 5.满足上述条件，调用Installer.java中的moveAb方法 

最终是调用dexopt.cpp的move_ab方法

OtaDexoptService也提供给shell命令一些方法来调用

### 6.在系统空闲的时候：

是通过BackgroundDexOptService来实现的，BackgroundDexOptService继承了JobService 

这里启动了两个任务 

- 1.开机的时候执行odex优化 JOB_POST_BOOT_UPDATE 

执行条件：开机一分钟内 

- 2.在系统休眠的时候执行优化 JOB_IDLE_OPTIMIZE 

执行条件：设备处于空闲，插入充电器，且每隔一分钟或者一天就检查一次（根据debug开关控制）



## jvm dalvik art

[https://blog.csdn.net/jason0539/article/details/50440669](https://blog.csdn.net/jason0539/article/details/50440669)

#### JVM 
JVM(Java Virtual Machine): 编译的是.class文件，基于栈的结构

JAVA程序经过编译，生成JAVA字节码保存在class文件中，JVM通过解码class文件中的内容来运行程序

#### Dalvik
而Dalvik虚拟机运行的是Dalvik字节码，所有的Dalvik字节码由JAVA字节码转换而来，并被打包到一个DEX（Dalvik Executable）可执行文件中，DVM通过解释DEX文件来执行这些字节码。

class文件中包含多个不同的方法签名，如果A类文件引用B类文件中的方法，方法签名也会被复制到A类文件中（在虚拟机加载类的连接阶段将会使用该签名链接到B类的对应方法），也就是说，多个不同的类会同时包含相同的方法签名，同样地，大量的字符串常量在多个类文件中也被重复使用，这些冗余信息会直接增加文件的体积，而JVM在把描述类的数据从class文件加载到内存时，需要对数据进行校验、转换解析和初始化，最终才形成可以被虚拟机直接使用的JAVA类型，因为大量的冗余信息，会严重影响虚拟机解析文件的效率。
为了减小执行文件的体积，安卓使用Dalvik虚拟机，SDK中有个dx工具负责将JAVA字节码转换为Dalvik字节码，dx工具对JAVA类文件重新排列，将所有JAVA类文件中的常量池分解，消除其中的冗余信息，重新组合形成一个常量池，所有的类文件共享同一个常量池，使得相同的字符串、常量在DEX文件中只出现一次，从而减小了文件的体积。

#### ART Android Runtime
**Dalvik虚拟机执行的是dex字节码，ART虚拟机执行的是本地机器码**
Dalvik执行的是dex字节码，依靠JIT编译器去解释执行，运行时动态地将执行频率很高的dex字节码翻译成本地机器码，然后在执行，但是将dex字节码翻译成本地机器码是发生在应用程序的运行过程中，并且应用程序每一次重新运行的时候，都要重新做这个翻译工作，因此，及时采用了JIT，Dalvik虚拟机的总体性能还是不能与直接执行本地机器码的ART虚拟机相比。

应用程序仍然是一个包含dex字节码的apk文件。所以在安装应用的时候，dex中的字节码将被编译成本地机器码，之后每次打开应用，执行的都是本地机器码。

ART优点：
①系统性能显著提升
②应用启动更快、运行更快、体验更流畅、触感反馈更及时
③续航能力提升
④支持更低的硬件

ART缺点
①更大的存储空间占用，可能增加10%-20%
②更长的应用安装时间


## links

[https://blog.csdn.net/jsqfengbao/article/details/52103439](https://blog.csdn.net/jsqfengbao/article/details/52103439)

[https://blog.csdn.net/long375577908/article/details/78190422](https://blog.csdn.net/long375577908/article/details/78190422)

[https://blog.csdn.net/lilian0118/article/details/25965171](https://blog.csdn.net/lilian0118/article/details/25965171)

[APK安装流程详解15——PMS中的新安装流程下(装载)补充](https://www.jianshu.com/p/6f8fc521512e)

[Android中dex文件的加载与优化流程](https://blog.csdn.net/jsqfengbao/article/details/52103439)

[dexopt优化和验证Dalvik](https://blog.csdn.net/lilian0118/article/details/25965171)

[dexopt 与 dex2oat 区别](https://www.jianshu.com/p/26a82119da49)

[dex2oat的原理及慢的原因](https://blog.csdn.net/long375577908/article/details/78190422)

[JVM、DVM(Dalvik VM)和ART虚拟机对比](https://blog.csdn.net/evan_man/article/details/52414390)

[Android可执行文件之谜 - DEX与ODEX, OAT与ELF](http://www.imooc.com/article/91519)

[Android运行时ART加载OAT文件的过程分析](https://blog.csdn.net/luoshengyang/article/details/39307813)

[Android运行时ART简要介绍和学习计划](https://blog.csdn.net/luoshengyang/article/details/39256813)

[Dalvik和ART运行时环境的区别](https://blog.csdn.net/watermusicyes/article/details/50526814)

[Android ART运行时无缝替换Dalvik虚拟机的过程分析](https://blog.csdn.net/luoshengyang/article/details/18006645)

[Android 系统（36）---Android O、N版本修改dex2oat编译选项](https://blog.csdn.net/zhangbijun1230/article/details/80026887)

[Android 系统（90）---JIT 编译器 --图](https://blog.csdn.net/zhangbijun1230/article/details/80590208)

[ART 的 interpret-only模式源码及调用流程 ＆ QuickCompiler后端调用流程](https://blog.csdn.net/Sumin_fushengruocha/article/details/51147776)

[Android 系统（89）--- 配置ART](https://blog.csdn.net/zhangbijun1230/article/details/80589875)

[Dalvik,ART与ODEX相爱相生](https://www.jianshu.com/p/389911e2cdfb)

[Implementing ART Just-In-Time (JIT) Compiler](https://medium.com/@erpragatisingh/implementing-art-just-in-time-jit-compiler-2b562f769f59)


## question

- oat art

- odex 弃用

- vdex

首次安装后已提取并校验（extracted， verified）过的dex文件，避免再次优化时进行重复的提取和校验

- cdex

Android 9（Pie）版本推出了一种新型的Dex文件，即Compact Dex（Cdex）。Cdex是一种ART内部文件格式，它压缩各种Dex数据结构（例如方法头）并对多索引文件中的常见数据blob（例如字符串）进行重复数据删除。来自输入应用程序的Dex文件的重复数据删除数据存储在Vdex容器的共享部分中。

- 开机一分钟执行

- jit

- speed-profile





## ppt

### JVM Dalvik ART
JVM(Java Virtual Machine): 编译的是.class文件，基于栈的结构

JAVA程序经过编译，生成JAVA字节码保存在class文件中，JVM通过解码class文件中的内容来运行程序


而Dalvik虚拟机运行的是Dalvik字节码，所有的Dalvik字节码由JAVA字节码转换而来，并被打包到一个DEX（Dalvik Executable）可执行文件中，DVM通过解释DEX文件来执行这些字节码。

class文件中包含多个不同的方法签名，如果A类文件引用B类文件中的方法，方法签名也会被复制到A类文件中（在虚拟机加载类的连接阶段将会使用该签名链接到B类的对应方法），也就是说，多个不同的类会同时包含相同的方法签名，同样地，大量的字符串常量在多个类文件中也被重复使用，这些冗余信息会直接增加文件的体积，而JVM在把描述类的数据从class文件加载到内存时，需要对数据进行校验、转换解析和初始化，最终才形成可以被虚拟机直接使用的JAVA类型，因为大量的冗余信息，会严重影响虚拟机解析文件的效率。
为了减小执行文件的体积，安卓使用Dalvik虚拟机，SDK中有个dx工具负责将JAVA字节码转换为Dalvik字节码，dx工具对JAVA类文件重新排列，将所有JAVA类文件中的常量池分解，消除其中的冗余信息，重新组合形成一个常量池，所有的类文件共享同一个常量池，使得相同的字符串、常量在DEX文件中只出现一次，从而减小了文件的体积。


Dalvik执行的是dex字节码，依靠JIT编译器去解释执行，运行时动态地将执行频率很高的dex字节码翻译成本地机器码，然后在执行，但是将dex字节码翻译成本地机器码是发生在应用程序的运行过程中，并且应用程序每一次重新运行的时候，都要重新做这个翻译工作，因此，及时采用了JIT，Dalvik虚拟机的总体性能还是不能与直接执行本地机器码的ART虚拟机相比。
**Dalvik虚拟机执行的是dex字节码，ART虚拟机执行的是本地机器码**

应用程序仍然是一个包含dex字节码的apk文件。所以在安装应用的时候，dex中的字节码将被编译成本地机器码，之后每次打开应用，执行的都是本地机器码。

### dex变迁

dalvik
android 2.2 +jit
android 4.4 dalvik + art
android 5.0 art
android 7.0 art + jit

### dex (Dalvik Executable)

### odex

### oat

### dexopt & dex2oat

### vdex cdex oat art
**找个机器root查看**


### 混合编译

### speed speed-profile
extract
verify
quicken
space-profile
space
speed-profile
speed
everything-profile
everything

### bg-dexopt-service




06-10 10:35:06.691  6369  6369 I dex2oat : /system/bin/dex2oat --input-vdex-fd=-1 --output-vdex-fd=19 --compiler-filter=speed-profile --classpath-dir=/data/app/com.xunmeng.pinduoduo-   _cGBEpbrRfhwNSkdGy6lhA== --class-loader-context=PCL[/system/framework/org.apache.http.legacy.boot.jar] --generate-mini-debug-info --compact-dex-level=none --compilation-reason=install  06-10 10:35:06.698  6369  6369 W dex2oat : Could not reserve sentinel fault page
 06-10 10:35:13.607  6369  6369 I dex2oat : Explicit concurrent copying GC freed 55062(9MB) AllocSpace objects, 0(0B) LOS objects, 99% free, 1232B/1537KB, paused 96us total 13.838ms
 06-10 10:35:15.221  6369  6369 I dex2oat : dex2oat took 8.535s (17.701s cpu) (threads: 4) arena alloc=27KB (27960B) java alloc=17KB (17616B) native alloc=15MB (16343304B) free=3MB      (3579640B)
 
 06-10 10:44:25.535  7930  7930 I dex2oat : /system/bin/dex2oat --input-vdex-fd=19 --output-vdex-fd=20 --compiler-filter=speed-profile --profile-file-fd=23 --classpath-dir=/data/app/    com.xunmeng.pinduoduo-_cGBEpbrRfhwNSkdGy6lhA== --class-loader-context=PCL[/system/framework/org.apache.http.legacy.boot.jar] --generate-mini-debug-info --compilation-reason=bg-dexopt


 A3A_10_4G:/data/app/com.xunmeng.pinduoduo-CfWqhUJYrwCFYRW2sTFEPw==/oat/arm # ls -l
total 15128
-rw-r--r-- 1 system all_a121    57344 2019-06-11 07:51 base.art
-rw-r--r-- 1 system all_a121   273504 2019-06-11 07:51 base.odex
-rw-r--r-- 1 system all_a121 15158668 2019-06-11 07:51 base.vdex

