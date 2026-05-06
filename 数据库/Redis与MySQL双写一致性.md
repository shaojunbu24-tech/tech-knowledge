# MySQL 和 Redis 的数据一致性解决方案

> 分类：数据库
> 标签：Redis, MySQL, 双写一致性, 缓存更新, 延迟双删, Canal, 消息队列, 分布式事务, 最终一致性

## 面试精答

> 适合面试时口述的简洁版本（2-3分钟能说完的要点）

MySQL 和 Redis 一致性的本质是**两个独立的存储系统之间如何保持数据同步**。因为 Redis 是内存缓存、MySQL 是磁盘数据库，中间没有事务保证，所以一定会出现不一致的窗口。核心思路不是追求"强一致"，而是**选择适合业务场景的一致性级别**。

**三种一致性级别对应的方案**：

1. **弱一致性（允许短暂不一致）** — Cache Aside 模式：先更新 MySQL，再删除 Redis 缓存。如果删除失败，缓存里就是旧数据，靠 TTL 过期兜底。适合用户画像、文章阅读数等场景。

2. **最终一致性（保证最终一定一致）** — 在 Cache Aside 基础上加保障：删除缓存失败时发 MQ 消息重试，或者用 Canal 监听 MySQL binlog 自动更新 Redis。适合订单状态、商品价格等场景。

3. **强一致性（任何时刻都必须一致）** — 读 Redis 失败后加分布式锁读 MySQL 并回填，写操作加分布式锁同时更新两个存储。性能差，只适合金融余额等极端场景。大多数业务不应该用这个级别。

**生产环境最常见的方案**：**Cache Aside + 延迟双删 + Canal 订阅 binlog 兜底**。三层保障，第一层正常删除缓存，第二层防并发导致的脏数据，第三层即使前两层都失败也能通过 binlog 修复。

**一句话总结**：一致性不是非黑即白的选择题，而是**根据业务容忍度选方案、用多层保障对冲各种异常**。

## 原理深入详解

> 深入底层原理，理解"为什么"

### 一、先搞清楚为什么会产生不一致

Redis 和 MySQL 是两个完全独立的存储系统，没有任何内置的同步机制。

```
正常情况下的数据流：

  写入：应用 → MySQL（主存储）→ Redis（缓存）
  读取：应用 → Redis（命中则返回）
                  ↓ 未命中
                MySQL → 返回数据 + 写入 Redis

问题出在"写入"这一步——MySQL 和 Redis 的更新不是原子操作：

  步骤 1：UPDATE MySQL    → 成功（耗时 5ms）
  步骤 2：DELETE Redis    → 失败（网络抖动 / Redis 重启 / 进程崩溃）

  结果：MySQL 是新数据，Redis 还是旧数据
        后续所有读请求都从 Redis 拿到旧数据
        直到缓存 TTL 过期才恢复

  即使两步都成功，中间也有时间窗口：

  T1: UPDATE MySQL 成功
  T2: ← 这个间隙里如果有读请求，会从 Redis 拿到旧数据
  T3: DELETE Redis 成功

  所以不一致是不可避免的，问题是"不一致多久"以及"怎么快速恢复"
```

### 二、四种缓存更新策略的原理与对比

在讨论一致性方案之前，必须理解四种基础的缓存更新模式。

#### 2.1 Cache Aside（旁路缓存）— 最常用

```
读操作：
  1. 先读 Redis
  2. 命中 → 直接返回
  3. 未命中 → 读 MySQL → 写入 Redis → 返回

写操作：
  1. 先更新 MySQL
  2. 再删除 Redis 缓存（注意是删除，不是更新）

为什么是"删除"而不是"更新"缓存？
  场景：并发写 + 并发读

  如果用"更新缓存"：
    T1: 线程 A 更新 MySQL price = 100
    T2: 线程 B 更新 MySQL price = 200
    T3: 线程 B 更新 Redis price = 200
    T4: 线程 A 更新 Redis price = 100  ← 后发先至！
    结果：MySQL = 200，Redis = 100 → 不一致

  如果用"删除缓存"：
    T1: 线程 A 更新 MySQL price = 100
    T2: 线程 B 更新 MySQL price = 200
    T3: 线程 A 删除 Redis price
    T4: 线程 B 删除 Redis price
    T5: 线程 C 读 Redis → 未命中 → 读 MySQL = 200 → 写入 Redis
    结果：最终一致（中间有短暂不一致窗口）
```

