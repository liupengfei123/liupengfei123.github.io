---
layout:     page
title:      "客户应用logback线程无法唤醒问题"
subtitle:   "客户应用logback线程无法唤醒问题"
date:       2025-06-02
author:     "Lpf"
header-img: "img/home-bg.jpg"
tags:
    - 问题排查
---

# 背景

根据客户反馈：系统卡死，系统无法工作。

# 数据分析

## 线程分析
通过jstack获取系统的线程运行栈信息：

发现所有的业务线程都在等待logback的put中等待：

![image-20251204205118560]({{site.baseurl}}/img/in-post/article0011/image-20251204205118560.png)

再查看logback消费线程也在等待异步队列的数据。

![image-20251204205128519]({{site.baseurl}}/img/in-post/article0011/image-20251204205128519.png)

发现一个ArrayBlockingQueue队列实例，在put和take方法相互阻塞了，这是相矛盾的。

即：

![image-20251204205136685]({{site.baseurl}}/img/in-post/article0011/image-20251204205136685.png)

![image-20251204205144014]({{site.baseurl}}/img/in-post/article0011/image-20251204205144014.png)

因为不知名的原因导致 在 notEmpty 和 notFull 这两个同时 休眠了，进而无法消费打印日志，所有的因为全部阻塞在了 日志这里，所有业务线程越积越多


## 堆数据分析

在通过dump分析其中的数据：

![image-20251204205252443]({{site.baseurl}}/img/in-post/article0011/image-20251204205252443.png)

这个 9点34 分的日志所在的线程还阻塞在这里。


在查看该 notEmpty属性中的 firstWaiter 是为空的，这个表明队列是有执行了Signal方法的了。

![image-20251204205319947]({{site.baseurl}}/img/in-post/article0011/image-20251204205319947.png)


![image-20251204205328777]({{site.baseurl}}/img/in-post/article0011/image-20251204205328777.png)

![image-20251204205336961]({{site.baseurl}}/img/in-post/article0011/image-20251204205336961.png)


但是Log Work线程的parkBlocker属性还存在。

![image-20251204205409554]({{site.baseurl}}/img/in-post/article0011/image-20251204205409554.png)


![image-20251204205351433]({{site.baseurl}}/img/in-post/article0011/image-20251204205351433.png)


说明 AsyncApperend-Work 线程在 java.util.concurrent.locks.LockSupport#park(java.lang.Object) 中没有被唤醒。

# 总结

可能是因为JVM的bug导致了，阻塞线程无法被唤醒。

如：在jdk1.6-45的linux版本中修改系统时间影响到了LockSupper.parkNanos 的唤醒，导致定时任务无法启动。