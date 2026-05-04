# 场景：设计一个监控日志分析 Agent

> 分类：场景问题
> 标签：Agent 设计, 日志分析, 工具设计, 多轮推理, ReAct, 系统设计

## 场景描述

面试官：如果你的系统每天产生大量监控日志（应用日志、访问日志、错误日志、系统指标等），现在要设计一个 AI Agent 来自动分析这些日志，给出分析结论和排障建议，你会怎么设计？Agent 的工具怎么设计？

## 答题思路

回答这类系统设计题，按照**"先定目标 → 再拆问题 → 逐层设计"**的框架：

```
1. 明确目标：这个 Agent 要解决什么问题？
2. 拆解能力：要完成目标，Agent 需要哪些能力？
3. 设计工具：每项能力怎么用工具实现？
4. 设计流程：Agent 的执行流程怎么编排？
5. 评估调优：怎么知道 Agent 做得好不好？
```

## 完整回答

### 一、先明确目标

不是"让 AI 看日志"这么简单。需要把目标拆清楚：

```
核心目标：给定一段时间范围内的监控日志，Agent 自动完成：
  1. 异常检测 — 发现哪里有问题
  2. 根因定位 — 找出问题根因
  3. 影响评估 — 判断影响范围有多严重
  4. 修复建议 — 给出可执行的排障或修复方案

和传统告警系统的区别：
  传统告警：规则匹配（CPU > 90% 就告警），只能发现已知模式
  Agent 分析：理解日志语义，关联多个信号，发现未知问题和复杂根因
```

### 二、Agent 需要哪些能力

分析一个排障工程师看到日志后的思考过程：

```
工程师的排障过程（Agent 要模仿这个）：

1. "先看看这段时间有没有报错"
   → 需要能力：检索日志（按时间、关键词、级别过滤）

2. "错误集中在哪个服务？哪个接口？"
   → 需要能力：聚合统计（按服务/接口/IP 分组计数）

3. "这个错误出现前，系统有什么变化？"
   → 需要能力：时序查询（错误发生前后的日志上下文）

4. "是不是数据库慢导致的？看看数据库指标"
   → 需要能力：查询指标数据（CPU、内存、QPS、延迟）

5. "看看最近有没有发版或配置变更"
   → 需要能力：查询变更记录

6. "综合以上信息，根因是 xxx，建议 xxx"
   → 需要能力：多步推理 + 综合判断（LLM 自身能力）
```

**提取出的 5 项核心能力**：

| 能力 | 说明 | 为什么不能只用 LLM |
|---|---|---|
| 日志检索 | 按条件搜索日志 | LLM 看不了几 GB 的原始日志，必须结构化查询 |
| 聚合统计 | 按维度汇总统计 | LLM 不擅长精确数学计算 |
| 指标查询 | 查系统指标（CPU/内存/延迟） | 指标存在时序数据库里，LLM 无法直接访问 |
| 变更查询 | 查发布/配置变更记录 | 变更记录在外部系统（CI/CD 平台） |
| 知识检索 | 查历史相似 case 和处理方案 | 公司内部排障知识库，LLM 预训练数据中没有 |

### 三、工具详细设计

每项能力设计一个工具（Tool），定义好输入、输出、描述。

#### 工具 1：`search_logs`（日志检索）

```json
{
  "name": "search_logs",
  "description": "从日志系统中检索日志。支持按时间范围、日志级别（ERROR/WARN/INFO）、服务名、关键词搜索。返回匹配的日志条目（最多 200 条）。",
  "parameters": {
    "start_time": "开始时间，ISO 8601 格式，如 2026-05-05T10:00:00Z",
    "end_time": "结束时间",
    "level": "日志级别过滤：ERROR | WARN | INFO | ALL",
    "service": "服务名过滤，如 'order-service', 'gateway'",
    "keyword": "关键词搜索，如 'timeout', 'NullPointerException'",
    "limit": "返回条数上限，默认 50，最大 200"
  }
}
```

**底层实现**：调用 Elasticsearch / Loki 的查询 API。