#### 2.2 Read/Write Through（读穿/写穿）

```
应用不直接操作 MySQL 和 Redis，而是通过一个"缓存代理层"：

  读操作：
    应用 → 缓存代理层
      → 缓存有数据 → 直接返回
      → 缓存没数据 → 代理层自动从 MySQL 加载 → 写入缓存 → 返回

  写操作：
    应用 → 缓存代理层
      → 更新缓存 → 异步更新 MySQL

  应用代码更简单，不用关心缓存逻辑
  但需要实现一个缓存代理组件（或用现成的如 Spring Cache）

  典型实现：
    Spring Cache + Redis + MySQL
    @Cacheable / @CachePut / @CacheEvict 注解
```

#### 2.3 Write Behind（异步回写）

```
写操作只更新缓存，不立即写 MySQL：
  应用 → 更新 Redis → 返回成功
  后台线程定期批量把 Redis 的变更同步到 MySQL

  优点：
    写入性能极高（只写内存）
    可以批量合并写入（多次更新同一 key 只写一次 MySQL）

  缺点：
    一致性最差（Redis 和 MySQL 可能差几分钟甚至几小时）
    如果 Redis 宕机，未同步到 MySQL 的数据丢失

  适用场景：浏览量计数、点赞数等写多读少且允许丢失的数据
```

#### 2.4 Write Ahead（写穿透）

```
写操作先更新 MySQL，再更新 Redis：

  应用 → UPDATE MySQL → UPDATE Redis → 返回

  问题：并发场景下和"更新缓存"一样有后发先至的问题
  实际生产中很少单独使用
```

### 三、Cache Aside 的并发问题与解决方案

Cache Aside 是最常用的模式，但它有两个经典的并发问题。

#### 问题 1：先删缓存还是先更新数据库？

```
方案 A：先删缓存，再更新数据库

  T1: 线程 A DELETE Redis（price 缓存被删除）
  T2: 线程 B 读 Redis → 未命中 → 读 MySQL（此时还是旧值 50）
  T3: 线程 B 写 Redis price = 50
  T4: 线程 A UPDATE MySQL price = 100

  结果：MySQL = 100，Redis = 50
  → 不一致，且会持续到缓存 TTL 过期

方案 B：先更新数据库，再删缓存（推荐）

  T1: 线程 A UPDATE MySQL price = 100
  T2: 线程 B 读 Redis → 命中旧值 → 返回 50
  T3: 线程 A DELETE Redis

  结果：不一致窗口极短（T1-T3 之间）
        T3 之后读请求会 miss → 读 MySQL 新值 → 回填

  但方案 B 也有极端并发问题：
  T1: 线程 B 读 Redis → 未命中（缓存刚好过期）
  T2: 线程 B 读 MySQL → 拿到旧值 50
  T3: 线程 A UPDATE MySQL price = 100
  T4: 线程 A DELETE Redis
  T5: 线程 B 写 Redis price = 50  ← 线程 B 的旧值写入了

  这个场景发生条件极其苛刻：
    1. 缓存刚好过期
    2. 读操作查 MySQL 和写操作更新 MySQL 几乎同时发生
    3. 读操作的数据库查询比写操作慢（通常不可能）

  实际上这种"读数据库比写数据库慢"的概率极低
  所以方案 B（先更新数据库再删缓存）是推荐的
```

#### 问题 2：删缓存失败怎么办？

```
方案 B 的关键假设：DELETE Redis 成功

  如果 DELETE 失败呢？

  T1: UPDATE MySQL price = 100  ✓
  T2: DELETE Redis              ✗（Redis 网络超时）
  → MySQL = 100，Redis = 50，永久不一致

  解决方案：延迟双删
```

