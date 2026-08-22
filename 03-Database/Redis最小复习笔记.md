---
tags:
  - Redis
  - 缓存
  - 数据库
  - 面试
  - 分布式锁
  - NoSQL
created: 2026-08-22
---

# Redis 最小复习笔记


> 来源：黑马程序员 Redis 入门到实战教程（BV1cr4y1671t）
> 范围：19 集最小必看清单，覆盖面试 80% 考点
> 更新：2026-08-22

---

## 一、基础认知（3 集）

> 对应视频：基础篇 03 / 07 / 08

### 1.1 Redis 是什么

- 内存型 Key-Value 数据库，读写极快（10万+ QPS）
- 单线程 + IO 多路复用，避免线程切换开销
- 数据存内存，可持久化到磁盘（RDB / AOF）

### 1.2 NoSQL vs 关系型数据库

| | Redis（NoSQL） | MySQL（关系型） |
|---|---|---|
| 存储位置 | 内存 | 磁盘 |
| 数据结构 | Key-Value，无表结构 | 表、行、列、关系 |
| 事务 | 弱（不支持回滚） | ACID 完整事务 |
| 速度 | 极快 | 相对慢 |
| 用途 | 缓存、会话、排行榜 | 持久化存储、复杂查询 |

### 1.3 五大基本数据结构

| 类型 | 特点 | 典型场景 |
|---|---|---|
| String | 最基本，存字符串/数字 | 缓存对象、计数器、分布式锁 |
| Hash | 字段-值映射 | 存对象（比 String 存 JSON 更省空间） |
| List | 有序列表 | 消息队列、最新消息 |
| Set | 无序、不重复 | 共同关注、去重 |
| SortedSet | 有序、不重复、带分数 | 排行榜、延时队列 |

### 1.4 通用命令

| 命令 | 作用 |
|---|---|
| `KEYS pattern` | 查找所有符合的 key（生产禁用，阻塞） |
| `DEL key` | 删除 |
| `EXISTS key` | 判断是否存在 |
| `EXPIRE key seconds` | 设置过期时间 |
| `TTL key` | 查看剩余过期时间 |
| `TYPE key` | 查看类型 |

---

## 二、核心数据结构（2 集）

> 对应视频：基础篇 09 / 11

### 2.1 String 类型

```
SET key value              # 设置
GET key                    # 获取
SETNX key value            # 不存在才设置（分布式锁基础）
SETEX key seconds value   # 设置 + 过期时间
INCR key                   # 自增（计数器）
INCRBY key increment       # 指定步长自增
```

**存对象两种方式**：

| 方式 | 示例 | 优劣 |
|---|---|---|
| JSON 字符串 | `SET user:1 '{"name":"张三","age":20}'` | 简单，但改单个字段要全量重写 |
| Hash | `HSET user:1 name 张三 age 20` | 可改单个字段，更省内存 |

### 2.2 Hash 类型

```
HSET key field value        # 设置字段
HGET key field              # 获取字段
HGETALL key                 # 获取所有字段
HDEL key field              # 删除字段
HINCRBY key field increment # 字段自增
```

**面试要点**：Hash 存对象比 String 存 JSON 更适合频繁修改单个字段的场景。

---

## 三、Java 客户端（3 集）

> 对应视频：基础篇 19 / 21 / 22

### 3.1 SpringDataRedis（RedisTemplate）

```java
@Autowired
private RedisTemplate<String, Object> redisTemplate;

// 写
redisTemplate.opsForValue().set("name", "张三");
redisTemplate.opsForValue().set("name", "张三", 30, TimeUnit.MINUTES); // 带过期

// 读
Object name = redisTemplate.opsForValue().get("name");

// 删
redisTemplate.delete("name");

// 判断存在
redisTemplate.hasKey("name");
```

### 3.2 StringRedisTemplate（推荐）

`RedisTemplate` 默认用 JDK 序列化，存进去的可读性差、占空间大。**实际开发用 `StringRedisTemplate`**：

```java
@Autowired
private StringRedisTemplate stringRedisTemplate;

// 存对象时手动序列化
String json = JSON.toJSONString(user);
stringRedisTemplate.opsForValue().set("user:1", json);

// 取出来反序列化
String json = stringRedisTemplate.opsForValue().get("user:1");
User user = JSON.parseObject(json, User.class);
```

| | RedisTemplate | StringRedisTemplate |
|---|---|---|
| 序列化 | JDK（不可读） | String（可读） |
| 推荐度 | 不推荐 | **推荐** |

### 3.3 Hash 操作

```java
// 存 Hash
stringRedisTemplate.opsForHash().put("user:1", "name", "张三");
stringRedisTemplate.opsForHash().put("user:1", "age", "20");

// 取单个
Object name = stringRedisTemplate.opsForHash().get("user:1", "name");

// 取所有
Map<Object, Object> map = stringRedisTemplate.opsForHash().entries("user:1");
```

---

## 四、Redis 替代 Session（3 集）★ 必看

