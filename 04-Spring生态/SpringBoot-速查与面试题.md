---
tags:
  - Spring
  - SpringBoot
  - 面试
  - 速查
created: 2026-08-17
---

# SpringBoot框架 — 速查与面试题

> 从 [[SpringBoot框架]] 拆分而来，包含对比表格、图解、速查清单、面试题和踩坑记录。

---

## 2️⃣ 对比表格

### 2-1 Spring Boot 自动配置原理

| 步骤 | 发生了什么 | 源码位置 |
|------|-----------|---------|
| `@SpringBootApplication` | 触发自动配置 | `spring-boot-autoconfigure` JAR |
| `@EnableAutoConfiguration` | 加载 `META-INF/spring.factories` | `AutoConfigurationImportSelector` |
| `spring.factories` | 列出所有自动配置类 | 第三方 JAR 中 |
| `@Conditional*` 注解 | 按条件选择性生效 | 配置类上的条件注解 |
| `application.yml` | 用户配置覆盖默认值 | 用户自定义 |

### 2-2 @Autowired / @Resource / @Inject 对比

| 维度 | @Autowired | @Resource | @Inject |
|------|-----------|-----------|---------|
| 来源 | Spring 自有 | JavaEE（JSR-250） | JavaCDI（JSR-330） |
| 注入方式 | 属性/Setter/构造器 | 属性/Setter | 属性/Setter |
| 默认按类型 | ✅ 按类型 | ❌ 按名称 | ✅ 按类型 |
| 指定名称 | `@Qualifier("name")` | `@Resource(name="name")` | `@Named("name")` |
| required 属性 | `@Autowired(required=false)` | ❌ 无 | `@Inject(optional=true)` |
| 推荐 | ✅ Spring 项目首选 | 老项目/Jakarta EE | 与框架无关时 |

### 2-3 Spring MVC 工作流程

```
用户请求 → Filter → DispatcherServlet → HandlerMapping
                                           ↓
                                     HandlerAdapter
                                           ↓
                                   Interceptor.preHandle()
                                           ↓
                                    Controller（执行业务）
                                           ↓
                                  Interceptor.postHandle()
                                           ↓
                                   ViewResolver（视图解析）
                                           ↓
                                       View（渲染）
                                           ↓
                                  Interceptor.afterCompletion()
                                           ↓
                                    响应用户
```

### 2-4 Spring Bean 作用域

| 作用域 | 说明 | 线程安全 | 典型场景 |
|--------|------|---------|---------|
| `singleton` | 整个应用只有一个（**默认**） | 否（需自己保证） | Service/Dao |
| `prototype` | 每次注入创建新实例 | 是 | 有状态 Bean |
| `request` | 每个 HTTP 请求一个实例 | 否 | Web 请求数据 |
| `session` | 每个 HTTP 会话一个实例 | 否 | 用户会话数据 |
| `application` | ServletContext 生命周期内唯一 | 否 | 全局缓存 |
| `websocket` | WebSocket 生命周期内唯一 | 否 | WebSocket 数据 |

### 2-5 Filter / Interceptor / AOP 对比

| 维度 | Filter | Interceptor | AOP（@Aspect） |
|------|--------|------------|----------------|
| 作用范围 | Servlet（所有请求） | Spring MVC | Spring Bean |
| 依赖 | Java EE | Spring MVC | Spring AOP |
| 执行时机 | 最先（Servlet 层面） | Controller 前后 | 方法级别 |
| 能拿到 | ServletRequest | HandlerMethod | JoinPoint |
| 能做的事 | 字符编码/鉴权/跨域 | 登录检查/权限/日志 | 业务切面（日志/事务/性能统计） |
| 能否获取方法参数 | ❌ | ✅ | ✅ |

### 2-6 Spring 事务传播行为

