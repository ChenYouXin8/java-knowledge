# Spring AI 与 RAG 实战

> 这是你 chen-ai-agent 项目的核心技术栈。本笔记从项目实际代码出发，覆盖：Spring AI 核心概念、多模型接入、Chat Memory、RAG 全流程、Function Calling、Prompt 工程、向量库选型。**理论 + 实战对照，学完就能改自己的项目。**

---

## 1️⃣ 代码模板

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

### 1-4 Chat Memory（记忆持久化）

```java
// ===== ChatMemory：AI 对话的"记忆"，让 AI 知道之前聊了什么 =====
// Spring AI 支持多种 ChatMemory 实现：

// ===== 1. InMemoryChatMemory（内存，进程重启丢失）=====
// 只适合测试，生产不用
@Configuration
public class MemoryConfig {
    @Bean
    public ChatMemory inMemoryChatMemory() {
        return new InMemoryChatMemory();
    }
}

// ===== 2. SimpleTokenMemory（基于 token 数限制，适合轻量场景）=====
// 按 token 数量自动截断对话历史，防止超出模型上下文限制
@Configuration
public class MemoryConfig {
    @Bean
    public ChatMemory simpleTokenMemory(TokenCountEstimator estimator) {
        // 最大 token 数，超出则自动截断旧消息
        return new SimpleMessageWindowChatMemory(estimator, 4096);
    }
}

// ===== 3. FileBasedChatMemory（文件持久化，你项目在用！）=====
// 对话历史保存到文件，进程重启后还能恢复

@Configuration
public class MemoryConfig {

    // 对话记忆文件存储目录
    private static final String MEMORY_DIR = "data/chat-memory/";

    @Bean
    public ChatMemory fileBasedChatMemory() {
        // 文件名 = chatId.txt，内容是 JSON 格式的对话历史
        // 对话文件示例：data/chat-memory/default.txt
        return FileBasedChatMemory.builder()
                .maxMessages(100)           // 最多存 100 条消息
                .build();
    }

    // 你项目实际代码（参考 LoveApp.java）：
    // public class LoveApp {
    //     private final ChatMemory chatMemory = FileBasedChatMemory.builder()
    //         .maxMessages(100)
    //         .build();
    // }
}

// ===== 4. RedisChatMemory（分布式，多实例共享）=====
// 生产环境推荐，多台机器共享同一个记忆
@Configuration
public class MemoryConfig {
    @Bean
    public ChatMemory redisChatMemory(RedisConnectionFactory factory) {
        return RedisChatMemory.builder()
                .redisTemplate(new StringRedisTemplate(factory))
                .build();
    }
}

// ===== 使用 ChatMemory 的完整例子 =====
// 参考：chen-ai-agent 中 LoveApp.java 的结构

@Service
public class LoveApp {

    // 记忆：每个 chatId 独立的对话历史
    private final ChatMemory chatMemory = FileBasedChatMemory.builder()
            .maxMessages(100)
            .build();

    private final ChatClient chatClient;

    public LoveApp(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    /**
     * 对话方法（核心）
     * @param message 用户消息
     * @param chatId  对话会话 ID（相当于 QQ 聊天窗口）
     * @return AI 回复
     */
    public String doChat(String message, String chatId) {
        // 1. 获取该会话的历史消息
        List<Message> history = chatMemory.get(chatId);

        // 2. 构造 Prompt（系统提示词 + 历史 + 当前消息）
        Prompt prompt = Prompt.builder()
                .messages(history)  // 先放历史消息
                .system("""
                    # 角色设定
                    你是一个专业、热情的恋爱顾问，帮助用户解决恋爱困惑。
                    ## 回答风格
                    - 温暖、有同理心
                    - 给出具体可操作的建议
                    - 如需更多背景信息，主动提问
                    ## 知识范围
                    - 单身人群的情感困惑
                    - 恋爱中的沟通与相处
                    - 婚姻家庭的经营
                    """)
                .user(message)  // 再加当前消息
                .build();

        // 3. 调用 AI
        String response = chatClient.prompt(prompt).call().content();

        // 4. 保存到记忆（用户消息 + AI 回复都要存）
        chatMemory.add(chatId,
                MessageBuilder.createUserMessage().text(message).build(),
                MessageBuilder.createAssistantMessage().text(response).build()
        );

        return response;
    }

    // ===== 清除某个会话的记忆 =====
    public void clearMemory(String chatId) {
        chatMemory.clear(chatId);
    }

    // ===== 读取历史 =====
    public List<Message> getHistory(String chatId) {
        return chatMemory.get(chatId);
    }
}
```

### 1-5 SSE 流式响应（打字机效果）

