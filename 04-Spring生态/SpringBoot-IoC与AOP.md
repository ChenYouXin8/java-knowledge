---
tags:
  - Spring
  - SpringBoot
  - IoC
  - AOP
created: 2026-08-17
---

# Spring Boot — IoC 与 AOP

> 从 [[Spring Boot 框架]] 拆分而来。

---

## 代码模板

### 1-4 IoC 与依赖注入（你的 chen-ai-agent 核心）

```java
// ===== @Component / @Service / @Repository / @Controller =====
// 位置：src/main/java/io/github/chenyouxin8/chenaiagent/

// @Component：通用组件（Bean）
@Component  // 默认 beanName = "myComponent"（首字母小写）
public class MyComponent {
    public void doSomething() {
        System.out.println("组件执行中...");
    }
}

// @Service：业务层（语义化的 @Component）
@Service
public class LoveAppService {
    // Bean 名称默认是类名首字母小写 = "loveAppService"
    @Autowired
    private LoveApp loveApp;  // 注入 LoveApp
}

// @Repository：数据访问层（语义化的 @Component）
// Spring 会自动将数据访问异常转换为 Spring 的统一异常（DataAccessException）
@Repository
public class UserRepository {
    // DAO 层
}

// @Controller：控制器层（Spring MVC）
// 等价于 @Component + Spring MVC 处理逻辑
@Controller
public class UserController {
    // HTTP 请求处理
}

// @RestController：REST 风格（返回 JSON），等价于 @Controller + @ResponseBody
// chen-ai-agent 的所有 Controller 都是 @RestController
@RestController
@RequestMapping("/api/ai/love")
public class LoveAppController {
    // 直接返回 JSON，不走视图解析
}

// ===== @Autowired 注入方式 =====
// 方式1：属性注入（最简单，但不符合 OOP，不方便测试）
@Autowired
private LoveApp loveApp;

// 方式2：Setter 注入（可以加 @Autowired 注解）
private LoveApp loveApp;
@Autowired
public void setLoveApp(LoveApp loveApp) {
    this.loveApp = loveApp;
}

// 方式3：构造器注入（⭐推荐，IDEA 默认提示，显示不可变依赖）
private final LoveApp loveApp;

@Autowired
public LoveAppService(LoveApp loveApp) {
    this.loveApp = loveApp;
}

// ===== @Bean（Java 配置类方式）=====
// 在 @Configuration 类中用 @Bean 显式注册 Bean
@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate() {
        // RestTemplate 是第三方类，无法加 @Component，只能用 @Bean
        return new RestTemplate();
    }

    @Bean
    public MyService myService(RestTemplate restTemplate) {
        // @Bean 方法参数会被自动注入（参数就是其他 Bean）
        return new MyService(restTemplate);
    }
}

// ===== @Primary / @Qualifier（多个同类型 Bean 时）=====
// 同类型 Bean 有多个时，用 @Primary 指定主 Bean
@Component
@Primary
public class PrimaryServiceImpl implements MyService { }

// 或用 @Qualifier 指定注入哪个
@Service("fastService")
public class FastServiceImpl implements MyService { }

@Service("slowService")
public class SlowServiceImpl implements MyService { }

@RestController
public class DemoController {
    @Autowired
    @Qualifier("fastService")
    private MyService myService;
}

// ===== @Value（注入配置值）=====
@Value("${ai.dashscope.api-key:default-value}")
private String apiKey;

@Value("${server.port:8080}")
private int port;

// 支持 SpEL 表达式
@Value("#{systemProperties['user.dir']}")
private String userDir;

// ===== @ConfigurationProperties（批量注入配置）=====
// application.yml:
// ai:
//   dashscope:
//     api-key: xxx
//     base-url: https://...

@Component
@ConfigurationProperties(prefix = "ai.dashscope")
public class DashScopeProperties {
    private String apiKey;
    private String baseUrl;
    // getter/setter 自动生成（Lombok @Data）
    // Spring Boot 会自动绑定配置值
}

// ===== @Lazy（延迟初始化）=====
// Bean 默认在启动时就创建，加 @Lazy 后第一次使用时才创建
@Lazy
@Service
public class HeavyService {
    public HeavyService() {
        System.out.println("HeavyService 创建了...");
    }
}

// ===== @Scope（作用域）=====
@Component
@Scope("singleton")    // 单例（默认）：整个应用只有一个实例
// @Scope("prototype") // 原型：每次注入都创建新实例
// @Scope("request")   // 请求：每个 HTTP 请求一个实例
// @Scope("session")   // 会话：每个 HTTP 会话一个实例
public class MyComponent { }
```

### 1-5 AOP（面向切面编程）

