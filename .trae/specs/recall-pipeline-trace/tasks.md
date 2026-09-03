# Tasks

- [x] Task 1: 编写 Recall 管道执行时序文档
  - [x] 梳理入口层：MemoryEngine.recall_async → _search_with_retries
  - [x] 梳理 Step 1（查询嵌入生成）：query embedding 生成流程
  - [x] 梳理 Step 2（四路并行检索）：semantic + BM25 + graph + temporal 的并行执行
  - [x] 梳理 Step 3（RRF 融合）：reciprocal_rank_fusion / interleave_fusion
  - [x] 梳理 Step 4（Neural 重排序）：cross-encoder + combined scoring + strategy boosts
  - [x] 梳理 Step 5（Token Budget 过滤）：_filter_by_token_budget 贪心截断
  - [x] 生成最终文档 `docs/recall-pipeline-trace.md`

# Task Dependencies
- 所有子任务无依赖关系，可并行执行