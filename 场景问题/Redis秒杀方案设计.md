# 场景：10 张票 10 万人抢 — Redis + MySQL 秒杀方案

> 分类：场景问题
> 标签：秒杀, Redis, MySQL, 高并发, 超卖, 分布式锁, Lua, 库存扣减, 消息队列, 限流

## 面试精答

> 适合面试时口述的简洁版本（2-3分钟能说完的要点）

10 张票 10 万人抢，核心矛盾是**极高并发写 + 极少量库存**。MySQL 直接抗会崩，必须用 Redis 做前置缓冲，MySQL 做最终落盘。

**方案四层**：

1. **前端限流** — 按钮 CAS 防抖、验证码、请求随机丢弃 90%，只放少量流量到后端
2. **Redis 原子扣库存** — 用 Lua 脚本把"判断库存 → 扣减 → 写订单记录"合成一个原子操作，彻底杜绝超卖。10 张票扣完直接拒绝后续请求
3. **异步落库** — Redis 扣减成功后发消息到 MQ（或直接异步写入），消费者拿到消息后写 MySQL 生成正式订单。数据库不再扛峰值并发
4. **兜底补偿** — 超时未支付的订单释放库存回 Redis，Redis 和 MySQL 之间做定期对账

**一句话总结**：Redis 抗并发写（原子扣减）+ MySQL 保数据一致性（最终落盘）+ MQ 做缓冲削峰 + 限流挡住洪峰。核心难点不是技术栈，而是**在超卖和少卖之间找平衡**。

## 原理深入详解

> 深入底层原理，理解"为什么"

### 一、先搞清楚问题到底有多难

```
10 张票，10 万人，假设开放抢购的时间窗口是 1 秒：

  并发量：100,000 QPS（每秒请求数）
  竞争资源：10 个

  也就是说 1 秒内 10 万人同时抢 10 个东西
  成功率：0.01%
  99,990 个请求注定失败

这意味着：
  1. 必须在极短时间内处理完 10 万个请求
  2. 其中 99,990 个要快速返回"已售罄"，不能浪费资源
  3. 关键的 10 个成功请求，一个都不能多（超卖是事故）
```

### 二、为什么不能只用 MySQL

先理解 MySQL 处理并发写的方式。

#### 行锁与并发扣减

```
假设库存表：
  ticket_id  |  stock
  1          |  10

扣减 SQL：
  UPDATE ticket SET stock = stock - 1 WHERE ticket_id = 1 AND stock > 0;

这条 SQL 看起来没问题，但在高并发下：

  线程 A：SELECT stock → 10 → 决定扣减
  线程 B：SELECT stock → 10 → 决定扣减  ← 还没来得及减
  线程 A：UPDATE stock = 9
  线程 B：UPDATE stock = 9  ← 应该是 8，但又是 9

MySQL 的 InnoDB 引擎通过行锁解决这个问题：
  UPDATE 语句会对目标行加 X 锁（排他锁）
  同一行同时只有一个事务能修改

实际执行：
  线程 A：UPDATE ... → 获取行锁 → stock 10→9 → 释放锁
  线程 B：UPDATE ... → 等待行锁 → 获取 → stock 9→8 → 释放锁
  线程 C：UPDATE ... → 等待行锁 → 获取 → stock 8→7 → 释放锁
  ...

10 万个请求排队等同一行的行锁：
  每个事务耗时 ≈ 1ms（网络 + 磁盘 IO）
  10 万个排队 ≈ 100 秒
  → 数据库连接池被占满，其他业务全部超时
  → 雪崩
```

#### MySQL 的性能天花板

```
单机 MySQL 的写入能力：
  单行更新（带行锁）：~1000-5000 TPS（取决于硬件）
  秒杀场景需要：100,000 TPS

差距：20-100 倍

所以结论是：MySQL 不能直接扛秒杀的峰值写入
但 MySQL 也不是没用的——它是数据的"最终归宿"
数据最终要持久化到 MySQL，只是不能让它直接扛峰值
```

### 三、为什么选 Redis 做前置缓冲

#### Redis 为什么快

