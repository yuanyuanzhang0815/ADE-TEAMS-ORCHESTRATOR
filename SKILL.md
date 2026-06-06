---
name: workflow-generator
description: 根据用户目标直接生成 ADE Teams 工作流 YAML。在生成 YAML 前，先完成工作流编排设计，包括任务拆解、节点选择、角色匹配、变量流转、串行/并行/分支/循环、异常处理和验收标准设计。缺少真实 ADE 或 Workspace 信息时，仍直接生成待绑定 YAML，agent/code 节点中的 ade_id、ade_name、ade_role、workspace_id、workspace_name 留空，由用户在 Web 前端图形化界面补齐后再保存和运行。
---

# ADE Teams Workflow Orchestrator & YAML Generator

本技能用于根据用户目标生成 ADE Teams 工作流 YAML。它不是单纯的 YAML formatter，也不是只负责套模板的 generator。它必须先完成 Workflow Orchestration，再生成可导入或可复制到 ADE Teams 配置文件编辑器中的 YAML。

核心定位：

1. 先编排，再生成 YAML。
2. 直接输出 YAML，不因为缺少 ADE / Workspace 信息而只输出文字草稿。
3. 缺少真实 ADE / Workspace 绑定信息时，生成待绑定 YAML。
4. 不编造 `ade_id`、`ade_name`、`ade_role`、`workspace_id`、`workspace_name`。
5. 具体 ADE 与 Workspace 由用户在 Web 前端图形化界面中选择并补齐。
6. YAML 必须语法合法，避免因标题、标签、冒号、中括号等特殊字符导致解析失败。

---

## 1. 适用场景

当用户需要以下能力时使用本技能：

- 创建新的 ADE Teams 工作流；
- 根据一句话目标自动设计工作流；
- 生成 `start -> agent -> end` 的最小工作流；
- 生成多个 ADE Agent 协作的工作流；
- 设计串行、并行、条件分支、循环、质量检查、汇总节点；
- 配置 Agent 节点任务描述、输入输出变量和验收标准；
- 生成符合 ADE Teams YAML 结构的配置文件；
- 生成等待前端绑定 ADE / Workspace 的 YAML。

---

## 2. 输入信息

### 2.1 必需信息

只强制要求用户提供：

1. **工作流目标**：工作流要完成什么任务。

如果工作流目标不明确，必须先追问。

### 2.2 可选信息

以下信息有则使用，没有则使用默认编排策略：

- 输入变量；
- 最终输出要求；
- 节点数量偏好；
- 是否需要并行；
- 是否需要条件分支；
- 是否需要循环重试；
- 是否需要代码节点；
- 是否需要人工确认；
- 可用 ADE Worker 列表；
- 可用 Workspace 列表；
- 业务规范、文档模板、验收标准。

### 2.3 ADE / Workspace 信息处理

ADE / Workspace 信息不是生成 YAML 的阻塞条件。

如果用户没有提供真实 ADE 和 Workspace 信息，仍然必须直接生成 YAML，但在相关节点中将绑定字段留空：

```yaml
ade_id: ""
ade_name: ""
ade_role: ""
workspace_id: ""
workspace_name: ""
```

不得使用示例 ID、猜测 ID、历史模板 ID 或无来源 ID。

建议绑定角色可以通过 YAML 注释或任务描述表达，不能写入未确认的 `ade_role` 字段。

推荐写法：

```yaml
# 建议绑定 ProductManager 角色的 ADE Worker，并选择可访问需求材料的 Workspace。
- type: agent
  title: ADE-需求分析
  _advanced:
    data:
      ade_id: ""
      ade_name: ""
      ade_role: ""
      workspace_id: ""
      workspace_name: ""
```

禁止写法：

```yaml
ade_id: 123456
ade_name: 示例ADE
ade_role: ProductManager
workspace_id: workspace_demo
workspace_name: 示例工作区
```

除非这些值由用户明确提供。

---

## 3. 总体流程

### 阶段 1：理解目标

识别用户希望工作流完成的最终目标，包括：

1. 输入是什么；
2. 输出是什么；
3. 是否涉及文件；
4. 是否涉及代码；
5. 是否涉及评审；
6. 是否涉及风险操作；
7. 是否需要多角色协作。

