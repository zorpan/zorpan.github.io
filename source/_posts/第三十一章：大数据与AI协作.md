---
title: 大数据生态与 AI 协作实践
date: 2026-06-22 14:00:00
tags:
  - 大数据
  - Flink
  - AI
  - RAG
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

## 31.1 Java 与大数据生态

```
Hadoop 生态：
├── HDFS：分布式文件存储
├── MapReduce：批处理计算框架
├── YARN：资源调度
└── Hive：SQL 查询引擎

Spark 生态：
├── Spark Core：内存计算引擎
├── Spark SQL：结构化数据处理
├── Spark Streaming：微批流处理
├── Spark MLlib：机器学习库
└── Structured Streaming：统一批流处理

Flink 生态（实时计算首选）：
├── Flink DataStream：流处理 API
├── Flink SQL：SQL 流处理
├── Flink CEP：复杂事件处理
├── Flink State：状态管理
└── Flink CDC：数据变更捕获
```

| 框架 | 延迟 | 吞吐 | 语义 | 场景 |
|------|------|------|------|------|
| MapReduce | 分钟级 | 高 | Exactly-Once | 离线批处理 |
| Spark | 秒级 | 高 | Exactly-Once | 批处理+微批流 |
| Flink | 毫秒级 | 高 | Exactly-Once | 实时流处理 |

---

## 31.2 Java 接入 AI 能力

### 31.2.1 Spring AI

```java
// Spring AI 接入大模型
@RestController
public class AIController {
    private final ChatClient chatClient;

    public AIController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        return chatClient.prompt()
            .user(message)
            .call()
            .content();
    }

    // RAG（检索增强生成）
    @GetMapping("/ask")
    public String ask(@RequestParam String question) {
        return chatClient.prompt()
            .user(question)
            .advisors(new QuestionAnswerAdvisor(vectorStore))
            .call()
            .content();
    }
}
```

### 31.2.2 LangChain4j

```java
// LangChain4j 接入 AI
interface Assistant {
    @SystemMessage("你是一个 Java 技术专家")
    @UserMessage("{{message}}")
    String chat(@V("message") String message);
}

Assistant assistant = AiServices.create(Assistant.class, model);
String answer = assistant.chat("什么是虚拟线程？");

// RAG 流程
// 1. 文档加载 → 分块 → 向量化 → 存入向量数据库
// 2. 用户提问 → 向量搜索相关文档 → 拼接上下文 → 调用大模型
```

---

## 31.3 向量数据库与 RAG

```
RAG（检索增强生成）流程：
1. 离线阶段：文档加载 → 文本分块 → Embedding 向量化 → 存入向量数据库
2. 在线阶段：用户提问 → Embedding → 向量相似度搜索 → 检索相关文档
             → 拼接 Prompt（上下文 + 问题）→ 调用大模型 → 生成回答

向量数据库：
├── Milvus：开源，高性能，适合大规模
├── Pinecone：云服务，易用
├── Weaviate：开源，支持多模态
└── Redis Vector：Redis 模块，轻量级
```

```java
// Spring AI + 向量数据库
@Bean
VectorStore vectorStore(EmbeddingModel embeddingModel) {
    return new SimpleVectorStore(embeddingModel);  // 或 MilvusVectorStore
}

// 文档导入
vectorStore.add(List.of(
    new Document("Java 虚拟线程是 JDK 21 引入的轻量级线程..."),
    new Document("G1 垃圾收集器是 JDK 9 的默认收集器...")
));

// 检索
List<Document> results = vectorStore.similaritySearch(
    SearchRequest.query("什么是虚拟线程").withTopK(5));
```

---

## 面试精选

### 1. Flink 和 Spark Streaming 的区别？

Flink 是真正的流处理（事件驱动，毫秒级延迟），Spark Streaming 是微批处理（将流切分为小批次，秒级延迟）。Flink 支持事件时间语义和精确的窗口机制，适合实时计算场景。Spark 在批处理方面更强。

### 2. 什么是 RAG？

检索增强生成（Retrieval-Augmented Generation）：先从知识库中检索与问题相关的文档，再将检索结果作为上下文传递给大模型生成答案。解决了大模型知识过时和幻觉问题。

### 3. Java 怎么接入 AI 能力？

Spring AI / LangChain4j 框架，对接 OpenAI、Ollama 等模型服务。支持 RAG（向量数据库检索+大模型生成）、Function Calling（让 AI 调用外部工具）、Streaming 流式输出。