### 四、延迟双删

```
延迟双删的流程：

  T1: DELETE Redis               ← 第一次删除
  T2: UPDATE MySQL
  T3: sleep(N 毫秒)              ← 等一段时间
  T4: DELETE Redis               ← 第二次删除

为什么需要第二次删除？

  第一次删除后，更新 MySQL 的过程中：
    可能有读请求发现缓存 miss → 读 MySQL（还是旧值）→ 回填 Redis
    → 第二次删除把这些"脏回填"清掉

  sleep 的时间 N 怎么定？
    N = 读操作（查 MySQL + 写 Redis）的典型耗时 + 几十毫秒缓冲
    通常设 500ms - 2s

  问题：sleep 会占用当前线程，高并发下线程池会被 sleep 占满

  解决：不用 sleep，用延迟消息

  方案 1：发一个延迟 MQ 消息
    UPDATE MySQL → DELETE Redis → 发 MQ 延迟消息（500ms 后投递）
    → 消费者收到消息后执行第二次 DELETE

  方案 2：用 Java 的 ScheduledExecutor
    UPDATE MySQL → DELETE Redis
    → schedule(() -> DELETE Redis, 500, TimeUnit.MILLISECONDS)
```

### 五、Canal 监听 binlog — 生产环境的核心方案

延迟双删仍然可能失败（MQ 消息丢了、进程在 sleep 前崩溃）。**Canal 是生产环境中一致性保障的终极手段**。

#### 5.1 什么是 binlog

```
MySQL 的 binlog（二进制日志）记录了所有数据变更操作：
  INSERT、UPDATE、DELETE 的每一行变更

  binlog 的三种格式：
    STATEMENT：记录 SQL 语句（"UPDATE user SET age=20 WHERE id=1"）
    ROW：记录行变更（"user 表 id=1 的 age 从 18 变为 20"）← 推荐
    MIXED：混合模式

  为什么用 ROW 格式？
    STATEMENT 格式在有些场景下会导致主从不一致
    （比如 UPDATE ... WHERE create_time > NOW()，每次执行结果不同）
    ROW 格式记录的是确定的数据变更，不会有歧义

  配置：
    [mysqld]
    log-bin=mysql-bin
    binlog-format=ROW
    server-id=1
```

#### 5.2 Canal 的工作原理

```
Canal 伪装成 MySQL 的从库（Slave），接收 binlog 推送：

  MySQL Master
      │
      │ binlog dump（Canal 伪装成 Slave 请求 binlog）
      ↓
  Canal Server
      │
      │ 解析 binlog → 转成结构化数据
      ↓
  Canal Client（你的应用）
      │
      │ 收到数据变更事件
      ↓
  更新 Redis / 删除 Redis 缓存

Canal 解析后的数据结构示例：
  {
    "table": "user",
    "type": "UPDATE",
    "data": [
      {"id": "1", "name": "张三", "age": "20"}       ← 变更后
    ],
    "old": [
      {"age": "18"}                                    ← 变更前
    ]
  }

  应用收到这个事件后：
    知道 user 表 id=1 的数据变了
    → DELETE Redis key: user:1
    → 或者根据 data 重新构建缓存
```

#### 5.3 Canal 的部署架构

```
生产环境 Canal 部署：

  ┌─────────────┐
  │ MySQL       │
  │ Master      │
  └──────┬──────┘
         │ binlog
  ┌──────▼──────┐
  │ Canal       │     ← 通常部署为独立服务，高可用用 ZooKeeper
  │ Server      │        多个 Canal 实例选 leader
  └──────┬──────┘
         │
  ┌──────▼──────────┐
  │ MQ（Kafka/RMQ） │  ← Canal 把变更事件写入 MQ
  └──────┬──────────┘     解耦：Canal 和业务应用不直接耦合
         │
  ┌──────▼──────┐
  │ 消费者服务   │     ← 你的业务应用消费 MQ 消息
  │ 处理变更事件 │        根据表名和操作类型更新 Redis
  └─────────────┘

为什么不直接消费 Canal 的事件？
  1. 解耦 — Canal 挂了不影响 MySQL，消费者挂了不影响 Canal
  2. 可靠性 — MQ 保证消息不丢，消费者可以重试
  3. 扩展性 — 多个消费者可以订阅同一个 binlog 变更
     （比如一个更新 Redis，一个更新 ES，一个发通知）
```

