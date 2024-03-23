# Java 内省(Introspector、PropertyDescriptor和MethodDescriptor)

> [Java 内省(Introspector)深入理解 : https://www.cnblogs.com/uu5666/p/8601983.html](https://www.cnblogs.com/uu5666/p/8601983.html)
> [java反射调用get/set方法，你还在拼接方法名吗？: https://www.cnblogs.com/1ning/p/10936129.html](https://www.cnblogs.com/1ning/p/10936129.html)

## 前言
最新工作中，遇到了通过反射调用get/set方法的地方，虽然反射的性能不是很好，但是相比较于硬编码的不易扩展，getDeclareFields可以拿到所有的成员变量，后续添加或删除成员变量时，不用修改代码，且应用次数只在修改数据时使用，故牺牲一些性能提高扩展性
## 传统的方式
见过很多人通过反射调用get/set方法都是通过获取属性的name，然后通过字符串截取将首字母大写，再拼上get/set来做
```java
String fieldName = field.getName();
String getMethodName = "get" + fieldName.substring(0, 1).toUpperCase() + fieldName.substring(1);
```
还有稍微好一点的同学，通过fieldName转成字符数组，首个字符-32来避免字符串截取的
```java
String fieldName = field.getName();
char[] chars = fieldName.toCharArray();
chars[0] = (char)(chars[0] - 32);
String getMethodName = "get" + new String(chars);
```
诚然，我觉得两种方式都可以，但是不知道有没有遇到过，生成的get/set方法并不是已get/set开头的，而是以is开头的，比如boolean类型的成员变量。这个时候我们就需要去判断属性的类型，然后用不同的前缀来拼接get/set方法名。其实，在jdk中已经包含了这样的工具类。
## Java 内省(Introspector)
一些概念：
内省(Introspector) 是Java 语言对 JavaBean 类属性、事件的一种缺省处理方法。
JavaBean是一种特殊的类，主要用于传递数据信息，这种类中的方法主要用于访问私有的字段，且方法名符合某种命名规则。如果在两个模块之间传递信息，可以将信息封装进JavaBean中，这种对象称为“值对象”(Value Object)，或“VO”。方法比较少。这些信息储存在类的私有变量中，通过set()、get()获得。
例如类UserInfo :
```java
package com.peidasoft.Introspector;
 
public class UserInfo {
   
  private long userId;
  private String userName;
  private int age;
  private String emailAddress;
   
  public long getUserId() {
    return userId;
  }
  public void setUserId(long userId) {
    this.userId = userId;
  }
  public String getUserName() {
    return userName;
  }
  public void setUserName(String userName) {
    this.userName = userName;
  }
  public int getAge() {
    return age;
  }
  public void setAge(int age) {
    this.age = age;
  }
  public String getEmailAddress() {
    return emailAddress;
  }
  public void setEmailAddress(String emailAddress) {
    this.emailAddress = emailAddress;
  }
   
}
```
在类UserInfo中有属性 userName, 那我们可以通过 getUserName,setUserName来得到其值或者设置新的值。通过 getUserName/setUserName来访问 userName属性，这就是默认的规则。 Java JDK中提供了一套 API 用来访问某个属性的 getter/setter 方法，这就是内省。
JDK内省类库：
PropertyDescriptor类:
PropertyDescriptor类表示JavaBean类通过存储器导出一个属性。主要方法：
1. getPropertyType()，获得属性的Class对象;
　　2. getReadMethod()，获得用于读取属性值的方法；getWriteMethod()，获得用于写入属性值的方法;
　　3. hashCode()，获取对象的哈希值;
　　4. setReadMethod(Method readMethod)，设置用于读取属性值的方法;
　　5. setWriteMethod(Method writeMethod)，设置用于写入属性值的方法。
实例代码如下：
```java
// 调用set方法
PropertyDescriptor propDesc=new PropertyDescriptor(userName,UserInfo.class);
Method methodSetUserName=propDesc.getWriteMethod();
methodSetUserName.invoke(userInfo, "wong");
System.out.println("set userName:"+userInfo.getUserName());
// 调用get方法
PropertyDescriptor proDescriptor =new PropertyDescriptor(userName,UserInfo.class);
Method methodGetUserName=proDescriptor.getReadMethod();
Object objUserName=methodGetUserName.invoke(userInfo);
System.out.println("get userName:"+objUserName.toString());
```
Introspector类:
将JavaBean中的属性封装起来进行操作。在程序把一个类当做JavaBean来看，就是调用Introspector.getBeanInfo()方法，得到的BeanInfo对象封装了把这个类当做JavaBean看的结果信息，即属性的信息。
getPropertyDescriptors()，获得属性的描述，可以采用遍历BeanInfo的方法，来查找、设置类的属性。
```
	getMethodDescriptors() ，获得方法的元信息，比如方法名，参数个数，参数字段类型等。
```
具体代码如下：
```java
BeanInfo beanInfo=Introspector.getBeanInfo(UserInfo.class);
PropertyDescriptor[] propertyDescriptors=beanInfo.getPropertyDescriptors();
MethodDescriptor[] methodDescriptors=beanInfo.getMethodDescriptors();
```
## 使用Introspector 、PropertyDescriptor获取Getter方法
```java
BeanInfo beanInfo;
try {
    beanInfo = Introspector.getBeanInfo(template.getClass());
} catch (IntrospectionException e) {
    log.info("xxxxxxxxxxxxxxxx", e);
    return null;
}

List<PropertyDescriptor> descriptors = Arrays.stream(beanInfo.getPropertyDescriptors()).filter(p -> {
    String name = p.getName();
    //过滤掉不需要修改的属性
    return !"class".equals(name) && !"id".equals(name);
}).collect(Collectors.toList());

for (PropertyDescriptor descriptor : descriptors) {
    //descriptor.getWriteMethod()方法对应set方法
    Method readMethod = descriptor.getReadMethod();
    System.out.println(descriptor.getName());
    try {
        Object o = readMethod.invoke(template);
        System.out.println(o);
    } catch (IllegalAccessException | InvocationTargetException e) {
        log.info("xxxxxxxxxxxxxxxx", e);
        return null;
    }
}
```
PropertyDescriptor类提供了getReadMethod和getWriteMethod，其实就是Getter、Setter方法，这样就可以避免方法名拼错。
另外PropertyDescriptor除了可以通过Introspector获取，也可以自己new来创建，其构造方法还是比较全的
通常传递一个属性的名称和类对象class就可以了
```java
List<Field> fields = Arrays.stream(template.getClass().getDeclaredFields()).filter(f -> {
    String name = f.getName();
    //过滤掉不需要修改的属性
    return !"id".equals(name) && !"serialVersionUID".equals(name);
}).collect(Collectors.toList());

for (Field field : fields) {
    try {
        PropertyDescriptor descriptor = new PropertyDescriptor(field.getName(), template.getClass());
        Method readMethod = descriptor.getReadMethod();
        Object o = readMethod.invoke(template);
        System.out.println(o);
    } catch (IntrospectionException | IllegalAccessException | InvocationTargetException e) {
        e.printStackTrace();
    }
}
```
通过上面两种不同的实现方式可以看到，Introspector会额外有一个class属性，但是类似serialVersionUID不会算在内；而自定义PropertyDescriptor需要通过反射拿到所有的属性，虽然不会有class属性，但是serialVersionUID会算在内，使用的时候需要注意一下。
## beanInfoCache
Introspector不同于普通的反射，反射一次，一段时间内可重复使用，为什么不是永久呢，看下源码。
```java
/**
     * Introspect on a Java Bean and learn about all its properties, exposed
     * methods, and events.
     * <p>
     * If the BeanInfo class for a Java Bean has been previously Introspected
     * then the BeanInfo class is retrieved from the BeanInfo cache.
     *
     * @param beanClass  The bean class to be analyzed.
     * @return  A BeanInfo object describing the target bean.
     * @exception IntrospectionException if an exception occurs during
     *              introspection.
     * @see #flushCaches
     * @see #flushFromCaches
     */
    public static BeanInfo getBeanInfo(Class<?> beanClass)
        throws IntrospectionException
    {
        if (!ReflectUtil.isPackageAccessible(beanClass)) {
            return (new Introspector(beanClass, null, USE_ALL_BEANINFO)).getBeanInfo();
        }
        ThreadGroupContext context = ThreadGroupContext.getContext();
        BeanInfo beanInfo;
        synchronized (declaredMethodCache) {
            beanInfo = context.getBeanInfo(beanClass);
        }
        if (beanInfo == null) {
            beanInfo = new Introspector(beanClass, null, USE_ALL_BEANINFO).getBeanInfo();
            synchronized (declaredMethodCache) {
                context.putBeanInfo(beanClass, beanInfo);
            }
        }
        return beanInfo;
    }
```
注意中间加粗标红的代码，这里除了同步之外，还做了一个本地的缓存
```java
BeanInfo getBeanInfo(Class<?> type) {
    return (this.beanInfoCache != null)
            ? this.beanInfoCache.get(type)
            : null;
}
```
这个beanInfoCache 其实是一个WeakHashMap，每次gc被回收，所以上面说一段时间内可以重复使用而不是永久，也是为了避免OOM吧。
