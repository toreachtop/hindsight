# Hindsight Reflect（CARA）推理时序流程追踪

> 基于最新代码库 `hindsight-api-slim` 梳理，更新日期：2026-08-03

## 总览

Reflect 实现 **CARA（Context-Aware Reasoning Agent）** 推理架构，核心是 **Agentic Loop**：LLM 通过工具调用自主检索记忆，最终合成回答。

```
查询 Q + Budget
     │
     ▼
┌─────────────────────────────────────────────────┐
│ 能力 1: Disposition Profile 集成                  │
│ 能力 2: 记忆集成（Agentic Loop + 四工具检索）       │
│ 能力 3: 观点形成与强化（consolidation 模块）        │
│ 能力 4: 背景合并（consolidation 模块）              │
└─────────────────────────────────────────────────┘
```

> ⚠️ **重要发现**：能力 3（观点形成与强化）和能力 4（背景合并）不在 reflect 管道内执行，而是由独立的 **consolidation** 后台作业异步处理。Reflect 本身是**只读操作**（见 agent.py 文档："Reflect is read-only: it synthesizes an answer from the bank's stored memories and persists nothing."）。

---

## 入口层：从 API 到 Agentic Loop

### 调用链

```
MemoryEngine.reflect_async()                       # memory_engine.py:9790
  │
  ├─ sanitize_text(query)
  ├─ _authenticate_tenant()
  ├─ [Profile] get_bank_profile()                  # 银行身份/使命
  ├─ [Directives] list_directives()                # 硬性规则
  ├─ [Config] _config_resolver.resolve_full_config()
  ├─ [Freshness] get_bank_freshness()              # 整合状态
  ├─ [Budget] 计算 max_iterations
  │
  ├─ 构建四工具回调：
  │   ├─ search_mental_models_fn()                  # 工具1: 心智模型
  │   ├─ search_observations_fn()                   # 工具2: 观察
  │   ├─ recall_fn()                                # 工具3: 原始事实
  │   └─ expand_fn()                                # 工具4: 上下文扩展
  │
  └─ run_reflect_agent()                            # agent.py:345
       └─ _run_reflect_agent_inner()                # agent.py:402
            ├─ get_reflect_tools()                  # tools_schema.py
            ├─ build_system_prompt_for_tools()      # prompts.py
            └─ [Agentic Loop] 迭代:
                 ├─ 强制顺序: mental_models → observations → recall
                 ├─ auto 模式: LLM 自主选择工具
                 ├─ "done" → 最终合成
                 └─ 末轮 → 最终合成
```

### 关键类/方法

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| API 入口 | `memory_engine.py` | `MemoryEngine.reflect_async()` | L9790 | 鉴权、参数校验、工具回调构建 |
| Agent 入口 | `reflect/agent.py` | `run_reflect_agent()` | L345 | 缓存管理、错误处理 |
| Agent 核心 | `reflect/agent.py` | `_run_reflect_agent_inner()` | L402 | 主循环、工具执行、最终合成 |
| 结果模型 | `reflect/models.py` | `ReflectAgentResult` | L109 | text, iterations, tool_trace, usage |

---

## 能力 1：Disposition Profile 集成

### 当前实现

```
reflect_async()                                       # memory_engine.py:9790
  ├─ profile = await get_bank_profile(bank_id)        # L9899
  │    └─ 返回 bank name + mission 字符串
  │
  └─ directives = await list_directives(bank_id)      # L10036-10055
       └─ 从 directives 表加载活跃指令
       └─ 支持 tag 隔离: apply_all_directives 参数
```

### 关键类/方法

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 银行档案 | `memory_engine.py` | `get_bank_profile()` | L9899 | 获取 bank 名称和使命 |
| 指令加载 | `memory_engine.py` | `list_directives()` | L10036 | 加载活跃指令（支持 tag 范围） |
| 指令注入 | `reflect/prompts.py` | `build_directives_section()` | L34 | 注入到 system prompt 开头 |
| 指令提醒 | `reflect/prompts.py` | `build_directives_reminder()` | L66 | 注入到 prompt 末尾 |
| 指令提取 | `reflect/prompts.py` | `_extract_directive_rules()` | L23 | 从指令中提取规则文本 |

