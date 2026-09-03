# Tasks

- [x] Task 1: 编写 Retain 管道执行时序文档
  - [x] 梳理入口层：MemoryEngine.retain_batch_async → _retain_batch_async_internal → orchestrator.retain_batch
  - [x] 梳理阶段 1（LLM 叙事事实抽取）：fact_extraction 模块的 chunk → extract → prompt 构建 → LLM 调用 → 结果解析
  - [x] 梳理阶段 2（嵌入向量生成）：embedding_processing 模块的 augment_texts_with_dates → generate_embeddings_batch
  - [x] 梳理阶段 3（实体解析与归一化）：entity_processing.resolve_entities → entity_resolver.resolve_entities_batch → 评分逻辑
  - [x] 梳理阶段 4（图链接构建）：temporal_links → semantic_links (ANN + within-batch) → causal_links
  - [x] 梳理阶段 5（观点自动强化）：在代码库中确认当前实现状态
  - [x] 梳理阶段 6（背景合并）：在代码库中确认当前实现状态
  - [x] 生成最终文档 `docs/retain-pipeline-trace.md`，包含完整的执行时序图

# Task Dependencies
- 所有子任务无依赖关系，可并行执行