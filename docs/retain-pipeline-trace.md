# Hindsight Retain 处理时序流程追踪

> 基于最新代码库 `hindsight-api-slim` 梳理，更新日期：2026-08-03

## 总览

retain 处理管道分为 **6 个阶段**，其中前 4 个阶段在 retain 管道内同步执行，后 2 个阶段（观点强化、背景合并）由独立的 **consolidation** 后台作业异步执行。

```
输入文本 D
     │
     ▼
┌─────────────────────────────────────────────────┐
│ 阶段 1: LLM 叙事事实抽取 (retain 管道内)           │
│ 阶段 2: 嵌入向量生成 (retain 管道内)               │
│ 阶段 3: 实体解析与归一化 (retain 管道内)            │
│ 阶段 4: 图链接构建 (retain 管道内)                 │
└──────────────────┬──────────────────────────────┘
                   │ (retain 完成, 事务提交)
     ▼
┌─────────────────────────────────────────────────┐
│ 阶段 5: 观点自动强化 (consolidation 后台作业)      │
│ 阶段 6: 背景合并 (consolidation 后台作业)          │
└─────────────────────────────────────────────────┘
```

---

## 入口层：从 API 到 Orchestrator

### 调用链

```
MemoryEngine.retain_batch_async()          # memory_engine.py:3558
  └─ _retain_batch_async_internal()       # memory_engine.py:3994
       └─ orchestrator.retain_batch()     # retain/orchestrator.py:706
            ├─ _try_delta_retain()        # orchestrator.py:2241 (delta 增量模式)
            └─ _streaming_retain_batch()  # orchestrator.py:1304 (流式批处理)
                 ├─ _llm_producer()       # orchestrator.py:1467 (LLM 生产者)
                 └─ _db_consumer()        # orchestrator.py:1545 (DB 消费者)
                      └─ _process_db_batch()           # orchestrator.py:1603
                           ├─ _extract_and_embed()      # orchestrator.py:579  (阶段 1+2)
                           ├─ _pre_resolve_phase1()     # orchestrator.py:374  (阶段 3)
                           └─ _insert_facts_and_links() # orchestrator.py:476  (阶段 4)
```

### 关键类/方法说明

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| API 入口 | `memory_engine.py` | `MemoryEngine.retain_batch_async()` | L3558 | 接收用户请求，验证、清理、分块 |
| 内部分发 | `memory_engine.py` | `_retain_batch_async_internal()` | L3994 | 解析配置，调用 orchestrator |
| 核心编排 | `retain/orchestrator.py` | `retain_batch()` | L706 | 内存防御、delta 检测、流式分派 |
| 流式处理 | `retain/orchestrator.py` | `_streaming_retain_batch()` | L1304 | 生产者-消费者模式，异步提取+写入 |
| 批处理 | `retain/orchestrator.py` | `_process_db_batch()` | L1603 | 单个批次的 Phase 1→2→3 |

---

## 阶段 1：LLM 叙事事实抽取

### 执行时序

```
_extract_and_embed()                                    # orchestrator.py:579
  └─ fact_extraction.extract_facts_from_contents()      # fact_extraction.py:2502
       ├─ [chunks 模式] _extract_facts_chunks()         # fact_extraction.py:2452
       ├─ [batch API] extract_facts_from_contents_batch_api()  # fact_extraction.py:1958
       └─ [默认] 并行调用 extract_facts_from_text()      # fact_extraction.py:1793
            └─ chunk_text()                             # fact_extraction.py:494 (文本分块)
            └─ 并行 _extract_facts_with_auto_split()    # fact_extraction.py:1683
                 └─ _extract_facts_from_chunk()         # fact_extraction.py:1306
                      ├─ _build_extraction_prompt_and_schema()  # fact_extraction.py:1058
                      │    ├─ 选择提取模式: concise/verbose/verbatim/custom
                      │    ├─ 构建系统提示词 (CONCISE_FACT_EXTRACTION_PROMPT 等)
                      │    └─ 构建响应 Pydantic Schema
                      ├─ _build_user_message()          # fact_extraction.py:1197
                      │    ├─ _retain_mission_preamble()  (银行级保留策略)
                      │    └─ 注入 Narrator、Event Date、Context
                      └─ llm_config.call()              # LLM 调用 (Temperature: 0.1)
                           └─ 解析 JSON → Fact 模型 → 合并为 ExtractedFact
```