| 传播行为 | 说明 | 常见场景 |
|---------|------|---------|
| REQUIRED | 有则加入，无则创建 | **默认**，嵌套业务方法 |
| REQUIRES_NEW | 总是创建新事务 | 日志记录（不影响主事务） |
| NESTED | 嵌套事务（MySQL SAVEPOINT） | 子事务失败只回滚子部分 |
| SUPPORTS | 有则加入，无则无事务 | 只读查询 |
| MANDATORY | 必须有事务，否则抛异常 | 强制在事务中执行 |
| NOT_SUPPORTED | 无事务执行，挂起当前 | 不需要事务的方法 |
| NEVER | 不能有事务，否则抛异常 | 非事务方法 |

---

## 3️⃣ 图解结构

### 3-1 Spring Boot 启动流程

```
SpringApplication.run() 执行流程：

第1步：创建 SpringApplication 对象
        ↓
    创建 BootstrapRegistryInitializer（引导注册）
    创建 ApplicationContextInitializer
    创建 ApplicationListener（从 spring.factories 加载）

第2步：执行 run()
        ↓
    1. 获取并启动监听器 SpringApplicationRunListeners
    2. 创建并配置 Environment（命令行参数 + 环境变量）
    3. 打印 Banner（Spring LOGO）
    4. 创建 ApplicationContext（AnnotationConfigServletWebServerApplicationContext）
           ↓
       5. 准备 Context
          - 将 Environment 绑定到 Context
          - 执行 initializers
          - 加载主配置类（@SpringBootApplication）
              ↓
          6. 刷新容器（Context.refresh()）
             ├─ 触发 BeanFactoryPostProcessor
             │   └─ ConfigurationClassPostProcessor
             │       └─ 扫描 @Component / @Bean / @Import 等
             ├─ 注册 BeanPostProcessor（AOP/事务的代理在此注入）
             ├─ 初始化 MessageSource（国际化）
             ├─ 初始化事件广播器
             ├─ onRefresh() → 启动内嵌 Tomcat/Jetty
             ├─ registerListeners() → 注册监听器
             └─ finishBeanFactoryInitialization()
                 └─ 所有单例 Bean 在此创建（懒加载除外）
                        ↓
                   7. 执行 Runner
                      - ApplicationRunner
                      - CommandLineRunner
                      （用于启动后执行初始化逻辑）
                        ↓
    8. 发布 ApplicationStartedEvent
```

### 3-2 IoC 容器分层

```
Spring IoC 容器（BeanFactory 体系）：

DefaultListableBeanFactory（基础实现）
        ↑
  AbstractApplicationContext.refresh()
        ↑
┌───────────────┬───────────────┬──────────────────┐
│ AnnotationConfigApplicationContext │ FileSystemXmlApplicationContext │ ClassPathXmlApplicationContext
│（Java 配置类扫描，Spring Boot 用这个）           │（XML 配置，Spring 传统用法）       │（classpath 下找 XML）
└───────────────┴───────────────┴──────────────────┘

Bean 创建顺序：
1. 扫描 @ComponentScan 包下的所有 @Component
2. 解析 @Bean 方法（@Configuration 类）
3. 处理 @Import（导入的类）
4. 执行 BeanFactoryPostProcessor（可修改 Bean 定义）
5. 实例化所有非懒加载的单例 Bean（按依赖顺序）
6. 处理 @PostConstruct 初始化方法
7. 如果是代理对象，替换原对象（ASM 生成代理）
```

### 3-3 AOP 代理原理

