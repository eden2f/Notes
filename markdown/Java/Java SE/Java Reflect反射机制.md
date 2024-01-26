# Java Reflect反射机制

## 类字节码

类字节码文件是在硬盘上存储的，是一个个的.class文件。我们在new一个对象时，**JVM会先把字节码文件的信息读出来放到内存中，第二次用时，就不用在加载了，而是直接使用之前缓存的这个字节码信息**。

字节码的信息包括：类名、声明的方法、声明的字段等信息。在Java中“万物皆对象”，这些信息当然也需要封装一个对象，这就是Class类、Method类、Field类。

通过Class类、Method类、Field类等等类可以得到这个类型的一些信息，甚至可以不用new关键字就创建一个实例，可以执行一个对象中的方法，设置或获取字段的值，这就是反射技术。

## 获取Class对象的三种方式

Java中有一个Class类用于代表某一个类的字节码。

1. Class.forName("")方法用于加载某个类的字节码到内存中，并使用class对象进行封装
2. 类名.class
3. 对象.getClass()

``` java
/**
    * 加载类的字节码的3种方式
    * @throws ClassNotFoundException
    */
@Test
public void test() throws ClassNotFoundException {
    // 方式一
    Class clazz1 = Class.forName("cn.huangmp.java.api.reflect.Person");
    // 方式二
    Class clazz2 = Person.class;

    // JVM会先把字节码文件的信息读出来放到内存中，第二次用时，就不用在加载了，
    // 而是直接使用之前缓存的这个字节码信息。
    // clazz1 == clazz2 : true
    System.out.println("clazz1 == clazz2 : " + (clazz1 == clazz2));

    // 方式三
    Person p1 = new Person();
    Class clazz3 = p1.getClass();

    System.out.println("clazz3 == clazz2 : " + (clazz3 == clazz2));
}
```
## 通过Class类获取类型的一些信息

1. getName()类的名称（全名，全限定名）
2. getSimpleName()类的的简单名称（不带包名）
3. getModifiers(); 类的的修饰符
4. 创建对象 无参数构造创建对象 newInstance()
5. 获取指定参数的构造器对象，并可以使用Constructor对象创建一个实例 Constructor<T> getConstructor(Class<?>... parameterTypes)

``` java
/**
    * 通过Class对象获取类的一些信息
    * @throws Exception
    */
@Test
public void test2() throws Exception {
    Class clazz1 = Class.forName("cn.huangmp.java.api.reflect.Person");
    // 获取类的名称
    String name = clazz1.getName();
    System.out.println(name);
    // 获取类的简单名称
    System.out.println(clazz1.getSimpleName()); // Person
    // 获取类的修饰符
    int modifiers = clazz1.getModifiers();
    System.out.println(modifiers);

    // 通过class对象找到所有的构造方法
    Constructor[] constructors = clazz1.getConstructors();
    for( Constructor constructor : constructors){
        System.out.println("    " + constructor);
    }

    // 通过class对象找到所有的构造方法,包括私有方法
    Constructor[] declaredConstructors = clazz1.getDeclaredConstructors();
    for( Constructor constructor : declaredConstructors){
        System.out.println("    " + constructor);
    }

    // 构建对象(默认调用无参数构造.)
    Object ins = clazz1.newInstance();
    Person p = (Person) ins;
    System.out.println(p);
    // 获取指定参数的构造函数
    Constructor<?> con = clazz1.getConstructor(String.class, int.class);
    // 使用Constructor创建对象.
    Object p1 = con.newInstance("jack", 12);
    System.out.println(((Person) p1).getName());

    // 通过私有构造方法创建对象
    Constructor constructor = clazz1.getDeclaredConstructor(String.class, int.class, int.class);
    // 设置构造方法的访问权限(暴力反射)
    constructor.setAccessible(true);
    Person p2 = (Person) constructor.newInstance("jj", 20, 1);
    System.out.println("p2.name = " + p2.getName());
}
```

## 通过Class类获取类型中的方法的信息

1.获取公共方法包括继承的父类的方法 getMethods()返回一个数组,元素类型是Method
2.获取指定参数的公共方法 getMethod("setName", String.class);
3.获得所有的方法,包括私有 Method[] getDeclaredMethods()
4.获得指定参数的方法,包括私有 Method getDeclaredMethod(String name, Class<?>... parameterTypes)

``` java
/**
    * 获取公有方法.
    * @throws Exception
    */
@Test
public void test3() throws Exception {
    Class clazz1 = Class.forName("cn.huangmp.java.api.reflect.Person");
    // 1.获取非私用方法(包括父类继承的方法)
    Method[] methods = clazz1.getMethods();
    System.out.println(methods.length);
    for (Method m : methods) {
        // System.out.println(m.getName());
        if ("eat".equals(m.getName())) {
            m.invoke(clazz1.newInstance(), null);
        }
        System.out.println("    " + m.getName());
    }
}

/**
    * 获取指定方法签名的方法
    *
    * @throws Exception
    */
public void test4() throws Exception {
    Class clazz1 = Class.forName("cn.huangmp.java.api.reflect.Person");
    // 获取指定名称的函数
    // 第一个参数是方法名, 第二个参数是形参列表的数据类型
    Method method1 = clazz1.getMethod("toString", null);
    // 执行方法
    // 第一个参数: 方法的调用者(对象),第二个参数: 方法执行的第二个参数
    method1.invoke(new Person(), null);
}

/**
    * 获取指定方法名且有参数的方法
    *
    * @throws Exception
    */
@Test
public void test5() throws Exception {
    Class clazz1 = Class.forName("cn.huangmp.java.api.reflect.Person");
    Method method = clazz1.getMethod("setName", String.class);
    method.invoke(new Person(), "包子");
}

/**
    * 获取指定方法名,参数列表为空的方法.
    *
    * @throws Exception
    */
@Test
public void test6() throws Exception {
    Class clazz1 = Class.forName("cn.huangmp.java.api.reflect.Person");
    // 获取指定名称的函数
    Method method1 = clazz1.getMethod("sayHello", null);
    method1.invoke(new Person(), null);
}

/**
    * 反射静态方法
    * @throws Exception
    */
@Test
public void test8() throws Exception {
    Class clazz1 = Class.forName("cn.huangmp.java.api.reflect.Person");
    Method method = clazz1.getMethod("sayHi", null);
    method.invoke(null, null);
}
```
	
## 通过Class类获取类型中的字段的信息

1. 获取公共字段 Field[] getFields()
2. 获取指定参数的公共字段 Field getField(String name)
3. 获取所有的字段 Field[] getDeclaredFields()
4. 获取指定参数的字段,包括私用 Field getDeclaredField(String name)

``` java
/**
    * 获取公有的字段
    */
@Test
public void test9() throws Exception {
    Class clazz1 = Class.forName("cn.huangmp.java.api.reflect.Person");
    Field[] fields = clazz1.getFields();
    Person p = new Person();
    System.out.println(fields.length);
    for (Field f : fields) {
        System.out.println(f.getName());
        if ("name".equals(f.getName())) {
            System.out.println(f.getType().getName());
            f.set(p, "jack");
        }
    }
    System.out.println(p.getName());
}

/**
    * 获取私有的字段
    * @throws Exception
    */
@Test
public void test10() throws Exception {
    Class clazz1 = Class.forName("cn.huangmp.java.api.reflect.Person");
    Field field = clazz1.getDeclaredField("age");
    System.out.println(field.getName());
    field.setAccessible(true);
    Person p = new Person();
    field.set(p, 100);
    System.out.println(p.getAge());
}
```