> 对应视频：实战篇 08 / 09 / 10
> 和你刚学的 Session/Cookie 直接关联

### 4.1 单机 Session 的问题

```
用户登录 → Session 存在服务器 A 内存里
    ↓
Nginx 负载均衡 → 下次请求打到服务器 B
    ↓
服务器 B 没有你的 Session → 被迫重新登录
```

**核心痛点**：多台服务器时 Session 默认不共享，用户可能反复被踢下线。

### 4.2 Redis 替代 Session 的方案

```
用户登录
    ↓
服务器验证通过
    ↓ 不再存本地 Session
    ↓ 改为存入 Redis
    ↓ key = "session:token:xxx"，value = 用户信息
    ↓
返回 token 给前端（放 Cookie 或请求头）
    ↓
后续请求带 token
    ↓
服务器从 Redis 查 token → 找到用户信息 → 确认身份
```

### 4.3 关键实现

```java
// 登录成功后
String token = UUID.randomUUID().toString();
String tokenKey = "login:token:" + token;
// 存入 Redis，设过期时间
stringRedisTemplate.opsForValue().set(tokenKey, JSON.toJSONString(user), 30, TimeUnit.MINUTES);
// token 返回前端

// 后续请求拦截器里校验
String token = request.getHeader("authorization");
String userJson = stringRedisTemplate.opsForValue().get("login:token:" + token);
if (userJson == null) {
    // 未登录或过期，拦截
}
User user = JSON.parseObject(userJson, User.class);
// 存入 ThreadLocal 供后续业务使用
```

### 4.4 面试标准答案

> 问：你项目登录怎么实现的？
>
> 答：验证密码 → 生成 token 存 Redis（key 带 token、value 存用户信息、设过期）→ token 返给前端放 Cookie → 拦截器从请求拿 token 查 Redis → 命中则放行并存 ThreadLocal → 未命中拦截跳登录。

> 问：为什么用 Redis 而不是本地 Session？
>
> 答：多台服务器时本地 Session 不共享，Redis 作为集中存储解决共享问题，且自带过期机制，方便管理登录态。

---

## 五、缓存三大问题（6 集）★ 面试必问

> 对应视频：商户缓存 01 / 02 / 04 / 06 / 08 / 09

### 5.1 基本缓存流程

```
查询请求
    ↓
先查 Redis → 命中 → 直接返回
    ↓ 未命中
查数据库 → 写入 Redis → 返回
```

### 5.2 缓存更新策略

| 策略 | 原理 | 适用 |
|---|---|---|
| 内存淘汰 | Redis 自己淘汰，不主动更新 | 一致性低，偶尔用 |
| 超时失效 | 给 key 设 TTL，到期重建 | 简单，有短暂不一致 |
| 主动更新 | 改数据库时同步更新 Redis | 一致性最高，代码复杂 |

**面试推荐答**：用"超时 + 主动更新"组合——改 DB 时删 Redis（不是改，让下次查询重建），再加 TTL 兜底。

### 5.3 三大问题对照（必背）

| | 缓存穿透 | 缓存击穿 | 缓存雪崩 |
|---|---|---|---|
| **什么问题** | 查根本不存在的数据 | 热点 key 过期瞬间 | 大量 key 同时过期 |
| **发生场景** | 恶意攻击、查不存在的 id | 秒杀、热门商品 | 缓存批量过期 |
| **后果** | 每次都打到数据库 | 瞬间高并发打到数据库 | 数据库瞬间压力暴增 |
| **解法** | 缓存空值 / 布隆过滤器 | 互斥锁 / 逻辑过期 | TTL 加随机值 / 多级缓存 |

### 5.4 三大问题详解

#### 缓存穿透

```
查 user:id=999999（不存在）
    ↓
Redis 没缓存
    ↓
打到数据库
    ↓
数据库也没有 → 返回 null
    ↓
不写 Redis → 下次查又打数据库
```

**解法**：
- 缓存空值：数据库没有也存 `null` 进 Redis，设短 TTL（如 2 分钟）
- 布隆过滤器：查之前先过滤，不存在的直接拦掉

#### 缓存击穿

```
热点 key（如秒杀商品）正好过期
    ↓
瞬间 1 万个请求同时发现缓存没了
    ↓
1 万个请求同时打数据库
    ↓
数据库扛不住
```

**解法**：
- 互斥锁：只让一个线程去查数据库重建缓存，其他线程等待
- 逻辑过期：不设真实 TTL，存一个逻辑过期时间，过期后异步重建

#### 缓存雪崩

```
大量 key 设了相同 TTL
    ↓
同一时刻集体过期
    ↓
数据库瞬间收到大量查询
```

**解法**：
- TTL 加随机值，避免同时过期
- 多级缓存（Redis + 本地 Caffeine）
- 限流降级保护数据库

### 5.5 面试标准答案

