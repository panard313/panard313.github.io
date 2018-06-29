---
layout: article
title: 优化启动时间
key: 20180529
tags:
  - android offical doc
  - 开机
  - 启动时间
lang: zh-Hans
---

# 优化启动时间

本文档提供了有关改进特定 Android 设备的启动时间的合作伙伴指南。启动时间是系统性能的重要组成部分，因为用户必须等待启动完成后才能使用设备。对于较常进行冷启动的汽车等设备而言，较短的启动时间至关重要（没有人喜欢在等待几十秒后才能输入导航目的地）。

Android 8.0 支持一系列组件的多项改进，因而可以缩短启动时间。下表对这些性能改进（在 Google Pixel 和 Pixel XL 设备上测得）进行了总结。

组件 and	改进

引导加载程序 	
- 通过移除 UART 日志节省了 1.6 秒
- 通过从 GZIP 更改为 LZ4 节省了 0.4 秒

设备内核 	
- 通过移除不使用的内核配置和减少驱动程序大小节省了 0.3 秒
- 通过 dm-verity 预提取优化节省了 0.3 秒
- 通过移除驱动程序中不必要的等待/测试，节省了 0.15 秒
- 通过移除 CONFIG_CC_OPTIMIZE_FOR_SIZE，节省了 0.12 秒

I/O 调整 	
- 正常启动时间节省了 2 秒
- 首次启动时间节省了 25 秒

init.*.rc 	
- 通过并行运行 init 命令节省了 1.5 秒
- 通过及早启动 zygote 节省了 0.25 秒
- 通过 cpuset 调整节省了 0.22 秒

启动动画 	
- 在未触发 fsck 的情况下，启动动画的开始时间提前了 2 秒，而触发 fsck 时启动动画则大得多
- 通过立即关闭启动动画在 Pixel XL 上节省了 5 秒

SELinux 政策 
- 通过 genfscon 节省了 0.2 秒

## 优化引导加载程序

要优化引导加载程序以缩短启动时间，请遵循以下做法：

- 对于日志记录：

停止向 UART 写入日志，因为如果日志记录很多，则可能需要很长时间来处理。（在 Google Pixel 设备上，我们发现这会使引导加载程序的速度减慢 1.5 秒）。

- 仅记录错误情况，并考虑将其他信息存储到具有单独检索机制的内存中。

- 对于内核解压缩，请考虑为当代硬件使用 LZ4 而非 GZIP（例如补丁程序）。请注意，不同的内核压缩选项具有不同的加载和解压缩时间，对于特定硬件，某些选项可能比其他选项更适合。
- 检查进入去抖动/特殊模式过程中是否有不必要的等待时间，并最大限度地减少此类时间。
- 将在引导加载程序中花费的启动时间以命令行的形式传递到内核。
- 检查 CPU 时钟并考虑内核加载和初始化 I/O 并行进行（需要多核支持）。

## 优化内核

请按照以下提示优化内核以缩短启动时间。

最大限度地减少设备 defconfig