如果缺少工作流目标，先追问。除此之外，不要因为缺少 ADE ID 或 Workspace ID 阻塞 YAML 生成。

### 阶段 2：Workflow Orchestration 编排设计

生成 YAML 前，必须完成编排设计：

1. 判断任务是否需要拆分；
2. 判断应使用哪些节点类型；
3. 判断节点应该串行、并行还是分支；
4. 判断是否需要循环重试；
5. 判断是否需要 code 节点；
6. 设计每个 Agent 节点的职责；
7. 设计每个节点的输入输出变量；
8. 设计变量如何从上游传递到下游；
9. 设计每个节点的验收标准；
10. 判断最终路径是否能闭环到 end 节点；
11. 形成内部 `workflow_intent` 编排设计，用于约束后续 YAML 生成。

### 阶段 2.5：编排设计产物

生成 YAML 前，必须先在内部完成 `workflow_intent` 编排设计。该设计用于约束任务拆解、节点选择、执行顺序、失败处理和人工确认点，不要求默认完整输出给用户；除非用户要求解释编排，否则只输出最终 YAML 和简要说明。

内部设计结构：

```yaml
workflow_intent:
  goal:
  final_output:
  risk_level: low | medium | high
  execution_mode: single | sequential | parallel_limited | parallel_fanout | gated_loop
  human_checkpoints:
  required_skills:
  worker_strategy:
  budget_or_time_hint:
  proof_requirements:
  failure_policy:
  nodes_plan:
    - node_id:
      node_type:
      role_or_capability:
      skill_invocation:
      responsibility:
      inputs:
      outputs:
      depends_on:
      acceptance:
      failure_handling:
```

设计要求：

1. `goal` 必须能直接映射到最终输出；
2. `execution_mode` 必须与任务复杂度匹配，不得为了复杂而复杂；
3. `nodes_plan` 中每个节点必须有独立职责、明确输入、明确输出和下游用途；
4. 并行节点必须说明互不依赖的理由；
5. 分支节点必须说明判断变量和 true / false 去向；
6. 循环节点必须说明最大次数、退出条件、每轮检查依据和失败后的处理；
7. 高风险动作必须进入 `human_checkpoints`，不得默认自动执行；
8. 如节点依赖已有技能，必须在 `required_skills` 和节点 `skill_invocation` 中记录，并在 YAML 的 `task_desc` / `user_message` 中优先使用 `/技能名` 直接调用；
9. `failure_policy` 必须区分可重试、需用户决策、失败终止和可接受部分结果。

更多细则见 `references/orchestration-design.md` 和 `references/skill-invocation.md`。

### 阶段 3：生成 YAML

按照 ADE Teams 工作流结构生成 YAML。

必须包含：

1. `nodes`；
2. 唯一 `start` 节点；
3. 一个或多个 `agent` / `code` / `if-else` 节点；
4. 唯一 `end` 节点；
5. `edges`；
6. 每个节点的 `_advanced.id`；
7. 每个节点的 `_advanced.style`；
8. 每个 Agent 节点的任务描述、用户消息、验收标准和输出变量。

### 阶段 4：自检

输出 YAML 前必须完成：

1. YAML 语法自检；
2. 节点结构自检；
3. 变量引用自检；
4. 边连接自检；
5. ADE / Workspace 绑定字段自检；
6. 特殊字符安全自检。

---

## 4. 编排原则

### 4.1 编排优先

本技能不允许直接套固定模板。必须根据用户目标判断工作流结构。

生成前必须判断：

1. 用户目标是否明确；
2. 是否需要多个节点；
3. 每个节点是否有独立职责；
4. 节点之间是否存在真实依赖；
5. 是否存在可并行执行的任务；
6. 是否需要条件分支；
7. 是否需要循环；
8. 是否需要代码节点；
9. 每个节点产物是否被后续使用；
10. 最终输出是否满足用户目标。

### 4.2 默认简单优先

默认优先使用最简单可闭环结构：

```text
start -> agent -> end
```

只有任务确实复杂时，才升级为多节点。

### 4.3 不为了复杂而复杂

禁止生成以下工作流：

