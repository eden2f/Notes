# Jackson JSON parse error

## 异常信息

JSON parse error ： Can not construct instance of java.util.Date from String value

## 将json反序列化为java对象

1. json串

``` json
{
"id" : "1",
"name" : "测试商品",
"addTime" : "2017/1/05 11:23:09"
}
```

2. java 类 （省略get/set方法）

``` java
public class Item {
    private int id ;
    private String name ;
    private Date addTime ;
}
```

## 解决方法

### 自定义json串解析器

``` java
public class OptimizedDateDeserializer  extends JsonDeserializer<Date> {
    @Override
    public Date deserialize(
            JsonParser jsonParser,
            DeserializationContext deserializationContext)
            throws IOException, JsonProcessingException {

        SimpleDateFormat format = new SimpleDateFormat("yyyy/MM/dd HH:mm:ss");
        String date = jsonParser.getText();
        try {
            return format.parse(date);
        } catch (ParseException e) {
            System.out.println("OptimizedDateDeserializer - 日期格式错误");
        }
        return null;
    }
}
```

### 指定解析器

``` java
@JsonDeserialize(using = OptimizedDateDeserializer.class)
private Date addTime ;
```


## 相关文档

* [Java JSON Tutorial](http://tutorials.jenkov.com/java-json/index.html)