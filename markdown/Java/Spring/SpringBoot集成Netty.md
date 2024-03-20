# SpringBoot集成Netty

## 前言
上一篇<<初识Netty>>, 我们讲解了如何用Netty实现一个WebSocket服务端,作为实时聊天室的服务器.
本文的场景是, 我们已经有一个SpringBoot的Web服务了, 希望将WebSocket的能力整合到SpringBoot开发的Web服务中,也就是随着Web服务的启动而启动, 随着Web服务的关闭而关闭.
## 监听ContextRefreshedEvent
ContextRefreshedEvent : 上下文刷新事件是在 Spring 应用上下文(ApplicationContext)刷新之后发送。
Spring Boot 启动事件顺序

- ApplicationStartingEvent : 这个事件在 Spring Boot 应用运行开始时，且进行任何处理之前发送(除了监听器和初始化器注册之外)。
- ApplicationEnvironmentPreparedEvent : 这个事件在当已知要在上下文中使用 Spring 环境(Environment)时，在 Spring 上下文(context)创建之前发送。
- ApplicationContextInitializedEvent : 这个事件在当 Spring 应用上下文(ApplicationContext)准备好了，并且应用初始化器(ApplicationContextInitializers)已经被调用，在 bean 的定义(bean definitions)被加载之前发送。
- ApplicationPreparedEvent : 这个事件是在 Spring 上下文(context)刷新之前，且在 bean 的定义(bean definitions)被加载之后发送。
- ApplicationStartedEvent : 这个事件是在 Spring 上下文(context)刷新之后，且在 application/ command-line runners 被调用之前发送。
- AvailabilityChangeEvent : 这个事件紧随上个事件之后发送，状态：ReadinessState.CORRECT，表示应用已处于活动状态。
- ApplicationReadyEvent : 这个事件在任何 application/ command-line runners 调用之后发送。
- AvailabilityChangeEvent : 这个事件紧随上个事件之后发送，状态：ReadinessState.ACCEPTING_TRAFFIC，表示应用可以开始准备接收请求了。
```java
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextRefreshedEvent;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;

@Component
public class NettyStartListener implements ApplicationListener<ContextRefreshedEvent> {

    @Resource
    private WebSocketNettyServer webSocketNettyServer;

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (event.getApplicationContext().getParent() == null) {
            try {
                webSocketNettyServer.start();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```
## Netty业务处理实现
除了 WebSocketNettyServer, 还需要 ChatHandler / WebSocketChannelInitializer, 这两个跟上一篇文章内容的编码完全一样, 懒得搬了, 有需要自己去找下吧. <<Netty基于WebSocket实现的简单聊天室>>
```java
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import org.springframework.stereotype.Component;

@Component
public class WebSocketNettyServer {

    // Netty服务器启动对象
    private final ServerBootstrap serverBootstrap;

    public WebSocketNettyServer() {
        serverBootstrap = new ServerBootstrap();
        // 初始化服务器启动对象
        // 主线程池
        NioEventLoopGroup mainGrp = new NioEventLoopGroup();
        // 从线程池
        NioEventLoopGroup subGrp = new NioEventLoopGroup();
        serverBootstrap
                // 指定使用上面创建的两个线程池
                .group(mainGrp, subGrp)
                // 指定Netty通道类型
                .channel(NioServerSocketChannel.class)
                // 指定通道初始化器用来加载当Channel收到事件消息后，
                // 如何进行业务处理
                .childHandler(new WebSocketChannelInitializer());
    }

    public void start() {
        // 绑定服务器端口，以异步的方式启动服务器
        ChannelFuture future = serverBootstrap.bind(9090);
    }
}
```
## 验证
客户端1:
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320231626.png#id=njIl1&originHeight=366&originWidth=1880&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
客户端2:
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320231646.png#id=QlMSe&originHeight=351&originWidth=2002&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
服务端运行日志:
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240320231702.png#id=RQAIJ&originHeight=1358&originWidth=2633&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
