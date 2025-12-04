---
layout:     page
title:      "客户应用IllegalStateException错误问题"
subtitle:   "客户应用IllegalStateException: unexpected message type: DefaultHttpResponse, state: 3 问题"
date:       2024-07-02
author:     "Lpf"
header-img: "img/home-bg.jpg"
tags:
    - 问题排查
---

# 描述问题

## 环境

- netty 4.1.31
- spring 5.1.3
- reactor-core 3.3.22
- reactor-netty-0.8.3

## 问题现象

- 27号晚上安装探针
- 28号有出现 java.lang.IllegalStateException: unexpected message type: DefaultHttpResponse, state: 3 异常 （以下称为 500错误）
- 29号出现 堆外内存溢出（以下称为OOM）导致应用宕机，并且重启应用
- 30号 只在15点时出现两个500错误，在17点时卸载探针并重启应用，之后就没有再出现500错误和OOM

## 错误堆栈

```
2023-06-29 09:25:55 [05653cfd] Error [io.netty.handler.codec.EncoderException: java.lang.IllegalStateException: unexpected message type: DefaultHttpResponse, state: 3] for HTTP POST "/api/messagepush/batchtransimission", but ServerHttpResponse already committed (500 INTERNAL_SERVER_ERROR)

2023-06-29 09:25:55 [id: 0x05653cfd, L:/192.168.0.53:8081 - R:/188.192.166.127:60881] Error finishing response. Closing connection
io.netty.handler.codec.EncoderException: java.lang.IllegalStateException: unexpected message type: DefaultHttpResponse, state: 3
	at io.netty.handler.codec.MessageToMessageEncoder.write(:106)
....
	at reactor.netty.http.server.HttpTrafficHandler.write(:265)
....
	at io.netty.channel.AbstractChannel.writeAndFlush(:300)
	at reactor.netty.http.HttpOperations.lambda$then$0(:113)
.....
	at org.springframework.http.server.reactive.ChannelSendOperator$WriteBarrier.onNext(:182)
	at reactor.core.publisher.Operators$ScalarSubscription.request(:2394) 
	at org.springframework.http.server.reactive.ChannelSendOperator$WriteBarrier.onSubscribe(:164)
	at reactor.core.publisher.FluxJust.subscribe(:68) 
	at org.springframework.http.server.reactive.ChannelSendOperator.subscribe(:75)
	at reactor.core.publisher.InternalMonoOperator.subscribe(:64) 
....
	at reactor.core.publisher.FluxSwitchIfEmpty$SwitchIfEmptySubscriber.onComplete(:76) 
	at reactor.core.publisher.Operators.complete(:136) 
	at reactor.core.publisher.MonoEmpty.subscribe(:46) 
	at reactor.core.publisher.Mono.subscribe(:4252) 
	at reactor.core.publisher.FluxOnErrorResume$ResumeSubscriber.onError(:97) 
	at reactor.core.publisher.MonoFlatMap$FlatMapMain.onError(:165) 
	at reactor.core.publisher.MonoFlatMap$FlatMapMain.secondError(:185) 
	at reactor.core.publisher.MonoFlatMap$FlatMapInner.onError(:251) 
	at reactor.core.publisher.FluxOnErrorResume$ResumeSubscriber.onError(:100) 
	at reactor.core.publisher.Operators.error(:197) 
	at reactor.core.publisher.MonoError.subscribe(:53) 
	at reactor.core.publisher.Mono.subscribe(:4252) 
	at reactor.core.publisher.FluxOnErrorResume$ResumeSubscriber.onError(:97) 
	at reactor.core.publisher.FluxPeek$PeekSubscriber.onError(:215) 
	at reactor.core.publisher.FluxPeek$PeekSubscriber.onError(:215) 
	at reactor.core.publisher.MonoIgnoreThen$ThenIgnoreMain.onError(:268) 
	at reactor.core.publisher.MonoFlatMap$FlatMapMain.onError(:165) 
	at reactor.core.publisher.MonoZip$ZipInner.onError(:343) 
	at reactor.core.publisher.MonoPeekTerminal$MonoTerminalPeekSubscriber.onError(:251) 
	at reactor.core.publisher.Operators$MonoSubscriber.onError(:1860) 
	at reactor.core.publisher.Operators$MultiSubscriptionSubscriber.onError(:2060) 
	at reactor.core.publisher.FluxOnErrorResume$ResumeSubscriber.onError(:100) 
	at reactor.core.publisher.Operators.error(:197) 
	at reactor.core.publisher.MonoError.subscribe(:53) 
	at reactor.core.publisher.Mono.subscribe(:4252) 
	at reactor.core.publisher.FluxOnErrorResume$ResumeSubscriber.onError(:97) 
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onError(:134) 
	at reactor.core.publisher.FluxContextStart$ContextStartSubscriber.onError(:110) 
	at reactor.core.publisher.FluxMapFuseable$MapFuseableConditionalSubscriber.onError(:326) 
	at reactor.core.publisher.FluxFilterFuseable$FilterFuseableConditionalSubscriber.onError(:375) 
	at reactor.core.publisher.MonoCollectList$MonoCollectListSubscriber.onError(:106) 
	at reactor.core.publisher.FluxMap$MapSubscriber.onError(:126) 
	at reactor.core.publisher.FluxPeek$PeekSubscriber.onError(:215) 
	at reactor.core.publisher.FluxMap$MapSubscriber.onError(:126) 
	at reactor.core.publisher.Operators$MultiSubscriptionSubscriber.onError(:2060) 
	at reactor.core.publisher.MonoIgnoreElements$IgnoreElementsSubscriber.onError(:77) 
	at reactor.core.publisher.Operators.error(:197) 
	at reactor.netty.FutureMono$DeferredFutureMono.subscribe(:215) 
	at reactor.core.publisher.Mono.subscribe(:4252) 
	at reactor.core.publisher.FluxConcatArray$ConcatArraySubscriber.onComplete(:208) 
	at reactor.core.publisher.FluxConcatArray.subscribe(:81) 
	at reactor.core.publisher.InternalFluxOperator.subscribe(:62) 
	at reactor.netty.ByteBufFlux.subscribe(:290) 
	at reactor.core.publisher.InternalFluxOperator.subscribe(:62) 
	at reactor.netty.ByteBufFlux.subscribe(:290) 
	at reactor.core.publisher.Mono.subscribe(:4252) 
....
	at reactor.core.publisher.FluxConcatMap$ConcatMapImmediate.innerNext(:275) 
	at reactor.core.publisher.FluxConcatMap$ConcatMapInner.onNext(:852) 
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(:121) 
....
	at reactor.core.publisher.InternalMonoOperator.subscribe(:64) 
	at reactor.netty.http.server.HttpServerHandle.onStateChange(:64) 
	at reactor.netty.tcp.TcpServerBind$ChildObserver.onStateChange(:226) 
	at reactor.netty.http.server.HttpServerOperations.onInboundNext(:434) 
	at reactor.netty.channel.ChannelOperationsHandler.channelRead(:141) 

	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(:340) 
	at reactor.netty.http.server.HttpTrafficHandler.channelRead(:159) 
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(:362) 
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(:348) 
	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(:340) 
	at io.netty.channel.CombinedChannelDuplexHandler$DelegatingChannelHandlerContext.fireChannelRead(:438) 
	at io.netty.handler.codec.ByteToMessageDecoder.fireChannelRead(:323) 
	at io.netty.handler.codec.ByteToMessageDecoder.channelRead(:297) 
	at io.netty.channel.CombinedChannelDuplexHandler.channelRead(:253) 
....
	at io.netty.channel.epoll.EpollEventLoop.run(:330)
	at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(:897)
	at java.lang.Thread.run(Thread.java:748) [?:1.8.0_222-ea]
Caused by: java.lang.IllegalStateException: unexpected message type: DefaultHttpResponse, state: 3
	at io.netty.handler.codec.http.HttpObjectEncoder.encode(:86) 
	at io.netty.handler.codec.MessageToMessageEncoder.write(:88)
	... 177 more
```

