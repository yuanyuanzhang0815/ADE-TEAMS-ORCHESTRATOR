# Skill Invocation

当 workflow 节点需要复用已有技能时，必须优先使用平台原生 `/技能名` 调用。

## 推荐方式

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

## 禁止方式

```yaml
task_desc: |-
  请先加载 lightweight-prd 技能，然后严格按照该技能的文档结构输出。
```

## 原因

1. `/技能名` 是平台原生技能调用机制。
2. 平台可以直接注入技能完整指令。
3. 技能格式约束会比自然语言描述更稳定。
4. 自然语言“请加载技能”依赖 Agent 自行理解，容易漏执行或执行不完整。

## 自检

生成 YAML 前检查：

1. 节点是否需要复用已有技能。
2. 技能名是否真实存在。
3. `task_desc` 是否使用 `/技能名`。
4. `user_message` 是否显式包含 `/技能名`。
5. 验收标准是否覆盖该技能的格式要求。
