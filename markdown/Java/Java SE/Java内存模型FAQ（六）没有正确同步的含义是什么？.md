# Java内存模型FAQ（六）没有正确同步的含义是什么？

> _转载自_[并发编程网 – ifeve.com](http://ifeve.com/)**本文链接地址:** [Java内存模型FAQ（六）没有正确同步的含义是什么？](http://ifeve.com/jmm-faq-incorrectlysync/)

没有正确同步的代码对于不同的人来说可能会有不同的理解。在Java内存模型这个语义环境下，我们谈到“没有正确同步”，我们的意思是：

1. 一个线程中有一个对变量的写操作，
2. 另外一个线程对同一个变量有读操作，
3. 而且写操作和读操作没有通过同步来保证顺序。

当这些规则被违反的时候，我们就说在这个变量上有一个“数据竞争”(data race)。一个有数据竞争的程序就是一个没有正确同步的程序。
## 原文
## What do you mean by “incorrectly synchronized”?
Incorrectly synchronized code can mean different things to different people. When we talk about incorrectly synchronized code in the context of the Java Memory Model, we mean any code where

1. there is a write of a variable by one thread,
2. there is a read of the same variable by another thread and
3. the write and read are not ordered by synchronization

When these rules are violated, we say we have a _data race_ on that variable. A program with a data race is an incorrectly synchronized program.
