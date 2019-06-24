---
layout: article
title: lmkd
key: 20190630
tags:
  - lmkd
lang: zh-Hans
---

# Android的lowmemorykiller演变分析

在学习Android的lowmemorykiller机制过程中，发现从KK到L再到M有一些新的变化，因此有必要进行一下总结。在开始分析前先厘清一些基础概念，便于描述的展开和结论的形成。本文内容并没涉及太多具体实现细节，目的只是为实际的开发提供纲领性指导意见。

## 一、基本概念

- 1、PageCache：Linux内核为加快文件预取而采用的特有机制(参考引用一)，就是尽可能的把空闲内存用于缓存最近访问过的磁盘文件数据。由于会占用大量的空闲内存，在某种情况下就会导致OOM的发生。Linux内核的内存管理是很重要的部分，需要花很多时间和精力来学习(参考引用二)。

- 2、OOM：即“Out of Memory”的缩写，这个其实是标准Linux内核的一种内存管理机制，在内存不足或无法分配(大)内存的时候会触发此机制，来完成内存的回收。具体到代码级别，内存页面回收机制是Cache Shrinker，通过内核线程kswapd监控内存页面情况和触发Shrinker回调函数进行内存页面回收。

- 3、lowmemorykiller(简称LMK，下同)：是Android基于标准Linux内核OOM机制修改而来，最初的方式是通过在内核实现一个staging形式的驱动，在此驱动中注册Shrinker回调函数和实现应用策略，剩下的工作就是定时检查及判断是否需要回收(或定时检测和内核触发相结合???)。关于OOM和LMK详细信息可参考引用三。

- 4、CGroup：简单的说就是对进程线程使用各种系统资源的分组管理和控制，包括如CPU、IO、MEMORY，等等。

- 20160308补充：

关于Linux的OOM和Android的LMK的差异对比在引用八有详细介绍。


## 二、LMK实现演变

- 1、在Android的KK(或之前)版本中，LMK机制和策略都是在内核空间驱动中实现的，驱动提供两个sys文件结点给AMS用于设置触发内存回收的阀值。

- 2、在Android的L(或之后)版本中，引入了一个处于用户空间的守护进程lmkd(参考引用四)，LMK机制和策略的实现多了一种选择---即可在用户空间完成。但不管是旧的还是新的方式，原来由AMS完成的设置工作都通过socket方式传递参数给lmkd来实际完成。这样一来AMS与内核的耦合性进一步降低，把精力关注于activity的管理。

## 三、lmkd进程分析

关于lmkd的作用在修改日志中很明确：
```shell
Author: Todd Poynor <toddpoynor@google.com>
Date:   Tue Jul 9 19:35:14 2013 -0700

    Add lmkd low memory killer daemon
    
    Move kernel low memory killer logic to new daemon lmkd.  ActivityManager
    communicates with this daemon over a named socket.
```

为什么导入lmkd，根据我的经验分析一来在用户空间实现策略更容易优化和管控，二来LMK驱动是Android特有的东西不太容易进入Linux内核的正式驱动列表(一直是staging形式)。在google groups的一个讨论组(android-platform)的贴子中也提到了类似的观点(引用五，需自备楼梯)：

Dual supported in L. For example N9 released with lmk due to its advanced architecture
giving an edge to lmk over lmkd. Low memory killing is an important feature on all 
Android Devices and its tuning and optimization is a requirement to provide the best 
and balanced user experience. The option of either improves the tuning options of the 
device manufacturers. Support for lmk scales because of its inclusion upstream in the 
kernel. Our goal is to improve lmkd as an optimized replacement with algorithms that 
live well in the android ecosystem.

考虑到使用Android版本内核的差异性，lmkd进程的实现是兼容了新旧两种不同方式的：

