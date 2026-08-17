---
tags:
  - Spring
  - SpringBoot
  - 配置
created: 2026-08-17
---

# Spring Boot — 基础与配置

> 从 [[Spring Boot 框架]] 拆分而来。

---

## 代码模板

### 1-1 项目结构与 pom.xml

```xml
<!-- Spring Boot 项目 pom.xml 骨架 -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- 父 POM：统一管理版本，不用自己写版本号 -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.x</version>  <!-- 推荐用 3.2.x（Java 17+） -->
        <relativePath/>
    </parent>

    <groupId>io.github.chenyouxin8</groupId>
    <artifactId>chen-ai-agent</artifactId>
    <version>1.0.0</version>
    <name>chen-ai-agent</name>

    <properties>
        <java.version>17</java.version>
        <spring-ai.version>1.0.0-M6</spring-ai.version>
    </properties>

    <dependencies>
        <!-- Web 开发：内嵌 Tomcat + Spring MVC -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- 导入 spring-boot-starter，无版本号（父 POM 管理） -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <!-- Spring AI（DashScope/DeepSeek） -->
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-alibaba-starter</artifactId>
        </dependency>

        <!-- 测试 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- Lombok（简化 POJO，自动生成 getter/setter/constructor） -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>

    <!-- Maven 插件 -->
    <build>
        <plugins>
            <!-- 打包成可执行 JAR（包含所有依赖） -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### 1-2 Spring Boot 启动类

```java
// src/main/java/io/github/chenyouxin8/chenaiagent/
//                   ChenAiAgentApplication.java

package io.github.chenyouxin8.chenaiagent;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ComponentScan;

/**
 * Spring Boot 启动类（主入口）
 *
 * @SpringBootApplication = 三个注解的组合：
 *   1. @Configuration      → 标记为配置类（等价于 <beans>）
 *   2. @EnableAutoConfiguration → 开启自动配置
 *   3. @ComponentScan      → 扫描当前包及子包的 @Component
 */
@SpringBootApplication
// @ComponentScan 扫描范围说明：
// 默认扫描启动类所在包及其子包
// 如果你的类在其他包，需要手动指定：
// @ComponentScan(basePackages = {"io.github.chenyouxin8.chenaiagent", "io.github.chenyouxin8.other"})
public class ChenAiAgentApplication {

    public static void main(String[] args) {
        // 启动 Spring Boot 应用
        SpringApplication.run(ChenAiAgentApplication.class, args);
    }
}

// ===== 多环境启动 =====
public static void main(String[] args) {
    // 指定环境：--spring.profiles.active=local
    SpringApplication.run(ChenAiAgentApplication.class, args);
    // 或者：
    // new SpringApplicationBuilder(ChenAiAgentApplication.class)
    //     .profiles("local")
    //     .run(args);
}

// ===== 关闭Banner（启动时的 Spring LOGO）=====
public static void main(String[] args) {
    SpringApplication app = new SpringApplication(ChenAiAgentApplication.class);
    app.setBannerMode(Banner.Mode.OFF);
    app.run(args);
}
```

### 1-3 配置文件（application.yml / application.properties）

```yaml
# ===== application.yml（YAML 格式，推荐）=====
# 文件名规则：application-{profile}.yml
#   application.yml        ← 默认配置（所有环境共享）
#   application-local.yml  ← 本地环境（不提交 Git）
#   application-dev.yml     ← 开发环境
#   application-prod.yml   ← 生产环境

server:
  port: 8123              # 端口（默认 8080）
  servlet:
    context-path: /api    # 上下文路径（URL 前缀）
  tomcat:
    threads:
      max: 200             # 最大线程数
    connection-timeout: 20000  # 连接超时 ms

spring:
  application:
    name: chen-ai-agent   # 应用名（注册到注册中心时用）

  # ===== 多环境激活 =====  （三种方式）
  profiles:
    active: local          # 方式1：配置文件指定
  # 方式2：命令行：java -jar app.jar --spring.profiles.active=prod
  # 方式3：环境变量：export SPRING_PROFILES_ACTIVE=prod

  # ===== 数据源配置 =====  （Spring Boot 自动配置 DataSource）
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?useUnicode=true&characterEncoding=utf8&useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: ${DB_PASSWORD}           # 环境变量（不写死）
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 10            # 最大连接数
      minimum-idle: 5                  # 最小空闲连接
      connection-timeout: 30000        # 连接超时 ms
      idle-timeout: 600000             # 空闲超时 ms
      max-lifetime: 1800000            # 最大生命周期 ms

  # ===== JSON 配置 =====
  jackson:
    time-zone: Asia/Shanghai           # 时区
    date-format: yyyy-MM-dd HH:mm:ss   # 日期格式
    serialization:
      write-dates-as-timestamps: false  # 日期序列化为字符串
      fail-on-empty-beans: false       # 空对象不抛异常

  # ===== 文件上传 =====
  servlet:
    multipart:
      enabled: true
      max-file-size: 10MB              # 单文件最大
      max-request-size: 100MB          # 请求最大

  # ===== 热部署（DevTools）=====
  devtools:
    restart:
      enabled: true                    # 开启热部署
      additional-paths: src/main/java  # 监控目录

# ===== 自定义配置 =====
ai:
  dashscope:
    api-key: ${AI_DASHSCOPE_API_KEY}   # 从环境变量读取
    base-url: https://dashscope.aliyuncs.com
  model: qwen-plus                      # 默认模型

# ===== 日志配置 =====
logging:
  level:
    root: INFO
    io.github.chenyouxin8.chenaiagent: DEBUG  # 项目日志 DEBUG
  pattern:
    console: '%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n'
  file:
    name: logs/app.log                  # 日志文件
    max-size: 10MB                      # 单文件最大
    max-history: 30                     # 保留天数

# ===== Actuator（健康检查）=====
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env   # 暴露的端点
  endpoint:
    health:
      show-details: when-authorized
```

```properties
# ===== application.properties 等效写法 =====（了解即可，YAML 更简洁）
server.port=8123
spring.application.name=chen-ai-agent
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=${DB_PASSWORD}
logging.level.io.github.chenyouxin8.chenaiagent=DEBUG
```


---

## 🔗 相关笔记

- [[Spring Boot 框架]]
- [[../05-AI应用开发/SpringAI与RAG实战|Spring AI 与 RAG 实战]]
- [[../01-Java基础/反射|反射]]
- [[../06-工具链/01-Maven实战|Maven 实战]]
- [[../03-Database/MySQL数据库|MySQL 数据库]]
