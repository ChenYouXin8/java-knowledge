---
tags:
  - Spring
  - SpringBoot
  - Web
  - MVC
  - 事务
created: 2026-08-17
---

# Spring Boot — Web 开发

> 从 [[Spring Boot 框架]] 拆分而来。

---

## 代码模板

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

## 🔗 相关笔记

- [[Spring Boot 框架]]
- [[../05-AI应用开发/SpringAI与RAG实战|Spring AI 与 RAG 实战]]
- [[../01-Java基础/反射|反射]]
- [[../06-工具链/01-Maven实战|Maven 实战]]
- [[../03-Database/MySQL数据库|MySQL 数据库]]