### 关键类/方法

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 提取入口 | `retain/orchestrator.py` | `_extract_and_embed()` | L579 | 阶段 1+2 的共享入口 |
| 批量提取 | `retain/fact_extraction.py` | `extract_facts_from_contents()` | L2502 | 并行处理多个 content |
| 文本分块 | `retain/fact_extraction.py` | `chunk_text()` | L494 | 结构化感知分块 (JSON/JSONL/纯文本) |
| 自动拆分 | `retain/fact_extraction.py` | `_extract_facts_with_auto_split()` | L1683 | 输出超长时自动拆分重试 |
| 单块提取 | `retain/fact_extraction.py` | `_extract_facts_from_chunk()` | L1306 | 核心 LLM 调用与解析 |
| 提示词构建 | `retain/fact_extraction.py` | `_build_extraction_prompt_and_schema()` | L1058 | 根据模式构建提示词和 Schema |
| 用户消息 | `retain/fact_extraction.py` | `_build_user_message()` | L1197 | 注入日期、上下文、Narrator |
| 事实模型 | `retain/fact_extraction.py` | `Fact` | L147 | 最终存储的事实模型 |
| 提取事实 | `retain/fact_extraction.py` | `ExtractedFact` | L200 | what/when/where/who/why + 实体 + 因果 |
| 事实类型 | `retain/fact_extraction.py` | `ExtractedFact` | L200 | fact_type: `world` / `assistant` → 映射为 `world` / `experience` |
| 因果抽取 | `retain/fact_extraction.py` | `FactCausalRelation` | L182 | 可选：因果链接 `caused_by` |
| 时间推断 | `retain/fact_extraction.py` | `_infer_temporal_date()` | L75 | 相对时间 → 绝对 ISO 时间戳 |
| Batch API | `retain/fact_extraction.py` | `extract_facts_from_contents_batch_api()` | L1958 | OpenAI/Groq Batch API 模式 |

### 提取模式

| 模式 | 配置值 | 特点 |
|------|--------|------|
| `concise` | 默认 | 选择性提取，每个 chunk 2-5 个事实，过滤问候语/填充词 |
| `verbose` | `retain_extraction_mode="verbose"` | 详细提取，保留所有细节 |
| `verbatim` | `retain_extraction_mode="verbatim"` | 原文保留，仅提取元数据 |
| `custom` | `retain_extraction_mode="custom"` | 使用自定义指令 |
| `chunks` | `retain_extraction_mode="chunks"` | 无 LLM，每块即一个 memory unit |

---

## 阶段 2：嵌入向量生成

### 执行时序

```
_extract_and_embed()                                    # orchestrator.py:579
  ├─ fact_extraction.extract_facts_from_contents()      # 阶段 1 (见上)
  └─ embedding_processing.augment_texts_with_dates()    # embedding_processing.py:15
       └─ 格式: "fact_text (happened in 2024-06-15) [entity1, entity2]"
  └─ embedding_processing.generate_embeddings_batch()   # embedding_processing.py:49
       └─ embedding_utils.generate_embeddings_batch()   # 批量生成 384 维向量
```

### 关键类/方法

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 文本增强 | `retain/embedding_processing.py` | `augment_texts_with_dates()` | L15 | 注入日期+实体，提升时序匹配 |
| 批量嵌入 | `retain/embedding_processing.py` | `generate_embeddings_batch()` | L49 | 批量生成嵌入向量 |
| 默认模型 | 配置 | `BAAI/bge-small-en-v1.5` | - | 384 维向量 |

### 文本增强格式
```
原始: "Alice works at Google"
增强: "Alice works at Google (happened in June 2024) [Alice, Google]"
```

---

## 阶段 3：实体解析与归一化

### 执行时序

