# MySQL 数据库

> MySQL 是后端开发最核心的技能，实战天天用，面试必问。本笔记覆盖：SQL 基础、索引原理与优化、事务与隔离级别、存储引擎、查询优化、设计范式、常见问题排查。

---

## 1️⃣ 代码模板

### 1-1 DDL（数据定义语言）

```sql
-- ============ 数据库操作 ============
CREATE DATABASE mydb DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE mydb;
DROP DATABASE mydb;

-- ============ 表操作 ============
-- 创建表（完整版）
CREATE TABLE IF NOT EXISTS `user` (
    `id`          BIGINT        UNSIGNED  NOT NULL  AUTO_INCREMENT  COMMENT '用户ID',
    `username`    VARCHAR(50)             NOT NULL               COMMENT '用户名',
    `email`       VARCHAR(100)            NOT NULL               COMMENT '邮箱',
    `age`         TINYINT       UNSIGNED  DEFAULT 18             COMMENT '年龄',
    `status`      TINYINT       UNSIGNED  DEFAULT 1              COMMENT '状态：1正常 0禁用',
    `create_time` DATETIME                NOT NULL  DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `update_time` DATETIME                NOT NULL  DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_username` (`username`),
    UNIQUE KEY `uk_email` (`email`),
    KEY `idx_status` (`status`),
    KEY `idx_age` (`age`),
    KEY `idx_create_time` (`create_time`)
) ENGINE=InnoDB  DEFAULT CHARSET=utf8mb4  COMMENT='用户表';

-- 查看表结构
DESC user;
DESCRIBE user;
SHOW CREATE TABLE user;

-- 修改表结构
ALTER TABLE user ADD COLUMN phone VARCHAR(20) NOT NULL COMMENT '手机号' AFTER email;
ALTER TABLE user MODIFY COLUMN phone VARCHAR(30) NOT NULL COMMENT '手机号';
ALTER TABLE user DROP COLUMN phone;
ALTER TABLE user ADD INDEX idx_phone (phone);
ALTER TABLE user RENAME TO user_new;

-- 删除表
DROP TABLE IF EXISTS user;
-- 危险操作：TRUNCATE（清空表，自增从头开始，比 DELETE 快）
TRUNCATE TABLE user;

-- ============ 索引操作 ============
-- 主键索引（聚簇索引，一张表只能有一个）
ALTER TABLE user ADD PRIMARY KEY (id);

-- 唯一索引
ALTER TABLE user ADD UNIQUE KEY uk_email (email);

-- 普通索引
ALTER TABLE user ADD INDEX idx_age (age);
CREATE INDEX idx_status ON user(status);

-- 联合索引（最左前缀原则）
ALTER TABLE user ADD INDEX idx_status_create (status, create_time);
CREATE INDEX idx_age_status ON user(age, status);

-- 查看索引
SHOW INDEX FROM user;
SHOW INDEX FROM user\G

-- 删除索引
DROP INDEX idx_age ON user;
ALTER TABLE user DROP INDEX uk_email;

-- ============ 常用数据类型速查 ============
-- 整数
TINYINT   -- 1字节，-128~127（或 0~255 无符号）
SMALLINT  -- 2字节，-32768~32767
INT       -- 4字节，约±21亿（够用）
BIGINT    -- 8字节，极大数

-- 小数
DECIMAL(10,2)  -- 精确小数，总10位，小数2位（工资/金额必须用这个）
FLOAT         -- 4字节，不精确
DOUBLE        -- 8字节，不精确

-- 字符串
CHAR(20)      -- 固定长度，不足右侧补空格（定长：身份证号/手机号/邮编）
VARCHAR(255)  -- 可变长度，最大 65535 字节（存用户名/描述）
TEXT          -- 最多 65535 字节（存文章/评论）
LONGTEXT      -- 最多 4GB（存大文本）

-- 日期
DATE          -- '2026-08-16'，只有日期
DATETIME      -- '2026-08-16 16:40:00'，范围大（'1000-01-01'~'9999-12-31'）
TIMESTAMP     -- '2026-08-16 16:40:00'，自动更新，受时区影响，最大 '2038-01-19'
-- 推荐 DATETIME，不要用 TIMESTAMP（2038 年问题）

