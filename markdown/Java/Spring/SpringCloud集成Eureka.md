# SpringCloud集成Eureka

> 《重新定义Spring Cloud实战》

## Eureka简介
Eureka是Netflix公司开源的一款服务发现组件，该组件提供的服务发现可以为负载均衡、failover等提供支持。Eureka包括EurekaServer及EurekaClient。EurekaServer提供REST服务，而EurekaClient则是使用Java编写的客户端，用于简化与EurekaServer的交互。
EurekaServer端采用的是P2P的复制模式，但是它不保证复制操作一定能成功，因此它提供的是一个最终一致性的服务实例视图；Client端在Server端的注册信息有一个带期限的租约，一旦Server端在指定期间没有收到Client端发送的心跳，则Server端会认为Client端注册的服务是不健康的，定时任务会将其从注册表中删除。Consul与Eureka不同，Consul采用Raft算法，可以提供强一致性的保证，Consul的agent相当于Netflix Ribbon+Netflix Eureka Client，而且对应用来说相对透明，同时相对于Eureka这种集中式的心跳检测机制，Consul的agent可以参与到基于gossip协议的健康检查，分散了Server端的心跳检测压力。除此之外，Consul为多数据中心提供了开箱即用的原生支持等。
## Hello World Demo
**环境配置**

1. JDK8
2. Maven
### Eureka Server

1. 依赖配置
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

2. 编码
```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

3. 配置
```yaml
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
### Eureka Client

1. 依赖配置
```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

2. 编码
```java
@SpringBootApplication
@EnableDiscoveryClient
public class ClientApplication {
	
    public static void main(String[] args) {
        SpringApplication.run(ClientApplication.class, args);
    }
}
```

3. 配置
```yaml
server:
  port: 7070
spring:
  application:
    name: client-a
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8888/eureka/
  instance:
    prefer-ip-address: true
management:
  security:
    enabled: false
```
### Eureka Admin 管控平台
Eureka Admin 管控平台是Spring Cloud中国社区为Eureka注册中心开源的一个节点监控、服务动态启停的项目。
由于目前尚未提交中央仓库，需下载源码构建！

1. 依赖配置
```xml
<dependency>
  <groupId>cn.springcloud.eureka</groupId>
  <artifactId>eureka-admin-utils</artifactId>
  <version>0.0.1-SNAPSHOT</version>
</dependency>

<dependency>
  <groupId>cn.springcloud.eureka</groupId>
  <artifactId>eureka-admin-view</artifactId>
  <version>0.0.1-SNAPSHOT</version>
</dependency>

<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

2. 编码
```java
@Configuration
public class WebMvcConfiguration extends WebMvcConfigurerAdapter {
	
	private final Logger logger = LoggerFactory.getLogger(WebMvcConfiguration.class);

	@Override
	public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
//		converters.add(new FastJsonHttpMessageConverter4());
	}

	@Override
	public void addInterceptors(InterceptorRegistry registry) {
		logger.info("添加拦截器");
		registry.addInterceptor(new PerformanceInterceptor());
	}

	@Override
	public void addViewControllers(ViewControllerRegistry registry) {
		registry.addRedirectViewController("/", "/eurekaindex.html");
	}
	
}
```
```java
@RestController
@RequestMapping("eureka")
public class EurekaClientController {

	@Resource
	private EurekaClient eurekaClient;
	
	/**
	 * @description 获取服务数量和节点数量
	 */
	@RequestMapping(value = "home", method = RequestMethod.GET)
	public ResultMap home(){
		List<Application> apps = eurekaClient.getApplications().getRegisteredApplications();
		int appCount = apps.size();
		int nodeCount = 0;
		int enableNodeCount = 0;
		for(Application app : apps){
			nodeCount += app.getInstancesAsIsFromEureka().size();
			List<InstanceInfo> instances = app.getInstances();
			for(InstanceInfo instance : instances){
				if(instance.getStatus().name().equals(InstanceStatus.UP.name())){
					enableNodeCount ++;
				}
			}
		}
		return ResultMap.buildSuccess().put("appCount", appCount).put("nodeCount", nodeCount).put("enableNodeCount", enableNodeCount);
	}
	
	/**
	 * @description 获取所有服务节点
	 */
	@RequestMapping(value = "apps", method = RequestMethod.GET)
	public ResultMap apps(){
		List<Application> apps = eurekaClient.getApplications().getRegisteredApplications();
		Collections.sort(apps, new Comparator<Application>() {
	        public int compare(Application l, Application r) {
	            return l.getName().compareTo(r.getName());
	        }
	    });
		for(Application app : apps){
			Collections.sort(app.getInstances(), new Comparator<InstanceInfo>() {
		        public int compare(InstanceInfo l, InstanceInfo r) {
		            return l.getPort() - r.getPort();
		        }
		    });
		}
		return ResultMap.buildSuccess().put("list", apps);
	}
	
	/**
	 * @description 界面请求转到第三方服务进行状态变更
	 */
	@RequestMapping(value = "status/{appName}", method = RequestMethod.POST)
	public ResultMap status(@PathVariable String appName, String instanceId, String status){
		Application application = eurekaClient.getApplication(appName);
		InstanceInfo instanceInfo = application.getByInstanceId(instanceId);
		instanceInfo.setStatus(InstanceStatus.toEnum(status));
		Map<String, String> headers = new HashMap<>();
		headers.put("Content-Type", "text/plain");
//		HttpUtil.post(instanceInfo.getHomePageUrl() + "eureka-admin-client/status", "status=" + status);
		HttpUtil.post(instanceInfo.getHomePageUrl() + "service-registry/instance-status", status, headers);
		
//		List<InstanceInfo> instanceInfos = application.getInstances();
//		for(InstanceInfo item : instanceInfos){
//			HttpUtil.post(item.getHomePageUrl() + "eureka-admin-client/status/" + appName, "instanceId=" + instanceId + "&status=" + status);
//		}
//		Set<String> regions = eurekaClient.getAllKnownRegions();
//		for(String region : regions){
//			Applications applications = eurekaClient.getApplicationsForARegion(region);
//			List<Application> apps = applications.getRegisteredApplications();
//			for(Application app : apps){
//				eurekaClient.getApplications().addApplication(app);
//			}
//		}
		return ResultMap.buildSuccess();
	}
}
```
```java
@EnableEurekaClient
@SpringBootApplication
public class EurekaAdminServer {

	public static void main(String[] args) {
        SpringApplication.run(EurekaAdminServer.class, args);
    }
}
```

3. 配置
```yaml
server:
  port: 8082
eureka:
  server: 
    eviction-interval-timer-in-ms: 30000
  client:
    register-with-eureka: false
    fetch-registry: true
    filterOnlyUpInstances: false
    serviceUrl:
       defaultZone: http://localhost:8888/eureka/
logging:
  level:
    com:
      itopener: DEBUG
    org:
      springframework: INFO
```
### 验证

1. 访问 eureka server 站点 :  [http://localhost:8888/](http://localhost:8888/)

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303161742.png#id=Rq6kQ&originHeight=499&originWidth=1240&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

2. 访问 eureka 管控平台站点 :  [http://localhost:8082/eurekaindex.html](http://localhost:8082/eurekaindex.html)

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303161802.png#id=jalm5&originHeight=230&originWidth=1240&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
