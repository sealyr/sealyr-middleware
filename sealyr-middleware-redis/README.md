# sealyr-middleware-redis

http://projects.spring.io/spring-data-redis/

Spring Data Redis 是 Spring Data 项目的一部分，是 Spring 提供的基于键值形式的数据存储的解决方案。

当前最新版本是 2.0.6.RELEASE。

## Spring Boot

在 Spring Boot 项目中添加依赖配置：

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

配置属性文件：

```
## 单机模式（不使用则不用开启）
# 连接工厂使用的数据库索引（集群不支持）
spring.redis.database=0
# 主机地址
spring.redis.host=10.7.112.125
# 端口号
spring.redis.port=6379
# 授权密码
spring.redis.password=foobared

## 哨兵模式（不使用则不用开启）
# 服务器名称
spring.redis.sentinel.master=
# 服务节点
spring.redis.sentinel.nodes=

## 集群模式（不使用则不用开启）
# 在群集中执行命令时要遵循的最大重定向数目（可选）
spring.redis.cluster.max-redirects=5
# 集群服务节点（必须）
# 可以写集群全部的节点，也可以只写主节点
spring.redis.cluster.nodes=10.7.112.125:6379,10.7.112.126:6379,10.7.112.127:6379

# Jedis 连接池配置（保持默认）
spring.redis.jedis.pool.max-active=8
spring.redis.jedis.pool.max-idle=8
spring.redis.jedis.pool.min-idle=0

# Lettuce 连接池配置（保持默认）
spring.redis.lettuce.pool.max-active=8
spring.redis.lettuce.pool.max-idle=8
spring.redis.lettuce.pool.min-idle=0
```

说明：如果是集群的只需要开启这两项：

```
# 在群集中执行命令时要遵循的最大重定向数目（可选）
spring.redis.cluster.max-redirects=5
# 集群服务节点（必须）
# 可以写集群全部的节点，也可以只写主节点
spring.redis.cluster.nodes=10.7.112.125:6379,10.7.112.126:6379,10.7.112.127:6379
```

注意：一旦开启了集群模式，那么基于单机的配置就会覆盖。

```
public class Example {

    // inject the actual template
    @Autowired
    private RedisTemplate<String, String> template;

    // inject the template as ListOperations
    // can also inject as Value, Set, ZSet, and HashOperations
    @Resource(name="redisTemplate")
    private ListOperations<String, String> listOps;

    public void addLink(String userId, URL url) {
        listOps.leftPush(userId, url.toExternalForm());
        // or use template directly
        redisTemplate.boundListOps(userId).leftPush(url.toExternalForm());
    }
}
```

## 使用向导

在 Spring Boot 项目中快速集成 Redis 的使用向导。

**1. 环境要求（建议）**

Spring Redis requires Redis 2.6 or above and Java SE 8.0 or above. In terms of language bindings (or connectors), Spring Redis integrates with Jedis and Lettuce, two popular open source Java libraries for Redis.

**2. 在 Spring Boot 项目中添加依赖和属性配置**

（见上文）

**3. 自动配置**

在 Spring Boot 的 autoconfigure 项目中已经添加了 RedisAutoConfiguration 类。

```
@Configuration
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean(name = "redisTemplate")
	public RedisTemplate<Object, Object> redisTemplate(
			RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
		RedisTemplate<Object, Object> template = new RedisTemplate<>();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

	@Bean
	@ConditionalOnMissingBean
	public StringRedisTemplate stringRedisTemplate(
			RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
		StringRedisTemplate template = new StringRedisTemplate();
		template.setConnectionFactory(redisConnectionFactory);
		return template;
	}

}
```

作为开发者，可直接在项目中注入 `RedisTemplate` 或者 `StringRedisTemplate`：

```
@Autowired
private RedisTemplate<String, String> redisTemplate;

@Autowired
private StringRedisTemplate stringRedisTemplate;
```

