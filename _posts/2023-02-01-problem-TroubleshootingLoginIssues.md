---
layout:     page
title:      "无法登录问题排查"
subtitle:   "在客户环境中，很奇怪的无法登录问题排查"
date:       2023-02-01
author:     "Lpf"
header-img: "img/home-bg.jpg"
tags:
    - 问题排查
---

# 问题描述

根据反馈：在开启探针采集之后，会导致系统登录失败。

通过查看登录错误信息发现，在当前的session查找不到登录的信息导致报错。

![image-20230215155946023]({{site.baseurl}}/img/in-post/problem-TroubleshootingLoginIssues/image-20230215155946023.png)

![image-20230215155933679]({{site.baseurl}}/img/in-post/problem-TroubleshootingLoginIssues/image-20230215155933679.png)

# spring-security登录原理

通过查看 Spring-security 源码进行分析：

由下述代码可以找到一个登录操作简单的流程：

1. 将 用户名称 设置到 session 中

   	2. 登录成功：将session 注册到 security 框架中
   	3. `invalidateSessionOnSuccessfulAuthentication`属性为 true时：销毁 当前session，获取新session，并且旧session中的属性备份到新session，之后将旧session在security框架中解除，将新session注册到security框架中。
   	4. response 重定向 到`login.do`（并且在响应头中携带新的`sessionID`），浏览器更新新的`sessionID`
   	5. 浏览器再次访问`login.do`，应用代码通过 请求中cookie携带新的`sessionID`获取到对应的session，在通过session获取到对应属性，从而完成登录。