最大限度地减少内核配置可以减小内核大小，从而更快速地进行加载、解压缩、初始化并缩小受攻击面。要优化设备 defconfig，请执行以下操作：

    识别未使用的驱动程序。查看 /dev 和 /sys 目录，并查找带有常规 SELinux 标签的节点（这种标签表示相应节点未配置为可由用户空间访问）。如果找到此类节点，请将其移除。
    取消设置未使用的配置。查看由内核版本生成的 .config 文件，以明确取消设置所有已默认启用但并未使用的配置。例如，我们从 Google Pixel 中移除了以下未使用的配置：

    CONFIG_ANDROID_LOGGER=y
    CONFIG_IMX134=y
    CONFIG_IMX132=y
    CONFIG_OV9724=y
    CONFIG_OV5648=y
    CONFIG_GC0339=y
    CONFIG_OV8825=y
    CONFIG_OV8865=y
    CONFIG_s5k4e1=y
    CONFIG_OV12830=y
    CONFIG_USB_EHCI_HCD=y
    CONFIG_IOMMU_IO_PGTABLE_FAST_SELFTEST=y
    CONFIG_IKCONFIG=y
    CONFIG_RD_BZIP2=y
    CONFIG_RD_LZMA=y
    CONFIG_TI_DRV2667=y
    CONFIG_CHR_DEV_SCH=y
    CONFIG_MMC=y
    CONFIG_MMC_PERF_PROFILING=y
    CONFIG_MMC_CLKGATE=y
    CONFIG_MMC_PARANOID_SD_INIT=y
    CONFIG_MMC_BLOCK_MINORS=32
    CONFIG_MMC_TEST=y
    CONFIG_MMC_SDHCI=y
    CONFIG_MMC_SDHCI_PLTFM=y
    CONFIG_MMC_SDHCI_MSM=y
    CONFIG_MMC_SDHCI_MSM_ICE=y
    CONFIG_MMC_CQ_HCI=y
    CONFIG_MSDOS_FS=y
    # CONFIG_SYSFS_SYSCALL is not set
    CONFIG_EEPROM_AT24=y
    # CONFIG_INPUT_MOUSEDEV_PSAUX is not set
    CONFIG_INPUT_HBTP_INPUT=y
    # CONFIG_VGA_ARB is not set
    CONFIG_USB_MON=y
    CONFIG_USB_STORAGE_DATAFAB=y
    CONFIG_USB_STORAGE_FREECOM=y
    CONFIG_USB_STORAGE_ISD200=y
    CONFIG_USB_STORAGE_USBAT=y
    CONFIG_USB_STORAGE_SDDR09=y
    CONFIG_USB_STORAGE_SDDR55=y
    CONFIG_USB_STORAGE_JUMPSHOT=y
    CONFIG_USB_STORAGE_ALAUDA=y
    CONFIG_USB_STORAGE_KARMA=y
    CONFIG_USB_STORAGE_CYPRESS_ATACB=y
    CONFIG_SW_SYNC_USER=y
    CONFIG_SEEMP_CORE=y
    CONFIG_MSM_SMEM_LOGGING=y
    CONFIG_IOMMU_DEBUG=y
    CONFIG_IOMMU_DEBUG_TRACKING=y
    CONFIG_IOMMU_TESTS=y
    CONFIG_MOBICORE_DRIVER=y
    # CONFIG_DEBUG_PREEMPT is not set

    移除导致每次启动时运行不必要测试的配置。虽然此类配置（即 CONFIG_IOMMU_IO_PGTABLE_FAST_SELFTEST）在开发过程中很有用，但应从正式版内核中移除。

最大限度地减小驱动程序大小

如果未使用相应功能，则可以移除设备内核中的某些驱动程序，以便进一步减小内核大小。例如，如果 WLAN 通过 PCIe 连接，则不会用到 SDIO 支持，因此应在编译时将其移除。有关详情，请参阅 Google Pixel 内核：网络：无线：CNSS：添加选项以停用 SDIO 支持。
移除针对大小的编译器优化

移除 CONFIG_CC_OPTIMIZE_FOR_SIZE 的内核配置。此标记是在最初假设较小的代码大小会产生热缓存命中（因此速度更快）时引入的。然而，随着现代移动 SoC 变得更加强大，这一假设不再成立。

此外，移除此标记可以使编译器针对未初始化的变量发出警告，当存在 CONFIG_CC_OPTIMIZE_FOR_SIZE 标记时，这一功能在 Linux 内核中是停用的（仅这一项更改就已帮助我们在某些 Android 设备驱动程序中发现了很多有意义的错误）。
延迟初始化

很多进程都在设备启动期间启动，但只有关键路径 (bootloader > kernel > init > file system mount > zygote > system server) 中的组件才会直接影响启动时间。在内核启动期间执行 initcall 来识别启动速度缓慢且对启动 init 进程不重要的外设/组件，然后通过将这些外设/组件移入可加载的内核模块将其延迟到启动过程的后期来启动。移入异步设备/驱动程序探测还有助于并行启动内核 > init 重要路径中启动速度缓慢的组件。

BoardConfig-common.mk:
    BOARD_KERNEL_CMDLINE += initcall_debug ignore_loglevel

driver:
    .probe_type = PROBE_PREFER_ASYNCHRONOUS,

注意：必须添加 EPROBEDEFER 支持来妥善解决驱动程序依赖问题。
优化 I/O 效率