> ⚠️ **与设计文档的差异**：当前代码**未实现**完整的三维行为空间 Θ = (S, L, E, β)。Profile 仅包含 bank name 和 mission 字符串，没有 Skepticism/Literalism/Empathy 维度，也没有 Bias Strength 参数。Profile 被直接注入 system prompt 而非通过 `φ(Θ)` 函数 verbalize。

### System Prompt 构建

```python
# prompts.py:96
build_system_prompt_for_tools(
    bank_profile,      # {name, mission}
    context,           # 额外上下文
    directives,        # 硬性规则列表
    has_mental_models,
    include_observations,
    budget,
)
```

生成的 system prompt 结构：
1. **Role**: 银行身份定义
2. **Directives**: 硬性规则（MUST follow）
3. **Retrieval Strategy**: 分层检索策略说明
4. **Workflow**: 强制顺序说明
5. **Tool Descriptions**: 工具说明（由 LLM native tool calling 提供）

---

## 能力 2：记忆集成 — Agentic Loop

### 执行流程

```
_run_reflect_agent_inner()                           # agent.py:402
  │
  ├─ [初始化] get_reflect_tools()                     # tools_schema.py
  │   └─ 根据配置决定启用哪些工具
  │
  ├─ [初始化] build_system_prompt_for_tools()         # prompts.py
  │
  └─ [Agentic Loop] for iteration in range(max_iterations):
       │
       ├─ is_last? → build_final_prompt() → LLM 调用 → 返回
       │
       ├─ context overflow? → build_final_prompt() → 返回
       │
       ├─ tool_choice 决定:
       │   ├─ iteration 0: forced "search_mental_models"
       │   ├─ iteration 1: forced "search_observations"
       │   ├─ iteration 2: forced "recall"
       │   └─ iteration 3+: LLM_TOOL_CHOICE_AUTO
       │
       ├─ llm_config.call_with_tools() → tool_calls
       │
       ├─ 无 tool_calls?
       │   ├─ 首次无工具 → ReflectToolCallError
       │   └─ 已有工具调用 → build_final_prompt() → 返回
       │
       ├─ "done" tool?
       │   ├─ 无证据 → 拒绝，要求先检索
       │   └─ 有证据 → _process_done_tool() → 返回
       │
       └─ 其他工具 → 并行执行 → 添加结果到 messages → 继续
```

### 强制顺序 + 智能短路

| 迭代 | tool_choice | 行为 |
|------|-------------|------|
| 0 | `search_mental_models` | 强制调用心智模型搜索 |
| 1 | `search_observations` | 强制调用观察搜索 |
| 2 | `recall` | 强制调用原始事实搜索 |
| 3+ | `auto` | LLM 自主选择工具或 "done" |

**智能短路**：当 `search_mental_models` 返回**全部新鲜且可用**的心智模型时（仅限 LOW/MID budget），立即释放为 `auto` 模式，允许 LLM 直接回答。

### 关键类/方法

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 主循环 | `reflect/agent.py` | `_run_reflect_agent_inner()` | L402 | 迭代循环 + 工具调度 |
| 工具定义 | `reflect/tools_schema.py` | `get_reflect_tools()` | - | 构建 OpenAI 格式工具定义 |
| 强制顺序 | `reflect/agent.py` | `forced_sequence` 逻辑 | L769-783 | 0→mental_models, 1→observations, 2→recall |
| 智能短路 | `reflect/agent.py` | `stop_forcing_from_iteration` | L1146-1156 | 新鲜心智模型 → 释放 auto |
| 证据守卫 | `reflect/agent.py` | `has_gathered_evidence` | L994-1017 | done 前必须检索过证据 |
| 幻觉检测 | `reflect/agent.py` | `enabled_tools` 过滤 | L1048-1081 | 拒绝未启用的工具调用 |
| 工具并行 | `reflect/agent.py` | `asyncio.gather(*tool_tasks)` | L1107 | 并行执行同轮次多个工具 |
| 上下文守卫 | `reflect/agent.py` | `_count_messages_tokens()` | L250 | 防止超过 max_context_tokens |
| 增量缓存 | `reflect/agent.py` | `_schedule_cache()` / `_resolve_pending_cache()` | L536-553 | 逐步缓存对话前缀 |

