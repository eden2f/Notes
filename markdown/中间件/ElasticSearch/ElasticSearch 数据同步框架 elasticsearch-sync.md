# ElasticSearch 数据同步框架 elasticsearch-sync

## 前言
在上文《Elasticsearch存储设计与MySQL数据同步方案》中提及，常见同步方案有三种。
为什么有这个框架？抱歉，当前应该说是Demo。
一方面，与团队商讨之后，根据当前情况，使用方案二定时Select同步方案落地最为适宜。
另一方面，通过Binlog 实现Elasticsearch与MySQL的数据同步已经有canal开源实现了，没有必要重复造轮子。
## 落地方案
MySQL数据表维护一个业务无关的更新时间，任何更新数据表内容的操作都会使该字段的更新；
生产者定时任务，按一定的时间周期扫描MySQL数据表，把该时间段内发生变化的数据标识（或者主键）push到MQ中；
消费者定时任务，负责消费MQ中的内容，组装数据同步到Elasticsearch中。
MySQL负责业务事物场景的数据存储，而Elasticsearch负责系统的数据检索和数据导出功能。
## 代码仓库

- [Github项目路径 : eden2f/elasticsearch-sync](https://github.com/eden2f/elasticsearch-sync)
## Elasticsearch存储设计
MySQL初始化脚本：mysql-init.sql
ElasticSearch初始化脚本：elasticsearch-init.txt
关系型数据库MySQL的关系表设计：user, user_extend, user_operation_log
MySQL关联关系如下：
```
user : user_extend : user_operation_log = 1 : 1 : n
```
Elasticsearch mapping 如下:
```json
{
    "properties": {
        "userId": {
            "type": "long"
        },
        "name": {
            "type": "keyword"
        },
        "age": {
            "type": "integer"
        },
        "email": {
            "type": "keyword"
        },
        "headPortrait": {
            "type": "text"
        },
        "imgs": {
            "type": "text"
        },
        "userOperationLogs": {
            "properties": {
                "id": {
                    "type": "long"
                },
                "userId": {
                    "type": "long"
                },
                "desc": {
                    "type": "text"
                }
            }
        }
    }
}
```
## 关键项解释（JAVA）
本系统是基于Java语言实现的，本文只展示主要代码，具体实现请从Github拉取
### Jar依赖
| 框架组件 | 版本 |
| --- | --- |
| spring-boot | 2.5.2 |
| mybatis | 3.3.2 |
| elasticsearch-rest-high-level-client | 7.12.1
注意根据Elasticsearch版本挑选，最好跟你的Elasticsearch版本对其，特别注意，别夸版本使用jar！ |
| Elasticsearch | 7.12.1
同上 |
| jedis | 3.6.1 |

### 主要Class解释

- ElasticsearchConfig：相关配置，初始化Elasticsearch操作客户端：RestHighLevelClient；
- BaseEsPo：是Elasticsearch Type的ORM对象基类；
- BaseDao：是对Elasticsearch的Type的curd操作实现的基类；
- BaseProducer：生产者定时任务基类，实现了定时从MySQL指定数据表中拉取更新的数据标识push到MQ中。
- BaseConsumer：消费者定时任务基类，实现了根据MQ中的数据标识组合数据同步到Elasticsearch的能力；
### 主要配置项
```properties
# 生产者配置示例
## [通用] 是否打印生产内容
es.producer.log.enable=0
## 是否启用
UserInfoProducer.enable=1
## 初始更新时间
UserExtendProducer.default.updateTimeStart=0
## 一次最大生产数量
UserExtendProducer.maxSize=1
## 队列名称
UserExtendProducer.queue.redis.key=UserInfoProducer:queue:9
## 上一次处理完成的更新时间
UserExtendProducer.update.time.start.redis.key=UserInfoProducer:updateTimeStart:9

# 消费者配置示例
## [通用] 队列堆积预警开关
es.consumer.queue.size.alarm.enable=0
## [通用] 队列堆积预警阈值，是最大消费数量的倍数
es.consumer.queue.size.threshold.multiple=0
## 是否启用
UserInfoConsumer.enable=1
## 一次消费最大数量
UserInfoConsumer.consumeSize=1
```
## 编码实现（JAVA）
注意：本文仅列出主要代码，具体详情请前往Github查看

- Elasticsearch 配置，初始化：RestHighLevelClient
```java
@Slf4j
@Configuration
public class ElasticsearchConfig {

    @Value("${elasticsearch.host}")
    private String host;

    @Value("${elasticsearch.port}")
    private String port;

    @Value("${elasticsearch.username:}")
    private String username;

    @Value("${elasticsearch.password:}")
    private String password;

    /**
     * 索引名称
     */
    public static final String ES_INDEX_NAME = "user";
    /**
     * 类型名称
     */
    public static final String ES_TYPE_NAME = "_doc";


    @Bean
    public RestHighLevelClient restHighLevelClient() {
        log.info("restHighLevelClient init start, host = {}, port = {}, username = {}, password = {}", host, port, username, password);
        CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        credentialsProvider.setCredentials(AuthScope.ANY, new UsernamePasswordCredentials(username, password));
        return new RestHighLevelClient(RestClient.builder(new HttpHost(host, Integer.parseInt(port), "http"))
                .setHttpClientConfigCallback((HttpAsyncClientBuilder httpAsyncClientBuilder) -> httpAsyncClientBuilder.setDefaultCredentialsProvider(credentialsProvider)));
    }
}
```

- BaseEsPo
```java
@Data
public abstract class BaseEsPo {

    private String esId;

    private Long userId;

    public void checkElseThrow() {
        if (userId == null || userId == 0) {
            throw new RuntimeException("userId不能为空");
        }
    }

}
```

- UserInfo 对应 user表的内容
```java
@Getter
@Setter
@NoArgsConstructor
@ToString(callSuper = true)
public class UserInfo extends BaseEsPo {

    private Long id;
    private String name;
    private Integer age;
    private String email;
}
```

- BaseDao负责Es对象的crud操作
```java
@Slf4j
@Getter
public abstract class BaseDao {

    @Resource
    protected RestHighLevelClient restHighLevelClient;

    protected static final Integer MAX_BULK_SIZE = 100;


    public <T extends BaseEsPo> Optional<T> findByUserId(Long userId, Class<T> clazz) {
        String className = this.getClass().getSimpleName();
        List<T> pos = new ArrayList<>();
        SearchRequest request = new SearchRequest();
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder().query(QueryBuilders.matchPhraseQuery("userId", userId));
        request.source(searchSourceBuilder);
        request.indices(ElasticsearchConfig.ES_INDEX_NAME);
        request.types(ElasticsearchConfig.ES_TYPE_NAME);
        try {
            log.info("className = {}, restHighLevelClient.search, req = {}", className, JSON.toJSONString(request));
            SearchResponse response = restHighLevelClient.search(request, RequestOptions.DEFAULT);
            log.info("className = {}, restHighLevelClient.search, res = {}", className, JSON.toJSONString(response));
            for (SearchHit hit : response.getHits().getHits()) {
                String id = hit.getId();
                T po = JSON.parseObject(hit.getSourceAsString(), clazz);
                po.setEsId(id);
                pos.add(po);
            }
        } catch (IOException e) {
            log.error(String.format("className = %s, es查询异常, userId = %s", className, userId), e);
            throw new RuntimeException(e);
        }
        if (CollectionUtils.isEmpty(pos)) {
            return Optional.empty();
        }
        if (CollectionUtils.size(pos) > 1) {
            log.info("es:存在多条数据:userId:{}", userId);
        }
        return Optional.of(pos.get(0));
    }


    public <T extends BaseEsPo> void insertOrUpdate(T po, Class<T> clazz) {
        String className = this.getClass().getSimpleName();
        log.info("className = {}, insertOrUpdate po = {}", className, JSON.toJSONString(po));
        po.checkElseThrow();
        Optional<T> optional = findByUserId(po.getUserId(), clazz);
        if (optional.isPresent()) {
            T poInEs = optional.get();
            log.info("className = {}, userId = {}, esId = {}", className, po.getUserId(), poInEs.getEsId());
            UpdateRequest request = new UpdateRequest(ElasticsearchConfig.ES_INDEX_NAME, ElasticsearchConfig.ES_TYPE_NAME, poInEs.getEsId());
            request.doc(JSON.toJSONString(po), XContentType.JSON);
            request.fetchSource(true);
            UpdateResponse response;
            try {
                log.info("className = {}, restHighLevelClient.update req = {}", className, JSON.toJSONString(request));
                response = restHighLevelClient.update(request, RequestOptions.DEFAULT);
                log.info("className = {}, restHighLevelClient.update res = {}", className, JSON.toJSONString(response));
            } catch (ElasticsearchException | IOException e) {
                log.info("className = {}, 同步es失败, userId = {}", className, po.getUserId());
                throw new RuntimeException(e);
            }
            if (response != null) {
                if (response.getResult() == DocWriteResponse.Result.CREATED) {
                    log.info("className = {}, 新增文档成功, userId = {}", className, po.getUserId());
                } else if (response.getResult() == DocWriteResponse.Result.UPDATED) {
                    log.info("className = {}, 修改文档成功, userId = {}", className, po.getUserId());
                }
            }
        } else {
            IndexRequest request = new IndexRequest(ElasticsearchConfig.ES_INDEX_NAME, ElasticsearchConfig.ES_TYPE_NAME);
            request.source(JSON.toJSONString(po), XContentType.JSON);
            IndexResponse response;
            try {
                log.info("className = {}, restHighLevelClient.index req = {}", className, JSON.toJSONString(request));
                response = restHighLevelClient.index(request, RequestOptions.DEFAULT);
                log.info("className = {}, restHighLevelClient.index res = {}", className, JSON.toJSONString(response));
            } catch (ElasticsearchException | IOException e) {
                log.info("className = {}, 同步es失败, userId = {}", className, po.getUserId());
                throw new RuntimeException(e);
            }
            if (response != null) {
                if (response.getResult() == DocWriteResponse.Result.CREATED) {
                    log.info("className = {}, 新增文档成功, userId = {}", className, po.getUserId());
                } else if (response.getResult() == DocWriteResponse.Result.UPDATED) {
                    log.info("className = {}, 修改文档成功, userId = {}", className, po.getUserId());
                }
            }
        }
    }

}
```

- UserInfoDao、UserInfoDaoImpl :负责UserInfo的curd操作实现
```java
public interface UserInfoDao {

    /**
     * 新增或更新用户基本信息
     *
     * @param userInfo 用户基本信息
     */
    void insertOrUpdate(UserInfo userInfo);

}

@Slf4j
@Service
public class UserInfoDaoImpl extends BaseDao implements UserInfoDao {


    @Override
    public void insertOrUpdate(UserInfo userInfo) {
        insertOrUpdate(userInfo, UserInfo.class);
    }

}
```

- UserSyncService、BaseUserSyncService、UserInfoSyncServiceImpl：Service层接口与实现
```java
public interface UserSyncService {

    /**
     * 根据userId同步人物画像
     *
     * @param userId userId
     */
    void syncByUserId(Long userId);
}

public abstract class BaseUserSyncService implements UserSyncService {

    @Resource
    protected EsSyncMapper esSyncMapper;
    @Resource
    private RedisUtil redisUtil;

    private static final String REDIS_KEY_PREFIX = "UserSyncService:";

    /**
     * 根据userId同步
     *
     * @param userId userId
     */
    @Override
    public void syncByUserId(Long userId) {
        String redisKey = REDIS_KEY_PREFIX + userId;
        try {
            redisUtil.repetitionRequestLockOrElseThrow(redisKey);
        } catch (Exception e) {
            throw new EsSyncConcurrentLockException(e);
        }
        BaseEsPo po = selectOneByUserId(userId);
        insertOrUpdate(po);
        redisUtil.remove(redisKey);
    }

    /**
     * 根据userId查询一条数据
     *
     * @param userId userId
     * @return po
     */
    protected abstract BaseEsPo selectOneByUserId(Long userId);

    /**
     * 同步数据到es
     *
     * @param po 对象
     */
    protected abstract void insertOrUpdate(BaseEsPo po);
}

@Service
public class UserInfoSyncServiceImpl extends BaseUserSyncService {

    @Resource
    private UserInfoDao userInfoDao;

    @Override
    protected BaseEsPo selectOneByUserId(Long userId) {
        return esSyncMapper.selectUserInfoByUserId(userId);
    }

    @Override
    protected void insertOrUpdate(BaseEsPo po) {
        userInfoDao.insertOrUpdate((UserInfo) po);
    }
}
```

- 生产者定时任务BaseProducer、UserInfoProducer
```java
@Slf4j
public abstract class BaseProducer {

    @Resource
    private EsSyncMapper esSyncMapper;
    @Resource
    private RedisUtil redisUtil;
    @Value("${es.producer.log.enable:0}")
    protected Integer logEnable;


    public void produce(Integer enable, String producerQueueRedisKey, String updateTimeStartRedisKey,
                        String defaultUpdateTimeStart, Integer maxSize, String primaryKeyColumnName,
                        String uniqueColumnName, String tableName, String updateColumnName) {
        long startTime = System.currentTimeMillis();
        log.info("produce start");
        String taskName = this.getClass().getSimpleName();
        if (enable != null && enable == 0) {
            log.info("taskName = {}, produce disable end, cost : {} ms", taskName, System.currentTimeMillis() - startTime);
            return;
        }
        String updateTimeStart = redisUtil.get(updateTimeStartRedisKey);
        if (StringUtils.isBlank(updateTimeStart)) {
            updateTimeStart = defaultUpdateTimeStart;
        }
        String updateTimeEnd = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date());
        Long latestMaxPrimaryKey = 0L;
        List<Long> updatedDataPrimaryKeys;
        do {
            log.info("taskName = {}, primaryKeyColumnName = {}, tableName = {}, updateColumnName = {}, updateTimeStart = {}, updateTimeEnd = {}, latestMaxPrimaryKey = {}, maxSize = {}",
                    taskName, primaryKeyColumnName, tableName, updateColumnName, updateTimeStart, updateTimeEnd, latestMaxPrimaryKey, maxSize);
            updatedDataPrimaryKeys = esSyncMapper.selectUpdatedDataPrimaryKey(primaryKeyColumnName, tableName, updateColumnName, updateTimeStart, updateTimeEnd, latestMaxPrimaryKey, maxSize);
            log.info("taskName = {}, updatedDataPrimaryKeys size = {}", taskName, updatedDataPrimaryKeys.size());
            if (CollectionUtils.isNotEmpty(updatedDataPrimaryKeys)) {
                if (DelStatus.DELETED.getCode() == logEnable) {
                    log.info("updatedDataPrimaryKeys = {}", JSON.toJSONString(updatedDataPrimaryKeys));
                }
                if (CollectionUtils.size(updatedDataPrimaryKeys) >= maxSize) {
                    log.info("更新数量超过阈值, maxSize = {}", maxSize);
                }
                Map<String, Double> scoreMembers = new HashMap<>(updatedDataPrimaryKeys.size());
                List<Map<String, Object>> updatedDatas = esSyncMapper.selectUniqueKeyByPrimaryKey(primaryKeyColumnName, tableName, uniqueColumnName, updateColumnName, updatedDataPrimaryKeys, maxSize);
                if (CollectionUtils.isNotEmpty(updatedDatas)) {
                    for (Map<String, Object> updatedData : updatedDatas) {
                        Object userIdObject = updatedData.get(uniqueColumnName);
                        String userIdString = String.valueOf(userIdObject);
                        Double score = redisUtil.zscore(producerQueueRedisKey, userIdString);
                        if (score != null) {
                            log.info("score != null, userIdString = {}", userIdString);
                        } else {
                            log.info("score == null, userIdString = {}", userIdString);
                            LocalDateTime updateTime = (LocalDateTime) updatedData.get(updateColumnName);
                            Double updateTimeDouble = (double) (updateTime == null ? 0L : updateTime.toEpochSecond(DateUtil.ZONE_OFFSET));
                            scoreMembers.put(userIdString, updateTimeDouble);
                        }
                    }
                    if (!scoreMembers.isEmpty()) {
                        if (DelStatus.DELETED.getCode() == logEnable) {
                            log.info("scoreMembers = {}", JSON.toJSONString(scoreMembers));
                        }
                        redisUtil.zadd(producerQueueRedisKey, scoreMembers);
                    }
                }
                latestMaxPrimaryKey = updatedDataPrimaryKeys.get(updatedDataPrimaryKeys.size() - 1);
            } else {
                latestMaxPrimaryKey = 0L;
            }
        } while (CollectionUtils.size(updatedDataPrimaryKeys) >= maxSize && latestMaxPrimaryKey > 0L);
        redisUtil.set(updateTimeStartRedisKey, updateTimeEnd, 60 * 60 * 24 * 30);
        log.info("taskName = {}, produce end, cost : {} ms", taskName, System.currentTimeMillis() - startTime);
    }
}
```
```java
@Slf4j
@Configuration
@EnableScheduling
public class UserInfoProducer extends BaseProducer {

    @Value("${UserInfoProducer.enable:0}")
    private Integer enable;
    @Value("${UserInfoProducer.default.updateTimeStart:2021-07-09 14:00:00}")
    private String defaultUpdateTimeStart;
    @Value("${UserInfoProducer.maxSize:1}")
    private Integer maxSize;
    @Value("${UserInfoProducer.queue.redis.key:UserInfoProducer:queue:9}")
    private String producerQueueRedisKey;
    @Value("${UserInfoProducer.update.time.start.redis.key:UserInfoProducer:updateTimeStart:9}")
    private String producerUpdateTimeStartRedisKey;

    @Scheduled(initialDelay = 10000, fixedDelayString = "${UserInfoProducer.fixedDelayString:1000}")
    public void produce() {
        try {
            String primaryKeyColumnName = "id";
            String uniqueColumnName = "userId";
            String tableName = "user_extend";
            String updateColumnName = "update_time";
            produce(enable, producerQueueRedisKey, producerUpdateTimeStartRedisKey,
                    defaultUpdateTimeStart, maxSize, primaryKeyColumnName,
                    uniqueColumnName, tableName, updateColumnName);
        } catch (Throwable throwable) {
            log.error(String.format("UserInfoProducer:发生未知异常, e = %s", throwable), throwable);
        }
    }

}
```

- 消费者定时任务BaseConsumer、UserInfoConsumer
```java
@Slf4j
public abstract class BaseConsumer {

    @Resource
    protected RedisUtil redisUtil;
    @Value("${es.consumer.queue.size.alarm.enable:0}")
    protected Integer queueSizeAlarmEnable;
    @Value("${es.consumer.queue.size.threshold.multiple:10}")
    protected Integer queueSizeThresholdMultiple;

    private final UserSyncService userSyncService;

    protected BaseConsumer(UserSyncService userSyncService) {
        this.userSyncService = userSyncService;
    }

    public void consume(Integer enable, Integer consumeSize, String queueName) {
        long startTime = System.currentTimeMillis();
        String taskName = this.getClass().getSimpleName();
        log.info("taskName = {}, consume start, enable = {}, consumeSize = {}, queueName = {}", taskName, enable, consumeSize, queueName);
        if (enable != null && enable == 0) {
            log.info("taskName = {}, consume end, cost : {} ms", taskName, System.currentTimeMillis() - startTime);
            return;
        }
        long currentSecond = LocalDateTime.now().toEpochSecond(ZoneOffset.of("+8"));
        Set<String> uniqueValueString = redisUtil.zrangeByScore(queueName, 0, currentSecond, 0, consumeSize);
        log.info("taskName = {}, userIdStrings size = {}, content = {}", taskName, uniqueValueString.size(), JSON.toJSONString(uniqueValueString));
        uniqueValueString.removeIf(Objects::isNull);
        uniqueValueString.removeIf(item -> Long.parseLong(item) == 0);
        for (String uniqueValue : uniqueValueString) {
            Long userId = null;
            try {
                userId = Long.valueOf(uniqueValue);
                userSyncService.syncByUserId(userId);
                redisUtil.zrem(queueName, uniqueValue);
            } catch (Throwable throwable) {
                if (throwable instanceof EsSyncConcurrentLockException) {
                    double updateTimeDouble = LocalDateTime.now().toEpochSecond(DateUtil.ZONE_OFFSET) + 60;
                    redisUtil.zadd(queueName, updateTimeDouble, uniqueValue);
                    log.info("并发同步ES, userId = {}", userId);
                } else {
                    throw throwable;
                }
            }
        }
        if (DelStatus.DELETED.getCode() == queueSizeAlarmEnable) {
            Long queueSize = redisUtil.zcard(queueName);
            if ((long) queueSizeThresholdMultiple * consumeSize < queueSize) {
                log.info("taskName = {}, 生产堆积, queueName = {}", taskName, queueName);
            }
        }
        log.info("taskName = {}, consume end, cost : {} ms", taskName, System.currentTimeMillis() - startTime);
    }

}
```
```java
@Slf4j
@Configuration
@EnableScheduling
public class UserInfoConsumer extends BaseConsumer {

    @Value("${UserInfoConsumer.enable:0}")
    private Integer enable;
    @Value("${UserInfoConsumer.consumeSize:1}")
    private Integer consumeSize;
    @Value("${UserInfoProducer.queue.redis.key:UserInfoProducer:queue:9}")
    private String producerQueueRedisKey;

    protected UserInfoConsumer(UserInfoSyncServiceImpl userPortraitSyncService) {
        super(userPortraitSyncService);
    }


    @Scheduled(initialDelay = 10000, fixedDelayString = "${UserInfoConsumer.fixedDelayString:1000}")
    public void consume() {
        try {
            super.consume(enable, consumeSize, producerQueueRedisKey);
        } catch (Throwable throwable) {
            log.error(String.format("UserInfoConsumer:发生未知异常, e = %s", throwable), throwable);
        }
    }
}
```
