---
tags:
  - 数据库
  - MySQL
  - 面试
  - 速查
created: 2026-08-17
---

# MySQL数据库 — 速查与面试题

> 从 [[MySQL数据库]] 拆分而来，包含对比表格、图解、速查清单、面试题和踩坑记录。

---

## 2️⃣ 对比表格

### 2-1 InnoDB vs MyISAM

| 维度 | InnoDB | MyISAM |
|------|--------|--------|
| 事务 | ✅ 支持（ACID） | ❌ 不支持 |
| 外键 | ✅ 支持 | ❌ 不支持 |
| 行锁 | ✅ 支持（行级锁） | ❌ 只支持表锁 |
| 全文索引 | ✅ 支持（MySQL 5.6+） | ✅ 支持 |
| 主键 | 必须有（聚簇索引） | 可无 |
| 索引类型 | 聚簇索引 | 非聚簇索引 |
| 存储结构 | 每个表两个文件（.frm + .ibd） | 每个表三个文件（.frm + .MYD + .MYI） |
| MVCC | ✅ 支持（多版本并发控制） | ❌ 不支持 |
| 崩溃恢复 | ✅ 自动恢复 | ❌ 需手动修复 |
| 适用场景 | 写多、需要事务、外键 | 读多、写少、全文搜索 |
| **面试结论** | **生产必用** | 淘汰 |

### 2-2 聚簇索引 vs 非聚簇索引

| 维度 | 聚簇索引（InnoDB 主键） | 非聚簇索引（普通索引） |
|------|----------------------|---------------------|
| 索引结构 | 索引即数据，数据按索引顺序存储 | 索引与数据分离 |
| 主键/主键索引 | 表按主键构建 B+ 树，叶子存整行数据 | — |
| 叶子节点存什么 | 整行数据 | 主键值 |
| 回表 | 聚簇索引查询直接返回数据 | 查到主键 → 再查聚簇索引（回表） |
| 插入速度 | 慢（需按主键顺序插入） | 快（叶子只存主键，不影响数据顺序） |
| 空间 | 小（索引即数据） | 大（需额外存主键） |
| 查询速度 | 主键/范围查询快 | 等值查询快 |

### 2-3 隔离级别对比

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 |
|---------|------|-----------|------|---------|
| **READ UNCOMMITTED** | ✅ 可能 | ✅ 可能 | ✅ 可能 | 无锁 |
| **READ COMMITTED** | ❌ 不可能 | ✅ 可能 | ✅ 可能 | MVCC + 行锁 |
| **REPEATABLE READ**（MySQL默认） | ❌ 不可能 | ❌ 不可能 | ✅ 可能 | MVCC + Gap Lock |
| **SERIALIZABLE** | ❌ 不可能 | ❌ 不可能 | ❌ 不可能 | 全表锁，串行执行 |

> **注意**：MySQL InnoDB 在 RR 级别下，通过 Next-Key Lock 基本解决了幻读，所以实际生产中够用。

### 2-4 SQL 执行顺序

```
FROM → ON → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
 |      |        |         |           |         |        |
 |   连接条件  匹配合并    过滤行      分组      聚合      选择列   去重       排序    分页
```

### 2-5 JOIN 类型对比

| 类型 | 说明 | 保留 |
|------|------|------|
| INNER JOIN | 只保留两边都匹配的行 | A ∩ B |
| LEFT JOIN | 左表全部保留 | A |
| RIGHT JOIN | 右表全部保留 | B |
| FULL OUTER JOIN | 两边都保留（MySQL 不直接支持） | A ∪ B |
| CROSS JOIN | 笛卡尔积（不推荐） | A × B |

### 2-6 B+ Tree vs Hash

| 维度 | B+ Tree 索引 | Hash 索引 |
|------|------------|---------|
| 查询类型 | 范围查询（> < BETWEEN）✅ | 等值查询（=）✅ |
| 排序 | 支持（叶子有序）✅ | 不支持 ❌ |
| 最左前缀 | 遵守（联合索引）✅ | 不适用 ❌ |
| 数据量 | 大数据量性能稳定 | 小数据量极快 |
| 内存 | 可部分加载 | 全量加载 |
| 适用场景 | 通用，**默认选 B+ Tree** | MEMORY 存储引擎 |
| 面试结论 | **必选** | 仅特定场景 |

