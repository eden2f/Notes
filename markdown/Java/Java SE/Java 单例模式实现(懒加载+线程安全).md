# Java 单例模式实现(懒加载+线程安全)

## 双检锁
```java
public class DoubleCheckLockingSingletonDemo {

    private DoubleCheckLockingSingletonDemo() {
    }

    private static volatile DoubleCheckLockingSingletonDemo instance;

    public static DoubleCheckLockingSingletonDemo getInstance() {
        if (null == instance) {
            synchronized (DoubleCheckLockingSingletonDemo.class) {
                if (null == instance) {
                    instance = new DoubleCheckLockingSingletonDemo();
                }
            }
        }
        return instance;
    }

}
```
## 枚举
```java
public enum EnumSingletonDemo {

    INSTANCE;

    EnumSingletonDemo() {
    }

    public void whateverMethod() {
    }
}
```
## 静态内部类
```java
public class InternalStaticClassSingletonDemo {

    private InternalStaticClassSingletonDemo() {
    }

    static class InternalStaticClass {
        public static volatile InternalStaticClassSingletonDemo instance = new InternalStaticClassSingletonDemo();
    }

    public static InternalStaticClassSingletonDemo getInstance() {
        return InternalStaticClass.instance;
    }

}
```
## 同步方法
```java
public class SynchronizedFuncSingletonDemo {

    private SynchronizedFuncSingletonDemo() {
    }

    private static SynchronizedFuncSingletonDemo instance;

    public synchronized SynchronizedFuncSingletonDemo getInstance1() {
        if (null == instance) {
            instance = new SynchronizedFuncSingletonDemo();
        }
        return instance;
    }

}
```
## 问题

- 为什么需要 volatile ? 
   - volatile可以禁止指令重排序
- 为什么需要禁止指令重排序? 
   - 因为new并不是一个原子操作, new可以简单分成三个操作: 
      - 分配一块内存
      - 将内存的地址赋值给变脸
      - 在内存上初始化对象
   - 所以不加volatile可能会抛出空指针