#### 5.4 Canal 的代码实现（生产级）

```java
// Canal 消费者的核心逻辑（Spring Boot + RocketMQ）

@Component
@RocketMQMessageListener(topic = "canal-topic", consumerGroup = "cache-sync-group")
public class CanalCacheSyncListener implements RocketMQListener<CanalMessage> {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    // 需要同步缓存的表配置
    private static final Map<String, String> TABLE_CACHE_MAPPING = Map.of(
        "user", "user:",           // user 表 → user:{id}
        "product", "product:",     // product 表 → product:{id}
        "order", "order:"          // order 表 → order:{id}
    );

    @Override
    public void onMessage(CanalMessage message) {
        String table = message.getTable();
        String cachePrefix = TABLE_CACHE_MAPPING.get(table);

        if (cachePrefix == null) {
            return; // 不需要同步的表，直接跳过
        }

        for (CanalRowData row : message.getData()) {
            String id = row.get("id");
            String cacheKey = cachePrefix + id;

            switch (message.getType()) {
                case "INSERT":
                case "UPDATE":
                    // 方案 1：删除缓存，等下次读时回填
                    redisTemplate.delete(cacheKey);

                    // 方案 2：直接更新缓存（适合热点数据，减少 miss）
                    // String json = JSON.toJSONString(row);
                    // redisTemplate.opsForValue().set(cacheKey, json, 30, TimeUnit.MINUTES);
                    break;

                case "DELETE":
                    redisTemplate.delete(cacheKey);
                    break;
            }
        }
    }
}
```

### 六、完整的分层一致性方案

生产环境不是选一种方案，而是**多层保障叠加**。

```
┌─────────────────────────────────────────────────────┐
│                   写请求进入                          │
└───────────────────────┬─────────────────────────────┘
                        │
          ┌─────────────▼──────────────┐
          │  第 1 层：Cache Aside       │
          │  UPDATE MySQL               │
          │  DELETE Redis               │  ← 正常情况下这就够了
          │  成功率：99.9%              │
          └─────────────┬──────────────┘
                   失败 ↓
          ┌─────────────▼──────────────┐
          │  第 2 层：MQ 重试           │
          │  DELETE 失败 → 发 MQ 消息   │
          │  消费者重试删除缓存          │  ← 应对 Redis 短暂不可用
          │  最多重试 3 次              │
          └─────────────┬──────────────┘
                   仍失败 ↓
          ┌─────────────▼──────────────┐
          │  第 3 层：Canal binlog 兜底 │
          │  监听 MySQL binlog 变更      │
          │  自动同步到 Redis            │  ← 应对应用层各种异常
          │  延迟：1-5 秒              │
          └─────────────┬──────────────┘
                   超长时间不一致 ↓
          ┌─────────────▼──────────────┐
          │  第 4 层：TTL 过期兜底      │
          │  所有缓存都设 TTL           │
          │  即使上面全失败             │  ← 最终保底
          │  TTL 过期后也会重新从 DB 加载│
          └────────────────────────────┘
```

### 七、不同业务场景的方案选择

