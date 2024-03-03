# Spring Bean的生命周期

## Bean的生命周期

过程 : bean创建 -> bean初始化 -> 销毁

### 对象的构造

单例模式：在容器启动的时候创建对象；

多例模式：在每次获取的时候创建对象；

1. 初始化方法
   1. 对象创建完成，并赋值好的时候调用初始化方法。
2. 销毁方法
   1. 单例模式：容器关闭的时候销毁
   2. 多例模式：容器不会管理这个bean；容器不会调用销毁方法；

### 指定对象的初始化和销毁方法

1. 通过@Bean指定 init-method和destroy-method
2. 通过让Bean实现InitializingBean、DisposableBean接口；
3. 使用注解@PostConstruct：在bean创建完成并且属性赋值完成时执行；@PreDestroy：在容器销毁bean之前执行；
4. BeanPostProcessor : bean的后置处理器,在bean初始化前后执行处理工作；