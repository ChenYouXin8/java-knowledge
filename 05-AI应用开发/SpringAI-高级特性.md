---
tags:
  - AI
  - SpringAI
  - FunctionCalling
  - Prompt
created: 2026-08-17
---

# Spring AI — 高级特性

> 从 [[Spring AI 与 RAG 实战]] 拆分而来。

---

## 代码模板

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

## 🔗 相关笔记

- [[Spring AI 与 RAG 实战]]
- [[../04-Spring生态/SpringBoot框架|Spring Boot 框架]]
- [[../03-Database/MySQL数据库|MySQL 数据库]]
