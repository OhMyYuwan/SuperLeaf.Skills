---
name: skill-optimizer
description: 在 SuperLeaf Skill Project 中选择一个 Data Project 作为外部优化数据源，分析 labeled_samples、records、responses 中的失败模式和好坏样例，并优化当前 Skill Project 的 SKILL.md、references 和 evals。只要用户提到用 Data Project、标注数据、训练候选、失败样例、好样例、评估结果来改进 Skill，就应该使用这个 Skill。
---

# Skill Optimizer

使用这个 Skill 时，你的任务不是“训练模型”，也不是把数据集打包进 Skill。你的任务是在 Skill Project 中选择一个 Data Project 作为外部数据源，把其中持续积累的标注信号转化为更好的 Skill 指令、脱敏参考材料和评估用例。

你通常在一个 **Skill Project** 中工作。目标文件一般包括：

- `SKILL.md`
- `README.md`
- `references/failure-patterns.md`
- `references/golden-examples.md`
- `references/negative-examples.md`
- `evals/data-project-evals.jsonl`
- `references/optimization-brief.md`

不要把原始 Data Project 导出包、`records.jsonl`、`responses.jsonl` 或 `labeled_samples.jsonl` 写入公开 Skill 包或 Skill Project 缓存。Skill 公开时应该只包含泛化后的规则、脱敏样例、brief 和 eval，不携带原始 datasets。

## 适用场景

在这些情况下使用本 Skill：

- 用户希望在 Skill Project 里选择某个 Data Project 来优化当前 Skill。
- 用户给出 Data Project 名称、数据源选择、`labeled_samples`、`records`、`responses` 或导出包作为临时分析输入。
- 用户说要根据失败样例、好样例、标注结果、训练候选、issue 分布来优化某个 Skill。
- 用户希望 Agent 根据某个 Skill、Agent 或 Workflow 的历史表现改写 `SKILL.md`。
- 用户要建立 Skill Project 主动拉取 Data Project 评估信号的闭环。

## 输入语义

优先读取用户在 Skill Project 中选择的 Data Project。系统如果提供结构化数据，应按以下优先级使用：

1. `labeled_samples.jsonl`
2. `records.jsonl`
3. `responses.jsonl`
4. `manifest.json`
5. 已有的 `SKILL.md`
6. 已有的 `references/` 与 `evals/`

这些数据是优化输入，不是 Skill 内容。除非用户明确要求导出审计材料，否则不要把原始 JSONL 复制到 Skill Project 文件树。

常见字段含义：

- `fields.source_text`：被处理的原始文本快照。
- `fields.agent_output`：Agent 当时输出的结果。
- `fields.chat`：多轮上下文或 trace 转换后的对话。
- `fields.trace`：Workflow / Agent 执行轨迹。
- `response.values.task_success`：任务是否成功。
- `response.values.helpfulness`：用户或标注者给出的帮助程度。
- `response.values.issues`：失败原因或问题标签。
- `response.values.comments`：标注者给出的解释。
- `response.values.training_candidate`：是否适合进入优化集。
- `provenance.skill_ids`：相关 Skill。
- `provenance.agent_ids`：相关 Agent。
- `provenance.workflow_id` / `workflow_definition_id`：相关 Workflow。

如果只有 `records` 而没有 `responses`，先判断记录里是否已经嵌入 `response`。如果没有明确标注，不要把它当作训练信号，只能作为待评估样本。

## 工作流程

### 1. 明确优化对象

先确认当前要优化的是哪个 Skill：

- 如果用户已经在一个 Skill Project 中，默认优化当前项目的 `SKILL.md`。
- 如果系统提供 Data Project 选择器，先让用户选择一个 Data Project；不要要求用户先把 Data Project 送到 Skill Project。
- 如果数据中有多个 `skill_ids`，先按 Skill 分组，找出目标 Skill。
- 如果用户只说“优化这个 Skill”，不要同时改多个 Skill。
- 如果数据同时涉及 Agent 和 Workflow，把它们作为上下文来源，不要把 Agent 私有偏好硬写进通用 Skill，除非用户明确要求。

### 2. 做数据体检

先产出一个简短的数据体检，不要急着改 `SKILL.md`。

统计：

- 样本总数。
- 已提交响应数。
- `training_candidate` 数量。
- 成功 / 部分成功 / 失败分布。
- 高频 `issues`。
- 高频失败场景。
- 最有代表性的 3 到 8 个好样例。
- 最有代表性的 3 到 8 个坏样例。

优先使用高信息量样本：

- `training_candidate` 为 yes / true 的样本。
- `task_success` 为 fail 或 partial 且有明确 comments 的样本。
- 用户指出了具体问题的样本。
- 高分样本中可复用的稳定行为。

降低权重：

- 没有 response 的样本。
- 只有笼统评价、没有 source_text 或 agent_output 的样本。
- 与目标 Skill 无关的样本。
- 标注相互矛盾且没有解释的样本。

### 3. 归纳失败模式

把失败归纳成可行动的模式，而不是样本流水账。