```
Spring AOP：基于代理模式

目标对象（Target）：业务方法所在的对象
代理对象（Proxy）：Spring 生成的包装对象

┌─────────────────────────────────────────────────────────────┐
│  客户端（调用方）                                           │
└────────────────────────┬────────────────────────────────────┘
                         │ proxy.doSomething()
                         ↓
              ┌──────────────────────┐
              │    代理对象（Proxy）   │
              │                       │
              │  1. 前置通知 @Before  │
              │  2. 调用目标方法      │
              │     ↓                 │
              │  ┌─────────────────┐  │
              │  │  目标对象        │  │
              │  │  (Target)       │  │
              │  │                 │  │
              │  │  doSomething()  │  │
              │  └─────────────────┘  │
              │  3. 返回通知 @AfterReturning │
              │  4. 异常通知 @AfterThrowing  │
              │  5. 后置通知 @After   │
              └──────────────────────┘

JDK 动态代理：目标类实现了接口 → Proxy.newProxyInstance()
               只生成接口的代理（Target 必须有接口）

CGLIB 代理：目标类没有接口 → Enhancer.create()
             生成子类代理（final 方法无法代理）

Spring Boot 2.x 默认：若目标类有接口用 JDK 代理，否则用 CGLIB
Spring Boot 3.x 默认：CGLIB（性能更好，限制更少）
```

---

## 4️⃣ 速查清单

### 4-1 Spring Boot 常用注解速查

```java
// ===== Spring Boot 核心 =====
@SpringBootApplication          // 启动类 = @Configuration + @EnableAutoConfiguration + @ComponentScan
@Configuration                  // 配置类（等价于 XML 配置）
@Bean                           // 在配置类中注册 Bean
@ComponentScan                  // 扫描组件（默认扫启动类所在包）
@EnableAutoConfiguration        // 开启自动配置

// ===== Bean 注入 =====
@Autowired                      // 按类型注入
@Qualifier("name")             // 按名称注入（解决多个同类型 Bean）
@Resource(name="name")         // 按名称注入（Java EE 标准）
@Primary                        // 同类型 Bean 中标记为主 Bean
@Value("${key:default}")       // 注入配置值
@ConfigurationProperties        // 批量注入配置前缀

// ===== 分层注解 =====
@Component                      // 通用组件
@Service                        // 业务层
@Repository                     // 数据访问层
@Controller / @RestController   // 控制层
@Configuration                  // 配置类

// ===== 请求处理 =====
@RequestMapping               // 通用路由（可加在类和方法上）
@GetMapping / @PostMapping   // GET/POST 路由
@PutMapping / @DeleteMapping  // PUT/DELETE 路由
@PatchMapping                 // PATCH（部分更新）
@PathVariable                 // 路径变量（URL 中 {id}）
@RequestParam                 // 请求参数（?key=value）
@RequestBody                  // 请求体（JSON → Java）
@RequestHeader                // 请求头
@CookieValue                  // Cookie
@ResponseBody                 // 返回 JSON（@RestController 已包含）
@CrossOrigin                  // 跨域

// ===== 校验 =====
@Valid / @Validated          // 触发校验
@NotNull / @NotBlank         // 非空校验
@Size / @Min / @Max          // 范围校验
@Email / @Pattern            // 格式校验
@DateTimeFormat              // 日期格式

// ===== AOP =====
@Aspect                       // 切面类
@Pointcut                     // 切入点定义
@Before / @AfterReturning    // 通知
@AfterThrowing / @After       // 异常/后置通知
@Around                       // 环绕通知

// ===== 其他 =====
@Lazy                         // 延迟初始化
@Conditional                  // 条件注册 Bean
@Profile("dev")              // 环境特定 Bean
@Async                        // 异步执行
@Scheduled(cron = "0 0 * * * ?")  // 定时任务
```

### 4-2 Spring Boot 常用配置速查

```yaml
# application.yml 常用配置
server:
  port: 8123
  servlet:
    context-path: /api
  tomcat:
    threads:
      max: 200
      min-spare: 10
    connection-timeout: 20000

spring:
  profiles:
    active: local          # 激活环境
  application:
    name: app-name
  datasource:
    url: jdbc:mysql://...
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: root
    password: ${ENV_VAR}
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 100MB

logging:
  level:
    root: INFO
    com.example: DEBUG
  file:
    name: logs/app.log
```

### 4-3 REST API 设计规范

