# Spring集成Jedis


## 单机版整合

### 添加配置文件
        
``` java
<!-- 连接池配置 -->
<bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
	<!-- 最大连接数 -->
	<property name="maxTotal" value="30" />
	<!-- 最大空闲连接数 -->
	<property name="maxIdle" value="10" />
	<!-- 每次释放连接的最大数目 -->
	<property name="numTestsPerEvictionRun" value="1024" />
	<!-- 释放连接的扫描间隔（毫秒） -->
	<property name="timeBetweenEvictionRunsMillis" value="30000" />
	<!-- 连接最小空闲时间 -->
	<property name="minEvictableIdleTimeMillis" value="1800000" />
	<!-- 连接空闲多久后释放, 当空闲时间>该值 且 空闲连接>最大空闲连接数 时直接释放 -->
	<property name="softMinEvictableIdleTimeMillis" value="10000" />
	<!-- 获取连接时的最大等待毫秒数,小于零:阻塞不确定的时间,默认-1 -->
	<property name="maxWaitMillis" value="1500" />
	<!-- 在获取连接的时候检查有效性, 默认false -->
	<property name="testOnBorrow" value="true" />
	<!-- 在空闲时检查有效性, 默认false -->
	<property name="testWhileIdle" value="true" />
	<!-- 连接耗尽时是否阻塞, false报异常,ture阻塞直到超时, 默认true -->
	<property name="blockWhenExhausted" value="false" />
</bean>	

<!-- jedis客户端单机版 -->
<bean id="redisClient" class="redis.clients.jedis.JedisPool">
	<constructor-arg name="host" value="xxxx.xxxx.xxxx.xxxx"></constructor-arg>
	<constructor-arg name="port" value="6379"></constructor-arg>
	<constructor-arg name="poolConfig" ref="jedisPoolConfig"></constructor-arg>
</bean>
```
        
### 验证
``` java
/**
	* 单机版测试 整合spring
	* <p>Title: testSpringJedisSingle</p>
	* <p>Description: </p>
	*/
@Test
public void testSpringJedisSingle() {
	ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:spring/applicationContext-*.xml");
	JedisPool pool = (JedisPool) applicationContext.getBean("redisClient");
	Jedis jedis = pool.getResource();
	String string = jedis.get("key1");
	System.out.println(string);
	jedis.close();
	pool.close();
}
```

## 集群版配置

## 配置内容

``` java
<bean id="redisClient" class="redis.clients.jedis.JedisCluster">
	<constructor-arg name="nodes">
		<set>
			<bean class="redis.clients.jedis.HostAndPort">
				<constructor-arg name="host" value="192.168.x.x"></constructor-arg>
				<constructor-arg name="port" value="7001"></constructor-arg>
			</bean>
			<bean class="redis.clients.jedis.HostAndPort">
				<constructor-arg name="host" value="192.168.x.x"></constructor-arg>
				<constructor-arg name="port" value="7002"></constructor-arg>
			</bean>
			<bean class="redis.clients.jedis.HostAndPort">
				<constructor-arg name="host" value="192.168.xx.xxx"></constructor-arg>
				<constructor-arg name="port" value="7003"></constructor-arg>
			</bean>
			<bean class="redis.clients.jedis.HostAndPort">
				<constructor-arg name="host" value="192.168.xx.xxx"></constructor-arg>
				<constructor-arg name="port" value="7004"></constructor-arg>
			</bean>
			<bean class="redis.clients.jedis.HostAndPort">
				<constructor-arg name="host" value="192.168.xx.xxx"></constructor-arg>
				<constructor-arg name="port" value="7005"></constructor-arg>
			</bean>
			<bean class="redis.clients.jedis.HostAndPort">
				<constructor-arg name="host" value="192.168.xxx.xxx"></constructor-arg>
				<constructor-arg name="port" value="7006"></constructor-arg>
			</bean>
		</set>
	</constructor-arg>
	<constructor-arg name="poolConfig" ref="jedisPoolConfig"></constructor-arg>
</bean>
```
        
### 验证
    
``` java
@Test
public void testSpringJedisCluster() {
	ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:spring/applicationContext-*.xml");
	JedisCluster jedisCluster =  (JedisCluster) applicationContext.getBean("redisClient");
	String string = jedisCluster.get("key1");
	System.out.println(string);
	jedisCluster.close();
}
```
    
## 添加 jedis dao 对象

### 接口
    
