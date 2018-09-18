---
layout: article
title: 配置ART
key: 20180918
tags:
  - art
  - speed-profile
lang: zh-Hans
---

# 配置ART

### [原文链接](https://blog.csdn.net/zhangbijun1230/article/details/80589875)

### 本页面介绍了如何配置 ART 及其编译选项。讨论的主题包括：系统映像预编译配置、dex2oat 编译选项，以及如何在系统分区空间、数据分区空间和性能这三者之间取得平衡。

请参阅 ART 和 Dalvik、Dalvik 可执行文件格式，以及 source.android.com 上的其他页面，了解如何使用 ART。请参阅在 Android Runtime (ART) 上验证应用行为，了解如何确保您的应用能够正常运行。

# ART 的运作方式

ART 使用预先 (AOT) 编译，并且从 Android 7.0（代号 Nougat，简称 N）开始结合使用 AOT、即时 (JIT) 编译和配置文件引导型编译。所有这些编译模式的组合均可配置，我们将在本部分中对此进行介绍。例如，Pixel 设备配置了以下编译流程：

- 最初安装应用时不进行任何 AOT 编译。应用前几次运行时，系统会对其进行解译，并对经常执行的方法进行 JIT 编译。
- 当设备闲置和充电时，编译守护进程会运行，以便根据在应用前几次运行期间生成的配置文件对常用代码进行 AOT 编译。
- 下一次重新启动应用时将会使用配置文件引导型代码，并避免在运行时对已经过编译的方法进行 JIT 编译。在应用后续运行期间进行了 JIT 编译的方法将会被添加到配置文件中，然后编译守护进程将会对这些方法进行 AOT 编译。

ART 包括一个编译器（dex2oat 工具）和一个为启动 Zygote 而加载的运行时 (libart.so)。dex2oat 工具接受一个 APK 文件，并生成一个或多个编译工件文件，然后运行时将会加载这些文件。文件的个数、扩展名和名称会因版本而异，但在 Android O 版本中，将会生成以下文件：

- .vdex：其中包含 APK 的未压缩 DEX 代码，另外还有一些旨在加快验证速度的元数据。
- .odex：其中包含 APK 中已经过 AOT 编译的方法代码。
- .art (optional)：其中包含 APK 中列出的某些字符串和类的 ART 内部表示，用于加快应用启动速度。

# 编译选项

ART 的编译选项分为以下两个类别：

- 系统 ROM 配置：编译系统映像时，会对哪些代码进行 AOT 编译。
- 运行时配置：ART 如何在设备上编译和运行应用。

用于配置这两个类别的一个核心 ART 选项是“编译过滤器”。编译过滤器可控制 ART 如何编译 DEX 代码，是一个传递给 dex2oat 工具的选项。从 Android O 开始，有四个官方支持的过滤器：

- verify：只运行 DEX 代码验证。
- quicken：运行 DEX 代码验证，并优化一些 DEX 指令，以获得更好的解译器性能。
- speed：运行 DEX 代码验证，并对所有方法进行 AOT 编译。
- speed-profile：运行 DEX 代码验证，并对配置文件中列出的方法进行 AOT 编译。

## 系统 ROM 配置

有一些 ART 编译选项可用于配置系统 ROM。如何配置这些选项取决于 /system 的可用存储空间以及预先安装的应用数量。编译到系统 ROM 中的 JAR/APK 可以分为以下四个类别：

- 启动类路径代码：默认使用 speed 编译过滤器进行编译。
- 系统服务器代码：默认使用 speed 编译过滤器进行编译。
- 产品专属的核心应用：默认使用 speed 编译过滤器进行编译。
- 所有其他应用：默认使用 quicken 编译过滤器进行编译。

## Makefile 选项

#### WITH_DEXPREOPT

是否对系统映像上安装的 DEX 代码调用 dex2oat。默认处于启用状态。

#### DONT_DEXPREOPT_PREBUILTS（从 Android 5.0 开始）

启用 DONT_DEXPREOPT_PREBUILTS 可防止对预编译的应用进行预先优化。这些都是在 Android.mk 中指定了 include $(BUILD_PREBUILT) 的应用，例如 Gmail。不对这些很可能要通过 Google Play 进行更新的预编译应用进行预先优化可以节省 /system 的空间，但是会增加首次启动时间。