```java
// ===== SSE：让 AI 回复"一个字一个字"出来 =====
// 参考：chen-ai-agent 中 LoveAppController.java 的 doChatWithSSE

@RestController
@RequestMapping("/api/ai/love")
public class LoveAppController {

    private final LoveApp loveApp;

    public LoveAppController(LoveApp loveApp) {
        this.loveApp = loveApp;
    }

    // ===== 流式接口 =====
    // 返回类型必须是 Flux<ServerSentEvent<String>>
    // 前端用 EventSource 接收（类似 chat.openai.com 的效果）

    @GetMapping(value = "/chat-stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> chatStream(
            @RequestParam String message,
            @RequestParam(defaultValue = "default") String chatId
    ) {
        // 调用流式 API
        Flux<String> stream = loveApp.doChatStream(message, chatId);

        // 包装为 SSE 格式
        return stream
                .map(content -> ServerSentEvent.<String>builder()
                        .data(content)          // 数据内容
                        .event("message")       // 事件类型
                        .build())
                .onErrorReturn(ServerSentEvent.<String>builder()
                        .data("[ERROR] AI 服务出错，请稍后重试")
                        .build());
    }

    // ===== 非流式接口（简单返回）=====
    @GetMapping("/chat")
    public ApiResponse<String> chat(
            @RequestParam String message,
            @RequestParam(defaultValue = "default") String chatId
    ) {
        String response = loveApp.doChat(message, chatId);
        return ApiResponse.success(response);
    }
}

// ===== LoveApp 中的流式实现 =====
@Service
public class LoveApp {

    private final ChatClient chatClient;
    private final ChatMemory chatMemory;

    public LoveApp(ChatClient.Builder builder) {
        this.chatClient = builder.build();
        this.chatMemory = FileBasedChatMemory.builder().maxMessages(100).build();
    }

    /**
     * 流式对话（返回 Flux<String>）
     */
    public Flux<String> doChatStream(String message, String chatId) {
        Prompt prompt = Prompt.of(List.of(
                MessageBuilder.createSystemMessage().text("你是一个恋爱专家。").build(),
                MessageBuilder.createUserMessage().text(message).build()
        ));

        // 流式调用：chat() 返回 Flux<String>
        // 每个 String 是一个 chunk（片段），可以一个字一个字地发送
        Flux<String> stream = chatClient.prompt(prompt).stream().content();

        // 保存记忆（流结束后统一保存）
        stream.subscribe();  // 触发执行

        chatMemory.add(chatId,
                MessageBuilder.createUserMessage().text(message).build()
        );

        return stream;
    }
}

// ===== 前端接收示例（Fetch + EventSource）=====
// 前端 JS：
// const eventSource = new EventSource(`/api/ai/love/chat-stream?message=${msg}&chatId=${chatId}`);
// eventSource.addEventListener('message', (e) => {
//     document.getElementById('response').textContent += e.data;
// });
// eventSource.addEventListener('error', () => eventSource.close());
```

### 1-6 RAG（检索增强生成）

```java
// ===== RAG = Retrieval Augmented Generation =====
// 核心思想：让 AI 回答时先检索知识库，再基于检索结果生成答案
// 好处：AI 能回答"你自己的知识"，而不只是训练数据

// RAG 完整流程：
// 文档 → 分块(Chunk) → Embedding → 向量数据库
//                                         ↓
// 用户问题 → Embedding → 向量检索 → 检索结果 → Prompt → AI 回复

// ===== 第一步：文档加载 =====
// 支持格式：PDF / Word / TXT / Markdown / HTML / CSV

@Service
public class DocumentLoaderService {

    @Autowired
    private DocumentReader documentReader;

    /**
     * 加载恋爱知识文档
     * 文档路径：src/main/resources/document/
     * - 单身指南.md
     * - 恋爱技巧.md
     * - 婚姻经营.md
     */
    public List<Document> loadDocuments() {
        // 方式1：Markdown / TXT
        MarkdownDocumentReader reader = new MarkdownDocumentReader(
                new Path("src/main/resources/document/单身指南.md")
        );
        List<Document> docs = reader.get();

        // 方式2：PDF
        PdfDocumentReader pdfReader = new PdfDocumentReader(
                new Path("src/main/resources/document/知识库.pdf")
        );
        List<Document> pdfDocs = pdfReader.get();

        // 方式3：加载目录下所有文件
        // DirectoryLoader loader = new DirectoryLoader(
        //     "src/main/resources/document/",
        //     new MarkdownParser()
        // );

        // 合并所有文档
        docs.addAll(pdfDocs);
        return docs;
    }
}

// ===== 第二步：文本分块（Chunking）=====
// 为什么要分块？大文档拆成小段，Embedding 时精度更高

@Service
public class TextChunkingService {

    /**
     * 按段落分块（简单）
     */
    public List<TextSegment> chunkByParagraph(List<Document> documents) {
        List<TextSegment> segments = new ArrayList<>();

        for (Document doc : documents) {
            String content = doc.getText();
            // 按双换行分割段落
            String[] paragraphs = content.split("\\n\\n+");

            for (int i = 0; i < paragraphs.length; i++) {
                String text = paragraphs[i].trim();
                if (text.length() > 50) {  // 过滤太短的段落
                    TextSegment segment = TextSegment.from(text);
                    // 可以加元数据（metadata）
                    segment.getMetadata().put("source", doc.getMetadata().get("source"));
                    segment.getMetadata().put("chunk_index", i);
                    segments.add(segment);
                }
            }
        }
        return segments;
    }

    /**
     * 按 token 数分块（更精确）
     * 推荐每块 500-1000 tokens
     */
    public List<TextSegment> chunkByToken(List<Document> documents, int maxTokens) {
        List<TextSegment> segments = new ArrayList<>();
        TokenCountEstimator estimator = new TokenCountEstimator();

        for (Document doc : documents) {
            String text = doc.getText();
            // 简单切分：按固定字符数
            int chunkSize = maxTokens * 4;  // 粗略估算：1 token ≈ 4 字符
            for (int i = 0; i < text.length(); i += chunkSize) {
                int end = Math.min(i + chunkSize, text.length());
                String chunk = text.substring(i, end);
                segments.add(TextSegment.from(chunk));
            }
        }
        return segments;
    }
}

// ===== 第三步：Embedding → 向量数据库 =====
// Embedding：把文字变成向量（数字列表），语义相近的文字向量也相近

@Configuration
public class VectorStoreConfig {

    // ===== Chroma（本地开发用，轻量）=====
    @Bean
    public VectorStore chromaVectorStore(EmbeddingModel embeddingModel) {
        return ChromaVectorStore.builder(embeddingModel)
                .build();
        // 需要先启动 Chroma：docker run -d -p 8000:8000 chromadb/chroma
    }

    // ===== PgVector（生产环境推荐，基于 PostgreSQL）=====
    @Bean
    public VectorStore pgVectorStore(DataSource dataSource, EmbeddingModel embeddingModel) {
        return PgVectorStore.builder(dataSource, embeddingModel)
                .build();
        // 需要 PostgreSQL 开启 vector 扩展：CREATE EXTENSION IF NOT EXISTS vector;
    }

    // ===== 内存向量库（测试用，不持久化）=====
    @Bean
    public VectorStore inMemoryVectorStore(EmbeddingModel embeddingModel) {
        return new InMemoryVectorStore(embeddingModel);
    }
}

// ===== 第四步：写入向量数据库（RAG 写入）=====
@Service
public class RagService {

    private final VectorStore vectorStore;
    private final DocumentReader documentReader;

    public RagService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    /**
     * 初始化知识库（项目启动时调用一次）
     */
    @PostConstruct
    public void initKnowledgeBase() {
        // 1. 加载文档
        List<Document> docs = loadDocuments();  // 见上方代码

        // 2. 写入向量库（幂等操作，可重复执行）
        vectorStore.add(documents);
        System.out.println("知识库初始化完成，共 " + docs.size() + " 篇文档");
    }

    /**
     * 检索相关文档（RAG 的 R 部分）
     */
    public List<String> retrieve(String query, int topK) {
        // 1. 把用户问题转成向量
        // 2. 在向量库中找最相似的 topK 条
        List<Document> docs = vectorStore.similaritySearch(
                SearchRequest.query(query)   // 用户问题
                        .withTopK(topK)      // 取前 topK 条
                        .withSimilarityThreshold(0.7)  // 相似度阈值
        );

        return docs.stream()
                .map(Document::getText)
                .collect(Collectors.toList());
    }

    private List<Document> loadDocuments() {
        // 简化：直接用 MarkdownDocumentReader
        try {
            MarkdownDocumentReader reader = new MarkdownDocumentReader(
                    new Path("src/main/resources/document/单身指南.md")
            );
            return reader.get();
        } catch (Exception e) {
            throw new RuntimeException("加载文档失败", e);
        }
    }
}

// ===== 第五步：RAG 增强对话（最关键！）=====
@Service
public class LoveApp {

    private final ChatClient chatClient;
    private final ChatMemory chatMemory;
    private final RagService ragService;  // 注入 RAG 服务

    private static final String SYSTEM_PROMPT = """
            # 角色
            你是一个专业、热情的恋爱顾问。

            # 知识库使用规则
            当用户问到具体问题时，请先在知识库中检索相关信息，
            如果检索到了，结合知识库内容回答。
            如果知识库没有相关信息，可以基于你的通用知识回答。

            # 回答风格
            - 温暖、有同理心
            - 给出具体可操作的建议
            - 如需更多背景信息，主动提问
            """;

    public String doChatWithRag(String message, String chatId) {
        // 1. RAG 检索（从知识库找相关文档）
        List<String> relevantDocs = ragService.retrieve(message, 3);

        // 2. 把检索结果加入 Prompt
        String knowledgeContext = relevantDocs.isEmpty() ? ""
                : "【参考知识】\n" + String.join("\n---\n", relevantDocs);

        // 3. 构造增强版 Prompt
        String enhancedPrompt = SYSTEM_PROMPT + "\n" + knowledgeContext;

        Prompt prompt = Prompt.of(List.of(
                MessageBuilder.createSystemMessage().text(enhancedPrompt).build(),
                MessageBuilder.createUserMessage().text(message).build()
        ));

        // 4. 调用 AI
        String response = chatClient.prompt(prompt).call().content();

        // 5. 保存记忆
        chatMemory.add(chatId,
                MessageBuilder.createUserMessage().text(message).build(),
                MessageBuilder.createAssistantMessage().text(response).build()
        );

        return response;
    }
}

// ===== 你项目中的 RAG 接口（参考 LoveAppController）=====
// @GetMapping("/rag")
// public ApiResponse<String> rag(@RequestParam String message) {
//     String response = loveApp.doChatWithRag(message, "default");
//     return ApiResponse.success(response);
// }
```