---

## 3️⃣ 图解结构

### 3-1 InnoDB 聚簇索引（B+ Tree）

```
聚簇索引结构（主键为索引，数据按主键顺序存储在叶子节点）：

                  ┌─────────────────────────────────────┐
                  │           非叶子节点（索引页）         │
                  │                                      │
                  │     [15]          [30]          [50] │
                  │    /    \        /    \        /    \ │
                  └─────────────────────────────────────┘
                       /           \           \
        ┌─────────────┐     ┌─────────────┐   ┌─────────────┐
        │  索引页    │     │  索引页     │   │  索引页     │
        │ [1][5][10] │     │[15][20][25]│   │[30][40][50]│
        └─────────────┘     └─────────────┘   └─────────────┘
              ↓                    ↓                  ↓
        ┌───────────┐       ┌───────────┐        ┌───────────┐
        │  叶子节点  │       │  叶子节点  │        │  叶子节点  │
        │ (数据页)   │       │ (数据页)   │        │ (数据页)   │
        │           │       │           │        │           │
        │ id=1→row1 │       │ id=15→row│        │ id=50→row│
        │ id=5→row2 │       │ id=20→row│        │ id=60→row│
        │ id=10→row3│       │ id=25→row│        │ id=70→row│
        │ (主键顺序) │       │ (主键顺序) │        │ (主键顺序) │
        │     ↓      │       │     ↓      │        │     ↓      │
        │  下一个叶子 │       │  下一个叶子 │        │   链表     │
        └───────────┘       └───────────┘        └───────────┘
                    ↘                ↘                    ↘
                  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
                  │  id=1      │ │ id=15       │ │ id=30       │
                  │ username:张 │ │ username:李  │ │ username:王  │
                  │ age: 25    │ │ age: 30     │ │ age: 22     │
                  │ email:...  │ │ email:...   │ │ email:...   │
                  │ (完整行数据)│ │ (完整行数据) │ │ (完整行数据) │
                  └─────────────┘ └─────────────┘ └─────────────┘

特点：
- 非叶子节点只存索引（主键值 + 指针），不存数据
- 叶子节点按主键顺序存储完整行数据（数据即索引）
- 叶子节点之间有双向链表（范围查询快：顺序访问）
- 树高：3 层可存 2000 万行（通常 3 层够用）
```

### 3-2 非聚簇索引（回表查询）

```
普通索引（username）：

非聚簇索引（非主键字段）：

                  ┌──────────────────────────────────────┐
                  │          非叶子节点                   │
                  │      [李]        [王]            [张] │
                  └──────────────────────────────────────┘
                        ↓              ↓              ↓
                  ┌───────────┐  ┌───────────┐   ┌───────────┐
                  │  叶子节点  │  │  叶子节点  │   │  叶子节点  │
                  │  叶子节点  │  │  叶子节点  │   │  叶子节点  │
                  │           │  │           │   │           │
                  │ 李→id=15  │  │ 王→id=30  │   │ 张→id=1   │
                  │ 张→id=1   │  │ 赵→id=50  │   │ 周→id=5   │
                  │ 周→id=5   │  │           │   │           │
                  │ (按username│  │ (按username│   │ (按username│
                  │  顺序排列) │  │  顺序排列) │   │  顺序排列) │
                  └───────────┘  └───────────┘   └───────────┘
                        ↓              ↓              ↓
                  主键值：id=15  主键值：id=30  主键值：id=1,5
                        ↓              ↓              ↓
                  回到聚簇索引，用主键值查完整数据（回表！）
                        ↓              ↓              ↓
                  ┌──────────────────────────────────────┐
                  │         聚簇索引（主键 id）            │
                  │      id=1→完整数据   id=5→完整数据     │
                  │     id=15→完整数据  id=30→完整数据     │
                  └──────────────────────────────────────┘

SELECT username, age FROM user WHERE username = '李四'
    │
    ↓
1. 在 username 索引找到 id=15
2. 用 id=15 回表查聚簇索引，取出完整行
3. 从行中取 username 和 age

优化：覆盖索引（不用回表）
  SELECT username FROM user WHERE username = '李四'  -- username 索引已覆盖
  SELECT username, age FROM user WHERE username = '李四' AND age = 30
  -- 如果有联合索引 (username, age)，则不用回表（Extra: Using index）
```