```
Redis 单线程为什么比 MySQL 多线程还快？

1. 纯内存操作
   内存读取：~100ns（纳秒）
   磁盘读取：~10ms（毫秒）
   差距：10 万倍

2. 单线程 + IO 多路复用（epoll）
   没有锁竞争的开销
   没有线程上下文切换的开销
   一个线程通过 epoll 同时管理数千个连接
   （详见 Redis IO 多路复用知识点）

3. 简单的数据结构
   不像 MySQL 要解析 SQL、优化查询计划、维护索引
   Redis 的命令直接操作内存数据结构

Redis 单机性能：
  GET/SET：~100,000 QPS
  扣减库存（DECR）：~100,000 QPS
  刚好能扛住 10 万并发的量级
```

#### Redis 的原子操作

```
方案一：先用 GET 再用 DECR（错误！）

  客户端 A：GET ticket:1:stock → 10  → 觉得有库存
  客户端 B：GET ticket:1:stock → 10  → 觉得有库存
  客户端 A：DECR ticket:1:stock → 9
  客户端 B：DECR ticket:1:stock → 8

  看起来没问题？但如果库存已经为 1：
  客户端 A：GET → 1  → 觉得有库存
  客户端 B：GET → 1  → 觉得有库存
  客户端 A：DECR → 0  ✓
  客户端 B：DECR → -1  ✗ 超卖！

  问题：GET 和 DECR 不是原子操作
  中间可以被其他客户端插入

方案二：只用 DECR（部分正确）

  DECR ticket:1:stock

  DECR 本身是原子操作（Redis 单线程，命令逐个执行）

  客户端 A：DECR → 9
  客户端 B：DECR → 8
  ...
  客户端 J：DECR → 0  ✓
  客户端 K：DECR → -1  ← 还是超卖！

  原因：DECR 不判断库存是否为 0，直接减
  需要在减之前先判断，但"判断 + 减"不是原子的

方案三：Lua 脚本（正确做法）

  把"判断 + 减"写成一个 Lua 脚本，Redis 保证整个脚本原子执行

  -- seckill.lua
  local stock = redis.call('GET', KEYS[1])
  if not stock or tonumber(stock) <= 0 then
      return -1  -- 库存不足
  end
  redis.call('DECR', KEYS[1])
  -- 记录已购买用户，防止重复购买
  redis.call('SADD', KEYS[2], ARGV[1])
  return 1  -- 扣减成功

  为什么 Lua 脚本是原子的？
  Redis 执行 Lua 脚本时会阻塞其他命令
  整个脚本要么全部执行完，要么都不执行
  不会被其他客户端的命令打断
```

### 四、完整架构设计

```
                    ┌──────────────┐
                    │   用户请求    │
                    │  （10万 QPS） │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  前端限流层    │
                    │  · 按钮防抖    │
                    │  · 验证码     │
                    │  · 随机丢弃   │
                    └──────┬───────┘
                           │  过滤后 ~1万 QPS
                    ┌──────▼───────┐
                    │  Nginx 限流   │
                    │  · 令牌桶     │
                    └──────┬───────┘
                           │  通过后 ~5000 QPS
                    ┌──────▼───────┐
                    │  应用服务层    │
                    │  · 校验参数    │
                    │  · 判断活动时间 │
                    │  · 判断用户资格 │
                    └──────┬───────┘
                           │
               ┌───────────▼───────────┐
               │   Redis（原子扣减）     │
               │   Lua 脚本：           │
               │   1. 判断库存          │
               │   2. 扣减库存          │
               │   3. 记录购买用户       │
               └───────────┬───────────┘
                       │         │
                 扣减成功     扣减失败
                       │         │
               ┌───────▼───┐  ┌──▼──────┐
               │ 发送 MQ   │  │返回售罄  │
               │ 消息      │  │99,990个  │
               └───────┬───┘  └─────────┘
                       │
               ┌───────▼───────────┐
               │  MQ 消费者        │
               │  · 写 MySQL 订单  │
               │  · 扣 MySQL 库存   │
               │  · 发短信/通知     │
               └───────────────────┘
```

### 五、逐层详解

#### 第一层：前端限流