```
根据业务特性选一致性级别，不要一刀切：

场景 1：用户资料（昵称、头像）
  特点：修改频率低、不一致影响小、用户改了头像几秒后才看到也能接受
  方案：Cache Aside + TTL
  实现：UPDATE MySQL → DELETE Redis（失败不管，靠 TTL 兜底）
  TTL：30 分钟
  一致性：弱一致（不一致窗口 < 30 分钟）

场景 2：商品价格
  特点：修改频率中、不一致会导致用户投诉（标价 100 实扣 200）
  方案：Cache Aside + MQ 重试 + Canal
  实现：UPDATE MySQL → DELETE Redis → 失败发 MQ → Canal 兜底
  TTL：5 分钟
  一致性：最终一致（不一致窗口 < 5 秒）

场景 3：商品库存（秒杀）
  特点：修改频率极高、超卖是事故
  方案：Redis 作为主存储 + 异步落 MySQL（上一题的方案）
  实现：Lua 原子扣减 Redis → MQ → MySQL
  一致性：Redis 为准，MySQL 追 Redis
  （这里反过来，Redis 是主、MySQL 是从）

场景 4：账户余额
  特点：强一致要求、钱不能错
  方案：直接读 MySQL + 分布式锁 + Redis 只做只读缓存
  实现：写操作加分布式锁，同时更新 MySQL 和 Redis
        读操作先读 Redis，miss 了加锁读 MySQL 回填
  TTL：不设 TTL 或设很短（10 秒）
  一致性：强一致（但有性能代价）
  实际上这种场景通常不用 Redis 缓存余额，直接查 MySQL

场景 5：文章阅读数 / 点赞数
  特点：写极高、不需要精确一致
  方案：Write Behind + 定期批量同步
  实现：INCR Redis → 每分钟批量写 MySQL
        Redis 挂了丢几分钟数据可以接受
  一致性：弱一致（允许丢失）
```

### 八、生产环境的监控与告警

```
一致性相关的监控指标：

1. 缓存命中率
   命中率 = Redis 命中次数 / 总读取次数

   正常：90%+（取决于业务）
   告警：< 70%

   命中率骤降的可能原因：
     - 大量 key 过期（缓存雪崩）
     - 缓存删除后没回填（一致性问题）
     - 热点 key 被意外删除

2. 一致性校验延迟
   定期任务：抽样对比 MySQL 和 Redis 的数据

   对账任务实现：
     每 5 分钟随机抽取 100 个 key
     对比 Redis 值和 MySQL 值
     不一致率 > 1% → 告警

3. Canal 消费延迟
     Canal 消费的 binlog 位点 vs MySQL 当前的 binlog 位点
     延迟 > 10 秒 → 告警
     延迟 > 60 秒 → 严重告警

4. MQ 重试次数
     缓存删除失败后的 MQ 重试
     重试超过 3 次 → 告警 + 记录到补偿表

5. Redis 操作失败率
     DELETE / SET 操作的失败率
     失败率 > 0.1% → 告警（检查 Redis 健康状态）
```

```
监控实现（Spring Boot + Prometheus）：

  // 自定义指标
  @Bean
  MeterRegistry meterRegistry() {
      return new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
  }

  // 记录缓存操作结果
  Counter cacheHitCounter = Counter.builder("cache.operation")
      .tag("result", "hit").register(registry);
  Counter cacheMissCounter = Counter.builder("cache.operation")
      .tag("result", "miss").register(registry);
  Counter cacheDeleteFailCounter = Counter.builder("cache.operation")
      .tag("operation", "delete").tag("result", "fail").register(registry);

  // 定时对账任务
  @Scheduled(fixedRate = 300000) // 5 分钟
  public void consistencyCheck() {
      List<String> sampleKeys = randomSampleKeys(100);
      int inconsistent = 0;

      for (String key : sampleKeys) {
          String redisValue = redisTemplate.opsForValue().get(key);
          String mysqlValue = queryFromMySQL(parseId(key));
          if (!Objects.equals(redisValue, mysqlValue)) {
              inconsistent++;
              // 自动修复：以 MySQL 为准更新 Redis
              redisTemplate.opsForValue().set(key, mysqlValue, 30, TimeUnit.MINUTES);
          }
      }

      double inconsistentRate = inconsistent / 100.0;
      gauge.builder("cache.consistency.inconsistent_rate", () -> inconsistentRate);
  }
```

### 九、Spring Cache 的生产级配置