## 已知线索

- 该错误只发生在 “/api/messagepush/batchtransimission” 接口，并且错误请求和正常请求为交替出现，请求量与错误数量比为 100 : 1左右。
- 该接口controller的方法参数  -> (@RequestBody String body)
- 该请求的客户端使用短连接，在C++中使用curl组件 请求类型为 POST，请求头Content-length 使用手动设置实际数据长度，Content-Type 没有设置使用默认 multipart/form-data
- 所有异常堆栈相同

# 问题分析

## 直接异常

在异常堆栈信息最后可以看到一个直接的错误信息：

```
Caused by: java.lang.IllegalStateException: unexpected message type: DefaultHttpResponse, state: 3
	at io.netty.handler.codec.http.HttpObjectEncoder.encode(:86) 
```

通过对客户版本的netty框架  io.netty.handler.codec.http.HttpObjectEncoder#encode 方法源码。

```java
private static final int ST_INIT = 0;
private static final int ST_CONTENT_NON_CHUNK = 1;
private static final int ST_CONTENT_CHUNK = 2;
private static final int ST_CONTENT_ALWAYS_EMPTY = 3;

private int state = ST_INIT;

@Override
protected void encode(ChannelHandlerContext ctx, Object msg, List<Object> out) throws Exception {
    ByteBuf buf = null;
    // 如果 msg 是响应头对象
    if (msg instanceof HttpMessage) {
        if (state != ST_INIT) {
            // 如果 state 等于 ST_INIT 则说明已经向响应流中写入过 响应头了，
            throw new IllegalStateException("unexpected message type: " + StringUtil.simpleClassName(msg)  + ", state: " + state);
        }

        // 判断 下一个状态为什么： 
        // ST_CONTENT_ALWAYS_EMPTY： 请求类型为 HEAD / 响应状态码为 1XX,204,304,205， 表示没有响应体数据
        // ST_CONTENT_CHUNK：响应头中 transfer-encoding 为 chunked， 即为 分块数据响应
        // ST_CONTENT_NON_CHUNK： 响应数据为连续字节流
        state = isContentAlwaysEmpty(m) ? ST_CONTENT_ALWAYS_EMPTY :
       			 HttpUtil.isTransferEncodingChunked(m) ? ST_CONTENT_CHUNK : ST_CONTENT_NON_CHUNK;

    	// 将响应头数据写入 响应流中
    }
    if (msg instanceof ByteBuf) {
        final ByteBuf potentialEmptyBuf = (ByteBuf) msg;
        if (!potentialEmptyBuf.isReadable()) {
            // // 将响应头数据写入 响应流中
            out.add(potentialEmptyBuf.retain());
            return;
        }
    }

    if (msg instanceof HttpContent || msg instanceof ByteBuf || msg instanceof FileRegion) {
        switch (state) {
            case ST_INIT:
                throw new IllegalStateException("unexpected message type: " + StringUtil.simpleClassName(msg));
            case ST_CONTENT_NON_CHUNK:
                final long contentLength = contentLength(msg);
                if (contentLength > 0) {
				  // ... 向流中写入数据
                    if (msg instanceof LastHttpContent) {
                        state = ST_INIT;
                    }
                    break;
                }
            case ST_CONTENT_ALWAYS_EMPTY:
				// ... 向流中写入数据
                break;
            case ST_CONTENT_CHUNK:
            	// ... 向流中写入数据
                break;
            default:
                throw new Error();
        }
        if (msg instanceof LastHttpContent) {
            state = ST_INIT;
        }
    } else if (buf != null) {
        out.add(buf);
    }
}
```

