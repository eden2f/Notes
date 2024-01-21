# Jedis客户端HelloWorld

## 单机版

需要把jedis的jar包添加到工程中，如果是maven需要添加jar包的坐标。

``` java

<properties>
    <jedis.version>2.7.2</jedis.version>
</properties>

<!-- Redis客户端 -->
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>${jedis.version}</version>
</dependency>
```

``` java
public class JedisTest {

    @Test
    public void testJedisSingle() {
        //创建一个jedis的对象。
        Jedis jedis = new Jedis("192.168.25.153", 6379);
        //调用jedis对象的方法，方法名称和redis的命令一致。
        jedis.set("key1", "jedis test");
        String string = jedis.get("key1");
        System.out.println(string);
        //关闭jedis。
        jedis.close();
    }
    
    /**
        * 使用连接池
        */
    @Test
    public void testJedisPool() {
        //创建jedis连接池
        JedisPool pool = new JedisPool("192.168.25.153", 6379);
        //从连接池中获得Jedis对象
        Jedis jedis = pool.getResource();
        String string = jedis.get("key1");
        System.out.println(string);
        //关闭jedis对象
        jedis.close();
        pool.close();
    }
}
```


## 集群版

``` java
@Test
public void testJedisCluster() {
    HashSet<HostAndPort> nodes = new HashSet<>();
    nodes.add(new HostAndPort("192.168.25.153", 7001));
    nodes.add(new HostAndPort("192.168.25.153", 7002));
    nodes.add(new HostAndPort("192.168.25.153", 7003));
    nodes.add(new HostAndPort("192.168.25.153", 7004));
    nodes.add(new HostAndPort("192.168.25.153", 7005));
    nodes.add(new HostAndPort("192.168.25.153", 7006));
    
    JedisCluster cluster = new JedisCluster(nodes);
    
    cluster.set("key1", "1000");
    String string = cluster.get("key1");
    System.out.println(string);
    
    cluster.close();
}
```