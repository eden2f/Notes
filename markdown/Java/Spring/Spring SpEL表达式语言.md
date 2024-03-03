# Spring SpEL表达式语言

> [Sharpest博客原文引用](https://www.cnblogs.com/sharpest/p/10885215.html)

## SpEL简介与功能特性
Spring表达式语言（简称SpEL）是一个支持查询并在运行时操纵一个对象图的功能强大的表达式语言。SpEL语言的语法类似于统一EL，但提供了更多的功能，最主要的是显式方法调用和基本字符串模板函数。
同很多可用的Java 表达式语言相比，例如OGNL，MVEL和JBoss EL，SpEL的诞生是为了给Spring社区提供一个可以给Spring目录中所有产品提供单一良好支持的表达式语言。其语言特性由Spring目录中的项目需求驱动，包括基于eclipse的SpringSource套件中的代码补全工具需求。也就是说，SpEL是一个基于技术中立的API允许需要时与其他表达式语言集成。
SpEL作为Spring目录中表达式求值的基础，它并不是直接依赖于Spring而是可以被独立使用。为了能够自包含，本章中的许多示例把SpEL作为一个独立的表达式语言来使用。这就需要创建一些如解析器的引导基础组件类。大多数Spring用户只需要为求值编写表达式字符串而不需要关心这些基础组件.
## 一、如何使用Spring表达式语言
Maven依赖配置
```xml
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>2.4.0</version>
	<relativePath/> <!-- lookup parent from repository -->
</parent>

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-expression</artifactId>
</dependency>
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.11</version>
    <scope>test</scope>
</dependency>
```
## 二、SpEL表达式Hello World
Spring表达式语言(SpEL)从3.X开始支持，它是一种能够支持运行时查询和操作对象图的强大的表达式，其表达式语法类似于统一表达式语言。
SpEL支持如下表达式：

-  基本表达式：字面量表达式、关系，逻辑与算数运算表达式、字符串连接及截取表达式、三目运算、正则表达式、括号优先级表达式； 
-  类相关表达式：类类型表达式、类实例化、instanceof表达式、变量定义及引用、赋值表达式、自定义函数、对象属性存取及安全导航表达式、对象方法调用、Bean引用； 
-  集合相关表达式：内联List、内联数组、集合，字典访问、列表，字典，数组修改、集合投影、集合选择；不支持多维内联数组初始化；不支持内联字典定义； 
-  其他表达式：模板表达式。 
```java
@Test
public void helloWorld() {
    //创建SpEL表达式的解析器
    ExpressionParser parser = new SpelExpressionParser();
    //解析表达式'Hello '+' World!'
    Expression exp = parser.parseExpression("'Hello '+' World!'");
    //取出解析结果
    String result = exp.getValue().toString();
    //输出结果
    System.out.println(result);
}
```
## 三、SpEL表达式 Demo
## 3.1、文字表达式
```java
/**
* 文字表达式
*/
@Test
public void literalExpression() {
    ExpressionParser ep = new SpelExpressionParser();
    System.out.println(ep.parseExpression("'HelloWorld'").getValue());
    System.out.println(ep.parseExpression("0xffffff").getValue());
    System.out.println(ep.parseExpression("1.234345e+3").getValue());
    System.out.println(ep.parseExpression("new java.util.Date()").getValue());
}
```
## 3.2、SPEL语言特性
### 3.2.1、数组
```java
/**
* SPEL语言特性 变量设置 array
*/
@Test
public void spelLanguageFeaturesArray() {
    ExpressionParser parser = new SpelExpressionParser();
    EvaluationContext ctx = new StandardEvaluationContext();
    String[] students = new String[]{"tom", "jack", "rose", "mark", "lucy"};
    ctx.setVariable("students", students);
    String student = parser.parseExpression("#students[3]").getValue(ctx, String.class);
    System.out.println(student);
}
```
### 3.2.2、列表
```java
/**
* SPEL语言特性 变量设置 list
*/
@Test
public void spelLanguageFeaturesList() {
    ExpressionParser parser = new SpelExpressionParser();
    EvaluationContext ctx = new StandardEvaluationContext();
    List<Integer> numbers = (List<Integer>) parser.parseExpression("{1,2,3,4,5}").getValue(List.class);
    System.out.println(String.format("numbers = %s", numbers));
    List<List<String>> listOfLists = (List<List<String>>) parser.parseExpression("{{'a','b'},{'x','y'}}").getValue(List.class);
    System.out.println(String.format("listOfLists = %s", listOfLists));
    System.out.println(String.format("listOfLists[0][0] = %s", (listOfLists.get(0)).get(0)));
}
```
### 3.2.3、索引器、与字典
```java
/**
* SPEL语言特性 变量设置 index
*/
@Test
public void spelLanguageFeaturesIndex() {
    ExpressionParser parser = new SpelExpressionParser();
    EvaluationContext ctx = new StandardEvaluationContext();
    User user1 = new User(1, "张三");
    User user2 = new User(2, "李四");
    List<User> users = Arrays.asList(user1, user2);
    ctx.setVariable("users", users);
    String name = parser.parseExpression("#users[1].name").getValue(ctx, String.class);
    System.out.println(String.format("name = %s", name));
}
```
### 3.2.4、方法
```java
/**
* SPEL语言特性 变量设置 动态编码
*/
@Test
public void spelLanguageFeaturesDynamicCode() {
    ExpressionParser parser = new SpelExpressionParser();
    String value = parser.parseExpression("'abcdef'.substring(2, 3)").getValue(String.class);
    System.out.println(String.format("value = %s", value));
}
```
### 3.2.5、操作符
```java
/**
* SPEL语言特性 变量设置 操作符
*/
@Test
public void spelLanguageFeaturesOperationalCharacter() {
    ExpressionParser parser = new SpelExpressionParser();
    //true
    boolean trueValue1 = parser.parseExpression("2 == 2").getValue(Boolean.class);
    System.out.println("trueValue1 = " + trueValue1);
    //false
    boolean falseValue1 = parser.parseExpression("2 < -5.0").getValue(Boolean.class);
    System.out.println("falseValue1 = " + falseValue1);
    //true
    boolean trueValue2 = parser.parseExpression("'black' < 'block'").getValue(Boolean.class);
    System.out.println("trueValue2 = " + trueValue2);
    //false，字符xyz是否为int类型
    boolean falseValue2 = parser.parseExpression("'xyz' instanceof T(int)").getValue(Boolean.class);
    System.out.println("falseValue2 = " + falseValue2);
    //true，正则是否匹配
    boolean trueValue3 = parser.parseExpression("'5.00' matches '^-?\\d+(\\.\\d{2})?$'").getValue(Boolean.class);
    System.out.println("trueValue3 = " + trueValue3);
    //false
    boolean falseValue3 = parser.parseExpression("'5.0067' matches '^-?\\d+(\\.\\d{2})?$'").getValue(Boolean.class);
    System.out.println("falseValue3 = " + falseValue3);
    // -- AND 与运算 --
    //false
    boolean falseValue4 = parser.parseExpression("true and false").getValue(Boolean.class);
    System.out.println("falseValue4 = " + falseValue4);
    // -- OR 或运算--
    //true
    boolean trueValue5 = parser.parseExpression("true or false").getValue(Boolean.class);
    System.out.println("trueValue5 = " + trueValue5);
    // -- ! 非运算--
    //false
    boolean falseValue5 = parser.parseExpression("!true").getValue(Boolean.class);
    System.out.println("falseValue5 = " + falseValue5);
    ExpressionParser ep = new SpelExpressionParser();
    // 关系操作符
    System.out.println(ep.parseExpression("5>2").getValue());
    System.out.println(ep.parseExpression("2 between {1,9}").getValue());
    // 逻辑运算符
    System.out.println(ep.parseExpression("(5>2) and (2==1)").getValue());
    // 算术操作符
    System.out.println(ep.parseExpression("100-2^2").getValue());
}
```
### 3.2.6、算术运算
```java
/**
* SPEL语言特性 变量设置 算术运算
*/
@Test
public void spelLanguageFeaturesArithmeticOperation() {
    ExpressionParser parser = new SpelExpressionParser();
    // Addition
    // 2
    int two = parser.parseExpression("1 + 1").getValue(Integer.class);
    // 'test string'
    String testString = parser.parseExpression("'test' + ' ' + 'string'").getValue(String.class); //
    // Subtraction
    // 4
    int four = parser.parseExpression("1 - -3").getValue(Integer.class);
    // -9000
    double d = parser.parseExpression("1000.00 - 1e4").getValue(Double.class);
    // Multiplication
    // 6
    int six = parser.parseExpression("-2 * -3").getValue(Integer.class);
    // 24.0
    double twentyFour = parser.parseExpression("2.0 * 3e0 * 4").getValue(Double.class);
    // Division
    // -2
    int minusTwo = parser.parseExpression("6 / -3").getValue(Integer.class);
    // 1.0
    double one = parser.parseExpression("8.0 / 4e0 / 2").getValue(Double.class);
    // Modulus
    // 3
    int three = parser.parseExpression("7 % 4").getValue(Integer.class);
    // 1
    int one2 = parser.parseExpression("8 / 5 % 2").getValue(Integer.class);
    // Operator precedence
    // -21
    int minusTwentyOne = parser.parseExpression("1+2-3*8").getValue(Integer.class);
}
```
### 3.2.7、变量与赋值
```java
/**
* SPEL语言特性 变量设置
*/
@Test
public void spelLanguageFeatures() {
    //创建SpEL表达式的解析器
    ExpressionParser parser = new SpelExpressionParser();
    User user = new User(9527, "周星驰");
    //解析表达式需要的上下文，解析时有一个默认的上下文
    EvaluationContext ctx = new StandardEvaluationContext();
    //在上下文中设置变量，变量名为user，内容为user对象
    ctx.setVariable("user", user);
    //从用户对象中获得id并+1900，获得解析后的值在ctx上下文中
    int id = (Integer) parser.parseExpression("#user.getId() + 1900").getValue(ctx);
    System.out.println(id);
    ctx.setVariable("name", "Hello");
    System.out.println(parser.parseExpression("#name").getValue(ctx));
}

public class User {

    public User(Integer id, String name) {
        this.id = id;
        this.name = name;
    }

    private Integer id;

    private String name;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", name='" + name + '\'' +
                '}';
    }
}
```
