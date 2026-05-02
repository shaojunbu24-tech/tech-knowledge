# MySQL 的 MVCC 机制

> 分类：数据库
> 标签：MySQL, MVCC, 事务隔离, InnoDB, Undo Log, Read View

## 面试精答

> 适合面试时口述的简洁版本（2-3分钟能说完的要点）

MVCC（Multi-Version Concurrency Control，多版本并发控制）是 InnoDB 引擎实现**读不加锁、读写不冲突**的核心机制。它让事务在读取数据时看到一个**一致性快照**，而不被其他事务的修改所影响。

**实现依赖三个核心组件**：

1. **隐藏列**：每行数据有两个隐藏列 — `DB_TRX_ID`（最近修改该行的事务 ID）和 `DB_ROLL_PTR`（回滚指针，指向 Undo Log 中该行的前一个版本）
2. **Undo Log 版本链**：每次修改数据时，旧版本通过 `DB_ROLL_PTR` 串成一条链表，形成该行的多版本历史
3. **Read View**：事务执行快照读时生成的一致性视图，包含四个关键字段 — `m_ids`（生成时刻所有活跃事务 ID 列表）、`min_trx_id`（活跃事务中最小 ID）、`max_trx_id`（下一个将要分配的事务 ID）、`creator_trx_id`（创建该 Read View 的事务 ID）

**可见性判断规则**：沿版本链从最新版本开始，用 Read View 逐个判断每个版本是否可见：
- 事务 ID < `min_trx_id` → 该版本在 Read View 创建前已提交，**可见**
- 事务 ID >= `max_trx_id` → 该版本在 Read View 创建后才出现，**不可见**
- `min_trx_id` <= 事务 ID < `max_trx_id`：若事务 ID 在 `m_ids` 中 → 未提交，**不可见**；不在 `m_ids` 中 → 已提交，**可见**
- 事务 ID = `creator_trx_id` → 自己修改的，**可见**

如果当前版本不可见，就沿 `DB_ROLL_PTR` 找上一个版本，重复判断，直到找到可见版本或返回空。

## 原理深入详解

> 深入底层原理，理解"为什么"

### 为什么需要 MVCC

传统并发控制用锁实现：读加共享锁，写加排他锁，读写互斥。这在读多写少的场景下性能很差。MVCC 的核心思路是：**让读操作访问数据的历史版本，写操作创建新版本，两者互不干扰**。

### InnoDB 中的行记录结构

InnoDB 中每行记录额外存储两个隐藏字段：

| 隐藏列 | 大小 | 含义 |
|---|---|---|
| `DB_TRX_ID` | 6 字节 | 最近一次插入或更新该行的事务 ID |
| `DB_ROLL_PTR` | 7 字节 | 回滚指针，指向 Undo Log 中该行的上一版本 |

此外还有 `DB_ROW_ID`（6 字节，单调递增的行 ID，当表没有主键时 InnoDB 自动创建）。

### Undo Log 版本链的构建过程

```
时间线：
T1: 事务 100 INSERT row (id=1, name='A')   → DB_TRX_ID=100, DB_ROLL_PTR=NULL
T2: 事务 200 UPDATE name='B'               → 新版本 DB_TRX_ID=200, DB_ROLL_PTR → 旧版本(trx=100, name='A')
T3: 事务 300 UPDATE name='C'               → 新版本 DB_TRX_ID=300, DB_ROLL_PTR → 版本(trx=200, name='B') → 版本(trx=100, name='A')

版本链：[trx=300, C] → [trx=200, B] → [trx=100, A]
```

每次 UPDATE 不是原地修改，而是：1）把旧版本写入 Undo Log；2）在数据页上写入新版本；3）新版本的 `DB_ROLL_PTR` 指向 Undo Log 中的旧版本。

### Read View 的生成时机

**这是 RC 和 RR 隔离级别的核心区别**：

- **RC（Read Committed）**：每次 SELECT 都生成新的 Read View → 能看到其他事务已提交的最新数据
- **RR（Repeatable Read）**：只在事务中第一次 SELECT 时生成 Read View，后续复用 → 保证同一事务内多次读取结果一致

这就是为什么 RR 级别能解决不可重复读问题，而 RC 不能。

### 快照读 vs 当前读

- **快照读**：普通 SELECT，使用 MVCC 机制读取历史版本，不加锁
- **当前读**：SELECT ... FOR UPDATE / SELECT ... LOCK IN SHARE MODE / INSERT / UPDATE / DELETE，读取最新已提交版本，并加锁

MVCC 只对快照读生效。当前读不走 MVCC，而是走锁机制。

### Undo Log 的清理（Purge）

旧版本不能永久保留，否则 Undo Log 会无限增长。InnoDB 有专门的 Purge 线程负责清理：
- 当**没有任何活跃的 Read View** 需要访问某个旧版本时，该版本可以被清理
- 因此长事务会阻止 Undo Log 的回收，导致表空间膨胀（这是线上常见问题）

## 常见追问与应对

**Q1: MVCC 能解决幻读吗？**
> InnoDB 的 RR 级别下，快照读通过 MVCC 避免了大部分幻读。但当前读（如 SELECT ... FOR UPDATE）不走 MVCC，可能看到其他事务新插入的行。InnoDB 通过 **Next-Key Lock**（行锁 + 间隙锁）来解决当前读下的幻读问题。所以 InnoDB 的 RR 级别基本解决了幻读，是 MVCC + 间隙锁共同作用的结果。

**Q2: MVCC 为什么不能完全替代锁？**
> MVCC 只解决了"读-写"冲突，但"写-写"冲突仍然需要锁来解决。两个事务同时修改同一行，必须通过锁保证串行化，否则会丢失更新。此外，当前读场景（FOR UPDATE 等）也必须用锁。

**Q3: 长事务有什么危害？与 MVCC 有什么关系？**
> 长事务持有 Read View 不释放，导致 Undo Log 中大量旧版本无法被 Purge 线程清理，造成：1）Undo Log 表空间膨胀，磁盘占用增大；2）Undo Log 版本链变长，后续快照读需要遍历更多版本，查询性能下降；3）长事务还持有锁资源，阻塞其他事务。应尽量避免长事务。

**Q4: RC 和 RR 的本质区别是什么？**
> Read View 的生成时机不同。RC 每次 SELECT 生成新 Read View，所以能看到其他事务中间提交的变更；RR 只在第一次 SELECT 时生成，后续复用同一个，所以同一事务内读到的数据始终一致。底层 MVCC 机制完全相同，区别仅在于 Read View 的生命周期。

## 知识关联

- [MySQL 事务隔离级别](./MySQL-事务隔离级别.md)
- [InnoDB 锁机制](./InnoDB-锁机制.md)
- [Undo Log 与 Redo Log](./Undo-Log与Redo-Log.md)
