# Spring Boot 框架

> Spring Boot 是 Java 后端开发的标准脚手架，你 chen-ai-agent 项目就是用它搭建的。本笔记覆盖：Spring Boot 核心、自动配置、IoC/AOP、Spring MVC、参数绑定、拦截器/过滤器、异常处理、事务管理、热部署。

---

## 1️⃣ 代码模板

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

### 1-6 Spring MVC 请求处理

```java
// ===== @RequestMapping（基础路由）=====
// 类上：统一前缀
@RestController
@RequestMapping("/api/ai/love")  // 类的所有方法都加此前缀
public class LoveAppController {

    // ===== 参数绑定 =====

    // 路径变量（RESTful 风格）
    @GetMapping("/detail/{id}")
    public ApiResponse<User> getById(@PathVariable Long id) {
        // URL: /api/ai/love/detail/123
        return ApiResponse.success(userService.getById(id));
    }

    // 多个路径变量
    @GetMapping("/user/{userId}/order/{orderId}")
    public ApiResponse<Order> getOrder(
        @PathVariable Long userId,
        @PathVariable Long orderId
    ) {
        return ApiResponse.success(orderService.get(userId, orderId));
    }

    // 请求参数（?key=value）
    @GetMapping("/list")
    public ApiResponse<List<User>> list(
        @RequestParam(defaultValue = "1") int page,    // 默认值
        @RequestParam(required = false) Integer size,    // 可选参数
        @RequestParam String keyword                       // 必填
    ) {
        // URL: /api/ai/love/list?page=1&size=20&keyword=java
    }

    // 请求头
    @GetMapping("/header")
    public ApiResponse<Void> getHeader(
        @RequestHeader("Authorization") String token,
        @RequestHeader(value = "User-Agent", required = false) String ua
    ) {
        return ApiResponse.success();
    }

    // Cookie
    @GetMapping("/cookie")
    public ApiResponse<Void> getCookie(
        @CookieValue(value = "SESSION_ID", required = false) String sessionId
    ) {
        return ApiResponse.success();
    }

    // ===== 请求体（JSON → Java 对象）=====
    // Spring Boot 自动用 Jackson 把 JSON 映射为 Java 对象
    @PostMapping("/chat")
    public ApiResponse<String> chat(@RequestBody ChatRequest request) {
        // ChatRequest 必须有 getter/setter（用 Lombok @Data）
        String answer = loveApp.doChat(request.getMessage(), request.getChatId());
        return ApiResponse.success(answer);
    }

    // ===== 文件上传 =====
    @PostMapping("/upload")
    public ApiResponse<String> upload(
        @RequestParam("file") MultipartFile file,  // 单文件
        @RequestParam("username") String username
    ) {
        String path = fileService.save(file);
        return ApiResponse.success(path);
    }

    @PostMapping("/upload-multiple")
    public ApiResponse<List<String>> uploadMultiple(
        @RequestParam("files") MultipartFile[] files  // 多文件
    ) {
        // 处理多个文件
    }

    // ===== 原生 Servlet 对象（特殊场景）=====
    @GetMapping("/servlet")
    public ApiResponse<Void> servlet(
        HttpServletRequest request,
        HttpServletResponse response,
        HttpSession session
    ) {
        String name = request.getParameter("name");
        session.setAttribute("key", "value");
        return ApiResponse.success();
    }

    // ===== JSON → Map（动态字段）=====
    @PostMapping("/dynamic")
    public ApiResponse<Void> dynamic(@RequestBody Map<String, Object> data) {
        Object value = data.get("key");
        return ApiResponse.success();
    }

    // ===== JSON → List =====
    @PostMapping("/batch")
    public ApiResponse<Void> batch(@RequestBody List<User> users) {
        return ApiResponse.success();
    }

    // ===== PUT / DELETE 请求 =====（表单提交需要加 @MethodFilter 或用 Postman）
    @PutMapping("/update")
    public ApiResponse<Void> update(@RequestBody User user) {
        userService.update(user);
        return ApiResponse.success();
    }

    @DeleteMapping("/{id}")
    public ApiResponse<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return ApiResponse.success();
    }
}

// ===== 请求体 DTO（Data Transfer Object）=====
@Data   // Lombok：自动生成 getter/setter/toString
public class ChatRequest {
    private String message;
    private String chatId;
    private String model;  // 可选字段
}

// ===== 统一响应体（你项目里的 ApiResponse）=====
@Data
public class ApiResponse<T> {
    private int code;
    private String message;
    private T data;
    private long timestamp;

    public static <T> ApiResponse<T> success(T data) {
        ApiResponse<T> resp = new ApiResponse<>();
        resp.setCode(200);
        resp.setMessage("success");
        resp.setData(data);
        resp.setTimestamp(System.currentTimeMillis());
        return resp;
    }

    public static <T> ApiResponse<T> error(int code, String message) {
        ApiResponse<T> resp = new ApiResponse<>();
        resp.setCode(code);
        resp.setMessage(message);
        resp.setTimestamp(System.currentTimeMillis());
        return resp;
    }
}
```

