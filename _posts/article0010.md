---
layout:     page
title:      "插件并发插码时空指针异常"
subtitle:   "特定ClassLoader场景下，插件并发插码时空指针异常"
date:       2024-08-23
author:     "Lpf"
header-img: "img/home-bg.jpg"
tags:
    - 问题排查
---


# 概述

在应用启动时在探针日志中有概率会触发以下报错，从报错信息中可以看出是在插码org/jboss/invocation/ChainedInterceptor 类时，ASM框架报的空指针。

![image-20251204204418995]({{site.baseurl}}/img/in-post/article0010/image-20251204204418995.png)

其调用栈中 PackageWeaveResult#getCompositeBytes 调用 WeaveUtils#convertToClassBytes 方法，表明已经完成weave的ClassNode的构建，要将ClassNode对象转换成 二进制字节码。

#  线索查找

## 正常逻辑

通关debug分析 WeaveUtils#convertToClassBytes 方法，在processInvocation方法的MethodNode中可以发现，其tryCatchBlocks 中的 labelNode 对象 存在于 其 instructions 指令集合中 。

![image-20251204204431304]({{site.baseurl}}/img/in-post/article0010/image-20251204204431304.png)

![image-20251204204442991]({{site.baseurl}}/img/in-post/article0010/image-20251204204442991.png)

在 MethodWriter#computeAllFrames 方法中，其 Handler 就是 visitTryCatchBlock 构建的对象。

![image-20251204204453419]({{site.baseurl}}/img/in-post/article0010/image-20251204204453419.png)

![image-20251204204504054]({{site.baseurl}}/img/in-post/article0010/image-20251204204504054.png)

## 有问题逻辑

通过复现了现象，进行debug发现，其weave之后的MethodNode 对象中 tryCatchBlocks 的 labelNode 对象不存在于其 instructions指令集合中。

![image-20251204204524068]({{site.baseurl}}/img/in-post/article0010/image-20251204204524068.png)
![image-20251204204537247]({{site.baseurl}}/img/in-post/article0010/image-20251204204537247.png)

## 排查原始数据MethodNode

通过查看原始类的字节码转换过来的MethodNode中的，发现其 tryCatchBlocks 的 labelNode 对象存在于其 instructions指令集合中的。

![image-20251204204551478]({{site.baseurl}}/img/in-post/article0010/image-20251204204551478.png)

![image-20251204204559716]({{site.baseurl}}/img/in-post/article0010/image-20251204204559716.png)

并且 weave之后的 MethodNode 中的 tryCatchBlocks 的 labelNode 和 原始MethodNode 的labelNode 对象有关联关系。

## 添加额外日志分析

在 weave 过程中( 将插码类代码和原始代码合并)，其instructions指令集合被替换了。

