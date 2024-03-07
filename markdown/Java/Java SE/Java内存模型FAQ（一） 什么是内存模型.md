# Java内存模型FAQ（一） 什么是内存模型

> 转载自[并发编程网 – ifeve.com](http://ifeve.com/)本文链接地址: [Java内存模型FAQ（一） 什么是内存模型](http://ifeve.com/memory-model/)

在多核系统中，处理器一般有一层或者多层的缓存，这些的缓存通过加速数据访问（因为数据距离处理器更近）和降低共享内存在总线上的通讯（因为本地缓存能够满足许多内存操作）来提高CPU性能。缓存能够大大提升性能，但是它们也带来了许多挑战。例如，当两个CPU同时检查相同的内存地址时会发生什么？在什么样的条件下它们会看到相同的值？
在处理器层面上，内存模型定义了一个充要条件，“让当前的处理器可以看到其他处理器写入到内存的数据”以及“其他处理器可以看到当前处理器写入到内存的数据”。有些处理器有很强的内存模型(strong memory model)，能够让所有的处理器在任何时候任何指定的内存地址上都可以看到完全相同的值。而另外一些处理器则有较弱的内存模型（weaker memory model），在这种处理器中，必须使用内存屏障（一种特殊的指令）来刷新本地处理器缓存并使本地处理器缓存无效，目的是为了让当前处理器能够看到其他处理器的写操作或者让其他处理器能看到当前处理器的写操作。这些内存屏障通常在lock和unlock操作的时候完成。内存屏障在高级语言中对程序员是不可见的。
在强内存模型下，有时候编写程序可能会更容易，因为减少了对内存屏障的依赖。但是即使在一些最强的内存模型下，内存屏障仍然是必须的。设置内存屏障往往与我们的直觉并不一致。近来处理器设计的趋势更倾向于弱的内存模型，因为弱内存模型削弱了缓存一致性，所以在多处理器平台和更大容量的内存下可以实现更好的可伸缩性
“一个线程的写操作对其他线程可见”这个问题是因为编译器对代码进行重排序导致的。例如，只要代码移动不会改变程序的语义，当编译器认为程序中移动一个写操作到后面会更有效的时候，编译器就会对代码进行移动。如果编译器推迟执行一个操作，其他线程可能在这个操作执行完之前都不会看到该操作的结果，这反映了缓存的影响。
此外，写入内存的操作能够被移动到程序里更前的时候。在这种情况下，其他的线程在程序中可能看到一个比它实际发生更早的写操作。所有的这些灵活性的设计是为了通过给编译器，运行时或硬件灵活性使其能在最佳顺序的情况下来执行操作。在内存模型的限定之内，我们能够获取到更高的性能。
看下面代码展示的一个简单例子：
```java
ClassReordering {

	int x = 0, y = 0;

    public void writer() {
        x = 1;
        y = 2;
    }

    public void reader() {
        int r1 = y;
        int r2 = x;
    }

}
```
让我们看在两个并发线程中执行这段代码，读取_Y_变量将会得到2这个值。因为这个写入比写到_X_变量更晚一些，程序员可能认为读取_X_变量将肯定会得到1。但是，写入操作可能被重排序过。如果重排序发生了，那么，就能发生对_Y_变量的写入操作，读取两个变量的操作紧随其后，而且写入到X这个操作能发生。程序的结果可能是r1变量的值是2，但是r2变量的值为0。
Java内存模型描述了在多线程代码中哪些行为是合法的，以及线程如何通过内存进行交互。它描述了“程序中的变量“ 和 ”从内存或者寄存器获取或存储它们的底层细节”之间的关系。Java内存模型通过使用各种各样的硬件和编译器的优化来正确实现以上事情。
Java包含了几个语言级别的关键字，包括：volatile, final以及synchronized，目的是为了帮助程序员向编译器描述一个程序的并发需求。Java内存模型定义了volatile和synchronized的行为，更重要的是保证了同步的java程序在所有的处理器架构下面都能正确的运行。
## 原文
## What is a memory model, anyway?
In multiprocessor systems, processors generally have one or more layers of memory cache, which improves performance both by speeding access to data (because the data is closer to the processor) and reducing traffic on the shared memory bus (because many memory operations can be satisfied by local caches.) Memory caches can improve performance tremendously, but they present a host of new challenges. What, for example, happens when two processors examine the same memory location at the same time? Under what conditions will they see the same value?
At the processor level, a memory model defines necessary and sufficient conditions for knowing that writes to memory by other processors are visible to the current processor, and writes by the current processor are visible to other processors. Some processors exhibit a strong memory model, where all processors see exactly the same value for any given memory location at all times. Other processors exhibit a weaker memory model, where special instructions, called memory barriers, are required to flush or invalidate the local processor cache in order to see writes made by other processors or make writes by this processor visible to others. These memory barriers are usually performed when lock and unlock actions are taken; they are invisible to programmers in a high level language.
It can sometimes be easier to write programs for strong memory models, because of the reduced need for memory barriers. However, even on some of the strongest memory models, memory barriers are often necessary; quite frequently their placement is counterintuitive. Recent trends in processor design have encouraged weaker memory models, because the relaxations they make for cache consistency allow for greater scalability across multiple processors and larger amounts of memory.
The issue of when a write becomes visible to another thread is compounded by the compiler’s reordering of code. For example, the compiler might decide that it is more efficient to move a write operation later in the program; as long as this code motion does not change the program’s semantics, it is free to do so. If a compiler defers an operation, another thread will not see it until it is performed; this mirrors the effect of caching.
Moreover, writes to memory can be moved earlier in a program; in this case, other threads might see a write before it actually “occurs” in the program. All of this flexibility is by design — by giving the compiler, runtime, or hardware the flexibility to execute operations in the optimal order, within the bounds of the memory model, we can achieve higher performance.
A simple example of this can be seen in the following code:
```java
Class Reordering {
  int x = 0, y = 0;
  public void writer() {
    x = 1;
    y = 2;
  }

  public void reader() {
    int r1 = y;
    int r2 = x;
  }
}
```
Let’s say that this code is executed in two threads concurrently, and the read of y sees the value 2. Because this write came after the write to x, the programmer might assume that the read of x must see the value 1. However, the writes may have been reordered. If this takes place, then the write to y could happen, the reads of both variables could follow, and then the write to x could take place. The result would be that r1 has the value 2, but r2 has the value 0.
The Java Memory Model describes what behaviors are legal in multithreaded code, and how threads may interact through memory. It describes the relationship between variables in a program and the low-level details of storing and retrieving them to and from memory or registers in a real computer system. It does this in a way that can be implemented correctly using a wide variety of hardware and a wide variety of compiler optimizations.
Java includes several language constructs, including volatile, final, and synchronized, which are intended to help the programmer describe a program’s concurrency requirements to the compiler. The Java Memory Model defines the behavior of volatile and synchronized, and, more importantly, ensures that a correctly synchronized Java program runs correctly on all processor architectures.