### 3-3 联合索引最左前缀

```
联合索引 (status, create_time, age)：

索引结构：
  ┌──────────────────────────────────────────────────┐
  │              B+ Tree（按 (status, create_time, age) 排序）│
  │                                                   │
  │  第一层（status）：先按 status 排序                │
  │  ┌─────────────────────────────────────────────┐  │
  │  │ 0 (禁用)    │  1 (正常)    │  2 (VIP)       │  │
  │  └─────────────────────────────────────────────┘  │
  │        ↓           ↓              ↓                │
  │   第二层      第二层        第二层                  │
  │ (create_time)(create_time)(create_time)            │
  │  按 status    按 status    按 status              │
  │  分组后，再    分组后，再    分组后，再              │
  │  按时间排序    按时间排序    按时间排序              │
  └──────────────────────────────────────────────────┘

匹配规则：
  ✅ 能命中索引：                   ❌ 不能命中索引：
  - WHERE status = 1               - WHERE age = 25
  - WHERE status = 1 AND create_time > '2026-01-01'   - WHERE create_time > '2026-01-01'
  - WHERE status = 1 AND age = 25  - WHERE create_time > '2026-01-01' AND age = 25
  - WHERE status = 1 AND create_time = '2026-01-01' AND age = 25

  ⚠️ 范围查询断链（中断点之后的列无法利用索引）：
  - WHERE status >= 1（可用 status）         ✅
  - WHERE status = 1 AND create_time > '2026-01-01' AND age = 25
      ✅ status=1 索引有序 → ✅ create_time 范围 → ❌ age 无法用索引（断链）
  - WHERE status IN (1, 2)（IN 内部等值，不算范围） ✅ 仍可用全索引

记忆口诀：带头（status）不走尾（age），中间范围（between/>）会断链
```

### 3-4 事务隔离级别与并发问题

```
三个并发问题：

脏读（Dirty Read）—— READ UNCOMMITTED 下发生
  T1: BEGIN; UPDATE account SET balance=0 WHERE id=1;
  T2: SELECT balance FROM account WHERE id=1;  -- 读到 0（脏数据，T1 还没提交）
  T1: ROLLBACK;  -- T1 回滚，数据恢复
  T2: 用 0 做业务决策（错误！）

不可重复读（Non-Repeatable Read）—— READ COMMITTED 下发生
  T1: BEGIN; SELECT balance FROM account WHERE id=1;  -- 100
  T2: UPDATE account SET balance=50 WHERE id=1; COMMIT;
  T1: SELECT balance FROM account WHERE id=1;  -- 50（同一事务内两次读结果不同！）
  T1: COMMIT;

幻读（Phantom Read）—— REPEATABLE READ 下发生
  T1: BEGIN; SELECT COUNT(*) FROM user WHERE age>20; -- 3条
  T2: INSERT INTO user (age) VALUES (25); COMMIT;    -- 新插入一条
  T1: SELECT COUNT(*) FROM user WHERE age>20; -- 4条！（幻行）
  T1: INSERT INTO user (age) VALUES (25);             -- 报错：Duplicate entry！
  -- InnoDB RR 级别用 Next-Key Lock 解决
```

### 3-5 MVCC 原理（Read View）

