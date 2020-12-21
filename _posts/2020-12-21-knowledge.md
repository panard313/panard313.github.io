---
layout: article
title: Network Note 网络笔记
key: 20201221
tags:
  - knowledge
  - cx
lang: zh-Hans
---

# Android



# performance

### PSI

#### PSI 用户接口定义

每类资源的压力信息都通过 proc 文件系统的独立文件来提供，路径为 /proc/pressure/ -- cpu, memory, and io.

其中 CPU 压力信息格式如下：

  some avg10=2.98 avg60=2.81 avg300=1.41 total=268109926

memory 和 io 格式如下：

  some avg10=0.30 avg60=0.12 avg300=0.02 total=4170757

  full avg10=0.12 avg60=0.05 avg300=0.01 total=1856503

avg10、avg60、avg300 分别代表 10s、60s、300s 的时间周期内的阻塞时间百分比。total 是总累计时间，以毫秒为单位。

some 这一行，代表至少有一个任务在某个资源上阻塞的时间占比，full 这一行，代表所有的非idle任务同时被阻塞的时间占比，这期间 cpu 被完全浪费，会带来严重的性能问题。我们以 IO 的 some 和 full 来举例说明，假设在 60 秒的时间段内，系统有两个 task，在 60 秒的周期内的运行情况如下图所示：

一般只关注some, full一旦有数值, 系统压力就超高


# XV6



# debug


# C


# java
