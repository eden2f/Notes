# Java内存模型FAQ（十）volatile是干什么用的

> _转载自_[并发编程网 – ifeve.com](http://ifeve.com/)**本文链接地址:** [Java内存模型FAQ（十）volatile是干什么用的](http://ifeve.com/jmm-faq-volatile/)

Volatile字段是用于线程间通讯的特殊字段。每次读volatile字段都会看到其它线程写入该字段的最新值；实际上，程序员之所以要定义volatile字段是因为在某些情况下由于缓存和重排序所看到的陈旧的变量值是不可接受的。编译器和运行时禁止在寄存器里面分配它们。它们还必须保证，在它们写好之后，它们被从缓冲区刷新到主存中，因此，它们立即能够对其他线程可见。相同地，在读取一个volatile字段之前，缓冲区必须失效，因为值是存在于主存中而不是本地处理器缓冲区。在重排序访问volatile变量的时候还有其他的限制。
在旧的内存模型下，访问volatile变量不能被重排序，但是，它们可能和访问非volatile变量一起被重排序。这破坏了volatile字段从一个线程到另外一个线程作为一个信号条件的手段。
在新的内存模型下，volatile变量仍然不能彼此重排序。和旧模型不同的时候，volatile周围的普通字段的也不再能够随便的重排序了。写入一个volatile字段和释放监视器有相同的内存影响，而且读取volatile字段和获取监视器也有相同的内存影响。事实上，因为新的内存模型在重排序volatile字段访问上面和其他字段（volatile或者非volatile）访问上面有了更严格的约束。当线程A写入一个volatile字段f的时候，如果线程B读取f的话 ，那么对线程A可见的任何东西都变得对线程B可见了。
如下例子展示了volatile字段应该如何使用：
```java
class VolatileExample {
  int x = 0;
  volatile boolean v = false;
  public void writer() {
    x = 42;
    v = true;
  }

  public void reader() {
    if (v == true) {
      //uses x - guaranteed to see 42.
    }
  }
}
```
假设一个线程叫做“writer”，另外一个线程叫做“reader”。对变量v的写操作会等到变量x写入到内存之后，然后读线程就可以看见v的值。因此，如果reader线程看到了v的值为true，那么，它也保证能够看到在之前发生的写入42这个操作。而这在旧的内存模型中却未必是这样的。如果v不是volatile变量，那么，编译器可以在writer线程中重排序写入操作，那么reader线程中的读取x变量的操作可能会看到0。
实际上，volatile的语义已经被加强了，已经快达到同步的级别了。为了可见性的原因，每次读取和写入一个volatile字段已经像一个半同步操作了
**重点注意**：对两个线程来说，为了正确的设置happens-before关系，访问相同的volatile变量是很重要的。以下的结论是不正确的：当线程A写volatile字段f的时候，线程A可见的所有东西，在线程B读取volatile的字段g之后，变得对线程B可见了。释放操作和获取操作必须匹配（也就是在同一个volatile字段上面完成）。
## 原文
## What does volatile do?
Volatile fields are special fields which are used for communicating state between threads. Each read of a volatile will see the last write to that volatile by any thread; in effect, they are designated by the programmer as fields for which it is never acceptable to see a “stale” value as a result of caching or reordering. The compiler and runtime are prohibited from allocating them in registers. They must also ensure that after they are written, they are flushed out of the cache to main memory, so they can immediately become visible to other threads. Similarly, before a volatile field is read, the cache must be invalidated so that the value in main memory, not the local processor cache, is the one seen. There are also additional restrictions on reordering accesses to volatile variables.
Under the old memory model, accesses to volatile variables could not be reordered with each other, but they could be reordered with nonvolatile variable accesses. This undermined the usefulness of volatile fields as a means of signaling conditions from one thread to another.
Under the new memory model, it is still true that volatile variables cannot be reordered with each other. The difference is that it is now no longer so easy to reorder normal field accesses around them. Writing to a volatile field has the same memory effect as a monitor release, and reading from a volatile field has the same memory effect as a monitor acquire. In effect, because the new memory model places stricter constraints on reordering of volatile field accesses with other field accesses, volatile or not, anything that was visible to thread A when it writes to volatile field `f` becomes visible to thread B when it reads `f`.
Here is a simple example of how volatile fields can be used:
```java
class VolatileExample {
  int x = 0;
  volatile boolean v = false;
  public void writer() {
    x = 42;
    v = true;
  }

  public void reader() {
    if (v == true) {
      //uses x - guaranteed to see 42.
    }
  }
}
```
Assume that one thread is calling `writer`, and another is calling `reader`. The write to `v` in `writer` releases the write to `x` to memory, and the read of `v` acquires that value from memory. Thus, if the reader sees the value `true` for v, it is also guaranteed to see the write to 42 that happened before it. This would not have been true under the old memory model. If `v` were not volatile, then the compiler could reorder the writes in `writer`, and `reader`‘s read of `x` might see 0.
Effectively, the semantics of volatile have been strengthened substantially, almost to the level of synchronization. Each read or write of a volatile field acts like “half” a synchronization, for purposes of visibility.
**Important Note:** Note that it is important for both threads to access the same volatile variable in order to properly set up the happens-before relationship. It is not the case that everything visible to thread A when it writes volatile field `f` becomes visible to thread B after it reads volatile field `g`. The release and acquire have to “match” (i.e., be performed on the same volatile field) to have the right semantics.
