# Java内存模型FAQ（三）JSR133是什么？

> _转载自_[并发编程网 – ifeve.com](http://ifeve.com/)**本文链接地址:** [Java内存模型FAQ（三）JSR133是什么？](http://ifeve.com/jsr133/)

从1997年以来，人们不断发现Java语言规范的17章定义的Java内存模型中的一些严重的缺陷。这些缺陷会导致一些使人迷惑的行为（例如final字段会被观察到值的改变）和破坏编译器常见的优化能力。
Java内存模型是一个雄心勃勃的计划，它是编程语言规范第一次尝试合并一个能够在各种处理器架构中为并发提供一致语义的内存模型。不过，定义一个既一致又直观的内存模型远比想象要更难。JSR133为Java语言定义了一个新的内存模型，它修复了早期内存模型中的缺陷。为了实现JSR133，final和volatile的语义需要重新定义。
完整的语义见：http://www.cs.umd.edu/users/pugh/java/memoryModel，但是正式的语义不是小心翼翼的，它是令人惊讶和清醒的，目的是让人意识到一些看似简单的概念（如同步）其实有多复杂。幸运的是，你不需要懂得这些正式语义的细节——JSR133的目的是创建一组正式语义，这些正式语义提供了volatile、synchronzied和final如何工作的直观框架。
JSR 133的目标包含了：

-  保留已经存在的安全保证（像类型安全）以及强化其他的安全保证。例如，变量值不能凭空创建：线程观察到的每个变量的值必须是被其他线程合理的设置的。 
-  正确同步的程序的语义应该尽量简单和直观。 
-  应该定义未完成或者未正确同步的程序的语义，主要是为了把潜在的安全危害降到最低。 
-  程序员应该能够自信的推断多线程程序如何同内存进行交互的。 
-  能够在现在许多流行的硬件架构中设计正确以及高性能的JVM实现。 
-  应该能提供 _安全地初始化_的保证。如果一个对象正确的构建了（意思是它的引用没有在构建的时候逸出，那么所有能够看到这个对象的引用的线程，在不进行同步的情况下，也将能看到在构造方法中中设置的final字段的值。 
-  应该尽量不影响现有的代码。 
## 原文
## What is JSR 133 about?
Since 1997, several serious flaws have been discovered in the Java Memory Model as defined in Chapter 17 of the Java Language Specification. These flaws allowed for confusing behaviors (such as final fields being observed to change their value) and undermined the compiler’s ability to perform common optimizations.
The Java Memory Model was an ambitious undertaking; it was the first time that a programming language specification attempted to incorporate a memory model which could provide consistent semantics for concurrency across a variety of architectures. Unfortunately, defining a memory model which is both consistent and intuitive proved far more difficult than expected. JSR 133 defines a new memory model for the Java language which fixes the flaws of the earlier memory model. In order to do this, the semantics of final and volatile needed to change.
The full semantics are available at [http://www.cs.umd.edu/users/pugh/java/memoryModel](http://www.cs.umd.edu/users/pugh/java/memoryModel), but the formal semantics are not for the timid. It is surprising, and sobering, to discover how complicated seemingly simple concepts like synchronization really are. Fortunately, you need not understand the details of the formal semantics — the goal of JSR 133 was to create a set of formal semantics that provides an intuitive framework for how volatile, synchronized, and final work.
The goals of JSR 133 include:

- Preserving existing safety guarantees, like type-safety, and strengthening others. For example, variable values may not be created “out of thin air”: each value for a variable observed by some thread must be a value that can reasonably be placed there by some thread.
- The semantics of correctly synchronized programs should be as simple and intuitive as possible.
- The semantics of incompletely or incorrectly synchronized programs should be defined so that potential security hazards are minimized.
- Programmers should be able to reason confidently about how multithreaded programs interact with memory.
- It should be possible to design correct, high performance JVM implementations across a wide range of popular hardware architectures.
- A new guarantee of _initialization safety_ should be provided. If an object is properly constructed (which means that references to it do not escape during construction), then all threads which see a reference to that object will also see the values for its final fields that were set in the constructor, without the need for synchronization.
- There should be minimal impact on existing code.