### 1-7 拦截器（Interceptor）与过滤器（Filter）

```java
// ===== Filter（Servlet 级别，拦截所有请求，包括静态资源）=====
// 位置：任意包，用 @WebFilter 注解
@WebFilter(urlPatterns = "/api/*", filterName = "myFilter")
public class MyFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) {
        System.out.println("MyFilter 初始化");
    }

    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) resp;

        // 前置逻辑
        String token = request.getHeader("Authorization");
        if (token == null) {
            response.setStatus(401);
            response.getWriter().write("{\"code\":401,\"message\":\"未登录\"}");
            return;
        }

        // 放行
        chain.doFilter(request, response);

        // 后置逻辑（响应之后）
        response.setHeader("X-Response-Time", "OK");
    }

    @Override
    public void destroy() {
        System.out.println("MyFilter 销毁");
    }
}

// 在启动类加注解注册（Filter 不会自动扫描）
@ServletComponentScan  // 扫描 @WebFilter
@SpringBootApplication
public class ChenAiAgentApplication { }

// ===== 注册 Filter 的另一种方式（@Configuration）=====
@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<MyFilter> myFilter() {
        FilterRegistrationBean<MyFilter> reg = new FilterRegistrationBean<>();
        reg.setFilter(new MyFilter());
        reg.addUrlPatterns("/api/*");
        reg.setOrder(1);  // 数字越小优先级越高
        return reg;
    }
}

// ===== Interceptor（Spring MVC 级别，不拦截静态资源）=====
// 位置：任意包，用 @Component 和 @ComponentScan 扫描

@Component
public class AuthInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        // 前置处理（Controller 方法执行前）
        String token = request.getHeader("Authorization");
        if (token == null || !token.startsWith("Bearer ")) {
            response.setStatus(401);
            response.getWriter().write("{\"code\":401,\"message\":\"未授权\"}");
            return false;  // 返回 false = 拦截，不再执行后续
        }
        return true;  // 返回 true = 放行
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                           Object handler, ModelAndView modelAndView) throws Exception {
        // 后置处理（Controller 方法执行后，视图渲染前）
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) throws Exception {
        // 渲染完成后（清理资源用）
        if (ex != null) {
            log.error("请求异常", ex);
        }
    }
}

// ===== 注册 Interceptor（@Configuration）=====
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Autowired
    private AuthInterceptor authInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
            .addPathPatterns("/api/**")          // 拦截路径
            .excludePathPatterns("/api/login/**") // 排除路径（登录接口不拦截）
            .excludePathPatterns("/api/public/**")
            .order(1);
    }
}

// ===== Filter vs Interceptor 对比 =====
// Filter：Servlet 级别，所有请求都过（Java EE 规范，不依赖 Spring）
// Interceptor：Spring MVC 级别，只过 DispatcherServlet（不拦截静态资源）
// 推荐：鉴权用 Filter，业务日志用 Interceptor
```

### 1-8 全局异常处理

```java
// ===== @RestControllerAdvice（统一异常处理）=====
// 你项目里已经有了 GlobalExceptionHandler，这里是完整版
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 处理业务异常
    @ExceptionHandler(BusinessException.class)
    public ApiResponse<Void> handleBusiness(BusinessException e) {
        return ApiResponse.error(e.getCode(), e.getMessage());
    }

    // 处理参数校验异常（@Valid 失败）
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ApiResponse<Void> handleValid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors()
            .stream()
            .map(FieldError::getDefaultMessage)
            .collect(Collectors.joining("; "));
        return ApiResponse.error(400, "参数错误：" + msg);
    }

    // 处理空指针异常
    @ExceptionHandler(NullPointerException.class)
    public ApiResponse<Void> handleNPE(NullPointerException e) {
        log.error("空指针异常", e);
        return ApiResponse.error(500, "系统内部错误");
    }

    // 处理所有未捕获异常（兜底）
    @ExceptionHandler(Exception.class)
    public ApiResponse<Void> handle(Exception e) {
        log.error("未知异常", e);
        return ApiResponse.error(500, "服务器错误：" + e.getMessage());
    }
}

// ===== @Validated（参数校验）=====
// 需要引入：spring-boot-starter-validation
// <dependency>
//     <groupId>org.springframework.boot</groupId>
//     <artifactId>spring-boot-starter-validation</artifactId>
// </dependency>

@Data
public class ChatRequest {
    @NotBlank(message = "消息内容不能为空")
    private String message;

    @Size(max = 100, message = "chatId 最多 100 字符")
    private String chatId;
}

@RestController
public class LoveAppController {
    @PostMapping("/chat")
    public ApiResponse<String> chat(@Valid @RequestBody ChatRequest request) {
        // @Valid 触发校验，失败则抛出 MethodArgumentNotValidException
        return ApiResponse.success(loveApp.doChat(request.getMessage(), request.getChatId()));
    }
}
```

