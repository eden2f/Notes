# SpringBoot集成Redis


## Maven 添加 redis 缓存支持
``` java
<!-- 添加 Redis 缓存支持 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
    
## application.properties 配置
``` java
##===========Spring data redis========================================
spring.redis.host=www.huangmp.cn
spring.redis.port=6379
spring.redis.pool.max-active=20
spring.redis.pool.max-wait=200000
spring.redis.pool.max-idle=20
spring.redis.pool.min-idle=1
#默认是索引为0的数据库
spring.redis.database=1 
```

## 添加 RedisConfiguration 配置类
``` java
@Configuration
@EnableCaching
public class RedisConfiguration {

    @Bean
    public JedisConnectionFactory jedisConnectionFactory(){
        JedisConnectionFactory factory = new JedisConnectionFactory();
        factory.setHostName("www.huangmp.cn");
        factory.setPort(6379);
        factory.setDatabase(1);
        factory.setTimeout(60000);
        return factory;
    }

    @Bean
    public CacheManager cacheManager(@SuppressWarnings("rawtypes") RedisTemplate redisTemplate){
        RedisCacheManager manager = new RedisCacheManager(redisTemplate);
        manager.setDefaultExpiration( 1000 * 60 * 60 * 12); // 12小时
        manager.setUsePrefix(true);
        return manager;
    }

}
```

## 添加 redis 对于 User 类的操作类 （dao层对象）
``` java
@Repository
public class UserRedis {
    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    public void add(String key, Long time, User user) {
        redisTemplate.opsForValue().set(key, JsonUtils.objectToJson(user) , time, TimeUnit.MINUTES);
    }

    public void add(String key, Long time, List<User> users) {
        redisTemplate.opsForValue().set(key, JsonUtils.objectToJson(users), time, TimeUnit.MINUTES);
    }
    public User get(String key) {
        User user = null;
        String json = redisTemplate.opsForValue().get(key);
        if(!StringUtils.isEmpty(json))
            user = JsonUtils.jsonToPojo(json, User.class);
        return user;
    }

    public List<User> getList(String key) {
        List<User> ts = null;
        String listJson = redisTemplate.opsForValue().get(key);
        if(!StringUtils.isEmpty(listJson))
            ts = JsonUtils.jsonToList(listJson, User.class);
        return ts;
    }

    public void delete(String key){
        redisTemplate.opsForValue().getOperations().delete(key);
    }
}
```
## 在业务代码 使用 redis dao对象，启用 redis 缓存
``` java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    @Autowired
    private UserRedis userRedis;
    private static final String keyHead = "mysql:get:user:";

    public User findById(Long id) {
        User user = userRedis.get(keyHead + id);
        if(user == null){
            user = userRepository.findOne(id);
            if(user != null)
                userRedis.add(keyHead + id, 30L, user);
        }
        return user;
    }

    public User create(User user) {
        User newUser = userRepository.save(user);
        if(newUser != null)
            userRedis.add(keyHead + newUser.getId(), 30L, newUser);
        return newUser;
    }

    public User update(User user) {
        if(user != null) {
            userRedis.delete(keyHead + user.getId());
            userRedis.add(keyHead + user.getId(), 30L, user);
        }
        return userRepository.save(user);
    }

    public void delete(Long id) {
        userRedis.delete(keyHead + id);
        userRepository.delete(id);
    }

```

启动项目进行测试

#### 相关文档
[Spring Data Redis 官方文档](http://docs.spring.io/spring-data/redis/docs/2.0.0.M2/reference/html/)
