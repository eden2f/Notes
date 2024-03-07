# Java内存模型FAQ（四）重排序意味着什么？

> _转载自_[并发编程网 – ifeve.com](http://ifeve.com/)**本文链接地址:** [Java内存模型FAQ（四）重排序意味着什么？](http://ifeve.com/jmm-faq-reordering/)

在很多情况下，访问一个程序变量（对象实例字段，类静态字段和数组元素）可能会使用不同的顺序执行，而不是程序语义所指定的顺序执行。编译器能够自由的以优化的名义去改变指令顺序。在特定的环境下，处理器可能会次序颠倒的执行指令。数据可能在寄存器，处理器缓冲区和主内存中以不同的次序移动，而不是按照程序指定的顺序。
例如，如果一个线程写入值到字段a，然后写入值到字段b，而且b的值不依赖于a的值，那么，处理器就能够自由的调整它们的执行顺序，而且缓冲区能够在a之前刷新b的值到主内存。有许多潜在的重排序的来源，例如编译器，JIT以及缓冲区。
编译器，运行时和硬件被期望一起协力创建好像是顺序执行的语义的假象，这意味着在单线程的程序中，程序应该是不能够观察到重排序的影响的。但是，重排序在没有正确同步了的多线程程序中开始起作用，在这些多线程程序中，一个线程能够观察到其他线程的影响，也可能检测到其他线程将会以一种不同于程序语义所规定的执行顺序来访问变量。
大部分情况下，一个线程不会关注其他线程正在做什么，但是当它需要关注的时候，这时候就需要同步了。
## 原文
## What is meant by reordering?
There are a number of cases in which accesses to program variables (object instance fields, class static fields, and array elements) may appear to execute in a different order than was specified by the program. The compiler is free to take liberties with the ordering of instructions in the name of optimization. Processors may execute instructions out of order under certain circumstances. Data may be moved between registers, processor caches, and main memory in different order than specified by the program.
For example, if a thread writes to field `a` and then to field `b`, and the value of `b` does not depend on the value of `a`, then the compiler is free to reorder these operations, and the cache is free to flush `b` to main memory before `a`. There are a number of potential sources of reordering, such as the compiler, the JIT, and the cache.
The compiler, runtime, and hardware are supposed to conspire to create the illusion of as-if-serial semantics, which means that in a single-threaded program, the program should not be able to observe the effects of reorderings. However, reorderings can come into play in incorrectly synchronized multithreaded programs, where one thread is able to observe the effects of other threads, and may be able to detect that variable accesses become visible to other threads in a different order than executed or specified in the program.
Most of the time, one thread doesn’t care what the other is doing. But when it does, that’s what synchronization is for.
