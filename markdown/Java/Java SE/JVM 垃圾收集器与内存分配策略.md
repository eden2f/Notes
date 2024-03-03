# JVM 垃圾收集器与内存分配策略

## 如何判断一个对象已经"死"了
### 引用记数算法
在对象中添加一个引用计数器，每当有一个地方引用他时，计数器值就加一；当引用失效时，计数器值就减一；任何时刻计数器为零的对象就是不可能再被使用的。客观地说，引用记数算法（Reference Counting）虽然占用了一些额外的内存空间来进行计数，但它的原理简单，判定效率也很高，在大多数情况下它都是一个不错的算法。但是，在Java领域，至少主流地Java虚拟机里面都没有选用引用记数算法来管理内存，主要原因是，记数算法需要配合大量额外处理才能保证正确地工作，譬如单纯的引用记数就很难解决对象之间相互循环引用的问题。
举个例子，请看下面代码清单，两个对象的instance属性分别赋值为对方，除此之外，这两个对象再无任何引用，实际上这两个对象已经不可能再被访问，但是由于他们呢相互引用着对方，导致他们呢的引用记数都不为零，引用记数算法也就无法回收它们。
```java
/**
 * @Description   对象之间相互循环引用是否会被回收
 */
public class ReferenceCountingGC {

    public Object instance = null;

    private static final int _1MB = 1024 * 1024;

    /**
     * 这个成员属性唯一的意义就是占点内存,以便在GC日志中看清楚是否有回收过
     */
    private byte[] bigSize = new byte[2 * _1MB];

    public static void main(String[] args) throws InterruptedException {
        ReferenceCountingGC referenceCountingGC1 = new ReferenceCountingGC();
        ReferenceCountingGC referenceCountingGC2 = new ReferenceCountingGC();
        referenceCountingGC1.instance = referenceCountingGC2;
        referenceCountingGC2.instance = referenceCountingGC1;
        referenceCountingGC1 = null;
        referenceCountingGC2 = null;
        // sleep 5秒 便于观察
        Thread.sleep(5000L);
        // 假设在这个时候发生gc，referenceCountingGC1和referenceCountingGC2能否被回收
        System.gc();
        Thread.currentThread().join();
    }

}
```
运行结果：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303140018.png#id=MyEud&originHeight=552&originWidth=716&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
从运行结果中可以看到尽管这两个对象相互引用，但是还是被回收了，这也从侧面说明了Java虚拟机并不是通用引用记数算法来判断对象是否存活的。
### 可达性分析算法
可达性分析（Feachability Analysis）算法判定对象是否存货的基本思路是通过一系列称为"GC Roots"的根对象作为起始节点集，从这些节点开始，根据引用关系向下搜索，搜索过程所走过的路径称为"引用镰"（Reference Chain），如果某个对象到GC Roots间没有任何引用链相连，或者用图论的话来说就是从GC Roots到这个对象不可达，则证明此对象是不可能再被使用的。
如果所示，对象object5、object6、object7虽然互有关联，但是它们到GC Roots是不可达的，因此它们将会被判定为可回收对象。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303140044.png#id=PKreA&originHeight=766&originWidth=1240&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在Java记数体系里面，固定可作为GC Roots的对象包括以下几种：

1. 在虚拟机栈（栈帧中的本地变量表）中引用的对象，譬如各个线程被调用的方法堆栈中使用到的参数、局部变量、临时变量等。
2. 在方法去中类静态属性引用的对象，譬如Java类的引用类型静态变量。
3. 在方法区中常量引用的对象，譬如字符串常量池（String Table）里的引用。
4. 在本地方法栈中JNI（即通常所说的Native方法）引用的对象。
5. Java虚拟机内部的引用，如基本数据类型对应的Class对象，一些常驻的异常对象（比如NullPointExcepition、OutOfMemoryError）等，还有系统类加载器。
6. 所有被同步锁（synchronized关键字）持有的对象。
7. 反映Java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等。
8. 除了上面列举的以外，根据用户所选用的垃圾收集器以及当前回收的内存区域不同，还可以有其他对象"临时性"地加入，共同构成完成GC Roots集合。譬如后文将会提到的分代收集和局部回收（Partial GC），如果只针对Java堆中某一块区域发起垃圾收集时（如最典型地只针对新生代地垃圾收集），必须考虑到内存区域是虚拟机自己的实现细节（在用户视角里任何内存区域都是不可见的），更不是鼓励封闭的，所以某个区域里的对象完全有可能被位于堆中其他区域的对象所引用，这时候就需要将这些关联区域的对象也一并假如GC Roots集合中去，才能保证可达性分析的正确性。
### 引用
在JDK1.2版之后，Java对引用的概念进行了扩充，将引用分为强引用（Strongly Reference）、软引用（Soft Reference）、弱引用（weak
Reference）和虚引用（Phantom Reference）4中，这4种引用强度一次逐渐减弱。