1. 为了凑节点而拆节点；
2. 多个节点重复做同一件事；
3. 下游节点不使用上游产物；
4. 条件分支没有真实判断依据；
5. 循环没有退出条件；
6. 汇总节点只是简单拼接；
7. 验收标准只有“非空”，但任务明显复杂；
8. 只有 YAML 合法，但业务目标无法完成。

---

## 5. 用户意图与工作流类型判断

### 5.1 单节点任务

满足以下条件时，优先使用单个 Agent 节点：

- 任务目标简单；
- 一个 ADE 可以完成；
- 不需要中间产物；
- 不需要多人协作；
- 不需要条件判断；
- 不需要代码处理；
- 不需要循环校验。

结构：

```text
start -> agent -> end
```

### 5.2 线性多节点任务

任务天然分阶段时，使用线性结构。

示例：

```text
start -> ADE-需求分析 -> ADE-PRD编写 -> ADE-质量评审 -> end
```

适用场景：

- 需求分析到文档输出；
- 技术方案到代码实现；
- 资料收集到汇总报告；
- 代码修改到测试验证。

### 5.3 并行任务

多个子任务互不依赖时，可以并行。

示例：

```text
start -> ADE-产品评审
start -> ADE-技术评审
start -> ADE-测试评审
ADE-产品评审 -> ADE-综合汇总
ADE-技术评审 -> ADE-综合汇总
ADE-测试评审 -> ADE-综合汇总
ADE-综合汇总 -> end
```

并行后必须有汇总节点。汇总节点必须进行去重、归类、冲突识别和统一结论输出，不能只是拼接文本。

### 5.4 条件分支任务

只有执行路径依赖明确判断结果时，才使用 `if-else` 节点。

适用情况：

- 风险等级不同，走不同处理路径；
- 评审通过或失败，走不同路径；
- 输入类型不同，走不同处理链路；
- 检查结果决定是否返工。

条件分支必须同时有 true / false 去向，并最终收敛到 end 或汇总节点。

### 5.5 循环任务

只有需要“生成 -> 检查 -> 修正 -> 再检查”的质量闭环时，才使用循环。

循环必须包含：

1. 最大循环次数；
2. 退出条件；
3. 验收标准；
4. 达到最大次数后的失败说明。

循环最大次数由模型根据任务复杂度自行判断，但不得超过系统支持上限 10 次。

推荐策略：

1. 简单质量检查：1-2 次；
2. 常规生成、检查、修正闭环：2-3 次；
3. 复杂文档、代码修复、多轮评审：3-5 次；
4. 高风险、强质量要求、容易反复返工的任务：可设置 6-10 次，但必须在任务描述或输出说明中解释较高轮次的必要性。

不得无脑使用 10 次。无论设置几次，都必须有明确退出条件、每轮检查依据、达到最大次数后的失败说明或人工介入路径。

### 5.6 Code 节点任务

只有确定性处理才使用 code 节点。

适用情况：

- 文件存在性检查；
- Markdown / JSON / CSV 解析；
- 字数、数量、比例统计；
- 格式转换；
- 规则校验；
- 批量处理。

禁止把开放式推理、分析、写作、决策放进 code 节点。

---

## 6. 节点设计约束

### 6.1 start 节点

start 节点必须唯一。它只负责定义输入变量，不执行任务。

变量设计要求：

1. 变量名语义清晰；
2. 必填变量设置 `required: true`；
3. 不定义后续不用的变量；
4. 长文本使用 `paragraph`；
5. 文件路径使用 `text-input` 或系统支持的文件类型；
6. 后续必须通过 `{{ref.start.<variable>}}` 引用。

### 6.2 agent 节点

agent 节点用于 ADE 执行需要理解、分析、生成、评审、修改的任务。

每个 agent 节点必须包含：

1. 清晰 title；
2. `command_config.task_desc`；
3. `command_config.user_message`；
4. `verification_config.task_desc`；
5. `verification_config.user_message`；
6. `_advanced.id`；
7. `_advanced.style`；
8. `_advanced.data`；
9. `output_variables`。

任务描述必须包含：

1. 角色定位；
2. 输入信息；
3. 任务步骤；
4. 输出要求；
5. 输出变量赋值方式；
6. 无法完成时的失败说明。