> 问：缓存穿透/击穿/雪崩分别是什么？怎么解决？
>
> 答：
> - 穿透：查不存在的数据，每次打 DB。解法：缓存空值 + 布隆过滤器
> - 击穿：热点 key 过期瞬间大量请求打 DB。解法：互斥锁或逻辑过期
> - 雪崩：大量 key 同时过期。解法：TTL 加随机值、多级缓存、限流降级

---

## 六、分布式锁概念（2 集）

> 对应视频：实战篇 09 / 17

### 6.1 为什么需要分布式锁

```
单机多线程 → 用 synchronized 或 ReentrantLock
    ↓
但多台服务器时，每台 JVM 独立
    ↓
本地锁管不了跨进程
    ↓
需要一把"所有服务器都认"的锁 → 分布式锁
```

### 6.2 Redis 实现分布式锁

核心原理：**SETNX + 过期时间**

```java
// 加锁：不存在才设置，设过期（防死锁）
Boolean locked = stringRedisTemplate.opsForValue()
    .setIfAbsent("lock:order:" + orderId, threadId, 30, TimeUnit.SECONDS);

if (locked) {
    // 拿到锁，执行业务
} else {
    // 没拿到锁，等待或失败
}

// 释放锁：删除 key（要判断是不是自己的锁，防误删别人的）
String value = stringRedisTemplate.opsForValue().get("lock:order:" + orderId);
if (threadId.equals(value)) {
    stringRedisTemplate.delete("lock:order:" + orderId);
}
```

### 6.3 自己实现的坑

| 问题 | 原因 | 解决 |
|---|---|---|
| 死锁 | 加锁后宕机没释放 | 加过期时间 |
| 误删别人的锁 | 锁过期被自动释放，原线程删了新线程的锁 | 存唯一标识，删之前判断 |
| 判断+删除不原子 | 判断完到删除之间锁过期 | Lua 脚本保证原子性 |
| 不可重入 | 同线程不能多次加锁 | Redisson 的 Hash 结构 |

### 6.4 Redisson（企业级方案）

**实际开发不自己写，用 Redisson**：

```java
@Autowired
private RedissonClient redissonClient;

RLock lock = redissonClient.getLock("lock:order:" + orderId);
try {
    if (lock.tryLock(10, TimeUnit.SECONDS)) {
        // 拿到锁，执行业务
    }
} finally {
    lock.unlock(); // 确保释放
}
```

Redisson 解决了：
- 可重入锁（Hash 结构计数）
- 自动续期（WatchDog 机制，默认 10 秒续一次）
- 锁重试（失败等待重试）
- 多锁联合（MultiLock）

### 6.5 面试标准答案

> 问：你项目怎么解决并发安全问题？
>
> 答：单机用 synchronized，多机用 Redis 分布式锁。底层 SETNX + 过期时间，但自己实现有误删、不原子等问题，实际用 Redisson，它提供可重入锁、WatchDog 自动续期、锁重试等完整方案。

---

## 七、复习 Checklist

看完后自测，答得上来打勾：

- [ ] Redis 是什么、和 MySQL 区别
- [ ] 五大数据类型及各自适用场景
- [ ] StringRedisTemplate 为什么比 RedisTemplate 好
- [ ] 单机 Session 有什么问题
- [ ] Redis 怎么替代 Session 实现登录
- [ ] token 流程能完整复述
- [ ] 缓存穿透是什么、怎么解
- [ ] 缓存击穿是什么、怎么解
- [ ] 缓存雪崩是什么、怎么解
- [ ] 三大问题能说清区别（不要混）
- [ ] 缓存更新策略有哪些
- [ ] 为什么需要分布式锁
- [ ] Redis SETNX 实现锁的原理
- [ ] 自己写分布式锁有什么坑
- [ ] Redisson 解决了哪些问题

---

## 八、可以跳过的部分

| 章节 | 原因 |
|---|---|
| 安装 / 图形界面 | 实操时再看 |
| Jedis 连接池 | 用 RedisTemplate 就够 |
| List / Set / SortedSet 细节 | 面试不深问，快进过 |
| 优惠券秒杀全章 | 业务向，实习不深问 |
| 达人探店 / Feed 流 | 和核心无关 |
| 主从 / 哨兵 / 分片集群 | 运维级，实习不要求 |
| 原理篇（IO 多路复用 / epoll / 内存淘汰） | 中高级深度题，初级不问 |
| 多级缓存 / OpenResty / Canal | 架构级，实习不要求 |


---

## 🔗 相关笔记

- [[MySQL数据库|MySQL 数据库]] — 关系型 vs NoSQL 对比
- [[../04-Spring生态/SpringBoot框架|Spring Boot 框架]] — SpringDataRedis 集成
- [[../02-Java进阶/Java并发编程|Java 并发编程]] — 分布式锁与并发安全
- [[../05-AI应用开发/SpringAI与RAG实战|Spring AI 与 RAG 实战]] — RAG 缓存层
- [[../07-面试与项目/Java学习情况分析与实习冲刺计划|学习情况分析与冲刺计划]] — Redis 是第一层必吃透
- [[../99-MOC/README|MOC 索引]]
