---
tags:
  - AI
  - SpringAI
  - 入门
created: 2026-08-17
---

# Spring AI — 快速入门

> 从 [[Spring AI 与 RAG 实战]] 拆分而来。

---

## 代码模板

### 1-1 项目依赖（pom.xml）

```xml
<!-- Spring Boot 父 POM 已管理版本，只需指定版本号 -->
<!-- chen-ai-agent 实际使用这些依赖： -->

<!-- Spring AI 核心（必选） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model</artifactId>
</dependency>

<!-- 阿里 DashScope（通义千问/Qwen）—— 你项目在用 -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-alibaba-starter</artifactId>
</dependency>

<!-- DeepSeek（性价比高，备用） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-deepseek</artifactId>
</dependency>

<!-- OpenAI（调用 Claude/GPT） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>

<!-- 向量数据库：Chroma（轻量，适合本地开发） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-store-chromadb</artifactId>
</dependency>

<!-- 向量数据库：PgVector（PostgreSQL 插件，PostgreSQL 够用） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-store-pgvector</artifactId>
</dependency>

<!-- 文档处理：PDF/Word/TXT -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-store-pdf</artifactId>
</dependency>

<!-- JSON 处理 -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>

<!-- Lombok（简化 POJO） -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>

<!-- 消息端点（SSE 流式响应） -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

### 1-2 配置文件（application.yml）

```yaml
# ===== chen-ai-agent 实际配置 =====

spring:
  application:
    name: chen-ai-agent

  # ===== DashScope（通义千问）—— 主模型 =====
  ai:
    dashscope:
      api-key: ${AI_DASHSCOPE_API_KEY}  # 从环境变量读取，不写死
      base-url: https://dashscope.aliyuncs.com/compatible-mode/v1
      # 可用模型：qwen-plus（主力）/ qwen-turbo（快且便宜）/ qwen-max（最强）
      chat:
        options:
          model: qwen-plus
          temperature: 0.7
          max-tokens: 2048

  # ===== DeepSeek（备用，性价比高）=====
  # ai:
  #   deepseek:
  #     api-key: ${AI_DEEPSEEK_API_KEY}
  #     base-url: https://api.deepseek.com
  #     chat:
  #       options:
  #         model: deepseek-chat
  #         temperature: 0.7

  # ===== OpenAI（Claude/GPT）=====
  # ai:
  #   openai:
  #     api-key: ${OPENAI_API_KEY}
  #     chat:
  #       options:
  #         model: gpt-4o-mini

  # ===== Chroma 向量库（本地开发）=====
  # spring:
  #   data:
  #     chromadb:
  #       client:
  #         host: localhost
  #         port: 8000

  # ===== PgVector（生产环境推荐）=====
  # spring:
  #   datasource:
  #     url: jdbc:postgresql://localhost:5432/vectordb
  #     username: postgres
  #     password: ${DB_PASSWORD}

  # ===== SSE 流式响应 =====
  # Server-Sent Events：AI 一边生成一边返回，不用等全部生成完
  # 在 Controller 层配置，参考 1-5 节

server:
  port: 8123

logging:
  level:
    root: INFO
    org.springframework.ai: DEBUG          # 开启 Spring AI DEBUG 日志
    io.github.chenyouxin8.chenaiagent: DEBUG  # 项目日志
```

### 1-3 简单对话（最小可用代码）

```java
// ===== 方式1：最简对话（Spring AI 2.x 推荐用法）=====
// 参考：chen-ai-agent 中 AiController.java 的简化版

@RestController
@RequestMapping("/api/demo")
public class SimpleChatController {

    // 注入 ChatClient（Spring AI 2.x 主入口）
    private final ChatClient chatClient;

    public SimpleChatController(ChatClient.Builder builder,
                                @Value("${spring.ai.dashscope.api-key}") String apiKey) {
        // 手动构建 ChatClient（用于自定义 Base URL 的情况）
        this.chatClient = builder
                .defaultApiKey(apiKey)
                .build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        // 最简调用：传入字符串，自动构造 Prompt
        String response = chatClient.prompt()
                .user(message)
                .call()
                .content();  // 直接拿文本响应
        return response;
    }
}

// ===== 方式2：用 System Prompt（设置角色）=====
@RestController
@RequestMapping("/api/demo2")
public class ChatWithRoleController {

    private final ChatClient chatClient;

    public ChatWithRoleController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @GetMapping("/chat-with-role")
    public String chatWithRole(@RequestParam String message) {
        // 设置系统提示词（让 AI 扮演特定角色）
        String response = chatClient.prompt()
                .system("""
                    你是一个专业、热情的恋爱顾问。
                    你的名字叫"恋爱专家小爱"。
                    回答要温暖、有同理心，同时给出实用建议。
                    """)
                .user(message)
                .call()
                .content();
        return response;
    }
}

// ===== 方式3：Message 对象（显式构造）=====
@RestController
@RequestMapping("/api/demo3")
public class ChatWithMessagesController {

    private final ChatClient chatClient;

    public ChatWithMessagesController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @GetMapping("/chat-messages")
    public String chatMessages(@RequestParam String message,
                               @RequestParam(defaultValue = "default") String chatId) {
        // 显式构造 Message 列表（适合复杂对话）
        List<Message> history = chatMemory.getMessages(chatId);  // 读取记忆

        Prompt prompt = Prompt.of(List.of(
                MessageBuilder
                        .createSystemMessage()
                        .text("你是一个恋爱专家。")
                        .build(),
                MessageBuilder
                        .createUserMessage()
                        .text(message)
                        .build()
        ));

        // 将历史消息注入到 Prompt
        if (!history.isEmpty()) {
            prompt.getInstructions().addAll(0, history);
        }

        String response = chatClient.prompt(prompt).call().content();

        // 保存记忆
        chatMemory.addMessage(chatId, MessageBuilder
                .createUserMessage().text(message).build());
        chatMemory.addMessage(chatId, MessageBuilder
                .createAssistantMessage().text(response).build());

        return response;
    }

    @Autowired
    private ChatMemory chatMemory;  // 见 1-4 节
}
```


---

## 🔗 相关笔记

- [[Spring AI 与 RAG 实战]]
- [[../04-Spring生态/SpringBoot框架|Spring Boot 框架]]
- [[../03-Database/MySQL数据库|MySQL 数据库]]
