# 开发顺序

按依赖从底层到上层实施。每阶段结束后，项目保持可运行并通过相应验证。

## 1. 项目基线

固定 Python 版本、依赖管理、格式化/静态检查工具；建立 `src/`、`tests/`、`docs/`、`.env.example` 和 Streamlit 入口；实现集中配置并校验 OpenAI、Ollama、Xinference 设置，Embedding 默认 OpenAI。

检查点：空应用可启动，缺失或错误配置能明确报错，密钥不进入日志。

## 2. 领域模型和协议

定义 KnowledgeBase、Document、Chunk、RetrievedChunk、ChatMessage；定义 provider、事件、状态和错误类型；定义 Chat、Embedding、VectorStore、MetadataStore、DocumentLoader 协议。

检查点：核心协议可由 fake 实现，单元测试无需外部服务。

## 3. 元数据与 Chroma 基础设施

实现 Chroma client 生命周期、受控 collection 命名、知识库/文档元数据存储、向量写入、metadata filter、相似度检索和删除。

检查点：临时 Chroma 测试覆盖单库、多库检索隔离、删除和元数据一致性。

## 4. 模型适配层

实现统一 Chat/Embedding Gateway；分别适配 OpenAI、Ollama、Xinference；支持 Chat 与 Embedding 独立配置，默认 Embedding provider 为 OpenAI。

检查点：fake 实现通过统一协议；各平台配置映射有测试；真实服务仅做单独冒烟测试。

## 5. Markdown 入库流水线

实现文件校验和读取；实现按标题/段落切分及 chunk 长度/overlap；实现内容哈希去重、Embedding 批处理、Chroma 写入和入库报告。

检查点：重复上传幂等，失败项可定位，来源元数据完整。

## 6. LangGraph Agent

定义最小 state 和提示词约束；实现只接收知识库 ID 的 `retrieve_knowledge` 工具；实现检索、回答、错误节点和图构建；映射框架事件为稳定 `ChatEvent`。

检查点：离线 fake 模型完成工具调用与流式回答，事件顺序和失败状态可验证。

## 7. 应用服务

实现知识库创建/列表/删除、Markdown 入库和带知识库范围的对话用例；统一输入校验、错误映射和日志关联 ID。

检查点：集成测试可脱离 Streamlit 完成两条核心业务流程。

## 8. Streamlit 页面

实现知识库管理页的新建、选择、上传、处理状态和结果；实现对话页多知识库选择、消息历史、流式回答、工具调用轨迹、来源和错误组件；统一 Session State。

检查点：端到端流程覆盖建库、上传 Markdown、选择多个库并完成回答。

## 9. 质量与运行准备

补齐三层测试；运行格式化、静态检查、类型检查和依赖安全检查；完善运行说明；明确本地 Chroma、上传文件和 secrets 的部署管理方式。

## 优先验证的风险

先验证不同平台的 Chat/Embedding 接口差异、多知识库检索隔离、LangGraph 流式事件到 Streamlit 的映射。这三项决定底层抽象是否稳定。