#### WITH_DEXPREOPT_BOOT_IMG_AND_SYSTEM_SERVER_ONLY（Android O MR1 中的新增选项）

如果启用 WITH_DEXPREOPT_BOOT_IMG_AND_SYSTEM_SERVER_ONLY，将只会对启动类路径和系统服务器 JAR 进行预先优化。

#### LOCAL_DEX_PREOPT

通过在模块定义中指定 LOCAL_DEX_PREOPT 选项，还可以针对个别应用启用或停用预先优化功能。这有助于避免对那些可能会立即收到 Google Play 更新的应用进行预先优化，因为更新之后，对系统映像中的代码所做的预先优化都成了无用功。此外，这还有助于在进行重大版本 OTA 升级时节省空间，因为用户的数据分区中可能已经有了更高版本的应用。

#### LOCAL_DEX_PREOPT 支持分别使用值“true”和“false”来启用和停用预先优化功能。

此外，如果在预先优化过程中不应将 classes.dex 文件从 APK 或 JAR 文件中剥离出来，则可以指定“nostripping”。通常情况下，该文件会被剥离出来，因为在进行预先优化之后将不再需要该文件；但若要使第三方 APK 签名仍保持有效，则必须使用最后这个选项。

#### PRODUCT_DEX_PREOPT_BOOT_FLAGS

x2oat 以控制如何编译启动映像。该选项可用于指定自定义映像类列表、已编译类列表，以及编译过滤器。

#### PRODUCT_DEX_PREOPT_DEFAULT_FLAGS

将选项传递给 dex2oat 以控制如何编译除启动映像之外的所有内容。

#### PRODUCT_DEX_PREOPT_MODULE_CONFIGS

用于为特定模块和产品配置传递 dex2oat 选项。可以在产品的 device.mk 文件中通过 $(call add-product-dex-preopt-module-config,<modules>,<option>) 设置该选项，其中 <modules> 是一个列表，用于列出 JAR 和 APK 文件各自的 LOCAL_MODULE 和 LOCAL_PACKAGE 名称。

#### PRODUCT_DEXPREOPT_SPEED_APPS (New in Android O)

一个应用列表，用于列出哪些应用已被确定为产品的核心应用并且应使用 speed 编译过滤器对其进行编译。例如，常驻应用（如 SystemUI）只有在下次系统重新启动时才有机会使用配置文件引导型编译，因此对于产品来说，让这些应用始终采用 AOT 编译可能会更好。

#### PRODUCT_SYSTEM_SERVER_APPS (New in Android O)

系统服务器加载的应用的列表。这些应用将默认使用 speed 编译过滤器进行编译。

#### PRODUCT_ART_TARGET_INCLUDE_DEBUG_BUILD(Post Android O)

是否在设备上包含 ART 的调试版本。默认情况下，系统会针对 userdebug 和 eng 细分版本启用该选项。可以通过将该选项明确设为“true”或“false”来覆盖此行为。

默认情况下，设备将使用非调试版本 (libart.so)。要进行切换，请将系统属性 persist.sys.dalvik.vm.lib.2 设为 libartd.so。

#### WITH_DEXPREOPT_PIC (Removed in Android O)

从 Android 5.1.0 到 Android 6.0.1 的所有版本中，都可以指定 WITH_DEXPREOPT_PIC 来启用位置无关代码 (PIC)。这样一来，就不必将来自映像的编译代码从 /system 迁移到 /data/dalvik-cache，因此可以节省数据分区中的空间。不过，因为该选项会停用根据位置相关代码进行的优化，所以会对运行时产生轻微的影响。通常情况下，需要节省 /data 空间的设备应启用 PIC 编译。

在 Android 7.0 中，PIC 编译默认处于启用状态。

WITH_DEXPREOPT_BOOT_IMG_ONLY（已在 Android O MR1 中移除）
此选项已被 WITH_DEXPREOPT_BOOT_IMG_AND_SYSTEM_SERVER_ONLY 取代，后者还会预先优化系统服务器 JAR。

## 启动类路径配置

### 预加载类列表

预加载类列表列出了 zygote 将在启动时初始化的类。利用该列表，每个应用无需单独运行这些类初始化程序，从而可以更快地启动并共享内存中的页面。