### 1-7 Function Calling / Tool Use（让 AI 调工具）

```java
// ===== Function Calling =====
让 AI 在回复之前，先"调用"你定义的 Java 方法，再基于结果回复。
好处：AI 能做实时查询、文件操作、API 调用等，而不是胡说八道。

// ===== 第一步：定义工具接口 =====
// 参考：chen-ai-agent 中 tools/ 包下的工具类

// 工具方法必须是 Spring Bean，且加 @Tool 或 @ToolDescription 注解
// Spring AI 会自动把工具暴露给 AI，让 AI 决定何时调用

@Service
public class WebSearchTool {

    // @Tool：标记为 AI 可调用的工具
    @Tool(name = "web_search", description = "搜索网络获取最新信息，输入是搜索关键词，返回搜索结果摘要")
    public String search(@UserParam(description = "搜索关键词") String query) {
        // 实际调用搜索 API（见 chen-ai-agent 中的 WebSearchTool）
        return callSearchApi(query);
    }

    private String callSearchApi(String query) {
        // 调用 SearchAPI（用户配置的 API Key）
        // ...
        return "搜索结果：...";
    }
}

@Service
public class FileOperationTool {

    @Tool(name = "read_file", description = "读取本地文件内容，输入是文件路径")
    public String readFile(@UserParam(description = "完整的文件路径") String filePath) {
        try {
            return Files.readString(Path.of(filePath));
        } catch (IOException e) {
            return "读取文件失败：" + e.getMessage();
        }
    }

    @Tool(name = "write_file", description = "写入内容到本地文件，输入是文件路径和内容")
    public String writeFile(
            @UserParam(description = "完整的文件路径") String filePath,
            @UserParam(description = "要写入的内容") String content
    ) {
        try {
            Files.writeString(Path.of(filePath), content);
            return "写入成功：" + filePath;
        } catch (IOException e) {
            return "写入失败：" + e.getMessage();
        }
    }
}

// ===== 第二步：注册工具 =====
// 参考：chen-ai-agent 中 ToolRegistration.java

@Configuration
public class ToolRegistration {

    @Autowired
    private WebSearchTool webSearchTool;

    @Autowired
    private FileOperationTool fileOperationTool;

    @Bean
    public ToolCallbackAware toolCallbackAware() {
        return ToolCallbackAware.of(
                webSearchTool,    // 注册搜索工具
                fileOperationTool // 注册文件工具
        );
    }
}

// ===== 第三步：使用工具调用 =====
// 参考：chen-ai-agent 中 AiController / doChatWithTools

@Service
public class AiService {

    private final ChatClient chatClient;

    // 自动注入所有 @Tool 方法
    private final List<ToolCallback> toolCallbacks;

    public AiService(ChatClient.Builder builder, List<ToolCallback> toolCallbacks) {
        this.chatClient = builder.build();
        this.toolCallbacks = toolCallbacks;
    }

    /**
     * 带工具调用的对话
     */
    public String doChatWithTools(String message) {
        Prompt prompt = Prompt.of(List.of(
                MessageBuilder.createSystemMessage().text("你是一个有帮助的助手。").build(),
                MessageBuilder.createUserMessage().text(message).build()
        ));

        // 告诉 AI 有哪些工具可用
        ToolCallback[] callbacks = toolCallbacks.toArray(new ToolCallback[0]);

        // 调用（AI 会自动决定是否调用工具）
        String response = chatClient.prompt(prompt)
                .tools(callbacks)  // ← 关键：注册工具
                .call()
                .content();

        return response;
    }

    /**
     * 完整工具调用流程（含工具执行 + 结果反馈）
     * Spring AI 会自动处理：AI调用工具 → 执行 → 把结果反馈给AI → AI生成最终回复
     */
    public String doChatWithToolsAdvanced(String message) {
        Prompt prompt = Prompt.of(List.of(
                MessageBuilder.createSystemMessage().text("你是一个有帮助的助手。").build(),
                MessageBuilder.createUserMessage().text(message).build()
        ));

        // 方式1：Spring AI 自动处理整个流程
        // chatClient 会自动：调用工具 → 执行 → 把结果注入 Prompt → AI 最终回复
        ChatResponse response = chatClient.prompt(prompt)
                .tools(toolCallbacks.toArray(new ToolCallback[0]))
                .toolContext(new SimpleToolContext())
                .call()
                .chatResponse();

        return response.getResult().getOutput().getText();

        // 方式2：手动处理（高级用法，可以自定义工具调用逻辑）
        // 需要自己实现循环：AI 请求调用工具 → 执行 → 反馈结果给 AI → ...
    }
}

// ===== 工具执行结果（Spring AI 自动处理）=====
// 完整流程（Spring AI 自动管理）：
// 1. AI 发请求：{"tool_calls": [{"name": "web_search", "arguments": {"query": "济南天气"}}]}
// 2. Spring AI 执行：webSearchTool.search("济南天气") → "济南今天晴天，28度"
// 3. AI 收到结果，生成最终回复
```

