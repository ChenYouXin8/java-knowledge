---
tags:
  - SpringBoot
  - Spring
  - 定时任务
  - 异步
  - 面试
created: 2026-08-22
---

# Spring Boot 定时任务与异步

> 定时任务和异步处理是企业应用最常见的两个功能：定时同步数据、异步发邮件/通知。

---

## 一、定时任务 @Scheduled

### 1.1 开启定时任务

```java
@SpringBootApplication
@EnableScheduling  // 开启定时任务
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

### 1.2 常用表达式

```java
@Component
public class TaskJob {

    // 固定间隔：上一次执行结束后等 5 秒再执行
    @Scheduled(fixedDelay = 5000)
    public void fixedDelayTask() { }

    // 固定频率：每 5 秒执行一次（不等上次完成）
    @Scheduled(fixedRate = 5000)
    public void fixedRateTask() { }

    // 首次延迟 10 秒，之后每 5 秒
    @Scheduled(initialDelay = 10000, fixedRate = 5000)
    public void initialDelayTask() { }

    // Cron 表达式：每天凌晨 2 点执行
    @Scheduled(cron = "0 0 2 * * ?")
    public void cronTask() { }

    // 每 10 秒执行
    @Scheduled(cron = "*/10 * * * * ?")
    public void everyTenSeconds() { }
}
```

### 1.3 Cron 表达式速查

```
秒 分 时 日 月 周
 0  0  2  *  *  ?    → 每天 2:00
 0  */5 * * * ?     → 每 5 分钟
 0 0 9 ? * MON-FRI  → 工作日 9:00
 0 0 0 1 * ?        → 每月 1 号 0:00
```

| 位置 | 值 | 通配符 |
|---|---|---|
| 秒 | 0-59 | `, - * /` |
| 分 | 0-59 | `, - * /` |
| 时 | 0-23 | `, - * /` |
| 日 | 1-31 | `, - * ? L` |
| 月 | 1-12 / JAN-DEC | `, - * /` |
| 周 | 1-7 / SUN-SAT | `, - * ? L #` |

### 1.4 单线程问题

> **默认单线程**：多个定时任务串行执行，一个卡住全部阻塞。

**解决：配置线程池**
```java
@Configuration
public class SchedulingConfig {
    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(10);
        scheduler.setThreadNamePrefix("scheduled-");
        return scheduler;
    }
}
```

### 1.5 fixedDelay vs fixedRate

| | fixedDelay | fixedRate |
|---|---|---|
| 计时起点 | 上一次**结束** | 上一次**开始** |
| 执行方式 | 串行 | 理论并行（实际受线程池限制） |
| 适用 | 任务耗时不固定 | 固定频率 |

---

## 二、异步任务 @Async

### 2.1 开启异步

```java
@SpringBootApplication
@EnableAsync  // 开启异步
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

### 2.2 使用

```java
@Service
public class EmailService {

    @Async  // 异步执行，不阻塞调用方
    public void sendEmail(String to, String content) {
        Thread.sleep(3000);  // 模拟耗时
        System.out.println("邮件已发送: " + to);
    }
}

@RestController
public class UserController {
    @Autowired
    private EmailService emailService;

    @PostMapping("/register")
    public String register() {
        saveUser();                    // 同步存库
        emailService.sendEmail(...);   // 异步发邮件，不阻塞
        return "注册成功";             // 立即返回
    }
}
```

> @Async 必须在 **public 方法** 上，且**不能在同一个类内调用**（Spring AOP 限制）。

### 2.3 自定义线程池

```java
@Configuration
public class AsyncConfig {
    @Bean("asyncExecutor")
    public Executor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(
            new ThreadPoolExecutor.CallerRunsPolicy()  // 拒绝策略：调用方执行
        );
        executor.initialize();
        return executor;
    }
}
```

```java
@Async("asyncExecutor")  // 指定线程池
public void sendEmail() { }
```

### 2.4 异步返回值

```java
@Async
public CompletableFuture<String> doSomething() {
    Thread.sleep(2000);
    return CompletableFuture.completedFuture("完成");
}

// 调用方
CompletableFuture<String> future = service.doSomething();
future.thenAccept(result -> System.out.println(result));
```

---

## 三、定时任务进阶

### 3.1 动态定时任务

```java
@RestController
@RequestMapping("/task")
public class TaskController {

    private final ScheduledTaskRegistrar registrar;
    private ScheduledTask<?> currentTask;

    @PostMapping("/start")
    public String start(@RequestParam String cron) {
        if (currentTask != null) currentTask.cancel();
        Runnable task = () -> System.out.println("执行: " + LocalDateTime.now());
        currentTask = registrar.schedule(task, new CronTrigger(cron));
        return "已启动: " + cron;
    }

    @PostMapping("/stop")
    public String stop() {
        if (currentTask != null) {
            currentTask.cancel();
            return "已停止";
        }
        return "无运行中的任务";
    }
}
```

### 3.2 分布式定时任务

> 单机 @Scheduled 在集群环境下会**重复执行**。

**方案对比：**

| 方案 | 原理 | 适用 |
|---|---|---|
| Redis 分布式锁 | 抢到锁才执行 | 简单场景 |
| XXL-JOB | 中心化调度平台 | 企业标配 |
| Quartz 集群 | 数据库锁 | 传统项目 |
| Spring Cloud Task | 云原生 | 微服务 |

```java
// Redis 锁方案
@Scheduled(cron = "0 0 2 * * ?")
public void dailyTask() {
    Boolean locked = redisTemplate.opsForValue()
        .setIfAbsent("task:daily", "1", 1, TimeUnit.HOURS);
    if (Boolean.TRUE.equals(locked)) {
        try {
            // 执行任务
        } finally {
            redisTemplate.delete("task:daily");
        }
    }
}
```

---

## 四、面试高频题

**Q1: @Scheduled 默认是几个线程？**
> 默认单线程。多个定时任务串行执行，一个卡住全部阻塞。需配置 TaskScheduler 线程池。

**Q2: fixedDelay 和 fixedRate 的区别？**
> fixedDelay = 上次结束后等待 N 秒再执行。fixedRate = 每隔 N 秒执行一次（不管上次是否完成）。

**Q3: @Async 有什么坑？**
> 1. 必须 public 方法；2. 不能同类内部调用（AOP 代理限制）；3. 默认用 SimpleAsyncTaskExecutor（每次新建线程），需自定义线程池。

**Q4: 集群环境下定时任务重复执行怎么办？**
> 用 Redis 分布式锁（setIfAbsent 抢锁）或 XXL-JOB 调度平台。抢到锁的节点执行，其他跳过。

**Q5: @Async 用了什么线程池？**
> 默认 SimpleAsyncTaskExecutor（每次 new 一个线程）。生产环境必须自定义 ThreadPoolTaskExecutor。

---

## 🔗 相关笔记

- [[SpringBoot-基础与配置|Spring Boot 基础与配置]] — 启动注解
- [[SpringBoot-Web开发|Spring Boot Web 开发]] — Controller 层
- [[../02-Java进阶/Java并发编程|Java 并发编程]] — 线程池原理
- [[../03-Database/Redis最小复习笔记|Redis 最小复习笔记]] — 分布式锁方案
- [[../09-消息队列/RabbitMQ实战|RabbitMQ 实战]] — 延迟队列替代定时任务