# 测试规范

## 测试目标

验证文档入库幂等、知识库检索隔离、Agent 工具调用、流式事件协议，以及 OpenAI、Ollama、Xinference 适配器遵循统一接口。

## 测试分层

### 单元测试

- Markdown 清洗、标题/段落切分、chunk 长度与 overlap。
- 文件扩展名、编码、大小、空文件和重复哈希校验。
- collection 名称生成、知识库 ID 校验和 provider 配置解析。
- 默认 OpenAI Embedding 行为。
- LangGraph state 更新、事件映射、错误转换。
- 多库结果合并、排序、去重和来源元数据保留。

单元测试不访问真实模型服务或持久化 Chroma。

### 集成测试

使用临时 Chroma 和 fake 模型验证：

- 创建知识库、入库 Markdown 后可检索到预期片段。
- 同时启用多个库时合并结果；只启用一个库时不能命中其他库。
- 重复上传同一内容保持幂等。
- 删除知识库后 collection 和元数据均不可用。
- Agent 工具流程产生有序的 `tool_start`、`tool_result`、`token`、`done` 事件。

### 端到端测试

在隔离配置下运行 Streamlit 并验证：

- 创建知识库、上传 Markdown 后看到处理统计。
- 对话页可以选择多个知识库并完成检索问答。
- 工具轨迹、流式回答、来源及错误状态均可见。
- 未选择知识库、文件格式错误、模型不可用时显示清楚提示。

真实模型平台冒烟测试单独运行，不作为离线测试的前置条件。

## 测试替身

- `FakeEmbeddingModel`：对固定输入返回可预测向量。
- `FakeChatModel`：产生固定 token，并可模拟工具调用。
- `InMemoryMetadataStore`：隔离外部元数据存储。
- 临时 Chroma client：每个测试使用独立 collection，结束后清理。

## 验收标准

- 所有单元测试通过。
- 集成测试覆盖多知识库隔离和重复入库。
- 事件协议测试覆盖正常顺序和错误路径。
- 不以删除测试、跳过断言或降低已有阈值作为通过手段。
- 覆盖率目标在首轮实现后结合复杂度设定；优先保证核心流程和错误分支有行为验证。

## 建议命令

以最终工具链为准，建议提供以下入口：

```text
pytest -q
pytest tests/unit -q
pytest tests/integration -q
pytest tests/e2e -q
ruff check .
ruff format --check .
mypy src
```

## 测试数据

- 使用短小、固定且可读的 Markdown 样例。
- 不提交客户数据、真实密钥、生产模型响应或完整生产文档。
- 检索断言关注来源 ID、知识库 ID 和关键内容，不依赖浮点分数精确相等。