```
_pre_resolve_phase1()                                   # orchestrator.py:374
  └─ entity_processing.resolve_entities()               # entity_processing.py:52
       └─ _prepare_facts_for_entity_processing()        # entity_processing.py:15
            └─ 合并 LLM 实体 + 用户提供的实体
       └─ link_utils.resolve_entities_only()            # link_utils.py:207
            └─ _prepare_entities_for_resolution()       # link_utils.py:144
            └─ entity_resolver.resolve_entities_batch() # entity_resolver.py:243
                 └─ _resolve_entities_batch_impl()      # entity_resolver.py:284
                      ├─ pg_trgm 候选查找 (相似度阈值 0.15)
                      ├─ 批量获取共现实体
                      └─ 评分与匹配 (阈值 0.6)
```

### 实体评分公式

```python
# entity_resolver.py:684-712
score = 0.0

# 1. 字符串相似度 (权重 0.5)
name_similarity = SequenceMatcher(entity_text, canonical_name).ratio()
score += name_similarity * 0.5

# 2. 共现重叠 (权重 0.3)
co_entity_score = overlap / len(nearby_entity_set)
score += co_entity_score * 0.3

# 3. 时间邻近度 (权重 0.2, 7 天窗口)
days_diff = abs(event_date - last_seen).days
temporal_score = max(0, 1.0 - days_diff / 7)
score += temporal_score * 0.2

# 阈值: score > 0.6 接受匹配
```

### 关键类/方法

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 阶段 1 入口 | `retain/orchestrator.py` | `_pre_resolve_phase1()` | L374 | 读密集型操作，独立连接 |
| 实体处理 | `retain/entity_processing.py` | `resolve_entities()` | L52 | 协调实体数据准备和解析 |
| 实体解析 | `retain/link_utils.py` | `resolve_entities_only()` | L207 | 格式转换 + 调用 resolver |
| 核心解析器 | `entity_resolver.py` | `EntityResolver` | L113 | 实体解析主类 |
| 批量解析 | `entity_resolver.py` | `resolve_entities_batch()` | L243 | 批量实体解析入口 |
| 内部实现 | `entity_resolver.py` | `_resolve_entities_batch_impl()` | L284 | trigram 候选 + 评分 |
| 实体类型 | `retain/types.py` | `ResolvedEntity` | - | entity_id + canonical_name |

---

## 阶段 4：图链接构建

### 执行时序

```
_insert_facts_and_links()                               # orchestrator.py:476
  │
  ├─ fact_storage.insert_facts_batch()                  # fact_storage.py:63
  │    └─ 插入 memory_units 行，返回 unit_ids
  │
  ├─ _remap_phase1_results()                            # orchestrator.py:439
  │    └─ 占位符 ID → 真实 UUID 映射
  │
  ├─ entity_resolver.reassert_entities_batch()          # 重新锁定实体
  ├─ entity_resolver.link_units_to_entities_batch()     # 插入 unit_entities
  │
  ├─ link_creation.create_temporal_links_batch()        # link_creation.py:15
  │    └─ Temporal Links: w = exp(-Δt/σ_t), ±24h 窗口
  │        每 unit 最多 20 条
  │
  ├─ link_creation.create_semantic_links_batch()        # link_creation.py:35
  │    ├─ Within-batch: numpy 余弦相似度 (内存计算)
  │    └─ Pre-computed ANN: HNSW 索引查询 (Phase 1)
  │        阈值: config.semantic_link_min_similarity (默认 0.7)
  │
  └─ link_creation.create_causal_links_batch()          # link_creation.py:80
       └─ Causal Links: w = 1.0, 单向 caused_by
```

### 4.1 Temporal Links（时间链接）

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 入口 | `retain/link_creation.py` | `create_temporal_links_batch()` | L15 | 转发到 link_utils |
| 核心实现 | `retain/link_utils.py` | `create_temporal_links_batch_per_fact()` | L287 | 双向索引扫描 |
| 权重公式 | - | `w = max(0.3, 1.0 - Δt/24h)` | - | 24 小时窗口 |
| 上限 | `retain/link_utils.py` | `MAX_TEMPORAL_LINKS_PER_UNIT = 20` | L30 | 防 hub 爆炸 |

### 4.2 Semantic Links（语义链接）

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 入口 | `retain/link_creation.py` | `create_semantic_links_batch()` | L35 | 合并 within-batch + ANN |
| ANN 搜索 | `retain/link_utils.py` | `compute_semantic_links_ann()` | L427 | 临时表 + LATERAL HNSW 查询 |
| 批内计算 | `retain/link_utils.py` | `compute_semantic_links_within_batch()` | L572 | numpy 余弦相似度 |
| 最终 ANN | `retain/orchestrator.py` | `_run_final_semantic_ann()` | L1149 | 流式模式：所有批次提交后统一 ANN |