- (1) 使用LMK驱动(旧的方式)，配置阀值参数保存在驱动中，LMK策略由驱动实现，内核通过Shrinker机制触发策略执行；
- (2) 不使用LMK驱动(新的方式)，配置阀值参数保存在lmkd进程中，LMK策略由lmkd进程实现。但为了触发策略的执行，必须定制内核加入memory cgroup，通过监控memory pressure event来触发LMK策略执行；

根据lmkd的实现意图，理论上来说应该使用第二种方式才对；

20160308补充：
        在引用七中，对于引入LMK守护进程的意图及其相比于LMK驱动的优势(比如优雅地结束进程和回收资源，更快的响应和处理low memory状况避免linux的oom触发，正确判断cache和计算free memory)有详细的介绍。有意思的是，这个邮件是由memory pressure event的提交人Anton Vorontsov发出的，让我不禁觉得他就是lmkd的发起者。


## 四、memory pressure event

这个其实也是标准Linux内核在2013年4月就已经实现的功能(提交者为Anton Vorontsov，patch的标题是“memcg: Add memory.pressure_level events”)。基于memory cgroup，内核可以分析当前内存的使用情况压力等级(low、medium、critical)，并把相关信息通过eventfd方式通知给用户空间的监听者执行相关动作(如内存回收)。
        
memory pressure event的实现(引用六)经过四次review(多人参与讨论，包括Linus Torvalds、Andrew Morton等大boss)才进入mainline，不得不配服国外开源环境的活跃及严谨治学态度。在这一系列讨论中也有人提及Android相关考虑，作者的想法就是实现一个通用的方式(包括对Android等移动设备支持)。

从lmkd是2013年7月提交时间来看，借助memory pressure event方式的实现，google终于如愿以标准Linux内核方式消除了staging形式LMK驱动这个“鱼刺”。至于使用memory pressure event方式的内存回收效率及实际效果如何，未进行相关测试前就不得而知了。

## 五、引用

