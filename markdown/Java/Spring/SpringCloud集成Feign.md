# SpringCloud集成Feign

> 《重新定义Spring Cloud实战》

## Feign 简介
Feign是一个声明式的WebService客户端。它的出现使开发WebService客户端变得很简单。使用Feign只需要创建一个接口加上对应的注解，比如：FeignClient注解。Feign有可插拔的注解，包括Feign注解和JAXRS注解。Feign也支持编码器和解码器，SpringCloudOpenFeign对Feign进行增强支持SpringMVC注解，可以像SpringWeb一样使用HttpMessageConverters等。
## Feign工作原理

-  当程序启动时，会进行包扫描，扫描所有@FeignClients的注解的类，并将这些信息注入SpringIOC容器中。当定义的Feign接口中的方法被调用时，通过JDK的代理的方式，来生成具体的RequestTemplate。当生成代理时，Feign会为每个接口方法创建一个RequetTemplate对象，该对象封装了HTTP请求需要的全部信息，如请求参数名、请求方法等信息都是在这个过程中确定的。 
-  然后由RequestTemplate生成Request，然后把Request交给Client去处理，这里指的Client可以是JDK原生的URLConnection、Apache的HttpClient，也可以是Okhttp。最后Client被封装到LoadBalanceClient类，这个类结合Ribbon负载均衡发起服务之间的调用。 
## Hello World Demo
**环境配置**

1. JDK8
2. Maven

在Demo服务中，调用 github openAPI 的接口

- maven依赖
```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

- Feign配置 + 启用Feign
```java
@Configuration
@EnableFeignClients
public class HelloFeignServiceConfig {

    /**
     *
     * Logger.Level 的具体级别如下：
         NONE：不记录任何信息
         BASIC：仅记录请求方法、URL以及响应状态码和执行时间
         HEADERS：除了记录 BASIC级别的信息外，还会记录请求和响应的头信息
         FULL：记录所有请求与响应的明细，包括头信息、请求体、元数据
     * @return
     */
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }

}
```

- OpenAPI的调用客户端编写
```java
@FeignClient(name = "github-client", url = "https://api.github.com", configuration = HelloFeignServiceConfig.class)
public interface HelloFeignService {

    @RequestMapping(value = "/search/repositories", method = RequestMethod.GET)
    String searchRepo(@RequestParam("q") String queryStr);

}
```

- HelloFeignService 的使用
```java
@RestController
public class HelloFeignController {

    @Autowired
    private HelloFeignService helloFeignService;

    @GetMapping(value = "/search/github")
    public String searchGithubRepoByStr(@RequestParam("str") String queryStr) {
        return helloFeignService.searchRepo(queryStr);
    }

}
```
## 实战运用
### Feign默认Client的替换
Feign在默认情况下使用的是JDK原生的URLConnection发送HTTP请求，没有连接池，但是对每个地址会保持一个长连接，即利用HTTP的persistenceconnection。我们可以用Apache的HTTPClient替换Feign原始的HTTPClient，通过设置连接池、超时时间等对服务之间的调用调优。SpringCloud从Brixtion.SR5版本开始支持这种替换，接下来介绍一下如何用HTTPClient和okhttp去替换Feign默认的Client。
#### 使用HTTPClient替换Feign默认Client

- 引入 httpclient 依赖
```xml
<dependency>
	<groupId>org.apache.httpcomponents</groupId>
	<artifactId>httpclient</artifactId>
</dependency>

<dependency>
	<groupId>com.netflix.feign</groupId>
	<artifactId>feign-httpclient</artifactId>
	<version>8.17.0</version>
</dependency>
```

- 开启httpclient为Feign默认的Client
```yaml
feign:
  httpclient:
      enabled: true
```
#### 使用okhttp替换Feign默认的Client

- 引入 okhttp 依赖
```xml
<dependency>
  <groupId>io.github.openfeign</groupId>
  <artifactId>feign-okhttp</artifactId>
</dependency>
```

- 开启okhttp为Feign默认的Client
```yaml
feign:
    httpclient:
         enabled: false
    okhttp:
         enabled: true
```

- 初始化OkHttpClient
```java
@Configuration
@ConditionalOnClass(Feign.class)
@AutoConfigureBefore(FeignAutoConfiguration.class)
public class FeignOkHttpConfig {
    @Bean
    public okhttp3.OkHttpClient okHttpClient(){
        return new okhttp3.OkHttpClient.Builder()
                 //设置连接超时
                .connectTimeout(60, TimeUnit.SECONDS)
                //设置读超时
                .readTimeout(60, TimeUnit.SECONDS)
                //设置写超时
                .writeTimeout(60,TimeUnit.SECONDS)
                //是否自动重连
                .retryOnConnectionFailure(true)
                .connectionPool(new ConnectionPool())
                //构建OkHttpClient对象
                .build();
    }

}
```
### Feign调用传递Token
在进行认证鉴权的时候，不管是jwt，还是security，当使用Feign时就会发现外部请求到A服务的时候，A服务是可以拿到Token的，然而当服务使用Feign调用B服务时，Token就会丢失，从而认证失败。解决方法相对比较简单，需要做的就是在Feign调用的时候，向请求头里面添加需要传递的Token。

- 实现 RequestInterceptor 接口，在发起请求前给 RequestTemplate 设置请求头
```java
@Component
public class FeignTokenInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate requestTemplate) {
        requestTemplate.header("oauthToken", "请求token");
    }

}
```
### Feign 使用 FastJson 进行序列化
Feign默认使用 Jackson 进行请求参数和响应数据的JSON序列化。

- 指定使用FastJson进行序列化
```java
@Configuration
public class FeignConfig {

    @Bean
    public Encoder feignEncoder() {
        return new SpringEncoder(feignHttpMessageConverter());
    }

    private ObjectFactory<HttpMessageConverters> feignHttpMessageConverter() {
        final HttpMessageConverters httpMessageConverters =
                new HttpMessageConverters(getFastJsonConverter());
        return () -> httpMessageConverters;
    }

    private FastJsonHttpMessageConverter getFastJsonConverter() {
        FastJsonHttpMessageConverter converter = 
                new FastJsonHttpMessageConverter();

        List<MediaType> supportedMediaTypes = new ArrayList<>();
        MediaType mediaTypeJson = 
                MediaType.valueOf(MediaType.APPLICATION_JSON_UTF8_VALUE);
        supportedMediaTypes.add(mediaTypeJson);
        converter.setSupportedMediaTypes(supportedMediaTypes);
        FastJsonConfig config = new FastJsonConfig();
        config.getSerializeConfig()
                .put(Json.class, new SwaggerJsonSerializer());
        config.setSerializerFeatures(
                SerializerFeature.DisableCircularReferenceDetect);
        converter.setFastJsonConfig(config);

        return converter;
    }

}
```
### 解决Feign首次请求失败问题
当Feign和Ribbon整合了Hystrix之后，可能会出现首次调用失败的问题，造成该问题出现的原因分析如下：Hystrix默认的超时时间是1秒，如果超过这个时间尚未做出响应，将会进入fallback代码。由于Bean的装配以及懒加载机制等，Feign首次请求都会比较慢。如果这个响应时间大于1秒，就会出现请求失败的问题。

- 将Hystrix的超时时间改为5秒
```properties
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=5000
```

- 禁用Hystrix的超时时间
```properties
hystrix.command.default.execution.timeout.enabled=false
```

- 使用Feign的时候直接关闭Hystrix，该方式不推荐使用
```properties
feign.hystrix.enabled=false
```