### 1-8 Prompt 工程（系统提示词编写）

```java
// ===== 好的 System Prompt =====
/*
 * 好的 System Prompt 应该包含：
 * 1. 角色定义（Who are you?）
 * 2. 能力边界（What can you do?）
 * 3. 回答风格（How should you respond?）
 * 4. 限制条件（What should you avoid?）
 * 5. 输出格式（How should you format?）
 */

// ===== 恋爱专家 System Prompt（你项目实际使用）=====
private static final String LOVE_EXPERT_PROMPT = """
    # 角色设定
    你是「恋爱专家小爱」，一个专业、温暖、有同理心的 AI 恋爱顾问。

    ## 核心能力
    - 分析用户的情感困惑和恋爱困境
    - 提供具体可执行的恋爱建议和沟通技巧
    - 帮助用户理解异性心理
    - 引导用户建立健康的恋爱关系

    ## 回答风格
    - 温暖亲切，像朋友聊天一样
    - 主动提问了解更多背景，不急于给建议
    - 建议要具体、可操作，不是泛泛而谈
    - 如涉及原则问题（家暴/PUA/违法），要明确指出并建议求助专业人士

    ## 禁止行为
    - 编造具体人物姓名或案例
    - 对用户进行道德评判
    - 给出可能造成伤害的建议

    ## 输出格式
    - 先表达理解和共情
    - 再给出分析和建议
    - 最后问一个开放式问题继续对话
    """;

// ===== ReAct Agent System Prompt（你项目中的 ChenManus）=====
private static final String CHEN_MANUS_PROMPT = """
    # ChenManus 智能助手
    你是一个能够自主思考和执行任务的 AI 助手。

    ## 工作模式（ReAct）
    你会按以下步骤工作：
    1. Thought（思考）：分析当前任务，决定下一步
    2. Action（行动）：调用工具或执行操作
    3. Observation（观察）：获取行动结果
    4. 返回第1步继续，直到任务完成

    ## 可用工具
    - web_search：搜索网络
    - read_file：读取文件
    - write_file：写入文件
    - terminal：执行命令行
    - rag_search：搜索知识库

    ## 输出格式
    请按以下格式输出：
    Thought: [你的思考]
    Action: [工具名称] [参数]
    Observation: [工具返回结果]
    ...（重复上述步骤）
    Answer: [最终答案]
    """;

// ===== Few-Shot 示例（给 AI 举例子）=====
private static final String FEW_SHOT_PROMPT = """
    # 你是恋爱专家小爱

    ## 示例对话

    用户：我觉得他不爱我了
    小爱：听起来你最近感受到了一些不确定和担心。能告诉我是什么让你有这样的感觉吗？
         最近你们的相处方式有什么变化吗？

    用户：他最近总是加班，没时间陪我
    小爱：工作忙确实会影响相处时间。我想了解一下：
         - 他之前也是这样吗，还是最近才开始的？
         - 你有没有和他沟通过你的感受？
         - 他有没有尝试过在有限的时间里给你一些特别的关注？
    """;

// ===== 动态 Prompt（根据上下文调整）=====
public String buildDynamicPrompt(String context, String userQuery) {
    StringBuilder sb = new StringBuilder();
    sb.append("【背景】").append(context).append("\n");
    sb.append("【用户问题】").append(userQuery).append("\n");
    sb.append("请基于以上背景，回答用户问题。");
    return sb.toString();
}
```

### 1-9 Embedding 模型配置