预加载类列表文件默认位于 frameworks/base/preloaded-classes 中，其中包含一个针对典型手机使用场景优化的列表。

其他设备（如穿戴式设备）可能不适合使用此列表，而应进行相应调整。

做调整时应格外小心，因为添加太多的类会造成加载未用到的类而浪费内存；而添加的类太少又会导致每个应用都各有一份副本，同样会造成内存浪费。

#### 使用示例（在产品的 device.mk 中）：

PRODUCT_COPY_FILES += <filename>:system/etc/preloaded-classes

注意：如果有任何从 build/target/product/base.mk 提取默认值的产品配置 Makefile，此行必须放在沿用此类 Makefile 的行之前。

### 映像类列表

映像类列表列出了 dex2oat 预先初始化并存储在 boot.art 文件中的类。利用该列表，zygote 可以在启动时从 boot.art 文件中加载这些结果，而无需在预加载期间自行运行这些类的初始化程序。这种做法的一个重要特点是，从映像加载并在进程之间共享的页面是干净的，因此可在内存不足的情况下轻松将它们交换出去。在 L 版本中，默认情况下，映像类列表和预加载类列表是同一个列表。从 L 之后的 AOSP 版本开始，可使用以下选项指定自定义映像类列表：

  PRODUCT_DEX_PREOPT_BOOT_FLAGS

使用示例（在产品的 device.mk 中）：

  PRODUCT_DEX_PREOPT_BOOT_FLAGS += --image-classes=<filename> 

### 已编译类列表

在 L 之后的 AOSP 版本中，可使用此列表来指定一个启动类路径的类子集，以便在预先优化期间编译这些类。如果设备存储空间非常紧张，无法完整容纳经过预先优化的启动映像，此选项就很有帮助。不过请注意，此列表未指定的类将不会被编译（即使在设备上也不会被编译），而必须对其进行解译，这可能会影响运行时性能。默认情况下，dex2oat 会在 $OUT/system/etc/compiled-classes 中查找已编译类列表，因此，可以通过 device.mk 将自定义类列表复制到该位置。也可使用以下选项指定文件位置：

  PRODUCT_DEX_PREOPT_BOOT_FLAGS

使用示例（在产品的 device.mk 中）：

  PRODUCT_COPY_FILES += <filename>:system/etc/compiled-classes

注意：如果有任何从 build/target/product/base.mk 提取默认值的产品配置 Makefile，此行必须放在沿用此类 Makefile 的行之前。

### 运行时配置

#### Jit 选项

仅在 ART JIT 编译器可用的情况下，以下选项才会影响 Android 版本。

- dalvik.vm.usejit：是否启用 JIT。
- dalvik.vm.jitinitialsize（默认为 64K）：代码缓存初始容量。代码缓存将定期进行垃圾回收 (GC)，并将视需要增加容量。
- dalvik.vm.jitmaxsize（默认为 64M）：代码缓存最大容量。
- dalvik.vm.jitthreshold（默认为 10000）：方法的“热度”计数器必须超过该阈值，系统才会对方法进行 JIT 编译。“热度”计数器是运行时的内部指标。它的影响因素包括调用次数、后向分支及其他因素。
- dalvik.vm.usejitprofiles：是否启用 JIT 配置文件；即使 dalvik.vm.usejit 为 false，也可以使用该选项。请注意，如果该选项为 false，编译过滤器 speed-profile 将不会对任何方法进行 AOT 编译，效果与 quicken 相同。
- dalvik.vm.jitprithreadweight（默认为 dalvik.vm.jitthreshold/20）：应用界面线程的 JIT“样本”（请参阅 jitthreshold）的权重。用于加快以下方法的编译速度：当用户与应用交互时，会直接影响用户体验的方法。
- dalvik.vm.jittransitionweight（默认为 dalvik.vm.jitthreshold/10）：调用时需要在编译代码和解译器之间进行转换的方法的权重。这有助于确保对所涉及的方法进行编译以尽可能减少转换（转换需要很大开销）。

### 软件包管理器选项

从 Android 7.0 开始，系统提供了一种通用方式来指定各个阶段的编译/验证级别。编译级别通过系统属性来配置，默认值如下：

   pm.dexopt.install=quicken