-- 布尔（MySQL TINYINT 实现）
BOOLEAN       -- TINYINT(1)，0=false，1=true
```

### 1-2 DML（数据操作语言）

```sql
-- ============ INSERT ============
-- 插入一行（全字段）
INSERT INTO user (username, email, age, status) VALUES ('张三', 'zhangsan@qq.com', 25, 1);

-- 插入多行
INSERT INTO user (username, email, age) VALUES
    ('李四', 'lisi@qq.com', 22),
    ('王五', 'wangwu@qq.com', 30),
    ('赵六', 'zhaoliu@qq.com', 28);

-- 插入查询结果
INSERT INTO user (username, email, age)
SELECT username, email, age FROM old_user WHERE status = 1;

-- 插入或更新（ON DUPLICATE KEY UPDATE）
INSERT INTO user (id, username, email) VALUES (1, '张三', 'new@qq.com')
ON DUPLICATE KEY UPDATE email = VALUES(email);

-- 插入或忽略（IGNORE）
INSERT IGNORE INTO user (username, email) VALUES ('张三', 'zhangsan@qq.com');

-- ============ UPDATE ============
-- 更新单行
UPDATE user SET age = 26, status = 1 WHERE id = 1;

-- 更新多行（根据条件批量）
UPDATE user SET status = 0 WHERE age < 18;

-- 更新前加条件检查（很重要！线上必须加 WHERE）
UPDATE user SET email = 'test@qq.com' WHERE id = 100;

-- ============ DELETE ============
-- 删除单行
DELETE FROM user WHERE id = 5;

-- 批量删除（根据条件）
DELETE FROM user WHERE status = 0 AND create_time < '2025-01-01';

-- 删除所有（慎用）
DELETE FROM user;  -- 逐行删除，有日志，可回滚
-- TRUNCATE TABLE user;  -- 整表重建，极快，无日志，不可回滚

-- ============ SELECT（最重要） ============
-- 查询所有字段
SELECT * FROM user;

-- 查询指定字段（避免 SELECT *）
SELECT id, username, email FROM user;

-- 别名（AS 可省略）
SELECT username AS '姓名', age AS '年龄' FROM user;

-- 去重
SELECT DISTINCT status FROM user;
SELECT COUNT(DISTINCT status) FROM user;  -- distinct 的数量

-- ============ WHERE 条件 ============
SELECT * FROM user WHERE id = 1;
SELECT * FROM user WHERE age > 18;
SELECT * FROM user WHERE age BETWEEN 18 AND 30;        -- 闭区间
SELECT * FROM user WHERE age IN (18, 20, 22, 25);
SELECT * FROM user WHERE username LIKE '张%';          -- % 任意字符
SELECT * FROM user WHERE username LIKE '张_';           -- _ 单个字符
SELECT * FROM user WHERE email LIKE '%@qq.com';        -- 邮箱后缀
SELECT * FROM user WHERE status = 1 AND age >= 18;     -- AND 优先级高于 OR
SELECT * FROM user WHERE (status = 1 OR status = 2) AND age > 20;
SELECT * FROM user WHERE status IS NULL;               -- NULL 判断必须用 IS NULL
SELECT * FROM user WHERE status IS NOT NULL;

-- 排序
SELECT * FROM user ORDER BY age ASC;                   -- 升序（默认）
SELECT * FROM user ORDER BY age DESC;                  -- 降序
SELECT * FROM user ORDER BY age DESC, create_time ASC;  -- 多字段排序

-- 分页（MySQL 用 LIMIT，Oracle 用 ROWNUM）
SELECT * FROM user LIMIT 10;                            -- 前10条
SELECT * FROM user LIMIT 20, 10;                        -- 第21~30条（OFFSET 20）
-- 性能注意：LIMIT 1000000, 10 会先扫100万行，再取10行（很慢）
-- 优化：WHERE id > 1000000 LIMIT 10（基于主键）

-- ============ 聚合函数 + GROUP BY ============
SELECT COUNT(*) FROM user;                    -- 总行数
SELECT COUNT(id) FROM user WHERE status = 1;  -- 非NULL的id数量
SELECT COUNT(DISTINCT status) FROM user;       -- 去重计数
SELECT SUM(age) FROM user;                    -- 求和
SELECT AVG(age) FROM user;                    -- 平均值
SELECT MAX(age) FROM user;                    -- 最大值
SELECT MIN(age) FROM user;                    -- 最小值