提高 I/O 效率对缩短启动时间来说至关重要，对任何不必要内容的读取都应推迟到启动之后再进行（在 Google Pixel 上，启动时大约要读取 1.2GB 的数据）。
调整文件系统

当从头开始读取某个文件或依序读取块时，预读的 Linux 内核便会启动，这就需要调整专门用于启动的 I/O 调度程序参数（与普通应用的工作负载特性不同）。

支持无缝 (A/B) 更新的设备在首次启动时会极大地受益于文件系统调整（例如，Google Pixel 的启动时间缩短了 20 秒）。例如，我们为 Google Pixel 调整了以下参数：

on late-fs
  # boot time fs tune
    # boot time fs tune
    write /sys/block/sda/queue/iostats 0
    write /sys/block/sda/queue/scheduler cfq
    write /sys/block/sda/queue/iosched/slice_idle 0
    write /sys/block/sda/queue/read_ahead_kb 2048
    write /sys/block/sda/queue/nr_requests 256
    write /sys/block/dm-0/queue/read_ahead_kb 2048
    write /sys/block/dm-1/queue/read_ahead_kb 2048

on property:sys.boot_completed=1
    # end boot time fs tune
    write /sys/block/sda/queue/read_ahead_kb 512
    ...

其他

    使用内核配置 DM_VERITY_HASH_PREFETCH_MIN_SIZE（默认大小为 128）来启用 dm-verity 哈希预提取大小。
    为了提升文件系统稳定性及取消每次启动时的强制检查，请在 BoardConfig.mk 中设置 TARGET_USES_MKE2FS，以使用新的 ext4 生成工具。

分析 I/O

要了解启动过程中的 I/O 活动，请使用内核 ftrace 数据（systrace 也使用该数据）：

trace_event=block,ext4 in BOARD_KERNEL_CMDLINE

要针对每个文件细分文件访问权限，请对内核进行以下更改（仅限开发版内核；请勿在正式版内核中应用这些更改）：

diff --git a/fs/open.c b/fs/open.c
index 1651f35..a808093 100644
--- a/fs/open.c
+++ b/fs/open.c
@@ -981,6 +981,25 @@
 }
 EXPORT_SYMBOL(file_open_root);