### 4.3 Causal Links（因果链接）

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 入口 | `retain/link_creation.py` | `create_causal_links_batch()` | L80 | 转发到 link_utils |
| 核心实现 | `retain/link_utils.py` | `create_causal_links_batch()` | L710 | 验证 target_index |
| 写入 | `retain/link_utils.py` | `_write_causal_links_batch()` | L754 | 仅允许 `caused_by` |

### 4.4 链接上限

```
每 unit 最大 20 条 temporal links（_cap_links_per_unit）
Entity links 不写入 memory_links（由 unit_entities 自连接衍生）
```

---

## 阶段 5：观点自动强化（Consolidation 后台作业）

> ⚠️ **此阶段不在 retain 管道内执行**，而是由独立的 consolidation 后台作业异步处理。

### 执行时序

```
consolidation/consolidator.py
  └─ run_consolidation_job()                  # consolidator.py:953
       └─ _run_consolidation_job()            # consolidator.py:1002
            ├─ 加载未整合的 memory_units
            ├─ 加载现有 observations
            └─ _consolidate_batch_with_llm()  # consolidator.py:2421
                 ├─ 构建 consolidation 提示词
                 ├─ LLM 调用：评估关系
                 │    ├─ reinforce (强化) → 增加 proof_count
                 │    ├─ weaken (弱化) → 减少 proof_count
                 │    ├─ contradict (矛盾) → 可能删除旧 observation
                 │    └─ neutral (无关) → 保留
                 ├─ 创建新 observation
                 └─ 更新/删除现有 observation
```

### 关键类/方法

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 公开入口 | `consolidation/consolidator.py` | `run_consolidation_job()` | L953 | 绑定 trace context，调用内部实现 |
| 核心流程 | `consolidation/consolidator.py` | `_run_consolidation_job()` | L1002 | 加载记忆、分批处理 |
| LLM 整合 | `consolidation/consolidator.py` | `_consolidate_batch_with_llm()` | L2421 | LLM 评估关系并更新 |
| 趋势模型 | `reflect/observations.py` | `Trend` | L15 | STABLE/STRENGTHENING/WEAKENING/NEW/STALE |
| 证据模型 | `reflect/observations.py` | `ObservationEvidence` | L33 | 记忆 ID + 引用 + 时间戳 |
| 提示词 | `consolidation/prompts.py` | `build_consolidation_system_prompt()` | - | 构建整合系统提示词 |

---

## 阶段 6：背景合并（Consolidation 后台作业）

> ⚠️ **此阶段与阶段 5 共用同一个 consolidation 后台作业**，在 LLM 调用中一并处理。

### 执行时序

与阶段 5 相同，`_consolidate_batch_with_llm()` 在一次 LLM 调用中同时处理：

1. **冲突解决**（新信息优先）：当新事实与现有 observation 矛盾时，以新事实为准
2. **补充信息追加**：新事实补充到现有 observation 的 evidence 中
3. **第一人称视角统一**：保持 observation 的叙述视角一致
4. **保持简洁**：合并冗余信息，避免 observation 膨胀

### 关键类/方法

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 合并逻辑 | `consolidation/consolidator.py` | `_consolidate_batch_with_llm()` | L2421 | LLM 驱动的合并 |
| 删除观察 | `consolidation/consolidator.py` | _delete_observation | L2249 | 删除被取代/矛盾的 observation |
| 提示词 | `consolidation/prompts.py` | `build_consolidation_system_prompt()` | - | 包含合并策略指令 |

---

## 完整执行时序图（流式批处理模式）