```
为什么要做前端限流？
  10 万请求不加以控制全部打到后端，后端必然崩溃
  目标：把 10 万请求过滤到 ~5000 个到后端

手段：

1. 按钮防抖（防止用户疯狂点击）
   点击"抢购"按钮后：
   - 按钮立即置灰（disabled）
   - 3 秒内不能再次点击
   - 前端生成唯一的 request_id 防止重复提交

2. 验证码（挡住机器脚本）
   - 点击抢购前弹出图形验证码 / 滑动验证
   - 自动化脚本很难通过验证码
   - 同时起到"错峰"效果——10 万人不会在同一毫秒提交

3. 前端随机丢弃
   - 页面 JS 中加入随机等待
   - Math.random() > 0.9 的请求才真正发出去
   - 90% 的请求在前端就被拦截了
```

#### 第二层：Nginx 限流

```
到达服务端的请求仍然可能有数万个，用 Nginx 做第二道拦截。

Nginx 限流配置（令牌桶算法）：

  # 按IP限流：每秒最多 10 个请求
  limit_req_zone $binary_remote_addr zone=seckill:10m rate=10r/s;

  location /seckill {
      limit_req zone=seckill burst=20 nodelay;
      # burst=20：允许突发 20 个请求
      # nodelay：超出的直接拒绝，不排队
      proxy_pass http://backend;
  }

令牌桶原理：
  想象一个桶，以固定速率（rate=10r/s）往里面放令牌
  每个请求需要拿一个令牌才能通过
  桶满了令牌就溢出（burst=20 是桶容量）
  没拿到令牌的请求直接返回 503

效果：
  如果后端服务有 10 台机器，每台配 rate=10
  总通过量 ≈ 100 QPS，远低于 Redis 的处理能力
```

#### 第三层：Redis 原子扣减（核心）

```lua
-- Lua 脚本：秒杀扣减库存
-- KEYS[1] = "seckill:ticket:1:stock"    库存 key
-- KEYS[2] = "seckill:ticket:1:users"    已购买用户集合
-- ARGV[1] = user_id                      当前用户 ID
-- ARGV[2] = 库存上限（用于初始化校验）

-- 1. 检查是否已经购买过（防重复购买）
if redis.call('SISMEMBER', KEYS[2], ARGV[1]) == 1 then
    return -2  -- 已购买
end

-- 2. 获取当前库存
local stock = redis.call('GET', KEYS[1])
if not stock then
    return -3  -- 活动不存在
end

-- 3. 判断库存
if tonumber(stock) <= 0 then
    return -1  -- 库存不足
end

-- 4. 扣减库存
redis.call('DECR', KEYS[1])

-- 5. 记录已购买用户
redis.call('SADD', KEYS[2], ARGV[1])

return 1  -- 扣减成功
```

```
为什么用 Lua 脚本而不是 Redis 事务（MULTI/EXEC）？

Redis 事务的问题：
  MULTI
  GET stock       → 返回 10
  ← 这里无法根据 GET 的结果决定是否 DECR
  ← 事务中的命令是全部执行或全部不执行
  ← 但无法在事务中间加条件判断
  DECR stock
  EXEC

Lua 脚本的优势：
  1. 可以在中间加条件判断（if tonumber(stock) <= 0 then return -1）
  2. 同样是原子的（Redis 执行 Lua 脚本时阻塞其他命令）
  3. 减少网络往返（一次发送整个脚本，比多次命令快得多）

Redis 数据结构选择：
  库存：String（DECR 原子递减）
  已购买用户：Set（SISMEMBER O(1) 判断是否重复）
```

#### 第四层：异步落库

```
Redis 扣减成功后，不能同步写 MySQL（慢），而是发到消息队列。

为什么用 MQ 而不是直接写 MySQL？
  1. Redis 扣减 1ms，MySQL 写入 5-50ms
     如果同步写，10 个订单的写入也会阻塞 Redis 扣减线程
  2. MQ 起到削峰作用——10 个订单消息慢慢消费，MySQL 不用扛峰值
  3. MQ 保证消息不丢失——即使 MySQL 临时故障，消息在 MQ 里等着

消息内容：
  {
    "user_id": "12345",
    "ticket_id": "1",
    "order_id": "ORD20240506001",
    "timestamp": 1714982400000,
    "request_id": "req_abc123"    ← 幂等键
  }

消费者处理逻辑：
  1. 检查 request_id 是否已处理（幂等性）
     → 防止 MQ 重复投递导致重复创建订单
  2. 开启数据库事务
  3. 创建订单记录（INSERT INTO orders）
  4. 扣减 MySQL 库存（UPDATE ticket SET stock = stock - 1 WHERE stock > 0）
  5. 提交事务
  6. 发送短信/推送通知
```

