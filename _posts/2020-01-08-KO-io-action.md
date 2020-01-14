---
layout: article
title: KO io action
key: 20200108
tags:
  - io
lang: zh-Hans
---

# KO io action

## FR 8751721

## 一、课题: [监控与异常修复]IO异常  

## 二、实现Comment: 

整体思路：从两个方面来进行： 

### 监测： 
- 1、参考MTK通过dump打开log开关的方式，增加接口，使user版本软件可以通过adb或者apk工具快速获取到blockio log与cmdq的ftrace 
- 2、开发APK，添加多种复杂IO情况的模拟，并增加抓取ftrace的功能，读取blockio log图例展示等功能，用于分析问题时查看历史的IO等情况。 
    - 丁鹏
- 3、移植iotop检测进程异常占用IO，TOKYO_TF验证已经有iotop指令， 后续开放给user版本使用 
    - 已有
- 4、移植：异常时抓取iostat，分析磁盘情况；（busybox）  Notes:长时间监控可能有性能损耗，建议仅在异常发生时记录/上报； 通过上报kernellog，服务端用MTK插件解析； 
    - 0108
- 5、调研：找华为机器研究下华为检测方案 
- 6、emmc驱动里面出现的IO异常LOG，需要检出查看分析 
- 7、以上监控异常信息可能需要某个统一的监测框架在底层或者FWK进行数据持久化或分析。 同时也需要通过某个渠道发送到服务器进行整体分析。如服务器收集到的信息80%的用户会发生此错误，则一定是问题。  

### 修复措施想法： 
- 1、IO压力大时减少readahead值，异常消失后恢复； 
- 2、重建file cache（文件系统层面） 
- 3、强制释放非脏文件页（内存层面）  

#### CX____:
- 1, 在现有的perf_auto_test中加入iotop/iostat数据记录和分析
- 2, 使用perf_auto_test中记录的io信息进行blkio的调优
- 3, 研究一下blktrace是否有用，如何使用

### 三、细项规划： 
任务|目标|计划完成时间|实际完成时间|交付物 
-|-|-|-|-
监测手段完善|user版本可通过adb指令抓取blockio log/cmdq ftrace/iotop/iostat/kernellog等信息|2020-4-30||代码提交 
模拟工具开发|apk工具开发完毕，并支持自动创造多种复杂IO情况，用于复现问题；|2020-5-29||填充工具 
上报机制|系统发生异常时上报异常信息即调试信息|2020-6-30||服务器后端与Android前端服务 
生产前推广|推广试用，并根据反馈意见改进工具；|2020-7-31||训练模型及数据 
可视化工具开发|将抓取到的log/trace可视化呈现|2020-7-31||分析工具 
生产交付|定版，课题结案报告|2020-8-28||课题结案报告与推广邮件  

## 四、人员设定： ARCI： GCSSAT 验收： hongyi.li / jianhua.he  

## 五、注意： 

- 1、由于技术原因本题Desc仅代表创建时最新描述。描述如有更新请寻 此题下方Comment 或 总Doc 8751714 
- 2、实际规划节点可能提前，请留意TL发布的信息(所有发布信息会同步到8751714 & 此题下方Comment)。 
- 3、实际实现过程中可以证伪，或转变方向，或增加减少子Action等等（所有信息请同步此题下方Comment & TL汇总到8751714） 
- 4、KO总表 & 其他约定 & 总体管理2020KO事项和文档等 参考在：总Doc 8751714 [SATD-2020][KO]Performance tech. improvement Doc （请留意） 

## Moving:

### blktrace

开始收集trace

    blktrace -d /dev/block/mtdblock1 -o /tmp/data1 -w 60

分析trace

    blkparse -i /tmp/data1.blktrace.0 