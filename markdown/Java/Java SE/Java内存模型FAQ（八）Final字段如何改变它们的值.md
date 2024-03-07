# Java内存模型FAQ（八）Final字段如何改变它们的值

> _转载自_[并发编程网 – ifeve.com](http://ifeve.com/)**本文链接地址:** [Java内存模型FAQ（八）Final字段如何改变它们的值](http://ifeve.com/jmm-faq-finalwrong/)

我们可以通过分析String类的实现具体细节来展示一个final变量是如何可以改变的。
String对象包含了三个字段：一个character数组，一个数组的offset和一个length。实现String类的基本原理为：它不仅仅拥有character数组，而且为了避免多余的对象分配和拷贝，多个String和StringBuffer对象都会共享相同的character数组。因此，String.substring()方法能够通过改变length和offset，而共享原始的character数组来创建一个新的String。对一个String来说，这些字段都是final型的字段。
```java
String s1 = "/usr/tmp";
String s2 = s1.substring(4);
```
字符串s2的offset的值为4，length的值为4。但是，在旧的内存模型下，对其他线程来说，看到offset拥有默认的值0是可能的，而且，稍后一点时间会看到正确的值4，好像字符串的值从“/usr”变成了“/tmp”一样。
旧的Java内存模型允许这些行为，部分JVM已经展现出这样的行为了。在新的Java内存模型里面，这些是非法的。
## 原文
## How can final fields appear to change their values?
One of the best examples of how final fields’ values can be seen to change involves one particular implementation of the `String` class.
A `String` can be implemented as an object with three fields — a character array, an offset into that array, and a length. The rationale for implementing `String` this way, instead of having only the character array, is that it lets multiple `String` and `StringBuffer`objects share the same character array and avoid additional object allocation and copying. So, for example, the method `String.substring()` can be implemented by creating a new string which shares the same character array with the original `String` and merely differs in the length and offset fields. For a `String`, these fields are all final fields.
```java
String s1 = "/usr/tmp";
String s2 = s1.substring(4);
```
The string `s2` will have an offset of 4 and a length of 4. But, under the old model, it was possible for another thread to see the offset as having the default value of 0, and then later see the correct value of 4, it will appear as if the string “/usr” changes to “/tmp”.
The original Java Memory Model allowed this behavior; several JVMs have exhibited this behavior. The new Java Memory Model makes this illegal.
