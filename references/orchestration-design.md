# 编排设计产物

生成 YAML 前必须先完成内部 `workflow_intent` 设计。该设计用于约束任务拆解、节点选择、执行顺序、失败处理和人工确认点。

## 核心字段

- `goal`：用户目标。
- `final_output`：最终交付物。
- `risk_level`：`low` / `medium` / `high`。
- `execution_mode`：`single` / `sequential` / `parallel_limited` / `parallel_fanout` / `gated_loop`。
- `human_checkpoints`：需要用户确认的节点。
- `worker_strategy`：串行、有限并行、扇出并行、带门禁循环。
- `budget_or_time_hint`：预算、时间、轮次或复杂度提示。
- `proof_requirements`：截图、测试报告、文件路径、评审结论等证明材料。
- `failure_policy`：可重试、需用户决策、失败终止、接受部分结果。
- `nodes_plan`：节点级职责、输入、输出、依赖、验收、失败处理。

## 设计原则

1. 先判断是否需要拆分，再决定节点数量。
2. 先判断依赖关系，再决定串行或并行。
3. 先判断风险，再决定是否需要人工确认。
4. 先判断验收方式，再设计输出变量。
5. 先设计失败处理，再生成边和循环。

## 节点计划要求

`nodes_plan` 中每个节点都必须说明：

1. 节点为什么存在；
2. 使用哪个节点类型；
3. 建议匹配什么 ADE 能力；
4. 消费哪些上游输入；
5. 产出哪些下游会使用的输出；
6. 通过什么标准验收；
7. 失败后是重试、转人工、终止还是接受部分结果。