+static void _trace_do_sys_open(struct file *filp, int flags, int mode, long fd)
+{
+       char *buf;
+       char *fname;
+
+       buf = kzalloc(PAGE_SIZE, GFP_KERNEL);
+       if (!buf)
+               return;
+       fname = d_path(&filp-<f_path, buf, PAGE_SIZE);
+
+       if (IS_ERR(fname))
+               goto out;
+
+       trace_printk("%s: open(\"%s\", %d, %d) fd = %ld, inode = %ld\n",
+                     current-<comm, fname, flags, mode, fd, filp-<f_inode-<i_ino);
+out:
+       kfree(buf);
+}
+
long do_sys_open(int dfd, const char __user *filename, int flags, umode_t mode)
 {
        struct open_flags op;
@@ -1003,6 +1022,7 @@
                } else {
                        fsnotify_open(f);
                        fd_install(fd, f);
+                       _trace_do_sys_open(f, flags, mode, fd);

使用以下脚本来帮助分析启动性能。

    packages/services/Car/tools/bootanalyze/bootanalyze.py：负责衡量启动时间，并详细分析启动过程中的重要步骤。
    packages/services/Car/tools/io_analysis/check_file_read.py boot_trace：提供每个文件的访问信息。
    packages/services/Car/tools/io_analysis/check_io_trace_all.py boot_trace：提供系统级细分信息。

优化 init.*.rc

Init 是从内核到框架建立之前的衔接过程，设备通常会在不同的 init 阶段花费几秒钟时间。
并行运行任务

虽然当前的 Android init 差不多算是一种单线程进程，但您仍然可以并行执行一些任务。

    在 Shell 脚本服务中执行缓慢命令，然后通过等待特定属性，在稍后加入。Android 8.0 通过新的 wait_for_property 命令支持此用例。
    识别 init 中的缓慢操作。系统会记录 init 命令 exec/wait_for_prop 或任何所需时间较长的操作（在 Android 8.0 中，指所需时间超过 50 毫秒的任何命令）。例如：

    init: Command 'wait_for_coldboot_done' action=wait_for_coldboot_done returned 0 took 585.012ms

    查看此日志可能会发现可以改进的机会。
    启动服务并及早启用关键路径中的外围设备。例如，有些 SOC 需要先启动安全相关服务，然后再启动 SurfaceFlinger。在 ServiceManager 返回“wait for service”（等待服务）时查看系统日志 - 这通常表明必须先启动依赖服务。
    移除 init.*.rc 中所有未使用的服务和命令。只要是早期阶段的 init 中没有使用的服务和命令，都应推迟到启动完成后再使用。

注意：“属性”服务是 init 进程的一部分，因此，在启动期间调用 setproperty 可能会导致较长时间的延迟（如果 init 忙于执行内置命令）。
使用调度程序调整

使用调度程序调整，以便及早启动设备。以下是取自 Google Pixel 的示例：

on init
    # update cpusets now that processors are up
    write /dev/cpuset/top-app/cpus 0-3
    write /dev/cpuset/foreground/cpus 0-3
    write /dev/cpuset/foreground/boost/cpus 0-3
    write /dev/cpuset/background/cpus 0-3
    write /dev/cpuset/system-background/cpus 0-3
    # set default schedTune value for foreground/top-app (only affects EAS)
    write /dev/stune/foreground/schedtune.prefer_idle 1
    write /dev/stune/top-app/schedtune.boost 10
    write /dev/stune/top-app/schedtune.prefer_idle 1

部分服务在启动过程中可能需要进行优先级提升。例如：

init.zygote64.rc:
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    priority -20
    user root
...

及早启动 zygote

采用文件级加密的设备可以在 zygote-start 触发器的早期阶段启动 zygote（默认情况下，zygote 会在 main 类中启动，比 zygote-start 晚得多）。这样做时，请确保允许 zygote 在所有 CPU 中运行（因为错误的 cpuset 设置可能会强制 zygote 在特定 CPU 中运行）。
停用节电设置

在设备启动期间，可以停用 UFS 和/或 CPU 调节器等组件的节电设置。

请注意：为了提高效率，应在充电器模式下启用节电设置。

on init
    # Disable UFS powersaving
    write /sys/devices/soc/${ro.boot.bootdevice}/clkscale_enable 0
    write /sys/devices/soc/${ro.boot.bootdevice}/clkgate_enable 0
    write /sys/devices/soc/${ro.boot.bootdevice}/hibern8_on_idle_enable 0
    write /sys/module/lpm_levels/parameters/sleep_disabled Y
on property:sys.boot_completed=1
    # Enable UFS powersaving
    write /sys/devices/soc/${ro.boot.bootdevice}/clkscale_enable 1
    write /sys/devices/soc/${ro.boot.bootdevice}/clkgate_enable 1
    write /sys/devices/soc/${ro.boot.bootdevice}/hibern8_on_idle_enable 1
    write /sys/module/lpm_levels/parameters/sleep_disabled N
on charger
    # Enable UFS powersaving
    write /sys/devices/soc/${ro.boot.bootdevice}/clkscale_enable 1
    write /sys/devices/soc/${ro.boot.bootdevice}/clkgate_enable 1
    write /sys/devices/soc/${ro.boot.bootdevice}/hibern8_on_idle_enable 1
    write /sys/class/typec/port0/port_type sink
    write /sys/module/lpm_levels/parameters/sleep_disabled N

推迟非关键初始化

非关键初始化（如 ZRAM）可以推迟到 boot_complete。

on property:sys.boot_completed=1
   # Enable ZRAM on boot_complete
   swapon_all /vendor/etc/fstab.${ro.hardware}

优化启动动画

请按照以下提示来优化启动动画。
配置为及早启动

Android 8.0 支持在装载用户数据分区之前，及早启动动画。然而，即使 Android 8.0 中使用了新的 ext4 工具链，系统也会出于安全原因定期触发 fsck，导致启动 bootanimation 服务时出现延迟。

为了使 bootanimation 及早启动，请将 fstab 装载分为以下两个阶段：

    在早期阶段，仅装载不需要运行检查的分区（例如 system/ 和 vendor/），然后启动启动动画服务及其依赖项（例如 servicemanager 和 surfaceflinger）。
    在第二个阶段，装载需要运行检查的分区（例如 data/）。

启动动画将会更快速地启动（且启动时间恒定），不受 fsck 影响。
干净利落地结束

在收到退出信号后，bootanimation 会播放最后一部分，而这一部分的长度会延长启动时间。快速启动的系统不需要很长的动画，如果启动动画很长，在很大程度上就体现不出所做的任何改进。我们建议缩短循环播放和结尾的时间。
优化 SELinux

请按照以下提示优化 SELinux 以缩短启动时间。

    使用简洁的正则表达式 (regex)。在为 file_contexts 中的 sys/devices 匹配 SELinux 政策时，格式糟糕的正则表达式可能会导致大量开销。例如，正则表达式 /sys/devices/.*abc.*(/.*)? 错误地强制扫描包含“abc”的所有 /sys/devices 子目录，导致 /sys/devices/abc 和 /sys/devices/xyz/abc 都成为匹配项。如果将此正则表达式修正为 /sys/devices/[^/]*abc[^/]*(/.*)? ，则只有 /sys/devices/abc 会成为匹配项。
    将标签移动到 genfscon。这一现有的 SELinux 功能会将文件匹配前缀传递到 SELinux 二进制文件的内核中，而内核会将这些前缀应用于内核生成的文件系统。这也有助于修复错误标记的内核创建的文件，从而防止用户空间进程之间可能出现的争用情况（试图在重新标记之前访问这些文件）。

工具和方法

请使用以下工具来帮助您收集用于优化目标的数据。
bootchart

bootchart 可为整个系统提供所有进程的 CPU 和 I/O 负载细分。该工具不需要重建系统映像，可以用作进入 systrace 之前的快速健全性检查。

要启用 bootchart，请运行以下命令：

adb shell 'touch /data/bootchart/enabled'
adb reboot

在设备启动后，获取启动图表：

$ANDROID_BUILD_TOP/system/core/init/grab-bootchart.sh

完成后，请删除 /data/bootchart/enabled 以防止每次都收集日期数据。
systrace

systrace 允许在启动期间收集内核和 Android 跟踪记录。 systrace 的可视化可以帮助分析启动过程中的具体问题。（不过，要查看整个启动过程中的平均数量或累计数量，直接查看内核跟踪记录更为方便）。

要在启动过程中启用 systrace，请执行以下操作：

    在 frameworks/native/atrace/atrace.rc 中，将

    write /sys/kernel/debug/tracing/tracing_on 0

    更改为：

    #write /sys/kernel/debug/tracing/tracing_on 0

    这将启用跟踪功能（默认处于停用状态）。
    在 device.mk 文件中，添加下面这行内容：

    PRODUCT_PROPERTY_OVERRIDES +=    debug.atrace.tags.enableflags=802922

    在设备 BoardConfig.mk 文件中，添加以下内容：

    BOARD_KERNEL_CMDLINE := ... trace_buf_size=64M trace_event=sched_wakeup,sched_switch,sched_blocked_reason,sched_cpu_hotplug

    要获得详细的 I/O 分析，还需要添加块和 ext4。
    在设备专用的 init.rc 文件中，进行以下更改：
        on property:sys.boot_completed=1（这会在启动完成后停止跟踪）
        write /d/tracing/tracing_on 0
        write /d/tracing/events/ext4/enable 0
        write /d/tracing/events/block/enable 0

在设备启动后，获取跟踪记录：

adb root && adb shell "cat /d/tracing/trace" < boot_trace
./external/chromium-trace/catapult/tracing/bin/trace2html boot_trace --output boot_trace.html

注意：Chrome 无法处理过大的文件。请考虑使用 tail、head 或 grep 分割 boot_trace 文件，以获得必要的部分。由于事件过多，I/O 分析通常需要直接分析获取的 boot_trace。

Except as otherwise noted, the content of this page is licensed under the Creative Commons Attribution 3.0 License, and code samples are licensed under the Apache 2.0 License. For details, see our Site Policies. Java is a registered trademark of Oracle and/or its affiliates.

上次更新日期：五月 21, 2018
