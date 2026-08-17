---
tags:
  - 数据库
  - MySQL
  - 面试
created: 2026-08-17
---

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

---

## 🔗 相关笔记

- [[../04-Spring生态/SpringBoot框架|Spring Boot 框架]]
- [[../05-AI应用开发/SpringAI与RAG实战|Spring AI 与 RAG 实战]]

