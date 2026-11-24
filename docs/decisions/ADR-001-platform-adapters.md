# ADR-001：通过统一 Gateway 适配模型平台

## 状态

Accepted

## 日期

2026-11-24

## 背景

系统需要支持 OpenAI、Ollama 和 Xinference，并默认使用 OpenAI Embedding。若在 Agent、应用服务或页面直接判断平台，配置、SDK 参数和错误处理会扩散到多个模块。

## 决策

在 `src/infrastructure/llm/` 建立统一 Chat Model Gateway 和 Embedding Gateway。应用层和 Agent 层只依赖抽象协议；适配器负责 SDK 初始化、地址、认证、模型名映射及平台错误转换。

Chat provider 和 Embedding provider 独立配置。Embedding 默认 provider 为 OpenAI，允许配置显式覆盖。

## 取舍

- 业务层不绑定平台 SDK，切换平台可控，fake 实现易于测试。
- 需要维护统一能力边界，避免把平台特有能力泄漏到业务接口。
- 首期约定通用聊天、Embedding 和流式 token 能力；平台特有多模态或高级参数另行评估。

## 影响

新增平台需要增加适配器、配置映射、测试和文档。不得在页面或 Agent 节点中直接增加平台分支。