```

// 在线程 169 中 result_MethodNode 中设置 tryCatch 指令，保存在tryCatchBlocks 中的labelNode 哈希值为 1445101728 和 228293463
2024-08-30 14:04:03 InlinedTryCatchBlockSorter visitTryCatchBlock threadID:169, data: MethodNode  549741571  instructions : 1692397686
start : 994441002 end : 1277311067  tryCatchBlocks : startLabel 1445101728 endLabel 228293463

//  在线程 169 中 result_MethodNode 中设置 label 指令,保存在instructions指令集合中，哈希值为 103517509
2024-08-30 14:04:03 InlinedTryCatchBlockSorter visitLabel threadID:169, data: MethodNode  549741571  instructions : 1692397686
label : 435364201 labelNode : 103517509

org.objectweb.asm.tree.LabelNode  793303794
org.objectweb.asm.tree.LabelNode  1131532990
org.objectweb.asm.tree.MethodInsnNode  reTxName  757761968
org.objectweb.asm.tree.LabelNode  723799005
org.objectweb.asm.tree.LabelNode  731065393
org.objectweb.asm.tree.LabelNode  1697121717
org.objectweb.asm.tree.LabelNode  1445101728
org.objectweb.asm.tree.MethodInsnNode  proceed  338822121
org.objectweb.asm.tree.LabelNode  228293463
org.objectweb.asm.tree.LabelNode  103517509

// 在线程 170 中 result_MethodNode 中设置 tryCatch 指令，保存在tryCatchBlocks 中的labelNode 哈希值为 100410656 和 1674267898
2024-08-30 14:04:03 InlinedTryCatchBlockSorter visitTryCatchBlock threadID:170, data: MethodNode  2018035932  instructions : 1174088077
start : 371412771 end : 1024597982  tryCatchBlocks : startLabel 100410656 endLabel 1674267898



//  在线程 169 中 result_MethodNode 中设置 label 指令,保存在instructions指令集合中，哈希值为 1354506004
// 但是这是打印出来的 instructions指令集合 已经被改变了
2024-08-30 14:04:03 InlinedTryCatchBlockSorter visitLabel threadID:169, data:MethodNode  549741571  instructions : 1692397686
label : 1145975532 labelNode : 1354506004

org.objectweb.asm.tree.LabelNode  793303794
org.objectweb.asm.tree.LabelNode  1131532990
org.objectweb.asm.tree.MethodInsnNode  reTxName  1258527553
org.objectweb.asm.tree.LabelNode  723799005
org.objectweb.asm.tree.LabelNode  731065393
org.objectweb.asm.tree.LabelNode  980035234


//  在线程 170 中 result_MethodNode 中设置 label 指令,保存在instructions指令集合中，哈希值为 100410656
2024-08-30 14:04:03 InlinedTryCatchBlockSorter visitLabel threadID:170, data: MethodNode  2018035932  instructions : 1174088077
label : 371412771 labelNode : 100410656 

org.objectweb.asm.tree.LabelNode  793303794
org.objectweb.asm.tree.LabelNode  1131532990
org.objectweb.asm.tree.MethodInsnNode  reTxName  1258527553
org.objectweb.asm.tree.LabelNode  723799005
org.objectweb.asm.tree.LabelNode  731065393
org.objectweb.asm.tree.LabelNode  980035234
org.objectweb.asm.tree.LabelNode  100410656

// 在线程 169 中 result_MethodNode 插码完毕，其中 instructions指令集合中 都已经变成了 170 线程中 MethodNode 对象中的instructions指令集合。 但是 各自的instructions对象是未改变了，只是内容更改了。
2024-08-30 14:04:03 InlinedTryCatchBlockSorter visitEnd threadID:169, data: MethodNode  549741571  instructions : 1692397686

org.objectweb.asm.tree.LabelNode  793303794
org.objectweb.asm.tree.LabelNode  1131532990
org.objectweb.asm.tree.MethodInsnNode  reTxName  1258527553
org.objectweb.asm.tree.LabelNode  723799005
org.objectweb.asm.tree.LabelNode  731065393
org.objectweb.asm.tree.LabelNode  980035234
org.objectweb.asm.tree.LabelNode  100410656
org.objectweb.asm.tree.MethodInsnNode  proceed  1982568253
org.objectweb.asm.tree.LabelNode  1674267898
org.objectweb.asm.tree.LabelNode  644725627

 tryCatchBlocks :  start: 1445101728   end: 228293463 
		   start: 103517509   end: 1354506004


2024-08-30 14:04:03 threadID:169, Class org/jboss/invocation/ChainedInterceptor cant find classResource, weaved fail.
java.lang.NullPointerException: null
	at org.objectweb.asm.MethodWriter.computeAllFrames(MethodWriter.java:1574) 
	at org.objectweb.asm.MethodWriter.visitMaxs(MethodWriter.java:1548) 
	at org.objectweb.asm.tree.MethodNode.accept(MethodNode.java:767)
	at org.objectweb.asm.tree.MethodNode.accept(MethodNode.java:647)
	at org.objectweb.asm.tree.ClassNode.accept(ClassNode.java:468)
	at com.bonree.brjapm.weave.utils.WeaveUtils.convertToClassBytes(WeaveUtils.java:218)
```

同时发现，在 com.bonree.brjapm.weave.MethodProcessors#inlineMethods 方法中 插件MethodNode 在并发情况下存在相同的情况，如果哈希值为 1462591418 的MethodNode对象。