```
MVCC = Multi-Version Concurrency Control（多版本并发控制）

核心思想：每个事务看到的是某个时间点的快照，而非最新数据。

InnoDB 每行数据有两个隐藏列：
  - DB_TRX_ID：最近修改的事务ID
  - DB_ROLL_PTR：指向 undo log 链表的指针

读已提交（RC）—— 每次 SELECT 都生成新的 Read View：
  T1: BEGIN; SELECT * FROM user WHERE id=1;  -- ReadView: {T1, T2可见}
  T2: COMMIT;  -- 提交
  T1: SELECT * FROM user WHERE id=1;  -- 重新生成 ReadView: {T1可见}，看到新数据

可重复读（RR）—— 事务开始时生成 Read View，之后一直用同一个：
  T1: BEGIN;                         -- 生成 ReadView: {T1}
  T2: BEGIN; UPDATE...; COMMIT;      -- T2 提交
  T1: SELECT * FROM user WHERE id=1;  -- 用同一个 ReadView，不见 T2 的修改
  T1: COMMIT;                        -- 提交后才看到 T2 的修改

快照读 vs 当前读：
  - 快照读：SELECT（不加锁），读取快照，MVCC 控制
  - 当前读：SELECT ... FOR UPDATE / LOCK IN SHARE MODE，读取最新数据，加锁
```

---

## 4️⃣ 速查清单

### 4-1 EXPLAIN 字段速查

| 字段 | 值 | 含义 |
|------|-----|------|
| `type` | system/const/eq_ref/ref/range/index/ALL | 连接类型，性能好→差 |
| `key` | 索引名/null | 实际用到的索引 |
| `rows` | 数字 | 估算扫描行数，越少越好 |
| `Extra` | Using index/Using where/Using filesort/Using temporary | 重要优化信号 |

**Extra 速查：**
- `Using index` = **覆盖索引**，不用回表，最优 ✅
- `Using index condition` = 索引下推（ICP），好 ✅
- `Using where` = 需要在 Server 层过滤，一般 ⚠️
- `Using filesort` = **需要文件排序**，必须优化 ❌
- `Using temporary` = **用了临时表**，必须优化 ❌
- `Using join buffer` = 嵌套循环查，小心 ❌

### 4-2 慢查询优化速查

```sql
-- 开启慢查询日志
SHOW VARIABLES LIKE 'slow_query_log';
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超过1秒记录

-- 查看慢查询
SHOW GLOBAL STATUS LIKE 'Slow_queries';
SHOW VARIABLES LIKE 'slow_query_log_file';
-- 或者：SELECT * FROM mysql.slow_log;

-- EXPLAIN 分析步骤
EXPLAIN SELECT ... FROM user WHERE ...;
EXPLAIN ANALYZE SELECT ... FROM user WHERE ...;  -- MySQL 8.0+，估算 vs 实际

-- 强制使用索引
SELECT * FROM user FORCE INDEX (idx_age) WHERE age > 20;
-- 忽略索引
SELECT * FROM user IGNORE INDEX (idx_old) WHERE ...;

-- 查看执行计划（MySQL 8.0）
EXPLAIN FORMAT=JSON SELECT * FROM user WHERE age > 20;
```

### 4-3 数据库设计范式速查

| 范式 | 要求 | 示例 |
|------|------|------|
| **1NF（第一范式）** | 属性不可再分 | `地址`拆为`省`+`市`+`区` |
| **2NF（第二范式）** | 非主属性完全依赖主键 | 不能存在部分依赖（联合主键时） |
| **3NF（第三范式）** | 非主属性不传递依赖主键 | 消除传递依赖（学号→院系→院长） |
| **BCNF** | 主属性对候选键无缺失依赖 | 解决主属性间的依赖 |
| **反范式** | 为性能牺牲范式（如冗余字段） | 订单表冗余商品名称 |

### 4-4 常用 SQL 速查

```sql
-- 分页
SELECT * FROM user LIMIT 0, 10;

-- 排名（MySQL 8.0+）
SELECT name, score,
    RANK() OVER (ORDER BY score DESC) AS 'rank',       -- 并列跳号（1,1,3）
    DENSE_RANK() OVER (ORDER BY score DESC) AS 'dense',-- 并列不跳（1,1,2）
    ROW_NUMBER() OVER (ORDER BY score DESC) AS 'row'   -- 不并列（1,2,3）
FROM student;

-- 分组 TopN（每组取前3）
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY score DESC) AS rn
    FROM employee
) t WHERE rn <= 3;

-- 时间序列（近7天每天数据）
SELECT DATE(create_time) AS day, COUNT(*)
FROM user
WHERE create_time >= DATE_SUB(CURDATE(), INTERVAL 7 DAY)
GROUP BY day ORDER BY day;

-- 同比/环比
SELECT
    DATE_FORMAT(create_time, '%Y-%m') AS month,
    COUNT(*),
    LAG(COUNT(*), 1) OVER (ORDER BY DATE_FORMAT(create_time, '%Y-%m')) AS last_month
FROM user GROUP BY month;
```