```yaml
# ===== DashScope Embedding（你项目 RAG 在用）=====
spring:
  ai:
    dashscope:
      # 文本 Embedding 模型（向量化）
      embedding:
        options:
          model: text-embedding-v3  # 阿里最新 Embedding 模型
          # 其他可选：
          # text-embedding-v2（上一代）
          # text-embedding-v1（更老）

# ===== DeepSeek Embedding ======
# spring:
#   ai:
#     deepseek:
#       embedding:
#         api-key: ${AI_DEEPSEEK_API_KEY}

# ===== OpenAI Embedding（参考）=====
# spring:
#   ai:
#     openai:
#       embedding:
#         api-key: ${OPENAI_API_KEY}
#         options:
#           model: text-embedding-3-small  # 便宜快速
#           # text-embedding-3-large      # 更强更贵
```

---

## 2️⃣ 对比表格

### 2-1 Spring AI 版本差异（1.x vs 2.x）

| 维度 | Spring AI 1.x | Spring AI 2.x（你项目用） |
|------|--------------|--------------------------|
| 发布 | 2023年 | 2024年 |
| Java 要求 | 17+ | 17+（推荐 21） |
| 主入口 | `RestTemplate` / `WebClient` | **ChatClient**（新统一入口） |
| ChatMemory | `ChatMemory` | `ChatMemory`（接口不变） |
| VectorStore | `VectorStore` | `VectorStore`（接口基本不变） |
| Function Calling | `@Tool` 注解 | `@Tool` 注解（更简洁） |
| Advisor | 不支持 | **Advisor**（新功能，拦截器链） |
| Starter 命名 | `spring-ai-alibaba-starter` | `spring-ai-alibaba-starter`（不变） |
| 升级建议 | 升级到 2.x | 直接用 2.x |

### 2-2 LLM 模型对比（DashScope / DeepSeek / OpenAI）

| 模型 | 提供商 | 特点 | 价格 | 适合场景 |
|------|--------|------|------|---------|
| **qwen-plus** | 阿里 DashScope | 国产强模型，中文好 | 中等 | **主力模型，你项目在用** |
| qwen-turbo | 阿里 DashScope | 速度快，便宜 | 低 | 简单对话、批处理 |
| qwen-max | 阿里 DashScope | 最强，复杂推理 | 高 | 高难度任务 |
| **deepseek-chat** | DeepSeek | 性价比极高，推理能力强 | **很低** | **备用首选** |
| deepseek-coder | DeepSeek | 代码能力强 | 低 | 代码相关任务 |
| gpt-4o-mini | OpenAI | 便宜，快速 | 低 | 简单任务 |
| gpt-4o | OpenAI | 最强通用模型 | 高 | 最高难度任务 |
| claude-3.5-sonnet | Anthropic | 写作强，长上下文 | 高 | 写作、分析 |

### 2-3 向量数据库选型

| 向量库 | 类型 | 优点 | 缺点 | 适合场景 |
|--------|------|------|------|---------|
| **Chroma** | 本地/轻量 | 轻量易用，Python/JS生态好 | 单机，生产需配合其他存储 | **本地开发** |
| **PgVector** | PostgreSQL 插件 | 基于已有 PostgreSQL，运维简单 | 需要额外扩展 | **生产环境首选** |
| Milvus | 分布式 | 支持TB级数据，高可用 | 运维复杂 | 超大规模 |
| Pinecone | 云服务 | 全托管，免运维 | 收费，境外 | 不想运维 |
| Weaviate | 混合 | 原生支持混合搜索（向量+关键词） | 资源占用较大 | 混合搜索场景 |
| Qdrant | Rust 实现 | 性能高，支持过滤 | 生态稍弱 | 高性能需求 |
| **Elasticsearch** | 企业搜索 | 已有ES集群可复用 | 不是专用向量库 | 已有 ES 场景 |

### 2-4 RAG 方案对比

| 方案 | 难度 | 效果 | 成本 | 推荐 |
|------|------|------|------|------|
| 纯 LLM（无 RAG） | 最低 | 一般（幻觉多） | 低 | 简单问答 |
| **知识库 + 检索** | 中 | 好（具体事实） | 中 | **你项目当前方案** |
| RAG + 重排序 | 高 | 更好（去噪音） | 高 | 精确度要求高 |
| Agentic RAG | 高 | 最好（自主决策） | 高 | 复杂多跳问答 |
| 长上下文（Context Window） | 低 | 好（无需检索） | 高 | 小规模知识库 |

### 2-5 ChatMemory 实现对比

| 实现 | 持久化 | 多实例共享 | 适用场景 |
|------|--------|----------|---------|
| InMemoryChatMemory | ❌（内存） | ❌ | 测试 |
| SimpleMessageWindowChatMemory | ❌ | ❌ | 轻量，单实例 |
| **FileBasedChatMemory** | ✅（文件） | ❌ | **你项目在用** |
| RedisChatMemory | ✅（Redis） | ✅ | 生产，多实例 |
| JPA / JDBC ChatMemory | ✅（数据库） | ✅ | 生产，已有数据库 |

### 2-6 Embedding 模型对比

| 模型 | 维度 | 特点 | 适用语言 |
|------|------|------|---------|
| **text-embedding-v3** | 1536 | 阿里最新，效果好 | **中文为主** |
| text-embedding-v2 | 1536 | 稳定 | 中文 |
| m3e-base | 768 | 国产开源，免费 | 中文 |
| bge-large-zh | 1024 | 国产开源，效果好 | 中文 |
| text-embedding-3-small | 1536 | OpenAI，快速 | 多语言 |
| text-embedding-3-large | 3072 | OpenAI，效果最强 | 多语言 |

---

## 3️⃣ 图解结构

### 3-1 RAG 完整流程