### 1-9 事务管理

```java
// ===== @Transactional（声明式事务，最常用）=====
// 位置：类上（所有方法）或方法上（单个方法）
@Service
public class TransferService {

    @Autowired
    private AccountMapper accountMapper;

    @Transactional(rollbackFor = Exception.class)
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        // 扣钱
        accountMapper.decreaseBalance(fromId, amount);
        // 加钱
        accountMapper.increaseBalance(toId, amount);
        // 如果抛异常，Spring 会自动回滚
    }

    // ===== 事务传播行为 =====
    // required（默认）：如果当前有事务，加入事务；没有则创建新事务
    // requires_new：总是创建新事务，挂起当前事务
    // nested：如果当前有事务，在事务嵌套中执行；没有则创建新事务
    // supports：如果当前有事务，加入；没有就不按事务执行
    // mandatory：必须在事务中执行，否则抛异常
    // never：不能在事务中执行，否则抛异常
    // not_supported：不在事务中执行，挂起当前事务

    // ===== isolation（隔离级别）=====
    // isolation = Isolation.DEFAULT  // 使用数据库默认（MySQL 是 RR）
    // isolation = Isolation.READ_COMMITTED
    // isolation = Isolation.REPEATABLE_READ
    // isolation = Isolation.SERIALIZABLE

    // ===== 回滚规则 =====
    // 默认：运行时异常（RuntimeException）和 Error 自动回滚
    // rollbackFor = Exception.class：所有异常都回滚（包括受检异常）
    @Transactional(rollbackFor = Exception.class)
    public void saveWithCheckException() throws IOException {
        // IOException 是受检异常，不加 rollbackFor 不会回滚
        userMapper.insert(user);
        throw new IOException("IO错误");  // 不会回滚？不，加了 rollbackFor 才会
    }

    // noRollbackFor：指定不回滚的异常
    @Transactional(noRollbackFor = BusinessException.class)
    public void handle() {
        throw new BusinessException("业务异常但不回滚");
    }

    // ===== 只读事务 =====
    @Transactional(readOnly = true)
    public List<User> listAll() {
        // 告诉数据库这是只读，数据库会做优化
        return userMapper.selectAll();
    }

    // ===== 超时 =====
    @Transactional(timeout = 5)  // 超过 5 秒自动回滚
    public void longOperation() throws InterruptedException {
        Thread.sleep(10000);  // 会超时回滚
    }

    // ===== 手动编程式事务（复杂场景）=====
    @Autowired
    private PlatformTransactionManager transactionManager;

    public void programmatic() {
        DefaultTransactionDefinition def = new DefaultTransactionDefinition();
        def.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
        TransactionStatus status = transactionManager.getTransaction(def);
        try {
            accountMapper.decreaseBalance(fromId, amount);
            accountMapper.increaseBalance(toId, amount);
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }

    // ===== 编程式事务简化版（TransactionTemplate）=====
    @Autowired
    private TransactionTemplate transactionTemplate;

    public void templateDemo() {
        transactionTemplate.executeWithoutResult(status -> {
            accountMapper.decreaseBalance(fromId, amount);
            accountMapper.increaseBalance(toId, amount);
        });
    }
}

// ===== @Transactional 失效场景（面试高频）=====
// 1. 非 public 方法 → private 方法，Spring AOP 无法代理
// 2. 类内部自调用 → this.method() 不走代理对象，直接失效
// 3. 异常被 catch 吞掉 → 异常未抛出，事务不知道要回滚
// 4. 多数据源 → 默认只管主数据源，其他数据源需配置
```

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
