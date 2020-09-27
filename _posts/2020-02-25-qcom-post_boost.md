---
layout: article
title:  qcom post_boost cpuset
date:   2020-02-25
catalog:  true
tags:
    - qcom
    - post_boost
    - cpuset
---

# qcom post_boost cpuset

```shell
# disable thermal core_control to update interactive gov settings       #禁用CPU模式调度。0是不禁用，1是禁用。
     echo 0 > /sys/module/msm_thermal/core_control/enabled

# enable governor for perf cluster
#下面设置大核群的调度模式。（因为骁龙615的核心0-3是大核，而且1-3是跟随0的设置的，所以只要设置CPU0的就可以了）
#设置CPU0工作（如改为0，CPU0-3就是高性能模式）
     echo 1 > /sys/devices/system/cpu/cpu0/online

#设置CPU0调度模式为interactive
     echo "interactive" > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

#以下是interactive模式的句柄
    echo "20000 1113600:50000" > /sys/devices/system/cpu/cpu0/cpufreq/interactive/above_hispeed_delay
     echo 85 > /sys/devices/system/cpu/cpu0/cpufreq/interactive/go_hispeed_load
     echo 20000 > /sys/devices/system/cpu/cpu0/cpufreq/interactive/timer_rate
     echo 1113600 > /sys/devices/system/cpu/cpu0/cpufreq/interactive/hispeed_freq
     echo 0 > /sys/devices/system/cpu/cpu0/cpufreq/interactive/io_is_busy
     echo "1 960000:85 1113600:90 1344000:80" > /sys/devices/system/cpu/cpu0/cpufreq/interactive/target_loads
     echo 50000 > /sys/devices/system/cpu/cpu0/cpufreq/interactive/min_sample_time
     echo 50000 > /sys/devices/system/cpu/cpu0/cpufreq/interactive/sampling_down_factor

#设置最低频率为960MHZ。更改scaling_min_freq为scaling_max_freq则为设置最高频率
     echo 960000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq

# enable governor for power cluster
#设置小核群的调度模式（因为骁龙615的核心4-7是大核，而且5-7是跟随4的设置的，所以只要设置CPU4的就可以了）
#设置CPU4工作（如改为0，CPU4-7就是高性能模式）
     echo 1 > /sys/devices/system/cpu/cpu4/online

#设置CPU4调度模式为interactive
     echo "interactive" > /sys/devices/system/cpu/cpu4/cpufreq/scaling_governor

#以下是interactive模式的句柄
     echo "25000 800000:50000" > /sys/devices/system/cpu/cpu4/cpufreq/interactive/above_hispeed_delay
     echo 90 > /sys/devices/system/cpu/cpu4/cpufreq/interactive/go_hispeed_load
     echo 40000 > /sys/devices/system/cpu/cpu4/cpufreq/interactive/timer_rate
     echo 998400 > /sys/devices/system/cpu/cpu4/cpufreq/interactive/hispeed_freq
     echo 0 > /sys/devices/system/cpu/cpu4/cpufreq/interactive/io_is_busy
     echo "1 800000:90" > /sys/devices/system/cpu/cpu4/cpufreq/interactive/target_loads
     echo 40000 > /sys/devices/system/cpu/cpu4/cpufreq/interactive/min_sample_time
     echo 40000 > /sys/devices/system/cpu/cpu4/cpufreq/interactive/sampling_down_factor

#设置最低频率为800MHZ。更改scaling_min_freq为scaling_max_freq则为设置最高频率
    echo 800000 > /sys/devices/system/cpu/cpu4/cpufreq/scaling_min_freq

# enable thermal core_control now
#温度控制核心
        echo 1 > /sys/module/msm_thermal/core_control/enabled


# Bring up all cores online
#使所有核心工作。1是工作。0是离线。
        echo 1 > /sys/devices/system/cpu/cpu1/online
        echo 1 > /sys/devices/system/cpu/cpu2/online
        echo 1 > /sys/devices/system/cpu/cpu3/online
        echo 0 > /sys/devices/system/cpu/cpu4/online
        echo 0 > /sys/devices/system/cpu/cpu5/online
        echo 0 > /sys/devices/system/cpu/cpu6/online
        echo 0 > /sys/devices/system/cpu/cpu7/online


# Enable low power modes
#允许低电量模式
        echo 0 > /sys/module/lpm_levels/parameters/sleep_disabled

# HMP scheduler (big.Little cluster related) settings
#大小核阈值设置
        echo 75 > /proc/sys/kernel/sched_upmigrate
        echo 60 > /proc/sys/kernel/sched_downmigrate

# cpu idle load threshold
#CPU空载阈值
        echo 30 > /sys/devices/system/cpu/cpu0/sched_mostly_idle_load
        echo 30 > /sys/devices/system/cpu/cpu1/sched_mostly_idle_load
        echo 30 > /sys/devices/system/cpu/cpu2/sched_mostly_idle_load
        echo 30 > /sys/devices/system/cpu/cpu3/sched_mostly_idle_load
        echo 30 > /sys/devices/system/cpu/cpu4/sched_mostly_idle_load
        echo 30 > /sys/devices/system/cpu/cpu5/sched_mostly_idle_load
        echo 30 > /sys/devices/system/cpu/cpu6/sched_mostly_idle_load
        echo 30 > /sys/devices/system/cpu/cpu7/sched_mostly_idle_load

# Enable core control
#允许核心控制
        insmod /system/lib/modules/core_ctl.ko

#貌似是控制大核上线数
        echo 2 > /sys/devices/system/cpu/cpu0/core_ctl/min_cpus
        echo 4 > /sys/devices/system/cpu/cpu0/core_ctl/max_cpus
#上线限值
        echo 68 > /sys/devices/system/cpu/cpu0/core_ctl/busy_up_thres
#离线限值
        echo 40 > /sys/devices/system/cpu/cpu0/core_ctl/busy_down_thres
#离线延迟时间，单位毫秒
             echo 100 > /sys/devices/system/cpu/cpu0/core_ctl/offline_delay_ms
```