通过对上代码观察和分析：

- state 字段只有 msg为LastHttpContent实例时 才可以重置为 0
- 当 msg 为 HttpMessage实例时 会设置成 1，2，3
- 当需要 status 为 3 只有  请求类型为 HEAD 或者 响应状态码为 1XX,204,304,205
- 当 msg 为 HttpMessage实例时 但是 status 不为 0 时，则就会抛出 IllegalStateException 异常 

所以有上信息可以推断出 HttpObjectEncoder 在响应 两个 HttpMessage 实例之间没有 LastHttpContent实例。

## HttpObjectEncoder 生命周期

查看源码可以发现 state 字段为 HttpObjectEncoder 类中的普通字段，即生命周期与 HttpObjectEncoder 对象一致。

同时在查看异常信息   **java.lang.IllegalStateException: unexpected message type: DefaultHttpResponse, state: 3** 发现该msg是为 DefaultHttpResponse 类型， HttpObjectEncoder 的具体实例为 io.netty.handler.codec.http.HttpServerCodec.HttpServerResponseEncoder。

![image-20251204203550296]({{site.baseurl}}/img/in-post/problem-EncoderExceptionANDIllegalStateException/image-20251204203550296.png)

通过查看代码及其结构，发现 HttpServerResponseEncoder 与 HttpServerCodec 为组合关系，其生命周期相等。

在通过查看 HttpServerCodec 类构造函数的调用方：仅在 HttpServerBind 类中有调用，用于 channel 的 initChannel 操作，将 HttpServerCodec  对象设置到 channel 的 pipeline 中，所以 HttpServerCodec  对象生命周期与 channel 相同。

```java
ChannelPipeline p = channel.pipeline();

p.addLast(NettyPipeline.HttpCodec, new HttpServerCodec(line, header, chunk, validate, buffer)); 
```

而 channel 的生命周期与 连接相同，请求又是使用的短链接，则  HttpObjectEncoder 生命周期 与 请求相同。

那么说明 status 在每次请求中都会独立的，则请求之间不会影响到 status 的状态处理。

## 异常发生位置

