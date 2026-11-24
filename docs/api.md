# API 与接口契约

首期入口为 Streamlit，因此本文件定义应用层和 Agent 层的内部 API，保证 UI 可替换、逻辑可测试。

## 核心数据对象

### KnowledgeBase

```text
id: str
name: str
description: str | None
collection_name: str
created_at: datetime
updated_at: datetime
```

### Document

```text
id: str
knowledge_base_id: str
source_name: str
source_hash: str
mime_type: str
chunk_count: int
created_at: datetime
```

### RetrievedChunk

```text
chunk_id: str
document_id: str
knowledge_base_id: str
content: str
score: float | None
metadata: dict[str, str | int | float]
```

## 应用服务

### KnowledgeBaseService

```python
create_knowledge_base(name: str, description: str | None) -> KnowledgeBase
list_knowledge_bases() -> list[KnowledgeBase]
get_knowledge_base(knowledge_base_id: str) -> KnowledgeBase
delete_knowledge_base(knowledge_base_id: str) -> None
```

名称不可为空。ID 和 collection 名称由服务生成。删除时同步处理向量 collection 和元数据。

### IngestionService

```python
ingest_markdown(
    knowledge_base_id: str,
    filename: str,
    content: bytes,
) -> IngestionReport
```

`IngestionReport` 至少包含 `document_id`、`chunk_count`、`embedded_count`、`skipped` 和 `errors`。只接收 Markdown；校验扩展名、大小、编码和空内容；内容哈希用于幂等；失败不能静默报告成功。

### ChatService

```python
stream_chat(
    question: str,
    enabled_knowledge_base_ids: list[str],
    history: list[ChatMessage],
) -> Iterator[ChatEvent]
```

问题不能为空；启用的知识库 ID 必须校验存在；空知识库集合必须返回明确的可识别状态。事件顺序应能被 UI 直接消费。

## LangGraph 检索工具

工具名：`retrieve_knowledge`

输入：

```json
{
  "query": "用户问题",
  "knowledge_base_ids": ["kb_a", "kb_b"],
  "top_k": 5
}
```

输出：

```json
{
  "query": "用户问题",
  "chunks": [
    {
      "chunk_id": "chunk_1",
      "document_id": "doc_1",
      "knowledge_base_id": "kb_a",
      "content": "检索片段",
      "score": 0.82,
      "metadata": {"source_name": "faq.md", "chunk_index": 3}
    }
  ],
  "hit_count": 1
}
```

工具不接受未经校验的 collection 名称；只能通过知识库 ID 获取 collection。

## 流式事件

每个事件含 `event_id`、`event_type`、`timestamp`、`run_id`。事件类型如下：

| 类型 | 负载字段 | 用途 |
|---|---|---|
| `tool_start` | `tool_name`, `input` | 展示工具开始和检索范围 |
| `tool_result` | `tool_name`, `hit_count`, `chunks` | 展示检索完成及来源 |
| `token` | `text`, `node` | 增量渲染助手回答 |
| `source` | `document_id`, `source_name`, `knowledge_base_id` | 关联引用来源 |
| `error` | `code`, `message`, `retryable` | 展示可识别错误 |
| `done` | `answer`, `sources` | 标识本轮结束 |

UI 可以折叠工具事件，但应将它们与回答文本分开显示。

Agent adapter 将 LangGraph `messages` stream 用于回答 token，将 `updates` 用于节点/工具阶段状态，将 `custom` 用于需要显式报告的检索进度，并统一转换成上述事件。UI 不消费原始 LangGraph payload。选定的 Chat 模型必须支持工具调用；应用启动或模型配置变更时应验证该能力，不支持时给出明确错误，避免静默退化成不检索的普通聊天。

## 模型 Gateway

应用层依赖以下能力：

```python
create_chat_model(provider: ProviderName, model_name: str, **options) -> ChatModel
create_embedding_model(provider: ProviderName = "openai", model_name: str | None = None) -> EmbeddingModel
```

合法 provider 为 `openai`、`ollama`、`xinference`。Chat 与 Embedding provider 独立配置；平台地址、认证、SDK 参数和错误转换由适配器处理。

Embedding provider/model 配置应与知识库 collection 一同记录。更换 Embedding 模型时不得把新旧向量写入同一 collection；通过新 collection 重建并完成切换。