- 1、[linux的page cache策略](http://blog.csdn.net/fall221/article/details/46290563)

- 2、[linux内存管理各文件简介](http://blog.csdn.net/u011955950/article/details/18860379)

- 3、[android Low Memory Killer介绍](http://blog.csdn.net/hgl868/article/details/6744883)

- 4、[Android——内存管理-lowmemorykiller 机制](http://blog.csdn.net/jscese/article/details/47317765)

- 5、[why lmkd and logd are created to move log and lmk from kernel to userspace in L?](https://groups.google.com/forum/#!topic/android-platform/PW85I7qRBP8)

- 6、memcg: Add memory.pressure_level events
[1](https://patchwork.kernel.org/patch/2123031/)
[2](https://patchwork.kernel.org/patch/2161601/)
[3](https://patchwork.kernel.org/patch/2318281/)
[4](https://patchwork.kernel.org/patch/2384491/)

- 7、[Userspace low memory killer daemon](https://lwn.net/Articles/511731/)

- 8、[What is the difference between low memory killer and out of memory killer?](https://www.quora.com/What-is-the-difference-between-low-memory-killer-and-out-of-memory-killer)


### [Source URL](https://blog.csdn.net/alien75/article/details/53322769)


# Lmkd pressure值计算(Android lmkd计算核心)
```java
Pressure=memory.usage_in_bytes*100/memory.memsw.usage_in_bytes
```
注：
memory.usage_in_bytes=cached+anonrss

memory.memsw.usage_in_bytes=cached+anonrss+swapuse

kernel会通过计算vmpressure的值当该值发生变化时（一般为low，当其由low变为medium或者有medium变为critical）会触发lmkd回调函数来执行进程清理工作。Kernel计算pressure的公式为：

```java
pressure= [1 – (reclaimed/scanned)] * 100
```

Scanned应该是固定时间段扫描的memory page个数，reclaim表示当前扫描的内存页数中可回收的memorypage个数，从公式可以看出压力值就是当前已扫描的内存中不可回收内存的比例。

然后kernel会根据得到的当前压力值来得出memory的压力状态时low还是medium或者critical：

```java
static enumvmpressure_levelsvmpressure_level(unsignedlong pressure)
{
if(pressure >= vmpressure_level_critical) //vmpressure_level_critical=95
returnVMPRESSURE_CRITICAL;
elseif (pressure >= vmpressure_level_med) //vmpressure_level_med=60
returnVMPRESSURE_MEDIUM;
returnVMPRESSURE_LOW;
}
```

### [Source URL](https://blog.csdn.net/zsj100213/article/details/79499297)


# Android O&Go lmkd执行流程

我们知道android lowmemorykiller机制有两套执行方案，在N之前的版本都是采用的kernel的lowmemorykiller.c里面的方式。最近查看了Android Go的代码结构发现，Android Go采用的是native的lmkd service的方式来起到lowmemorykiller的作用。具体实现流程如下：

我们知道kernel lowmemorykiller启动杀进程的条件是file_page-shmem-swapcache(-unevictable)的值低于AMS所预设的minfree的各档的阈值。这里如果是采用的lmkd的方式，则采用的是另外一套机制启动lmkd来杀进程。

首先讲一下lmkd的执行流程，其主要代码位于/system/core/lmkd/lmkd.c,其执行流程如下：

```cpp
int main(int argc __unused, char **argv __unused) {
    struct sched_param param = {
            .sched_priority = 1,
    };

//………………

    if (!init())//启动init()初始化函数
        mainloop();//进入mainloop，检测是否有所监听的事件发生变化，如有，则调用event poll回调函数

    ALOGI("exiting");
    return 0;

static int init(void) {
    struct epoll_event epev;
    int i;
    int ret;

//………………

    ctrl_lfd = android_get_control_socket("lmkd");//lmkd socket通信，获取AMS设置的minfree和adj的值
    if (ctrl_lfd < 0) {
        ALOGE("get lmkd control socket failed");
        return -1;
    }

    ret = listen(ctrl_lfd, 1);
    if (ret < 0) {
        ALOGE("lmkd control socket listen failed (errno=%d)", errno);
        return -1;
    }

    epev.events = EPOLLIN;
    epev.data.ptr = (void *)ctrl_connect_handler;//epoll callback，当socket里面有数据变化是调用此函数
    if (epoll_ctl(epollfd, EPOLL_CTL_ADD, ctrl_lfd, &epev) == -1) {//将ctrl_lfd添加到epollfd中去，当其有变化是，启东callback函数
        ALOGE("epoll_ctl for lmkd control socket failed (errno=%d)", errno);
        return -1;
    }
    maxevents++;

    has_inkernel_module = !access(INKERNEL_MINFREE_PATH, W_OK);
    use_inkernel_interface = has_inkernel_module && !is_go_device;//后面用来判断是否采用kernel的lowmemorykiller机制

    if (use_inkernel_interface) {
        ALOGI("Using in-kernel low memory killer interface");
    } else {
        ret = init_mp_medium();//如果不采用kernel lowmemorykiller机制，则执行此处初始化进程
        ret |= init_mp_critical();
        if (ret)
            ALOGE("Kernel does not support memory pressure events or in-kernel low memory killer");
    }

//………………

    return 0;
}
```

如果使用kernel的lowmemorykiller机制，则此时监听的lmkd socket会传输AMS设置的minfree和adj值，设置函数如下：
```cpp
static void cmd_target(int ntargets, int *params) {

//………………

    for (i = 0; i < ntargets; i++) {//将参数赋给本地变量
        lowmem_minfree[i] = ntohl(*params++);
        lowmem_adj[i] = ntohl(*params++);
    }

//…………

 if (has_inkernel_module) {
    char minfreestr[128];
        char killpriostr[128];

        minfreestr[0] = '\0';
        killpriostr[0] = '\0';

        for (i = 0; i < lowmem_targets_size; i++) {
            char val[40];

            if (i) {
                strlcat(minfreestr, ",", sizeof(minfreestr));
                strlcat(killpriostr, ",", sizeof(killpriostr));
            }

            snprintf(val, sizeof(val), "%d", use_inkernel_interface ? lowmem_minfree[i] : 0);//通过use_inkernel_interface决定是否采用kernel lowmemorykiller机制 
            strlcat(minfreestr, val, sizeof(minfreestr));
            snprintf(val, sizeof(val), "%d", use_inkernel_interface ? lowmem_adj[i] : 0);
            strlcat(killpriostr, val, sizeof(killpriostr));
        }

        writefilestring(INKERNEL_MINFREE_PATH, minfreestr);//往kernel节点里面写值
        writefilestring(INKERNEL_ADJ_PATH, killpriostr);
    }
```

而本文讲的是在Android Go上面采用另外的lmkd的方式来实现lowmemorykiller机制，下面是他的初始化：
```cpp
static int init_mp_common(char *levelstr, void *event_handler, bool is_critical)
{
    int mpfd;
    int evfd;
    int evctlfd;
    char buf[256];
    struct epoll_event epev;
    int ret;
    int mpevfd_index = is_critical ? CRITICAL_INDEX : MEDIUM_INDEX;
 
    mpfd = open(MEMCG_SYSFS_PATH "memory.pressure_level", O_RDONLY | O_CLOEXEC);//memory.presure_level句柄
    if (mpfd < 0) {
        ALOGI("No kernel memory.pressure_level support (errno=%d)", errno);
        goto err_open_mpfd;
    }
 
    evctlfd = open(MEMCG_SYSFS_PATH "cgroup.event_control", O_WRONLY | O_CLOEXEC);//cgroup.event_control句柄
    if (evctlfd < 0) {
        ALOGI("No kernel memory cgroup event control (errno=%d)", errno);
        goto err_open_evctlfd;
    }
 
    evfd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);//创建事件描述符，后面需要启动lmkd会通过往此文件描述符发信号来实现
    if (evfd < 0) {
        ALOGE("eventfd failed for level %s; errno=%d", levelstr, errno);
        goto err_eventfd;
    }
 
    ret = snprintf(buf, sizeof(buf), "%d %d %s", evfd, mpfd, levelstr);//将关心的事件描述符，文件节点句柄和当前内存缺少状态写到buf中
    if (ret >= (ssize_t)sizeof(buf)) {
        ALOGE("cgroup.event_control line overflow for level %s", levelstr);
        goto err;
    }
 
    ret = write(evctlfd, buf, strlen(buf) + 1);//将上面buf的内容写到cgroup.event_control,调用这个函数会触发后面要讲到的event_control的memcg_write_event_control函数执行
    if (ret == -1) {
        ALOGE("cgroup.event_control write failed for level %s; errno=%d",
              levelstr, errno);
        goto err;
    }
 
    epev.events = EPOLLIN;
    epev.data.ptr = event_handler;//设置回调函数
    ret = epoll_ctl(epollfd, EPOLL_CTL_ADD, evfd, &epev);//将当前的监听的事件添加到监听事件列表中
    if (ret == -1) {
        ALOGE("epoll_ctl for level %s failed; errno=%d", levelstr, errno);
        goto err;
    }
    maxevents++;
    mpevfd[mpevfd_index] = evfd;
    return 0;
 
err:
    close(evfd);
err_eventfd:
    close(evctlfd);
err_open_evctlfd:
    close(mpfd);
err_open_mpfd:
    return -1;
}
```

当往cgroup.event_control函数里面写event_fd,memory.pressure_level,memory缺少状态时，会执行以下函数：

```cpp
static ssize_t memcg_write_event_control(struct kernfs_open_file *of,
					 char *buf, size_t nbytes, loff_t off)
{
//………………
	efd = simple_strtoul(buf, &endp, 10);//也就是上文的evfd
	if (*endp != ' ')
		return -EINVAL;
	buf = endp + 1;
 
	cfd = simple_strtoul(buf, &endp, 10);//要监听的文件节点的fd
	if ((*endp != ' ') && (*endp != '\0'))
		return -EINVAL;
	buf = endp + 1;
//………………
	efile = fdget(efd);//获取当前fd对应的文件结构体
	if (!efile.file) {
		ret = -EBADF;
		goto out_kfree;
	}
 
	event->eventfd = eventfd_ctx_fileget(efile.file);//将当前efd赋值给event->eventfd
	if (IS_ERR(event->eventfd)) {
		ret = PTR_ERR(event->eventfd);
		goto out_put_efile;
	}
 
	cfile = fdget(cfd);//获取cfd对应的file，也就是memory.pressure_level
	if (!cfile.file) {
		ret = -EBADF;
		goto out_put_eventfd;
	}
//………………
	} else if (!strcmp(name, "memory.pressure_level")) {
		event->register_event = vmpressure_register_event;//event注册
		event->unregister_event = vmpressure_unregister_event;
//………………
	ret = event->register_event(memcg, event->eventfd, buf);执行注册函数
	if (ret)
		goto out_put_css;
//………………
spin_lock(&memcg->event_list_lock);
	list_add(&event->list, &memcg->event_list);//将当前的event添加到memcg的event_list里面
	spin_unlock(&memcg->event_list_lock);
//………………
}

int vmpressure_register_event(struct mem_cgroup *memcg,
			      struct eventfd_ctx *eventfd, const char *args)
{
//………………
ev->efd = eventfd;将上文提到的eventfd赋值给efd
	ev->level = level;
 
	mutex_lock(&vmpr->events_lock);
	list_add(&ev->node, &vmpr->events);//将当前ev->node添加到vmpr->event链表
	mutex_unlock(&vmpr->events_lock);
//………………
}
```

以上就是lmkd的完整初始化流程，从以上我们可以看出，主要是监听memory.pressure_level节点的变化，那么合适会触发监听的回调函数的执行呢？往下看：
在shrink_zone中有如下操作：

```cpp
static bool shrink_zone(struct zone *zone, struct scan_control *sc,
			bool is_classzone)
{
//………………
		vmpressure(sc->gfp_mask, sc->target_mem_cgroup,
			   sc->nr_scanned - nr_scanned,
			   sc->nr_reclaimed - nr_reclaimed);//vmpressure函数执行，往下看
//……………………
}

void vmpressure(gfp_t gfp, struct mem_cgroup *memcg,
		unsigned long scanned, unsigned long reclaimed)
{
//………………
	schedule_work(&vmpr->work);//启动work的执行函数vmpressure_work_fn
}

static void vmpressure_work_fn(struct work_struct *work)
{
//………………
vmpressure_event(vmpr, scanned, reclaimed)
//………………
}
static bool vmpressure_event(struct vmpressure *vmpr,
			     unsigned long scanned, unsigned long reclaimed)
{
//………………
level = vmpressure_calc_level(scanned, reclaimed);//vmpressure level计算
list_for_each_entry(ev, &vmpr->events, node) {
		if (level >= ev->level) {
			eventfd_signal(ev->efd, 1);//触发lmkd回调函数开始工作
			signalled = true;
		}
	}
//………………
}
```

以上就是lmkd整个执行流程，部分可能理解有误，还请多多指正。从目前的代码来看，Android Go用到了当前的lmkd的执行。



### [Source URL](https://blog.csdn.net/zsj100213/article/details/78974979)


#### Other reference resources

[Android 6.0的lowmemorykiller机制](https://blog.csdn.net/u012440406/article/details/51960387)

[Android——内存管理-lowmemorykiller 机制](https://blog.csdn.net/jscese/article/details/47317765)
