# JVM 类加载器

> 《深入理解Java虚拟机》- 第三版
《深入浅出Java虚拟机》- 李国

## 前言
Java 虚拟机设计团队有意把类加载阶段中的 “通过一个类的全限定名来获取描述该类的二进制字节流” 这个动作放到 Java 虚拟机外部去实现，以便让应用程序自己决定如何去获取所需的类。 实现这个动作的代码被称为 “类加载器”（ Class Loader）。
## 类与类加载器
类加载器虽然只用于实现类的加载动作，但它在 Java 程序中起到的作用却远超类加载阶段。对于任意一个类，都必须由加载它的类加载器和这个类本身一起共同确立其在 Java 虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间。这句话可以表达得更通俗一些：比较两个类是否 “相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个 Class 文件， 被同一个 Java 虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等。这里所指的 “相等”，包括代表类的 Class 对象 的 equals() 方法、 isAssignableFrom() 方法、isInstance() 方法的返回结果，也包括使用 instanceof 关键字做对象所属关系判定等各种情况。如果没有注意到类加载器的影响，在某些情况下可能会产生具有迷惑性的结果。
## 双亲委派模型
站在 Java 虚拟机的角度来看，只存在两种不同的类加载器：一种是启动类加载器（ Bootstrap ClassLoader），这个类加载器使用 C++ 语言实现，是虚拟机自身的一部分；另外一种就是其他所有的类加载器，这些类加载器都由 Java 语言实现，独立存在于虚拟机外部，并且全都继承自抽象类 java.lang.ClassLoader。站在 Java 开发人员的角度来看，类加载器就应当划分得更细致一些。自 JDK 1. 2 以来，Java 一直保持着三层类加载器、双亲委派的类加载架构，尽管这套架构在 Java 模块化系统出现后有了一些调整变动，但依然未改变其主体结构，我们将 在 7. 5 节中专门讨论模块化系统下的类加载器。本节内容将针对 JDK 8 及之前版本的 Java 来介绍什么是三层类加载器，以及什么是双亲委派模型。对于这个时期的 Java 应用，绝大多数 Java 程序都会使用到以下 3 个系统提供的类加载器来进行加载。
双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应有自己的父类加载器。不过这里类加载器之间的父子关系一般不是以继承（Inheritance）的关系来实现的，而是通常使用组合（Composition）关系来复用父加载器的代码。
强调一下 “通常” 一词, 类加载器的双亲委派模型在 JDK 1. 2 时期被引入，并被广泛应用于此后几乎所有的 Java 程序中，但它并不是一个具有强制性约束力的模型，而是 Java 设计者们推荐给开发者的一种类加载器实现的最佳实践。
双亲委派模型的工作过程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到最顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去完成加载。
![](https://gitee.com/eden2f/ImageHosting/raw/master/imgs/20210429161833.jpg#id=MC8k0&originHeight=397&originWidth=797&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 双亲委派模型的好处
使用双亲委派模型来组织类加载器之间的关系，一个显而易见的好处就是 Java 中的类随着它的类加载器一起具备了一种带有优先级的层次关系。例如类 java.lang.Object，它存放在 rt.jar 之中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的启动类加载器进行加载，因此 Object 类在程序的各种类加载器环境中都能够保证是同一个类。反之，如果没有使用双亲委派模型，都由各个类加载器自行去加载的话，如果用户自己也编写了一个名为 java.lang.Object 的类，并放在程序的 ClassPath 中，那系统中就会出现多个不同的 Object 类，Java 类型体系中最基础的行为也就无从 保证，应用程序将会变得一片混乱。如果读者有兴趣的话，可以尝试去写一个与 rt.jar 类库中已有类重名的 Java 类，将会发现它可以正常编译，但永远无法被加载运行。
## 破坏双亲委派模型
上文提到过双亲委派模型并不是一个具有强制性约束的模型，而是 Java 设计者推荐给开发者们的类加载器实现方式。在 Java 的世界中大部分的类加载器都遵循这个模型，但也有例外的情况，直到 Java 模块化出现为止，双亲委派模型主要出现过 3 次较大规模 “被破坏” 的情况。
双亲委派模型的第一次 “被破坏” 其实发生在双亲委派模型出现之前——即 JDK 1.2 面世以前的 “远古” 时代。由于双亲委派模型在 JDK 1.2 之后才被引入，但是类加载器的概念和抽象类 java.lang.ClassLoader 则在 Java 的第一个版本中就已经 存在，面对已经存在的用户自定义类加载器的代码，Java 设计者们引入双亲委派模型时不得不做出一些妥协，为了兼容这些已有代码，无法再以技术手段避免 loadClass() 被子类覆盖的可能性，只能在 JDK 1.2 之后的 java.lang.ClassLoader 中添加一个新的 protected 方法 findClass()，并引导用户编写的类加载逻辑时尽可能去重写这个方法，而不是在 loadClass() 中编写代码。上节我们已经分析过 loadClass() 方法，双亲委派的具体逻辑就实现在这里面，按照 loadClass() 方法的逻辑，如果父类加载失败，会自动调用自己的 findClass() 方法来完成加载，这样既不影响用户按照自己的意愿去加载类，又可以保证新写出来的类加载器是符合双亲委派规则的。
双亲委派模型的第二次 “被破坏” 是由这个模型自身的缺陷导致的，双亲委派很好地解决了各个类加载器协作时基础类型的 一致性问题（越基础的类由越上层的加载器进行加载），基础类型之所以被称为 “基础”，是因为它们总是作为被用户代码继承、调用的 API 存在，但程序设计往往没有绝对不变的完美规则，如果有基础类型又要调用回用户的代码，那该怎么办 呢？
这并非是不可能出现的事情，一个典型的例子便是 JNDI 服务，JNDI 现在已经是 Java 的标准服务，它的代码由启动类加载器来完成加载（ 在 JDK 1.3 时加入到 rt.jar 的），肯定属于 Java 中很基础的类型了。但 JNDI 存在的目的就是对资源 进行查找和集中管理，它需要调用由其他厂商实现并部署在应用程序的 ClassPath 下 的 JNDI 服务提供者接口（Service Provider Interface，SPI）的代码，现在问题来了，启动类加载器是绝不可能认识、加载这些代码的，那该怎么办？
为了解决这个困境，Java 的设计团队只好引入了一个不太优雅的设计：线程上下文类加载器（ Thread Context ClassLoader）。这个类加载器可以通过 java.lang.Thread 类的 setContextClassLoader() 方法进行设置，如果创建线程时 还未设置，它将会从父线程中继承一个，如果在应用程序的全局范围内都没有设置过的话，那这个类加载器默认就是应用 程序类加载器。
有了线程上下文类加载器，程序就可以做 一些 “舞弊” 的事情了。 JNDI 服务使用这个线程上下文类加载器去加载所需的 SPI 服务代码，这是一种父类加载器去请求子类加载器完成类加载的行为，这种行为实际上是打通了双亲委派模型的层次 结构来逆向使用类加载器，已经违背了双亲委派模型的一般性原则，但也是无可奈何的事情。Java 中涉及 SPI 的加载基本上都采用这种方式来完成，例如 JNDI、JDBC、JCE、JAXB 和 JBI 等。不过，当 SPI 的服务提供者多于一个的时候， 代码就只能根据具体提供者的类型来硬编码判断，为了消除这种极不优雅的实现方式，在 JDK 6 时，JDK 提供了 java.util.ServiceLoader 类，以 META-INF/services 中的配置信息，辅以责任链模式，这才算是给 SPI 的加载提供了一种相对合理的解决方案。
双亲委派模型的第三次 “被破坏” 是由于用户对程序动态性的追求而导致的，这里所说的 “ 动态性” 指的是一些非常 “热” 门的名词：代码热替换（Hot Swap）、模块热部署（Hot Deployment）等。说白 了就是希望 Java 应用程序能像我们的电脑外设那样，接上鼠标、U 盘，不用重启机器就能立即使用，鼠标有问题或要升级就换个鼠标，不用关机也不用重启。对于个人电脑来说，重启一次其实没有什么大不了的，但对于一些生产系统来说，关机重启一次可能就要被列为生产事故，这种情况下热部署就对软件开发者，尤其是大型系统或企业级软件开发者具有很大的吸引力。
## 一个 SPI 打破双亲委派模型的例子
Java 中有一个 SPI 机制，全称是 Service Provider Interface，是 Java 提供的一套用来被第三方实现或者扩展的 API，它可以用来启用框架扩展和替换组件。拿我们常用的数据库驱动加载做个引子, 在使用 JDBC 写程序之前，通常会调用`Class.forName("com.mysql.jdbc.Driver")`，用于加载所需要的驱动类。
在新版本中, 即使删除了 Class.forName 这一行代码，也能加载到正确的驱动类, 因为 `mysql-connector-java-8.0.15.jar!/META-INF/services/java.sql.Driver`  的内容是 `com.mysql.cj.jdbc.Driver` .
通过在 META-INF/services 目录下，创建一个以接口全限定名为命名的文件（内容为实现类的全限定名），即可自动加载这一种实现，这就是 SPI。
![](https://gitee.com/eden2f/ImageHosting/raw/master/imgs/20210429165318.jpg#id=JU8Xd&originHeight=268&originWidth=756&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
SPI 实际上是“基于接口的编程＋策略模式＋配置文件”组合实现的动态加载机制，主要使用 java.util.ServiceLoader 类进行动态装载。
java.sql.DriverManager 类和 java.util.ServiceLoader 类都是属于 rt.jar 。它们的类加载器是 Bootstrap ClassLoader，也就是最上层的那个。而具体的数据库驱动，却属于业务代码，这个启动类加载器是无法加载的。现在当前类加载器没有能力去做这件事情，怎么办？
```java
//part1:DriverManager::static
/**
 * Load the initial JDBC drivers by checking the System property
 * jdbc.properties and then use the {@code ServiceLoader} mechanism
 */
static {
    loadInitialDrivers();
    println("JDBC DriverManager initialized");
}
//part2:DriverManager::loadInitialDrivers
//jdk1.8 之后，变成了lazy的ensureDriversInitialized
ServiceLoader <Driver> loadedDrivers = ServiceLoader.load(Driver.class);
Iterator<Driver> driversIterator = loadedDrivers.iterator();

//part3:ServiceLoader::load
public static <T> ServiceLoader<T> load(Class<T> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}
```
把当前的类加载器，设置成了线程的上下文类加载器。那么，对于一个刚刚启动的应用程序来说，它当前的加载器就是启动 main 方法的那个加载器，到底是哪一个？ jre 中用于启动入口函数 main 的类是 Launcher . 从 Launcher  中得知 main线程上下文的类加载器，是应用程序类加载器。使用它来加载第三方驱动，是没有什么问题的。
```java
public Launcher() {
    Launcher.ExtClassLoader var1;
    try {
        var1 = Launcher.ExtClassLoader.getExtClassLoader();
    } catch (IOException var10) {
        throw new InternalError("Could not create extension class loader", var10);
    }
    try {
        this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
    } catch (IOException var9) {
        throw new InternalError("Could not create application class loader", var9);
    }
    Thread.currentThread().setContextClassLoader(this.loader);
   	// 删除无关的代码
    }
}
```