如果 agent 节点需要调用已有技能，必须优先使用平台原生 `/技能名` 方式，而不是用自然语言描述“请加载某技能”。

推荐写法：

```yaml
task_desc: |-
  使用 /lightweight-prd 技能，将以下 PRD 原文改写为轻量化 PRD。
  严格按照 /lightweight-prd 技能规定的文档结构和写作原则输出。
user_message:
  - type: text
    text: |-
      /lightweight-prd

      请将以下 PRD 原文改写为轻量化 PRD。
```

禁止写法：

```yaml
task_desc: |-
  请先加载 lightweight-prd 技能，然后严格按照该技能的文档结构输出。
```

原因：`/技能名` 是平台原生技能调用机制，可以直接注入技能完整指令；自然语言描述依赖 Agent 自行理解和执行，约束力更弱。

### 6.3 code 节点

code 节点必须包含：

1. 明确函数入口；
2. 明确输入参数；
3. 明确输出字段；
4. 输入校验；
5. 异常处理；
6. 结构化返回 dict；
7. `_advanced.data.inputs`；
8. `_advanced.data.outputs`；
9. `_advanced.data.variables`。

如果 code 节点依赖上游文件路径，必须先检查文件是否存在。

### 6.4 if-else 节点

if-else 节点必须基于上游变量判断。

每个条件必须满足：

1. 判断变量存在；
2. 判断操作符明确；
3. true 分支有目标；
4. false 分支有目标；
5. 所有分支最终收敛；
6. 条件与业务目标相关。

### 6.5 end 节点

end 节点必须唯一。它只负责输出最终结果。

end 节点必须引用真实存在的上游输出变量。不要堆砌无意义中间字段。

---

## 7. ADE 与 Workspace 绑定约束

### 7.1 直接生成待绑定 YAML

即使用户没有提供 ADE / Workspace，也必须直接生成 YAML。

Agent 和 code 节点中的绑定字段留空：

```yaml
ade_id: ""
ade_name: ""
ade_role: ""
workspace_id: ""
workspace_name: ""
```

用户会在 Web 前端图形化界面中选择具体 ADE 和 Workspace。

### 7.2 不允许编造绑定信息

禁止使用：

1. 示例 ID；
2. 模板 ID；
3. 猜测 ID；
4. 历史上下文中的 ID；
5. 与当前用户无关的 ADE / Workspace；
6. 伪造名称。

只有用户明确提供完整真实信息时，才可以写入实际值。

### 7.3 建议角色写法

编排阶段可以判断建议角色，但不要把建议角色写进未确认的 `ade_role` 字段。

推荐把建议角色写在 YAML 注释或 task_desc 中：

```yaml
# 建议绑定 ProductManager 角色的 ADE Worker。
title: ADE-需求分析
```

不推荐写入 title：

```yaml
title: ADE-需求分析 [建议角色: ProductManager]
```

这容易触发 YAML 解析错误，也会让前端显示变乱。

### 7.4 角色匹配规则

建议角色应按任务性质判断：

- `ProductManager`：需求分析、PRD、用户场景、业务流程、验收口径；
- `DevelopManager`：技术方案、研发拆分、架构影响、代码评审、研发风险；
- `Developer` / `SeniorDeveloper`：代码实现、Bug 修复、接口联调、脚本处理；
- `Tester`：测试用例、测试执行、缺陷复现、回归验证、测试报告。

如果无法判断，优先在 task_desc 中说明“建议选择熟悉该任务上下文的 ADE”。

### 7.5 Workspace 选择提示

如果任务依赖文件或代码仓库，应在注释或任务描述中提醒用户选择可访问相关材料的 Workspace。

推荐写法：

```yaml
# 建议选择可访问 PRD 文件和输出目录的 Workspace。
```

不要新增非 Schema 字段，例如：

```yaml
binding_status: unbound
required_workspace_hint: xxx
required_ade_role: xxx
```

除非已确认 ADE Teams Schema 支持这些字段。

---

## 8. 变量流转约束

### 8.1 变量声明

所有被引用的变量必须先声明。

允许引用来源：

1. start 节点变量；
2. agent 节点 `output_variables`；
3. code 节点 `outputs`；
4. if-else 判断变量；
5. end 节点输出引用。