在异常堆栈中有发现了一连串的 onError  方法调用，即表明该位置发生了异常只是因为reactor机制掩盖住了实际发生的异常错误，并且因为这个错误直接激活了向响应流中写入错误信息。

```
	at reactor.core.publisher.FluxFilterFuseable$FilterFuseableConditionalSubscriber.onError(:375) 
	at reactor.core.publisher.MonoCollectList$MonoCollectListSubscriber.onError(:106) 
	at reactor.core.publisher.FluxMap$MapSubscriber.onError(:126) 
	at reactor.core.publisher.FluxPeek$PeekSubscriber.onError(:215) 
	at reactor.core.publisher.FluxMap$MapSubscriber.onError(:126) 
	at reactor.core.publisher.Operators$MultiSubscriptionSubscriber.onError(:2060) 
	at reactor.core.publisher.MonoIgnoreElements$IgnoreElementsSubscriber.onError(:77) 
	at reactor.core.publisher.Operators.error(:197) 
	at reactor.netty.FutureMono$DeferredFutureMono.subscribe(:215) 
	at reactor.core.publisher.Mono.subscribe(:4252) 
	at reactor.core.publisher.FluxConcatArray$ConcatArraySubscriber.onComplete(:208) 
	at reactor.core.publisher.FluxConcatArray.subscribe(:81) 
	at reactor.core.publisher.InternalFluxOperator.subscribe(:62) 
	at reactor.netty.ByteBufFlux.subscribe(:290) 
	at reactor.core.publisher.InternalFluxOperator.subscribe(:62) 
	at reactor.netty.ByteBufFlux.subscribe(:290) 
```

并且在第一个 error 之前有一个 ByteBufFlux  reactor对象：用于读取请求流数据的对象，并且该对象还只是处于订阅阶段说明该异常发生在 还没有将请求body数据处理完成阶段，也就说明错误是发生在执行Controller方法之前。

同时在堆栈中没有发现 FluxReceive 也可以证明这一点：

![image-20251204203602218]({{site.baseurl}}/img/in-post/problem-EncoderExceptionANDIllegalStateException/image-20251204203602218.png)

FluxReceive 与 HttpServerOprations 为组合关系，FluxReceive在ChannelOperations构造函数中创建。

同时 FluxReceive 中的 receiverQueue 队列用于存储请求的body数据，

```java
final public void channelRead(ChannelHandlerContext ctx, Object msg) {
    // ops 在 HttpTrafficHandler#channelRead 方法中设置到 channel中的 HttpServerOperations
    ChannelOperations<?, ?> ops = ChannelOperations.get(ctx.channel());
    if (ops != null) {
        ops.onInboundNext(ctx, msg);
    } else {
        // ... 记录日志
    }
}
```

在 reactor.netty.http.server.HttpServerOperations#onInboundNext 方法中：

```java
@Override
protected void onInboundNext(ChannelHandlerContext ctx, Object msg) {
   if (msg instanceof HttpRequest) {
      listener().onStateChange(this, ConnectionObserver.State.CONFIGURED);
      if (msg instanceof FullHttpRequest) {
         super.onInboundNext(ctx, msg); // 该方法会将msg压入到FluxReceive#receiverQueue的队列
      }
      return;
   }
   if (msg instanceof HttpContent) {
      if (msg != LastHttpContent.EMPTY_LAST_CONTENT) {
         super.onInboundNext(ctx, msg); // 该方法会将msg压入到FluxReceive#receiverQueue的队列
      }
      if (msg instanceof LastHttpContent) {
         onInboundComplete();
      }
   } else {
      super.onInboundNext(ctx, msg);
   }
}
```

等到所有数据接收完成的时候就会所有的 HttpContent 读取出来解析成Controller中需要的参数，并且再执行Controller 的方法。

# 测试环境排查

通过对测试环境开启debug日志，然后对从消息中心数据中台发起的请求进行观察，发现在日志中有出现了 响应状态码为 100 （continue）的响应 HttpFullRespose 对象，响应状态码为 1XX 时，就会将 HttpObjectEncoder 对象中 status 字段设置为 3。

![image-20251204203616138]({{site.baseurl}}/img/in-post/problem-EncoderExceptionANDIllegalStateException/image-20251204203616138.png)

则在框架源码中寻找  HttpFullRespose 对象是从何处创建的：

在 reactor.netty.http.server.HttpServerOperations#CONTINUE 字段为一个常量（以下称为 Continue对象）。

```java
// 为 HttpResponse、LastHttpContent 的实例
final static FullHttpResponse CONTINUE =
	new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.CONTINUE,EMPTY_BUFFER);
```

# 客户端请求分析（消息中心数据中台）

