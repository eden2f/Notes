# SpringBoot 基于AOP实现的通用实验组件(AB实验/AB测试)

> [基于 Spring AOP 实现的 通用实验组件 AB实验/AB测试:https://www.jianshu.com/p/7caa224c2c80](https://www.jianshu.com/p/7caa224c2c80)

本文主要讲解SpringAOP的实现方式有哪些, 业务代码场景来自于上一篇文章 : 《基于 Spring AOP 实现的 通用实验组件 AB实验/AB测试》, 对该业务感兴趣的同学可以看看。
## 使用 [@Aspect ](/Aspect ) 注解(或者XML配置) 
```java
/**
 * 实验能力切面
 */
@Slf4j
@Aspect
@Component
public class ExperimentAspect {
    /**
     * 定义切点
     */
    @Pointcut("@within(com.eden.springbootwebdemo.web.experiment.Experimentable)")
    public void pointCut() {
    }

    /**
     * @param proceedingJoinPoint 被织入的目标
     * @return 方法执行结果
     */
    @Around("pointCut()")
    public Object doAround(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        return ExperimentInterceptor.experimentInvoke(proceedingJoinPoint.proceed(), proceedingJoinPoint.getTarget(), proceedingJoinPoint.getSignature().getName());
    }

}
```
## 实现 MethodInterceptor 接口
为什么可以通过这种方式实现AOP ? 如果你也有这样的问题, 挺好的, 一起学习. 这个涉及到 Spring Bean 加载过程, 流程比较复杂, 后面会做专题分享
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
                return queryExperimentValue(experimentParamMetedata.getPropertyName(), experimentParamMetedata.getPropertyTypeClass());
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
## 扫包的实现方式有哪些
AspectJExpressionPointcut \ AnnotationMatchingPointcut \ JdkRegexpMethodPointcut
```java
@Configuration
public class ExperimentInterceptorConfig {

    @Bean
    public DefaultPointcutAdvisor defaultPointcutAdvisor() {
        ExperimentInterceptor interceptor = new ExperimentInterceptor();
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
        pointcut.setExpression("execution(* com.eden.springbootwebdemo.service.*.*(..))");
        // AnnotationMatchingPointcut pointcut = new AnnotationMatchingPointcut(HfiTrace.class, true);
        // JdkRegexpMethodPointcut pointcut = new JdkRegexpMethodPointcut();
        // pointcut.setPatterns("com.eden.springbootwebdemo.service.*");
        DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor();
        advisor.setPointcut(pointcut);
        advisor.setAdvice(interceptor);
        return advisor;
    }
}
```