**为什么限制返回 200 条**：LLM 上下文窗口有限，把几千条日志全塞进去既浪费 Token 又降低分析质量。200 条足够让 Agent 看到错误模式，如果需要更多可以通过聚合统计工具获取。

**设计要点 — 工具描述为什么这么写**：

```
描述里的每一个细节都是在帮 LLM 做意图识别（参考「Agent 意图识别」的知识点）：

"支持按时间范围、日志级别、服务名、关键词搜索"
→ 告诉 LLM 这个工具能做什么，什么场景该用它

"返回匹配的日志条目（最多 200 条）"
→ 告诉 LLM 这个工具的边界，避免它期望拿到全量数据

参数里写清楚格式和例子
→ 减少 LLM 传错参数的概率（工具调用准确率）
```

#### 工具 2：`aggregate_logs`（聚合统计）

```json
{
  "name": "aggregate_logs",
  "description": "对日志做聚合统计分析。返回按指定维度分组后的统计结果（数量、趋势），不返回原始日志。适合回答'哪个服务错误最多'、'错误数随时间的变化趋势'等问题。",
  "parameters": {
    "start_time": "开始时间",
    "end_time": "结束时间",
    "group_by": "分组维度：service | level | endpoint | ip | hour",
    "level": "日志级别过滤",
    "service": "服务名过滤",
    "metric": "统计指标：count | error_rate | p99_latency"
  }
}
```

**底层实现**：ES 的 aggregation API / Prometheus 的 range_query。

**和 search_logs 的区别**：

```
search_logs：给我原始日志 → "到底报了什么错？"
aggregate_logs：给我统计结果 → "哪个服务最差？趋势是恶化还是好转？"

Agent 典型流程：
  第 1 步：用 aggregate_logs 看全局概览（哪个服务错误最多）
  第 2 步：用 search_logs 深入看具体错误内容
```

#### 工具 3：`query_metrics`（系统指标查询）

```json
{
  "name": "query_metrics",
  "description": "查询系统性能指标。支持 CPU 使用率、内存使用率、磁盘 I/O、网络流量、QPS、响应延迟（P50/P95/P99）等。可以按服务或实例查询。",
  "parameters": {
    "start_time": "开始时间",
    "end_time": "结束时间",
    "metric_name": "指标名：cpu_usage | memory_usage | disk_io | qps | latency_p99 | error_rate",
    "service": "服务名",
    "instance": "实例 IP（可选）",
    "step": "数据点间隔，如 '1m'（每分钟一个点）、'5m'"
  }
}
```

**底层实现**：调用 Prometheus / InfluxDB 的 HTTP API。

**为什么需要独立的指标查询工具**：

```
日志和指标是两套系统：
  日志 → Elasticsearch / Loki（全文检索，非结构化文本）
  指标 → Prometheus / InfluxDB（时序数据，结构化数值）

Agent 分析"是不是数据库慢导致服务报错"时：
  第 1 步：search_logs → 发现 order-service 有大量 timeout 错误
  第 2 步：query_metrics → 查 order-service 的 P99 延迟，确实飙升
  第 3 步：query_metrics → 查 MySQL 的 CPU 和慢查询数，发现 CPU 打满
  结论：MySQL 慢查询导致 CPU 打满 → order-service 请求超时
```

这就是 Agent 的多步推理能力——跨系统关联分析，这是传统规则告警做不到的。

#### 工具 4：`query_changes`（变更记录查询）

```json
{
  "name": "query_changes",
  "description": "查询指定时间范围内的系统变更记录，包括代码发布、配置变更、扩缩容事件。返回变更的时间、内容、操作人。",
  "parameters": {
    "start_time": "开始时间",
    "end_time": "结束时间",
    "change_type": "变更类型：deploy | config | scale | ALL",
    "service": "服务名（可选）"
  }
}
```

**底层实现**：调用 CI/CD 平台（Jenkins/GitLab CI）和配置中心（Apollo/Nacos）的 API。

**为什么这个工具很关键**：生产环境的故障 70% 以上和变更有关（发布新版本、改配置、扩缩容）。一个排障 Agent 如果不会查变更记录，就像医生不问"最近吃了什么药"就诊断一样。

