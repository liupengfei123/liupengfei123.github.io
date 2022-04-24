---
layout:     page
title:      "弱引用局部变量进入老年代问题"
subtitle:   "弱引用局部变量本应在年轻代回收，却进入老年代导致full gc频繁问题"
date:       2022-04-24
author:     "Lpf"
header-img: "img/home-bg.jpg"
tags:
    - 问题排查
---

# 前述

在项目中忽然发现频繁的出现full gc，经过排查发现一个局部变量通过弱引用链接到全局缓存中，在minor gc 中无法被回收，导致进入老年代。

其简单的类结构关系：

![1650806442593]({{site.baseurl}}/img/in-post/problem-WeakObjectGoInTenuredGeneration/1650806442593.png)

# 问题复现

## 测试环境

```
JDK:jdk1.8.0_151
JVM args: -Xmx500m -Xms500m
```

## 运行代码

```java
public class WeakRefTest {
    @Test
    public void test() throws InterruptedException {
        Thread.sleep(5000);
        Map<Object, ValueObject> map = new WeakHashMap<>();
        while (true) {
            mapPut(map);
            map.size();
        }
    }
    private void mapPut(Map<Object, ValueObject> map) {
        Object o = new Object();
        map.put(o, new ValueObject(o));
    }
    private static class ValueObject {
        private Object key;
        private int[] array;
        public ValueObject(Object key) {
            this.key = key;
            this.array = new int[100];
        }
    }
}
```

## 结果

![1650806840379]({{site.baseurl}}/img/in-post/problem-WeakObjectGoInTenuredGeneration/1650806840379.png)

![1650806887952]({{site.baseurl}}/img/in-post/problem-WeakObjectGoInTenuredGeneration/1650806887952.png)

# 正确测试

## 测试环境

```
JDK:jdk1.8.0_151
JVM args: -Xmx500m -Xms500m
```

## 运行代码

```java
public class WeakRefTest {
    @Test
    public void test() throws InterruptedException {
        Thread.sleep(5000);
        Map<Object, ValueObject> map = new WeakHashMap<>();
        while (true) {
            mapPut(map);
            map.size();
        }
    }
    private void mapPut(Map<Object, ValueObject> map) {
        Object o = new Object();
        map.put(o, new ValueObject(o));
    }
    private static class ValueObject {
        private Object key;
        private int[] array;
        public ValueObject(Object key) {
            // this.key = key; 仅取消 value 对 key 的强引用
            this.array = new int[100];
        }
    }
}
```

##  结果

![1650807108712]({{site.baseurl}}/img/in-post/problem-WeakObjectGoInTenuredGeneration/1650807108712.png)

![1650807156889]({{site.baseurl}}/img/in-post/problem-WeakObjectGoInTenuredGeneration/20220424220856.png)

# 结果猜测

通过查看资料知道，无论是年轻代回收还是老年代回收都是通过可达性分析进行判断对象是否需要回收的。

而对弱引用的可达性分析在官方文档中：

> - An object is *weakly reachable* if it is neither strongly nor softly reachable but can be reached by traversing a weak reference. When the weak references to a weakly-reachable object are cleared, the object becomes eligible for finalization.
> - 一个对象是弱可达的，如果它既不是强可达也不是软可达，但可以通过遍历弱引用来达到。 当对弱可达对象的弱引用被清除时，该对象就有资格进行终结。

那么如果是 minor gc 和 major gc 的可达性分析是一样的，那么不管弱可达是否支持上述的情况，都不符合实验的结构。

那么 minor gc 和 major gc 所执行的可达性分析标准应该是不一样的。

## minor gc

在 minor gc中由于需要快速的响应所以应该没有分析类关联结构（即没有分析强引用是否来自Reference对象），只是分析了 key对象是否存在强引用链。

> WeakHashMap(GC ROOT) ---> WeakEntry --->  ValueObejct ---> Object(key)

导致 key 无法被回收，进而再经过多次gc之后进入老年代。

## major/full gc

再Major/full gc中则有判断类关联结构来判断对象是否为弱可达。从而可以回收 key 对象，进而通过 WeakHashMap 解除 Value 和 WeakEntry 的引用关系。






PS:希望有了解的童鞋，可以给俺留一个 issue，帮忙回答一下