禁止引用不存在的变量。

### 8.2 变量命名

变量名使用英文小写、数字和下划线。

推荐：

```text
requirement_doc_path
risk_level
test_report_path
review_result
summary
implementation_plan_path
```

不推荐：

```text
result
data
text
file
output
aaa
tmp
```

如果是文件路径变量，必须以 `_path` 结尾。

### 8.3 变量引用语法

引用上游变量统一使用：

```text
{{ref.<node_id>.<variable_name>}}
```

当前节点输出赋值统一使用：

```text
{{out.<variable_name>}}
```

禁止混用：

```text
{{start.query}}
{{ade_1.result}}
{{global.var1}}
```

除非已确认当前系统支持这些语法。

### 8.4 value_selector

在 code 节点变量映射和 end 节点输出中使用：

```yaml
value_selector:
  - <node_id>
  - <variable_name>
```

value_selector 必须指向真实存在的节点和变量。

---

## 9. 验收标准约束

每个 agent 节点必须有 `verification_config`。

### 9.1 基础验收

简单任务可以使用：

```text
{{out.<variable>}} 不为空
```

但复杂任务不能只检查非空。

### 9.2 文档类任务验收

文档类任务应检查：

1. 文件路径不为空；
2. 文件已生成；
3. 文件格式正确；
4. 必要章节齐全；
5. 覆盖用户指定主题；
6. 不包含明显占位符。

示例：

```text
{{out.prd_doc_path}} 不为空，且文件已生成，且文档包含背景、目标、范围、功能需求、异常场景、验收标准章节。
```

### 9.3 代码类任务验收

代码类任务应检查：

1. 修改文件不为空；
2. 代码已保存；
3. 未引入无关改动；
4. 测试通过，或说明无法执行测试的原因。

### 9.4 测试类任务验收

测试类任务应检查：

1. 测试用例或报告路径不为空；
2. 覆盖正常流程；
3. 覆盖异常流程；
4. 覆盖边界条件；
5. 输出明确结论。

### 9.5 评审类任务验收

评审类任务应检查：

1. 是否有通过/不通过结论；
2. 是否列出问题；
3. 是否标注风险等级；
4. 是否给出修改建议；
5. 是否说明是否可进入下一阶段。

---

## 10. 边与执行顺序约束

### 10.1 基础约束

每条 edge 必须满足：

1. source 节点存在；
2. target 节点存在；
3. source 与 target 不相同；
4. 非 start 节点至少有一个入边；
5. 非 end 节点至少有一个出边；
6. 不存在孤立节点；
7. 所有路径最终到达 end。

### 10.2 串行边

如果 target 依赖 source 输出，必须串行。

### 10.3 并行边

如果多个节点只依赖 start 输入，且互不依赖，可以并行。

并行后必须有汇总节点。

### 10.4 分支边

if-else 节点边必须使用明确 `sourceHandle`：

```yaml
sourceHandle: "true"
```

或：

```yaml
sourceHandle: "false"
```

普通边使用：

```yaml
sourceHandle: source
```

---

## 11. 安全与人工确认约束

涉及高风险动作时，不得默认自动执行，应拆成“生成建议 / 方案 / 命令”，由用户确认后再执行。

高风险动作包括：

1. 删除文件；
2. 修改核心配置；
3. 执行生产环境操作；
4. 调用外部服务；
5. 提交代码；
6. 发布内容；
7. 处理密钥、Token、API Key；
8. 涉及安全风险放行；
9. 涉及不可逆操作。

敏感信息处理要求：

1. 不输出 API Key、Token、密码明文；
2. 不把密钥写入文件；
3. 不写入日志；
4. 使用 `<API_KEY>` 等占位符；
5. 只说明配置位置和使用方式。

---

## 12. YAML 语法安全约束

生成的 YAML 必须能被标准 YAML 解析器解析。

### 12.1 字符串安全

字段值中如果包含以下字符，必须使用双引号包裹，或改写为不含特殊符号的普通文本：