```
                    ┌─────────────────────────────────────────────┐
                    │           RAG（检索增强生成）               │
                    │                                             │
┌─────────┐        │  ┌──────────────────┐                      │
│  文档   │        │  │ 1. 文档加载（Loader）│                   │
│ .md/.txt│ ────────▶│ - MarkdownDocument │                   │
│ .pdf    │        │  │ - PdfDocumentReader│                   │
└─────────┘        │  └────────┬─────────┘                      │
                    │           │                                │
                    │           ▼                                │
                    │  ┌──────────────────┐                      │
                    │  │ 2. 文本分块（Chunk）│                  │
                    │  │ - 按段落           │                   │
                    │  │ - 按 token 数       │                   │
                    │  │ - 重叠分块          │                   │
                    │  └────────┬─────────┘                      │
                    │           │                                │
                    │           ▼                                │
                    │  ┌──────────────────┐                      │
                    │  │ 3. Embedding 向量化│                   │
                    │  │ - DashScope Embed │  ┌──────────────┐ │
                    │  │ - OpenAI Embed   │──▶│   向量数据库   │ │
                    │  │ - 本地模型        │  │  (Chroma /  │ │
                    │  └──────────────────┘  │   PgVector)  │ │
                    │                         └──────────────┘ │
                    │                                            │
                    │  ┌──────────────────┐                      │
                    │  │ 4. 用户问题向量化  │                      │
                    │  │ - query → vector │                      │
                    │  └────────┬─────────┘                      │
                    │           │                                 │
                    │           ▼                                 │
                    │  ┌──────────────────┐                      │
                    │  │ 5. 向量相似度检索  │                      │
                    │  │ - topK = 3        │                      │
                    │  │ - 相似度阈值 > 0.7 │                   │
                    │  └────────┬─────────┘                      │
                    │           │                                 │
                    │           ▼                                 │
                    │  ┌──────────────────┐                      │
                    │  │ 6. 构造增强 Prompt │                      │
                    │  │ System Prompt      │                      │
                    │  │ + 检索结果片段      │                      │
                    │  │ + 用户问题          │                      │
                    │  └────────┬─────────┘                      │
                    │           │                                 │
                    │           ▼                                 │
                    │  ┌──────────────────┐                      │
                    │  │ 7. LLM 生成回答   │                      │
                    │  │ (qwen-plus /     │                      │
                    │  │  deepseek-chat)   │                      │
                    │  └──────────────────┘                      │
                    │           │                                 │
                    │           ▼                                 │
                    │  ┌──────────────────┐                      │
                    │  │ 8. 返回给用户      │                      │
                    │  └──────────────────┘                      │
                    └─────────────────────────────────────────────┘
```

### 3-2 Spring AI 核心架构

```
Spring AI 2.x 架构：

┌──────────────────────────────────────────────────────────────┐
│                      你的应用代码（Java）                      │
│                                                               │
│   ┌────────────────┐     ┌────────────────┐                  │
│   │  Controller层  │     │  Service层     │                  │
│   │  (REST API)    │ ──▶ │  (业务逻辑)     │                  │
│   └────────────────┘     └───────┬────────┘                  │
│                                  │                           │
│                                  ▼                           │
│                        ┌────────────────┐                    │
│                        │   ChatClient   │ ← 统一主入口       │
│                        └───────┬────────┘                    │
│                                │                             │
│         ┌──────────────────────┼──────────────────────┐     │
│         │                      │                       │     │
│         ▼                      ▼                       ▼     │
│  ┌─────────────┐     ┌──────────────┐     ┌─────────────┐   │
│  │   Prompt    │     │    Advisor    │     │   Tool      │   │
│  │  (提示词)   │     │  (拦截器链)   │     │  (工具)     │   │
│  │             │     │              │     │             │   │
│  │ - System    │     │ - Logger     │     │ - @Tool     │   │
│  │ - User      │     │ - Memory     │     │ - Function  │   │
│  │ - Messages  │     │ - RAG        │     │   Calling   │   │
│  └─────────────┘     └──────────────┘     └─────────────┘   │
│                                 │                            │
└─────────────────────────────────┼────────────────────────────┘
                                  ▼
┌──────────────────────────────────────────────────────────────┐
│                    模型层（Model Abstraction）                │
│                                                               │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│   │  DashScope  │  │   DeepSeek   │  │      OpenAI         │  │
│   │  (通义千问)  │  │  (性价比高)  │  │  (Claude/GPT)      │  │
│   │             │  │             │  │                     │  │
│   │ API: REST  │  │ API: REST   │  │ API: REST          │  │
│   └─────────────┘  └─────────────┘  └─────────────────────┘  │
│                                                               │
│                    Embedding 模型（同上）                       │
│                    向量存储（同上）                             │
└───────────────────────────────────────────────────────────────┘
```

### 3-3 chen-ai-agent 整体架构

```
chen-ai-agent 项目架构（对应你的实际代码）：

src/main/java/io/github/chenyouxin8/chenaiagent/

├── ChenAiAgentApplication.java          ← 启动类
│
├── controller/
│   ├── AiController.java                ← /api/ai/* 路由
│   └── LoveAppController.java            ← /api/ai/love/* 恋爱专家
│
├── service/
│   ├── LoveApp.java                     ← 核心：对话 + 记忆 + RAG
│   └── LoveReport.java                   ← 恋爱报告生成
│
├── memory/
│   └── FileBasedChatMemory.java          ← 文件持久化记忆（你自己写的）
│
├── tools/                                ← Function Calling 工具
│   ├── WebSearchTool.java                ← 网络搜索
│   ├── FileOperationTool.java            ← 文件读写
│   ├── WebScrapingTool.java              ← 网页抓取
│   ├── TerminalOperationTool.java         ← 命令行执行（安全白名单）
│   ├── PDFGenerationTool.java            ← PDF 生成
│   └── ToolRegistration.java             ← 工具注册
│
├── agent/
│   ├── BaseAgent.java                    ← 基础 Agent
│   ├── ReActAgent.java                   ← ReAct 思考模式
│   └── ChenManus.java                    ← 你的 AI 助手
│
├── rag/                                  ← RAG 知识库
│   ├── LoveDocumentLoader.java           ← 恋爱文档加载
│   └── RagService.java                   ← 检索服务
│
├── advisor/
│   └── MyLoggerAdvisor.java              ← 日志拦截（CallAdvisor）
│
└── common/
    ├── ApiResponse.java                  ← 统一响应格式
    └── BusinessException.java             ← 业务异常

src/main/resources/
├── document/                             ← RAG 知识库文档
│   ├── 单身指南.md
│   ├── 恋爱技巧.md
│   └── 婚姻经营.md
└── application.yml                       ← 配置

data/chat-memory/                         ← 对话记忆文件（运行生成）
    ├── default.txt
    └── ...
```

