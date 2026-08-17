---
tags:
  - AI
  - SpringAI
  - RAG
  - 记忆
created: 2026-08-17
---

# Spring AI — 记忆与 RAG

> 从 [[Spring AI 与 RAG 实战]] 拆分而来。

---

## 代码模板

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


---

## 🔗 相关笔记

- [[Spring AI 与 RAG 实战]]
- [[../04-Spring生态/SpringBoot框架|Spring Boot 框架]]
- [[../03-Database/MySQL数据库|MySQL 数据库]]