响应Continue对象的作用：在客户端发起Post请求，如果body数据比较大，就会先将带有  "Expect:100-continue" 头的请求头发送到服务端，待服务端根据请求头判断可以接收body数据时就会先响应Continue对象给客户端，客户端收到Continue对象响应后就会将body发送给服务端，之后服务器收到body数据就进行处理业务响应结果。

客户端查看源码是使用libCurl，查看资料发现 [libCurl][https://www.w3.org/Protocols/rfc2616/rfc2616-sec8.html#sec8.2.3] 在发送post数据时，如果body大于1024子节就会默认使用 "Expect:100-continue" 。

如果不想使用Continue，可以通过手动在请求头中添加 "Expect:" 头（Expect等于空）

# 探针处理逻辑

探针会在 HttpObjectEncoder#encode 方法中加入类似以下 addHeader 方法的逻辑：

```java
public class HttpObjectEncoder {
    protected void encode(ChannelHandlerContext ctx, Object msg, List<Object> out) {
        if (msg instanceof HttpResponse) {
             msg.addHeader("Access-Control-Expose-Headers", "traceresponse");
             msg.addHeader("Access-Control-Expose-Headers", "x-br-response");
        }
          
        // 原始逻辑代码
    }
}
```

当 msg 为普通对象是无影响。

当 请求有携带"Expect:100-continue" 头并且框架使用的是 reactor-netty（Spring-WebFLux）， msg 就会为上述中 HttpServerOperations#CONTINUE 常量对象，addHeader 的数据会一直保留着，并且 CONTINUE 常量每调用一次 HttpObjectEncode#encode 方法就会增加一些响应头数据。

# 实验复现

将 jmeter 的压测请求中 添加请求头 "Expect:100-continue" 使得这些请求会响应 Continue对象。

在使用带有 Continue头的请求 1000并发 和 没有带 Continue 头的请求 1000并发 压测 二十多分钟，复现问题。

# 问题原因分析

```java
@Override
protected void encode(ChannelHandlerContext ctx, Object msg, List<Object> out) throws Exception {
    ByteBuf buf = null;
    if (msg instanceof HttpMessage) {
        if (state != ST_INIT) {
            throw new IllegalStateException("unexpected message type: " + StringUtil.simpleClassName(msg)+ ", state: " + state);
        }
        H m = (H) msg;

        buf = ctx.alloc().buffer((int) headersEncodedSizeAccumulator);

        encodeInitialLine(buf, m);
        state = isContentAlwaysEmpty(m) ? ST_CONTENT_ALWAYS_EMPTY :
                HttpUtil.isTransferEncodingChunked(m) ? ST_CONTENT_CHUNK : ST_CONTENT_NON_CHUNK;
		// 将 header数据写到 buf 中
        encodeHeaders(m.headers(), buf);
        ByteBufUtil.writeShortBE(buf, CRLF_SHORT);
    }
}
```

随着Continue对象中的header数量越来越多，其体积也就越来越大，在 encodeHeaders 方法中将 Continue对象 中的header写入到 buf (堆外内存中)。

当带有 continue 头的请求并发过多时 buf 可能就无法在堆外内存中找到足够的空间，就会抛出OOM异常，但是这个异常会被Spring-WebFlux 捕获（在错误日志堆栈中 reactor.core.publisher.Operators.error(:197) ），在对应的异常处理器中，将其错误信息输出到响应流中，这时就会生成 DefaultHttpResponse 对象执行 HttpObjectEncoder#encode 方法，但是这时的 status 数值被Continue对象改成了 3 ，因为抛出异常的原因所以没有在encode方法末尾将 status 改回成 0， 这样就抛出了日志中的错误。

![image-20230804004714374]({{site.baseurl}}/img/in-post/problem-EncoderExceptionANDIllegalStateException/image-20230804004714374.png)

对于为什么500只会在一个接口产生：

- 只有这个接口/api/messagepush/batchtransimission有携带 Continue头数据。
- 在前期 Continue 对象携带的header数量还不多时，需要出现500错误就需要 /api/messagepush/batchtransimission 请求的并发高才行（这样才能同时把堆外内存占用满，在连接关闭之后内存就会释放了），因为并发高了netty处理业务的线程就只有几个，所以正在处理运行的请求都是 /api/messagepush/batchtransimission 接口。
- 在后期 Continue 对象非常庞大时，只需要一两个/api/messagepush/batchtransimission的请求就可以把堆外内存给占用满了， 对于其他请求进来就直接无法申请到堆外内存了，也就直接报OOM了。

# 解决方案

探针将修改 HttpObjectEncoder#encode 方法中addHeader的逻辑，使得httpRespose对象即使为常量也不会无限制增加header。