1. 强引用是最传统的"引用"的定义，是在指程序代码之中普遍存在的引用赋值，即类似"Object obj=new Object()"这种引用关系。无论任何情况下，只要强引用关系还存在，垃圾收集器就永远不会回收掉被引用的对象。
2. 软引用是用来描述一些还有用，但非必须的对象。只被软引用关联着的对象，在系统将要发生内存溢出异常前，会把这些对象列进回收范围之中进行第二次回收，如果这次回收还没有足够的内存，才会抛出内存溢出异常。在JDK1.2版之后提供了SoftReference类来实现软引用。
3. 弱引用也是用来描述哪些非必须对象，但是它的强度比软引用更若一些，被弱引用关联的对象职能生存到下一次垃圾收集发生为止。当垃圾收集器开始工作，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。在JDK1.2之后提供了WeakReference类来实现弱引用。
4. 虚引用也称为"幽灵引用"或者"幻影引用"，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影，也无法通过虚引用来取得一个对象示例。为一个对象设置虚引用关联的唯一目的只是为了能再这个对象被收集器回收时收到一个系统通知。在JDK1.2版之后提供了PhantomReference类来实现虚引用。
### 对象什么时候被回收？
要真正宣告一个对象死亡，至少要经历两次标记过程：如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记，随后进行一次筛选，筛选的条件是此对象是否有必要执行finalize()方法。假如对象没有覆盖finalize()方法，或者finalize()方法已经被虚拟机调用过，那么虚拟机将这两种情况都视为"没有必要执行"。
如果这个对象被判定为确有必要执行finalize()方法，那么该对象将会被放置在一个名为F-Queue的队列之中，并在稍候由一条由虚拟机自动建立的、低调度优先级的Finalizer线程去执行它们的finalize()方法。这里所说的"执行"是指虚拟机会触发这个方法开始运行，但并不承诺一定会等待它运行结束。这样做的原因是，如果某个对象finalize()方法执行缓慢，或者更极端的发生了死循环，将很可能导致F-Queue队列中的其他对象永久处于等待，甚至导致整个内存回收子系统的崩溃。finalize()方法是对象逃脱死亡名运的最后一次机会，稍后收集器将对F-Queue中的对象进行第二次小规模的标记，如果对象要再finalize()中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可，譬如把自己（this关键字）赋值给某个类变量或者对象的成员变量，那在第二次标记时它将被移出"即将回收"的集合；如果对象这时候还没有逃脱，那基本上它就真的要被回收了。
```java
/**
 * @Description 一个对象的自我拯救
 * @Author minpeng.huang
 * @Date 2020/2/5
 */
public class FinalizeEscapeGC {

    public static FinalizeEscapeGC SAVE_HOOK = null;

    private static final int _1MB = 1024 * 1024;

    /**
     * 这个成员属性唯一的意义就是占点内存,以便在GC日志中看清楚是否有回收过
     */
    private byte[] bigSize = new byte[2 * _1MB];


    public void isAlive(){
        System.out.println("yes, i am still alive :)");
    }

    @Override
    public void finalize() throws Throwable {
        super.finalize();
        System.out.println("finalize method executed!");
        SAVE_HOOK = this;
        System.out.println("bigSize is null : " + bigSize == null);
    }

    public static void main(String[] args) throws InterruptedException {
        // 暂停5秒方便查看内存变化
        Thread.sleep(5000L);

        SAVE_HOOK = new FinalizeEscapeGC();
        // 对象第一次拯救自己
        SAVE_HOOK = null;
        // 暂停5秒方便查看内存变化
        Thread.sleep(5000L);
        System.gc();
        // 因为Finalize方法优先级很低，暂停1秒，以等待它
        Thread.sleep(1000L);
        if(SAVE_HOOK != null){
            SAVE_HOOK.isAlive();
        } else {
            System.out.println("no, i am dead :(");
        }

        // 下面这段代码与上面完全一样，但是这次自救失败了
        SAVE_HOOK = null;
        // 暂停5秒方便查看内存变化
        Thread.sleep(5000L);
        System.gc();
        // 因为Finalize方法优先级很低，暂停1秒，以等待它
        Thread.sleep(1000L);
        if(SAVE_HOOK != null){
            SAVE_HOOK.isAlive();
        } else {
            System.out.println("no, i am dead :(");
        }
    }
}
```
运行结果：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303140129.png#id=fQWmM&originHeight=194&originWidth=716&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303140149.png#id=g7lhM&originHeight=464&originWidth=746&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
从运行结果可以看到，SAVE_HOOK对象的finalize()方法确实被垃圾收集器触发过，并且在被收集前成功逃脱了。但是任何一个对象的finalize()方法都只会被系统自动调用一次，如果对象面临下一次回收，他的finalize()方法不会被再次执行，因此第二段代码的自救行动失败了。
疑问：从程序运行过程的内存变化可以看到，其实第一次gc的时候，SAVE_HOOK对象的内存就已经被回收了，这是为什么呢?加了些日志运行结果如下
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303140219.png#id=jTVRp&originHeight=272&originWidth=602&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 回收方法区
方法区的垃圾收集主要回收两部分内容：废弃的常量和不再使用的类型。回收废弃常量与回收Java堆中的对象非常类似。举个常量池中字面量回收的例子，例如一个字符串"java"，换句话说，已经没有其他地方引用这个字面量对象的值是"java"，换句话说，已经没有任何字符串对象引用常量池中的"java"常量，且虚拟机中也没有其他地方引用这个字面量。如果在这时发生内存回收，而且垃圾收集器判断确有必要的话，这个"java"常量就将会被系统清理出常量池。常量池中其他类（接口）、方法、字段的符号引用也与此类似。
判定一个常量是否"废弃"还是相对简单，而要判定一个类型是否属于"不再被使用的类"的条件就比较苛刻了。需要同时满足下面是三个条件：