```
MySQL 中的表设计：

-- 票务库存表
CREATE TABLE ticket (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    total_stock INT NOT NULL,       -- 总库存（10）
    available_stock INT NOT NULL,   -- 当前可用库存
    price DECIMAL(10, 2),
    seckill_start_time DATETIME,    -- 秒杀开始时间
    seckill_end_time DATETIME,
    version INT DEFAULT 0,          -- 乐观锁版本号
    INDEX idx_seckill_time (seckill_start_time, seckill_end_time)
);

-- 订单表
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_no VARCHAR(64) UNIQUE NOT NULL,  -- 订单号（唯一）
    user_id BIGINT NOT NULL,
    ticket_id BIGINT NOT NULL,
    status TINYINT DEFAULT 0,             -- 0待支付 1已支付 2已取消
    request_id VARCHAR(64) UNIQUE,        -- 幂等键
    create_time DATETIME DEFAULT NOW(),
    pay_deadline DATETIME,                 -- 支付截止时间
    INDEX idx_user_ticket (user_id, ticket_id),
    INDEX idx_request_id (request_id),
    INDEX idx_status_deadline (status, pay_deadline)
);

MySQL 扣减用乐观锁（因为经过 Redis 过滤后只剩 10 个写请求）：

  UPDATE ticket
  SET available_stock = available_stock - 1,
      version = version + 1
  WHERE id = 1
    AND available_stock > 0
    AND version = #{当前版本号};

  affected rows = 0 → 说明并发冲突，重试或返回失败
  affected rows = 1 → 扣减成功
```

### 六、库存同步与补偿机制

Redis 和 MySQL 之间存在数据不一致的窗口，需要补偿。

```
问题场景：

  T1: Redis stock = 10
  T2: Redis Lua 扣减成功，stock → 9，发 MQ 消息
  T3: MQ 消费者还没来得及写 MySQL
  T4: 此时 Redis 显示 9，MySQL 还是 10
      → 不一致

  更极端的场景：
  Redis 扣减成功，MQ 消息丢了
  → Redis 显示 9，MySQL 还是 10（永久不一致）

补偿方案：

1. 消息可靠性保证
   - 使用 RocketMQ 的事务消息 或 Kafka 的 exactly-once
   - 消费者处理成功后才 ACK
   - 处理失败自动重试

2. 定期对账
   每 5 分钟跑一次对账任务：
   Redis stock + MySQL orders 表中成功订单数 = MySQL ticket.total_stock

   如果不等：
     Redis 少了 → 说明有已支付订单但 Redis 没更新（不影响，Redis 只是缓存）
     Redis 多了 → 说明有扣减但没落库，需要补写或回滚 Redis

3. 超时未支付释放库存
   用户抢到了但 15 分钟没付款：

   定时任务扫描 orders 表：
     WHERE status = 0            -- 待支付
       AND pay_deadline < NOW()   -- 已超时

   对每条超时订单：
     1. 更新订单状态为"已取消"
     2. Redis DECRBY 恢复库存（也需要 Lua 脚本保证原子）
     3. MySQL UPDATE 恢复库存

   注意：释放操作也要保证幂等
```

### 七、防超卖的数学证明

```
Redis Lua 脚本执行过程是严格串行的：

时间线：
  T1: 用户 A 执行 Lua → GET stock=10 → DECR → stock=9  → SADD users=A  → return 1
  T2: 用户 B 执行 Lua → GET stock=9  → DECR → stock=8  → SADD users=B  → return 1
  ...
  T10: 用户 J 执行 Lua → GET stock=1 → DECR → stock=0 → SADD users=J → return 1
  T11: 用户 K 执行 Lua → GET stock=0 → tonumber(stock)<=0 → return -1

结果：恰好 10 个人成功，第 11 个人开始全部被拒绝
超卖不可能发生，因为 Lua 脚本执行期间不会有其他命令插入

MySQL 侧的二次保障：
  UPDATE ... WHERE available_stock > 0
  即使 Redis 出了 bug 放过了 11 个人
  MySQL 的 available_stock > 0 条件也能拦住第 11 个
  → 双重保险
```