```
-- MethodProcessors inlineMethods subjectMethod in current file --
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:165, MethodNode:1689300374
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:172, MethodNode:1462591418
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:165, MethodNode:1462591418
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:179, MethodNode:1462591418
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:169, MethodNode:1462591418
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:170, MethodNode:1462591418
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:171, MethodNode:1462591418
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:178, MethodNode:2017744182
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:177, MethodNode:2114223225
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:178, MethodNode:1424346482
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:177, MethodNode:1940819611
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:176, MethodNode:81791278  
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:176, MethodNode:1528287433
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:175, MethodNode:1618129816
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:175, MethodNode:1484684688
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:173, MethodNode:1335011797
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:173, MethodNode:1616567413
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:174, MethodNode:1446080522
14:04:03 MethodProcessors inlineMethods subjectMethod threadID:174, MethodNode:149110452 
19 occurrences have been found in 1 files.

--  Class org/jboss/invocation/ChainedInterceptor cant find classResource, weaved fail.--
14:04:03 threadID:169, Class ChainedInterceptor cant find classResource, weaved fail.
14:04:03 threadID:170, Class ChainedInterceptor cant find classResource, weaved fail.
14:04:03 threadID:179, Class ChainedInterceptor cant find classResource, weaved fail.
14:04:03 threadID:171, Class ChainedInterceptor cant find classResource, weaved fail.
14:04:03 threadID:172, Class ChainedInterceptor cant find classResource, weaved fail.
14:04:03 threadID:165, Class ChainedInterceptor cant find classResource, weaved fail.
6 occurrences have been found in 1 files.
```

并且对比发现，错误日志中只有 MethodNode 相同的几个线程会出现报错。

# 问题分析

## labelNode的拷贝原理

在 LabelNode 类中：

```java
public Label getLabel() {
    if (value == null) {
        value = new Label();
    }
    return value;
}

public void accept(final MethodVisitor methodVisitor) {
    methodVisitor.visitLabel(getLabel());
}
```

在 MethodNode 类中：

```java
@Override
public void visitLabel(final Label label) {
    instructions.add(getLabelNode(label));
}

protected LabelNode getLabelNode(final Label label) {
    if (!(label.info instanceof LabelNode)) {
        label.info = new LabelNode();
    }
    return (LabelNode) label.info;
}
```

在 MethodNode 对象  instructions指令集合中存储的是 LabelNode 对象，在 accept 过程中会获取 LabelNode 中的value属性（新生成的 Label对象），在visitLabel方法中，有通关 label对象生成新的 LabelNode 对象存储在新的 MethodNode  instructions 命令集合中。

## MethodNode 的 accept方法线程不安全

在单线程，在执行MethodNode 的 accept 方法将 instructions指令集合 拷贝到另外一个 MethodNode 对象时：

![image-20251204204619179]({{site.baseurl}}/img/in-post/article0010/image-20251204204619179.png)

其中 LabelNode 指令会产生上述的引用关系。

但是当存储在并发的情况时：

![image-20251204204628362]({{site.baseurl}}/img/in-post/article0010/image-20251204204628362.png)

就可能出现在第一个线程将 subjectMethod 指令拷贝到 result1 差不多时，第二个线程开始遍历 subjectMethod 指令集合，因为labelNode是同一个对象存储到result2指令集合中的 LabelNode 对象和 在 result1 指令集合中的 LabelNode对象是同一个。

所以当在result2在插入下一个OtherNode指令时，LabelNode 的引用就会更改为result2中的 OtherNode 引用，这就导致了 result1中的指令集合直接变成了 result2中的内容。

PS：因为MethodNode  instructions 命令集合为链表结构。

## computeAllFrames 方法NPE问题原因

因为 MethodNode 线程不安全的原因，在并发情况下就会出现，MethodNode中 tryCatchBlocks 集合中的中labelNode对象在其 instructions 指令集合中遍历不到的现象存在。

当 MethodNode 对象被 MethodWriter 对象访问时，在先将 tryCatchBlocks 集合整理成了 firstHandler 结构，再遍历指令结合时 visitLabel 方法中为 label对象设置了相关属性，该label对象与firstHandler 中的label对象为同一个对象。

但是因为并发的问题， tryCatchBlocks 集合中的中labelNode对象在其 instructions 指令集合中遍历不到，就会导致firstHandler 中的label属性都是空的，就导致了在 computeAllFrames 方法NPE问题。