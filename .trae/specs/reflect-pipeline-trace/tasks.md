# Tasks

- [x] Task 1: 编写 Reflect 管道执行时序文档
  - [x] 梳理入口层：MemoryEngine.reflect_async → run_reflect_agent → _run_reflect_agent_inner
  - [x] 梳理 Disposition Profile 集成：bank profile + directives 加载
  - [x] 梳理 Agentic Loop：强制顺序检索 → auto 模式 → 终止条件
  - [x] 梳理四工具实现：search_mental_models / search_observations / recall / expand
  - [x] 梳理最终合成：build_final_prompt → LLM 调用 → 结构化输出
  - [x] 标注观点形成与强化、背景合并在当前代码中的实现位置（consolidation 模块）
  - [x] 生成最终文档 `docs/reflect-pipeline-trace.md`

# Task Dependencies
- 所有子任务无依赖关系，可并行执行