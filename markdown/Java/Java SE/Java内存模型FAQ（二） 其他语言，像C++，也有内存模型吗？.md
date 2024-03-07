# Java内存模型FAQ（二） 其他语言，像C++，也有内存模型吗？

> _转载自_[并发编程网 – ifeve.com](http://ifeve.com/)**本文链接地址:** [Java内存模型FAQ（二） 其他语言，像C++，也有内存模型吗？](http://ifeve.com/java-faq-otherlanguages/)

大部分其他的语言，像C和C++，都没有被设计成直接支持多线程。这些语言对于发生在编译器和处理器平台架构的重排序行为的保护机制会严重的依赖于程序中所使用的线程库（例如pthreads），编译器，以及代码所运行的平台所提供的保障。
## 原文
## Do other languages, like C++, have a memory model?
Most other programming languages, such as C and C++, were not designed with direct support for multithreading. The protections that these languages offer against the kinds of reorderings that take place in compilers and architectures are heavily dependent on the guarantees provided by the threading libraries used (such as pthreads), the compiler used, and the platform on which the code is run.