### 四工具体系

| 工具 | 文件 | 函数 | 行号 | 检索目标 | 质量 |
|------|------|------|------|----------|------|
| `search_mental_models` | `reflect/tools.py` | `tool_search_mental_models()` | L59 | 用户创建的心智模型 | 最高 |
| `search_observations` | `reflect/tools.py` | `tool_search_observations()` | L162 | 自动整合的观察 | 高 |
| `recall` | `reflect/tools.py` | `tool_recall()` | L244 | 原始世界/经验事实 | 基础 |
| `expand` | `reflect/tools.py` | `tool_expand()` | L310 | Chunk/Document 上下文 | 辅助 |

---

## 工具实现详解

### 工具 1：search_mental_models

```
tool_search_mental_models()                           # tools.py:59
  └─ HNSW 语义搜索 mental_models 表
  └─ compute_mental_model_is_stale()                 # 检查新鲜度
  └─ 返回: {mental_models: [...], is_stale, staleness_reason}
```

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 工具实现 | `reflect/tools.py` | `tool_search_mental_models()` | L59 | 语义搜索 mental_models 表 |
| 新鲜度检查 | `memory_engine.py` | `compute_mental_model_is_stale()` | - | 检查是否有新记忆 |
| 工具 Schema | `reflect/tools_schema.py` | `TOOL_SEARCH_MENTAL_MODELS` | L13 | OpenAI 格式工具定义 |

### 工具 2：search_observations

```
tool_search_observations()                            # tools.py:162
  └─ memory_engine.recall_async(fact_type=["observation"])
  └─ 返回: {observations: [...], is_stale, freshness, source_facts}
```

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 工具实现 | `reflect/tools.py` | `tool_search_observations()` | L162 | 调用 recall_async 搜索 observation |
| 新鲜度 | `reflect/tools.py` | `pending_consolidation` 判断 | L226-232 | up_to_date / slightly_stale / stale |
| 源事实 | `reflect/tools.py` | `source_facts_max_tokens` | L173 | 可配置的源事实预算 |
| 工具 Schema | `reflect/tools_schema.py` | `TOOL_SEARCH_OBSERVATIONS` | L43 | OpenAI 格式工具定义 |

### 工具 3：recall

```
tool_recall()                                         # tools.py:244
  └─ memory_engine.recall_async(fact_type=["world","experience"])
  └─ 返回: {memories: [...], chunks: {...}}
```

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 工具实现 | `reflect/tools.py` | `tool_recall()` | L244 | 调用 recall_async 搜索原始事实 |
| 默认 token | - | `max_tokens=2048` | L249 | 内部调用默认 token 预算 |
| 块上下文 | `reflect/tools.py` | `include_chunks` + `max_chunk_tokens` | L256 | 可选 chunk 上下文 |
| 工具 Schema | `reflect/tools_schema.py` | `TOOL_RECALL` | L74 | OpenAI 格式工具定义 |

### 工具 4：expand

```
tool_expand()                                         # tools.py:310
  └─ 根据 depth 获取 chunk 级或 document 级上下文
  └─ 返回: {chunks: [...], documents: [...]}
```

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 工具实现 | `reflect/tools.py` | `tool_expand()` | L310 | 批量扩展记忆上下文 |
| 工具 Schema | `reflect/tools_schema.py` | `TOOL_EXPAND` | L109 | OpenAI 格式工具定义 |

---

## 最终合成

### 执行流程

