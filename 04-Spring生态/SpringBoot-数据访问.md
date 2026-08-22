---
tags:
  - SpringBoot
  - Spring
  - MyBatis
  - Redis
  - 数据访问
  - 面试
created: 2026-08-22
---

# Spring Boot 数据访问

> Spring Boot 整合 MyBatis-Plus + Redis + 连接池，一站式数据访问配置。

---

## 一、整合 MyBatis-Plus

### 1.1 依赖

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.5</version>
</dependency>
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
</dependency>
```

### 1.2 数据源配置

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/mydb?useUnicode=true&characterEncoding=utf-8&useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: 123456
    type: com.zaxxer.hikari.HikariDataSource  # 默认连接池
    hikari:
      maximum-pool-size: 10          # 最大连接数
      minimum-idle: 5                # 最小空闲
      idle-timeout: 30000            # 空闲超时(ms)
      connection-timeout: 30000      # 连接超时
      max-lifetime: 1800000          # 连接最大生命周期
```

### 1.3 MyBatis-Plus 配置

```yaml
mybatis-plus:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.example.entity
  configuration:
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
  global-config:
    db-config:
      id-type: assign_id
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
```

### 1.4 启动类注解

```java
@SpringBootApplication
@MapperScan("com.example.mapper")  // 扫描 Mapper 接口
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

---

## 二、整合 Redis

### 2.1 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### 2.2 配置

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: 
      database: 0
      lettuce:
        pool:
          max-active: 8
          max-idle: 4
          min-idle: 0
          max-wait: -1ms
```

### 2.3 序列化配置（推荐）

```java
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        // Key 用 String 序列化
        template.setKeySerializer(new StringRedisSerializer());
        // Value 用 JSON 序列化
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        // Hash 同理
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

> 不配置序列化，默认用 JDK 序列化，存进去的可读性极差、占空间大。

---

## 三、多环境配置

### 3.1 Profile 切换

```yaml
# application.yml
spring:
  profiles:
    active: dev  # 默认开发环境

---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:mysql://localhost:3306/dev_db
    username: root
    password: 123456

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:mysql://prod-db:3306/prod_db
    username: prod_user
    password: ${DB_PASSWORD}  # 环境变量注入
```

### 3.2 启动时指定环境

```bash
java -jar app.jar --spring.profiles.active=prod
```

---

## 四、Druid 连接池（可选）

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.2.20</version>
</dependency>
```

```yaml
spring:
  datasource:
    druid:
      initial-size: 5
      max-active: 20
      min-idle: 5
      stat-view-servlet:
        enabled: true                # 监控页面
        login-username: admin
        login-password: admin
      web-stat-filter:
        enabled: true                # Web 监控
```

> Druid 比 HikariCP 多了 SQL 监控和慢查询日志。HikariCP 性能更好但功能少。

---

## 五、连接池对比

| | HikariCP（默认） | Druid |
|---|---|---|
| 性能 | 最快 | 快 |
| 监控 | 弱 | 强（SQL 监控面板） |
| 配置 | 简单 | 稍复杂 |
| 防SQL注入 | 无 | 有 |
| 企业使用 | Spring Boot 默认 | 阿里系流行 |
| 推荐 | 新项目默认 | 需要监控时 |

---

## 六、面试高频题

**Q1: Spring Boot 整合 MyBatis 步骤？**
> 加依赖 → 配数据源 → 配 mybatis-plus → @MapperScan 扫描接口 → 写 Mapper 继承 BaseMapper → 配分页插件。

**Q2: RedisTemplate 默认序列化有什么问题？**
> 默认 JDK 序列化，存入 Redis 的数据可读性差、体积大、跨语言不兼容。应配置 String + JSON 序列化。

**Q3: HikariCP 和 Druid 怎么选？**
> HikariCP 性能最优、Spring Boot 默认、配置简单。Druid 有 SQL 监控、慢日志、防注入。新项目用 HikariCP，需要监控用 Druid。

**Q4: Spring Boot 多环境配置怎么做？**
> 用 Profile：application-dev.yml / application-prod.yml，通过 spring.profiles.active 切换。生产用环境变量注入密码。

**Q5: @MapperScan 和 @Mapper 的区别？**
> @Mapper 逐个标注接口。@MapperScan 在启动类批量扫描整个包。推荐 @MapperScan。

---

## 🔗 相关笔记

- [[SpringBoot-基础与配置|Spring Boot 基础与配置]] — 项目结构 / 启动类
- [[../08-数据访问层/MyBatis核心概念|MyBatis 核心概念]] — ORM 原理
- [[../08-数据访问层/MyBatis-Plus实战|MyBatis-Plus 实战]] — 通用 CRUD / 分页
- [[../03-Database/Redis最小复习笔记|Redis 最小复习笔记]] — 缓存三大问题
- [[../03-Database/MySQL数据库|MySQL 数据库]] — 底层数据库