---
tags:
  - AI
  - SpringAI
  - 面试
  - 速查
created: 2026-08-17
---

# SpringAI与RAG实战 — 速查与面试题

> 从 [[SpringAI与RAG实战]] 拆分而来，包含对比表格、图解、速查清单、面试题和踩坑记录。

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

---

## 🔗 相关笔记

- [[SpringAI与RAG实战]]