```
MemoryEngine.retain_batch_async()                         # memory_engine.py:3558
│
└─ _retain_batch_async_internal()                         # memory_engine.py:3994
   │
   └─ orchestrator.retain_batch()                         # orchestrator.py:706
      │
      ├─ [Memory Defense 预筛选]                            # orchestrator.py:833
      │
      ├─ [_try_delta_retain()]                            # orchestrator.py:2241
      │   └─ 增量模式：仅处理变更的 chunks
      │
      └─ _streaming_retain_batch()                        # orchestrator.py:1304
         │
         ├─ chunk_text() 预分块                            # fact_extraction.py:494
         │
         ├─ [Producer] _llm_producer()                    # orchestrator.py:1467
         │   │
         │   └─ 每个 chunk 并行:
         │       └─ _extract_and_embed()                  # orchestrator.py:579
         │          ├─ 【阶段 1】fact_extraction.extract_facts_from_contents()
         │          │   └─ extract_facts_from_text()
         │          │       └─ _extract_facts_with_auto_split()
         │          │           └─ _extract_facts_from_chunk()
         │          │               ├─ _build_extraction_prompt_and_schema()
         │          │               ├─ _build_user_message()
         │          │               └─ llm_config.call()
         │          │
         │          └─ 【阶段 2】embedding_processing.augment_texts_with_dates()
         │              embedding_processing.generate_embeddings_batch()
         │
         └─ [Consumer] _db_consumer()                     # orchestrator.py:1545
            │
            └─ 每个 batch:
                └─ _process_db_batch()                    # orchestrator.py:1603
                   │
                   ├─ 【阶段 3】_pre_resolve_phase1()     # orchestrator.py:374
                   │   ├─ entity_processing.resolve_entities()
                   │   │   └─ entity_resolver.resolve_entities_batch()
                   │   │       └─ 评分: sim_str*0.5 + sim_co*0.3 + sim_temp*0.2
                   │   │
                   │   └─ [跳过] 流式模式不在此处做 ANN
                   │
                   └─ 【阶段 4】_insert_facts_and_links()  # orchestrator.py:476
                       ├─ fact_storage.insert_facts_batch()
                       ├─ entity_resolver.link_units_to_entities_batch()
                       ├─ link_creation.create_temporal_links_batch()
                       ├─ link_creation.create_semantic_links_batch()
                       └─ link_creation.create_causal_links_batch()

         └─ 【最终 ANN】_run_final_semantic_ann()          # orchestrator.py:1149
            └─ 所有批次提交后，统一 ANN 搜索

────────────────────  retain 事务提交  ────────────────────

【阶段 5+6】consolidation 后台作业（异步触发）
  └─ consolidation/consolidator.py:run_consolidation_job()
       └─ _run_consolidation_job()
            └─ _consolidate_batch_with_llm()
                 ├─ 评估关系: reinforce / weaken / contradict / neutral
                 ├─ 合并背景 (新信息优先、补充追加、视角统一)
                 └─ 创建/更新/删除 observations
```

---

## 关键数据类型

| 类型 | 文件 | 说明 |
|------|------|------|
| `RetainContent` | `retain/types.py` | 输入内容项 |
| `ExtractedFact` | `retain/types.py` | LLM 提取的事实 (fact_text, fact_type, entities, causal_relations) |
| `ProcessedFact` | `retain/types.py` | 处理后的可存储事实 (含 embedding, document_id, chunk_id) |
| `ChunkMetadata` | `retain/types.py` | 分块元数据 |
| `ResolvedEntity` | `retain/types.py` | 已解析的实体 (entity_id, canonical_name) |
| `Phase1Result` | `retain/types.py` | Phase 1 结果 (entities + semantic_ann_links) |
| `Fact` | `retain/fact_extraction.py` | LLM 返回的原始事实 Pydantic 模型 |
| `CausalRelation` | `retain/fact_extraction.py` | 因果关系 (target_fact_index, relation_type) |

---

## 关键配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `retain_chunk_size` | 3000 | 文本分块大小（字符） |
| `retain_chunk_batch_size` | 100 | 流式批处理大小（chunk 数） |
| `retain_extraction_mode` | `concise` | 提取模式 |
| `retain_extract_causal_links` | False | 是否抽取因果链接 |
| `llm_temperature_retain` | 0.1 | 提取温度 |
| `semantic_link_min_similarity` | 0.7 | 语义链接阈值 |
| `entity_lookup` | `trigram` | 实体查找策略 |
| `entity_resolution_batch_size` | 100 | 实体解析批量大小 |
| `enable_observations` | True | 是否启用 consolidation |
| `consolidation_batch_size` | - | 整合批处理大小 |