1. 该类所有的示例都已经被回收，也就是Java堆中不存在该类及其任何派生子类的实例。
2. 加载该类的类加载器已经被回收，这个条件除非是经过精心设计的可替换加载器的场景，如OSGi、JSP的重加载等，否则通常是很难达成的。
3. 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

Java虚拟机被允许对满足上述三个条件的无用类进行回收，这里说的仅仅是"被允许"，而并不是和对象一样，没有引用了就必然会回收。关于是否要对类型进行回收，HotSpot虚拟机提供了-Xnoclassgc参数进行控制，还可以使用-verbose:class以及-XX:+TraceClass-Loading、-XX:+TraceClassUnLoading产看类加载和卸载信息，其中-verbose:class和-XX:+TraceClassLoading可以再Product版的虚拟机中使用，-XX:+TraceClassUnLoading参数需要FastDebug版的虚拟机支持。
在大量使用反射、动态代理、CGLib等字节码框架，动态生成JSP以及OSGi这类频繁自定义类加载器的场景中，通常都需要Java虚拟机具备类型卸载的能力，以保证不会对方法区造成过大的内存压力。
## 垃圾收集算法
从如何判定对象消亡的角度出发，垃圾收集算法可以划分为"引用记数式垃圾收集（Reference Counting GC）"和"追踪式垃圾收集"（Tracing GC）两大类，这两类也常被称作"直接垃圾收集"和"间接垃圾收集"。
### 分代收集理论
当前商业虚拟机的垃圾收集器，大多数都遵循了"分代收集"（Generational Collection）的理论进行设计，分代收集名为理论，实质是一套符合大多数程序运行实际情况的经验法则，它建立在两个分代假说之上：

