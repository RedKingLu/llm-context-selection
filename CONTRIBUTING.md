# 参与指南 · Contributing

感谢你的兴趣！这是一篇用于引发讨论的探索性笔记，欢迎以下形式的参与。
Thanks for your interest! This is an exploratory note meant to spark discussion. Contributions of the following kinds are welcome.

## 💬 观点讨论 · Ideas

在 Discussions 中发帖：赞成、反对、补充、指出漏洞都欢迎。请尽量说明理由或给出依据。
Post in Discussions: agreement, disagreement, additions, and holes you spot are all welcome. Please give reasons or evidence where possible.

## 🧪 分享实验 · Sharing experiments

正反结论都有价值。建议按以下模板发布：
Both positive and negative results are valuable. Suggested template:

```markdown
### 任务 / Task
（任务类型、数量、平均轮数 / task type, count, average turns）

### 设置 / Setup
- 生成模型 / Generator model:
- 选择器 / Selector: （向量检索 / LLM / Jev / 本地小模型 …）
- 对比方案 / Baselines: （A 全量+压缩 / B 选择 / B' 同等长度 …）
- 上下文预算 / Context budget:

### 结果 / Results
| 方案 | 成功率 | 约束违反率 | 实际成本（含缓存） | 缓存命中率 | 首字延迟 |
|---|---|---|---|---|---|

### 观察 / Observations
（意外发现、失败案例 / surprises, failure cases）

### 复现 / Reproduction
（代码或数据链接，如可公开 / link to code or data, if shareable）
```

## 📚 文献与勘误 · References and errata

发现相关工作、引用错误或事实错误，请提 Issue；直接修改请提 PR，并说明依据（论文链接、文档链接等）。中英文版本请尽量同步修改。
Found related work, a wrong citation, or a factual error? Open an Issue, or a PR with sources (paper or documentation links). Please keep the Chinese and English versions in sync when possible.

## 行为准则 · Code of conduct

就事论事，友善交流。
Be kind, and argue about ideas, not people.