```
Spring Cache 是 Java 项目中最常用的缓存抽象层
但它默认的配置在生产环境不够健壮，需要定制：

@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration
            .defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))         // 默认 TTL 30 分钟
            .serializeValuesWith(                      // 用 JSON 序列化，便于调试
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();               // 不缓存 null（防止穿透用布隆过滤器代替）

        // 不同缓存空间单独配置
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        cacheConfigurations.put("user", config.entryTtl(Duration.ofHours(1)));
        cacheConfigurations.put("product", config.entryTtl(Duration.ofMinutes(5)));
        cacheConfigurations.put("hotData", config.entryTtl(Duration.ofMinutes(1)));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withInitialCacheConfigurations(cacheConfigurations)
            .transactionAware()                       // 支持事务：事务回滚时缓存操作也回滚
            .build();
    }
}

使用：

  @Cacheable(value = "user", key = "#id")
  public User getUser(Long id) {
      return userMapper.selectById(id);  // 缓存 miss 时自动查 MySQL 并回填
  }

  @CacheEvict(value = "user", key = "#user.id")
  public void updateUser(User user) {
      userMapper.updateById(user);       // 更新 MySQL 后自动删除 Redis 缓存
  }
```

### 十、常见踩坑与实战经验

#### 坑 1：大 Key 删除导致 Redis 阻塞

```
问题：
  一个 Hash 类型的缓存存了 10 万个字段
  DEL 命令删除时，Redis 单线程被阻塞直到删除完成
  → 其他请求全部超时

解决：
  1. 用 UNLINK 代替 DEL（Redis 4.0+）
     UNLINK 是异步删除，先从 keyspace 移除，后台线程异步释放内存

  2. 分批删除
     HSCAN + HDEL 每次删一部分

  3. 控制缓存粒度
     不要把整张表塞进一个 Hash key
     拆成 user:1, user:2, user:3 这样的小 key
```

#### 坑 2：缓存雪崩 — 大量 key 同时过期

```
问题：
  10 万个 key 都设 TTL = 30 分钟
  30 分钟后全部过期 → 10 万请求同时打到 MySQL
  → MySQL 瞬间被打爆

解决：
  TTL 加随机偏移量：
    int ttl = 1800 + random.nextInt(300);  // 30 分钟 + 0~5 分钟随机
    redisTemplate.expire(key, ttl, TimeUnit.SECONDS);

  或者启动时预热 + 永不过期 + Canal 驱动更新
    不设 TTL，完全靠 Canal 监听 binlog 更新缓存
    避免了过期导致的突发 miss
```

#### 坑 3：热点 key 的缓存击穿

```
问题：
  某个明星的商品页缓存过期 → 瞬间几万 QPS 打到 MySQL
  单个 key 的失效导致数据库压力暴增

解决：
  1. 互斥锁
     缓存 miss 时，用 Redis SETNX 加锁
     只有拿到锁的线程查 MySQL 并回填
     其他线程等待或返回旧数据

     public String getWithLock(String key) {
         String value = redis.get(key);
         if (value == null) {
             // 尝试获取锁
             if (redis.setnx(lockKey, "1", 10, SECONDS)) {
                 try {
                     value = mysql.query(key);
                     redis.set(key, value, ttl);
                 } finally {
                     redis.delete(lockKey);
                 }
             } else {
                 // 没拿到锁，等一下重试
                 Thread.sleep(50);
                 return getWithLock(key);
             }
         }
         return value;
     }

  2. 逻辑过期（不设 TTL，数据里带过期时间）
     缓存永不过期（Redis 层面），但数据里存一个 logicalExpire 字段
     读到过期数据时：返回旧数据 + 异步线程更新缓存
     → 用户永远不会等待，但可能看到旧数据

  3. 热点 key 永不过期 + Canal 驱动更新
     核心数据不设 TTL，完全靠 binlog 更新
```

#### 坑 4：Canal 消费延迟导致的不一致

```
问题：
  Canal 消费 binlog 有延迟（正常 1-5 秒，异常时可能几分钟）
  在这个窗口内，用户可能看到旧数据

缓解措施：
  1. 关键写操作后主动删缓存（不依赖 Canal）
     Canal 只做兜底，正常路径还是主动删

  2. 前端配合：修改操作成功后提示"数据更新中"
     用户刷新后能看到最新数据

  3. 写操作后设一个短期缓存（如 5 秒）
     防止在 Canal 消费延迟期间大量请求打到 MySQL
```