``` java
public interface JedisClient {

	String get(String key);
	String set(String key,String value);
	String hget(String hkey, String key);
	long hset(String hkey, String key, String value);
	long incr(String key);
	long expire(String key, int second);
	long ttl(String key);
	long hdel(String key);
	long hdel(String key, String field);

}
```
    
### 单机版实现
    
``` java
public class JedisClientSingle implements JedisClient{

	@Autowired
	private JedisPool jedisPool; 
	
	@Override
	public String get(String key) {
		Jedis jedis = jedisPool.getResource();
		String string = jedis.get(key);
		jedis.close();
		return string;
	}

	@Override
	public String set(String key, String value) {
		Jedis jedis = jedisPool.getResource();
		String string = jedis.set(key, value);
		jedis.close();
		return string;
	}

	@Override
	public String hget(String hkey, String key) {
		Jedis jedis = jedisPool.getResource();
		String string = jedis.hget(hkey, key);
		jedis.close();
		return string;
	}

	@Override
	public long hset(String hkey, String key, String value) {
		Jedis jedis = jedisPool.getResource();
		Long result = jedis.hset(hkey, key, value);
		jedis.close();
		return result;
	}

	@Override
	public long incr(String key) {
		Jedis jedis = jedisPool.getResource();
		Long result = jedis.incr(key);
		jedis.close();
		return result;
	}

	@Override
	public long expire(String key, int second) {
		Jedis jedis = jedisPool.getResource();
		Long result = jedis.expire(key, second);
		jedis.close();
		return result;
	}

	@Override
	public long ttl(String key) {
		Jedis jedis = jedisPool.getResource();
		Long result = jedis.ttl(key);
		jedis.close();
		return result;
	}

	@Override
	public long hdel(String key) {
		Jedis jedis = jedisPool.getResource();
		Long result = jedis.del(key);
		jedis.close();
		return result;
	}

	@Override
	public long hdel(String key, String fields) {
		Jedis jedis = jedisPool.getResource();
		Long result = jedis.hdel(key, fields);
		jedis.close();
		return result;
		
	}

}
```
    
### 集群版实现
    
``` java
public class JedisClientCluster implements JedisClient {

	@Autowired
	private JedisCluster jedisCluster;
	
	@Override
	public String get(String key) {
		return jedisCluster.get(key);
	}

	@Override
	public String set(String key, String value) {
		return jedisCluster.set(key, value);
	}

	@Override
	public String hget(String hkey, String key) {
		return jedisCluster.hget(hkey, key);
	}

	@Override
	public long hset(String hkey, String key, String value) {
		return jedisCluster.hset(hkey, key, value);
	}

	@Override
	public long incr(String key) {
		return jedisCluster.incr(key);
	}

	@Override
	public long expire(String key, int second) {
		return jedisCluster.expire(key, second);
	}

	@Override
	public long ttl(String key) {
		return jedisCluster.ttl(key);
	}

	@Override
	public long hdel(String key) {
		return jedisCluster.del(key);
	}

	@Override
	public long hdel(String key, String field) {
		return jedisCluster.hdel(key, field);
	}

}
```
    
### 配置文件增加 项目启动创建单例的dao对象

#### 单机版
``` java
<bean id="jedisClient" class="com.taotao.rest.dao.impl.JedisClientSingle"/>
```
#### 集群版
``` java
<bean id="jedisCluster" class="com.taotao.rest.dao.impl.JedisClientCluster"/>
```

### 把缓存添加到业务逻辑
    
    ``` java
    @Override
	public List<TbContent> getContentList(long contentCid) {
		//从缓存中取内容 开始
		try {
			String result = jedisClient.hget(INDEX_CONTENT_REDIS_KEY, contentCid + "");
			if (!StringUtils.isBlank(result)) {
				//把字符串转换成list
				List<TbContent> resultList = JsonUtils.jsonToList(result, TbContent.class);
				return resultList;
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
		//从缓存中取内容 结束

        //根据内容分类id查询内容列表
		TbContentExample example = new TbContentExample();
		Criteria criteria = example.createCriteria();
		criteria.andCategoryIdEqualTo(contentCid);
		//执行查询
		List<TbContent> list = contentMapper.selectByExample(example);

		//向缓存中添加内容  开始
		try {
			//把list转换成字符串
			String cacheString = JsonUtils.objectToJson(list);
			jedisClient.hset(INDEX_CONTENT_REDIS_KEY, contentCid + "", cacheString);
		} catch (Exception e) {
			e.printStackTrace();
		}
		//向缓存中添加内容  结束
		
		return list;
	}
    ```
### 缓存同步
    
当后台管理系统，修改内容之后需要通知redis把修改的内容对应的分类id的key删除。