- 冒号 `:`；
- 中括号 `[ ]`；
- 大括号 `{ }`；
- 井号 `#`；
- 逗号 `,`；
- 引号 `' "`；
- 竖线 `|`；
- 大于号 `>`；
- 星号 `*`；
- 与号 `&`；
- 感叹号 `!`；
- at 符号 `@`；
- 反引号；
- 以 `-`、`?`、`:` 开头的特殊字符串。

尤其要保护以下字段：

- `title`；
- `label`；
- `variable`；
- `id`；
- `source`；
- `target`；
- `sourceHandle`；
- `comparison_operator`。

### 12.2 title 字段约束

默认 title 只使用简洁文本，不放角色说明、冒号、中括号。

推荐：

```yaml
# 建议绑定 ProductManager 角色的 ADE Worker。
title: ADE-需求分析
```

不推荐：

```yaml
title: ADE-需求分析 [建议角色: ProductManager]
```

如果必须写特殊符号，必须加双引号：

```yaml
title: "ADE-需求分析 [建议角色: ProductManager]"
```

但默认不要这样写。

### 12.3 block scalar 使用

长任务描述必须使用：

```yaml
task_desc: |-
  第一行内容
  第二行内容
```

长文本不要写成单行，避免冒号、括号、Markdown 符号导致 YAML 解析失败。

### 12.4 注释安全

YAML 注释只能用于提示用户绑定角色或 Workspace，不得承载系统必须读取的关键字段。

注释示例：

```yaml
# 建议绑定 DevelopManager 角色的 ADE Worker。
```

---

## 13. 标准 YAML 模板

### 13.1 Start 节点

```yaml
nodes:
  - type: start
    title: 开始
    _advanced:
      id: start
      style: '{"x":40,"y":80}'
      data:
        variables:
          - type: text-input
            label: 任务主题
            required: true
            variable: topic
```

### 13.2 Agent 节点，待绑定

```yaml
  # 建议绑定 ProductManager 角色的 ADE Worker，并选择可访问输入材料的 Workspace。
  - type: agent
    title: ADE-需求分析
    data:
      command_config:
        task_desc: |-
          你是一位资深产品经理。请基于以下输入完成需求分析。

          【任务主题】{{ref.start.topic}}

          请完成以下任务：
          1. 梳理用户目标和业务背景。
          2. 拆解核心功能需求。
          3. 梳理异常场景和边界条件。
          4. 输出待确认问题。

          请将需求分析结果保存为 Markdown 文件，并将文件路径赋值给 {{out.requirement_doc_path}}。

          如果无法完成任务，请说明失败原因、缺失信息、已完成部分和建议下一步。
        user_message:
          - type: text
            text: |-
              你是一位资深产品经理。请基于以下输入完成需求分析。

              【任务主题】{{ref.start.topic}}

              请完成以下任务：
              1. 梳理用户目标和业务背景。
              2. 拆解核心功能需求。
              3. 梳理异常场景和边界条件。
              4. 输出待确认问题。

              请将需求分析结果保存为 Markdown 文件，并将文件路径赋值给 {{out.requirement_doc_path}}。

              如果无法完成任务，请说明失败原因、缺失信息、已完成部分和建议下一步。
        user_scenario: default
      verification_config:
        task_desc: "{{out.requirement_doc_path}} 不为空，且文件已生成，且文档包含业务背景、核心需求、异常场景、边界条件和待确认问题。"
        user_message:
          - type: text
            text: "{{out.requirement_doc_path}} 不为空，且文件已生成，且文档包含业务背景、核心需求、异常场景、边界条件和待确认问题。"
        user_scenario: default
      variable_assignments: []
      loop_break_conditions: []
      loop_logical_operator: and
      loop_validation_enabled: false
      loop_max_count: 10
    _advanced:
      id: ade_1
      style: '{"x":280,"y":80}'
      data:
        ade_id: ""
        ade_name: ""
        ade_role: ""
        workspace_id: ""
        workspace_name: ""
        output_variables:
          - type: string
            label: 需求分析文件路径
            variable: requirement_doc_path
```

### 13.3 Code 节点，待绑定

