# Java内存模型FAQ（十三）为什么我需要关注java内存模型

> _转载自_[并发编程网 – ifeve.com](http://ifeve.com/)**本文链接地址:** [Java内存模型FAQ（十三）为什么我需要关注java内存模型](http://ifeve.com/jmm-faq-conclusion/)

为什么你需要关注java内存模型？并发程序的bug非常难找。它们经常不会在测试中发生，而是直到你的程序运行在高负荷的情况下才发生，非常难于重现和跟踪。你需要花费更多的努力提前保证你的程序是正确同步的。这不容易，但是它比调试一个没有正确同步的程序要容易的多。
## 原文
## Why should I care?
Why should you care? Concurrency bugs are very difficult to debug. They often don’t appear in testing, waiting instead until your program is run under heavy load, and are hard to reproduce and trap. You are much better off spending the extra effort ahead of time to ensure that your program is properly synchronized; while this is not easy, it’s a lot easier than trying to debug a badly synchronized application.
