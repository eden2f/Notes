# Jackson parser LocalDataTime 异常

## 问题

LocalDataTime 是 jdk8 的新特性，时间操作工具类。

jackson 解析 LocalDataTime 时会抛异常。

## 解决方法

使用以下注解，标明解析的对象

``` java

@JsonSerialize
@JsonDeserialize
```

``` java
@JsonSerialize(using = LocalDateTimeSerializer.class)
@JsonDeserialize(using = LocalDateTimeDeserializer.class)
private LocalDateTime updateTime ;
```

## 相关文档

[Java JSON Tutorial](http://tutorials.jenkov.com/java-json/index.html)