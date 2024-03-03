# Spring 事件驱动 ApplicationListener的实现分析

> [wolf_lb博客原文链接 : ApplicationListener原理分析](https://www.cnblogs.com/wolf-bin/p/11604227.html)

## 观察者模式
观察者模式定义：对象间一种一对多的依赖关系，当一个被观察的对象改变状态，则会自动通知它的依赖对象。观察者模式属于行为型模式。这是比较概念性的定义，下面我用一种接近生活的例子来诠释观察者模式。上大学的时候，很多学生经常旷课，但是快到期末考试那两三节课基本是全到的，为什么呢？不错，老师会划考试重点！！！这时老师就是被学生观察的对象，学生就是观察者。当老师说以下这个知识点考试会考时，下面刷刷刷响起来，同学们都在用笔画标记！当然不同的学生用的办法不一样，比如学霸会用五颜六色的表把重中之重区分出来，学渣可能就不管全用2B铅笔画（我就是这学渣中的其一）。看一下老师和学生的关系图：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240304005327.png#id=jbHRB&originHeight=371&originWidth=489&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
郑重声明一下，本学渣比较懒，没有自己实现一个观察者模式，直接用的是jdk提供的Observer和Obervable。接下来直接用代码说明一切。
### 被观察对象
```java
/**
 * 被观察对象
 * @author Greiz
 */
public class TeacherObservable extends Observable {
    private String examKeyPoints;
    public String getExamKeyPoints() {
        return examKeyPoints;
    }
    public void setExamKeyPoints(String examKeyPoints) {
        this.examKeyPoints = examKeyPoints;
        // 修改状态
        super.setChanged();
        // 通知所有观察者
        super.notifyObservers(examKeyPoints);
    }
}
```
被观察对象（老师），继承了Observable。当变量考试重点（examKeyPoints）变了通知观察者
```java
public class Observable {
    private boolean changed = false;
    private Vector<Observer> obs;
    public synchronized void addObserver(Observer o) {
        if (o == null)
            throw new NullPointerException();
        if (!obs.contains(o)) {
            obs.addElement(o);
        }
    }
    public void notifyObservers() {
        notifyObservers(null);
    }
    public void notifyObservers(Object arg) {
        Object[] arrLocal;
        synchronized (this) {
            if (!changed)
                return;
            arrLocal = obs.toArray();
            clearChanged();
        }
        for (int i = arrLocal.length-1; i>=0; i--)
            ((Observer)arrLocal[i]).update(this, arg);
    }
    protected synchronized void setChanged() {
        changed = true;
    }  
}
```
JDK中的Observable类。成员变量维护一个观察者的列表，通知的时候遍历该列表逐个调用update()方法。哈哈，是不是很简单。
### 观察者
```java
public interface Observer {
    void update(Observable o, Object arg);
}
```
监听者用的也是JDK的，这家伙比我还懒，就一个方法的接口，要干的活都交给后代处理。
```java
/**
 * @author Greiz
 */
public class ExcellentStudentObserver implements Observer {
    public static final String STUDENT = "我学霸";
    @Override
    public void update(Observable o, Object arg) {
        System.out.println(STUDENT + "用各种颜色的笔划重点:" + arg.toString());
    }
}
```
监听者 -- 学霸，当TeacherObservable成员变量改变时最终会调该类的update()方法。
```java
/**
 * @author Greiz
 */
public class PoorStudentObserver implements Observer {
    public static final String STUDENT = "我学渣一枚";
    @Override
    public void update(Observable o, Object arg) {
        System.out.println(STUDENT + "用2B铅笔划重点:" + arg.toString());
    }
}
```
监听者 -- 学渣（我），当TeacherObservable成员变量改变时最终会调该类的update()方法。
### 管理者
老师和学生都有了，剩下的就差把他们联系起来了，总不能随手一把抓吧，专业不对口划重点也没有啊！
```java
/**
 * @author Greiz
 */
public class ObserverManager {
    public static void main(String[] args) {
        TeacherObservable observable = new TeacherObservable();
        // 给被观察者对象添加观察者
        observable.addObserver(new PoorStudentObserver());
        observable.addObserver(new ExcellentStudentObserver());
        // 修改被观察者
        observable.setExamKeyPoints("这是考试重点！！！");
    }
}
```
一个简单的观察者模式完整的列子完成了。
### 小结
优点

1.  被观察对象和观察者之间解耦。 
2.  建立回调机制模型。 

缺点

1. 如果被观察对象维护的观察者列表中成员过多，遍历通知会耗时相当长。
2. 如果被观察对象和观察者之间出现相互调用容易形成死循环。
3. 观察者不清楚被观察者对象变化的细节
4. 只能本地，不能分布式。
## ApplicationListener 源码解析
ApplicationListener 跟上面观察者模式有什么关系呢？我们先看源码，后面一起分析一下他们的关系。这节分两个阶段，一个调用阶段，另一个组装阶段。
### 调用阶段
下面我画出调用过程一些重要接口调用时序图。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240304005455.png#id=fxSzp&originHeight=411&originWidth=1240&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
源码解析调用阶段都是围绕这个图步骤进行。
```java
public class GreizEvent extends ApplicationEvent {
    public GreizEvent(Object source) {
        super(source);
    }
    private String name = "Greiz";
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
}
```
自定义事件，需要继承 ApplicationEvent。
```java
@Component
public class GreizListener implements ApplicationListener<GreizEvent> {
    @Override
    public void onApplicationEvent(GreizEvent event) {
        System.out.println("=============" + event.getName());
    }
}
```
添加自定义事件监听者，必须加入Spring容器管理相关注解如 @Component，否则不起作用。
```java
public static void main(String[] args) {
    ApplicationContext context = new AnnotationConfigApplicationContext("com.greiz.demo.listener");
    context.publishEvent(new GreizEvent("Greiz"));
}
```
启动Spring，发布自定义事件
接下来进入时序图中1-6 接口。
```java
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
   ApplicationEvent applicationEvent;
   if (event instanceof ApplicationEvent) {
      applicationEvent = (ApplicationEvent) event;
   }
   ... 省略代码
   if (this.earlyApplicationEvents != null) {
      this.earlyApplicationEvents.add(applicationEvent);
   }
   else {
      getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
   }
   ... 省略代码
}
```
对应时序图方法1， AbstractApplicationContext.publishEvent()。publishEvent方法是在 ApplicationEventPublisher 定义的，ApplicationEventPublisher 可以理解成事件发射器。会调用 getApplicationEventMulticaster()
```java
ApplicationEventMulticaster getApplicationEventMulticaster() throws IllegalStateException {
   if (this.applicationEventMulticaster == null) {
      throw new IllegalStateException("ApplicationEventMulticaster not initialized - " +
            "call 'refresh' before multicasting events via the context: " + this);
   }
   return this.applicationEventMulticaster;
}
```
对应时序图方法2，AbstractApplicationContext.getApplicationEventMulticaster()获取事件广播者。applicationEventMulticaster 在Spring启动refresh过程 调用 initApplicationEventMulticaster() 初始化，是 SimpleApplicationEventMulticaster 实例。
```java
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
   ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
   Executor executor = getTaskExecutor();
   for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
      if (executor != null) {
         executor.execute(() -> invokeListener(listener, event));
      }
      else {
         invokeListener(listener, event);
      }
   }
}
```
对应时序图方法3，SimpleApplicationEventMulticaster.multicastEvent()。根据事件类型，获取所有对应的监听者，然后遍历通知（俗称广播）。
```java
protected Collection<ApplicationListener<?>> getApplicationListeners(
      ApplicationEvent event, ResolvableType eventType) {
   Object source = event.getSource();
   Class<?> sourceType = (source != null ? source.getClass() : null);
   ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);
   ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
   if (retriever != null) {
      return retriever.getApplicationListeners();
   }

   if (this.beanClassLoader == null ||
         (ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
               (sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
      synchronized (this.retrievalMutex) {
         retriever = this.retrieverCache.get(cacheKey);
         if (retriever != null) {
            return retriever.getApplicationListeners();
         }
         retriever = new ListenerRetriever(true);
         Collection<ApplicationListener<?>> listeners =
               retrieveApplicationListeners(eventType, sourceType, retriever);
         this.retrieverCache.put(cacheKey, retriever);
         return listeners;
      }
   }
   else {
      return retrieveApplicationListeners(eventType, sourceType, null);
   }
}
```
对应时序图方法4，AbstractApplicationEventMulticaster.getApplicationListeners()。根据事件类型，先查询缓存，如果缓存中没有调用 retrieveApplicationListeners() 获取，然后存到缓存中。
```java
private Collection<ApplicationListener<?>> retrieveApplicationListeners(
      ResolvableType eventType, @Nullable Class<?> sourceType, @Nullable ListenerRetriever retriever) {

   List<ApplicationListener<?>> allListeners = new ArrayList<>();
   Set<ApplicationListener<?>> listeners;
   Set<String> listenerBeans;
   synchronized (this.retrievalMutex) {
      listeners = new LinkedHashSet<>(this.defaultRetriever.applicationListeners);
      listenerBeans = new LinkedHashSet<>(this.defaultRetriever.applicationListenerBeans);
   }
   for (ApplicationListener<?> listener : listeners) {
      if (supportsEvent(listener, eventType, sourceType)) {
         if (retriever != null) {
            retriever.applicationListeners.add(listener);
         }
         allListeners.add(listener);
      }
   }
   if (!listenerBeans.isEmpty()) {
      BeanFactory beanFactory = getBeanFactory();
      for (String listenerBeanName : listenerBeans) {
         try {
            Class<?> listenerType = beanFactory.getType(listenerBeanName);
            if (listenerType == null || supportsEvent(listenerType, eventType)) {
               ApplicationListener<?> listener = beanFactory.getBean(listenerBeanName, ApplicationListener.class);
               if (!allListeners.contains(listener) && supportsEvent(listener, eventType, sourceType)) {
                  if (retriever != null) {
                     if (beanFactory.isSingleton(listenerBeanName)) {
                        retriever.applicationListeners.add(listener);
                     }
                     else {
                        retriever.applicationListenerBeans.add(listenerBeanName);
                     }
                  }
                  allListeners.add(listener);
               }
            }
         }
         catch (NoSuchBeanDefinitionException ex) {
         }
      }
   }
   AnnotationAwareOrderComparator.sort(allListeners);
   if (retriever != null && retriever.applicationListenerBeans.isEmpty()) {
      retriever.applicationListeners.clear();
      retriever.applicationListeners.addAll(allListeners);
   }
   return allListeners;
}
```
对应时序图方法5，AbstractApplicationEventMulticaster.retrieveApplicationListeners()。 所有事件监听者都在this.defaultRetriever 对象中，该对象的值初始化过程我们在下一节分析。返回过滤后符合本次事件的监听者。接下来我们回到 时序图方法3 中继续调用 SimpleApplicationEventMulticaster.invokeListener()。
```java
	private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
		try {
			listener.onApplicationEvent(event);
		}
		catch (ClassCastException ex) {
			... 省略代码
		}
	}
```
对应时序图方法6，SimpleApplicationEventMulticaster.doInvokeListener()。 这里就是真正调用监听者的方法。
前面提出 ApplicationListener 跟观察者模式有什么关系呢？分析 publishEvent(...) 到 onApplicationEvent(...) 调用，是不是很像前面观察者模式列子中 setExamKeyPoints() --> notifyObservers() --> update()。这一个阶段可以看作就是观察者模式中调用阶段。接下来我们继续分析观察者和被观察者对象绑定过程---组装阶段。
### 组装阶段
照旧，以图开篇，接下来全靠编！！！
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240304005631.png#id=HWIAC&originHeight=440&originWidth=1240&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在调用阶段时序图5中得知符合对应事件的监听者是从 AbstractApplicationEventMulticaster 成员 (ListenerRetriever)defaultRetriever 的 applicationListeners 和 applicationListenerBeans 属性获取的。组装阶段就是解析 defaultRetriever 初始化负值过程。
applicationListenerBeans 初始化负值过程：
```java
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      ... ... 省略代码
      try {
         ... ... 省略代码
         // ListenerRetriever applicationListenerBeans 初始化负值过程在这里面
         registerListeners();

         // ListenerRetriever applicationListeners 初始化负值过程在这里面
         finishBeanFactoryInitialization(beanFactory);
         ... ... 省略代码
      }

      catch (BeansException ex) {
      	... 省略代码
      }
      finally {
      	... 省略代码
      }
   }
}
```
对应时序图方法1，AbstractApplicationContext.refresh()。省略了一下与本次目的无关的代码。看注释就好，哈哈。
```java
protected void registerListeners() {
   // 初始化阶段getApplicationListeners()返回空列表
   for (ApplicationListener<?> listener : getApplicationListeners()) {
      getApplicationEventMulticaster().addApplicationListener(listener);
   }
	 // 根据bean类型获取
   String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
   for (String listenerBeanName : listenerBeanNames) {
      getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
   }
  ... 省略代码
}
```
对应时序图方法2，AbstractApplicationContext.registerListeners()。注意看注释，看注释，看注释！！！此处根据bean类型获取，反应了前面 “添加自定义事件监听者，必须加入Spring容器管理相关注解如 @Component，否则不起作用”的说法。
```java
public void addApplicationListenerBean(String listenerBeanName) {
   synchronized (this.retrievalMutex) {
      this.defaultRetriever.applicationListenerBeans.add(listenerBeanName);
      this.retrieverCache.clear();
   }
}
```
对应时序图方法3，AbstractApplicationEventMulticaster.addApplicationListenerBean()。
AbstractApplicationEventMulticaster 成员 (ListenerRetriever)defaultRetriever 的applicationListenerBeans 属性负值完成。
applicationListeners 初始化负值过程：
```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		... 省略代码
		beanFactory.preInstantiateSingletons();
	}
```
对应时序图方法4，AbstractApplicationContext.finishBeanFactoryInitialization()。
偷懒一次，跟着这个方法debug下去，最终调用AbstractApplicationContext.addApplicationListener()。
```java
public void addApplicationListener(ApplicationListener<?> listener) {
   Assert.notNull(listener, "ApplicationListener must not be null");
   if (this.applicationEventMulticaster != null) {
      this.applicationEventMulticaster.addApplicationListener(listener);
   }
   this.applicationListeners.add(listener);
}
```
对应时序图方法14，AbstractApplicationContext.addApplicationListener()。
```java
public void addApplicationListener(ApplicationListener<?> listener) {
   synchronized (this.retrievalMutex) {
      Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);
      if (singletonTarget instanceof ApplicationListener) {
         this.defaultRetriever.applicationListeners.remove(singletonTarget);
      }
      this.defaultRetriever.applicationListeners.add(listener);
      this.retrieverCache.clear();
   }
}
```
对应时序图方法15，AbstractApplicationEventMulticaster.addApplicationListener()。恩，是不是很熟悉的defaultRetriever。
AbstractApplicationEventMulticaster 成员 (ListenerRetriever)defaultRetriever 的applicationListeners 属性负值完成。
