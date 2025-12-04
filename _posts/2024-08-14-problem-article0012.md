---
layout:     page
title:      "Resin中启动JS注入后JSP页面空白"
subtitle:   "Resin中启动JS探针注入之后，JSP页面一片空白"
date:       2024-08-14
author:     "Lpf"
header-img: "img/home-bg.jpg"
tags:
    - 问题排查
---

# 概述

发现开启JS注入后页面则会空白，通过排查：

![image-20251204210010748]({{site.baseurl}}/img/in-post/article0012/image-20251204210010748.png)


开启探针注入之后 这个接口返回就会为空，导致页面初始化的流程中断。

同时通过添加额外日志，发现：
```
2024-08-14 14:00:20 [242397 213] bonree WARN: UemIntrospection EnterServlet URI : /login/UpgradeMessage.jsp， reqClass weaver.filter.MonitorXFixIPHttpRequest, resClass com.caucho.server.http.ToCharResponseAdapter[UemIntrospection.java:98]
java.lang.RuntimeException: null
	at com.brj.agent.instrumentation.uem.javax.UemIntrospection.EnterServlet(UemIntrospection.java:98) [na:na]
	at com.brj.agent.instrumentation.servlet24.ServletUtils.EnterServlet(ServletUtils.java:25) [na:na]
	at com.caucho.jsp.JavaPage.service(JavaPage.java) [resin.jar:4.0.58]
	at com.caucho.jsp.Page.pageservice(Page.java:557) [resin.jar:4.0.58]
	at com.caucho.server.dispatch.PageFilterChain.doFilter(PageFilterChain.ja
```
在jsp_servlet中service的 response 对象为 com.caucho.server.http.ToCharResponseAdapter 实例。

# 复现

## 步骤

创建filter过滤器，将 response对象包装成 ToCharResponseAdapter 实例。
```
public class TestFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws ServletException, IOException {
        filterChain.doFilter(servletRequest, ToCharResponseAdapter.create((HttpServletResponse) servletResponse));
    }

    @Override
    public void destroy() {
    }
}
```
```
<filter>
    <filter-name>TestFilter</filter-name>
    <filter-class>com.bonree.testtomat.servlet.filter.TestFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>TestFilter</filter-name>
    <url-pattern>/**.jsp</url-pattern>
</filter-mapping>
```
在将filter配置到 web.xml 配置文件中，并且拦截jsp请求。

# 原理

## 在无探针应用场景中

在 filer 将 resp 包装成 ToCharResponseAdapter，然后传递给JSP_Servlet 处理，在Jsp_servlet中也有判断如果 resp 不是 ToCharResponseAdapter 也包装成 ToCharResponseAdapter 实例。

![image-20251204210039190]({{site.baseurl}}/img/in-post/article0012/image-20251204210039190.png)


这样 JSP中的数据都写入到 ToCharResponseAdapter 的缓存中，待JSP的service结束时，在将  ToCharResponseAdapter 的缓存写入到 响应流中。

![image-20251204210048087]({{site.baseurl}}/img/in-post/article0012/image-20251204210048087.png)

```

	at java.io.PrintWriter.write(PrintWriter.java:426) [na:1.8.0_151]
	at com.caucho.server.http.ToCharResponseAdapter$ToCharResponseStreamWrapper.writeNext(ToCharResponseAdapter.java:175) [resin.jar:4.0.58]
	at com.caucho.server.http.ToCharResponseStream.flushCharBuffer(ToCharResponseStream.java:403) [resin.jar:4.0.58]
	at com.caucho.server.http.ToCharResponseStream.flushBuffer(ToCharResponseStream.java:351) [resin.jar:4.0.58]
	at com.caucho.jsp.JspWriterAdapter.close(JspWriterAdapter.java:292) [resin.jar:4.0.58]
	at com.caucho.jsp.JspWriterAdapter.popWriter(JspWriterAdapter.java:275) [resin.jar:4.0.58]
	at com.caucho.jsp.PageContextImpl.release(PageContextImpl.java:1421) [resin.jar:4.0.58]
	at com.caucho.jsp.PageManager.freePageContext(PageManager.java:215) [resin.jar:4.0.58]
	at _jsp._login._UpgradeMessage__jsp._jspService(_UpgradeMessage__jsp.java:35) [work:na]
	at com.caucho.jsp.JavaPage.service(JavaPage.java:64) [resin.jar:4.0.58]
	at com.caucho.jsp.Page.pageservice(Page.java:557) [resin.jar:4.0.58]
	at com.caucho.server.dispatch.PageFilterChain.doFilter(PageFilterChain.java:194) [resin.jar:4.0.58]
	at wscheck.FileCheckFilter.doFilter(FileCheckFilter.java:423) [classbean:na]
	at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:89) [resin.jar:4.0.58]

```

即正常时  resp --> ToCharResponseAdapter， 在 _jspService 结束的时候 将ToCharResponseAdapter缓存数据写入到 响应流中

## 开启JS注入应用场景中

在开启JS注入之后，因为其应用是在 Filter 就将 resp 包装成了 ToCharResponseAdapter，而JS注入是在 servlet 中将 resp 包装成 BrResponse，而在JSP中又会判断resp是否为 ToCharResponseAdapter 实例，否则将 resp 包装成 ToCharResponseAdapter 实例。

所以对象包装链就变成了： resp --> ToCharResponseAdapter1  --> br_resp  -->  ToCharResponseAdapter2 。 

ToCharResponseAdapter1 是 filter中包装的， ToCharResponseAdapter2 是 _jspService 中包装的。

_jspService 结束之后执行freePageContext方法，触发ToCharResponseAdapter2的flush，将数据写到 br_resp 中，br_resp 再将数据转发到 ToCharResponseAdapter1 中。但是 ToCharResponseAdapter1 就没有人再去触发他的flush了，就导致数据都没有写入到响应流中。

# 解决方式

为了防止以后再遇到这种汉堡式包装链，应该把JS注入的位置放置在第一个入口：如果有使用Filter就放置在第一个Filter中，如果没有则放置在Servlet中。

同时因为Filter会存在多个，所以需要处理放置多次注入，所以通过使用 request 中的 getAttribute 和 setAttribute 方法来防止多次包装问题：在包装前判断是否 request.getAttribute("xxx") 存在值，如果不存在则该对应的response对象没有被包装过则将其包装并调用 request.setAttribute("xxx","xxx")设置值，如果有存在则说明对应的response对象已经被包装过了。