### 3-4 ReAct 思考模式流程

```
ReAct = Thought + Action + Observation 循环

用户问题：济南明天天气怎么样？

Step 1: Thought（思考）
  → 用户想知道济南明天天气，我需要先搜索。
  Action: web_search
  Arguments: {"query": "济南 明天 天气"}

Step 2: Action（执行工具）
  → 调用 WebSearchTool.search("济南 明天 天气")
  Observation: "济南明天多云转晴，24-30度"

Step 3: Thought（思考）
  → 搜索到了天气信息，我可以回答用户了。
  Action: 无需更多工具调用
  Observation: N/A

Step 4: Answer（最终回复）
  → 根据 Observation 给出最终答案
  济南明天天气不错！多云转晴，温度24-30度，出门记得带伞哦~

---

ReAct vs 纯 LLM：
- 纯 LLM：直接回答，可能编造错误信息
- ReAct：先搜索 → 再回答，事实性更强
- 适用：实时信息查询、文件操作、多步骤任务
```

---

## 4️⃣ 速查清单

### 4-1 Spring AI 常用注解速查

```java
// ===== 核心注解 =====

// @Tool：标记为 AI 可调用工具
@Tool(name = "search", description = "搜索网络")
public String search(@UserParam("搜索关键词") String query) { }

// @UserParam：工具方法参数描述（AI 看到参数含义）
@Tool(description = "读取文件")
public String read(@UserParam(description = "文件路径") String path) { }

// ===== ChatMemory =====
// ChatMemory 接口方法
List<Message> memory.get(String chatId);      // 获取会话历史
void memory.add(String chatId, Message... msgs);  // 添加消息
void memory.clear(String chatId);             // 清除会话

// ===== ChatClient 链式调用 =====
chatClient.prompt()
    .system("系统提示词")     // 系统提示
    .user("用户消息")        // 用户消息
    .messages(List<>)        // 直接传 Message 列表
    .tools(callbacks)        // 注册工具
    .advisors(advisors)      // 注册 Advisor
    .call().content()        // 非流式返回 String
    .stream().content()       // 流式返回 Flux<String>
    .chatResponse()          // 返回 ChatResponse 对象

// ===== Document =====
// Document 常用方法
doc.getText();                    // 获取文本内容
doc.getMetadata().get("source"); // 获取元数据

// ===== VectorStore =====
// VectorStore 常用方法
vectorStore.add(documents);                              // 添加文档
vectorStore.similaritySearch(SearchRequest.query("问题").withTopK(3));  // 检索
vectorStore.delete(List.of(id));                         // 删除
```

### 4-2 你项目的 API 速查

```
chen-ai-agent 实际接口（参考 LoveAppController.java）：

GET  /api/ai/love/chat?message=xxx&chatId=default     → AI 对话（非流式）
GET  /api/ai/love/chat-stream?message=xxx&chatId=def  → AI 对话（流式/SSE）
GET  /api/ai/love/rag?message=xxx                     → RAG 增强对话
GET  /api/ai/love/report?chatId=default               → 生成恋爱报告
GET  /api/ai/tools?message=xxx                       → 工具调用对话

响应格式（统一）：
{
  "code": 200,
  "message": "success",
  "data": "AI 回复内容",
  "timestamp": 1723804800000
}
```

### 4-3 模型参数速查

| 参数 | 含义 | 推荐值 | 说明 |
|------|------|--------|------|
| `temperature` | 随机性 | 0.7（对话）/ 0.2（编程）/ 0.9（创意） | 越低越确定性，越高越随机 |
| `max_tokens` | 最大输出 token | 2048（对话）/ 4096（报告） | 控制回复长度上限 |
| `top_p` | 采样策略 | 0.9 | 与 temperature 配合 |
| `frequency_penalty` | 重复惩罚 | 0.5 | 降低重复输出 |
| `presence_penalty` | 新话题惩罚 | 0.5 | 鼓励引入新话题 |

### 4-4 RAG 分块策略速查

| 策略 | 块大小 | 重叠 | 适用场景 |
|------|--------|------|---------|
| 固定字符数 | 500-1000 字符 | 50-100 | 简单通用 |
| 固定 token | 500-1000 tokens | 50-100 | 精确控制 |
| 按段落 | 自然段落边界 | 0 | 结构清晰的文档 |
| 按句子 | 完整句子 | 0 | 短句重要（如问答） |
| 语义分块 | 语义相似段落 | — | 最优但复杂 |

---

## 5️⃣ 场景选择器

### 5-1 该选哪个 LLM 模型？

```
需要什么样的模型？
         │
    ┌────┴────────────────────────────────────────┐
    │                                             │
  中文为主               英文为主 / 需要最强模型
  追求性价比               │
    │                     ↓
    ↓                最高质量？
  ┌─┴──┐                  │
  │是  │否                ↓
  │    ↓              OpenAI GPT-4o
  │    ↓              或 Claude 3.5 Sonnet
  │  多语言？
  │    ↓
  │  DeepSeek
  │  deepseek-chat
  │    ↓
  │  OpenAI
  │  gpt-4o-mini
  └──────┘

  ★ 你项目主力：qwen-plus（中文好，便宜）
  ★ 备用：deepseek-chat（性价比极高）
```

### 5-2 该选哪个向量数据库？

```
团队规模和场景？
         │
    ┌────┴────────────────────────────────────────┐
    │                                             │
  小团队 / 个人开发         企业 / 大规模数据
  / 本地测试               / 需要高可用
    │                        │
    ↓                        ↓
  Chroma（Docker 一行启动）   PgVector（基于已有 PG）
  或 InMemory（测试用）      或 Milvus（TB级）
```

### 5-3 RAG 还是长上下文？