#### 工具 5：`search_knowledge`（历史知识检索）

```json
{
  "name": "search_knowledge",
  "description": "从历史排障知识库中检索相似 case 的处理方案。输入问题描述，返回最相关的历史记录，包含问题描述、根因、解决方案。",
  "parameters": {
    "query": "问题描述，如 'order-service timeout, MySQL CPU high'",
    "top_k": "返回最相关的 K 条记录，默认 5"
  }
}
```

**底层实现**：RAG（Retrieval-Augmented Generation）。

```
历史排障知识库的构建：
  每次故障复盘后 → 把"问题 → 根因 → 方案"写入知识库
  对文本做 Embedding → 存入向量数据库（如 Milvus、Pinecone）
  查询时 → 对 query 做 Embedding → 向量相似度检索 → 返回 Top-K

为什么不用 LLM 的内置知识：
  公司内部的排障经验不在 LLM 的预训练数据中
  RAG 让 Agent 能访问"公司私有知识"
```

### 四、Agent 的执行流程设计

#### 为什么选 ReAct 范式

回顾三种范式的核心区别（参考「ReAct/Plan-and-Execute/Reflection 三种范式」）：

```
日志分析的特点：
  - 不确定性强：不知道问题出在哪，需要边查边分析
  - 每一步的结果决定下一步怎么走：看了日志才知道要不要查指标
  - 步骤数量不固定：简单问题 3 步搞定，复杂问题可能 10+ 步

→ 天然适合 ReAct（边想边干，走一步看一步）

不适合 Plan-and-Execute：
  因为没法提前规划好完整步骤（排查过程中会不断调整方向）
```

#### 完整流程

```
用户输入："今天 10:00-10:30 order-service 报错增多，帮我分析原因"
                │
                ↓
        ┌───────────────┐
        │  意图识别       │  LLM 理解用户意图，提取关键信息
        │  目标：分析报错  │  服务：order-service，时间：10:00-10:30
        └───────┬───────┘
                │
                ↓  Thought 1："先看全局概览，了解错误分布"
        ┌───────────────┐
        │ aggregate_logs │  按服务聚合，确认 order-service 错误确实最多
        │ group_by=service│  按小时聚合，确认 10:00 后错误激增
        └───────┬───────┘
                │
                ↓  Thought 2："看看具体报了什么错"
        ┌───────────────┐
        │  search_logs   │  搜索 order-service 的 ERROR 日志
        │  service=order │  发现大量 "MySQL query timeout" 错误
        │  level=ERROR   │
        └───────┬───────┘
                │
                ↓  Thought 3："日志指向 MySQL 超时，查一下数据库指标"
        ┌───────────────┐
        │ query_metrics  │  查 MySQL 的 CPU 使用率 → 10:00 起飙到 95%
        │ metric=cpu     │  查 MySQL 的慢查询数 → 同步激增
        │ service=mysql  │
        └───────┬───────┘
                │
                ↓  Thought 4："CPU 打满可能是慢查询，也可能是刚发了版"
        ┌───────────────┐
        │ query_changes  │  查 09:50-10:05 的变更记录
        │                │  发现 09:58 order-service 发版了
        └───────┬───────┘
                │
                ↓  Thought 5："发版时间吻合，看看新版本有没有新 SQL"
        ┌───────────────┐
        │ search_logs    │  搜索发版后的 SQL 日志
        │  keyword=SQL   │  发现新接口的查询没加索引，全表扫描
        └───────┬───────┘
                │
                ↓  Thought 6："看看历史有没有类似 case"
        ┌───────────────┐
        │search_knowledge│  检索到 3 条相似案例
        │                │  根因都是"新查询缺少索引"
        └───────┬───────┘
                │
                ↓  综合分析，生成报告
        ┌───────────────┐
        │  最终输出       │
        │               │
        │  根因：09:58    │
        │  order-service │
        │  发版，新接口   │
        │  SQL 缺少索引  │
        │  → MySQL 慢查询│
        │  → CPU 打满   │
        │  → 全局超时   │
        │               │
        │  建议：        │
        │  1. 紧急：给   │
        │  xxx 字段加索引│
        │  2. 短期：    │
        │  回滚到上一版本│
        │  3. 长期：    │
        │  CI 加 SQL    │
        │  审查步骤     │
        └───────────────┘
```

