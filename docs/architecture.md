# 系统架构

## 分层关系

```text
Streamlit UI
  -> Application Services
    -> LangGraph Agent Runtime
      -> Retrieval Tool -> Knowledge Base Service -> Chroma Adapter
      -> Model Gateway -> OpenAI / Ollama / Xinference
    -> Ingestion Service
      -> Markdown Loader -> Chunker -> Embedding Gateway -> Chroma Adapter
```

依赖向内收敛：UI 依赖应用服务；应用服务依赖领域协议；基础设施实现协议。领域层不依赖 Streamlit、LangGraph、Chroma 或特定模型 SDK。

## 推荐项目目录

```text
rag-customer/
├── AGENTS.md
├── README.md
├── pyproject.toml
├── .env.example
├── .gitignore
├── app.py                         # Streamlit st.Page/st.navigation 入口
├── src/
│   ├── config/
│   │   ├── settings.py            # 环境变量、secrets、默认参数
│   │   └── providers.py           # Provider 配置校验与解析
│   ├── domain/
│   │   ├── models.py              # KnowledgeBase、Document、Chunk、ChatMessage
│   │   ├── enums.py               # Provider、状态和事件类型
│   │   ├── protocols.py           # Model、Embedding、VectorStore 等接口
│   │   └── errors.py              # 领域和应用错误
│   ├── application/
│   │   ├── knowledge_base_service.py
│   │   ├── ingestion_service.py
│   │   ├── chat_service.py
│   │   ├── dto.py                 # 用例输入输出对象
│   │   └── events.py              # UI 可消费的应用事件
│   ├── agent/
│   │   ├── state.py               # LangGraph State
│   │   ├── graph.py               # 图构建与编译
│   │   ├── nodes.py               # 检索、回答和错误节点
│   │   ├── tools.py               # 多知识库检索工具
│   │   └── prompts.py             # 系统提示词和回答约束
│   ├── infrastructure/
│   │   ├── chroma/
│   │   │   ├── client.py          # Chroma 生命周期
│   │   │   └── repository.py      # collection、写入、查询、删除
│   │   ├── llm/
│   │   │   ├── gateway.py         # Chat/Embedding 工厂
│   │   │   ├── openai_adapter.py
│   │   │   ├── ollama_adapter.py
│   │   │   └── xinference_adapter.py
│   │   ├── ingestion/
│   │   │   ├── markdown_loader.py
│   │   │   └── chunker.py
│   │   └── persistence/
│   │       └── sqlite_metadata_store.py # 知识库和文档元数据
│   └── ui/
│       ├── pages/
│       │   ├── knowledge_base.py  # 知识库管理页面
│       │   └── chat.py            # 智能客服对话页面
│       ├── components/
│       │   ├── tool_trace.py
│       │   ├── source_list.py
│       │   └── upload_status.py
│       └── session.py
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── docs/
│   ├── README.md
│   ├── project-conventions.md
│   ├── architecture.md
│   ├── api.md
│   ├── testing.md
│   ├── development-plan.md
│   └── decisions/
└── data/                           # 运行时目录，不提交数据
    ├── chroma/
    └── uploads/
```

## 模块职责

| 模块 | 负责 | 不负责 |
|---|---|---|
| `config` | 配置读取、默认值、Provider 和参数校验 | 业务流程和 UI 状态 |
| `domain` | 领域模型、协议、错误和纯业务规则 | SDK 调用、Streamlit 控件 |
| `application` | 创建/删除知识库、入库、对话等用例编排 | 具体模型和 Chroma 实现 |
| `agent` | LangGraph state、节点、检索工具、流式事件映射 | 页面布局和文件控件 |
| `infrastructure/llm` | 三个平台的 Chat/Embedding 适配 | 业务提示词和页面行为 |
| `infrastructure/chroma` | collection 生命周期、向量写入和检索 | 决定启用哪些知识库 |
| `infrastructure/ingestion` | Markdown 读取、切分、chunk 元数据 | 页面反馈 |
| `infrastructure/persistence` | 知识库及文档元数据存储 | 向量相似度计算 |
| `ui/pages` | 表单、交互、消息和事件展示 | 直接访问基础设施 |
| `ui/components` | 工具轨迹、来源、上传状态等可复用展示 | 业务决策 |

## 核心数据流

### 文档入库

用户选择知识库并上传 Markdown → `IngestionService` 校验文件并计算哈希 → Loader 读取文本 → Chunker 按标题/段落切分 → Embedding Gateway 生成向量 → Chroma Repository 写入向量与来源元数据 → 返回文档数、chunk 数、跳过数和错误明细。

### 对话检索

页面提交问题和启用的知识库 ID → `ChatService` 初始化 LangGraph state → Agent 调用 `retrieve_knowledge` → 工具按知识库 ID 映射到受控 collections 并检索 → 以事件流返回工具过程 → 回答节点基于检索上下文生成 token → 页面展示回答、来源和完成/错误状态。

## 关键架构选择

- 多知识库通过 collection 隔离，并在检索请求中严格过滤已启用知识库。
- Agent 访问向量数据的唯一边界是检索工具。
- 模型平台差异集中在 Gateway，业务层依赖统一协议。
- UI 消费应用事件，不直接依赖 LangGraph 内部节点事件格式。
- 推荐以 SQLite 保存知识库和文档目录元数据，以 Chroma 保存 chunk 向量；通过 `MetadataStore` 协议隔离存储实现。此选择适合单机 V1，部署到多实例时再评估共享关系数据库。
- 每个 Chroma collection 绑定一个 Embedding provider/model 配置。更换模型时创建新 collection 并重建索引，避免不同向量空间混用。

## 框架落地参考

- Streamlit 入口使用 `st.Page` 和 `st.navigation` 注册页面；在入口集中初始化稳定的 widget keys 和 Session State。Session State 用于当前浏览器会话，不等同于跨会话的对话持久化。
- LangGraph 将流程表达为共享 state、nodes 和 edges；通过 `messages`、`updates`、`custom` 等 stream mode 提取模型 token、节点更新和自定义工具进度，再映射为本项目的 `ChatEvent`。
- Chroma 写入记录使用稳定唯一 ID，并保留文档和来源元数据。

官方参考：[Streamlit 多页应用](https://docs.streamlit.io/develop/concepts/multipage-apps/page-and-navigation)、[Streamlit Session State](https://docs.streamlit.io/develop/concepts/architecture/session-state)、[LangGraph Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)、[LangGraph Streaming](https://docs.langchain.com/oss/python/langgraph/streaming)、[Chroma 写入数据](https://docs.trychroma.com/docs/collections/add-data)。
