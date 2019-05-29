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
odex

在Android N 之前，Dalvik虚拟机执行程序dex文件前，系统会对dex文件做优化，生成可执行文件odex，保存到data/dalvik-cache目录，最后把apk文件中的dex文件删除。

优点：

- 1. 减少了启动时间（省去了系统第一次启动应用时从apk文件中读取dex文件，并对dex文件做优化的过程）和对RAM的占用（apk文件中的dex如果不删除，同一个应用就会存在两个dex文件：apk中和data/dalvik-cache目录下）。

- 2.防止第三方用户反编译系统的软件（odex文件是跟随系统环境变化的，改变环境会无法运行；而apk文件中又不包含dex文件，无法独立运行）。

在Android O 之后，odex 是从vdex 这个文件中 提取了部分模块生成的一个新的 可执行二进制码 文件 ， odex 从vdex 中提取后，vdex 的大小就减少了。

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

