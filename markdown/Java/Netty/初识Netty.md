# 初识Netty

> 《Netty 实战》

## 前言
之前有个这样的一个bug，服务A的接口1依赖服务B的接口2，由于服务B接口2耗时升高，由于服务A的接口1是同步等待服务B接口2的响应，导致服务A的可用线程几乎都在等待服务B接口2的响应，而服务A其他功能接口也因为没有可用线程导致功能不可用。
上面这是做一个简单的描述，当然实际情况远比这样复杂，但是从这个bug中还是发现了我们系统设计上的很多优化点。

1. 将特殊场景（并发量比较大的功能）与业务主流程从服务上区分开，避免由于某个功能影响整个系统的稳定。
2. 非主流程业务依赖接口超过预期最大时间还没响应，应该做快速失败处理，防止线程长时间挂起等待中。
3. 当前服务调用线程模型用的是BIO，是一种同步阻塞的IO模式，在业务使用上，可以理解为一个请求对应服务的一个线程。如果是AIO，异步非阻塞，即一个有效请求对应一个线程。

好了，终于说到IO模式，这才是本文的主题。
## 常见的IO模型有哪些
IO模式通常分为几种，同步阻塞的BIO、同步非阻塞的NIO、异步非阻塞的AIO（NIO2）。

-  BIO模式
BIO 就是传统的 java.io 包，它是基于流模型实现的，交互的方式是同步、阻塞方式，也就是说在读入输入流或者输出流时，在读写动作完成之前，线程会一直阻塞在那里。
让我们考虑一下这种方案的影响。第一，在任何时候都可能有大量的线程处于休眠状态，只是等待输入或者输出数据就绪，这可能算是一种资源浪费。第二，需要为每个线程的调用栈都分配内存，其默认值大小区间为64KB到1MB，具体取决于操作系统。第三，即使Java虚拟机（JVM）在物理上可以支持非常大数量的线程，但是远在到达该极限之前，上下文切换所带来的开销就会带来麻烦，例如，在达到10000个连接的时候。 

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240319230815.png#id=O6cLY&originHeight=1167&originWidth=2602&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

-  NIO模式
Java 1.4 引入， class java.nio.channels.Selector是Java的非阻塞I/O实现的关键。它使用了事件通知API以确定在一组非阻塞套接字中有哪些已经就绪能够进行I/O相关的操作。因为可以在任何的时间检查任意的读操作或者写操作的完成状态，如图所示，一个单一的线程便可以处理多个并发的连接。总体来看，与阻塞I/O模型相比，这种模型提供了更好的资源管理：使用较少的线程便可以处理许多连接，因此也减少了内存管理和上下文切换所带来开销；当没有I/O操作需要处理的时候，线程也可以被用于其他任务。 

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240319230835.png#id=ghcvs&originHeight=1413&originWidth=2058&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

- AIO（NIO2）模式

AIO 是 Java 1.7 之后引入的包，是 NIO 的升级版本，提供了异步非堵塞的IO操作方式，所以人们叫它AIO（Asynchronous IO），异步IO是基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会堵塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。
## Netty 的优势在哪
Netty 是由 JBOSS 提供的一个 java 开源框架.  Netty 提供异步/事件驱动的网络应用程序框架和工具, 用以快速开发高性能/高可靠性的网络服务器和客户端程序.
也就是说, Netty 是一个基于 NIO 的客户/服务端编程框架, 使用 Netty 可以确保你快速和简单的开发出一个网络应用, 例如实现了某种协议的客户/服务端应用. Netty 相当于简化和流线化了网络应用的编程开发过程, 例如: 基于 TCP 和 UDP 的 socket 服务开发.
"快速" 和 "简单" 并不用产生维护性或性能上的问题. Netty 是一个吸收了多种协议(包括 FTP/SMTP/HTTP 等各种二进制文本协议) 的实现经验, 并经过相当精心设计的项目. 最终, Netty 成功的找到了一种方式, 在保证易于开发的同时还保证了其应用的性能/稳定性和伸缩性.

| 分类 | Netty的特性 |
| --- | --- |
| 设计 | 统一API，支持多种传输类型，阻塞的和非阻塞的；简单而强大的线程模型；真正的无连接数据报套接字支持；链接逻辑组件以支持复用； |
| 易于使用 | 详实的Javadoc和大量示例集；不需要超过JDK1.6+的依赖； |
| 性能 | 拥有比Java核心API更高的吞吐量以及更低的延迟；得益于池化和复用，拥有更低的资源消耗；最少的内存复制； |
| 健壮性 | 不会因为慢速、快速或者超载的连接而导致OutOfMemoryError；消除在高速网络中NIO应用常见的不公平读/写比例； |
| 安全性 | 完整的SSL/TLS以及StartTLS支持；可用于受限环境下，如Applet和OSGI； |
| 社区驱动 | 发布快速而且频繁； |