```java
// ===== AOP 术语理解 =====
// 目标：你希望在每个 Controller 方法执行前后记录日志
// 传统方式：每个方法里写 log.info(...) → 重复
// AOP 方式：写一个切面，自动织入 → 解耦

// ===== 常用注解 =====
// @Aspect         → 标记这是一个切面类
// @Pointcut       → 定义切入点（哪些方法要被拦截）
// @Before         → 前置通知（方法执行前）
// @AfterReturning → 返回通知（方法正常返回后）
// @AfterThrowing  → 异常通知（方法抛异常后）
// @After          → 后置通知（finally，方法返回或异常后都执行）
// @Around         → 环绕通知（最强大，自己决定何时调用目标方法）

// ===== pom.xml 需要引入 AOP starter =====
// <dependency>
//     <groupId>org.springframework.boot</groupId>
//     <artifactId>spring-boot-starter-aop</artifactId>
// </dependency>

// ===== 完整 AOP 示例 =====
// 切入点表达式语法：
// execution(返回值 类名.方法名(参数))
// 常见写法：
//   execution(* io.github.chenyouxin8.chenaiagent..*.*(..))
//    ↑返回任意  ↑包名  ↑当前包及子包  ↑任何类  ↑任何方法  ↑任何参数

package io.github.chenyouxin8.chenaiagent.aspect;

import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

@Slf4j  // Lombok：自动生成 log
@Component
@Aspect  // 标记为切面
public class LoggingAspect {

    // ===== 切入点定义 =====
    // 所有 REST Controller 的所有方法
    @Pointcut("execution(* io.github.chenyouxin8.chenaiagent..*Controller.*(..))")
    public void controllerPointcut() {}

    // 所有 @GetMapping 注解的方法
    @Pointcut("@annotation(org.springframework.web.bind.annotation.GetMapping)")
    public void getMappingPointcut() {}

    // ===== 前置通知 =====
    @Before("controllerPointcut()")
    public void beforeAdvice(JoinPoint joinPoint) {
        String className = joinPoint.getTarget().getClass().getSimpleName();
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        log.info(">>> 进入 {}.{}，参数：{}", className, methodName, args);
    }

    // ===== 返回通知 =====
    @AfterReturning(pointcut = "controllerPointcut()", returning = "result")
    public void afterReturningAdvice(JoinPoint joinPoint, Object result) {
        String methodName = joinPoint.getSignature().getName();
        log.info("<<< {} 执行完成，返回：{}", methodName, result);
    }

    // ===== 异常通知 =====
    @AfterThrowing(pointcut = "controllerPointcut()", throwing = "e")
    public void afterThrowingAdvice(JoinPoint joinPoint, Exception e) {
        String methodName = joinPoint.getSignature().getName();
        log.error("!!! {} 执行异常：{}", methodName, e.getMessage(), e);
    }

    // ===== 环绕通知（最强大）=====
    @Around("controllerPointcut()")
    public Object aroundAdvice(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        String methodName = pjp.getSignature().getName();

        // 执行目标方法
        Object result = pjp.proceed();

        long cost = System.currentTimeMillis() - start;
        log.info("环绕：{}.{} 耗时 {}ms", pjp.getTarget().getClass().getSimpleName(), methodName, cost);

        return result;  // 可以修改返回值
    }
}

// ===== 自定义注解 + AOP =====
@Target(ElementType.METHOD)       // 注解用在方法上
@Retention(RetentionPolicy.RUNTIME)  // 运行时保留
public @interface NoRepeatSubmit {}

// 使用注解
@PostMapping("/submit")
@NoRepeatSubmit
public ApiResponse<Void> submit(@RequestBody OrderForm form) {
    // 业务逻辑
}

// 切面拦截自定义注解
@Aspect
@Component
public class NoRepeatSubmitAspect {
    @Around("@annotation(noRepeatSubmit)")
    public Object around(ProceedingJoinPoint pjp, NoRepeatSubmit noRepeatSubmit) throws Throwable {
        String key = "repeat:submit:" + userId;  // Redis Key
        Boolean success = redisTemplate.opsForValue().setIfAbsent(key, "1", 5, TimeUnit.SECONDS);
        if (!success) {
            throw new BusinessException("请勿重复提交");
        }
        return pjp.proceed();
    }
}
```


---

## 🔗 相关笔记

- [[Spring Boot 框架]]
- [[../05-AI应用开发/SpringAI与RAG实战|Spring AI 与 RAG 实战]]
- [[../01-Java基础/反射|反射]]
- [[../06-工具链/01-Maven实战|Maven 实战]]
- [[../03-Database/MySQL数据库|MySQL 数据库]]