---

## 5️⃣ 场景选择器

### 5-1 什么时候该建索引？

```
该不该建索引？
         │
    ┌────┴─────────────────────────────┐
    │                                   │
  数据量 < 10 万行？                  数据量大，且查询频繁？
    │                                   │
    │                                   ↓
    ↓                               是唯一字段？
  不用，线性扫描够快                  │
    │                                   ↓
    │                               WHERE / ORDER BY / JOIN ON 中出现的字段？
    │                                   │
    ↓                                   ↓
    不建索引                        是 → 建索引
```

**必须建索引的字段：**
- WHERE 条件中的等值字段（`WHERE status = 1`）
- ORDER BY / GROUP BY 中的字段
- JOIN 的 ON 条件字段（外键）
- DISTINCT 字段

**不建索引的字段：**
- 重复度高的字段（如性别，只有 0/1，B+ Tree 扫描成本高）
- 频繁更新的字段（维护成本高）
- TEXT/BLOB/LONGTEXT 整列

### 5-2 慢查询怎么优化？

```
发现慢查询（>1秒）？
         │
    ┌────┴──────────────────────────────┐
    │                                      │
  EXPLAIN 分析 type 列？                    │
    │                                      │
    ↓                                      ↓
  type = ALL（全表扫描）？           type = index/range？
    │                                      │
    ↓                                      ↓
  建索引                                看 key 列（用了索引？）
  WHERE 条件字段加索引                      │
  ORDER BY 字段加索引                      ↓
    │                                 key = null（没用到索引）？
    ↓                                      │
    ↓                                 检查字段是否有函数/隐式转换
    继续分析 Extra                          │
    │                                      ↓
    ↓                                 有 USING filesort / temporary？
    Extra 有 Using filesort？               │
    │                                      ↓
    ↓                                 ORDER BY 字段加入联合索引
  联合索引覆盖 ORDER BY + WHERE            按最左前缀排序
    │
    ↓
  Extra 有 Using temporary？
    ↓
  GROUP BY 字段加联合索引
  减少 SELECT *，只用必要字段
```

### 5-3 分库分表时机？

```
什么时候考虑分库分表？
         │
    ┌────┴──────────────────────────────┐
    │                                     │
  单表 > 500 万行？                    单库 QPS > 2000？
    │                                     │
    ↓                                     ↓
    ↓                                     ↓
  MySQL 单表性能急剧下降             数据库连接数成为瓶颈
  考虑垂直分（拆字段）               考虑水平分（拆库）
  或水平分（拆数据）                    │
    │                                     ↓
    ↓                                 TiDB / CockroachDB
  ShardingSphere                     （分布式 NewSQL）
```

---

## ❓ 常见面试题

**Q1: InnoDB 为什么用 B+ Tree 而不是 B Tree 或红黑树？**
> B+ Tree 非叶子节点不存数据，只存索引（一个节点可存更多索引），树高更低（3 层可索引 2000 万行），且叶子节点有双向链表，范围查询只需顺序遍历叶子节点，无需回旋。红黑树高且随机 IO 多，B Tree 非叶子节点存数据，单节点索引密度低。

**Q2: 什么是回表？如何避免？**
> 非聚簇索引查到主键后，还需再查一次聚簇索引才能取到完整数据，叫回表。避免方法：尽量用覆盖索引（SELECT 的字段和 WHERE 条件都在索引中）、尽量不回表（只选必要字段）。