| 方法 | 语义 | 示例 |
|------|------|------|
| GET | 查询 | `GET /api/users` |
| GET | 单个查询 | `GET /api/users/{id}` |
| POST | 新增 | `POST /api/users` |
| PUT | 全量更新 | `PUT /api/users/{id}` |
| PATCH | 部分更新 | `PATCH /api/users/{id}` |
| DELETE | 删除 | `DELETE /api/users/{id}` |

### 4-4 常见 HTTP 状态码速查

| 状态码 | 含义 | 常见场景 |
|--------|------|---------|
| 200 | 成功 | GET/PUT/PATCH 成功 |
| 201 | 创建成功 | POST 新增成功 |
| 204 | 无内容 | DELETE 成功（不返回 body） |
| 400 | 请求参数错误 | 参数校验失败 |
| 401 | 未授权 | 未登录/Token 过期 |
| 403 | 无权限 | 无访问权限 |
| 404 | 资源不存在 | ID 不存在 |
| 409 | 冲突 | 重复提交/数据冲突 |
| 500 | 服务器内部错误 | 未捕获异常 |
| 503 | 服务不可用 | 维护/限流 |

---

## 5️⃣ 场景选择器

### 5-1 该用 @Bean 还是 @Component？

```
该用哪个？
         │
    ┌────┴──────────────────────────┐
    │                               │
  Bean 来自第三方类？             Bean 来自自己写的类？
  （RestTemplate / ObjectMapper）   （Service / Controller）
    │                               │
    ↓                               ↓
    @Configuration + @Bean          @Component / @Service
    + @Bean 方法参数自动注入        + @Autowired
```

### 5-2 @Transactional 失效了？

```
事务没回滚？
         │
    ┌────┴──────────────────────────────┐
    │                                       │
  方法是 public？                     方法内部自调用？
    │（private 方法代理不了）              │（不走代理对象）
    ↓                                       ↓
    ↓ 是                                   ↓ 不是
    ↓                                   把调用的方法拆到另一个 Bean
    ↓
  异常被 catch 吞了？
    │（异常没抛到外层，代理感知不到）
    ↓
  抛的是否是 RuntimeException？
    │（默认只回滚运行时异常）
    ↓
  是 → @Transactional(rollbackFor = Exception.class)

  ❌ 否 → 检查是否 catch 后重新抛了
  ❌ 否 → 加上 rollbackFor = Exception.class
```

### 5-3 拦截器 vs Filter 选哪个？

```
需要拦截什么？
         │
    ┌────┴──────────────────────────────┐
    │                                       │
  所有请求（包括静态资源）？              只有 Controller？
    │（CSS/JS/图片/HTML）                  │
    ↓                                       ↓
    Filter                                Interceptor
    （字符编码/鉴权/跨域/CORS）             （业务日志/权限/登录检查）
    用 @WebFilter + @ServletComponentScan   用 @Component + WebMvcConfigurer
```

---

## ❓ 常见面试题

**Q1: Spring Boot 的自动配置原理是什么？**
> Spring Boot 在 `spring-boot-autoconfigure` JAR 的 `META-INF/spring.factories` 中定义了所有自动配置类（AutoConfiguration）。启动时 `@EnableAutoConfiguration` 通过 `AutoConfigurationImportSelector` 读取这些类，每个配置类上有 `@Conditional*` 条件注解，按条件选择性生效。用户配置（application.yml）会覆盖自动配置的默认值。

**Q2: IoC 和 DI 是什么关系？**
> IoC（Inversion of Control，控制反转）是一种设计思想，把对象的创建和依赖关系的管理从程序代码移到框架/容器。DI（Dependency Injection，依赖注入）是 IoC 的具体实现方式，通过 `@Autowired` 等注解让 Spring 容器把依赖对象"注入"进来。Spring 通过反射+字节码增强实现 DI。

**Q3: Spring Bean 的生命周期？**
> 实例化（Instantiation）→ 属性填充（Populate）→ 初始化（Initialization）→ 销毁（Destruction）。初始化阶段依次执行：`BeanNameAware.setBeanName()` → `BeanFactoryAware.setBeanFactory()` → `ApplicationContextAware.setApplicationContext()` → `@PostConstruct` → `InitializingBean.afterPropertiesSet()` → `@Bean(initMethod)` → `BeanPostProcessor.postProcessBeforeInitialization()` 之后 → `BeanPostProcessor.postProcessAfterInitialization()`。