### 十一、方案选型速查表

```
┌─────────────────┬──────────────┬──────────────┬───────────────┐
│     方案         │  一致性级别   │   复杂度      │  适用场景      │
├─────────────────┼──────────────┼──────────────┼───────────────┤
│ Cache Aside+TTL │ 弱一致       │ 低           │ 用户资料、配置 │
│ 延迟双删         │ 弱一致+      │ 低           │ 低并发写入     │
│ Cache+MQ重试    │ 最终一致      │ 中           │ 商品信息、订单 │
│ Canal binlog    │ 最终一致+     │ 高           │ 核心业务数据   │
│ 分布式锁双写     │ 强一致       │ 高           │ 账户余额       │
│ Redis为主+落DB  │ Redis 为准   │ 中           │ 秒杀库存       │
│ Write Behind    │ 弱一致       │ 中           │ 计数器、点赞   │
└─────────────────┴──────────────┴──────────────┴───────────────┘

生产环境推荐组合：
  日常：Cache Aside + MQ 重试
  核心数据：Cache Aside + MQ 重试 + Canal binlog 兜底
  热点数据：永不过期 + Canal 驱动更新 + 互斥锁防击穿
```

## 常见追问与应对

**Q1: 为什么不直接更新缓存，而要删除缓存？**
> 因为并发场景下"更新缓存"会产生脏数据——线程 A 更新 MySQL 为 100，线程 B 更新 MySQL 为 200，但线程 A 后更新缓存为 100，覆盖了 B 的 200。"删除缓存"就没有这个问题——删了之后下次读自然从 MySQL 拿最新值回填。另外一个原因是有些缓存的计算逻辑复杂（比如聚合多张表的数据），每次更新 MySQL 都重新计算缓存成本太高，不如删除后按需计算。

**Q2: 强一致性怎么实现？为什么不推荐？**
> 强一致性的做法是：写操作加分布式锁，同时在同一个"事务"中更新 MySQL 和 Redis。读操作先读 Redis，miss 后也加锁读 MySQL 回填。不推荐是因为：1）分布式锁本身有性能开销和可用性风险（Redis 锁的 master 切换可能丢锁）；2）每次写操作都要同步更新两个存储，延迟翻倍；3）两个独立的存储系统不可能做到真正的原子事务，总会有极端情况出问题。大部分业务用最终一致性就够了。

**Q3: Canal 监听 binlog 和应用层主动删缓存冲突吗？**
> 不冲突，它们是互补的。正常路径是应用层主动删缓存（低延迟），Canal 做兜底（应对应用层删除失败的情况）。唯一的"冲突"是 Canal 可能重复删——应用删了，Canal 又删一次。但删一个不存在的 key 只是返回 0，没有副作用。如果 Canal 是"更新缓存"而不是"删除缓存"，就可能有冲突——应用删了缓存，Canal 又用旧数据回填了。所以 Canal 侧也用"删除"策略最安全。

**Q4: Spring Cache 的 @CacheEvict 和手动删缓存有什么区别？**
> @CacheEvict 是声明式的，在方法执行后自动删除指定 key 的缓存。优点是代码简洁、和 Spring 事务管理集成（transactionAware 模式下事务回滚时缓存操作也回滚）。缺点是灵活性不够——比如删除失败时无法做重试、无法发 MQ、无法做条件删除。生产环境中，简单场景用 @CacheEvict，需要可靠性的核心数据用手动删 + MQ 重试。

**Q5: Redis 和 MySQL 数据类型不同怎么办？**
> 比如 MySQL 存的是关系型数据（多表关联），Redis 存的是反序列化后的 JSON 或 Hash。一致性方案不受影响——无论 Redis 存什么格式，变更时统一"删除缓存"就行，下次读时用新数据重新构建缓存。唯一需要注意的是：如果缓存是聚合多张表的数据（比如 user 表 + order 表聚合成用户详情缓存），那么 user 或 order 任意一张表变更都要删除这个缓存，否则会不一致。Canal 在这种场景特别有用——监听多张表的 binlog，任一表变更都触发缓存失效。