# 官方默认调度模式解析
target_loads         合理值50~95

当cpu负载超过target_loads时，cpu提升频率，越小频率提升越快，流畅度越好，更耗电

可以对应频率分别设置各个频率时的target_load，比如75 918000:80 1350000:85 这表示，

小于918000的频率target_load为75;

918000到1350000之间的频率target_load为80;

1350000以上的频率target_load为85;

甚至可以对应到每一个频率，但是，采用for循环遍历表的方式实现，效能太差，人为的设置频率的target_load也一点不智能合理。

故删除此参数，新增target_load_base，target_load_slope参数

go_hispeed_load  合理值80~99

hispeed_freq        合理值 （最低+最高）*2/5 到 （最低+最高）/2

当cpu负载超过go_hispeed_load时，直接提升频率到hispeed_freq  

不要以为hispeed_freq设的越高性能越好，hispeed_freq还在降频的时候起到阻挠一次频率下降过快的作用，设的太高就失去了这个意义。

这三个值是这个调度器最重要的参数，
- 追求性能，target_loads 55，go_hispeed_load 85
- 追求平衡，target_loads 70，go_hispeed_load 90
- 追求省电，target_loads 85，go_hispeed_load 95

target_load_base      合理值50~95

target_load的基准值，与上面的target_loads相同，删除了对应频率设置的功能。

target_load_slope    合理值54000~324000

target_load斜率的倒数。频率每提升一个target_load_slope，target_load提升1。

实际对应频率的target_load = target_load_base + 频率/target_load_slope

这个参数的作用就是，频率低的时候升频快，提升响应，频率高的时候抑制频率提升。

设置为0，则所有频率的target_load为target_load_base

若设置为负数，则频率低的时候升频慢，频率高的时候快速提升，当然这种方式好像不合理。

target_load_base 必须小于 100 - 最高频率/target_load_slope，才能确保频率可以达到最高频率。

当target_load等于100时，频率就不会再往上提升，如果有需要的话，甚至可以结合target_load_base和target_load_slope来限制最高频率。

最好为54000的倍数，因为msm8260 cpu的每个频率间隔为54000

### io_is_busy

非零开启，开启后，计算cpu负载的时候，把io等待时间算作cpu负载的时间。开了这一项等同于开启了高性能模式，频率飙升。这也侧面体现了米1性能的不够用，追求响应速度的可以开。其实在高性能的手机里这一项都是打开的，米1不建议开启，米1的发热和续航控制不住，宁可调低target_loads 和 go_hispeed_load 也不开启这一项。

由于开启这一项太耗电，关闭这一项响应太差，故删除此参数，新增iowait_div参数，来细微的设置。

### iowait_div     常用值-1，0，1，2

iowait时间的右移值。0则表示iowait时间全部算为空闲。

1则iowait/2为空闲时间，2则iowait/4为空闲时间，3则表示iowait/8为空闲时间，以此类推。

越大负载计算的越高，频率越容易提升，性能越好。

为-1时，所有iowait时间都不算为空闲，性能最佳。

0等同于关闭io_is_busy，-1等同于开启io_is_busy。

