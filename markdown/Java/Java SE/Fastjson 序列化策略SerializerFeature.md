# Fastjson 序列化策略SerializerFeature

> 引用博文 : [JSONObject 项目启动时初始化fastjson的Provider -> https://blog.csdn.net/zxl2016/article/details/80987414](https://blog.csdn.net/zxl2016/article/details/80987414)

## SerializerFeature 是啥?
```java
JSONObject.toJSONString(实体对象, SerializerFeature.WriteMapNullValue))
```
## SerializerFeature 源码
```java
package com.alibaba.fastjson.serializer;

public enum SerializerFeature {
   QuoteFieldNames,
   UseSingleQuotes,
   WriteMapNullValue,
   WriteEnumUsingToString,
   WriteEnumUsingName,
   UseISO8601DateFormat,
   WriteNullListAsEmpty,
   WriteNullStringAsEmpty,
   WriteNullNumberAsZero,
   WriteNullBooleanAsFalse,
   SkipTransientField,
   SortField,
   /** @deprecated */
   @Deprecated
   WriteTabAsSpecial,
   PrettyFormat,
   WriteClassName,
   DisableCircularReferenceDetect,
   WriteSlashAsSpecial,
   BrowserCompatible,
   WriteDateUseDateFormat,
   NotWriteRootClassName,
   /** @deprecated */
   DisableCheckSpecialChar,
   BeanToArray,
   WriteNonStringKeyAsString,
   NotWriteDefaultValue,
   BrowserSecure,
   IgnoreNonFieldGetter;

   public final int mask = 1 << this.ordinal();
   public static final SerializerFeature[] EMPTY = new SerializerFeature[0];
   public static final int WRITE_MAP_NULL_FEATURES = WriteMapNullValue.getMask() | WriteNullBooleanAsFalse.getMask() | WriteNullListAsEmpty.getMask() | WriteNullNumberAsZero.getMask() | WriteNullStringAsEmpty.getMask();

   private SerializerFeature() {
   }

   public final int getMask() {
       return this.mask;
   }

   public static boolean isEnabled(int features, SerializerFeature feature) {
       return (features & feature.mask) != 0;
   }

   public static boolean isEnabled(int features, int fieaturesB, SerializerFeature feature) {
       int mask = feature.mask;
       return (features & mask) != 0 || (fieaturesB & mask) != 0;
   }

   public static int config(int features, SerializerFeature feature, boolean state) {
       if (state) {
           features |= feature.mask;
       } else {
           features &= ~feature.mask;
       }

       return features;
   }

   public static int of(SerializerFeature[] features) {
       if (features == null) {
           return 0;
       } else {
           int value = 0;
           SerializerFeature[] var2 = features;
           int var3 = features.length;

           for(int var4 = 0; var4 < var3; ++var4) {
               SerializerFeature feature = var2[var4];
               value |= feature.mask;
           }

           return value;
       }
   }
}
```
## SerializerFeature属性解释
| 名称 | 含义 | 备注 |
| --- | --- | --- |
| QuoteFieldNames | 输出key时是否使用双引号,默认为true |  |
| UseSingleQuotes | 使用单引号而不是双引号,默认为false |  |
| WriteMapNullValue | 是否输出值为null的字段,默认为false |  |
|  |  |  |
| WriteEnumUsingToString | Enum输出name()或者ordinal |  |
| WriteEnumUsingName |  |  |
| UseISO8601DateFormat | Date使用ISO8601格式输出，默认为false |  |
| WriteNullListAsEmpty | List字段如果为null,输出为[],而非null |  |
| WriteNullStringAsEmpty | 字符类型字段如果为null,输出为”“,而非null |  |
| WriteNullNumberAsZero | 数值字段如果为null,输出为0,而非null |  |
| WriteNullBooleanAsFalse | Boolean字段如果为null,输出为false,而非null |  |
| SkipTransientField | 如果是true，类中的Get方法对应的Field是transient，序列化时将会被忽略。 |  |
| 默认为true |  |  |
| SortField | 按字段名称排序后输出。默认为false |  |
| WriteTabAsSpecial | 把\\t做转义输出，默认为false | 不推荐 |
| PrettyFormat | 结果是否格式化,默认为false | 不推荐 |
| WriteClassName | 序列化时写入类型信息，默认为false。反序列化是需用到 | 不推荐 |
| DisableCircularReferenceDetect | 消除对同一对象循环引用的问题，默认为false | 不推荐 |
| WriteSlashAsSpecial | 对斜杠’/’进行转义 | 不推荐 |
| BrowserCompatible | 将中文都会序列化为\\uXXXX格式，字节数会多一些，但是能兼容IE 6，默认为false | 不推荐 |
| WriteDateUseDateFormat | 全局修改日期格式,默认为false。 | 不推荐 |
| DisableCheckSpecialChar | 一个对象的字符串属性中如果有特殊字符如双引号，将会在转成json时带有反斜杠转移符。如果不需要转义，可以使用这个属性。默认为false | 不推荐 |
| NotWriteRootClassName |  | 不推荐 |
| BeanToArray | 将对象转为array输出 | 不推荐 |
| WriteNonStringKeyAsString |  | 不推荐 |
| NotWriteDefaultValue |  | 不推荐 |
| BrowserSecure |  | 不推荐 |
| IgnoreNonFieldGetter |  | 不推荐 |

## 注意
### WriteEnumUsingToString

1. 目前版本的fastjon默认对enum对象使用WriteEnumUsingName属性，因此会将enum值序列化为其Name。
2. 使用WriteEnumUsingToString方法可以序列化时将Enum转换为toString()的返回值；同时override toString函数能够将enum值输出需要的形式。但是这样做会带来一个问题，对应的反序列化使用的Enum的静态方法valueof可能无法识别自行生成的toString()，导致反序列化出错。
3. 如果将节省enum序列化后的大小，可以将enum序列化其ordinal值，保存为int类型。fastJson在反序列化时，如果值为int，则能够使用ordinal值匹配，找到合适的对象。

fastjson要将enum序列化为ordinal只需要禁止WriteEnumUsingName feature。
首先根据默认的features排除WriteEnumUsingName,然后使用新的features序列化即可。
```java
int features=SerializerFeature.config(JSON.DEFAULT_GENERATE_FEATURE, SerializerFeature.WriteEnumUsingName, false)
JSON.toJSONString(obj,features,SerializerFeature.EMPTY);
```
### DisableCircularReferenceDetect
当进行toJSONString的时候，默认如果重用对象的话，会使用引用的方式进行引用对象。
```json
[
    {
        "$ref": "$.itemSkuList[0].itemSpecificationList[0]"
    },
    {
        "$ref": "$.itemSkuList[1].itemSpecificationList[0]"
    }
]
```
#### 循环引用
很多场景中，我们需要序列化的对象中存在循环引用，在许多的json库中，这会导致stackoverflow。在功能强大的fastjson中，你不需要担心这个问题。例如：
```java
A a = new A();  
B b = new B(a);  
a.setB(b);  
String text = JSON.toJSONString(a); //{"b":{"a":{"$ref":".."}}}  
A a1 = JSON.parseObject(text, A.class);  
Assert.assertTrue(a1 == a1.getB().getA());
```
引用是通过"$ref"来表示的
#### 引用描述
```java
"$ref":".."  上一级
"$ref":"@"   当前对象，也就是自引用
"$ref":"$"   根对象
"$ref":"$.children.0"   基于路径的引用，相当于 root.getChildren().get(0)
```
### WriteDateUseDateFormat
```java
JSON.DEFFAULT_DATE_FORMAT = “yyyy-MM-dd”;
JSON.toJSONString(obj, SerializerFeature.WriteDateUseDateFormat);
```