### 五、System Prompt 设计

```markdown
你是一个专业的监控日志分析 Agent。你的任务是分析用户提供的监控日志，
定位问题根因并给出修复建议。

## 工作流程
1. 先用 aggregate_logs 了解全局状况（哪个服务/时间段异常）
2. 再用 search_logs 查看具体错误内容
3. 根据错误内容，用 query_metrics 查相关系统指标
4. 用 query_changes 检查是否有变更
5. 用 search_knowledge 查历史相似案例
6. 综合所有信息，生成分析报告

## 输出格式
### 问题描述
（简洁描述异常现象）

### 根因分析
（一步步推导过程，每一步说明依据）

### 影响范围
（受影响的服务、接口、时间段、用户数）

### 修复建议
- 紧急处理：（可立即执行的操作）
- 短期方案：（1-2 天内完成）
- 长期改进：（防止再次发生）

## 注意事项
- 不要跳过工具调用直接给出结论，必须基于数据
- 如果工具返回的数据不足以定位根因，主动调用更多工具
- 如果多次分析后仍无法确定根因，明确说明并建议人工介入
- 修复建议要具体到可执行的命令或操作步骤
```

**Prompt 设计的关键点**（串联「Prompt Engineering 核心技巧」）：

1. **角色设定**：专业日志分析 Agent，锚定输出风格
2. **工作流程**：规定了工具调用的推荐顺序，减少 Agent 乱调用（相当于 Few-shot 的流程级引导）
3. **输出格式约束**：结构化输出，便于下游系统解析
4. **约束条件**：防止 Agent 的常见问题——跳过分析直接编结论、死循环调用工具

### 六、工程化考虑

#### 日志预处理（Agent 调用工具之前）

```
原始日志可能是这样的：
  2026-05-05 10:00:01.234 [http-nio-8080-exec-1] ERROR c.o.s.OrderService -
  java.sql.SQLException: Query execution was interrupted
    at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2457)
    at com.order.service.dao.OrderDao.queryOrder(OrderDao.java:156)
    at com.order.service.OrderService.getOrder(OrderService.java:89)
    ... 30 more lines

Agent 不需要看堆栈的每一行。预处理时做：
1. 日志清洗：去除重复堆栈，保留关键信息
2. 日志压缩：相同的错误模式合并计数（"该错误出现 156 次"）
3. 日志结构化：提取时间戳、级别、服务名、错误类型等字段

预处理后的日志：
  [10:00:01] ERROR order-service getOrder - MySQL query timeout
  [10:00:02] ERROR order-service getOrder - MySQL query timeout
  ... (共 156 条相同错误，已压缩)
```

#### Token 预算控制

```
一次分析的总 Token 预算：约 8000 Token

分配：
  System Prompt：      ~500 Token
  用户输入：            ~200 Token
  工具调用结果（总计）：~5000 Token（最重要）
  Agent 推理输出：      ~2000 Token
  最终报告：            ~3000 Token（单独一轮）

控制工具返回数据量：
  search_logs → 限制 200 条，每条截断到 200 字符
  aggregate_logs → 只返回聚合结果，不返回原始数据
  query_metrics → 只返回异常时段的数据，不返回全部
```

#### 容错与兜底

```
工具调用失败怎么办？
  - ES 查询超时 → 重试 1 次，仍失败则用已有数据分析 + 标注"数据不完整"
  - 返回空结果 → 告知用户"指定时间范围内无匹配日志"，建议调整查询条件
  - Agent 推理卡住（循环调用同一工具）→ 设最大步数（10 步），超限后强制输出当前结论

Agent 结论不可信怎么办？
  - 对最终报告加"置信度"标注（高/中/低）
  - 低置信度时建议人工复核
  - 记录 Agent 的完整推理链路（每步的 Thought + Action + Observation），方便人工审计
```

### 七、如何评估这个 Agent

串联「Agent 调优评估与评测集构建」的知识：

