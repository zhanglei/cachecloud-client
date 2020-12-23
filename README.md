[CacheCloud](https://github.com/sohutv/cachecloud)客户端开源项目，包含对原生[jedis](https://github.com/redis/jedis)的封装，指标采集和上报；支持redis cluster部署下客户端pipeline操作；应用双机房部署下客户端双写，熔断保护等功能；以及对redis开源客户端[redisson](https://github.com/redisson/redisson)和[lettuce](https://github.com/lettuce-io/lettuce-core)的适配。

#### 各模块主要功能：

- cachecloud-client-redis：jedis客户端构建器；
- cachecloud-client-redisson：redisson客户端构建器；
- cachecloud-client-lettuce：lettuce客户端构建器；
- cachecloud-jedis: 原生jedis封装，增加支持redis集群部署Pipeline操作的PipelineCluster，支持双机房部署的CrossRoomCluster；依赖cachecloud-client-spectator模块，对jedis命令执行进行埋点，采集命令调用次数、耗时和异常等指标进行上报；
- cachecloud-client-spectator：jedis指标收集和上报模块，及其他工具类；

#### 接入和使用：
### 1. maven坐标

```
<!-- jedis客户端依赖 -->
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.2.0-CC-1-SNAPSHOT</version>
</dependency>
<dependency>
    <groupId>com.sohu.tv</groupId>
    <artifactId>cachecloud-client-redis</artifactId>
    <version>>2.1.0-SNAPSHOT</version>
</dependency>
<!-- lettuce客户端依赖 -->
<dependency>
    <groupId>com.sohu.tv</groupId>
    <artifactId>cachecloud-client-lettuce</artifactId>
    <version>1.0-RELEASE</version>
</dependency>
<!-- redisson客户端依赖 -->
<dependency>
    <groupId>com.sohu.tv</groupId>
    <artifactId>cachecloud-client-redisson</artifactId>
    <version>1.0-RELEASE</version>
</dependency>
```
### 2. cachecloud-client-redis接入方式

**Configuration:**

```
@Configuration
public class RedisConfiguration {

    /**
     * Redis Cluster
     */
    @Bean(destroyMethod = "close")
    public PipelineCluster pipelineCluster(@Value("${cachecloud.demo.appId}") long appId) {
        //默认配置
        PipelineCluster pipelineCluster = ClientBuilder.redisCluster(appId).build();
        return pipelineCluster;
    }

    /**
     * Redis Sentinel
     */
    @Bean(destroyMethod = "destroy")
    public JedisSentinelPool jedisSentinelPool(@Value("${cachecloud.demo.appId}") long appId) {
        //默认配置
        JedisSentinelPool jedisSentinelPool = ClientBuilder.redisSentinel(appId).build();
        return jedisSentinelPool;
    }

    /**
     * Redis Standalone
     */
    @Bean(destroyMethod = "destroy")
    public JedisPool jedisPool(@Value("${cachecloud.demo.appId}") long appId) {
        //默认配置
        JedisPool jedisPool = ClientBuilder.redisStandalone(appId).build();
        return jedisPool;
    }
}


```
**Usage:**
```
@Component
@Slf4j
public class RedisDao {

    @Autowired
    private PipelineCluster pipelineCluster;

    @Autowired
    private JedisSentinelPool jedisSentinelPool;

    @Autowired
    private JedisPool jedisPool;

    public String getFromCluster(String key) {
        String value = pipelineCluster.get(key);
        log.info("value={}", value);
        return value;
    }

    public String getFromSentinel(String key) {
        Jedis jedis = null;
        try {
            jedis = jedisSentinelPool.getResource();
            String value = jedis.get(key);
            log.info("value={}", value);
            return value;
        } catch (Exception e) {
            log.error(e.getMessage(), e);
        } finally {
            if (jedis != null) {
                jedis.close();
            }
        }
        return null;
    }

    public String getFromStandalone(String key) {
        Jedis jedis = null;
        try {
            jedis = jedisPool.getResource();
            String value = jedis.get(key);
            log.info("value={}", value);
            return value;
        } catch (Exception e) {
            log.error(e.getMessage(), e);
        } finally {
            if (jedis != null) {
                jedis.close();
            }
        }
        return null;
    }
}

```


### 3. cachecloud-client-lettuce接入方式

**Configuration:**
```
@Configuration
public class LettuceConfiguration {

    @Bean
    public ClientResources.Builder clientResourcesBuilder() {
        return DefaultClientResources.builder()
                .ioThreadPoolSize(8)
                .computationThreadPoolSize(10);
    }

    @Bean
    public ClusterClientOptions.Builder clusterClientOptionsBuilder() {
        SocketOptions socketOptions = SocketOptions.builder().keepAlive(true).tcpNoDelay(false)
                .connectTimeout(Duration.ofSeconds(5)).build();

        ClusterClientOptions.Builder clientOptionsBuilder = ClusterClientOptions.builder()
                .timeoutOptions(TimeoutOptions.enabled(Duration.ofSeconds(5)))
                .socketOptions(socketOptions);

        return clientOptionsBuilder;
    }

    @Bean(destroyMethod = "shutdown")
    public RedisClusterClient redisClusterClient(@Value("${cachecloud.demo.appId}") long appId,
                                                 @Value("${cachecloud.demo.password}") String password,
                                                 ClientResources.Builder clientResourcesBuilder,
                                                 ClusterClientOptions.Builder clusterClientOptionsBuilder) {

        RedisClusterClient redisClusterClient = LettuceClientBuilder
                .redisCluster(appId, password)
                .setClientResourcesBuilder(clientResourcesBuilder)
                .setClusterClientOptionsBuilder(clusterClientOptionsBuilder)
                .build();

        return redisClusterClient;
    }

    @Bean(destroyMethod = "close")
    public StatefulRedisClusterConnection<String, String> clusterConnection(RedisClusterClient redisClusterClient) {

        StatefulRedisClusterConnection<String, String> connection = redisClusterClient.connect();
        connection.setReadFrom(ReadFrom.REPLICA_PREFERRED);
        return connection;
    }

}

```
**usage:**
```
@Component
@Slf4j
public class RedisDao {

    @Autowired
    private StatefulRedisClusterConnection<String, String> clusterConnection;

    public String get(String key){
        RedisAdvancedClusterCommands<String, String> clusterCommands = clusterConnection.sync();
        String value = clusterCommands.get(key);
        log.info("value={}", value);
        return value;
    }
}
```