**Q4: Spring 是如何解决循环依赖的？**
> Spring 通过**三级缓存**解决普通 Bean 的循环依赖：singletonObjects（一级，完全初始化好）→ earlySingletonObjects（二级，提前暴露，未填充属性）→ singletonFactories（三级，ObjectFactory，提前暴露工厂）。构造器注入的循环依赖无法解决（必须改造构造器或用 `@Lazy` 懒加载）。Spring Boot 2.6+ 默认禁止循环依赖。

**Q5: Spring AOP 和 AspectJ 的区别？**
> Spring AOP 是**运行时代理**（JDK 动态代理或 CGLIB），只支持方法级别的拦截，不能拦截构造器/属性。AspectJ 是**编译时/加载时织入**（修改字节码），功能更强但配置复杂。Spring AOP 用于声明式事务/日志/安全，AspectJ 用于自定义切面/自定义注解。

**Q6: Spring Boot 如何自定义 Starter？**
> 新建 `xxx-spring-boot-starter` 模块，`pom.xml` 引入 `spring-boot-autoconfigure`；在 `META-INF/spring.factories` 注册自动配置类；自动配置类用 `@Configuration + @Conditional*` 注解实现开关逻辑；用户只需要引入 Starter 依赖即可开箱即用。

**Q7: Spring Boot 如何实现热部署（DevTools）？**
> 引入 `spring-boot-devtools` 依赖，修改代码后保存时自动重启（类加载器级别的重启，比整应用重启快）；或者 JRebel（商业软件，类级别热替换，无需重启）。Spring Boot 3.x 也支持 Spring Native（AOT 编译）提升启动速度。

## 📊 学习状态

- [x] Spring Boot 自动配置原理
- [x] Spring IoC/DI（@Component / @Bean / @Autowired）
- [x] Spring Bean 生命周期与作用域
- [x] Spring AOP（@Aspect / 通知类型 / 切入点表达式）
- [x] Spring MVC（请求流程 / 参数绑定 / REST API）
- [x] Filter / Interceptor / AOP 区别
- [x] 全局异常处理（@RestControllerAdvice）
- [x] Spring 事务（@Transactional / 传播行为 / 失效场景）
- [x] 配置文件（application.yml 多环境）
- [ ] Spring Boot 整合 MyBatis
- [ ] Spring Boot 整合 Redis
- [ ] Spring Boot 监控（Actuator）

## 🐛 踩坑记录

- **`@Bean` 方法参数自动注入**：Spring 会自动注入参数对应的 Bean，不需要 `@Autowired`
- **`@Transactional` 在 private 方法上无效**：Spring AOP 基于代理，private 方法不在代理范围内
- **类内部方法自调用不走代理**：`this.xxx()` 绕过了 Spring 代理对象，事务/日志/缓存都失效
- **BeanName 首字母大小写**：`@Service` 默认 BeanName 是类名首字母小写，不是类全名
- **@ComponentScan 默认只扫启动类所在包**：其他包的 `@Component` 扫不到，除非手动指定
- **`@Value` 注入 List/Map**：`@Value("#{'${list}'.split(',')}")` 或用 `@ConfigurationProperties`
- **application.yml 中文乱码**：确保文件是 UTF-8 编码
- **devtools 热部署不生效**：确认 IDEA 开启了 "Build project automatically"
- **@Async 默认单线程**：线程池默认单线程，高并发会阻塞，改用 `TaskExecutor` 配置线程池
- **循环依赖**：Spring Boot 2.6+ 默认禁用循环依赖，`application.yml` 加 `spring.main.allow-circular-references=true` 临时解决

---

## 🔗 相关笔记

- [[SpringBoot框架]]
