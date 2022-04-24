---
layout:     page
title:      "ClassCircularityError异常问题排查"
subtitle:   "一次奇怪的ClassCircularityError异常问题排查"
date:       2022-03-28
author:     "Lpf"
header-img: "img/home-bg.jpg"
tags:
    - 问题排查
---

# 前述

在客户的机器上出现了异常报错导致了应用启动不起来。其中报错是 *java.lang.ClassCircularityError: java/security/Permission*，都是在自定义的SecurityManager方法中抛出的。

```
Exception in thread "Thread-22" java.lang.ClassCircularityError: java/security/Permission
	at flex.cc.interfaceservlet.SieralCtlServlet$1.check(SieralCtlServlet.java:59)
	at flex.cc.interfaceservlet.SieralCtlServlet$1.checkPermission(SieralCtlServlet.java:93)
	at java.lang.SecurityManager.checkRead(SecurityManager.java:871)
	at java.io.File.exists(File.java:731)
	at org.apache.log4j.helpers.FileWatchdog.checkAndConfigure(FileWatchdog.java:77)
	at org.apache.log4j.helpers.FileWatchdog.run(FileWatchdog.java:108)


Exception in thread "[ACTIVE] ExecuteThread: '0' for queue: 'weblogic.kernel.Default (self-tuning)'" java.lang.ClassCircularityError: java/security/Permission
	at flex.cc.interfaceservlet.SieralCtlServlet$1.check(SieralCtlServlet.java:59)
	at flex.cc.interfaceservlet.SieralCtlServlet$1.checkPermission(SieralCtlServlet.java:93)
	at java.lang.SecurityManager.checkRead(SecurityManager.java:871)
	at java.io.File.list(File.java:971)
	at java.io.File.listFiles(File.java:1129)
	at weblogic.i18ntools.DefaultL10NClassLoaderFactory.createLoader(DefaultL10NClassLoaderFactory.java:312)
	at weblogic.i18ntools.DefaultL10NClassLoaderFactory.access$000(DefaultL10NClassLoaderFactory.java:30)
	at weblogic.i18ntools.DefaultL10NClassLoaderFactory$CreateLoaderAction.run(DefaultL10NClassLoaderFactory.java:298)
	at java.security.AccessController.doPrivileged(Native Method)
	at weblogic.i18ntools.DefaultL10NClassLoaderFactory.getClassLoader(DefaultL10NClassLoaderFactory.java:246)
	at weblogic.i18ntools.L10nLookup.getL10NCustomClassLoader(L10nLookup.java:456)
	at weblogic.i18ntools.L10nLookup.getLocalizerBundle_inner(L10nLookup.java:513)
	at weblogic.i18ntools.L10nLookup.getLocalizerBundle(L10nLookup.java:462)
	at weblogic.i18ntools.L10nLookup.getLocalizer(L10nLookup.java:419)
	at weblogic.i18n.logging.CatalogMessage.<init>(CatalogMessage.java:48)
	at weblogic.kernel.KernelLogger.logExecuteFailed(KernelLogger.java:100)
	at weblogic.work.ExecuteThread.execute(ExecuteThread.java:274)
	at weblogic.work.ExecuteThread.run(ExecuteThread.java:221)
```

```java
public class SieralCtlServlet extends HttpServlet {
  public void init() throws ServletException {
      SecurityManager originalSecurityManager = System.getSecurityManager();
      if (originalSecurityManager == null) {
          // 创建自己的SecurityManager
          SecurityManager sm = new SecurityManager() {
              private void check(Permission perm) {
            	String aaa =   perm.toString();  // 59 行
			   一些逻辑判断无关.....
            }
            public void checkPermission(Permission perm) {
                check(perm); // 93 行
            }
            public void checkPermission(Permission perm, Object context) {
                check(perm);
            }
        };
        System.setSecurityManager(sm);
      }
  }
}
```

# ClassCircularityError 说明

JVM 规范指定 `ClassCircularityError` 的抛出条件是：

> 类或接口由于是自己的超类或超接口而不能被装入。

这个错误是在链接阶段的解析过程中抛出的。这个错误有点奇怪，因为 Java 编译器不允许发生这种循环情况。但是，如果独立地编译类，然后再把它们放在一起，就可能发生这个错误。请设想以下场景。首先，编译清单 7 和清单 8 中的类：

**清单 7. A.java**

```java
public class A extends B {}
```

**清单 8. B.java**

```java
public class B {}
```

然后，分别编译清单 9 和清单 10 中的类:

**清单 9. A.java**

```java
 public class A { }
```

**清单 10. B.java**

```java
 public class B extends A { }
```

最后，采用清单 7 的类 `A` 和清单 10 的类 `B`，并运行一个应用程序，试图装入 `A` 或者 `B`。这个情况看起来可能不太可能，但是在复杂的系统中，在把不同部分放在一起的时候，可能会发生类似的情况。

# 问题分析

可以从上述的 ClassCircularityError 异常说明可以看出，该异常是由于类的循环依赖，如 A 继承于 B，B 又继承于 A。

但是应用日志中的 Permission 类为 java 的核心类应该不会存在循环应用问题。

所以应用中的报错异常可能并不是由于类的循环依赖导致的。