此处需要注意的是：`RedisTemplate`类默认使用了`JdkSerializationRedisSerializer`作为序列化类，存储为二进制格式。

Spring Data Redis 提供的序列化类：

- `JdkSerializationRedisSerializer`： 作为`RedisCache` 和`RedisTemplate`的默认序列化类。
- `StringRedisSerializer`：作为 `StringRedisTemplate` 的默认序列化类。
- `OxmSerializer`：可为 `Object/XML`提供序列化支持。
- `Jackson2JsonRedisSerializer` or `GenericJackson2JsonRedisSerializer`：可为 `JSON` 提供序列化支持。

同时，也可以自定义实现类，只需实现 `RedisSerializer`接口即可。

**4. 自定义序列化**

由于RedisTemplate使用了`JdkSerializationRedisSerializer`作为序列化类，缺点非常明显：1）性能差 2）存储为二进制格式，通过redis-cli不易查看。如果想要改变RedisTemplate的默认配置，编写自定义配置类即可：

```
@Bean(name = "redisTemplate")
public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory)
    throws UnknownHostException {
  RedisTemplate<Object, Object> template = new RedisTemplate<>();
  template.setConnectionFactory(redisConnectionFactory);
  template.setDefaultSerializer(new StringRedisSerializer());
  return template;
}
```

_注意：Spring Data Redis 提供的JSON序列化类较为复杂，不易使用。想要存储JSON格式数据，可以通过第三方JSON包，先将对象转换为string。然后直接使用StringRedisTemplate接口提供的方法更好。_

**5. 操作接口**

通过操作接口可简化方法调用：

```
// inject the actual template
@Autowired
private RedisTemplate<String, String> template;

// inject the actual template
@Autowired
private StringRedisTemplate stringRedisTemplate;

// inject the template as ListOperations
// can also inject as Value, Set, ZSet, and HashOperations
@Resource(name="redisTemplate")
private ListOperations<String, String> listOps;

public void addLink(String userId, URL url) {
    listOps.leftPush(userId, url.toExternalForm());
    // or use template directly
    redisTemplate.boundListOps(userId).leftPush(url.toExternalForm());
}
```

Key Type Operations:

- GeoOperations
- HashOperations
- HyperLogLogOperations
- ListOperations
- SetOperations
- ValueOperations
- ZSetOperations

Key Bound Operations:

- BoundGeoOperations
- BoundHashOperations
- BoundKeyOperations
- BoundListOperations
- BoundSetOperations
- BoundValueOperations
- BoundZSetOperations

## Redis Messaging/PubSub

Redis 也可以发布/订阅消息。

**1. 发送消息**

```
// send message through connection RedisConnection con = ...
byte[] msg = ...
byte[] channel = ...
con.publish(msg, channel); // send message through RedisTemplate
RedisTemplate template = ...
template.convertAndSend("sealyr.msg", "world");
```

**2. 接收消息**

```
# 1. 定义委托接口（选择任意一个方法即可）
public interface MessageDelegate {
  void handleMessage(String message);
  void handleMessage(Map message); 
  void handleMessage(byte[] message);
  void handleMessage(Serializable message);
  // pass the channel/pattern as well
  void handleMessage(Serializable message, String channel);
 }

# 2. 实现接口
public class DefaultMessageDelegate implements MessageDelegate {
  // implementation elided for clarity...
}

# 3. 创建Bean
@Bean
public MessageDelegate defaultMessageDelegate() {
  return new DefaultMessageDelegate();
}

@Bean
public MessageListener messageListener() {
  return new MessageListenerAdapter(defaultMessageDelegate(), "handleMessage");
}

@Bean
public RedisMessageListenerContainer redisContainer(
    RedisConnectionFactory redisConnectionFactory) {
  RedisMessageListenerContainer container = new RedisMessageListenerContainer();
  container.setConnectionFactory(redisConnectionFactory);
  container.addMessageListener(messageListener(), new ChannelTopic("sealyr.msg"));
  return container;
}
```

## Redis Transaction