### 八、极端情况分析

```
情况 1：Redis 宕机
  应对：秒杀开始前在 MySQL 中也预加载库存
        Redis 不可用时降级到 MySQL 乐观锁扣减
        性能下降但不会超卖

情况 2：MQ 消息积压
  应对：增大消费者数量
        如果积压超过阈值，暂停 Redis 扣减
        先消化积压消息再继续

情况 3：用户抢到但不支付
  应对：设置支付超时时间（如 15 分钟）
        超时自动取消订单并释放库存
        释放的库存回到 Redis 供后续用户购买

情况 4：恶意刷单
  应对：用户维度的频率限制
        Redis 记录每个用户的请求次数
        超过限制直接拒绝
        IP 维度 + 用户 ID 维度双重限制

情况 5：Redis 和 MySQL 库存不一致
  应对：以 MySQL 为准
        定期对账任务修正 Redis
        活动结束后清空 Redis，以 MySQL 数据为准
```

## 常见追问与应对

**Q1: 不用 MQ，直接写 MySQL 行不行？**
> 可以，但要看并发量。如果经过前端限流 + Nginx 限流后，到达后端的请求只有几十个，同步写 MySQL 完全没问题。MQ 的价值在并发更高时——如果 1000 个请求同时通过限流层，同步写 MySQL 会拖慢整个接口响应时间，用户体验变差。10 张票这个量级，MQ 的作用主要是解耦和保消息不丢，不是削峰。

**Q2: 用 Redis 的 SETNX 做分布式锁来扣库存可以吗？**
> 可以但没必要。SETNX 分布式锁的问题：1）需要设置超时时间防止死锁，但超时时间设多少不好定——太短业务没处理完锁就释放了，太长异常时锁迟迟不释放；2）锁的释放可能不安全——A 的锁超时释放了，B 拿到锁，A 这时候执行 DEL 把 B 的锁删了；3）每个请求都要获取锁 → 执行 → 释放锁，至少 3 次 Redis 通信，而 Lua 脚本只需要 1 次。Lua 脚本天然原子，比分布式锁方案更简单更高效。

**Q3: 为什么不直接把库存全放 MySQL，用乐观锁？**
> 乐观锁的方案是 `UPDATE ... SET stock = stock - 1 WHERE stock > 0 AND version = ?`。10 万人同时抢，假设只有 10 个能成功，那 99,990 个请求的 UPDATE 都会失败（version 不匹配或 stock=0），但这些请求依然要执行到 MySQL 层。10 万次 SQL 解析 + 行锁等待，MySQL 单机扛不住。Redis 的价值就是在 MySQL 之前挡掉 99.99% 的请求，只有扣减成功的才到 MySQL。

**Q4: 如果是 10 万张票呢？方案有什么变化？**
> 核心方案不变，但限流策略要调整。10 张票只需要放 50-100 个请求到 Redis 就够了，99.9% 的流量在前端和 Nginx 就挡掉了。10 万张票需要放更多流量（比如每秒 1 万），Redis 扣减成功后 MQ 消费者要批量写 MySQL（比如 100 条一批 INSERT），MySQL 端需要分库分表来扛写入量。关键是**库存越多，放过的流量越多，后端压力越大**。

**Q5: Redis 的数据什么时候初始化？怎么保证和 MySQL 一致？**
> 活动开始前（比如提前 5 分钟），从 MySQL 读取库存总量写入 Redis。`SET seckill:ticket:1:stock 10`。活动结束后，以 MySQL 数据为准清空 Redis 缓存。活动过程中，Redis 是"热数据"，MySQL 是"冷数据"——Redis 的库存数是实时的，MySQL 的库存数有延迟（通过 MQ 异步更新）。最终一致性由对账任务保证。

**Q6: 幂等性怎么保证？**
> 幂等性就是"同一个操作执行一次和执行多次效果一样"。三个层面保证：1）Redis 层——Lua 脚本中用 SISMEMBER 判断用户是否已购买，防止同一用户重复扣减；2）MQ 层——消息携带 request_id，消费者处理前先查 orders 表是否已存在该 request_id；3）MySQL 层——orders 表的 request_id 字段设为 UNIQUE，INSERT 重复 request_id 会报唯一键冲突，捕获异常即可。

