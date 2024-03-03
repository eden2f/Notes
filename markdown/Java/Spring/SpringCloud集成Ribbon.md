# SpringCloud集成Ribbon

> 《重新定义Spring Cloud实战》

## 负载均衡
负载均衡（LoadBalance），即利用特定方式将流量分摊到多个操作单元上的一种手段，它对系统吞吐量与系统处理能力有着质的提升，毫不夸张地说，当今极少有企业没有用到负载均衡器或是负载均衡策略的。

- 服务端负载均衡（硬负载、集中式负载均衡）[例子：Nginx]

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303162718.png#id=ztWxb&originHeight=602&originWidth=978&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

- 客户端负载均衡（软负载、进程内负载均衡）[例子：Ribbon]
## Ribbon简介
Ribbon是一个客户端负载均衡器，它赋予了应用一些支配HTTP与TCP行为的能力，可以得知，这里的客户端负载均衡（许多人称之为后端负载均衡）也是进程内负载均衡的一种。它在SpringCloud生态内是一个不可缺少的组件，少了它，服务便不能横向扩展，这显然是有违云原生12要素的。此外，Feign与Zuul中已经默认集成了Ribbon，在我们的服务之间凡是涉及调用的，都可以集成它并应用，从而使我们的调用链具备良好的伸缩性。
### Hello World Demo
**示例环境**

1. JDK8
2. Maven

**示例简介**

1. 服务1 - 注册中心 eureka服务端
2. 服务2 - 提供接口功能的eureka客户端
3. 服务3 - 服务2的调用者 ribbon
#### 编码实现
**code**

- 服务1 - 注册中心 eureka服务端

maven依赖
```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```
代码
```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```
配置文件
```properties
server:
  port: 8888
eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

- 服务2 - 提供接口功能的eureka客户端

maven依赖
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```
编码
```java
@SpringBootApplication
@EnableDiscoveryClient
public class ClientAApplication {
	
    public static void main(String[] args) {
        SpringApplication.run(ClientAApplication.class, args);
    }
}

@RestController
public class TestController {

	@GetMapping("/add")
	public String add(Integer a, Integer b, HttpServletRequest request){
		return " From Port: "+ request.getServerPort() + ", Result: " + (a + b);
	}
}
```
配置文件
```properties
server:
  port: 7070
spring:
  application:
    name: client-a
eureka:
  client:
    serviceUrl:
      defaultZone: http://${eureka.host:127.0.0.1}:${eureka.port:8888}/eureka/
  instance:
    prefer-ip-address: true
```

- 服务3 - 服务2的调用者 ribbon (可以修改端口启动两个服务)

maven依赖
```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```
编码
```java
@SpringBootApplication
@EnableDiscoveryClient
public class RibbonLoadbalancerApplication {

    public static void main(String[] args) {
        SpringApplication.run(RibbonLoadbalancerApplication.class, args);
    }
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@RestController
public class TestController {
	
    @Autowired
    private RestTemplate restTemplate;

	@GetMapping("/add")
	public String add(Integer a, Integer b) {
		String result = restTemplate
				.getForObject("http://CLIENT-A/add?a=" + a + "&b=" + b, String.class);
		System.out.println(result);
		return result;
	}
}
```
#### 测试验证

- eureka 服务端站点查看服务注册情况是否正常
- 调用 ribbon 客户端服务
```shell
curl -X GET \
  'http://127.0.0.1:7777/add?a=1111&b=2222' \
  -H 'Postman-Token: 08f23b90-de98-4145-9f70-0e50c5caa22a' \
  -H 'cache-control: no-cache'
```

- 第一次触发

