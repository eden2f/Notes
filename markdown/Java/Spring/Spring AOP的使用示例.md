# Spring AOP的使用示例

## 什么是AB实验
AB Test 实验一般有 2 个目的：

1. 判断哪个更好：例如，有 2 个 UI 设计，究竟是 A 更好一些，还是 B 更好一些，我们需要实验判定
2. 计算收益：例如，最近新上线了一个直播功能，那么直播功能究竟给平台带了来多少额外的 DAU，多少额外的使用时长，多少直播以外的视频观看时长等

以上例子取自文章 : 什么是 A/B 测试？: [https://www.zhihu.com/question/20045543](https://www.zhihu.com/question/20045543)
实际上, 一个产品需求, 可能会有多种落地策略 (重点:不一就2种,可能有3456种) , 选取小部分流量, 通过AB实验实现分流, 最终根据实验结构选择最终的落地方案.
## 为什么要做AB实验
If you are not running experiments，you are probably not growing！——by Sean Ellis
Sean Ellis 是增长黑客模型（AARRR）之父，增长黑客模型中提到的一个重要思想就是“AB实验”。
从某种意义上讲，自然界早就给了我们足够多的启示。为了适应多变的环境，生物群体每天都在发生基因的变异，最终物竞天择，适者生存，留下了最好的基因。这个精巧绝伦的生物算法恐怕是造物者布置的最成功的AB实验吧。
> AB实验的必要性可以查看下面文章链接, 这里不再赘述.
本文首发｜微信公众号 友盟数据服务 （ID：umengcom），转载请注明出处
[BAT 都在用的方法，详解 A/B 测试的那些坑！:https://leeguoren.blog.csdn.net/article/details/103994848](https://leeguoren.blog.csdn.net/article/details/103994848)

## 基于后端的AB实验实现方案
举一个场景, 假设有如下产品需求 : 对于商品信息展示页面, 对于商品名称的展示上有两个方案, 但是不知道哪个方案好, 所以需要做个测试一下;
方案一 : 在商品名称改成 “Success” ; 方案二 : 在商品名称改成 “Fail” ;
需求就是这么个需求, 接下来看看怎么实现吧! 如有雷同, 纯属巧合~
### 项目代码仓库
下面的代码实现放在这里哈, 项目可以直接运行.
[Github eden2f/springboot-web-demo](https://github.com/eden2f/springboot-web-demo.git)
[Gitee eden2f/springboot-web-demo](https://gitee.com/eden2f/springboot-web-demo.git)
## 效果显示
### 后端接口定义

- 服务端口 : 8080
- 测试接口 : 
   - 接口协议 : Http , 方法 : GET , URL : /experiment/experimentableTest
   - 返回数据结构 :
```json
{
  "code": 200,
  "msg": "ok",
  "data": "Success",
  "traceId": "a8002fa2-3fdf-450d-8c9e-e4ff4bed078c"
}
```
### 效果展示
执行 Curl 调用接口 :
```shell
curl -X GET "http://localhost:8080/experiment/experimentableTest" -H "accept: */*"
```
结果 : 50% 的机率返回 "data": "Success"; 50% 的机率返回 "data": "Fail";
## 实现思路
本文重点讲解如何在不更新业务代码的情况下, 实现服务端逻辑分流? 至于实验投放算法的实现、投放人群选取……等等这些本文不涉及. 而且是个大课题, 本文也讲不完
对于后端服务, 一般有分布式配置中心(例如: Apollo、Nacos), 配置中心一般使用 Key : Value 方式帮我们托管着服务必要的配置信息;
以 Spring 项目为例, 在后端代码中实现获取分布式配置中心上的配置信息, 也是非常简单的, 如使用@Value, 下面是一个获取配置的使用示例 :
```java
@Value("${value:experimentableTest}")
private String name;
```
如果能够在调用属性的 Getter 方法时候根据不同场景获取不同的实验值, 再提供一个业务场景与实验值的配置管理, 那么就可以无缝支持上面的AB实验? 本组件也是围绕这思路来实现的.
为什么是 Spring AOP ?
为了实现AB实验能力接入对业务开发无感, 另一方面当前已经存在很多正在运行的代码, 如何不改动当前的业务实现又能使其拥有AB实验的能力? 到这里, 我想到了面向切面编程, 实现上就选取了 Spring 的 AOP.
## 组件编码实现
### 组件使用示例

- 开放 HTTP 接口
```java
@Slf4j
@RestController
@RequestMapping("experiment")
public class ExperimentController {

    @Resource
    private ExperimentService experimentService;

    @GetMapping("experimentableTest")
    public RetResult<String> experimentableTest() {
        String name = experimentService.getName();
        return RetResult.success(name);
    }

}
```

- Service 业务处理, 提供商品名称查询能力, getName() 方法返回从配置中心拿到的名称; 默认配置是 experimentableTest, 我们希望 getName()   根据场景返回 Success 和 Fail.
```java
@Slf4j
@Getter
@Service
@Experimentable
public class ExperimentService {

    @Value("${value:experimentableTest}")
    private String name;

}
```
### 组件实现编码
划重点~下面开始讲实验组件的编码实现了

- 自定义一个功能标记注解：可实验 @Experimentable, 加在需要增强AB实验能力的Class上, 如下面的 ExperimentService.
```java
/**
 * 功能标记注解：可实验
 */
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Experimentable {
}
```

- 业务场景与实验值的映射管理, 为了读者能快速理解, 本例一切从简, ExperimentSettingDemo.EXPERIMENT_SETTINGMAP 进行映射管理, Key是属性名, Value 是一组实验值. 回到需求, 对于 name 这个字段, 有 "Fail", "Success" 两种展示方案, 在本例中配置如下 :
```java
/**
 * 实验配置示例
 */
public class ExperimentSettingDemo {

    /**
     * 实验参数配置
     */
    public static final Map<String, List<String>> EXPERIMENT_SETTINGMAP;

    /**
     * 实验参数配置
     */
    public static final Set<String> EXPERIMENT_PROPERTY_NAME;

    static {
        EXPERIMENT_SETTINGMAP = new HashMap<>();
        EXPERIMENT_SETTINGMAP.put("name", Lists.newArrayList("Fail", "Success"));
        EXPERIMENT_SETTINGMAP.put("wallet", Lists.newArrayList("100", "200"));
        EXPERIMENT_SETTINGMAP.put("age", Lists.newArrayList("1", "2"));
        // 这个配置应该是无效的因为没有 @Value
        EXPERIMENT_SETTINGMAP.put("birthday", Lists.newArrayList("1000", "2000"));
        EXPERIMENT_PROPERTY_NAME = EXPERIMENT_SETTINGMAP.keySet();
    }
}
```

- ExperimentAspect 是AB实验能力增强切面, 实现了对 Experimentable 的对象的属性 Getter 方法的增强; 同时, 作为实验室的实现, 实现了 “判断某个属性是不是是否在实验中” 和 “根据实验属性和特定场景查询实验值” 的能力; 
   - 判断某个属性是不是是否在实验中 : 读取 ExperimentSettingDemo 进行判断
   - 根据实验属性和特定场景查询实验值 : 多个实验值随机选择
```java
@Slf4j
public class ExperimentInterceptor implements MethodInterceptor {

    private static final ExperimentParamMetedata nonExperimentableMetedata = new ExperimentParamMetedata();


    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        return ExperimentInterceptor.experimentInvoke(methodInvocation.proceed(), methodInvocation.getThis(), methodInvocation.getMethod().getName());
    }

    public static Object experimentInvoke(Object originalValue, Object target, String targetMethodName) {
        ExperimentParamMetedata experimentParamMetedata;
        try {
            experimentParamMetedata = ExperimentInterceptor.inExperiment(target, targetMethodName, ExperimentSettingDemo.EXPERIMENT_SETTINGMAP);
        } catch (RuntimeException exception) {
            log.error("ExperimentAspect-未知异常", exception);
            return originalValue;
        }
        if (experimentParamMetedata.isExperimentable()) {
            try {
                return Laboratory.queryExperimentValue(experimentParamMetedata.getPropertyName(), experimentParamMetedata.getPropertyTypeClass());
            } catch (NoSuchMethodException | InvocationTargetException | InstantiationException | IllegalAccessException e) {
                log.error("ExperimentAspect-结果解析异常", e);
                throw new RuntimeException("ExperimentAspect-结果解析异常", e);
            }
        }
        return originalValue;
    }


    /**
     * 判断当前目标方法是不是需要进行实验
     *
     * @param experimentSettingMap 实验配置
     * @return 方法可实验性校验结果
     */
    private static ExperimentParamMetedata inExperiment(Object target, String targetMethodName, Map<String, List<String>> experimentSettingMap) {
        Class<?> targetClass = AopUtils.isAopProxy(target) ? AopUtils.getTargetClass(target) : target.getClass();
        // 是否在实验中
        NonExperimentable nonExperimentable = AnnotationUtils.findAnnotation(targetClass, NonExperimentable.class);
        if (null != nonExperimentable) {
            return nonExperimentableMetedata;
        }
        // 实验配置是否有数据
        if (null == experimentSettingMap || experimentSettingMap.isEmpty()) {
            return nonExperimentableMetedata;
        }
        BeanInfo targetBeanInfo;
        try {
            targetBeanInfo = Introspector.getBeanInfo(target.getClass());
        } catch (IntrospectionException e) {
            throw new RuntimeException(e);
        }
        Optional<PropertyDescriptor> propertyDescriptorOptional = Arrays.stream(targetBeanInfo.getPropertyDescriptors())
                .filter(item -> item.getReadMethod().getName().equals(targetMethodName)).findFirst();
        if (propertyDescriptorOptional.isPresent()) {
            PropertyDescriptor propertyDescriptor = propertyDescriptorOptional.get();
            String propertyName = propertyDescriptor.getName();
            Field propertyField = ReflectionUtils.findField(targetClass, propertyName);
            if (propertyField != null) {
                Value valueAnnotation = propertyField.getDeclaredAnnotation(Value.class);
                if (null != valueAnnotation && ExperimentSettingDemo.EXPERIMENT_PROPERTY_NAME.contains(propertyName)) {
                    ExperimentParamMetedata experimentParamMetedata = new ExperimentParamMetedata();
                    experimentParamMetedata.setExperimentable(true);
                    experimentParamMetedata.setPropertyTypeClass(propertyDescriptor.getPropertyType());
                    experimentParamMetedata.setPropertyName(propertyName);
                    return experimentParamMetedata;
                }
            }
        }
        return nonExperimentableMetedata;
    }

    /**
     * 查询属性对应的实验值
     *
     * @param experimentPropertyName 实验属性名称
     * @param propertyTypeClass      属性类型
     * @return 实验值
     */
    public static Object queryExperimentValue(String experimentPropertyName, Class<?> propertyTypeClass) throws InvocationTargetException, NoSuchMethodException, InstantiationException, IllegalAccessException {
        List<String> experimentReturnStringValues = ExperimentSettingDemo.EXPERIMENT_SETTINGMAP.get(experimentPropertyName);
        // 几个配置随机选一个返回
        int index = RandomUtils.nextInt(0, experimentReturnStringValues.size());
        return StringCastUtil.cast(experimentReturnStringValues.get(index), propertyTypeClass);
    }
}
```

- 实验参数元数据
```java
/**
 * 实验参数元数据
 */
@Getter
@Setter
@ToString
public class ExperimentParamMetedata {

    /**
     * 可实验性
     */
    private boolean experimentable = false;

    /***
     * 实验属性名称
     */
    private String propertyName = null;

    /***
     * 方法返回结果类型
     */
    private Class<?> propertyTypeClass = null;

}
```

- 功能标记注解：不必实验的
```java
/**
 * 功能标记注解：不必实验的
 */
@Target({ElementType.TYPE, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface NonExperimentable {
}
```

- 将字符串转值对象的工具类（仅支持转基本类型）
```java
/**
 * 将字符串转值对象的工具类（仅支持转基本类型）
 */
public class StringCastUtil {

    private static final Map<Class<?>, Class<?>> BASIC_TYPE_CLASS_MAP;
    private static final Set<Class<?>> basicTypeClassSet;

    static {
        BASIC_TYPE_CLASS_MAP = new HashMap<>(32);
        BASIC_TYPE_CLASS_MAP.put(byte.class, Byte.class);
        BASIC_TYPE_CLASS_MAP.put(short.class, Short.class);
        BASIC_TYPE_CLASS_MAP.put(int.class, Integer.class);
        BASIC_TYPE_CLASS_MAP.put(long.class, Long.class);
        BASIC_TYPE_CLASS_MAP.put(float.class, Float.class);
        BASIC_TYPE_CLASS_MAP.put(double.class, Double.class);
        BASIC_TYPE_CLASS_MAP.put(boolean.class, Boolean.class);
        BASIC_TYPE_CLASS_MAP.put(char.class, Character.class);
        BASIC_TYPE_CLASS_MAP.put(Byte.class, Byte.class);
        BASIC_TYPE_CLASS_MAP.put(Short.class, Short.class);
        BASIC_TYPE_CLASS_MAP.put(Integer.class, Integer.class);
        BASIC_TYPE_CLASS_MAP.put(Long.class, Long.class);
        BASIC_TYPE_CLASS_MAP.put(Float.class, Float.class);
        BASIC_TYPE_CLASS_MAP.put(Double.class, Double.class);
        BASIC_TYPE_CLASS_MAP.put(Boolean.class, Boolean.class);
        BASIC_TYPE_CLASS_MAP.put(Character.class, Character.class);
        basicTypeClassSet = BASIC_TYPE_CLASS_MAP.keySet();
    }

    /**
     * 将字符串转成值对象
     *
     * @param valueString    值字符串
     * @param valueTypeClass 值类型
     * @return 值对象
     */
    public static Object cast(String valueString, Class<?> valueTypeClass) throws NoSuchMethodException, InvocationTargetException, InstantiationException, IllegalAccessException {
        if (String.class.equals(valueTypeClass)) {
            return valueString;
        }
        if (basicTypeClassSet.contains(valueTypeClass)) {
            return BASIC_TYPE_CLASS_MAP.get(valueTypeClass).getConstructor(String.class).newInstance(valueString);
        }
        throw new RuntimeException("不支持的属性类型, valueTypeClass = {}" + valueTypeClass);
    }
}
```