-- GROUP BY（分组，SELECT 后的字段必须是 GROUP BY 的字段，或聚合函数）
SELECT status, COUNT(*) AS cnt FROM user GROUP BY status;
SELECT status, AVG(age) FROM user GROUP BY status HAVING AVG(age) > 20;  -- HAVING 过滤分组后
SELECT DATE(create_time) AS day, COUNT(*) FROM user GROUP BY DATE(create_time);

-- 完整 SELECT 顺序（面试常考）
-- SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY ... LIMIT ...
-- 记忆口诀：W G H O L（Where Group Having Order Limit）
```

### 1-3 多表查询

```sql
-- ============ 连接（JOIN） ============
-- 创建示例表
CREATE TABLE department (
    id INT PRIMARY KEY,
    name VARCHAR(50)
);
CREATE TABLE employee (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    dept_id INT,
    salary DECIMAL(10,2)
);

-- 内连接（INNER JOIN）：只保留两边都匹配的行
SELECT e.name, d.name AS dept
FROM employee e
INNER JOIN department d ON e.dept_id = d.id;

-- 左外连接（LEFT JOIN）：左表全部保留，右表没有匹配则为 NULL
SELECT e.name, d.name AS dept
FROM employee e
LEFT JOIN department d ON e.dept_id = d.id;

-- 右外连接（RIGHT JOIN）：右表全部保留（尽量少用）
SELECT e.name, d.name AS dept
FROM employee e
RIGHT JOIN department d ON e.dept_id = d.id;

-- 满外连接（MySQL 不直接支持，用 UNION 模拟）
(SELECT e.name, d.name FROM employee e LEFT JOIN department d ON e.dept_id = d.id)
UNION
(SELECT e.name, d.name FROM employee e RIGHT JOIN department d ON e.dept_id = d.id);

-- 自然连接（NATURAL JOIN）：自动按同名列合并（不推荐，容易出错）
SELECT * FROM employee NATURAL JOIN department;

-- USING 简化（同名列）
SELECT e.name, d.name FROM employee e JOIN department d USING(id);

-- ============ 子查询 ============
-- 标量子查询（返回单个值）
SELECT * FROM user WHERE age > (SELECT AVG(age) FROM user);

-- 列子查询（返回一列）
SELECT * FROM user WHERE status IN (SELECT status FROM vip_user);

-- 行子查询（返回一行）
SELECT * FROM user WHERE (age, status) = (SELECT MAX(age), MAX(status) FROM user);

-- 表子查询（FROM 后用）
SELECT t.status, COUNT(*) FROM (
    SELECT * FROM user WHERE status = 1
) t GROUP BY t.status;

-- EXISTS 子查询（判断是否存在）
SELECT * FROM user WHERE EXISTS (
    SELECT 1 FROM order WHERE order.user_id = user.id
);

-- ============ UNION ============
SELECT username FROM user WHERE age < 20
UNION
SELECT username FROM vip_user WHERE age < 20;
-- UNION 自动去重，UNION ALL 不去重（更快）

-- ============ 常用函数 ============
-- 字符串函数
SELECT CONCAT('Hello', ' ', 'World');          -- Hello World
SELECT UPPER('hello');                          -- HELLO
SELECT LOWER('HELLO');                          -- hello
SELECT LENGTH('hello');                         -- 5（字节数）
SELECT CHAR_LENGTH('hello');                    -- 5（字符数）
SELECT SUBSTRING('HelloWorld', 1, 5);           -- Hello（从1开始）
SELECT TRIM('  hello  ');                       -- hello（去首尾空格）
SELECT REPLACE('hello world', 'world', 'mysql');-- hello mysql

