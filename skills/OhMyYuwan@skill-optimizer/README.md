# Skill Optimizer

中文版本的 SuperLeaf Skill 优化 Skill。

它用于在 Skill Project 中选择一个 Data Project 作为外部优化数据源，把其中持续积累的标注信号转化为可维护的 Skill 改进：

- 分析选定 Data Project 的 `labeled_samples`、`records`、`responses` 和 `manifest`。
- 识别目标 Skill 的失败模式、高质量样例和反例。
- 修改 Skill Project 中的 `SKILL.md`。
- 生成 `references/failure-patterns.md`、`references/golden-examples.md`、`references/negative-examples.md` 等辅助材料。
- 生成 `references/optimization-brief.md` 和可选的 `evals/data-project-evals.jsonl`。

它不要求 Data Project 把数据送进 Skill Project，也不应该把原始 datasets 带进公开 Skill 包或 Skill 缓存。

## 推荐使用流程

1. 在 Data Project 中按 Skill / Agent / Workflow 筛选并标注数据。
2. 标注并提交有代表性的样本。
3. 在目标 Skill Project 中选择要使用的 Data Project。
4. 让装配了本 Skill 的 Agent 分析该 Data Project 并优化 `SKILL.md`。
5. 人工检查修改。
6. 在 Skill Project 的版本面板中点击 **更新 Skill 缓存**。
7. 用新的 Data Project 批次继续评估。

## 运行边界

这是 instruction-only Skill。它不会直接训练模型，不会自动更新 Skill 缓存，也不会把原始数据集发布进 Skill。它只指导 Agent 根据被选择的 Data Project 信号和当前项目文件修改 Skill Project。