```yaml
  # 建议绑定 Developer 角色的 ADE Worker，并选择可访问上游文件的 Workspace。
  - type: code
    title: 代码-文件检查
    data:
      code: |-
        def main(file_path: str) -> dict:
            import os

            exists = bool(file_path) and os.path.exists(file_path)
            return {
                "file_exists": exists,
                "checked_path": file_path,
                "message": "文件存在" if exists else "文件不存在或路径为空"
            }
    _advanced:
      id: code_1
      style: '{"x":520,"y":80}'
      data:
        ade_id: ""
        ade_name: ""
        ade_role: ""
        workspace_id: ""
        workspace_name: ""
        code_language: python3
        inputs:
          file_path:
            type: string
            required: true
        outputs:
          file_exists:
            type: boolean
          checked_path:
            type: string
          message:
            type: string
        variables:
          - id: var-file-path
            variable: file_path
            value_selector:
              - ade_1
              - requirement_doc_path
```

### 13.4 End 节点

```yaml
  - type: end
    title: 结束
    _advanced:
      id: end
      style: '{"x":760,"y":80}'
      data:
        outputs:
          - id: out-requirement-doc
            variable: 需求分析文件路径
            value_selector:
              - ade_1
              - requirement_doc_path
```

### 13.5 Edges

```yaml
edges:
  - source: start
    target: ade_1
    sourceHandle: source
    _advanced:
      id: edge_1
  - source: ade_1
    target: end
    sourceHandle: source
    _advanced:
      id: edge_2
```

---

## 14. 默认编排模板

默认模板只用于编排判断，不允许机械套用。生成 YAML 前必须先根据用户目标选择或调整模板。

### 14.1 简单单节点

```text
start -> ADE-执行任务 -> end
```

适用：

- 摘要、简单文案、单轮分析；
- 单个 ADE 可以闭环完成；
- 不需要中间产物、评审、分支或循环。

### 14.2 线性生产

```text
start -> ADE-资料分析 -> ADE-产物生成 -> ADE-质量检查 -> end
```

适用：

- PRD、方案、手册、会议材料；
- 技术方案、实现计划、测试计划；
- 上游产物会被下游明确使用。

要求：

1. 每个节点必须消费上游输出；
2. 质量检查节点必须给出通过/不通过结论；
3. end 只输出最终有效产物，不堆砌中间字段。

### 14.3 并行评审

```text
start -> ADE-产品评审
start -> ADE-技术评审
start -> ADE-测试评审
ADE-产品评审 -> ADE-综合汇总
ADE-技术评审 -> ADE-综合汇总
ADE-测试评审 -> ADE-综合汇总
ADE-综合汇总 -> end
```

适用：

- 方案评审、上线前检查、可行性分析；
- 多角色从不同视角独立判断；
- 需要冲突识别和统一结论。

要求：

1. 并行节点必须互不依赖；
2. 汇总节点必须去重、归类、识别冲突、输出统一结论；
3. 不允许只拼接多个评审结果。

### 14.4 质量闭环

```text
start -> ADE-生成产物 -> ADE-质量检查 -> if-else
if pass -> end
if fail -> ADE-修正产物 -> ADE-质量检查
```

适用：

- 文档质量检查；
- 代码修复和测试失败重试；
- 安全审查整改；
- 需要多轮生成、检查、修正的任务。

要求：

1. 循环最大次数不得超过 10；
2. 循环次数由模型按复杂度判断；
3. 必须有退出条件；
4. 达到最大次数后必须输出失败说明或转人工处理。

### 14.5 受控高风险执行

```text
start -> ADE-风险分析 -> ADE-生成执行方案 -> ADE-人工确认材料 -> if-else
if approved -> ADE-执行或生成命令 -> ADE-结果验证 -> end
if rejected -> end
```

适用：

- 删除文件、修改核心配置、生产环境操作；
- 提交代码、发布内容、调用外部服务；
- 涉及密钥、权限、安全放行或不可逆动作。

要求：

1. 高风险动作不得默认自动执行；
2. 必须先生成方案、影响范围、回滚方式和人工确认材料；
3. 未获得明确确认时，只能输出建议或命令草案；
4. 执行后必须有结果验证节点。

更多模板细则见 `references/workflow-patterns.md`。

---

## 15. 输出要求

### 15.1 默认输出

默认直接输出完整 YAML。

如果是在文件系统环境中执行，应将 YAML 保存为：

```text
workflow-<主题关键词>-<日期>.yml
```