推荐分类：

- 触发失败：Skill 没有在该用时被使用，或使用时机不对。
- 输入理解失败：没有正确读取 `source_text`、上下文、trace 或用户目标。
- 推理流程失败：跳过关键检查步骤，或顺序错误。
- 输出格式失败：没有遵守 JSON、Markdown、结构化段落或字段约束。
- 证据失败：没有引用原文，没有指出证据缺口，或凭空推断。
- 过度泛化：给出泛泛建议，无法落到当前样本。
- Workflow 协作失败：多个 Agent / Workflow 节点之间信息交接不清。
- 安全边界失败：试图访问不存在的数据、假装读取文件、或混淆运行时能力。

每个失败模式都要写成：

```md
### [失败模式名称]
- 证据：来自哪些样本或 issue。
- 原因：当前 Skill 指令哪里不足。
- 修改方向：应该补充哪条触发条件、步骤、约束或反例。
```

### 4. 提取好样例和反例

好样例不是越多越好。选择能教会 Skill 行为差异的样例。

每个好样例写成：

```md
### Golden Example: [短标题]
- Source: 简短 source_text 摘要。
- Good behavior: Agent 做对了什么。
- Skill lesson: 应该固化到 SKILL.md 的规则。
```

每个反例写成：

```md
### Negative Example: [短标题]
- Source: 简短 source_text 摘要。
- Bad behavior: Agent 做错了什么。
- Correction: 下次应该怎样做。
- Skill lesson: 应该加入的禁止项或检查项。
```

不要把大量原始隐私内容直接写进 `SKILL.md`。长样例如果必须保留，也要先脱敏、压缩，再放入 `references/`。`SKILL.md` 只引用结论和小片段。

### 5. 修改 SKILL.md

修改 `SKILL.md` 时遵循这些原则：

- 保留原有身份、适用场景和核心能力。
- 只加入从数据中反复出现、可泛化的规则。
- 把“偶发样本”写进 references，不要写成硬规则。
- 把失败模式转化为触发条件、步骤、检查清单或输出约束。
- 不要把某个 Agent 的临时偏好当成 Skill 的永久规则。
- 不要为了提高某批样本表现而破坏 Skill 的通用性。

推荐加入这些章节：

```md
## When to Use
## Inputs to Inspect
## Workflow
## Output Contract
## Quality Checks
## Failure Modes to Avoid
## Examples
```

如果原 Skill 已有类似章节，就在原结构内增量修改，不要整篇重写。

### 6. 生成 optimization-brief.md

每次优化都应该写一个 brief，但 brief 只能记录数据摘要、结论和脱敏证据，不保存原始数据集。推荐保存为：

`references/optimization-brief.md`

使用这个结构：

```md
# Skill Optimization Brief

## Dataset
- Data Project:
- Data source mode: selected Data Project / temporary export / manual JSONL
- Target Skill:
- Sample count:
- Submitted count:
- Training candidates:

## Main Findings

## Issue Distribution

## Failure Patterns

## Golden Examples

## Negative Examples

## Changes Applied to SKILL.md

## Suggested Evals

## Remaining Risks
```

不要创建 `datasets/` 目录来长期保存原始 Data Project 数据。公开 Skill 或更新 Skill 缓存前，确认 brief 不含大段原始 source_text、完整 chat、完整 trace 或用户隐私内容。

### 7. 生成评估用例

如果用户希望以后持续验证 Skill，生成：

`evals/data-project-evals.jsonl`

每行一个 JSON 对象：

```json
{"id":"dp-001","source_text":"...","task":"...","expected_behavior":["..."],"avoid":["..."],"tags":["missing_context"]}
```

评估用例应该来自真实数据，但要脱敏、压缩、去掉无关上下文。它们是“代表性测试”，不是原始数据集副本。

## 输出方式

完成后向用户报告：

1. 读了哪些数据文件。
1. 选择了哪个 Data Project。
2. 目标 Skill 是什么。
3. 发现了哪些主要失败模式。
4. 修改了哪些文件。
5. 哪些规则来自高置信数据，哪些只是建议。
6. 是否需要用户进入 **版本 -> 项目大版本 -> 更新 Skill 缓存**。

如果你已经修改了 Skill Project 文件，最后提醒：

> 这些修改还没有进入 Agent runtime。请在 Skill Project 的“版本 -> 项目大版本 -> Skill 缓存”中点击“更新 Skill 缓存”，然后再用新的 Data Project 批次验证效果。

## 边界

- 不要直接声明模型已经被训练。
- 不要把所有样本塞进 `SKILL.md`。
- 不要把原始 datasets 放进公开 Skill 包。
- 不要把 Data Project 作为文件长期复制到 Skill Project；Skill Project 应该选择 Data Project 作为外部优化源。
- 不要假装能读取数据库；只能读取用户提供或项目中存在的文件。
- 不要把未标注数据当成高置信优化依据。
- 不要删除旧规则，除非数据明确证明它造成系统性错误。
- 不要修改与目标 Skill 无关的 Agent、Workflow 或项目配置。
