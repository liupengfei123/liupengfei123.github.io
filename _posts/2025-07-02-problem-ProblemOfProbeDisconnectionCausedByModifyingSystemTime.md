
---
layout:     page
title:      "修改系统时间导致探针掉线问题"
subtitle:   "修改系统时间，线程无法被唤起"
date:       2025-07-02
author:     "Lpf"
header-img: "img/home-bg.jpg"
tags:
    - 问题排查
    - JDK
---

# 问题描述

系统在每天凌晨的时候会将系统时间向前回拨一天的日期。如06-19 00：00时将时间更改未06-18 00：00。

在修改系统时间之后，发现探针日志就没有在打印日志了。

怀疑修改系统时间之后影响到了 jdk中ScheduledThreadPoolExecutor定时任务的调度，并且尝试在本地复现，未能成功。

# 环境复现

因为本地无法复现所以尝试写个demo在客户环境中复现：

<img src="{{site.baseurl}}/img/in-post/problem-ProblemOfProbeDisconnectionCausedByModifyingSystemTime/wps1.jpeg" alt="img" style="zoom:150%;" />

<img src="{{site.baseurl}}/img/in-post/problem-ProblemOfProbeDisconnectionCausedByModifyingSystemTime/wps2.jpeg" alt="img" style="zoom:150%;" />

在demo中使用 ScheduledThreadPoolExecutor 设置了两个每隔60秒执行一次的定时任务。

在 7月2号下午5点左右（系统时间6月18号15：37）启动，在7月3号9-10点左右获取数据。

## 复现现象

在输出日志中 打印的时间戳只有 6月18号15：37 到6月19号0点的日志。

![img]({{site.baseurl}}/img/in-post/problem-ProblemOfProbeDisconnectionCausedByModifyingSystemTime/wps3.jpeg)

![img]({{site.baseurl}}/img/in-post/problem-ProblemOfProbeDisconnectionCausedByModifyingSystemTime/wps4.jpeg)

查看线程栈信息，发现定时任务卡在 LockSupport.parkNanos 方法中：

<img src="{{site.baseurl}}/img/in-post/problem-ProblemOfProbeDisconnectionCausedByModifyingSystemTime/wps5.jpeg" alt="img" style="zoom:150%;" />

在查看堆栈信息，其任务对象还存在。

<img src="{{site.baseurl}}/img/in-post/problem-ProblemOfProbeDisconnectionCausedByModifyingSystemTime/wps6.jpeg" alt="img" style="zoom:150%;" />

<img src="{{site.baseurl}}/img/in-post/problem-ProblemOfProbeDisconnectionCausedByModifyingSystemTime/wps7.jpeg" alt="img" style="zoom:150%;" />

# 问题分析

所以通过上述现象怀疑是 修改系统时间影响到了System.nanoTime()纳秒值的获取，或者影响到park的唤醒。

通过 arthas的 ognl 表达式去执行 System.nanoTime() 方法等到：

![img]({{site.baseurl}}/img/in-post/problem-ProblemOfProbeDisconnectionCausedByModifyingSystemTime/wps8.jpeg)

发现纳秒时间没有问题。

在通过计算定时任务的纳秒间隔：

<img src="{{site.baseurl}}/img/in-post/problem-ProblemOfProbeDisconnectionCausedByModifyingSystemTime/wps9.jpeg" alt="img" style="zoom:150%;" />

<img src="{{site.baseurl}}/img/in-post/problem-ProblemOfProbeDisconnectionCausedByModifyingSystemTime/wps10.jpeg" alt="img" style="zoom:150%;" />

在19号0点时执行最后一次定时任务之后更新了任务执行的时间戳之后，就在也没有执行任务了。

 

通过时间在当前获取的纳秒进行计算，已经超过了定时任务执行的time值。所以在结合线程栈可以得出是修改了系统时间之后影响到了jdk的LockSupport#parkNanos 唤醒操作。

# 影响原因

https://bugs.java.com/bugdatabase/view_bug?bug_id=6311057

https://bugs.java.com/bugdatabase/view_bug?bug_id=7139684

https://bugs.java.com/bugdatabase/view_bug?bug_id=6900441

为jdk的bug：在JDK-6900441：Linux 上的 PlatformEvent.park(millis) 仍然可能受到时钟变化的影响

修复版本：![img]({{site.baseurl}}/img/in-post/problem-ProblemOfProbeDisconnectionCausedByModifyingSystemTime/wps11.jpeg)

# 总结

在jdk1.6-45的linux 版本中修改系统时间影响到了 LockSupper.parkNanos 的唤醒，导致定时任务无法启动，所以导致探针中的任务无法执行，进而导致了探针掉线。