好吧，发现手抄一边特性后，更迫不及待想试试了。
## Hello Netty
服务端实现
```java
public class NettyServer {

    public static void main(String[] args) throws Exception {
        //1. 创建一个线程组：接收客户端连接
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        //2. 创建一个线程组：处理网络操作
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        //3. 创建服务器端启动助手来配置参数
        ServerBootstrap b = new ServerBootstrap();
        //4.设置两个线程组
        b.group(bossGroup, workerGroup)
                //5.使用NioServerSocketChannel作为服务器端通道的实现
                .channel(NioServerSocketChannel.class)
                //6.设置线程队列中等待连接的个数
                .option(ChannelOption.SO_BACKLOG, 128)
                //7.保持活动连接状态
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                //9. 往Pipeline链中添加自定义的handler类
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    //8. 创建一个通道初始化对象
                    @Override
                    public void initChannel(SocketChannel sc) {
                        sc.pipeline().addLast(new NettyServerHandler());
                    }
                });
        System.out.println("......Server is ready......");
        //10. 绑定端口 bind方法是异步的  sync方法是同步阻塞的
        ChannelFuture cf = b.bind(9999).sync();
        System.out.println("......Server is starting......");

        //11. 关闭通道，关闭线程组
        cf.channel().closeFuture().sync();
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }
}
```
```java
public class NettyServerHandler extends ChannelInboundHandlerAdapter {

    /**
     * 读取数据事件
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        System.out.println("Server:" + ctx);
        ByteBuf buf = (ByteBuf) msg;
        System.out.println("客户端发来的消息：" + buf.toString(CharsetUtil.UTF_8));
    }

    /**
     * 数据读取完毕事件
     */
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.writeAndFlush(Unpooled.copiedBuffer("就是没钱", CharsetUtil.UTF_8));
    }

    /**
     * 异常发生事件
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable t) {
        ctx.close();
    }

}
```
服务端启动日志
```shell
......Server is ready......
......Server is starting......
Server:ChannelHandlerContext(NettyServerHandler#0, [id: 0xd3d1c749, L:/127.0.0.1:9999 - R:/127.0.0.1:51856])
客户端发来的消息：老板，还钱吧

Process finished with exit code 130 (interrupted by signal 2: SIGINT)
```
客户端实现
```java
public class NettyClient {

    public static void main(String[] args) throws Exception {

        //1. 创建一个线程组
        EventLoopGroup group = new NioEventLoopGroup();
        //2. 创建客户端的启动助手，完成相关配置
        Bootstrap b = new Bootstrap();
        //3. 设置线程组
        b.group(group)
                //4. 设置客户端通道的实现类
                .channel(NioSocketChannel.class)
                //5. 创建一个通道初始化对象
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel socketChannel) {
                        //6.往Pipeline链中添加自定义的handler
                        socketChannel.pipeline().addLast(new NettyClientHandler());
                    }
                });
        System.out.println("......Client is  ready......");

        //7.启动客户端去连接服务器端  connect方法是异步的   sync方法是同步阻塞的
        ChannelFuture cf = b.connect("127.0.0.1", 9999).sync();

        //8.关闭连接(异步非阻塞)
        cf.channel().closeFuture().sync();

    }
}
```
```java
public class NettyClientHandler extends ChannelInboundHandlerAdapter {

    /**
     * 通道就绪事件
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        System.out.println("Client:" + ctx);
        ctx.writeAndFlush(Unpooled.copiedBuffer("老板，还钱吧", CharsetUtil.UTF_8));
    }

    /**
     * 读取数据事件
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf buf = (ByteBuf) msg;
        System.out.println("服务器端发来的消息：" + buf.toString(CharsetUtil.UTF_8));
    }

}
```
客户端启动日志
```shell
......Client is  ready......
Client:ChannelHandlerContext(NettyClientHandler#0, [id: 0xc181fc12, L:/127.0.0.1:51856 - R:/127.0.0.1:9999])
服务器端发来的消息：就是没钱

Process finished with exit code 130 (interrupted by signal 2: SIGINT)
```
有兴趣的同学可以试试，用Java核心API分别用BIO、NIO、NIO2实现一下上述程序，然后再用Netty实现一遍……