## 完整面试回答

> 模拟面试中的实际对话，展示如何从头到尾回答秒杀设计问题

**面试官**：给你一道场景题——10 张票，10 万人同时抢，用 Redis + MySQL 怎么设计？

**候选人**：这个问题的核心矛盾是**极高并发写 + 极少量库存**。10 万人抢 10 个东西，MySQL 直接扛肯定不行——单行更新有行锁，10 万人排队等同一行的锁，连接池瞬间就满了。所以我的思路是**Redis 抗并发、MySQL 保一致、中间用 MQ 做缓冲**。

**面试官**：具体说一下流程。

**候选人**：分四层。第一层是**前端限流**——按钮防抖、验证码、前端随机丢弃 90% 的请求。这一步能挡掉大部分无效流量。

第二层是 **Nginx 限流**，用令牌桶算法控制到达后端的 QPS，比如每秒只放 100 个请求通过。

第三层是核心——**Redis 原子扣减**。库存提前加载到 Redis，用 Lua 脚本做三件事：判断库存是否大于 0、扣减库存、记录已购买用户防止重复。Lua 脚本在 Redis 中是原子执行的，所以不会有超卖问题。10 张票扣完，后续请求直接返回"售罄"。

第四层是**异步落库**。Redis 扣减成功后发一条消息到 MQ，消费者拿到消息后写 MySQL 创建正式订单、扣减 MySQL 库存。这样 MySQL 不需要扛峰值并发。

**面试官**：你说 Lua 脚本保证原子性，能具体讲讲为什么不直接用 Redis 的事务或者分布式锁吗？

**候选人**：Redis 的事务（MULTI/EXEC）虽然能保证命令一次性执行，但事务中间不能加条件判断。比如我想"先 GET 看库存够不够，不够就不 DECR"，MULTI/EXEC 做不到——它不支持在事务中间做 if 判断。

分布式锁也能解决问题，但有三个缺点：一是每个请求需要 3 次 Redis 通信（获取锁、执行、释放锁），Lua 脚本只需 1 次；二是锁超时时间不好定，太短业务没完锁就释放，太长异常时锁不释放；三是释放锁可能误删别人的锁。Lua 脚本天然原子，更简单更高效。

**面试官**：Redis 扣减成功了但 MySQL 写入失败了怎么办？比如 MQ 消息丢了。

**候选人**：三个保障。第一，MQ 本身的可靠性——用 Kafka 或者 RocketMQ 的事务消息，消费者处理成功才 ACK，失败了自动重试。第二，定期对账——每 5 分钟跑一次任务，用 MySQL 的总库存减去订单表的成功订单数，和 Redis 的库存数对比，不一致就以 MySQL 为准修正 Redis。第三，活动结束后统一清空 Redis 缓存，以 MySQL 数据为最终结果。

另外还有一个常见场景要处理——用户抢到了但超时不付款。设置 15 分钟支付截止时间，定时任务扫描超时未支付的订单，取消订单并释放库存回 Redis，这样其他人还有机会买到。

**面试官**：设计得挺完整的。你觉得这个方案最大的风险点在哪？

**候选人**：我觉得最大的风险是**Redis 宕机**。Redis 是整个方案的核心，如果 Redis 挂了，扣减逻辑就没法执行了。应对方案是降级——Redis 不可用时直接走 MySQL 乐观锁扣减，性能差一些但不会超卖，保证核心业务可用。另外也可以用 Redis Sentinel 或者 Redis Cluster 保证 Redis 本身的高可用。

## 知识关联

- [Redis I/O 多路复用与网络 I/O 模型](../数据库/Redis-IO多路复用.md)
- [Redis 缓存穿透、击穿、雪崩与预热](../数据库/Redis-缓存穿透-击穿-雪崩-预热.md)
- [Redis 与 MySQL 双写一致性](../数据库/Redis与MySQL双写一致性.md)
- [MySQL 的 MVCC 机制](../数据库/MySQL-MVCC.md)
- [Kafka 消息队列核心原理](../系统设计/Kafka-消息队列.md)
- [进程、线程、协程的区别与联系](../操作系统/进程-线程-协程.md)