```
[末轮或 context overflow 或 done 工具]
  │
  ├─ build_final_prompt()                             # prompts.py
  │   └─ 构建: query + 工具调用历史 + 上下文
  │   └─ 受 max_context_tokens 限制 (80% 给内容)
  │
  ├─ build_final_system_prompt()                      # prompts.py
  │   └─ 角色: "thoughtful assistant"
  │   └─ 注入: mission + directives
  │
  ├─ llm_config.call()                                # LLM 调用
  │   └─ scope="reflect"
  │   └─ max_completion_tokens=max_tokens
  │
  └─ [可选] _generate_structured_output()             # agent.py:104
       └─ 使用 Pydantic 动态模型提取结构化数据
       └─ 额外 LLM 调用
```

### 关键类/方法

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 最终 prompt | `reflect/prompts.py` | `build_final_prompt()` | - | 构建最终合成 prompt |
| 最终 system | `reflect/prompts.py` | `build_final_system_prompt()` | - | 最终 system prompt |
| 上下文预算 | `reflect/prompts.py` | `_FINAL_PROMPT_CONTEXT_FRACTION = 0.8` | L17 | 80% 给工具结果上下文 |
| 结构化输出 | `reflect/agent.py` | `_generate_structured_output()` | L104 | 从答案提取结构化数据 |
| 工具名称归一化 | `reflect/agent.py` | `_normalize_tool_name()` | L72 | 兼容多种 LLM 输出格式 |
| done 检测 | `reflect/agent.py` | `_is_done_tool()` | L99 | 检测 done 工具调用 |
| Token 计数 | `reflect/tokenization.py` | `count_cl100k_tokens()` | - | cl100k_base 编码计数 |

---

## 能力 3&4：观点形成与强化 / 背景合并

> ⚠️ **这两个能力不在 reflect 管道内执行**，而是由 consolidation 后台作业独立处理。

```
consolidation/consolidator.py
  └─ run_consolidation_job()                          # consolidator.py:953
       └─ _run_consolidation_job()                    # consolidator.py:1002
            └─ _consolidate_batch_with_llm()          # consolidator.py:2421
                 ├─ 评估: reinforce / weaken / contradict / neutral
                 ├─ 合并: 冲突解决 + 补充追加 + 视角统一
                 └─ 创建/更新/删除 observations
```

### 相关文件

| 步骤 | 文件 | 类/方法 | 行号 | 说明 |
|------|------|---------|------|------|
| 整合入口 | `consolidation/consolidator.py` | `run_consolidation_job()` | L953 | 后台整合作业 |
| LLM 整合 | `consolidation/consolidator.py` | `_consolidate_batch_with_llm()` | L2421 | LLM 评估关系并更新 |
| 趋势模型 | `reflect/observations.py` | `Trend` | L15 | STABLE/STRENGTHENING/WEAKENING/NEW/STALE |
| 证据模型 | `reflect/observations.py` | `ObservationEvidence` | L33 | 记忆 ID + 引用 + 时间戳 |
| 提示词 | `consolidation/prompts.py` | `build_consolidation_system_prompt()` | - | 整合策略指令 |

---

## 完整执行时序图