-- 日期函数
SELECT NOW();                                  -- 2026-08-16 16:40:00
SELECT CURDATE();                              -- 2026-08-16
SELECT DATE('2026-08-16 16:40:00');            -- 2026-08-16
SELECT YEAR(NOW());                            -- 2026
SELECT MONTH(NOW());                           -- 8
SELECT DAY(NOW());                             -- 16
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s');-- 2026-08-16 16:40:00
SELECT DATE_ADD(NOW(), INTERVAL 7 DAY);        -- 7天后
SELECT DATEDIFF('2026-08-20', '2026-08-16');   -- 4（天数差）
SELECT DATE_SUB(NOW(), INTERVAL 1 MONTH);      -- 1个月前

-- 条件函数
SELECT IF(age > 18, '成年', '未成年') FROM user;
SELECT IFNULL(phone, '无手机') FROM user;       -- NULL 替换
SELECT NULLIF(a, b);                            -- a=b 返回 NULL，否则返回 a
SELECT CASE status WHEN 1 THEN '正常' WHEN 0 THEN '禁用' ELSE '未知' END FROM user;
-- CASE 表达式（搜索型）
SELECT
    CASE
        WHEN age < 18 THEN '未成年'
        WHEN age < 30 THEN '青年'
        WHEN age < 60 THEN '中年'
        ELSE '老年'
    END AS '年龄段',
    COUNT(*)
FROM user
GROUP BY 1;
```

### 1-4 事务（Transaction）

```sql
-- ============ 事务控制 ============
-- 开启事务（方式1）
START TRANSACTION;
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- 提交，原子性生效

-- 开启事务（方式2）
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;
-- 出错回滚
ROLLBACK;  -- 回滚所有未提交的操作

-- 自动提交（MySQL 默认每条 SQL 一个事务）
SET autocommit = 0;  -- 关闭自动提交，之后每条 SQL 需手动 COMMIT
SET autocommit = 1;  -- 开启自动提交

-- ============ 保存点（部分回滚） ============
START TRANSACTION;
INSERT INTO user (name) VALUES ('A');
SAVEPOINT sp1;           -- 设置保存点
INSERT INTO user (name) VALUES ('B');
ROLLBACK TO sp1;          -- 回滚到保存点，只保留 A
COMMIT;

-- ============ 事务隔离级别 ============
-- 查看当前隔离级别
SELECT @@tx_isolation;  -- MySQL 5.7
SELECT @@transaction_isolation;  -- MySQL 8.0

-- 设置隔离级别（只影响当前会话）
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 隔离级别测试
-- READ UNCOMMITTED：最低，可脏读（基本不用）
-- READ COMMITTED（RC）：Oracle 默认，可重复读
-- REPEATABLE READ（RR）：MySQL InnoDB 默认，可能幻读
-- SERIALIZABLE：最高，完全串行，最安全但最慢

-- 脏读演示（RC 级别）
-- Session A: SET SESSION tx_isolation='READ-UNCOMMITTED';
-- Session A: SELECT balance FROM account WHERE id=1; -- 100
-- Session B: UPDATE account SET balance=50 WHERE id=1; -- 未提交
-- Session A: SELECT balance FROM account WHERE id=1; -- 50（脏读！脏数据）
-- Session B: ROLLBACK;
```

### 1-5 索引与 EXPLAIN

```sql
-- ============ EXPLAIN 分析查询 ============
EXPLAIN SELECT * FROM user WHERE id = 1;

-- 字段说明
-- id: 查询序号，值越大优先级越高，相同时从上往下执行
-- select_type: 查询类型
--   SIMPLE（简单查询） / PRIMARY（最外层） / SUBQUERY（子查询）
--   DERIVED（派生表-from 子句） / UNION / UNION RESULT
-- type: 连接类型（性能从好到差）
--   system > const > eq_ref > ref > range > index > ALL
--   const: 主键/唯一索引等值查询，最多1条
--   ref: 非唯一索引等值查询
--   range: 索引范围查询（> < BETWEEN IN）
--   ALL: 全表扫描（最差，要优化）
-- key: 实际使用的索引
-- key_len: 索引长度，越短越好
-- rows: 估算扫描行数，越少越好
-- Extra: 额外信息
--   Using index（覆盖索引） ✅ 好
--   Using where（需要回表查）⚠️ 一般
--   Using filesort（文件排序）❌ 差，需要优化
--   Using temporary（使用临时表）❌ 差，需要优化
--   Using index condition（索引下推）✅ 好