```
知识库有多大？
         │
    ┌────┴────────────────────────────┐
    │                                   │
  < 10 万字               > 10 万字
  (< 100 tokens × 1000块)    │
    │                        ↓
    ↓                   多跳问答？
  长上下文（整个丢给 LLM）      │（需要关联多个文档）
  或简单 RAG                  ↓
                               是 → Agentic RAG
                               否 → 标准 RAG（topK 调大）
```

---

## ❓ 常见面试题

**Q1: RAG 的核心原理是什么？解决什么问题？**
> RAG = Retrieval（检索）+ Augmented（增强）+ Generation（生成）。核心原理：先把知识库文档切块+向量化存到向量库，用户提问时将问题也向量化，检索最相关的块，再把这些块作为上下文注入 Prompt，让 LLM 基于真实文档内容回答。解决的问题：LLM 幻觉（编造事实）、知识陈旧、无法回答私域知识（公司内部文档/个人数据）。

**Q2: Spring AI 的 ChatClient 和传统 RestTemplate 调 API 有什么区别？**
> RestTemplate 需要自己拼接 JSON、处理响应、流式响应需要手动实现。ChatClient 统一了所有模型的调用接口，自动处理 Prompt 构造、Response 解析、流式响应（Flux<String>）、ChatMemory 记忆管理、Tool Calling，降低了学习成本，一个代码可以切换不同模型。

**Q3: 向量数据库是如何实现相似度检索的？**
> 通过 Embedding 模型把文本转为高维向量（1536维等），语义相近的文本向量在高维空间中距离更近。检索时将查询文本同样向量化，然后在向量库中用余弦相似度（Cosine）或欧氏距离（Euclidean）计算相似度，返回距离最近的 topK 条。余弦相似度 = cos(θ)，越接近 1 表示越相似。

**Q4: 什么是 Embedding？它和 LLM 的区别是什么？**
> Embedding 是将文字转为固定长度的高维向量（数字数组），用于计算语义相似度（检索场景）。LLM（大语言模型）是生成式模型，用于生成文本（对话/写作）。Embedding 通常比 LLM 小得多，速度更快，专门训练用于语义匹配。DashScope text-embedding-v3 是阿里自研的 Embedding 模型。

**Q5: 什么是 Function Calling？解决了什么问题？**
> Function Calling 让 LLM 能够主动调用外部工具（Java 方法），实现：实时查询（搜索/查天气/数据库）、文件操作、API 调用、代码执行等。解决了 LLM 不知道实时信息、无法执行操作的问题。比如问"济南天气"，AI 可以调用 search 工具查真实天气，而不是编造答案。

**Q6: 什么是 ReAct 模式？**
> ReAct = Reasoning + Acting。核心思想：让 AI 循环执行"思考→行动→观察"三步，直到任务完成。好处：AI 的推理过程可追溯（不是黑盒），复杂任务拆解成多步完成，适合多工具协作场景。实现方式是让 AI 输出特定格式的文本（Thought/Action/Observation），Spring AI 的 `ReActAgent` 模板支持这个模式。

**Q7: Spring AI 的 Advisor 是什么？**
> Advisor 是 Spring AI 2.x 的拦截器链（类似 Spring MVC 的 Interceptor）。可以拦截每次 AI 调用，注入通用逻辑：日志记录（`MyLoggerAdvisor`）、记忆管理（`MessageChatMemoryAdvisor`）、RAG 检索（`ragAdvisor`）、限流/鉴权等。`@ReadUntilToolCalls` 注解可以让 Advisor 在 AI 调用工具前后执行自定义逻辑。

## 📊 学习状态

- [x] Spring AI 2.x 架构与 ChatClient
- [x] DashScope / DeepSeek 多模型接入
- [x] ChatMemory（InMemory / FileBased / Redis）
- [x] SSE 流式响应
- [x] RAG 全流程（文档加载→分块→Embedding→向量库→检索→增强）
- [x] Function Calling / @Tool
- [x] Prompt 工程（System Prompt / Few-Shot / 动态 Prompt）
- [x] Embedding 模型配置
- [x] Agent 架构（ReAct 模式）
- [x] chen-ai-agent 整体架构
- [ ] MCP 协议（Model Context Protocol）
- [ ] Multi-Agent 协作
- [ ] Agentic RAG（自主决策的 RAG）

## 🐛 踩坑记录

- **API Key 泄漏**：`application-local.yml` 已加入 `.gitignore`，但别直接写到代码里；用环境变量 `${AI_DASHSCOPE_API_KEY}`
- **FileBasedChatMemory 并发写入**：同一 chatId 并发写入会冲突，项目玩具阶段够用，生产用 Redis ChatMemory
- **向量库启动**：Chroma 需要先 `docker run -d -p 8000:8000 chromadb/chroma`；PgVector 需要 PostgreSQL + `CREATE EXTENSION vector`
- **Embedding 模型选错**：中文场景用 `text-embedding-v3`（阿里）或 `bge-large-zh`（开源），不要用英文 Embedding 模型
- **RAG 检索结果为空**：检查向量库是否真的写入了（重启后 InMemoryVectorStore 数据会丢）；检查 Embedding 模型配置是否正确
- **Function Calling 调用失败**：检查 @Tool 方法是否 public、参数是否有 @UserParam 注解、方法是否在 Spring Bean 中
- **SSE 跨域**：如果前端和后端端口不同，需要配置 CORS；`@CrossOrigin` 加在 Controller 或配置 `WebMvcConfigurer`
- **Token 超限**：ChatMemory 要设 `maxMessages`，否则对话历史越来越长，超过 LLM 的 Context Window 会报错
- **System Prompt 膨胀**：`MyLoggerAdvisor` 的 `adviseCall` 中如果每次都加新内容到 Prompt 里，会导致 Prompt 无限增长（你项目之前修过这个问题），应该在外部维护历史，Prompt 里只放固定系统提示词
- **模型切换**：你项目同时支持 DashScope 和 DeepSeek，注意 `application.yml` 中只有一个 `spring.ai.dashscope` 是生效的，切换时要改配置
