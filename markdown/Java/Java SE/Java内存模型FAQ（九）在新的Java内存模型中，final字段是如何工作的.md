# Java内存模型FAQ（九）在新的Java内存模型中，final字段是如何工作的

> _转载自_[并发编程网 – ifeve.com](http://ifeve.com/)**本文链接地址:** [Java内存模型FAQ（九）在新的Java内存模型中，final字段是如何工作的](http://ifeve.com/jmm-faq-finalright/)

一个对象的final字段值是在它的构造方法里面设置的。假设对象被正确的构造了，一旦对象被构造，在构造方法里面设置给final字段的的值在没有同步的情况下对所有其他的线程都会可见。另外，引用这些final字段的对象或数组都将会看到final字段的最新值。
对一个对象来说，被正确的构造是什么意思呢？简单来说，它意味着这个正在构造的对象的引用在构造期间没有被允许逸出。（参见[安全构造技术](http://www.ibm.com/developerworks/java/library/j-jtp0618/index.html)）。换句话说，不要让其他线程在其他地方能够看见一个构造期间的对象引用。不要指派给一个静态字段，不要作为一个listener注册给其他对象等等。这些操作应该在构造方法之后完成，而不是构造方法中来完成。
```java
class FinalFieldExample {
  final int x;
  int y;
  static FinalFieldExample f;
  public FinalFieldExample() {
    x = 3;
    y = 4;
  }

  static void writer() {
    f = new FinalFieldExample();
  }

  static void reader() {
    if (f != null) {
      int i = f.x;
      int j = f.y;
    }
  }
}
```
上面的类展示了final字段应该如何使用。一个正在执行reader方法的线程保证看到f.x的值为3，因为它是final字段。它不保证看到f.y的值为4，因为f.y不是final字段。如果FinalFieldExample的构造方法像这样：
```java
public FinalFieldExample() { // bad!
  x = 3;
  y = 4;
  // bad construction - allowing this to escape
  global.obj = this;
}
```
那么，从global.obj中读取this的引用线程不会保证读取到的x的值为3。
能够看到字段的正确的构造值固然不错，但是，如果字段本身就是一个引用，那么，你还是希望你的代码能够看到引用所指向的这个对象（或者数组）的最新值。如果你的字段是final字段，那么这是能够保证的。因此，当一个final指针指向一个数组，你不需要担心线程能够看到引用的最新值却看不到引用所指向的数组的最新值。重复一下，这儿的“正确的”的意思是“对象构造方法结尾的最新的值”而不是“最新可用的值”。
现在，在讲了如上的这段之后，如果在一个线程构造了一个不可变对象之后（对象仅包含final字段），你希望保证这个对象被其他线程正确的查看，你仍然需要使用同步才行。例如，没有其他的方式可以保证不可变对象的引用将被第二个线程看到。使用final字段的程序应该仔细的调试，这需要深入而且仔细的理解并发在你的代码中是如何被管理的。
如果你使用JNI来改变你的final字段，这方面的行为是没有定义的。
## 原文
## How do final fields work under the new JMM?
The values for an object’s final fields are set in its constructor. Assuming the object is constructed “correctly”, once an object is constructed, the values assigned to the final fields in the constructor will be visible to all other threads without synchronization. In addition, the visible values for any other object or array referenced by those final fields will be at least as up-to-date as the final fields.
What does it mean for an object to be properly constructed? It simply means that no reference to the object being constructed is allowed to “escape” during construction. (See [Safe Construction Techniques](http://www-106.ibm.com/developerworks/java/library/j-jtp0618.html) for examples.) In other words, do not place a reference to the object being constructed anywhere where another thread might be able to see it; do not assign it to a static field, do not register it as a listener with any other object, and so on. These tasks should be done after the constructor completes, not in the constructor.
```java
class FinalFieldExample {
  final int x;
  int y;
  static FinalFieldExample f;
  public FinalFieldExample() {
    x = 3;
    y = 4;
  }

  static void writer() {
    f = new FinalFieldExample();
  }

  static void reader() {
    if (f != null) {
      int i = f.x;
      int j = f.y;
    }
  }
}
```
The class above is an example of how final fields should be used. A thread executing `reader` is guaranteed to see the value 3 for `f.x`, because it is final. It is not guaranteed to see the value 4 for `y`, because it is not final. If `FinalFieldExample`‘s constructor looked like this:
```java
public FinalFieldExample() { // bad!
  x = 3;
  y = 4;
  // bad construction - allowing this to escape
  global.obj = this;
}
```
then threads that read the reference to `this`from `global.obj` are **not** guaranteed to see 3 for `x`.
The ability to see the correctly constructed value for the field is nice, but if the field itself is a reference, then you also want your code to see the up to date values for the object (or array) to which it points. If your field is a final field, this is also guaranteed. So, you can have a final pointer to an array and not have to worry about other threads seeing the correct values for the array reference, but incorrect values for the contents of the array. Again, by “correct” here, we mean “up to date as of the end of the object’s constructor”, not “the latest value available”.
Now, having said all of this, if, after a thread constructs an immutable object (that is, an object that only contains final fields), you want to ensure that it is seen correctly by all of the other thread, you **still** typically need to use synchronization. There is no other way to ensure, for example, that the reference to the immutable object will be seen by the second thread. The guarantees the program gets from final fields should be carefully tempered with a deep and careful understanding of how concurrency is managed in your code.
There is no defined behavior if you want to use JNI to change final fields.