-- ============ 索引下推（ICP） ============
EXPLAIN SELECT * FROM user WHERE age > 20 AND username LIKE '张%';
-- 旧流程：先根据 age 索引找到所有 >20 的主键，再回表查完整行，最后在 Server 层过滤 username
-- 新流程（ICP）：在 Index 上直接过滤 username，减少回表次数

-- ============ MRR（Multi-Range Read） ============
-- 优化回表：按主键排序后批量读取，减少随机 IO
SET optimizer_switch='mrr=on';

-- ============ 强制使用索引 ============
SELECT * FROM user FORCE INDEX (idx_age) WHERE age > 20;
-- （谨慎使用，让优化器做决定）
```

### 1-6 锁（InnoDB）

```sql
-- ============ 共享锁/排他锁 ============
-- 共享锁（S锁）：读锁，多个事务可同时持有，不互斥
SELECT * FROM user WHERE id = 1 LOCK IN SHARE MODE;

-- 排他锁（X锁）：写锁，排斥其他所有锁
SELECT * FROM user WHERE id = 1 FOR UPDATE;  -- 悲观锁

-- ============ 间隙锁（Gap Lock） ============
-- 在索引记录之间的间隙加锁，防止幻读（RR 级别）
SELECT * FROM user WHERE age BETWEEN 20 AND 30 FOR UPDATE;
-- 锁住 age>20 和 age<30 之间的间隙，新插入 age=25 会被阻塞

-- ============ 临键锁（Next-Key Lock） ============
-- InnoDB 默认加 Next-Key Lock = 记录锁 + 间隙锁
-- 可通过配置降级为记录锁：
SET tx_isolation='REPEATABLE-READ';
ALTER TABLE user ADD INDEX idx_age (age);
SET autocommit=0;
SELECT * FROM user WHERE age=20 FOR UPDATE;  -- 只锁 age=20 这一行（记录锁）
COMMIT;

-- ============ 死锁演示 ============
-- T1: BEGIN; UPDATE user SET status=0 WHERE id=1;  -- 锁1
-- T2: BEGIN; UPDATE user SET status=0 WHERE id=2;  -- 锁2
-- T1: UPDATE user SET status=0 WHERE id=2;  -- 等 T2 的锁2
-- T2: UPDATE user SET status=0 WHERE id=1;  -- 等 T1 的锁1 → 死锁！
-- InnoDB 自动检测并回滚小事务

-- 查看死锁日志
SHOW ENGINE INNODB STATUS;

-- 设置锁等待超时
SET innodb_lock_wait_timeout = 5;  -- 等锁最多5秒
```

### 1-7 视图与存储过程

```sql
-- ============ 视图（View） ============
-- 视图：虚拟表，封装复杂查询
CREATE VIEW v_user_stats AS
SELECT
    status,
    COUNT(*) AS cnt,
    AVG(age) AS avg_age
FROM user
GROUP BY status;

-- 查询视图（像普通表一样用）
SELECT * FROM v_user_stats WHERE status = 1;

-- 视图分类
CREATE VIEW v_normal AS SELECT * FROM user;                 -- 可更新
CREATE VIEW v_group AS SELECT status, COUNT(*) FROM user GROUP BY status; -- 不可更新（聚合视图）

-- 删除视图
DROP VIEW IF EXISTS v_user_stats;

-- ============ 存储过程（Stored Procedure） ============
DELIMITER $$   -- 改分隔符（因为过程体里有分号）
CREATE PROCEDURE proc_get_user_by_status(IN p_status INT)
BEGIN
    SELECT * FROM user WHERE status = p_status;
END$$
DELIMITER ;

-- 调用存储过程
CALL proc_get_user_by_status(1);

-- 带 OUT 参数
DELIMITER $$
CREATE PROCEDURE proc_count_by_status(
    IN p_status INT,
    OUT p_count INT
)
BEGIN
    SELECT COUNT(*) INTO p_count FROM user WHERE status = p_status;
END$$
DELIMITER ;

-- 调用
CALL proc_count_by_status(1, @cnt);
SELECT @cnt;

-- 删除存储过程
DROP PROCEDURE IF EXISTS proc_get_user_by_status;
```

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
