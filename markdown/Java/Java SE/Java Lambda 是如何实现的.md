# Java Lambda 是如何实现的

> [引用 -> Java Lambda表达式 实现原理分析](https://blog.csdn.net/jiankunking/article/details/79825928)
[引用 -> java8 探讨与分析匿名内部类、lambda表达式、方法引用的底层实现](https://segmentfault.com/a/1190000018607400)

## 如何使用函数式编程

-  定义函数式接口 
```java
@FunctionalInterface
public interface Flyable {
    void fly();
}
```

-  Demo 
```java
public class LambdaTest {
    public static void main(String[] args) {
        Flyable flyable = () -> System.out.println("飞起来了");
        flyable.fly();
    }
}
```

-  编译 
```shell
javac -encoding utf-8 LambdaTest.java
```
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240309002115.png#id=CjtSw&originHeight=69&originWidth=250&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

-  运行 
```shell
java LambdaTest
```
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240309002147.png#id=fNJeZ&originHeight=39&originWidth=189&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## Lambda 实现方式
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240309002227.png#id=PDsKs&originHeight=99&originWidth=277&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

- 反编译 : Flyable.class
```shell
javap -p Flyable.class
```

- 反编译 : LambdaTest.class
```shell
javap -p LambdaTest.class
```
![](https://gitee.com/eden2f/ImageHosting/raw/master/imgs/20210418231230.png#id=DhOAu&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240309002259.png#id=qjebt&originHeight=136&originWidth=400&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
由此可见 : 自动生成了 private static void lambda$main$0();

-  查看反编译细节 : LambdaTest.class 
```shell
javap -p -v LambdaTest.class
```
```shell
Classfile LambdaTest.class
  Last modified 2021-1-23; size 998 bytes
  MD5 checksum 49cbf38c127aef6ab71674fd641638a2
  Compiled from "LambdaTest.java"
public class LambdaTest
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #8.#18         // java/lang/Object."<init>":()V
   #2 = InvokeDynamic      #0:#23         // #0:fly:()LFlyable;
   #3 = InterfaceMethodref #24.#25        // Flyable.fly:()V
   #4 = Fieldref           #26.#27        // java/lang/System.out:Ljava/io/PrintStream;
   #5 = String             #28            // 飞起来了
   #6 = Methodref          #29.#30        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #7 = Class              #31            // LambdaTest
   #8 = Class              #32            // java/lang/Object
   #9 = Utf8               <init>
  #10 = Utf8               ()V
  #11 = Utf8               Code
  #12 = Utf8               LineNumberTable
  #13 = Utf8               main
  #14 = Utf8               ([Ljava/lang/String;)V
  #15 = Utf8               lambda$main$0
  #16 = Utf8               SourceFile
  #17 = Utf8               LambdaTest.java
  #18 = NameAndType        #9:#10         // "<init>":()V
  #19 = Utf8               BootstrapMethods
  #20 = MethodHandle       #6:#33         // invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/
invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #21 = MethodType         #10            //  ()V
  #22 = MethodHandle       #6:#34         // invokestatic LambdaTest.lambda$main$0:()V
  #23 = NameAndType        #35:#36        // fly:()LFlyable;
  #24 = Class              #37            // Flyable
  #25 = NameAndType        #35:#10        // fly:()V
  #26 = Class              #38            // java/lang/System
  #27 = NameAndType        #39:#40        // out:Ljava/io/PrintStream;
  #28 = Utf8               飞起来了
  #29 = Class              #41            // java/io/PrintStream
  #30 = NameAndType        #42:#43        // println:(Ljava/lang/String;)V
  #31 = Utf8               LambdaTest
  #32 = Utf8               java/lang/Object
  #33 = Methodref          #44.#45        // java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/Method
Handle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #34 = Methodref          #7.#46         // LambdaTest.lambda$main$0:()V
  #35 = Utf8               fly
  #36 = Utf8               ()LFlyable;
  #37 = Utf8               Flyable
  #38 = Utf8               java/lang/System
  #39 = Utf8               out
  #40 = Utf8               Ljava/io/PrintStream;
  #41 = Utf8               java/io/PrintStream
  #42 = Utf8               println
  #43 = Utf8               (Ljava/lang/String;)V
  #44 = Class              #47            // java/lang/invoke/LambdaMetafactory
  #45 = NameAndType        #48:#52        // metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType
;)Ljava/lang/invoke/CallSite;
  #46 = NameAndType        #15:#10        // lambda$main$0:()V
  #47 = Utf8               java/lang/invoke/LambdaMetafactory
  #48 = Utf8               metafactory
  #49 = Class              #54            // java/lang/invoke/MethodHandles$Lookup
  #50 = Utf8               Lookup
  #51 = Utf8               InnerClasses
  #52 = Utf8               (Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #53 = Class              #55            // java/lang/invoke/MethodHandles
  #54 = Utf8               java/lang/invoke/MethodHandles$Lookup
  #55 = Utf8               java/lang/invoke/MethodHandles
{
  public LambdaTest();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 6: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=2, args_size=1
         0: invokedynamic #2,  0              // InvokeDynamic #0:fly:()LFlyable;
         5: astore_1
         6: aload_1
         7: invokeinterface #3,  1            // InterfaceMethod Flyable.fly:()V
        12: return
      LineNumberTable:
        line 9: 0
        line 10: 6
        line 11: 12

  private static void lambda$main$0();
    descriptor: ()V
    flags: ACC_PRIVATE, ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=2, locals=0, args_size=0
         0: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #5                  // String 飞起来了
         5: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 9: 0
}
SourceFile: "LambdaTest.java"
InnerClasses:
     public static final #50= #49 of #53; //Lookup=class java/lang/invoke/MethodHandles$Lookup of class java/lang/invoke/MethodHandles
BootstrapMethods:
  0: #20 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invok
e/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #21 ()V
      #22 invokestatic LambdaTest.lambda$main$0:()V
      #21 ()V
```

-  将生成的内部类class保存下来 
```shell
java -Djdk.internal.lambda.dumpProxyClasses LambdaTest
```
```java
import java.lang.invoke.LambdaForm.Hidden;

// $FF: synthetic class
final class LambdaTest$$Lambda$1 implements Flyable {
    private LambdaTest$$Lambda$1() {
    }

    @Hidden
    public void fly() {
        LambdaTest.lambda$main$0();
    }
}
```

-  小结 
   1. 在类编译时，会生成一个私有静态方法+一个内部类；
   2. 在内部类中实现了函数式接口，在实现接口的方法中，会调用编译器生成的静态方法；
   3. 在使用lambda表达式的地方，通过传递内部类实例，来调用函数式接口方法。
## 匿名内部类 实现方式

- 实现匿名内部类
```java
public class LambdaTest {
    public static void main(String[] args) {
        Flyable flyable = new Flyable() {
            @Override
            public void fly() {
                System.out.println("飞起来了");
            }
        };
        flyable.fly();
    }
}
```

-  编译 LambdaTest 

可以看到,在编译期就生成了匿名内部类的class文件
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240309002435.png#id=Sc1Uh&originHeight=89&originWidth=254&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

-  反编译匿名内部类的class文件 
```shell
javap -p LambdaTest$1.class
```
```shell
Compiled from "LambdaTest.java"
final class LambdaTest$1 implements Flyable {
  LambdaTest$1();
  public void fly();
}
```
## 两种实现方式的总结
| 方式 | javac编译 | javap反编译 | jvm调参并第二次编译 (运行) |
| --- | --- | --- | --- |
| 匿名内部类 | 额外生成class | 未见`invoke dynamic`
指令 | 无变化 |
| lambda表达式 | 未生成class，但额外生成了一个static的方法 | 发现`invoke dynamic` | 发现额外的class |

## 对于lambda表达式，为什么java8要这样做？
下面的译本，原文[Java-8-Lambdas-A-Peek-Under-the-Hood](https://www.infoq.com/articles/Java-8-Lambdas-A-Peek-Under-the-Hood)
匿名内部类具有可能影响应用程序性能的不受欢迎的特性。

1. 编译器为每个匿名内部类生成一个新的类文件。生成许多类文件是不可取的，因为每个类文件在使用之前都需要加载和验证，这会影响应用程序的启动性能。加载可能是一个昂贵的操作，包括磁盘I/O和解压缩JAR文件本身。
2. 如果lambdas被转换为匿名内部类，那么每个lambda都有一个新的类文件。由于每个匿名内部类都将被加载，它将占用JVM的元空间(这是Java 8对永久生成的替代)。如果JVM将每个此类匿名内部类中的代码编译为机器码，那么它将存储在代码缓存中。此外，这些匿名内部类将被实例化为单独的对象。因此，匿名内部类会增加应用程序的内存消耗。为了减少所有这些内存开销，引入一种缓存机制可能是有帮助的，这将促使引入某种抽象层。
3. 最重要的是，从第一天开始就选择使用匿名内部类来实现lambdas，这将限制未来lambda实现更改的范围，以及它们根据未来JVM改进而演进的能力。
4. 将lambda表达式转换为匿名内部类将限制未来可能的优化(例如缓存)，因为它们将绑定到匿名内部类字节码生成机制。

基于以上4点，lambda表达式的实现不能直接在编译阶段就用匿名内部类实现
，而是需要一个稳定的二进制表示，它提供足够的信息，同时允许JVM在未来采用其他可能的实现策略。
解决上述解释的问题，Java语言和JVM工程师决定将翻译策略的选择推迟到运行时。Java 7 中引入的新的 `invokedynamic` 字节码指令为他们提供了一种高效实现这一目标的机制。将lambda表达式转换为字节码需要两个步骤:

1. 生成 `invokedynamic` 调用站点 ( 称为lambda工厂 )，当调用该站点时，返回一个函数接口实例，lambda将被转换到该接口;
2. 将lambda表达式的主体转换为将通过invokedynamic指令调用的方法。
## 课外拓展

- 试试运行这个 LambdaTest
```java
import java.util.Random;

public class LambdaTest {

    public static void printString(String s, Integer i, Print<String, Integer> print) {
        print.print(s, i);
    }

    public static void printCount(Count<String> count) {
        String sss = count.count();
        System.out.println("sss = " + sss);
    }

    public static void main(String[] args) {
        printString("testPrint1", 1, (s, ii) -> System.out.println(s + ii));
        printString("testPrint2", 2, (s, ii) -> System.out.println(s + ii));
        printString("testPrint3", 3, (s, ii) -> System.out.println(s + ii));
        printString("testPrint4", 4, (s, ii) -> System.out.println(s + ii));
        printString("testPrint5", 5, (s, ii) -> System.out.println(s + ii));
        printString("testPrint6", 6, (s, ii) -> System.out.println(s + ii));
        printString("testPrint7", 7, (s, ii) -> System.out.println(s + ii));
        printString("testPrint8", 8, (s, ii) -> System.out.println(s + ii));
        printString("testPrint9", 9, (s, ii) -> System.out.println(s + ii));
        printString("testPrint10", 10, (s, ii) -> System.out.println(s + ii));
        printString("testPrint11", 11, (s, ii) -> System.out.println(s + ii));
        printString("testPrint12", 12, (s, ii) -> System.out.println(s + ii));
        printString("testPrint13", 13, (s, ii) -> System.out.println(s + ii));
        printString("testPrint14", 14, (s, ii) -> System.out.println(s + ii));
        printString("testPrint15", 15, (s, ii) -> System.out.println(s + ii));
        printString("testPrint16", 16, (s, ii) -> System.out.println(s + ii));

        printCount(() -> "testCount" + 1);
        Random random = new Random();
        printCount(() -> String.valueOf(random.nextInt(1000000)));
        printCount(() -> String.valueOf(random.nextInt(1000000)));
        printCount(() -> String.valueOf(random.nextInt(1000000)));
        printCount(() -> String.valueOf(random.nextInt(1000000)));
        printCount(() -> String.valueOf(random.nextInt(1000000)));

        Print<String, Integer> print_2_1 = new Print<String, Integer>() {
            @Override
            public void print(String x, Integer o) {
                System.out.println(x + o);
            }
        };
        print_2_1.print("print_2_1", 20);


        Count<String> count_2_1 = new Count<String>() {
            @Override
            public String count() {
                return String.valueOf(random.nextInt(1000000));
            }
        };
        count_2_1.count();

    }

    @FunctionalInterface
    interface Print<T, E> {
        void print(T x, E e);
    }

    @FunctionalInterface
    interface Count<T> {
        T count();
    }
}
```

- 编译 & 运行
```shell
# 编译
javac -encoding utf-8 LambdaTest.java
# 运行
java -Djdk.internal.lambda.dumpProxyClasses LambdaTest
```
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240309002604.png#id=XKZfh&originHeight=620&originWidth=337&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://gitee.com/eden2f/ImageHosting/raw/master/imgs/20210418231325.png#id=ZpLmC&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