这是通过 Google Play 安装应用时使用的编译过滤器。要加快安装速度，可以尝试使用 quicken 编译过滤器。

  pm.dexopt.bg-dexopt=speed-profile

这是在设备闲置、充电以及充满电时使用的编译过滤器。要充分利用配置文件引导型编译并节省存储空间，可以尝试使用 speed-profile 编译过滤器。

  pm.dexopt.boot=verify

OTA 更新后使用的编译过滤器。对于该选项，我们强烈建议使用 verify 编译过滤器，以免启动时间过长。

  pm.dexopt.first-boot=quicken

在设备初次启动时使用的编译过滤器。此过滤器只会影响出厂后第一次启动的时间。我们建议使用 quicken 过滤器，以免用户在首次使用手机时需要花很长时间等待手机启动。请注意，如果 /system 中的所有应用都已使用 quicken 编译过滤器进行编译，或者已使用 speed 或 speed-profile 编译过滤器进行编译，pm.dexopt.first-boot 将不会产生任何影响。

### Dex2oat 选项

请注意，这些选项在设备编译期间以及预先优化期间都会影响 dex2oat，但是前面讨论的大多数选项都只会影响预先优化。

在 dex2oat 编译启动映像时对其进行控制：

- dalvik.vm.image-dex2oat-Xms：初始堆大小
- dalvik.vm.image-dex2oat-Xmx：最大堆大小
- dalvik.vm.image-dex2oat-filter：编译过滤器选项
- dalvik.vm.image-dex2oat-threads：要使用的线程数

在 dex2oat 编译除启动映像之外的所有内容时对其进行控制：

- dalvik.vm.dex2oat-Xms：初始堆大小
- dalvik.vm.dex2oat-Xmx：最大堆大小
- dalvik.vm.dex2oat-filter：编译过滤器选项

Android 6.0 之前的版本提供了一个适用于编译除启动映像之外的所有内容的附加选项：

  dalvik.vm.dex2oat-threads：要使用的线程数

自 Android 6.1 起，该选项变成了两个适用于编译除启动映像之外的所有内容的附加选项：

- dalvik.vm.boot-dex2oat-threads：启动时要使用的线程数
- dalvik.vm.dex2oat-threads：启动后要使用的线程数

Android 7.1 及之后的版本提供了两个选项来控制编译除启动映像之外的所有内容时的内存使用方式：

- dalvik.vm.dex2oat-very-large：停用 AOT 编译的最小总 dex 文件大小（以字节为单位）
- dalvik.vm.dex2oat-swap：使用 dex2oat 交换文件（用于内存较小的设备）

不应减小用于控制 dex2oat 初始堆大小和最大堆大小的选项数值，因为它们可能会限制可对哪些应用进行编译。

### A/B 专有配置

#### ROM 配置

从 Android 7.0 开始，设备可以使用两个系统分区来实现 A/B 系统更新。为了减小系统分区大小，可以将经过预先优化的文件安装在未使用的第二个系统分区中。在系统首次启动时，这些文件会被复制到数据分区。

使用示例（在 device-common.mk 中）：

  PRODUCT_PACKAGES += \ cppreopts.sh PRODUCT_PROPERTY_OVERRIDES += \ ro.cp_system_other_odex=1

在设备的 BoardConfig.mk 中：

  BOARD_USES_SYSTEM_OTHER_ODEX := true

请注意，启动类路径代码、系统服务器代码以及产品专属的核心应用始终会被编译到系统分区中。默认情况下，所有其他应用都会被编译到未使用的第二个系统分区中。可以使用 SYSTEM_OTHER_ODEX_FILTER 控制此行为，其值默认为：

  SYSTEM_OTHER_ODEX_FILTER ?= app/% priv-app/%

#### 后台 dexopt OTA

在启用了 A/B 的设备上，可以在后台对应用进行编译，以更新到新的系统映像。如需在系统映像中选择性地加入编译脚本和二进制文件，请参阅在后台编译应用。可通过以下选项控制用于此类编译的编译过滤器：

  pm.dexopt.ab-ota=speed-profile

我们建议使用 speed-profile，以利用配置文件引导型编译并节省存储空间。