Redis 通过 `multi`、`exec` 和 `discard`命令实现简单的事务处理。注意：在Redis中，不要过多的使用事务，以免损失性能。

```
//execute a transaction
List<Object> txResults = redisTemplate.execute(new SessionCallback<List<Object>>() {
  public List<Object> execute(RedisOperations operations) throws DataAccessException {
    operations.multi();
    operations.opsForSet().add("key", "value1");

    // This will contain the results of all ops in the transaction
    return operations.exec();
  }
});
System.out.println("Number of items added to set: " + txResults.get(0));
```

## Pipelining

管道，可批量处理数据。

```
//pop a specified number of items from a queue
List<Object> results = stringRedisTemplate.executePipelined(
  new RedisCallback<Object>() {
    public Object doInRedis(RedisConnection connection) throws DataAccessException {
      StringRedisConnection stringRedisConn = (StringRedisConnection)connection;
      for(int i=0; i< batchSize; i++) {
        stringRedisConn.rPop("myqueue");
      }
    return null;
  }
});
```

## Reactive Redis

响应式。

**1. 创建连接工厂**

```
@Bean
public ReactiveRedisConnectionFactory connectionFactory() {
  return new LettuceConnectionFactory("localhost", 6379);
}
```

**2. 模板配置**

```
@Configuration
class RedisConfiguration {

  @Bean
  ReactiveRedisTemplate<String, String> reactiveRedisTemplate(ReactiveRedisConnectionFactory connectionFactory) {
    return new ReactiveRedisTemplate<>(connectionFactory, RedisSerializationContext.string());
  }
}
```

**3. 使用**

```
public class Example {

  @Autowired
  private ReactiveRedisTemplate<String, String> template;

  public Mono<Long> addLink(String userId, URL url) {
    return template.opsForList().leftPush(userId, url.toExternalForm());
  }
}
```

**4. Key Type Operations**

- ReactiveGeoOperations
- ReactiveHashOperations
- ReactiveHyperLogLogOperations
- ReactiveListOperations
- ReactiveSetOperations
- ReactiveValueOperations
- ReactiveZSetOperations

## Lettuce

Spring Boot 2.0 开始，Redis 客户端驱动由 Jedis 变成了 Lettuce，这是为什么呢？

https://lettuce.io/

Lettuce 的优秀特性如下：

- 基于 netty，支持事件模型
- 支持 同步、异步、响应式的方式
- 可以方便的连接 Redis Sentinel
- 完全支持 Redis Cluster
- SSL 连接
- Streaming API
- CDI 和 Spring 的集成
- 兼容 Java 8 和 9

**重要特性一：多线程共享**

Jedis 是直连模式，在多个线程间共享一个 Jedis 实例时是线程不安全的，如果想要在多线程环境下使用 Jedis，需要使用连接池，每个线程都去拿自己的 Jedis 实例，当连接数量增多时，物理连接成本就较高了。

Lettuce 是基于 netty 的，连接实例可以在多个线程间共享，所以，一个多线程的应用可以使用一个连接实例，而不用担心并发线程的数量。

**重要特性二：异步**

异步的方式可以让我们更好的利用系统资源，而不用浪费线程等待网络或磁盘I/O。

Lettuce 是基于 netty 的，netty 是一个多线程、事件驱动的 I/O 框架，所以 Lettuce 可以帮助我们充分利用异步的优势。

**重要特性三：很好的支持 Redis Cluster**

对 Redis Cluster 的支持包括：

- 支持所有的 Cluster 命令
- 基于哈希槽的命令路由
- 对 cluster 命令的高层抽象
- 在多节点上执行命令
- 根据槽和地址端口直接连接cluster中的节点
- SSL和认证
- cluster 拓扑的更新
- 发布/订阅

**重要特性四：Streaming API**

Redis 中可能会有海量的数据，当你获取一个大的数据集合时，有可能会被撑爆，Lettuce 可以让我们使用流的方式来处理。
