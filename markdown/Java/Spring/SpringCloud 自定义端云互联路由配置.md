# SpringCloud 自定义端云互联路由配置

## 前言
在日常工作中，公司测试环境的服务集群和开发者本机所在的集群并不是同一个。假如集群内有A、B、C、D四个服务，而某功能P需要A、B、C三个服务共同完成，如下图所示 :
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303163948.png#id=rDxfE&originHeight=412&originWidth=1190&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在这个场景下，你更新了A服务的代码，如果你想在本地进行集成测试功能P是否ok，那就需要你在本地部署A、B、C服务才能完成功能P的集成测试；尽管你可能只是改动了极少的代码，但想测试起来还真得非不小的精力（可能B、C这两服务并不是你维护的，你对其也不熟悉，部署过程中遇到各种问题不能快速解决）
## 将本地服务注册到"云"上
有没有其他的解决方式呢？我也很好奇。这里有另一种解决方式：将开发者本机的服务A注册到公司测试环境的服务中去。如下图所示 :
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303164012.png#id=IbQaQ&originHeight=592&originWidth=1240&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
如果你本地更新的是服务B呢？情况是如下图所示 :
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303164027.png#id=G41ci&originHeight=619&originWidth=1240&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
用这种方式，那在本地做集成测试的时候，你改了啥服务，就部署啥服务，其他的服务你都无需关心；当然，测试环境的请求也会打到你的机器来，如果这个时候，你已更新的服务还有Bug，就会影响测试环境的功能P了。所以，这个时候就需要我们根据自己的环境做下评估，如何取舍，当然，这并不是本文的主题。
我就当你默许这种解决方式了，那进入下一个环节，如何实现？
## Spring Cloud 实现方式

- 服务依赖 
   - Spring Cloud 环境
   - 使用 Ribbon 做负载均衡
   - 使用 Feign 做为服务间调用的客户端实现(非必须)
### gradle依赖配置
```groovy
compileOnly 'org.springframework.boot:spring-boot-autoconfigure:2.1.12.RELEASE'
compileOnly 'org.springframework:spring-context:5.1.13.RELEASE'
compileOnly 'org.springframework.cloud:spring-cloud-starter-openfeign:2.2.1.RELEASE'
compileOnly 'org.springframework.cloud:spring-cloud-starter-netflix-ribbon:2.2.1.RELEASE'
compile group: 'org.projectlombok', name: 'lombok', version: '1.18.10'
annotationProcessor 'org.projectlombok:lombok:1.18.8'
```
### 自定义路由策略
检查被调服务是有本地环境提供的实例，如果有就选取本地开发环境的实例作为调用目标；
```java
@Slf4j
@AllArgsConstructor
@NoArgsConstructor
public class DevRibbonRule extends AbstractLoadBalancerRule {


    /**
     * 本地IP集合,可以只写ip前缀,例如:112.95
     */
    private List<String> localIpList;


    /**
     * Randomly choose from all living servers
     */
    @edu.umd.cs.findbugs.annotations.SuppressWarnings(value = "RCN_REDUNDANT_NULLCHECK_OF_NULL_VALUE")
    public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            return null;
        }
        Server server = null;

        while (server == null) {
            if (Thread.interrupted()) {
                return null;
            }
            List<Server> upList = lb.getReachableServers();
            List<Server> allList = lb.getAllServers();

            int serverCount = allList.size();
            if (serverCount == 0) {
                /*
                 * No servers. End regardless of pass, because subsequent passes
                 * only get more restrictive.
                 */
                return null;
            }

            Optional<Server> localOptional = localRandomOne(upList);
            if (localOptional.isPresent()) {
                server = localOptional.get();
            } else {
                Optional<Server> randomOptional = nonLocalRandomOne(upList);
                if (randomOptional.isPresent()) {
                    server = randomOptional.get();
                }
            }


            if (server == null) {
                /*
                 * The only time this should happen is if the server list were
                 * somehow trimmed. This is a transient condition. Retry after
                 * yielding.
                 */
                Thread.yield();
                continue;
            }

            if (server.isAlive()) {
                return (server);
            }

            // Shouldn't actually happen.. but must be transient or a bug.
            server = null;
            Thread.yield();
        }

        return server;

    }


    /**
     * 从所有实例中随机取一个开发者本机的实例
     */
    private Optional<Server> localRandomOne(List<Server> allValidServers) {
        List<Server> localServerList = getLocalServer(allValidServers);
        if (CollectionUtils.isEmpty(localServerList)) {
            return Optional.empty();
        }
        int randomIndex = chooseRandomInt(localServerList.size());
        return Optional.of(localServerList.get(randomIndex));
    }


    /**
     * 从所有实例中排除掉开发者本机的实例后随机取一个
     */
    private Optional<Server> nonLocalRandomOne(List<Server> allValidServers) {
        List<Server> localServerList = getLocalServer(allValidServers);
        List<Server> nonLocalServerList = allValidServers.stream().filter(item -> !localServerList.contains(item)).collect(Collectors.toList());
        if (CollectionUtils.isEmpty(nonLocalServerList)) {
            return Optional.empty();
        }
        int randomIndex = chooseRandomInt(nonLocalServerList.size());
        return Optional.of(nonLocalServerList.get(randomIndex));
    }


    private List<Server> getLocalServer(List<Server> allServers) {
        return allServers.stream().filter(serverItem -> {
            String hostItem = serverItem.getHost();
            for (String localIpItem : localIpList) {
                if (hostItem.contains(localIpItem)) {
                    return true;
                }
            }
            return false;
        }).collect(Collectors.toList());
    }


    protected int chooseRandomInt(int serverCount) {
        return ThreadLocalRandom.current().nextInt(serverCount);
    }


    @Override
    public Server choose(Object key) {
        Server currentServer = choose(getLoadBalancer(), key);
        log.info("本次调用的实例 ：Server : {}, key = {}, host : {}, port = {}", currentServer.toString(), key, currentServer.getHost(), currentServer.getPort());
        return currentServer;
    }

    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {
        // TODO Auto-generated method stub

    }
}
```
### Feign统一配置
测试环境可以通过配置 dev.ribbon.rule.enable = true 使上面的自定义路由策略生效
通过 dev.ribbon.rule.host.list 设置 开发者本地ip
```java
@Configuration
@Slf4j
public class FeignConfiguration {

    @Bean
    Logger.Level feignLoggerLevel() {
        //这里记录所有，根据实际情况选择合适的日志level
        return Logger.Level.FULL;
    }


    @ConditionalOnProperty(value = "dev.ribbon.rule.enable")
    @Bean
    public IRule ribbonRule(@Value("#{'${dev.ribbon.rule.host.list:}'.split(',')}") List<String> localIpList) {
        log.info("localIpList = {}", localIpList);
        return new DevRibbonRule(localIpList);
    }
}
```
### 使用 springboot 的扩展机制 spring factories 使上面的配置生效
我这边是想要把这个路由逻辑封装到jar中,需要的服务直接应用jar就可以了,无需编码;如果你没有这个需求,那么下面两个文件是不需要的 
```java
@Configuration(proxyBeanMethods = false)
@Import(FeignConfiguration.class)
public class CommonFeignAutoConfiguration {
}
```
```properties
# 需放在 META-INF 文件夹中, 文件名为 : spring.factories
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.zuopeng.common.feign.CommonFeignAutoConfiguration
```
