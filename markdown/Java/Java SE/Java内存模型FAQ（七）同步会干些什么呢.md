# Java内存模型FAQ（七）同步会干些什么呢

> _转载自_[并发编程网 – ifeve.com](http://ifeve.com/)**本文链接地址:** [Java内存模型FAQ（七）同步会干些什么呢](http://ifeve.com/jmm-faq-synchronization/)

同步有几个方面的作用。最广为人知的就是互斥 ——一次只有一个线程能够获得一个监视器，因此，在一个监视器上面同步意味着一旦一个线程进入到监视器保护的同步块中，其他的线程都不能进入到同一个监视器保护的块中间，除非第一个线程退出了同步块。
但是同步的含义比互斥更广。同步保证了一个线程在同步块之前或者在同步块中的一个内存写入操作以可预知的方式对其他有相同监视器的线程可见。当我们退出了同步块，我们就**释放**了这个监视器，这个监视器有刷新缓冲区到主内存的效果，因此该线程的写入操作能够为其他线程所见。在我们进入一个同步块之前，我们需要**获取**监视器，监视器有使本地处理器缓存失效的功能，因此变量会从主存重新加载，于是其它线程对共享变量的修改对当前线程来说就变得可见了。
依据缓存来讨论同步，可能听起来这些观点仅仅会影响到多处理器的系统。但是，重排序效果能够在单一处理器上面很容易见到。对编译器来说，在获取之前或者释放之后移动你的代码是不可能的。当我们谈到在缓冲区上面进行的获取和释放操作，我们使用了简述的方式来描述大量可能的影响。
新的内存模型语义在内存操作（读取字段，写入字段，锁，解锁）以及其他线程的操作（start 和 join）中创建了一个部分排序，在这些操作中，一些操作被称为_happen before_其他操作。当一个操作在另外一个操作之前发生，第一个操作保证能够排到前面并且对第二个操作可见。这些排序的规则如下：

-  线程中的每个操作_happens before_该线程中在程序顺序上后续的每个操作。 
-  解锁一个监视器的操作_happens before_随后对**相同**监视器进行锁的操作。 
-  对volatile字段的写操作_happens_ _before_后续对**相同**volatile字段的读取操作。 
-  线程上调用start()方法_happens_ _before_这个线程启动后的任何操作。 
-  一个线程中所有的操作都_happens before_从这个线程join()方法成功返回的任何其他线程。（注意思是其他线程等待一个线程的jion()方法完成，那么，这个线程中的所有操作_happens_ _before_其他线程中的所有操作） 

这意味着：任何内存操作，这个内存操作在退出一个同步块前对一个线程是可见的，对任何线程在它进入一个被相同的监视器保护的同步块后都是可见的，因为所有内存操作happens before释放监视器以及释放监视器happens before获取监视器。
其他如下模式的实现被一些人用来强迫实现一个内存屏障的，不会生效：
```
synchronized (new Object()) {}
```
这段代码其实不会执行任何操作，你的编译器会把它完全移除掉，因为编译器知道没有其他的线程会使用相同的监视器进行同步。要看到其他线程的结果，你必须为一个线程建立happens before关系。
**重点注意**：对两个线程来说，为了正确建立happens before关系而在相同监视器上面进行同步是非常重要的。以下观点是错误的：当线程A在对象X上面同步的时候，所有东西对线程A可见，线程B在对象Y上面进行同步的时候，所有东西对线程B也是可见的。释放监视器和获取监视器必须匹配（也就是说要在相同的监视器上面完成这两个操作），否则，代码就会存在“数据竞争”。
## 原文
## What does synchronization do?
Synchronization has several aspects. The most well-understood is mutual exclusion — only one thread can hold a monitor at once, so synchronizing on a monitor means that once one thread enters a synchronized block protected by a monitor, no other thread can enter a block protected by that monitor until the first thread exits the synchronized block.
But there is more to synchronization than mutual exclusion. Synchronization ensures that memory writes by a thread before or during a synchronized block are made visible in a predictable manner to other threads which synchronize on the same monitor. After we exit a synchronized block, we **release** the monitor, which has the effect of flushing the cache to main memory, so that writes made by this thread can be visible to other threads. Before we can enter a synchronized block, we **acquire** the monitor, which has the effect of invalidating the local processor cache so that variables will be reloaded from main memory. We will then be able to see all of the writes made visible by the previous release.
Discussing this in terms of caches, it may sound as if these issues only affect multiprocessor machines. However, the reordering effects can be easily seen on a single processor. It is not possible, for example, for the compiler to move your code before an acquire or after a release. When we say that acquires and releases act on caches, we are using shorthand for a number of possible effects.
The new memory model semantics create a partial ordering on memory operations (read field, write field, lock, unlock) and other thread operations (start and join), where some actions are said to _happen before_ other operations. When one action happens before another, the first is guaranteed to be ordered before and visible to the second. The rules of this ordering are as follows:

- Each action in a thread happens before every action in that thread that comes later in the program’s order.
- An unlock on a monitor happens before every subsequent lock on **that same** monitor.
- A write to a volatile field happens before every subsequent read of **that same** volatile.
- A call to `start()` on a thread happens before any actions in the started thread.
- All actions in a thread happen before any other thread successfully returns from a `join()`on that thread.

This means that any memory operations which were visible to a thread before exiting a synchronized block are visible to any thread after it enters a synchronized block protected by the same monitor, since all the memory operations happen before the release, and the release happens before the acquire.
Another implication is that the following pattern, which some people use to force a memory barrier, doesn’t work:
```java
synchronized (new Object()) {}
```
This is actually a no-op, and your compiler can remove it entirely, because the compiler knows that no other thread will synchronize on the same monitor. You have to set up a happens-before relationship for one thread to see the results of another.
**Important Note:** Note that it is important for both threads to synchronize on the same monitor in order to set up the happens-before relationship properly. It is not the case that everything visible to thread A when it synchronizes on object X becomes visible to thread B after it synchronizes on object Y. The release and acquire have to “match” (i.e., be performed on the same monitor) to have the right semantics. Otherwise, the code has a data race.