并返回文件路径。

### 15.2 输出说明

除非用户明确只要 YAML，否则输出 YAML 后应简要说明：

1. 节点数量；
2. 执行顺序；
3. 哪些节点需要在前端绑定 ADE 和 Workspace；
4. 是否包含分支、循环、并行、code 节点；
5. 需要用户手动确认的事项。

### 15.3 不确定项处理

如果存在不确定项，必须明确列出，但不能隐藏在 YAML 中。

例如：

```text
以下字段已留空，需在 Web 前端选择后补齐：ade_id、ade_name、ade_role、workspace_id、workspace_name。
```

---

## 16. 生成前自检清单

输出 YAML 前必须检查：

### 16.1 结构自检

1. 只有一个 start；
2. 只有一个 end；
3. 所有节点 id 唯一；
4. 所有节点 title 简洁且 YAML 安全；
5. 所有节点 type 合法；
6. 非 start 节点都有入边；
7. 非 end 节点都有出边；
8. 所有路径最终到达 end；
9. 无孤立节点。

### 16.2 变量自检

1. 所有 `{{ref.node.variable}}` 指向真实变量；
2. 所有 `{{out.variable}}` 已在 output_variables 或 outputs 中声明；
3. 所有 value_selector 指向真实节点和变量；
4. code 节点 inputs 都有 variables 映射；
5. end 节点 outputs 都引用真实上游变量；
6. 文件路径变量以 `_path` 结尾。

### 16.3 技能调用自检

1. 需要复用已有技能时，优先使用 `/技能名` 直接调用；
2. 不用自然语言描述“请加载某技能”替代 `/技能名`；
3. `task_desc` 和 `user_message` 中的技能调用方式一致；
4. `workflow_intent.required_skills` 与节点实际调用的技能一致；
5. 不编造不存在或未确认的技能名。

### 16.4 编排自检

1. 节点拆分必要；
2. 无重复节点；
3. 无错误串行或错误并行；
4. 分支有明确判断依据；
5. 循环有退出条件，最大次数不超过 10 次，并与任务复杂度匹配；
6. 并行后有汇总节点；
7. 高风险动作有人工确认；
8. ADE 建议角色与任务匹配。

### 16.5 绑定自检

1. 用户提供真实 ADE / Workspace 时，才写入真实值；
2. 用户未提供时，绑定字段使用空字符串；
3. 不出现示例 ID；
4. 不出现伪造 ID；
5. 不新增未确认的 Schema 字段；
6. 绑定建议只放在注释或 task_desc 中。

### 16.6 YAML 语法自检

1. title 不含未加引号的冒号、中括号、大括号；
2. label 不含未加引号的特殊字符；
3. 长文本使用 `|-`；
4. 缩进统一使用两个空格；
5. 列表项 `-` 缩进正确；
6. 字符串中需要保留特殊符号时使用双引号；
7. 不在 YAML 中嵌入 Markdown 代码围栏。

---

## 17. 禁止事项

禁止生成：

1. YAML 语法不合法的内容；
2. 引用不存在变量的 YAML；
3. 没有 end 的工作流；
4. 孤立节点；
5. 无出口分支；
6. 无退出循环；
7. 伪造 ADE / Workspace 绑定信息；
8. 角色建议写入未确认的 `ade_role`；
9. 使用模板里的真实 ID 冒充当前用户 ID；
10. 用自然语言描述“请加载某技能”替代平台原生 `/技能名` 调用；
11. 包含敏感明文的工作流；
12. 自动执行高风险操作的工作流；
13. title 中包含未加引号的 `:`、`[`、`]`；
14. 只保证 YAML 能保存，但业务流程无法闭环的工作流。

---

## 18. 最终质量标准

一个合格的输出必须同时满足：

1. YAML 语法合法；
2. ADE Teams 结构尽量符合当前模板；
3. 直接给出 YAML；
4. 未绑定字段留空，不编造；
5. 节点职责清晰；
6. 执行顺序合理；
7. 变量流转闭环；
8. 验收标准有效；
9. 高风险动作受控；
10. 需要复用已有技能时，使用 `/技能名` 直接调用；
11. 用户能在前端补齐 ADE / Workspace 后保存和运行。
