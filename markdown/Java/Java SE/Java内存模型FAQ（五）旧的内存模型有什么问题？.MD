# Java内存模型FAQ（五）旧的内存模型有什么问题？

> _转载自_[并发编程网 – ifeve.com](http://ifeve.com/)**本文链接地址:**  [Java内存模型FAQ（五）旧的内存模型有什么问题？](http://ifeve.com/jmm-faq-old-mm/)

旧的内存模型中有几个严重的问题。这些问题很难理解，因此被广泛的违背。例如，旧的存储模型在许多情况下，不允许JVM发生各种重排序行为。旧的内存模型中让人产生困惑的因素造就了JSR-133规范的诞生。
例如，一个被广泛认可的概念就是，如果使用final字段，那么就没有必要在多个线程中使用同步来保证其他线程能够看到这个字段的值。尽管这是一个合理的假设和明显的行为，也是我们所期待的结果。实际上，在旧的内存模型中，我们想让程序正确运行起来却是不行的。在旧的内存模型中，final字段并没有同其他字段进行区别对待——这意味着同步是保证所有线程看到一个在构造方法中初始化的final字段的唯一方法。结果——如果没有正确同步的话，对一个线程来说，它可能看到一个字段的默认值，然后在稍后的时间里，又能够看到构造方法中设置的值。这意味着，一些不可变的对象，例如String，能够改变它们值——这实在很让人郁闷。
旧的内存模型允许volatile变量的写操作和非volaitle变量的读写操作一起进行重排序，这和大多数的开发人员对于volatile变量的直观感受是不一致的，因此会造成迷惑。
最后，我们将看到的是，程序员对于程序没有被正确同步的情况下将会发生什么的直观感受通常是错误的。JSR-133的目的之一就是要引起这方面的注意。
## 原文
## What was wrong with the old memory model?
There were several serious problems with the old memory model. It was difficult to understand, and therefore widely violated. For example, the old model did not, in many cases, allow the kinds of reorderings that took place in every JVM. This confusion about the implications of the old model was what compelled the formation of JSR-133.
One widely held belief, for example, was that if final fields were used, then synchronization between threads was unnecessary to guarantee another thread would see the value of the field. While this is a reasonable assumption and a sensible behavior, and indeed how we would want things to work, under the old memory model, it was simply not true. Nothing in the old memory model treated final fields differently from any other field — meaning synchronization was the only way to ensure that all threads see the value of a final field that was written by the constructor. As a result, it was possible for a thread to see the default value of the field, and then at some later time see its constructed value. This means, for example, that immutable objects like String can appear to change their value — a disturbing prospect indeed.
The old memory model allowed for volatile writes to be reordered with nonvolatile reads and writes, which was not consistent with most developers intuitions about volatile and therefore caused confusion.
Finally, as we shall see, programmers’ intuitions about what can occur when their programs are incorrectly synchronized are often mistaken. One of the goals of JSR-133 is to call attention to this fact.