![](https://notes.yourui.website/upload/2022/06/image-1655909664234.png#id=YSfEC&originHeight=455&originWidth=1240&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

- 第二次触发

![](https://notes.yourui.website/upload/2022/06/image-1655909675151.png#id=Cjnki&originHeight=439&originWidth=1240&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 实战
### Ribbon负载均衡策略与自定义配置

1.  ribbon支持的负载均衡策略 
| 策略类 | 命名 | 描述 |
| --- | --- | --- |
| RandomRule | 随机策略 | 随机选择server |
| RoundRobinRule（默认） | 轮询策略 | 按顺序循环选择server |
| RetryRule | 重试策略 | 在一个配置时间段内，当选择server不成功，则尝试选择一个可用的server |
| BestAvailableRule | 最低并发策略 | 逐个考察server，如果server断路器打开，则忽略，再选择其中并发连接最低的server |
| AvailabilityFilteringRule | 可用过滤策略 | 过滤掉一直连接失败并被标记为circuit tripped的server，过滤掉哪些高并发连接的server（active connections超过设置的阈值） |
| ResponseTimeWeightedRule | 响应时间加权策略 | 根据server的响应时间分配权重。响应时间越长，权重越低，被选择到的概率就越低；响应时间越短，权重越高，被选择到的概率就越高。这个策略很贴切，中和了各种因素，如：网络、磁盘、IO等，这些因素直接影响着响应时间。 |
| ZoneAvoidanceRule | 区域权衡策略 | 综合判断server所在区域的性能和server的可用性轮询选择server，并且判定一个AWS Zone的运行性能是否可用，剔除不可用的Zone中的所有server |

1.  自定义配置 
   - 全局策略设置
```
@Configuration
public class TestConfiguration {
	
	@Bean
	public IRule ribbonRule(IClientConfig config) {
		return new RandomRule();
	}
}
```

   -  基于注解的策略配置（针对某个源服务进行配置） 
      - 方式一 ： 在@RibbonClients中配置

**注意 : 这里使用@ComponentScan注解的意思是让Spring不去扫描被@AvoidScan注解标记的配置类，因为我们的配置是对单个源服务生效的，所以不能应用于全局，如果不排除，启动就会报错。**
```java
@SpringBootApplication
@EnableDiscoveryClient
@RibbonClients(value = {
		@RibbonClient(name = "client-a", configuration = TestConfiguration.class),
		@RibbonClient(name = "client-b", configuration = TestConfiguration.class)
})
public class RibbonLoadbalancerApplication {

    public static void main(String[] args) {
        SpringApplication.run(RibbonLoadbalancerApplication.class, args);
    }
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```
```
@Configuration
@AvoidScan
public class TestConfiguration {
	
	@Autowired
    IClientConfig config;

	@Bean
	public IRule ribbonRule(IClientConfig config) {
		return new RandomRule();
	}
}
```

      - 方式二
```java
@Configuration
@AvoidScan
public class TestConfiguration {
	
	@Autowired
    IClientConfig config;

	@Bean
	public IRule ribbonRule(IClientConfig config) {
		return new RandomRule();
	}
}
```
```java
@SpringBootApplication
@EnableDiscoveryClient
@RibbonClient(name = "client-a", configuration = TestConfiguration.class)
@ComponentScan(excludeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION, value = {AvoidScan.class})})
public class RibbonLoadbalancerApplication {

    public static void main(String[] args) {
        SpringApplication.run(RibbonLoadbalancerApplication.class, args);
    }
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

- 基于配置文件的策略配置
```properties
clienta.ribbon.NFLoadBalancerRuleClassName=com.netflix.loadbalancer.RandomRule
```
### Ribbon超时与重试
```yaml
client-a:
  ribbon:
    ConnectTimeout: 3000
    ReadTimeout: 60000
    MaxAutoRetries: 1 #对第一次请求的服务的重试次数
    MaxAutoRetriesNextServer: 1 #要重试的下一个服务的最大数量（不包括第一个服务）
    OkToRetryOnAllOperations: true
```
### Ribbon的饥饿加载
Ribbon在进行客户端负载均衡的时候并不是在启动时就加载上下文，而是在实际请求的时候才去创建，因此这个特性往往会让我们的第一次调用显得颇为疲软乏力，严重的时候会引起调用超时。所以我们可以通过指定Ribbon具体的客户端的名称来开启饥饿加载，即在启动的时候便加载所有配置项的应用程序上下文。
```yaml
ribbon:
  eager-load:
    enabled: true
    clients: client-a, client-b, client-c
```
### 利用配置文件自定义Ribbon客户端
使用配置文件来指定一些默认加载类，从而更改Ribbon客户端的行为方式，并且使用这种方式优先级最高，优先级高于使用注解@RibbonClient指定的配置和源码中加载的相关Bean。
```yaml
client:
  ribbon:
    NIWSServerListClassName: com.netflix.loadbalancer.ConfigurationBasedServerList
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule
```
