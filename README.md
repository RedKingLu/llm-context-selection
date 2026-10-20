# 让大模型自己挑选上下文 · Letting LLMs Choose Their Own Context

> 一份探索性笔记：如果让模型（或一个专门的判断模型）自己决定每一轮看哪些上下文，会不会更准？会遇到哪些问题？
>
> An exploratory note: what if the model (or a dedicated decision model) chose which context to look at on each turn? Would it be more accurate, and what problems would arise?

📄 **正文 / Paper**：[中文](paper.zh.md) · [English](paper.en.md)　｜　讨论稿 / Discussion draft **v0.3**

⚠️ 本文为未经同行评审的个人探索性观点，文中判断大多是推测，引用数据未经作者复现。详见 [DISCLAIMER.md](DISCLAIMER.md)。
This is a personal, non-peer-reviewed exploratory essay; most claims are conjectures and quoted numbers have not been reproduced. See [DISCLAIMER.md](DISCLAIMER.md).

---

## 这篇文章讨论什么

- **核心问题**：让模型按"当前需要什么"、而不是按"时间顺序"组装上下文，能否提高准确率？
- **不只是省 token**：即使上下文长度不变，把窗口填满"真正需要的内容"也可能让模型更聚焦。
- **和记忆机制的区别**：记忆机制解决"记住什么、怎么找回来"（长期记忆），本文关心"这一轮把什么摆到模型眼前、怎么摆"（工作记忆与注意力）。
- **几条实现路线**：模型自管理、选择模型（如 Jev）+ 生成模型、压缩、服务端分段缓存。
- **会遇到的问题**：前缀缓存失效、漏选导致静默违约、选择器延迟、切分粒度、评测困难等，以及缓解思路。
- **开放问题清单** 和 **一个周末就能做的最小实验**。

## What this essay covers

- **Core question**: does assembling context by "what is needed now" instead of "chronological order" improve accuracy?
- **Not just saving tokens**: even at the same context length, filling the window with what is actually needed may sharpen the model's focus.
- **Different from memory mechanisms**: memory is about what to remember and how to retrieve it (long-term memory); this essay is about what to put in front of the model on this turn and how (working memory and attention).
- **Implementation routes**: self-management, selector model (e.g., Jev) + generator, compression, server-side segment caching.
- **Problems**: prefix-cache invalidation, silent violations from missed segments, selector latency, segmentation, evaluation, plus mitigation ideas.
- **Open problems** and **a minimal weekend experiment**.

## 参与讨论 · Get involved

- 💬 **观点讨论 / Ideas**：到 [Discussions](../../discussions) 发帖，赞成、反对、补充都欢迎
- 🧪 **实验结果 / Experiments**：按正文第 7 节做了实验？欢迎分享，正反结论都有价值
- 📚 **文献与勘误 / References & errata**：提 Issue 或 PR

详见 / See [CONTRIBUTING.md](CONTRIBUTING.md)。

## 引用 · Citation

见 [CITATION.cff](CITATION.cff)，或使用 / or use:

```bibtex
@misc{redkinglu2026contextselection,
  author       = {RedKingLu},
  title        = {Letting LLMs Choose Their Own Context: Questions, Approaches, and Open Problems},
  year         = {2026},
  note         = {Discussion draft v0.3},
  howpublished = {\url{https://github.com/RedKingLu/llm-context-selection}}
}
```

## 许可 · License

[CC BY 4.0](LICENSE)：可自由转载、改编和商用，请注明作者与出处。
Free to share and adapt, including commercially, with attribution.