最常用的也就-1，0，1，2这几个，其他数值基本不用。

### timer_rate                                单位：微妙

提升频率的采样时间

above_hispeed_delay                单位：微妙

提升频率到hispeed_freq之上的等待时间

这两个值一般默认20000即可，追求性能的可以设置成10000

### timer_slack                                单位：微妙

空闲时的最大等待时间，-1关闭，默认即可

### min_sample_time                        单位：微妙

频率下降前的最小采样时间，默认即可

sampling_down_factor        单位：微妙

当频率最高时，sampling_down_factor会替代min_sample_time，成为频率下降前的最小采样时间，默认0即为min_sample_time

### boost

非零等同于将最低频率直接设置为hispeed_freq，一般这一项不开设为零

### sync_freq

合理值：等于或者略低于up_threshold_any_cpu_freq，0为关闭此功能

### up_threshold_any_cpu_load

合理值：70~85左右

### up_threshold_any_cpu_freq

合理值：（最低+最高）*2/3

这三项是一块的，当频率小于sync_freq时，如果其他核心的频率大于up_threshold_any_cpu_freq，并且负载大于up_threshold_any_cpu_load，就将当前核心的频率提升到sync_freq

对于异步cpu来说，这是一个非常有用的同步cpu频率的功能，有效的解决了两个异步核心切换时频率差距太大引起的忽快忽慢的问题。

其实还不够理想，不过也足够了。理想情况应该是设置一个同步率，来限制低频核心和高频核心之间的频率最大差距百分比。

要省电的sync_freq设为0，关闭此功能



命令行别看，路径别看！！！

### 关闭mpdecision

Snapdragon有一个叫做mpdecision的程序管理CPU各个核心的开、关和频率。所以如果想手动开关CPU的核心或者设置CPU核心的频率就必须把这个程序关闭。


#### stop mpdecision

需要注意的是，这个程序会在每次启动后执行，所以每次重启后都需要重新执行上面的命令停止mpdecisiopn。

### 设置CPU的核心数

在/sys/devices/system/cpu目录下可以看到你的CPU有几个核心，如果是双核，就是cpu0和cpu1，如果是四核，还会加上cpu2和cpu3。

随便进一个文件夹，比如cpu1，里面有个online文件。我们可以用cat命令查看该文件的内容

    cat /sys/devices/system/cpu/cpu1/online

这个文件只有一个数字，0或1。0表示该核心是offline状态的，1表示该核心是online状态的。所以，如果你想关闭这个核心，就把online文件的内容改为“0”；如果想打开该核心，就把文件内容改为“1”。

    echo "0" > /sys/devices/system/cpu/cpu1/online # 关闭该CPU核心

    echo "1" > /sys/devices/system/cpu/cpu1/online # 打开该CPU核心

### 设置CPU的频率

首先我们要修改governor的模式，但在修改前需要查下CPU支持哪些governor的模式

cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors

我用的是Nexus 4手机，所以有以下5个选择，其他的手机型号可能略有不同

### interactive ondemand userspace powersave performance

这里performance表示不降频，ondemand表示使用内核提供的功能，可以动态调节频率，powersvae表示省电模式，通常是在最低频率下运行，userspace表示用户模式，在此模式下允许其他用户程序调节CPU频率。

在这里，我们将模式调整为“userspace”。

echo "userspace" > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

然后我们对CPU的频率进行修改，CPU的频率不是可以任意设置的，需要查看scaling_available_frequencies文件，看CPU支持哪些频率。

cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies

从我的手机中可以获得以下的值

384000 486000 594000 702000 810000 918000 1026000 1134000 1242000 1350000 1458000 1512000

这里的频率是以Hz为单位的，我准备将cpu0设置为1.242GHz，那就将1242000写入scaling_setspeed即可。

echo "1242000" > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed

设置好后，我们可以通过scaling_cur_freq文件查看当前这个核心的频率

cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

最后我们也可以设置下CPU的最大和最小频率，只需要将需要设置的频率值写入scaling_max_freq和scaling_min_freq即可

echo "1350000" > /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq # 设置最大频率

echo "384000" > /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq # 设置最小频率

这里要注意的是“最大值”需要大于等于“最小值”。

注意，这里设置的仅为某个CPU核心的频率，你需要对每个online的CPU核心都进行设置，同时以上对文件的修改均需要root权限。

通过减少online的核心数和限制CPU频率固然可以起到节省电量的目的，但是性能也是显著降低，所以需要做一个权衡。


## stune

echo 15 > dev/stune/foreground/schedtune.boost
echo 30 > dev/stune/top-app/schedtune.boost