1.  弱分代假说（Weak Generational Hypothesis）：绝大多数对象都是朝生夕灭的。 
2.  强分代假说（Strong Generational Hypothesis）：熬过越多次垃圾收集过程的对象就越难以消亡。
这两个分代假说共同奠定了多款常用的垃圾收集器的一致的设计原则：收集器应该将Java堆划分出不同的区域，然后将回收对象依据其年龄（年龄即对象熬过垃圾收集过程的次数）分配到不同的区域之中存储。 
3.  跨代引用假说（Intergenerational Reference Hypothesis）：跨代引用相对于同代引用来说仅占极少数。
这其实是可根据前两条假说逻辑推理得出的隐含推论：存在互相引用关系的两个对象，是应该倾向于同时生存或者同时消亡的。举个例子，如果某个新生代对象存在跨代引用，由于老年代对象难以消亡，该引用会使得新生代对象在收集时同样得意存活，进而在年龄增长之后晋升到老年代中，这是跨代引用也随之被消除了。
依据这条假说，我们就不应再为了少量的跨代引用去扫描整个老年代，也不必浪费空间专门记录每一个对象是否存在及存在哪些跨代引用，只需在新生代上建立一个全局的数据结构（该结构被称为"记忆集"，Remembered Set），这个结构把老年代划分成若干小块，标识出老年代的那一块内存会存在跨代引用。
由"分代收集"带来的一些名词，先把定义统一一下： 
4.  部分收集（Partail GC）：指目标不是完整收集整个Java堆的垃圾收集，其中又分为： 
   1. 新生代收集（Minor GC/Young GC）：指目标只是新生代地垃圾收集。
   2. 老年代收集（Major GC/Old GC）：指目标只是老年代的垃圾收集。目前只有CMS收集器会有单独收集老年代的行为。另外请注意"Major GC"这个说法现在有点混淆，在不同资料上常有不同所指，读者需按上下文区分到底是指老年代的收集还是整堆收集。
5.  混合收集（Mixed GC）：指目标是收集整个新生代以及部分老年代的垃圾收集。目前只有G1收集器会有这种行为。 
6.  整堆收集（Full GC）：收集整个Java堆和方法区的垃圾收集。 
### 标记-清除算法
"标记-清除"（Mark-Sweep）算法分为"标记"和"清除"两个阶段：首先标记出所有需要回收的对象，在标记完成后，统一回收掉所有被标记的对象，也可以反过来，标记存活的对象，统一回收所有未被标记的对象。标记过程就是对象是否属于垃圾的判定过程。
"标记-清除"算法是最基础的收集算法，是因为后续的收集算法大多都是以标记-清除算法为基础，对其缺点进行改进而得到的。它的主要缺点有两个：第一个是执行效率不稳定，如果Java堆中包含大量对象，而且其中大部分是需要被回收的，这时必须进行大量标记和清除的动作，导致标记和清除两个过程的执行效率都随对象数量增长而降低；第二个是内存空间的碎片化问题，标记、清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致当以后再程序运行过程中需要分配较大对象时无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303140314.png#id=tQymM&originHeight=591&originWidth=1240&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 标记-复制算法
标记-复制算法常被简称为复制算法。为了解决标记-清除算法面对大量可回收对象时执行效率低的问题，1969年Fenichel提出了一种称为"半区复制"（Semispace Copying）的垃圾收集算法，它将可用内存按容量划分为大小相等的两块，每次只使用其中一块。当这一块的内存用完了，就将还存活这的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。如果内存中多数对象都是存活的，这种算法将会产生大量的内存间复制的开销，但对于多数对象都是可回收的情况，算法需要复制的就是占少数的存活对象，而且每次都是针对整个半区进行内存回收，分配内存时也就不用考虑有空间碎片的复杂情况，只要移动堆顶指针，按顺序分配即可。这样实现简单，运行高效，不过其缺陷也显而易见，这种复制回收算法的代价时将可用内存缩小为了原来的一般，空间浪费未免太多了一点。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303140339.png#id=iXOXr&originHeight=612&originWidth=1240&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 标记-整理算法
针对老年代对象的存亡特征，1974年Edward Lueders提出了另外一种有针对性的“标记-整理”(Mark-Compact)算法，其中的标记过程仍然与“标记-清楚”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向内存空间一端移动，然后直接清理掉边界以外的内存，“标记-整理”算法的示意图如下图所示。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303140404.png#id=wgHxP&originHeight=272&originWidth=523&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)