在晚上查找了一番，发现有一个 tomcat 的bug与该异常非常相似 [58199](https://bz.apache.org/bugzilla/show_bug.cgi?id=58199) 和 [58125](https://bz.apache.org/bugzilla/show_bug.cgi?id=58125) 这两个是相同的问题。

通过在 tomcat 8.0.20 上，执行测试用例 ，确实可以得到相同的异常日志。

```java
public class MyStartupListener implements ServletContextListener {
    public void contextDestroyed(ServletContextEvent evt) {
    }

    public void contextInitialized(ServletContextEvent evt) {
        // Class<Permission> permissionClass = Permission.class;
        System.setSecurityManager(new MySecurityManager());
    }
}

public class MySecurityManager extends SecurityManager {
    public MySecurityManager() {
        super();
    }

    @Override
    public void checkPermission(java.security.Permission perm) {
        if (perm == null) {
            throw new NullPointerException("permission can't be null");
        }
        String propertyName = perm.getName();
    }

    @Override
    public void checkPermission(java.security.Permission perm, Object context) {
        checkPermission(perm);
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
       version="2.5">
  <welcome-file-list>
    <welcome-file>index.html</welcome-file>
  </welcome-file-list>

  <listener>
    <listener-class>com.mytest.MyStartupListener</listener-class>
  </listener>
</web-app>
```

# 原因分析

## debug分析

在 MySecurityManager 中打个断点。

![image]({{site.baseurl}}/img/in-post/problem-ClassCircularityError/1649859351073.png)

在 WebappClassLoaderBase 中也打算条件断点。

![image]({{site.baseurl}}/img/in-post/problem-ClassCircularityError/1649859363654.png)


启动tomcat，在第一次进去 MySecurityManager 时是一个加载 httpmethodConstraintElement 类需要进行安全检查。而后遇到 Permission 类（通过 -verbose，可以查看到该类时已经被加载过来，但是tomcat还是选择使用tomcat的类加载器尝试一边，猜测是因为 该类加载的时候tomcat类加载器还没有创建，就没有缓存。jvm底层缓存类加载情况的不太清楚），tomcat没有加载过，所以通过类加载器进行加载。

![image]({{site.baseurl}}/img/in-post/problem-ClassCircularityError/1649859392031.png)

![image]({{site.baseurl}}/img/in-post/problem-ClassCircularityError/1649859404046.png)

在tomcat类加载器中先使用 bootstrap classload 尝试加载 Permission 类。

![image]({{site.baseurl}}/img/in-post/problem-ClassCircularityError/1649859416560.png)

![image]({{site.baseurl}}/img/in-post/problem-ClassCircularityError/1649859426130.png)

并 Permission 类处于 jar包中，而后需要对 rt.jar 这个文件进行判断是否有权限，

![image]({{site.baseurl}}/img/in-post/problem-ClassCircularityError/1649859439358.png)

则又进入 MySecurityManager 中，但是这时 Permission  并没有加载完成，所以还是会触发 Permission 的类加载情况，则导致了循环加载，进而抛出 ClassCircularityError 错误。

![image]({{site.baseurl}}/img/in-post/problem-ClassCircularityError/1649859449441.png)

![image]({{site.baseurl}}/img/in-post/problem-ClassCircularityError/1649859456697.png)


## 验证

通过上述描述，其中一个原因是 Permission 没有被加载导致的，所以如果在设置 SecurityManager 之前触发一下 Permission  的加载，则可以不会报错。

```java
public class MyStartupListener implements ServletContextListener {
    public void contextDestroyed(ServletContextEvent evt) {
    }

    public void contextInitialized(ServletContextEvent evt) {
        Class<Permission> permissionClass = Permission.class;
        System.setSecurityManager(new MySecurityManager());
    }
}
```

将代码修改为上述时，启动 tomcat，则可以正常启动。

## 奇怪点

将 Permission.class 改为其子类  FilePermission.class 时，并无作用还是会报错。通过设置 -verbose 在很早之前 FilePermission  就有加载记录。

```java
public class MyStartupListener implements ServletContextListener {
    public void contextDestroyed(ServletContextEvent evt) {
    }

    public void contextInitialized(ServletContextEvent evt) {
        Class<FilePermission> filePermissionClass = FilePermission.class;
        System.setSecurityManager(new MySecurityManager());
    }
}
```

但是 将 Permission.class 改为其自定义子类  MyPermission.class 时，则可以正常启动。

```
public class MyStartupListener implements ServletContextListener {
    public void contextDestroyed(ServletContextEvent evt) {
    }

    public void contextInitialized(ServletContextEvent evt) {
        Class<MyPermission> filePermissionClass = MyPermission.class;
        System.setSecurityManager(new MySecurityManager());
    }
}

public class MyPermission extends Permission {
}
```

推测 由于 FilePermission 属于 rt.jar 中，所以 tomcat 类加载器将该类委托给 bootstrap classloader 加载，所以其父类也是被其 boostrap classloader 加载。 但是 MyPermission 是自定义的类，所以是有 tomcat 类加载器加载的，所以其父类也会由 tomcat 类加载器加载 而后委托给 boostrap classloader 加载，因为由进入过 tomcat 类加载器所以就有来加载记录，下次遇到 Permission 类是就不会触发类加载。

PS:希望有了解的童鞋，可以给俺留一个 issue，帮忙回答一下