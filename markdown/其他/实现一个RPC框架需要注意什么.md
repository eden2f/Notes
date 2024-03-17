# 实现一个RPC框架需要注意什么

> 《重新定义Spring Cloud》
《趣谈网络协议》-- 刘超

## 1. 什么是RPC？如何写一个RPC框架？
Remote Procedure Call，即远程过程调用。之前看了《从零开始写Java Web框架》跟着书中的思路，模仿借鉴SpringMVC实现了一个“自制的SpringMVC”，利用URL路径作为不同方法的调用入口，请求数据、响应数据以JSON的格式利用HTTP进行传输。我认为它就是一个简单的RPC框架，由于基于Java/Servlet的能力进行拓展，底层实现并不我们操作，那Java/Servlet帮我们处理了什么？或者说它帮我们解决了什么问题？（退一步说，其实Servlet本身就是一种远程方法调用实现，我们不过是在其上做功能增强。）
## 2. 写一个RPC框架会有什么问题呢？

- 问题一：如何规定远程调用的语法？
- 问题二：如果传递参数？
- 问题三：如何表示数据？
- 问题四：如何知道一个服务端都实现了哪些远程调用？从哪个端口可以访问这个远程调用？
- 问题五：发生了错误、重传、丢包、性能等问题怎么办？
## 3. RPC框架实现的标准模式
正因为上述问题增加了实现复杂度，个人要去实现RPC框架注定险阻重重。所幸，大牛 Bruce Jay Nelson 定义了RPC的调用标准，它也成了后面提及的RPC框架实现的标准模式。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240317204056.png#id=iNdha&originHeight=707&originWidth=1999&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
图中，Stub 负责将调用接口、方法和参数，通过约定的协议规范进行编码和解码，并通过本地的 RPCRuntime 进行客户端与服务端的数据传输。
有了 Stub 和 RPCRuntime 上面的五个问题中的前三个就基本解决了，分别是：如何规定远程调用的语法？如果传递参数？如何表示数据？
## 4. 服务的注册与发现
如何知道一个服务端都实现了哪些远程调用？从哪个端口可以访问这个远程调用？这不是服务注册与发现的问题吗，Nacos、Eureka、Consul 已经按捺不住，它们的区别主要是开发环境整合、AP与CP选择。
Eureka Server端采用的是P2P的复制模式，但是它不保证复制操作一定能成功，因此它提供的是一个最终一致性的服务实例视图；Client端在Server端的注册信息有一个带气象的租约，一旦Server端在指定期间没有收到Client端发送的心跳，则Server端会认为Client端注册的服务是不健康的，定时任务会将其从注册表中删除。
Nacos、Consul是采用Raft算法，可以提供强一致性的保证，Consul的agent相当于Netflix Ribbon + Netflix Eureka Client,而且对应用来说相对透明，同时相对于Eureka这种集中式的心跳检测机制，Consul的agent可以参与到基于gossip协议的健康检查，分散了Server端心跳的压力。除此之外，Consul为多数据中心提供了开箱即用的原生支持等。
## 5. 网络传输问题
错误、重传、丢包、性能等问题，我们统称为传输问题。
