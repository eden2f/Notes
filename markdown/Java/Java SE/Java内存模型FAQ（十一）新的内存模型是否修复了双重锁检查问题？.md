# Java内存模型FAQ（十一）新的内存模型是否修复了双重锁检查问题？

> _转载自_[并发编程网 – ifeve.com](http://ifeve.com/)**本文链接地址:** [Java内存模型FAQ（十一）新的内存模型是否修复了双重锁检查问题？](http://ifeve.com/jmm-faq-dcl/)

臭名昭著的双重锁检查（也叫多线程单例模式）是一个骗人的把戏，它用来支持lazy初始化，同时避免过度使用同步。在非常早的JVM中，同步非常慢，开发人员非常希望删掉它。双重锁检查代码如下：
```java
// double-checked-locking - don't do this!

private static Something instance = null;

public Something getInstance() {
  if (instance == null) {
    synchronized (this) {
      if (instance == null)
        instance = new Something();
    }
  }
  return instance;
}
```
这看起来好像非常聪明——在公用代码中避免了同步。这段代码只有一个问题 —— 它不能正常工作。为什么呢？最明显的原因是，初始化实例的写入操作和实例字段的写入操作能够被编译器或者缓冲区重排序，重排序可能会导致返回部分构造的一些东西。就是我们读取到了一个没有初始化的对象。这段代码还有很多其他的错误，以及为什么对这段代码的算法修正是错误的。在旧的java内存模型下没有办法修复它。更多深入的信息可参见：[Double-checkedlocking: Clever but broken](http://www.javaworld.com/jw-02-2001/jw-0209-double.html) and [The “DoubleChecked Locking is broken” declaration](http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)
许多人认为使用volatile关键字能够消除双重锁检查模式的问题。在1.5的JVM之前，volatile并不能保证这段代码能够正常工作（因环境而定）。在新的内存模型下，实例字段使用volatile可以解决双重锁检查的问题，因为在构造线程来初始化一些东西和读取线程返回它的值之间有happens-before关系。
然后，对于喜欢使用双重锁检查的人来说（我们真的希望没有人这样做），仍然不是好消息。双重锁检查的重点是为了避免过度使用同步导致性能问题。从java1.0开始，不仅同步会有昂贵的性能开销，而且在新的内存模型下，使用volatile的性能开销也有所上升，几乎达到了和同步一样的性能开销。因此，使用双重锁检查来实现单例模式仍然不是一个好的选择。（修订—在大多数平台下，volatile性能开销还是比较低的）。
使用IODH来实现多线程模式下的单例会更易读：
```java
private static class LazySomethingHolder {
  public static Something something = new Something();
}

public static Something getInstance() {
  return LazySomethingHolder.something;
}
```
这段代码是正确的，因为初始化是由static字段来保证的。如果一个字段设置在static初始化中，对其他访问这个类的线程来说是是能正确的保证它的可见性的。
## 原文
## Does the new memory model fix the “double-checked locking” problem?
The (infamous) double-checked locking idiom (also called the multithreaded singleton pattern) is a trick designed to support lazy initialization while avoiding the overhead of synchronization. In very early JVMs, synchronization was slow, and developers were eager to remove it — perhaps too eager. The double-checked locking idiom looks like this:
```java
// double-checked-locking - don't do this!

private static Something instance = null;

public Something getInstance() {
  if (instance == null) {
    synchronized (this) {
      if (instance == null)
        instance = new Something();
    }
  }
  return instance;
}
```
This looks awfully clever — the synchronization is avoided on the common code path. There’s only one problem with it — **it doesn’t work**. Why not? The most obvious reason is that the writes which initialize `instance` and the write to the `instance` field can be reordered by the compiler or the cache, which would have the effect of returning what appears to be a partially constructed `Something`. The result would be that we read an uninitialized object. There are lots of other reasons why this is wrong, and why algorithmic corrections to it are wrong. There is no way to fix it using the old Java memory model. More in-depth information can be found at [Double-checked locking: Clever, but broken](http://www.javaworld.com/jw-02-2001/jw-0209-double.html) and [The “Double Checked Locking is broken” declaration](http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)
Many people assumed that the use of the `volatile`keyword would eliminate the problems that arise when trying to use the double-checked-locking pattern. In JVMs prior to 1.5, `volatile` would not ensure that it worked (your mileage may vary). Under the new memory model, making the `instance` field volatile will “fix” the problems with double-checked locking, because then there will be a happens-before relationship between the initialization of the `Something` by the constructing thread and the return of its value by the thread that reads it.
However, for fans of double-checked locking (and we really hope there are none left), the news is still not good. The whole point of double-checked locking was to avoid the performance overhead of synchronization. Not only has brief synchronization gotten a LOT less expensive since the Java 1.0 days, but under the new memory model, the performance cost of using volatile goes up, almost to the level of the cost of synchronization. So there’s still no good reason to use double-checked-locking. _Redacted — volatiles are cheap on most platforms._
Instead, use the Initialization On Demand Holder idiom, which is thread-safe and a lot easier to understand:
```java
private static class LazySomethingHolder {
  public static Something something = new Something();
}

public static Something getInstance() {
  return LazySomethingHolder.something;
}
```
This code is guaranteed to be correct because of the initialization guarantees for static fields; if a field is set in a static initializer, it is guaranteed to be made visible, correctly, to any thread that accesses that class.