**Q3: 事务的 ACID 是什么？MySQL 是怎么保证的？**
> 原子性（Atomicity）：undo log 回滚未提交事务。隔离性（Isolation）：MVCC + 锁机制。一致性（Consistency）：前两者保证+数据库约束（主键/唯一/外键）。持久性（Durability）：redo log 崩溃恢复 + 事务提交后写入磁盘。

**Q4: MySQL 的 MVCC 是什么？解决了什么问题？**
> Multi-Version Concurrency Control，通过 Read View 快照实现。每个事务在开始时生成 Read View，之后读到的都是该快照的数据（RR），或每次 SELECT 重新生成（RC）。解决了读写并发阻塞问题，实现非阻塞读。

**Q5: 主从同步原理是什么？如何解决延迟？**
> 主库写操作记录 binlog，从库 IO 线程拉取 binlog 写入 relay log，SQL 线程执行 relay log。延迟原因：主从库网络、从库硬件差、并发大、事务太长。解决：并行复制（MySQL 8.0 原生支持）、缩短事务、监控延迟、用读写分离中间件。

**Q6: CHAR 和 VARCHAR 的区别？**
> CHAR 是定长，VARCHAR 是变长。CHAR(N) 无论实际长度，都占 N 字节，查询快（无需计算长度）。VARCHAR(N) 只存实际长度+1~2字节长度前缀，省空间但查询稍慢。**MySQL 5.7+ VARCHAR 自动变长（1~2 字节存长度前缀）**。

**Q7: DELETE/TRUNCATE/DROP 的区别？**
> DELETE 是 DML，逐行删除，记录日志，可回滚，支持 WHERE 条件触发 Trigger。TRUNCATE 是 DDL，清空整表，不记录逐行日志，快速，不触发 Trigger，不可回滚。DROP 是 DDL，删除表结构+数据，释放空间，不可恢复。

## 📊 学习状态

- [x] DDL（建表/约束/索引）
- [x] DML（INSERT/UPDATE/DELETE/SELECT）
- [x] 多表查询（JOIN/子查询/UNION）
- [x] 聚合函数与 GROUP BY
- [x] 事务（ACID/隔离级别/MVCC）
- [x] InnoDB 锁机制（行锁/表锁/间隙锁）
- [x] 索引原理（B+ Tree）
- [x] 聚簇索引 vs 非聚簇索引
- [x] 联合索引与最左前缀
- [x] EXPLAIN 分析
- [x] 慢查询优化
- [ ] 主从复制与读写分离
- [ ] 分库分表（ShardingSphere）
- [ ] 数据库备份与恢复

## 🐛 踩坑记录

- **SELECT \*** 是性能杀手，线上必须只查需要的字段，减少网络传输和回表
- **TEXT/BLOB 字段不要建索引**，单独存一张表或用对象存储
- **JOIN 慎用**，超过 3 张表 JOIN 要考虑拆分；小表驱动大表（INNER JOIN 时小表放左边）
- **分页 LIMIT 大 OFFSET 会慢**：`LIMIT 1000000, 10` 先扫 100 万行再取 10 行，优化用 `WHERE id > ? LIMIT 10`
- **NULL 列的索引问题**：MySQL 5.7+，NULL 值可以用索引（IS NULL 能用索引，IS NOT NULL 不一定）
- **隐式类型转换**：字符串字段存数字，WHERE 条件写数字会全表扫描（MySQL 自动转换导致索引失效）
- **OR 条件索引失效**：OR 连接的每个条件都必须有索引，否则全表扫描；改用 UNION 分离
- **分页 count(\*) 慢**：数据量大时分页查询 COUNT 很慢，可用缓存或游标分页
- **大事务风险**：一个事务处理太多数据，导致主从延迟大、锁持有时间长；拆成小事务
- **唯一索引 vs 普通索引**：唯一索引查到数据后停止，普通索引需扫描到第一个不满足；写入性能唯一索引稍慢
- **ORDER BY RAND() 禁止在生产用**：随机取一条 `ORDER BY RAND()` 会生成临时表排序，10万行以上极慢，改用 `ORDER BY id LIMIT 1` 随机起点

---

## 🔗 相关笔记

- [[MySQL数据库]]
