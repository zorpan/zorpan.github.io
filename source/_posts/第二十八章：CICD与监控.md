---
title: CI/CD 流水线与监控告警体系
date: 2026-06-21 12:00:00
tags:
  - CI/CD
  - 监控
  - Prometheus
  - 面试
categories:
  - Java 进阶之路
---

> 本文是「Java 进阶之路」系列的一部分，聚焦实战与面试。

<!-- more -->

## 28.1 CI/CD 流水线

```yaml
# GitHub Actions 示例
name: CI/CD Pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - run: mvn clean test
      - run: mvn package -DskipTests
      - run: docker build -t myapp:${{ github.sha }} .
      - run: docker push myapp:${{ github.sha }}
```

**CI/CD 流程：**
```
代码提交 → 代码检查 → 单元测试 → 构建 → 镜像推送 → 部署到测试环境 → 集成测试 → 部署到生产环境
```

---

## 28.2 日志体系（ELK Stack）

```
架构：
应用 → Filebeat（采集）→ Kafka（缓冲）→ Logstash（解析）→ Elasticsearch（存储）→ Kibana（展示）

日志规范：
- 统一格式：时间戳 + 级别 + 类名 + 方法 + 消息 + TraceId
- 结构化日志：JSON 格式，便于解析
- 日志级别：ERROR > WARN > INFO > DEBUG
- 生产环境只开 INFO，排查问题时临时开 DEBUG
```

```java
// MDC 传递 TraceId
MDC.put("traceId", TraceUtils.generateTraceId());
log.info("用户登录成功, userId={}", userId);
```

---

## 28.3 监控告警（Prometheus + Grafana）

```
架构：
应用（暴露 /actuator/prometheus 端点）
    → Prometheus（拉取指标数据）
    → Grafana（可视化展示）
    → AlertManager（告警通知）

核心指标（RED 方法）：
- Rate：请求速率（QPS）
- Errors：错误率
- Duration：响应时间（P50/P95/P99）
```

---

## 28.4 APM 与分布式链路追踪

```
SkyWalking 架构：
应用 Agent → OAP Server → Elasticsearch → UI

核心概念：
- Trace：一次完整请求的链路
- Span：链路中的一个操作
- Service Mesh：服务网格（Istio/Linkerd）
```

---

## 面试精选

### 1. CI/CD 的流程？

代码提交触发流水线：代码检查（Lint）→ 单元测试 → 构建打包 → Docker 镜像 → 部署测试环境 → 集成测试 → 部署生产环境。GitHub Actions / Jenkins 实现自动化。

### 2. 怎么做线上问题排查？

1）通过 TraceId 在 ELK 中查找完整日志链路。2）通过 SkyWalking 查看链路拓扑和各节点耗时。3）通过 Prometheus/Grafana 查看 QPS、错误率、响应时间趋势。4）通过 Arthas 在线诊断（watch/trace/stack 命令）。

### 3. Prometheus 的工作原理？

Pull 模型：Prometheus 定期从应用的 /metrics 端点拉取指标数据，存储在时序数据库中。PromQL 查询语言做聚合分析。AlertManager 配置告警规则，触发后通过邮件/钉钉/Slack 通知。
