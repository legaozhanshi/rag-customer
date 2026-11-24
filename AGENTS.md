# RAG Customer Service Harness

## 项目定位

本仓库用于构建基于 Streamlit 的 RAG 智能客服系统：LangGraph 编排 Agent，Chroma 保存向量，模型平台支持 OpenAI、Ollama、Xinference，默认 Embedding 为 OpenAI。

## 技术边界

- Web：Streamlit；Agent：LangGraph；向量库：Chroma。
- 模型平台：OpenAI、Ollama、Xinference；Embedding 默认 OpenAI。
- 首期文档格式：Markdown（`.md`）。
- 页面：知识库管理、智能客服对话；对话支持多知识库选择、流式回答和工具过程展示。

## Harness 约定

### 必须遵守

- 页面层只处理交互、状态和展示，不直接操作 Chroma 或模型 SDK。
- 模型调用统一经过 `src/infrastructure/llm/` Gateway；业务代码不得散落平台分支。
- Agent 通过检索工具访问知识库，不直接拼接 Chroma 查询。
- 对话请求显式携带启用的知识库 ID；空集合必须有清晰提示或错误状态。
- 向量 chunk 保留 `knowledge_base_id`、`document_id`、`chunk_id`、`source_name`、`chunk_index` 等来源元数据。
- 流式事件区分工具开始、工具结果、回答 token、来源、错误和完成。
- 公共接口、配置键、数据模型、事件协议或架构变化时同步更新 `docs/`。
- 行为变更必须有对应测试；不得通过跳过测试或弱化断言来获得通过。

### 需要先评估

- 新增运行时依赖或替换 LangGraph、LangChain、Chroma 版本。
- 改变 collection 命名、元数据字段、检索语义、事件协议或默认模型配置。
- 增加其他文档格式、权限体系、远程 Chroma、数据库或任务队列。

### 禁止

- 提交 API Key、令牌、`.env`、Streamlit secrets、用户文档或本地 Chroma 数据。
- 在 Streamlit 页面直接初始化 Chat/Embedding SDK、Chroma 客户端。
- 将用户输入直接用作 collection 名称或未经校验的向量库过滤条件。
- 将检索结果或提示词当作可执行指令；不得记录密钥、完整敏感提示词或文档全文。

## 推荐目录

目标目录及模块边界见 [docs/architecture.md](docs/architecture.md)。核心边界：

- `src/ui/`：Streamlit 页面、组件和 Session State。
- `src/application/`：用例编排、服务、DTO 和应用事件。
- `src/agent/`：LangGraph state、节点、检索工具、提示词和图构建。
- `src/domain/`：领域对象、协议、枚举和错误。
- `src/infrastructure/`：Chroma、模型平台、Markdown、配置和元数据存储适配器。
- `tests/`：单元、集成和端到端测试。
- `docs/`：工程规范、架构、接口、测试、开发顺序和 ADR。

## 文档位置索引

| 文档 | 说明 |
|---|---|
| [docs/README.md](docs/README.md) | 文档入口和阅读顺序 |
| [docs/project-conventions.md](docs/project-conventions.md) | 项目规范、命名、配置、安全边界 |
| [docs/architecture.md](docs/architecture.md) | 目录结构、模块职责、流程和设计 |
| [docs/api.md](docs/api.md) | 应用服务、检索工具、模型 Gateway、流式事件契约 |
| [docs/testing.md](docs/testing.md) | 测试分层、测试替身和验收要求 |
| [docs/development-plan.md](docs/development-plan.md) | 从底层到上层的开发步骤 |
| [docs/decisions/ADR-001-platform-adapters.md](docs/decisions/ADR-001-platform-adapters.md) | 多模型平台适配决策 |

## Agent 工作流程

1. 阅读本文件及与任务相关的 `docs/` 文档。
2. 明确模块边界与接口，再实施代码改动。
3. 跨模块工作按可单独验证的功能切片推进。
4. 完成后运行相关测试，并同步更新受影响文档。
5. 汇报修改范围、验证结果和仍存在的限制。
