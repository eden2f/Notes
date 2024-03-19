# Java 运行程序至少会创建几个线程

> [JVM故障分析系列](https://blog.csdn.net/wuxiao2020/article/details/100162559)
[使用Java提供的MXBean来监控jvm创建了哪些线程](https://www.cnblogs.com/jiaoyiping/p/9250668.html)

## 开门见山
```java
public class Hello {

    public static void main(String[] args) {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        long[] threadIds = threadMXBean.getAllThreadIds();
        ThreadInfo[] threadInfos = threadMXBean.getThreadInfo(threadIds);
        for (ThreadInfo threadInfo : threadInfos) {
            System.out.println(threadInfo.getThreadId()+": "+threadInfo.getThreadName());
        }
    }
}
```
程序输出如下：
```shell
5: Attach Listener
4: Signal Dispatcher
3: Finalizer
2: Reference Handler
1: main
```
可以看到，这个Hello程序执行过程创建了5个线程。
为什么需要这些线程？ThreadMXBean是干嘛用的呢？
## JVM内部线程
### Attach Listener
该线程负责接收外部命令，执行该命令并把结果返回给调用者，此种类型的线程通常在桌面程序中出现。
```java
"Attach Listener" daemon prio=5 tid=0x00007fc6b6800800 nid=0x3b07 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
```
### Signal Dispatcher
Attach Listener线程的职责是接收外部jvm命令，当命令接收成功后，会交给signal dispather 线程去进行分发到各个不同的模块处理命令，并且返回处理结果。
signal dispather线程也是在第一次接收外部jvm命令时，进行初始化工作。
```java
"Signal Dispatcher" daemon prio=10 tid=0x00007fbea81bf800 nid=0x5ef runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
```
### Finalizer
这个线程也是在main线程之后创建的，其优先级为10，主要用于在垃圾收集前，调用对象的finalize()方法；关于Finalizer线程的几点：
（1）只有当开始一轮垃圾收集时，才会开始调用finalize()方法；因此并不是所有对象的finalize()方法都会被执行；
（2）该线程也是daemon线程，因此如果虚拟机中没有其他非daemon线程，不管该线程有没有执行完finalize()方法，JVM也会退出；
（3）JVM在垃圾收集时会将失去引用的对象包装成Finalizer对象（Reference的实现），并放入ReferenceQueue，由Finalizer线程来处理；最后将该Finalizer对象的引用置为null，由垃圾收集器来回收；
（4）JVM为什么要单独用一个线程来执行finalize()方法呢？
如果JVM的垃圾收集线程自己来做，很有可能由于在finalize()方法中误操作导致GC线程停止或不可控，这对GC线程来说是一种灾难。
```java
"Finalizer" daemon prio=10 tid=0x00007fbea80da000 nid=0x5eb in Object.wait() [0x00007fbeac044000]
   java.lang.Thread.State: WAITING (on object monitor)
    at java.lang.Object.wait(Native Method)
    at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:135)
    - locked <0x00000006d173c1a8> (a java.lang.ref.ReferenceQueue$Lock)
    at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:151)
    at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)
```
### Reference Handler
JVM在创建main线程后就创建Reference Handler线程，其优先级最高，为10，它主要用于处理引用对象本身（软引用、弱引用、虚引用）的垃圾回收问题 。
```java
"Reference Handler" daemon prio=10 tid=0x00007fbea80d8000 nid=0x5ea in Object.wait() [0x00007fbeac085000]
   java.lang.Thread.State: WAITING (on object monitor)
    at java.lang.Object.wait(Native Method)
    at java.lang.Object.wait(Object.java:503)
    at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:133)
    - locked <0x00000006d173c1f0> (a java.lang.ref.Reference$Lock)
```
## MXBean
MBean是一种JavaBean,MBean往往代表的是JMX中的一种可以被管理的资源。MBean会通过接口定义，给出这些资源的一些特定操作：

- 属性的读和写操作
- 可以被执行的操作
- 关于自己的描述信息

MXBean是一种特殊的MBean,不仅特殊在名字不一样,主要是在于在接口中会引用到一些其他类型的类时，其表现方式的不一样。在MXBean中，如果一个MXBean的接口定义了一个属性是一个自定义类型，如果MXBean定义了一种自定义的类型，当JMX使用这个MXBean时，这个自定义类型就会被转换成一种标准的类型，这些类型被称为开放类型，是定义在javax.management.openmbean包中的。
而这个转换的规则是，如果是原生类型，如int或者是String，则不会有变化，但如果是其他自定义类型，则被转换成CompositeDataSupport类,这样,JMX调用这个MXBean提供的接口的时候,classpath下没有这个自定义类型也是可以调用成功的,但是换做MBean,则调用发的classpath下必须存在这个自定义类型的类定义
java本身提供了一些关于线程,内存,垃圾回收和日志等管理的MXBean和一个ManagementFactory的静态工厂类,通过这些事先提供的类,我们可以监控java进程的线程创建,内存日志级别和垃圾回收等,当然,我们也可以通过创建我们自己的MXBean来实现我们想实现的一些功能。
## 其他
如果你是用JProfiler启动应用程序，你能看到如下线程：
```shell
9: _jprofiler_sampler
6: _jprofiler_control_sampler
5: Attach Listener
4: Signal Dispatcher
3: Finalizer
2: Reference Handler
1: main
```
