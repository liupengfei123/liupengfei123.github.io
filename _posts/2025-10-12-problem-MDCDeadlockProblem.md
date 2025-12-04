---
layout:     page
title:      "死锁问题排查"
subtitle:   "升级新系统后，应用出现死锁问题"
date:       2025-07-02
author:     "Lpf"
header-img: "img/home-bg.jpg"
tags:
    - 问题排查
---

# 问题现象

应用卡死，通过jstack导出应用堆栈发现出现死锁情况：background-preinit 与 main 线程卡死。

# 问题分析

通过查看死锁信息：

```
Found one Java-level deadlock:
=============================

"background-preinit":
  waiting to lock Monitor@0x00007f355c1c3560 (Object@0x0000000080ac45b8, a org/springframework/boot/loader/net/protocol/jar/UrlNestedJarFile),
  which is held by "main"
"main":
  waiting to lock Monitor@0x00007f355c3821f0 (Object@0x000000008060ba90, a org/springframework/boot/loader/launch/LaunchedClassLoader),
  which is held by "background-preinit"

```



通过查看两个线程持有锁情况：

![image-20251204202836369]({{site.baseurl}}/img/in-post/problem-MDCDeadlockProblem/image-20251204202836369.png)

![image-20251204202849280]({{site.baseurl}}/img/in-post/problem-MDCDeadlockProblem/image-20251204202849280.png)

对其进行分析：

![image-20251204202909573]({{site.baseurl}}/img/in-post/problem-MDCDeadlockProblem/image-20251204202909573.png)



在对main线程中的探针堆栈进行分析：

![image-20251204202929548]({{site.baseurl}}/img/in-post/problem-MDCDeadlockProblem/image-20251204202929548.png)

在 Spring boot执行类加载的过程中，探针对其目标类进行插码操作，在其插码过程中出现异常情况，探针对其进行日志记录，从而触发了logback中的 calculatePackagingData 逻辑（打印异常堆栈中各个类所在的jar包名称）。

在通过对 ch.qos.logback.classic.spi.ThrowableProxy.calculatePackagingData 其源码进行分析：

可以发现 calculatePackagingData 该方法功能为获取异常堆栈中获取类和方法所在的jar信息，所以会通过 ClassLoader.loadClass 获取对应class对象信息，从而去尝试获取 LaunchedClassLoader 锁对象。

# 详细分析

在探针代码中，如果遇到异常需要打印，就会调用logback中的 log方法，并将 Exception 作为参数传入其中：

![image-20251204202949231]({{site.baseurl}}/img/in-post/problem-MDCDeadlockProblem/image-20251204202949231.png)

在 LoggingEvent 的构造函数中，如果存在异常对象，并且开启了 packagingDataEnabled 就会对其执行 calculatePackagingData 方法：

![image-20251204203001868]({{site.baseurl}}/img/in-post/problem-MDCDeadlockProblem/image-20251204203001868.png)

![image-20251204203009562]({{site.baseurl}}/img/in-post/problem-MDCDeadlockProblem/image-20251204203009562.png)

![image-20251204203018172]({{site.baseurl}}/img/in-post/problem-MDCDeadlockProblem/image-20251204203018172.png)

![image-20251204203028127]({{site.baseurl}}/img/in-post/problem-MDCDeadlockProblem/image-20251204203028127.png)

![image-20251204203035824]({{site.baseurl}}/img/in-post/problem-MDCDeadlockProblem/image-20251204203035824.png)

![image-20251204203044635]({{site.baseurl}}/img/in-post/problem-MDCDeadlockProblem/image-20251204203044635.png)

![image-20251204203051716]({{site.baseurl}}/img/in-post/problem-MDCDeadlockProblem/image-20251204203051716.png)

在logback内部处理，最终会调用到 ClassLoader.loader方法，从而去尝试获取  LaunchedClassLoader 锁对象。

# 总结

所以其导致死锁问题原因：在探针插码过程中需要打印异常情况，导致低版本的logback在使用 calculatePackagingData  需要持有 LaunchedClassLoader 锁对象，进而导致与 Spring boot 3.0 的机制冲突。

# 解决方案

在用 8.4.1版本的探针用的是 logback 1.0.13，这个版本的logback 默认是开启了 packagingDataEnabled，即打印出问会额外打印类的所在包信息（PackagingDataCalculator.calculate），继而会触发类加载器。

探针9.3.0之后的版本，探针使用logback升级到了 1.1.11（JDK8 版本升级到 1.2.13） packagingDataEnabled 默认关闭，所以在打印日志的时候就不会去调用loadClass。 也就可以规避当前的死锁问题。