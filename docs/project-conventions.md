# 项目规范

## 目标

保持 Streamlit、LangGraph、Chroma 和多模型平台之间的边界清晰，使替换模型、扩展检索策略或增加页面时不必重写业务逻辑。

## Python 规范

- 使用 Python 3.11 或项目最终确定的单一版本，并在运行环境中固定。
- 公共函数、协议、领域对象、LangGraph state 和事件字段添加类型注解。
- 模块和函数使用 `snake_case`，类使用 `PascalCase`，常量使用 `UPPER_SNAKE_CASE`。
- 核心数据通过数据类或 Pydantic 模型传递，避免跨模块使用无结构字典。
- 依赖方向保持单向：UI → application → domain/protocol；infrastructure 实现 domain 定义的协议。

## 配置和密钥

- 配置从环境变量或 Streamlit secrets 读取，仓库只保留非敏感默认值。
- 模型、Embedding、Chroma 路径、切分和检索参数集中配置，采用 `RAG_` 前缀。
- API Key、服务令牌和敏感 endpoint 不写入源码或日志。
- `.env`、secrets、上传目录、Chroma 持久化目录和缓存必须加入 `.gitignore`。

## 知识库和文档

- 知识库具有稳定逻辑 ID、展示名称、受控 collection 名称及时间戳。
- 使用内容哈希进行重复文档检测；重复上传不得重复写入向量。
- 每个 chunk 至少保留 `knowledge_base_id`、`document_id`、`chunk_id`、`source_name`、`source_hash`、`chunk_index`。
- 删除或重建知识库时同步处理元数据与 Chroma collection，避免孤儿数据。
- 首期只接收 Markdown；文件扩展名、大小、编码和空内容均需校验。

## Agent、检索和 UI

- 检索工具接收问题和知识库 ID 列表；collection 名称由受控映射得到。
- 工具结果保留来源信息；无结果时回答必须体现知识不足，不能伪造引用。
- 被选用的 Chat 模型必须支持工具调用；配置校验需尽早发现不支持的模型。
- 一个 Chroma collection 只包含同一 Embedding provider/model 产生的向量；切换模型需重新建索引。
- LangGraph state 只保存必要字段，明确区分对话、启用知识库、检索结果、事件和错误。
- 页面只调用应用服务；不能直接访问模型 SDK 或 Chroma。多页导航由入口通过 `st.Page` / `st.navigation` 组织。
- Session State 在入口集中初始化并使用稳定 widget keys；它管理当前 Streamlit 会话状态，不承担跨会话聊天持久化。页面应展示启用库、工具过程、回答、来源和错误状态。

## 错误、日志和变更

- 用户可恢复错误显示清楚的提示；内部错误结构化记录并携带关联 ID。
- 模型、Embedding、Chroma、校验错误使用可区分类型。
- 提交类型采用 `docs:`、`feat:`、`fix:`、`test:`、`chore:` 等约定。
- 公共接口、事件、配置或架构变化必须同步修改文档和测试。
- 不提交生成文件、缓存、密钥、客户数据和本地向量库。
