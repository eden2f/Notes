# JVM 垃圾收集器日志及参数配置

## 虚拟机及垃圾收集器日志
在JDK 9以前，HotSpot并没有提供统一的日志处理框架，虚拟机功能模块的日志开关分布在不同的参数上，日志级别、循环日志大小、输出格式、重定向等设置在不同功能上都要单独解决。直到JDK 9，这种混乱不堪的局面才终于消失，Hot Spot所有功能的日志都受到了“-Xlog”参数上。
## 查看GC基本信息
	在JDK 9之前使用-XX: +PrintGC，JDK 9后使用-Xlog:gc
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303160905.png#id=yVBvP&originHeight=71&originWidth=549&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 查看GC详细信息
在JDK 9之前使用-XX:+PrintGCDetails，在JDK 9之后使用-Xlong:gc_，用通配符 _ 将GC标签下所有细分过程都打印出来，如果把日志级别调整到Debug或者Trace，还能获得更多细节信息:
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303161011.png#id=akQM3&originHeight=296&originWidth=938&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 查看GC前后的堆、方法去可用容量变化
在JDK 9之前使用-XX:+PrintHeapAtGC，在JDK 9之后使用-Xlog:gc+heap=debug
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303161041.png#id=GWAIG&originHeight=404&originWidth=903&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 查看GC过程中用户线程并发时间以及停顿的时间
在JDK 9之前使用-XX:+PrintGCApplicationConcurrentTime以及-XX:+PrintGCApplicationStoppedTime，在JDK 9之后使用 -Xlog:safepoint
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303161101.png#id=GGy27&originHeight=174&originWidth=954&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 查看收集器Ergonomics机制自动调节的相关信息
（自动设置堆空间各分代区域大小、收集目标等内容，从Parallel收集器开始支持）。在JDK 9之前使用-XX:+PrintAdaptive-SizePolicy，JDK 9之后使用-Xlog:gc+ergo*=trace