```
评测集构建：
  1. 从历史故障工单中收集 200 个真实 case
  2. 每个 case 标注：输入描述 + 期望的根因 + 期望的工具调用链
  3. 按难度分级：
     - 简单：单服务单错误，直接定位（如磁盘满了）
     - 中等：跨服务关联，需 3-5 步推理（如本案例）
     - 困难：多根因交织，需 8+ 步推理和排除法

评测维度：
  | 指标             | 目标    |
  |-----------------|--------|
  | 根因定位准确率     | > 80%  |
  | 工具调用准确率     | > 90%  |
  | 平均分析步数       | 4-6 步 |
  | 平均耗时          | < 30s  |
  | 无效调用率         | < 10%  |

评测方法：
  - 根因准确率：人工判断（LLM-as-Judge 辅助）
  - 工具调用准确率：和标注的 golden trajectory 对比
  - 效率指标：自动统计
```

## 知识点串联

本题涉及的知识脉络，按回答中出现的顺序：

**Agent 设计**：
- [Agent 意图识别](../大模型-AI/Agent-意图识别.md) — 工具描述怎么写才能让 LLM 选对工具
- [ReAct/Plan-and-Execute/Reflection 三种范式](../大模型-AI/Agent-ReAct-PlanAndExecute-Reflection.md) — 为什么选 ReAct 而不是 Plan-and-Execute
- [Agent 调优评估与评测集构建](../大模型-AI/Agent-调优评估与评测集构建.md) — 评测集怎么建、指标怎么定
- [Prompt Engineering 核心技巧](../大模型-AI/Prompt-Engineering-核心技巧.md) — System Prompt 的角色设定、输出格式约束、Few-shot 流程引导

**基础设施**（工具的底层实现涉及）：
- [Kafka 消息队列](../系统设计/Kafka-消息队列.md) — 日志采集管道（应用 → Kafka → ES）
- [Redis 缓存穿透/击穿/雪崩](../数据库/Redis-缓存穿透-击穿-雪崩-预热.md) — Agent 查询结果可以用 Redis 缓存，避免重复调 ES

**操作系统 & 网络**（日志中常见的问题）：
- [进程、线程、协程](../操作系统/进程-线程-协程.md) — 线程池满、进程 OOM 等日志的分析
- [TCP 三次握手/四次挥手](../计算机网络/传输层.md) — 连接超时、TIME_WAIT 堆积等网络日志的分析

## 追问预判

**Q1: 如果日志量特别大（每天 TB 级），Agent 怎么处理？**
> Agent 不直接处理原始日志，而是在预处理阶段做三层过滤：1）日志采集时按级别过滤（只采 WARN 及以上）；2）ES 查询时按时间范围和条件精确过滤；3）工具返回时做摘要压缩（相同模式合并计数）。Agent 只看到高度浓缩后的信息，不接触原始全量日志。

**Q2: 如果多个工具返回的信息矛盾怎么办？**
> 比如 search_logs 显示是网络超时，query_metrics 显示 CPU 正常。Agent 的推理能力就在这里发挥作用——它会注意到矛盾并做进一步验证（比如查网络指标、查下游服务状态）。System Prompt 中引导 Agent "如果数据不足以定位根因，主动调用更多工具"就是应对这种情况。

**Q3: 怎么防止 Agent 产生幻觉（编造不存在的结论）？**
> 四道防线：1）System Prompt 要求"不要跳过工具调用直接给出结论"；2）最终报告的每个结论必须引用具体数据（"根据 search_logs 返回的 156 条 timeout 错误..."）；3）输出加置信度标注；4）记录完整推理链路供人工审计。本质上是用工具调用的事实数据"锚定"Agent 的输出，减少自由发挥的空间。

**Q4: 这个 Agent 能做到实时分析吗？还是只能事后分析？**
> 当前设计是事后分析（给定时间范围查历史日志）。要做到实时分析需要加一层：用 Kafka/Flink 做实时日志流处理 → 预设规则检测异常 → 触发 Agent 自动分析。Agent 不需要实时看所有日志，而是在异常被检测到后才介入，分析最近的日志上下文。这样既保证实时性，又控制了 Agent 的调用成本。