```
MemoryEngine.reflect_async()                           # memory_engine.py:9790
│
├─ sanitize_text(query + context)
├─ _authenticate_tenant()
├─ request_context.raise_if_cancelled()
│
├─ [Profile] get_bank_profile(bank_id)                # L9899
│   └─ 返回 {name, mission}
│
├─ [Config] _config_resolver.resolve_full_config()    # L9905
│   ├─ max_iterations = base × budget_multiplier
│   ├─ max_context_tokens
│   └─ wall_timeout
│
├─ [Freshness] get_bank_freshness(bank_id)            # L9924
│   └─ last_consolidated_at, pending_consolidation
│
├─ [Directives] list_directives(bank_id)              # L10036
│   └─ build_directives_section() → system prompt
│   └─ build_directives_reminder() → prompt 末尾
│
├─ [Tools] 构建四工具回调:
│   ├─ search_mental_models_fn → tool_search_mental_models()
│   │   └─ embedding_utils.generate_embeddings_batch()
│   │   └─ HNSW 搜索 mental_models 表
│   │   └─ compute_mental_model_is_stale()
│   │
│   ├─ search_observations_fn → tool_search_observations()
│   │   └─ memory_engine.recall_async(fact_type=["observation"])
│   │
│   ├─ recall_fn → tool_recall()
│   │   └─ memory_engine.recall_async(fact_type=["world","experience"])
│   │
│   └─ expand_fn → tool_expand()
│       └─ 获取 chunk/document 级上下文
│
└─ run_reflect_agent()                                 # agent.py:345
   └─ _run_reflect_agent_inner()                      # agent.py:402
      │
      ├─ get_reflect_tools()                           # tools_schema.py
      │   └─ 根据配置决定启用哪些工具
      │
      ├─ build_system_prompt_for_tools()               # prompts.py:96
      │   └─ 注入 profile + directives + strategy
      │
      └─ [Agentic Loop]
         │
         ├─ iter 0: forced "search_mental_models"
         ├─ iter 1: forced "search_observations"
         ├─ iter 2: forced "recall"
         ├─ iter 3+: auto (LLM 自主选择)
         │
         │   每轮:
         │   ├─ call_with_tools() → tool_calls
         │   ├─ "done" + 有证据 → _process_done_tool() → 返回
         │   ├─ 其他工具 → asyncio.gather() 并行执行
         │   └─ 结果追加到 messages
         │
         └─ [最终合成]
            ├─ build_final_prompt(query, history, profile, context)
            ├─ build_final_system_prompt(mission, language, directives)
            ├─ llm_config.call(scope="reflect", max_tokens)
            └─ [可选] _generate_structured_output(answer, schema)
               └─ 额外 LLM 调用提取结构化数据

────────────────────  reflect 返回 ────────────────────

[能力 3&4] consolidation 后台作业（异步触发）
  └─ consolidation/consolidator.py:run_consolidation_job()
       └─ _consolidate_batch_with_llm()
            ├─ 评估关系: reinforce / weaken / contradict / neutral
            ├─ 合并背景 (新信息优先、补充追加、视角统一)
            └─ 创建/更新/删除 observations
```

---

## 关键数据类型

| 类型 | 文件 | 说明 |
|------|------|------|
| `ReflectAgentResult` | `reflect/models.py` | Agent 最终返回 |
| `ToolCall` | `reflect/models.py` | 单次工具调用跟踪 |
| `LLMCall` | `reflect/models.py` | 单次 LLM 调用跟踪 |
| `TokenUsageSummary` | `reflect/models.py` | 聚合 token 使用统计 |
| `DirectiveInfo` | `reflect/models.py` | 应用的指令信息 |
| `StructuredOutputResult` | `reflect/models.py` | 结构化输出结果 |
| `ReflectAction` | `reflect/models.py` | 单次 agent 动作 |
| `ReflectActionBatch` | `reflect/models.py` | 并行执行的动作批次 |
| `ObservationSection` | `reflect/models.py` | 观察中的章节 |

## 关键配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `reflect_max_iterations` | 配置 | 基础最大迭代次数 |
| `budget multipliers` | LOW=0.5x, MID=1x, HIGH=2x | 预算乘数 |
| `reflect_max_context_tokens` | 100,000 | 上下文 token 上限 |
| `reflect_wall_timeout` | 配置 | 整体超时 |
| `reflect_prompt_cache_enabled` | True | 增量 prompt 缓存 |
| `_FINAL_PROMPT_CONTEXT_FRACTION` | 0.8 | 最终 prompt 中工具结果占比 |
| `reflect_source_facts_max_tokens` | 配置 | 观察工具源事实预算 |
| `DEFAULT_RECALL_MAX_TOKENS` | 2048 | 内部 recall 默认 token 预算 |

## 与设计文档的关键差异

| 设计文档 | 当前代码实现 |
|----------|-------------|
| 三维行为空间 Θ = (S, L, E, β) | Bank profile 仅含 name + mission 字符串 |
| φ(Θ) verbalize 函数 | Profile 直接注入 system prompt |
| 观点形成与强化在 reflect 内 | 在 consolidation 后台作业中 |
| 背景合并在 reflect 内 | 在 consolidation 后台作业中 |
| Flan-T5-small 时间解析器 | DateparserQueryAnalyzer |