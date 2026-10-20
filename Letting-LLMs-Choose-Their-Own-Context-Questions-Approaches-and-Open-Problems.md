# Letting LLMs Choose Their Own Context: Questions, Approaches, and Open Problems

> Author: RedKingLu　|　Discussion draft v0.3　|　2026-10-20　|　[中文版](<让大模型自己挑选上下文：一份探索性笔记.md>)
>
> ⚠️ **Disclaimer**: This is a personal, exploratory essay. It has **not been peer-reviewed** and does not constitute technical, product, or investment advice. All numbers quoted come from the cited sources and have not been reproduced by the author. See [Section 9](#9-disclaimers) and [DISCLAIMER.md](DISCLAIMER.md) for the full statement.
>
> **Before you read**: This is not a validated, ready-to-use paper. It is a note that **raises questions and maps out ideas**. Most claims here are conjectures, and none of the proposed approaches has been systematically tested by me. The goal is to put a question I find worth discussing on the table and see whether others want to dig into it. Counterarguments, additions, and experimental results are all very welcome.
>
> *This English version is a translation of the Chinese original. If the two differ, the Chinese version reflects the author's latest intent.*

---

## Abstract

In today's LLM applications, what goes into the context on each turn is largely decided by rules the developer wrote in advance: the system prompt goes first, history is appended in order, and everything is compressed when the window fills up. A natural question follows: **if the model (or a dedicated model) decided for itself which context to look at on each turn, would results be noticeably better?**

Two clarifications up front. First, this question is related to, but different from, the "memory mechanisms" that have become popular recently (MemGPT, Mem0, A-MEM, etc.). Memory mechanisms mainly answer "**what to remember across sessions and how to retrieve it**"; this essay is about "**what to put in front of the model on this turn, and in what order**". The former resembles long-term memory; the latter is closer to working memory and attention. Second, the value of self-selection is **not just saving tokens**: even with exactly the same context length, filling the window with "what is actually needed now" instead of "whatever piled up in chronological order" may help the model focus on the core problem.

Building on this, the essay has four parts: (1) what existing evidence says, both signs that self-selection improves accuracy and reasons for skepticism; (2) several possible implementation routes, including self-management by the model, a two-stage "selector model + generator model" design (using the recently popular decision model Jev as an example), compression, and server-side segment caching, plus how these differ from RAG; (3) problems these routes will run into, the most overlooked being that **dynamic selection breaks prefix caching**, along with "silent violations" caused by missed segments, selector latency, segmentation granularity, and evaluation difficulty; (4) mitigation ideas for each problem. Every part ends with **directions you can try yourself**, and the essay closes with a list of open problems.

---

## 1 Where the Problem Comes From

### 1.1 How context is assembled today

For agents, the mainstream approach can be summarized as "**fixed skeleton + append-only + compress when full**":

```
[system prompt][tool definitions]   ← never changes
[turn 1][turn 2]...[turn n]          ← appended at the end every turn
[current input]
```

When the context approaches the window limit, the history is summarized once, and appending continues. RAG applications insert retrieved results at the end, but what to retrieve and how much is again decided by fixed rules (e.g., top-k by similarity).

This approach is simple and stable, and it has one inconspicuous but very important advantage: **the prefix barely changes, so the prompt cache almost always hits** (discussed in detail in Section 4.1).

### 1.2 What is wrong with it

In long tasks, the context fills up with content that is "not needed right now": tool outputs from many turns ago, rejected plans, contents of files that have since been modified. This has two consequences:

- **Noise**: irrelevant content interferes with the model. Research has found that information in the middle of long contexts is more easily overlooked [Liu et al., 2023].
- **What should stay gets compressed away**: compression decides by "old vs. new" rather than "useful vs. useless", so key constraints set early on may be weakened or lost in the summary.

Hence the question of this essay: **can context be assembled according to "what is needed now" rather than "chronological order"?**

### 1.3 How is this different from memory mechanisms?

Many readers' first reaction may be: "Isn't this just agent memory?" Indeed, many memory systems have appeared recently: MemGPT/Letta [Packer et al., 2023], Generative Agents [Park et al., 2023], Mem0 [Chhikara et al., 2025], A-MEM [Xu et al., 2025], MemoryOS [Kang et al., 2025], Jev-Mem (introduced later), and the "memory" features of commercial chat assistants. The two areas overlap, but they focus on different things.

An analogy: **memory mechanisms resemble human long-term memory; context selection resembles working memory and attention.** You remember many things (long-term memory), but when solving the problem in front of you, only a handful of things are "on your mind" at once (working memory). The former is about "can I remember it, can I recall it"; the latter is about "what should I be looking at right now".

| Dimension | Memory mechanisms | Context selection (this essay) |
|---|---|---|
| Core question | What to remember, how to store it, how to retrieve it | What to look at this turn, and in what order |
| Time scale | Across sessions and days; can be very long | Within the current task, on every turn |
| Main stages | Write → organize (classify, link, forget) → retrieve | Candidates → select → order → assemble |
| Candidate source | The memory store | All materials of the current task: dialogue, tool outputs, files, **including memory retrieval results** |
| Optimization goal | Accurate recall, efficient storage, no forgetting | Reasoning quality on this turn, focused attention, cache cost |
| Typical failure | Didn't remember what mattered, or can't find it | Attention diluted by irrelevant content, or a required segment missed |
| Relation to caching | Weak (retrieved results usually go at the end) | **Strong**, because the prompt is rewritten every turn (Section 4.1) |
| Without long-term memory | Nothing to do | Still needed: a single long task is enough to create the problem |

**What they share**: both pick a small subset from a large pool for the model, and both can be implemented with retrieval plus judgment. Jev-Mem's retrieval controller is essentially doing selection.

**How they relate**: context selection can be seen as the "**last mile**" of a memory system. The memory system brings back what might be useful; context selection decides how that material, together with everything else in the current task (the latest tool output, the file being edited, constraints set earlier), **fits into a limited window**. Most existing memory systems simply append retrieved results to the end of the prompt and pay little attention to the composition and order of the whole window, let alone caching.

**The boundary is blurry**: MemGPT's paging of its "main context" already embodies the idea of context selection. This essay deliberately treats the "assembly layer" on its own because it exists **even in a single long task with no long-term memory at all**, and it has its own distinct problems, such as cache invalidation and order dependence.

---

## 2 The Core Question: Does Self-Selected Context Improve Accuracy?

### 2.1 Supporting signs

- **"Put everything in" is not necessarily best.** Jev-Mem [Jiang et al., 2026] reports on the long-term conversational memory benchmark LoCoMo: feeding the full history directly to GPT-4o-mini scores 0.481 overall, while letting a decision model control memory writing and retrieval and passing only the selected evidence to the generator scores 0.777. These are the paper's own numbers, and the benchmark is memory QA rather than agentic tasks, but it at least shows that "selecting a part" can clearly beat "giving everything".
- **Learned context management shows positive results.** MemAct models context edits such as deletion and insertion as reinforcement-learning actions and reports maintaining or improving accuracy while shortening context; PACE adjusts the granularity of history according to the predicted relevance to the next action and reports significant gains on long-horizon tasks; ACE [Chen et al., 2026] models "retrieve or think" as a dynamic decision (see Section 8 for references).
- **Common sense**: human experts don't "recall everything at once" when solving a problem; they fetch what they need. Attention is a scarce resource, and this may hold for models too.

### 2.2 The core value: same context length, sharper focus

When people discuss context management, the most cited benefit is "fewer tokens, lower cost". I think a different point deserves more attention: **even at the same context length, selection can make the model more focused.**

Imagine a coding task at turn 60, with the same 128k window budget:

```
Full + compress:  [system][summary of early history...][raw turns 45–60: lots of test logs, abandoned plans]
                  → the window is full, but content directly relevant to the current step may be a small fraction

Self-selection:   [system][constraint ledger][raw interface contract from turn 3][current file][2 relevant test outputs][recent turns]
                  → the window is equally full, but nearly every segment is relevant to the current question
```

The lengths are identical, yet the **signal-to-noise ratio** may differ greatly. This matters for several reasons:

- **Irrelevant content actively distracts the model**, rather than merely "taking up space". Prior work shows that adding irrelevant information to a problem noticeably lowers reasoning accuracy [Shi et al., 2023], and information in the middle of long contexts is more easily ignored [Liu et al., 2023].
- **Compression trades off by time; selection trades off by need.** Compression shortens early content uniformly, yet early content may contain exactly the raw text most needed now (e.g., the exact wording of an interface contract). Selection can bring that text back verbatim while removing irrelevant logs from recent turns.
- **It survives the "long context keeps getting better" objection.** A bigger window does not automatically improve the signal-to-noise ratio; a bigger window just lets more irrelevant content in. So the goal of "making the window's content more relevant under a given budget" does not become meaningless as windows grow; only the budget point changes.

In other words, **self-selection is essentially about optimizing content composition under a fixed attention budget**; saving tokens is a by-product.

- **Directions to try**:
  - **Length-controlled comparison**: make the context length of both approaches **exactly equal** (e.g., both filled to 32k), vary only the composition, and compare accuracy. This separates the "more focused" effect from the "shorter" effect.
  - **Distractor injection**: at fixed length, gradually replace relevant content with irrelevant but similar-looking content and plot the accuracy decline, quantifying the value of focus.
  - **Inspect attention** (open-weight models): compare attention weights on key segments, or retrieval-head activations, under the two assembly methods.

### 2.3 Reasons for skepticism

- **Long-context ability is improving fast.** If models reliably handle a million tokens at low enough cost, the marginal value of careful selection drops. But as Section 2.2 argues, this mainly weakens the "saving tokens" side, much less the "focus" side.
- **Wrong selections can be costly.** If the selector misses a key piece of information, the main model doesn't even know it exists and may confidently give a wrong answer (Section 4.2).
- **Evidence comes mostly from memory QA.** Direct comparisons of "self-selection vs. full + compression" on real agentic tasks such as code editing and multi-step tool use are still rare.
- **Short tasks may not benefit.** For tasks that end in a few turns, the context is short anyway; the selector only adds latency and cost.

### 2.4 My judgment (to be tested)

> On **long-horizon, noisy-context** tasks, self-selection **is likely** to bring a substantial accuracy gain, and this gain should persist even at the same context length, coming mainly from sharper focus; on short tasks the benefit is limited or even negative. "How much gain" and "how long a task must be to pay off" have no reliable answers yet, which is itself worth researching.

### 2.5 Directions to try

- **Minimal controlled experiment**: take a batch of your own long tasks (≥ 30 turns) and compare task success under "full + compression" versus "assemble after selection". This is the most direct way to answer this section's question.
- **Find the break-even point**: bucket tasks by length and plot "number of turns vs. accuracy difference" to see from which point selection starts to win.
- **Separate "noise" from "forgetting"**: design two kinds of tests, one that adds irrelevant content to the context and one that places key information in very early turns, and see which kind selection helps more.

---

## 3 Implementation Routes

### 3.1 Route 1: the model manages itself (single model)

The main model manages its context through tool calls. For example, MemGPT [Packer et al., 2023] lets the model page information in and out between main context and external storage like an operating system; MemAct goes further and trains editing actions into the model.

- **Pros**: the main model knows best what it needs.
- **Cons**: consumes the main model's reasoning steps and tokens; training raises the bar; the model may "not know what it doesn't know".
- **Direction to try**: without training, give an off-the-shelf model just two tools, `recall(id)` and `drop(id)`, and observe whether and when it uses them.

### 3.2 Route 2: selector model + generator model (two-stage)

This is the scheme I originally had in mind: **split the history into segments, turn them into options, and when a new question arrives, have a selector model pick the needed segments first, then assemble them for a regular LLM.**

```
History segments → segment cards (ID + short summary)
New question → [selector model] judges "needed?" for each segment → selected segments assembled in order → [generator model] answers
```

#### Jev: a model naturally suited to be a selector

**Jev** (TypeSafe AI) [TypeSafe, 2026], which has drawn attention recently, provides exactly the tools for this. It calls itself a "System One model", named after "System 1" (fast, intuitive judgment) in Kahneman's *Thinking, Fast and Slow*. How it differs from a regular LLM:

- **It does not generate text**; it only answers typed questions. There are currently three question primitives: **Choice** (pick one of several, with a probability per option), **Score** (rate on ordered levels), and **Noul** (yes/no, returns the probability of "yes").
- **Outputs are calibrated probabilities and confidence**, so code can branch on thresholds directly without parsing text.
- **One request can ask many questions at once** (the docs call this speculative fan-out).
- The official documentation states explicitly that Jev **cannot** replace the LLM behind a coding agent; it is meant for decisions inside an agent such as routing, scoring, and filtering.

These features map neatly onto the needs of context selection:

| What context selection needs | What Jev offers |
|---|---|
| Judge "needed?" for each history segment | One Noul per segment: "Is this segment needed to answer the current question?" |
| Judge dozens of segments at once | Multiple questions per request (fan-out) |
| Keep more when unsure | Confidence gating: fall back to keeping more at low confidence |
| Can't be too slow or expensive | No text generation, no long output decoding |

People have already tried this route:
- **Jev-Mem** [Jiang et al., 2026]: uses Jev as a "System One controller" for memory typing, relation judgments, retrieval routing, and deciding "is the evidence sufficient", while a "System Two" LLM synthesizes the answer. This is the two-stage architecture above, applied to memory (numbers in Section 2.1).
- **Classifying RAG passages** in the official cookbook: Jev asks four yes/no questions of each retrieved passage (relevant? usable evidence? contradicts the question's premise? trying to instruct the model?), then code decides whether the passage goes into the evidence block, the conflict block, or is dropped.
- **Skill suggestion** in the official cookbook: picks at most one of 182 skills for an agent turn, which is again "selecting from candidates".
- On the open-source side there are similar models such as **Laya** (fine-tunable with the RLCD method) and the 0.6B replica **NanoJev**, which can run locally; **System1-Arena** [Chen et al., 2026b] studies specifically "which decisions can be left to System 1".

#### Isn't this just RAG?

Structurally it is very similar: "candidate pool → select → concatenate". You can think of it as "**RAG with your own history as the corpus**". But a few key differences make it a problem of its own:

| | Standard RAG | History context selection |
|---|---|---|
| Candidate pool | External documents; large, mutually independent | Own history; small, **chronologically ordered, with inter-segment dependencies** |
| Selection criterion | Does it "look like" the question | Is it "needed" to answer correctly, including implicit dependencies |
| Typical difficulty | Incomplete recall | Coreference ("that function"), early constraints lexically unrelated to the current question |
| Cost of a wrong pick | Incomplete answer, usually noticeable | The model **quietly violates earlier agreements**, hard to notice |

In one sentence: **RAG selects what is "relevant"; context selection must select what is "required".** Similarity search is good at the former; the latter needs judgment, which is why decision models like Jev are better suited as selectors than pure vector retrieval.

- **Directions to try**:
  - On the same tasks, compare three selectors: pure vector top-k, LLM listwise selection, and per-segment Jev Noul judgments. Compare required-segment recall, latency, and cost.
  - Design richer Jev question sets: besides "needed?", ask "contains a constraint?", "outdated?", "contradicts another segment?", and combine the answers in code.
  - Reproduce locally with Laya or NanoJev to see how well a small model can serve as the selector.

### 3.3 Route 3: compress instead of select

Selection "picks the original text"; compression "rewrites it into something shorter". Examples include LLMLingua [Jiang et al., 2023], RECOMP [Xu et al., 2023], and ACON, which optimizes compression strategies specifically for agents.

- **Pros**: information is not lost wholesale.
- **Cons**: rewriting can distort; once compressed, original details cannot be recovered (unless stored separately).
- **Direction to try**: **hybrid selection + compression**. Selected segments stay verbatim; unselected ones remain in context as a one-line summary with an ID, so the main model at least knows "something is there" and can recall the original when needed.

### 3.4 Route 4: server-side segment caching

If you control the inference engine, each segment can have its own KV cache, and selected segments are spliced together, known as "position-independent caching" (PIC), e.g., Prompt Cache [Gim et al., 2024], CacheBlend [Yao et al., 2025], SparseX [Zhang et al., 2026], HYPIC [Liu et al., 2026]. This route mainly addresses the caching problem in Section 4.1, at the cost of losing cross-attention between segments, which requires partial recomputation.

- **Direction to try**: experiment on vLLM / SGLang + LMCache and measure how much accuracy "segment caching + selection" loses relative to "prefix caching + full context".

---

## 4 Problems and Mitigation Ideas

### 4.1 Problem 1: cache invalidation (the most overlooked)

#### Why the cache breaks

The prompt caches offered by mainstream commercial APIs (Anthropic, OpenAI, DeepSeek, etc.) all use **prefix matching**: only the part of the current request identical to a previous request from the very beginning hits the cache. Cached input is typically priced at a small fraction of uncached input and also lowers time-to-first-token.

Dynamic selection, by nature, swaps content every turn, which conflicts directly:

```
Previous turn: [system][seg A][seg B][seg C][seg D][question]
This turn:     [system][seg A][seg C][seg D][seg E][question]
                               ↑ seg B removed — everything from here on misses
```

**The earlier the deletion or insertion, the larger the invalidated range.** This can lead to a counterintuitive outcome: fewer tokens, but because of cache misses, higher actual cost and longer time-to-first-token. One engineering project noted in its changelog that injecting a different context bundle each turn "structurally must rewrite the prompt-cache prefix", and that its previously reported large savings had never been computed alongside cache hit rate.

A simple estimate of input cost per turn:

$$
\text{cost} \approx p_{\text{hit}} \times \text{hit length} + p_{\text{miss}} \times (\text{total length} - \text{hit length})
$$

When evaluating any selection scheme, use this formula to compute **actual cost**, not just token count.

#### Mitigation ideas

1. **Layered layout: push changes to the end.** Select only within an "attachment zone" at the end; everything before it is append-only:
   ```
   [system prompt + tools]       never changes
   [constraint ledger]           append-only
   [dialogue spine (condensed)]  append-only, compressed once when full
   [selectable attachment zone]  ← the only part selected each turn
   [current question]
   ```
   Bulky items such as long tool outputs and file contents are moved to the attachment zone, leaving a one-line reference in the spine. Most of the prefix is preserved.
2. **Sticky ordering.** Within the attachment zone, segments selected for many consecutive turns go first; newly selected ones go at the end. Persistently needed content becomes a de facto stable prefix.
3. **Do the math before deleting.** When a segment becomes irrelevant, don't necessarily delete it immediately. If deleting it would invalidate a large cached tail and the segment itself is short, keeping it may be cheaper. A rough rule:

   $$
   \text{delete iff:} \quad \text{remaining turns} \times p_{\text{hit}} \times \text{segment length} \;>\; (p_{\text{miss}} - p_{\text{hit}}) \times \text{total length after it}
   $$

   Intuitively: **delete large segments near the end promptly; keep small segments near the front for now.**
4. **Batch deletions.** When several segments become useless, delete them together in one turn so the invalidation cost is paid once.
5. **Hysteresis thresholds.** Make the threshold for "adding" higher than for "dropping", so a segment doesn't flip in and out on adjacent turns.
6. **Server-side options** (if self-hosting): segment caching (Section 3.4); making the server aware of context lifecycles so obsolete caches are reclaimed promptly (Helios); or aligning shared prefixes across requests at assembly time (ContextPilot).

- **Directions to try**: on the same tasks, record cost under both "token count only" and "cache-aware pricing" and see whether conclusions flip; toggle each mitigation above and measure how much hit rate each recovers.

### 4.2 Problem 2: missed segments and "silent violations"

When the selector misses a segment, the main model doesn't know it exists. The most dangerous miss is a **constraint**: at turn 3 the user says "don't change the public API"; if that is not selected at turn 40, the model may change it without a second thought. Such errors raise no exceptions and are hard to notice.

#### Mitigation ideas

1. **Constraint ledger**: identify information that "must be followed from now on" (constraints, decisions made, user preferences, rejected options), extract each into a short record, and keep it **permanently resident, outside selection**. This step can itself be delegated to a decision model with one Noul: "Does this message establish something that must be followed going forward?" Jev's official simple-memory example uses exactly this kind of "gate".
2. **Leave a table of contents**: unselected segments remain in context as one-line summaries with IDs, so the main model knows they exist.
3. **Recall tool**: give the main model `recall(id)` to fetch originals when it notices something missing; recalled content is appended at the end, preserving the prefix.
4. **Confidence fallback**: when the selector is unsure, keep more. This is the textbook use of Jev's confidence gating.

- **Directions to try**: define a "constraint violation rate" (the share of turns violating established constraints) to measure this class of errors specifically; track recall-tool usage. Frequent recalls indicate frequent misses, and recalled segments can serve as training signals to improve the selector.

### 4.3 Problem 3: extra latency and cost from the selector

One more call per turn increases time-to-first-token.

#### Mitigation ideas

1. **Use a fast model for selection.** Decision models don't generate long text and are inherently faster than general LLMs; local small models (e.g., NanoJev, Laya) are another option.
2. **Select ahead of time.** Pre-select based on available information after the previous turn ends, while the user is still typing; then make small corrections once the input arrives.
3. **Coarse filter, then fine selection.** Use vector and keyword retrieval to shortlist a few dozen candidates, then let the decision model choose among them, instead of judging the entire history every turn.
4. **Trigger on demand.** Skip selection while the context is short; enable it beyond a length threshold.

- **Directions to try**: measure the ratio "selector latency / main model time-to-first-token", and where the length threshold for enabling selection pays off best.

### 4.4 Problem 4: the selector doesn't know what the main model needs

The selector sees only segment cards and the current question, while the main model discovers what it needs during reasoning. There is an information gap between them.

#### Mitigation ideas

- **Close the loop with recall signals**: the main model's recalls reveal what the selector missed, which can be used to adjust the selector's questions or fine-tune it (Laya supports local fine-tuning).
- **Let the main model state its needs**: the main model appends a line "information likely needed next" to its answer as extra input for the next turn's selection.
- **Two-pass**: the main model first drafts a plan from the selected content, then a second selection is made based on the plan, at the cost of one extra call.

### 4.5 Problem 5: how to segment

Too coarse, and a segment mixes useful and useless content; too fine, and there are too many candidates, selection gets harder, and coreference chains break.

- **Direction to try**: compare segmenting by dialogue turn, by tool call, by semantic chunk, and by file or function, and see which yields the highest required-segment recall. Code tasks may naturally suit file- and function-level segmentation.

### 4.6 Problem 6: broken chronology and coreference

If selected segments aren't kept in original order, the model may confuse the sequence of events (e.g., the same test failing and then passing); yet the sticky ordering in Section 4.1 changes the order.

- **Mitigation idea**: keep selected segments in original order by default; force chronological order for order-sensitive groups (multiple versions of the same object) and label them with timestamps or turn numbers; apply sticky ordering only to mutually independent segments.

### 4.7 Problem 7: selected content may be harmful

Tool outputs and web content in the history may carry prompt injections. A selection mechanism that looks only at relevance may pick the injected content.

- **Mitigation idea**: during selection, also ask "is this segment trying to instruct the model?", a ready-made question from Jev's RAG cookbook; down-weight or isolate suspicious segments.

### 4.8 Problem 8: how to evaluate

There is no standard benchmark dedicated to "agent history context selection" yet.

- **Directions to try**:
  - Borrow existing benchmarks: LoCoMo [Maharana et al., 2024] and LongMemEval [Wu et al., 2024] (long-term memory), SWE-bench [Jimenez et al., 2024] (coding agents), τ-bench [Yao et al., 2024] (rule-governed tool interaction, good for testing constraint violations).
  - Build "required segment" annotations: for each turn, label the history segments necessary to answer correctly, to compute selector recall.
  - Metrics should include at least: task success rate, constraint violation rate, required-segment recall, **actual cost under cache-aware pricing**, and time-to-first-token.

---

## 5 One Possible Combination (for Reference Only)

Putting the ideas together gives roughly the following shape. It is only my sketch, not a recommended design:

```
New input arrives
  │
  ├─ Ingest: route new content
  │    constraints / decisions / preferences → constraint ledger (resident)
  │    long tool outputs / files → attachment pool (segment cards); one-line reference in the spine
  │    everything else → dialogue spine (append-only; compressed once when full)
  │
  ├─ Select: attachment pool only
  │    coarse filter (vector + keyword + recency + bonus if selected last turn)
  │    → decision model, one Noul per segment (needed? / outdated? / injection?)
  │    → fall back to keeping more at low confidence
  │
  ├─ Cache-aware adjustment
  │    do-the-math deletion + batch deletion + hysteresis + sticky ordering
  │
  └─ Assemble: [system][ledger][spine][attachment zone][current input] → generator model
                                                   ↑ main model can call recall(id) for missed content
```

In multi-agent systems, this structure extends naturally to "**role-based context routing**": a coordinator selects attachments for each sub-agent; for example, a testing agent sees only test outputs and related source files. All agents share the same system prompt and ledger so they can share caches.

---

## 6 Open Problems

For interested readers. Each could be a small project on its own:

1. On real agentic tasks, how large is the accuracy gain of self-selection over "full + compression"? From how many turns does it pay off?
2. At exactly the same context length, how much does "sharper focus" alone contribute? Does this gain shrink as models get stronger?
3. What is the best division of labor between context selection and existing memory systems: memory "retrieves", the assembly layer "arranges", or should they be unified?
4. As long-context ability keeps improving, will the gain disappear or persist?
5. Which makes a better selector, a decision model (e.g., Jev) or a general LLM? On which kinds of segments is the gap largest?
6. On commercial APIs with prefix caching only, can "selection + cache-aware adjustment" beat "full + compression" on both accuracy and cost?
7. Who should extract the constraint ledger? How severe are the consequences of wrong or missed extraction?
8. Can recall signals be used to improve the selector cheaply and continuously?
9. Does the optimal segmentation granularity differ by task type?
10. In multi-agent settings, can role-based context routing reduce interference between agents?
11. Could the selector become a new attack surface (e.g., crafted content that induces it to include or drop specific segments)?

---

## 7 If You Want to Try It

A minimal experiment you can start over a weekend:

1. Pick 5–10 long tasks of your own (30+ turns) and save the full transcripts.
2. Implement two versions: A, full context + compression when full; B, layered layout + attachment-zone selection (start with LLM listwise selection, then swap in Jev or a local small model).
3. Replay the tasks with the same generator model under both assembly methods, recording success rate, constraint violations, and the **cached-token counts returned by the API**. Also run a group B' in which B's context length is made exactly equal to A's, to isolate the "focus" effect.
4. Compute actual cost for both using the formula in Section 4.1.

Even a result of "B is no better than A" is a valuable finding. Please share your results.

---

## 8 References

### Academic papers

- [Asai et al., 2023] A. Asai et al. *Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection.* arXiv:2310.11511.
- [Chhikara et al., 2025] P. Chhikara, D. Khant et al. *Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory.* arXiv:2504.19413.
- [Chen et al., 2026] R. Chen, J. Wang, W. Li et al. *To Retrieve or To Think? Cross-Boundary Context Evolution for Multi-hop Complex Reasoning* (ACE). arXiv:2601.08747.
- [Chen et al., 2026b] T. Chen, X. Liu, R. An et al. *System1-Arena: Which Decisions Can Be Left to System 1?* arXiv:2610.21244.
- [Gim et al., 2024] I. Gim et al. *Prompt Cache: Modular Attention Reuse for Low-Latency Inference.* MLSys 2024. arXiv:2311.04934.
- [Jiang et al., 2023] H. Jiang et al. *LLMLingua: Compressing Prompts for Accelerated Inference of Large Language Models.* EMNLP 2023. arXiv:2310.05736.
- [Jiang et al., 2026] D. Jiang, Y. Li, B. Li. *Jev-Mem: System-One-Controlled Agentic Memory for Efficient AI Agents.* arXiv:2609.23986.
- [Jimenez et al., 2024] C. E. Jimenez et al. *SWE-bench: Can Language Models Resolve Real-World GitHub Issues?* ICLR 2024. arXiv:2310.06770.
- [Kang et al., 2025] J. Kang, M. Ji et al. *Memory OS of AI Agent* (MemoryOS). arXiv:2506.06326.
- [Liu et al., 2023] N. F. Liu et al. *Lost in the Middle: How Language Models Use Long Contexts.* TACL 2024. arXiv:2307.03172.
- [Liu et al., 2026] Y. Liu, J. Wu, Y. Liu et al. *HYPIC: Accelerating Hybrid-Attention LLM Serving with Position-Independent Caching.* arXiv:2607.01299.
- [Maharana et al., 2024] A. Maharana et al. *Evaluating Very Long-Term Conversational Memory of LLM Agents* (LoCoMo). ACL 2024. arXiv:2402.17753.
- [Packer et al., 2023] C. Packer et al. *MemGPT: Towards LLMs as Operating Systems.* arXiv:2310.08560.
- [Park et al., 2023] J. S. Park et al. *Generative Agents: Interactive Simulacra of Human Behavior.* UIST 2023. arXiv:2304.03442.
- [Shi et al., 2023] F. Shi, X. Chen et al. *Large Language Models Can Be Easily Distracted by Irrelevant Context.* ICML 2023. arXiv:2302.00093.
- [Wu et al., 2024] D. Wu et al. *LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory.* arXiv:2410.10813.
- [Xu et al., 2025] W. Xu, Z. Liang et al. *A-MEM: Agentic Memory for LLM Agents.* arXiv:2502.12110.
- [Xu et al., 2023] F. Xu, W. Shi, E. Choi. *RECOMP: Improving Retrieval-Augmented LMs with Compression and Selective Augmentation.* arXiv:2310.04408.
- [Yao et al., 2024] S. Yao et al. *τ-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains.* arXiv:2406.12045.
- [Yao et al., 2025] J. Yao et al. *CacheBlend: Fast Large Language Model Serving for RAG with Cached Knowledge Fusion.* EuroSys 2025. arXiv:2405.16444.
- [Zhang et al., 2026] Q. Zhang, K. Chen, N. Liao et al. *SparseX: Efficient Segment-Level KV Cache Sharing for Interleaved LLM Serving.* arXiv:2606.01751.

### Conference papers (⚠ author information to be added)

- *PACE: Predictive Adaptive Context Extraction for Long-Horizon LLM Agents.* ACL 2026. https://aclanthology.org/2026.acl-long.1252.pdf
- *Memory as Action: Autonomous Context Curation for Long-Horizon Agentic Tasks* (MemAct). Findings of ACL 2026. https://aclanthology.org/2026.findings-acl.956/
- *ACON: Optimizing Context Compression for Long-horizon LLM Agents.* ICML 2026. https://icml.cc/virtual/2026/poster/66270
- *Helios: Exposing Context Lifecycles for Memory-Efficient Agentic LLM Serving.* ICPP 2026 Workshops. doi:10.1145/3816891.3834846
- *ContextPilot.* MLSys 2026. https://mlsys.org/media/mlsys-2026/Slides/3587.pdf

### Documentation and open-source projects

- [TypeSafe, 2026] TypeSafe AI. *Jev documentation* (System One, Primitives, Confidence, Patterns, Cookbooks: Classifying RAG passages / Simple memory system / Skill suggestion). https://docs.typesafe.ai
- Jev-Mem code: https://github.com/libingzheren/Jev-Mem
- NanoJev: https://github.com/TianyuCodings/NanoJev
- Jev Cookbook (Datawhale, Chinese tutorial including local fine-tuning of Laya): https://github.com/datawhalechina/jev-cookbook
- GraphFlow changelog (engineering discussion of prefix churn and cache locality): https://github.com/Roarpeng/GraphFlow
- Official prompt / context caching documentation from Anthropic, OpenAI, and DeepSeek (current pricing and semantics prevail).

---

## 9 Disclaimers

1. **Nature of this work**: this is a personal, exploratory discussion draft that **has not been peer-reviewed**. Its judgments, proposals, and "directions to try" are conjectures or suggestions **not systematically validated by the author's own experiments**, and should not be taken as established conclusions.
2. **Data sources**: all experimental numbers quoted (e.g., Jev-Mem's LoCoMo scores) come from the respective papers' or projects' own reports. The author **has not reproduced them**; the original sources are authoritative.
3. **No advice**: this essay does not constitute technical-selection, procurement, business, or investment advice. Any consequences of implementing the ideas herein (including but not limited to API costs, system failures, data or security issues) are borne by the implementer.
4. **Conflicts of interest**: the author has **no employment, investment, or partnership relationship** with any company, product, or project mentioned (including TypeSafe AI / Jev, Anthropic, OpenAI, DeepSeek), and received no compensation for this essay. Specific products are mentioned only to illustrate ideas.
5. **Trademarks**: product, model, and company names belong to their respective owners.
6. **Timeliness**: descriptions of vendor pricing, caching semantics, and model capabilities reflect public information at the time of writing (October 2026) and may change at any time; consult the latest official documentation.
7. **AI-assisted writing**: the ideas emerged from multiple rounds of discussion between the author and AI assistants (DeepSeek, Claude); drafting, literature search, and the English translation were done with AI assistance. References have been checked one by one where possible but may still contain errors; corrections are welcome. The author is responsible for the full content.
8. **License**: released under [CC BY 4.0](LICENSE). Redistribution, adaptation, and commercial use are permitted with attribution to the author and source.

---

## Appendix A: Glossary

| Term | Meaning |
|---|---|
| Context assembly | Deciding what goes into each request and in what order |
| Prefix caching | The serving system reuses computation for the identical leading part of requests; only content identical from the very beginning hits |
| Position-independent caching (PIC) | Techniques that allow caches of independent segments to be spliced and reused at any position; requires control of the inference engine |
| System One / decision model | A model that outputs only typed decisions and probabilities, without generating text, e.g., Jev |
| Noul / Choice / Score | Jev's three question primitives: yes/no / pick one / rate on levels |
| Segment / segment card | A unit of split history / a short description used for selection (ID, type, summary, etc.) |
| Constraint ledger | Constraints, decisions, and preferences extracted from history that must be followed continuously; resident in context |
| Silent violation | The model violates an earlier agreement without any error because the key constraint was not selected into context |
| Sticky ordering | Placing segments selected for many consecutive turns first, to keep the prefix stable |
| Signal-to-noise ratio (of context) | The share of context content that is actually relevant to the current question |

## Appendix B: Version History

| Version | Date | Main changes |
|---|---|---|
| v0.1 | 2026-10 | First draft (methods-paper style, unpublished) |
| v0.2 | 2026-10 | Reframed as an exploratory note; added Jev and the two-stage route; problems and mitigations |
| v0.3 | 2026-10-20 | Added "differences from memory mechanisms" and "sharper focus at equal length"; disclaimers, glossary, English version |

## How to Cite

```bibtex
@misc{redkinglu2026contextselection,
  author       = {RedKingLu},
  title        = {Letting LLMs Choose Their Own Context: Questions, Approaches, and Open Problems},
  year         = {2026},
  note         = {Discussion draft v0.3},
  doi          = {10.5281/zenodo.23494075},
  howpublished = {\url{https://github.com/RedKingLu/llm-context-selection}}
}
```