在 [AuthenticationProcessingFilter](https://github.com/spring-projects/spring-security/blob/2.0.5.RELEASE/core/src/main/java/org/springframework/security/ui/webapp/AuthenticationProcessingFilter.java)类中：

```java
public Authentication attemptAuthentication(HttpServletRequest request) throws AuthenticationException {
    String username = obtainUsername(request).trim();
    String password = obtainPassword(request);
    UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(username, password);
    // 1、将 用户名称 设置到 session 中
    HttpSession session = request.getSession(false);
    if (session != null || getAllowSessionCreation()) {
        request.getSession().setAttribute(SPRING_SECURITY_LAST_USERNAME_KEY, TextUtils.escapeEntities(username));
    }
    return this.getAuthenticationManager().authenticate(authRequest);
}
```

在 [AbstractProcessingFilter](https://github.com/spring-projects/spring-security/blob/4361211c218a0394fd7b7b74078d999fc8aafcc8/core/src/main/java/org/springframework/security/ui/AbstractProcessingFilter.java#L303)中：

```java
// 2、登录成功：将session 注册到 security 框架中
protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response,
                                        Authentication authResult) {
    if (logger.isDebugEnabled()) logger.debug("Authentication success: " + authResult.toString());
    // 其他操作。。
    if (invalidateSessionOnSuccessfulAuthentication) {
        // 3、invalidateSessionOnSuccessfulAuthentication属性为 true时：销毁 当前session，获取新session，并且旧session中的属性备份到新session，之后将旧session在security框架中解除，将新session注册到security框架中。
        SessionUtils.startNewSessionIfRequired(request, migrateInvalidatedSessionAttributes, sessionRegistry); 
    }
    String targetUrl = determineTargetUrl(request);  // 获取重定向url 即login.do
	// 其他操作。。
    sendRedirect(request, response, targetUrl);     // 4、response 重定向 到`login.do`（并且在响应头中携带新的`sessionID`），浏览器更新新的`sessionID`
}
```

在 [SessionUtils](https://github.com/spring-projects/spring-security/blob/4361211c218a0394fd7b7b74078d999fc8aafcc8/core/src/main/java/org/springframework/security/util/SessionUtils.java#L27)中

```java
public static void startNewSessionIfRequired(HttpServletRequest request, boolean migrateAttributes, 
                                             SessionRegistry sessionRegistry) {
    HttpSession session = request.getSession(false);
    if (session == null)  return;
	// 备份 旧session 中设置的属性
    session.invalidate();  // 销毁当前 session
    session = request.getSession(true); // 获取 新的session
  	// 将 旧session中设置的属性，设置到 新session 中
    // 移除 旧sessionID 在 security 中的注册信息， 在 security 中注册 新sessionID
}
```

# weblogic更新session操作

`weblogic`在更新了session之后，在什么时机去设置响应头来更新cookie的`sessionID`呢。

在获取通过 [getNewSession](weblogic.servlet.internal.ServletRequestImpl.SessionHelper#getNewSession)方法获取到新的session时：

```java
private void getNewSession(String var1) {
    if (HTTPDebugLogger.isEnabled()) this.request.trace("Creating new session");
    // 创建出新的session
    this.session = this.request.getContext().getSessionContext().getNewSession(var1, this.request, this.request.getResponse());
    this.sessionID = ((SessionInternal)this.session).getInternalId();
    if (HTTPDebugLogger.isEnabled()) {
        this.request.trace("New Session: " + this.sessionID);
    }
	// 将session设置到 response cookie中
    this.request.getResponse().setSessionCookie(this.session);
}
```

```java
final void setSessionCookie(HttpSession var1) {
    SessionContext var2 = this.context.getSessionContext();
    //删除 旧sessionId cookie (即 flexjsessionid)
    this.removeCookie(var2.getConfigMgr().getCookieName(), var2.getConfigMgr().getCookiePath());
    // 设置新 cookie
    Cookie var3 = new Cookie(var2.getConfigMgr().getCookieName(), ((SessionInternal)var1).getIdWithServerInfo());
    if (!this.requestHadCookie(var3)) {
        if (HTTPDebugLogger.isEnabled()) {
            this.trace("Setting cookie for " + this.request.toString());
        }
        this.addCookie(var3);	// 将 flexjsessionid cookie 设置到 response 中
    }
}
```

在response响应时，会执行 writeHeaders 方法来设置响应头。

```java
final void writeHeaders() throws IOException {
    if (HTTPDebugLogger.isEnabled()) {
        this.trace("Writing headers for " + this.request.toString());
    }
    String var7;
    if (this.cookies.size() > 0) {
        for(int var4 = 0; var4 < this.cookies.size(); ++var4) {
            Cookie var5 = (Cookie)this.cookies.get(var4);
            var7 = CookieParser.formatCookie(var5, var6);
            if (HTTPDebugLogger.isEnabled()) {
                this.trace("Wrote cookie: " + var7);
            }
            this.headers.addHeader("Set-Cookie", var7);
        }
    }
}
```

# 问题排查分析

通过上面的描述对应用的登录状态是已经由一些了解了，那么在结合登录正常的日志和登录失败的日志对比一下，可以找到问题的所在。

## 两个请求的问题

![image-20230215175147805]({{site.baseurl}}/img/in-post/problem-TroubleshootingLoginIssues/image-20230215175147805.png)

（PS：“yingda WebAppServletContext execute method” 在 weblogic.servlet.internal.WebAppServletContext#execute，开头和结尾处打印，该方法为处理请求，request 后跟的数字为 request对象的哈希值）

![image-20230215175249203]({{site.baseurl}}/img/in-post/problem-TroubleshootingLoginIssues/image-20230215175249203.png)

通过观察 红色框 和 绿色框的发现，/flex/j_spring_security_check 接口接收到了两个请求生成了两个不同的 request 和 response 对象，在间隔（1673859799835 - 1673859799828）7ms 内，都是由 10.1.33.135 主机发送的。

并且通过查看浏览器开发者工具发现浏览器端只有一个请求发出，和查看错误日志发现第一个请求总时会有 Connection reset 的异常。

![image-20230215175550908]({{site.baseurl}}/img/in-post/problem-TroubleshootingLoginIssues/image-20230215175550908.png)

![image-20230215175620942]({{site.baseurl}}/img/in-post/problem-TroubleshootingLoginIssues/image-20230215175620942.png)

所以有理由怀疑，该请求的socket因为未知原因生成了两个 request 和 response。

而无法登录问题就是与该现象有关，先解释一下无法登录问题会与该现象有关，然后该现象的出现的原因在后面讨论。

## 登录正常日志

![image-20230215173019374]({{site.baseurl}}/img/in-post/problem-TroubleshootingLoginIssues/image-20230215173019374.png)

对比应用日志事件是登录成功的

![image-20230215173326295]({{site.baseurl}}/img/in-post/problem-TroubleshootingLoginIssues/image-20230215173326295.png)

通过查看日志发现：两个request都创建了新的session对象，结合上面的代码（`SessionUtils#startNewSessionIfRequired`），第一个request 和 第二个request 在获取（`getSession(false)`）session的时候，session对象并没有被销毁，所以两个销毁了旧的session(`session.invalidate()`) 并且创建了新的session对象（`getSession(true)`）。

## 登录失败日志

![image-20230215181532085]({{site.baseurl}}/img/in-post/problem-TroubleshootingLoginIssues/image-20230215181532085.png)

通过查看日志发现：只有第一个请求才会生成了 新的Session，，结合上面的代码（`SessionUtils#startNewSessionIfRequired`）， 第二个request 在获取（`getSession(false)`）session的时候，session对象已经被销毁了（第一个request销毁的时机 在 第二个request获取session 之前），所以只有第一个request销毁了旧的session(`session.invalidate()`) 并且创建了新的session对象（`getSession(true)`），第二个request没有创建新的session对象，导致也没有更新cookie中的sessionID，进而导致 访问 login.do 接口携带的 sessionID 还是旧的session，但是旧的session 已经被销毁了，所以登录失败。

## 结论

前提：在上述同一个socket生成了两个request 和 response，因为同占用同一个socket，所以在第二个response生成出来时，第一个 response 已经被重置了，所以在第一个response重定向的时候，会抛出 Connection reset 的异常，这也就意味着第一个 response的响应数据 不可能到达浏览器，也就是说只有第二个 response 才能向浏览器响应数据。

当探针启用的状态下，探针采集时间消耗的问题，导致了 第一个 request 比 第二个 request 早到了一小段时间（当第一个request 已经执行了**session.invalidate()** 方法，第二个request才执行到 `SessionUtils#startNewSessionIfRequired` 方法 ）这样就会到导致 第二个`request.getSession(false) `为null，进而无法执行后续操作，进而在响应头中没有更新 `sessionID`。

当探针禁用状态下，探针没有采集数据库访问等数据，出现了 两个 request 对象同时到达了 `SessionUtils#startNewSessionIfRequired` 方法（当第一个request 执行了**session.invalidate()** 方法之前，第二个request已经执行到`SessionUtils#startNewSessionIfRequired` 方法，并且 已经从 `request.getSession(false)` 方法中获取到session ）这样第二个request就可以执行后续操作，从而更新响应头中的`sessionID`。

通过临时的解决方案的成功也可以佐证该结论。

# 临时解决方案

既然是因为第一个request执行 **session.invalidate()** 和第二个request执行 **request.getSession(false) ** 的出现了时间差。

那么通过在执行 **session.invalidate()** 方法之前 sleep 40ms，就可以实现 第一个request执行 **session.invalidate()** 的时间 在 第二个request 执行 **request.getSession(false) ** 之后。

# 留存问题说明

现在还剩下一个问题：在访问 /flex/j_spring_security_check 接口（其他接口不会出现这种状况）的时候 同一个socket生成了两个request 和 response 这种现象，还不知道是什么原因导致的。

结合在没有安装探针的时候也偶尔会出现无法登录的问题，并且该现象应该属于底层问题所以该现象应该与探针无关。

并且要验证是否与探针有关需要卸载探针进行对照，那么在探针上的排查手段就都将失效，所以这个需要麻烦客户帮忙验证：开启debug日志后，在安装探针的情况下访问接口出现了该现象日志，卸载探针访问接口该现象日志是否消失。