**Q6: 如何做数据一致性对账？离线对账还是实时对账？**
> 两种都做。**实时对账**：在读写路径中加入校验——每次从 Redis 读到数据后，以一定概率（比如 1%）同时查 MySQL 对比，不一致则告警并自动修复。优点是发现快，缺点是有额外 MySQL 查询开销。**离线对账**：定时任务（比如每小时）批量抽样对比 Redis 和 MySQL 的数据，计算不一致率。适合发现趋势性问题（比如某次发版后不一致率上升了）。两者结合：实时对账抓单个异常，离线对账看整体趋势。

## 完整面试回答

> 模拟面试中的实际对话，展示如何从头到尾回答一致性问题

**面试官**：项目中 Redis 和 MySQL 的数据一致性是怎么保证的？

**候选人**：这个问题我从三个层面回答——**一致性级别选型、具体实现方案、异常兜底机制**。

首先一致性级别要根据业务来选，不是所有数据都需要强一致。我们项目里把数据分三类：用户资料这种改了也无所谓几秒后生效的，用弱一致性；商品价格这种不一致会出问题的，用最终一致性；账户余额这种不能错的，直接查 MySQL 不走缓存。

**面试官**：最终一致性具体怎么实现的？

**候选人**：用 Cache Aside 模式——先更新 MySQL 再删除 Redis 缓存。为什么是删除不是更新呢？因为并发场景下更新缓存可能产生后发先至的问题，线程 A 和 B 同时更新，B 先更新了 MySQL 但 A 后更新了缓存，就出现脏数据。删除就没这个问题，删了之后下次读自然回填最新值。

但单纯"更新 DB + 删缓存"有个风险——删除缓存可能失败，比如 Redis 网络抖动。所以我们加了三层保障：第一层是正常删除缓存；第二层是删除失败时发 MQ 消息，消费者重试删除；第三层是用 Canal 监听 MySQL binlog 变更，自动同步到 Redis 做兜底。即使应用层的删除全部失败，Canal 也能在 1-5 秒内把缓存更新过来。

**面试官**：Canal 监听 binlog 的延迟你提到是 1-5 秒，这个窗口内用户看到旧数据怎么办？

**候选人**：两个应对。首先，正常情况下应用层的主动删除是成功的，Canal 只做兜底，所以大部分时候不会有这个窗口。其次，对热点数据我们用了"永不过期 + Canal 驱动更新"的策略——不设 TTL，完全靠 binlog 变更来更新缓存，这样不存在"过期瞬间大量 miss"的问题。另外还加了互斥锁防击穿——如果缓存真的 miss 了，用 SETNX 保证只有一个线程去查 MySQL 回填，其他线程等或返回旧数据。

**面试官**：那你们怎么监控一致性的？万一 Canal 挂了呢？

**候选人**：我们做了两级监控。第一级是实时监控——每次缓存操作都记录指标，包括缓存命中率、删除失败率、MQ 重试次数。命中率低于 70% 或删除失败率超过 0.1% 就告警。第二级是离线对账——每 5 分钟跑一个定时任务，随机抽 100 个 key 对比 Redis 和 MySQL 的值，不一致率超过 1% 就告警并自动以 MySQL 为准修复 Redis。Canal 挂了的情况我们也考虑了——Canal 消费延迟超过 60 秒会触发严重告警，运维介入修复。在这期间靠应用层的主动删缓存和 TTL 兜底。

## 知识关联

- [Redis 缓存穿透、击穿、雪崩与预热](./Redis-缓存穿透-击穿-雪崩-预热.md)
- [Redis I/O 多路复用与网络 I/O 模型](./Redis-IO多路复用.md)
- [MySQL 的 MVCC 机制](./MySQL-MVCC.md)
- [Kafka 消息队列核心原理](../系统设计/Kafka-消息队列.md)
- [10 张票 10 万人抢 — Redis + MySQL 秒杀方案](../场景问题/Redis秒杀方案设计.md)
