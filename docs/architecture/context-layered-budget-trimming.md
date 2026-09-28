# 长上下文治理：分层上下文管理与预算裁剪机制

> 源码位置：`pico/context_manager.py`（509 行，全部机制都在这个文件里）  
> 配套：`pico/prompt_prefix.py`（prefix 层怎么来）、`pico/features/memory.py`（memory / relevant_memory 层怎么来）、`pico/checkpoint.py`（checkpoint 文本怎么拼进 prefix）



---

## 0. 一句话概括

整套机制是**两道闸门**：

| 闸门                | 时机     | 粒度 | 解决什么                           |
| ----------------- | ------ | -- | ------------------------------ |
| 闸门一：分层配额 + 单段局部裁剪 | 每段渲染时  | 段内 | 防止**单段自己**失控（一条长笔记、一段长历史把整段吃光） |
| 闸门二：超预算反向收缩       | 拼完总长超了 | 段间 | 防止**加总**失控，按"牺牲顺序"逐段往回砍        |

**当前请求（current_request）两层都不参与**，它是唯一被结构上豁免的段。

---

## 1. 分层剖面：这轮 prompt 到底由哪几层拼成

```
┌─────────────────────────────────────────────────┐
│ 1. prefix            工作手册 + 仓库现状         │ 预算 3600  底线 900   最后一个被牺牲
│   ├ 系统规则 / 工具说明 / 调用示例                │
│   ├ workspace.text()  目录树 + 文档 + 最近提交    │
│   └ checkpoint 文本（当前目标/阻塞/下一步）        │
├─────────────────────────────────────────────────┤
│ 2. memory            工作记忆"仪表盘"            │ 预算 1600  底线 400   第三个被牺牲
│   任务摘要 / 最近文件 / 文件短摘要 / 笔记计数      │
│   ← 只有目录和计数，正文不展开                     │
├─────────────────────────────────────────────────┤
│ 3. relevant_memory   按需召回的相关笔记           │ 预算 1200  底线 300   第一个被牺牲
│   最多 3 条，tag 精确命中 > 关键词重叠 > 新旧      │
├─────────────────────────────────────────────────┤
│ 4. history           Transcript 事件流           │ 预算 5200  底线 1300  第二个被牺牲
│   最近 6 条保真 → 更旧的降级成摘要/折叠           │
├─────────────────────────────────────────────────┤
│ 5. current_request   本轮用户请求                 │ 无预算     无底线    永不裁剪
└─────────────────────────────────────────────────┘
```

### 为什么是这五层，而不是"塞一段完整历史"

| 层               | 提供者                                    | 存在理由                                                                                                 |
| --------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| prefix          | `WorkspaceContext` + `PromptPrefix`    | 模型每轮都要重新知道"我是谁、工具怎么调、仓库什么状态"。这部分**跨轮稳定**，所以放最前面，让 prompt cache 能命中（`prompt_cache_key = prefix_hash`） |
| memory          | `LayeredMemory.render_memory_text()`   | session history 是完整事件流，太贵。这一层只留"工作集"：任务摘要 + 最近碰过的文件 + 文件短摘要。**给目录不给正文**                              |
| relevant_memory | `LayeredMemory.retrieval_candidates()` | 笔记全量展开会炸预算，所以默认不展开；只有 query 命中才按需捞最多 3 条出来                                                           |
| history         | `session["history"]`                   | 回答"上一轮我干了什么"，但只能给最近的；旧的要降级                                                                           |
| current_request | 用户                                     | 本轮唯一不可替代的输入，必须完整                                                                                     |

分层不是为了好看，而是**让裁剪有抓手**：知道哪一段是"规则"、哪一段是"参考"、哪一段是"事实"，才能决定谁先被牺牲。

---


## 2. 常量即策略（`context_manager.py:13-30`）

```python
DEFAULT_TOTAL_BUDGET = 12000
DEFAULT_SECTION_BUDGETS = {
    "prefix": 3600,
    "memory": 1600,
    "relevant_memory": 1200,
    "history": 5200,
}
DEFAULT_SECTION_FLOORS = {
    "prefix": 1200,
    "memory": 400,
    "relevant_memory": 300,
    "history": 1500,
}
# 当 prompt 超预算时，会优先压缩这些 section。
DEFAULT_REDUCTION_ORDER = ("relevant_memory", "history", "memory", "prefix")
SECTION_ORDER = ("prefix", "memory", "relevant_memory", "history", "current_request")
CURRENT_REQUEST_SECTION = "current_request"
RELEVANT_MEMORY_LIMIT = 3
```

**备注 1：三个顺序，三个含义，别读混。**

| 常量                        | 含义       | 读法                |
| ------------------------- | -------- | ----------------- |
| `SECTION_ORDER`           | **拼接顺序** | 决定 prompt 文本里谁在前面 |
| `DEFAULT_REDUCTION_ORDER` | **牺牲顺序** | 决定超预算时先砍谁         |
| `DEFAULT_SECTION_BUDGETS` | **单段上限** | 局部裁剪的额度           |

**备注 2：分段预算之和 = 11600 < 总预算 12000。**  
3600 + 1600 + 1200 + 5200 = 11600，剩下 400 是留给 `"\n\n"` 分隔符和 current_request 的空间。也就是说：

> **总预算里并没有给 current_request 单独留额度。**

这是整套机制的关键伏笔——用户请求一长，总长立刻超 12000，闸门二被触发，其它段开始收缩。**"当前请求不被裁坏"不是一个静态保证，而是靠"别人让路"实现的动态保证。**

**备注 3：`DEFAULT_SECTION_FLOORS` 定义了但没被读取。**  
真正的底线来自 `_compute_section_floors()`，用的是 `budget // 4`：

| 段               | 常量写的底线 | 实际生效的底线（budget // 4） |
| --------------- | ------ | -------------------- |
| prefix          | 1200   | **900**              |
| memory          | 400    | 400                  |
| relevant_memory | 300    | 300                  |
| history         | 1500   | **1300**             |

prefix 和 history 两处对不上。这是一处**死常量**（提一下，不动它）。

**备注 4：`DEFAULT_REDUCTION_ORDER` 里没有 `current_request`。**  
不是"忘了写"，而是这就是"永不裁剪"的第一道结构保证——循环根本不会遍历到它。

---


## 3. 主流程 `build()`（`context_manager.py:78-182`）

```python
def build(self, user_message):
    user_message = str(user_message)
    self.section_floors = self._compute_section_floors()          # ① 每轮重算底线
    memory_enabled = True
    relevant_memory_enabled = True
    context_reduction_enabled = True
    if hasattr(self.agent, "feature_enabled"):
        memory_enabled = self.agent.feature_enabled("memory")
        relevant_memory_enabled = self.agent.feature_enabled("relevant_memory")
        context_reduction_enabled = self.agent.feature_enabled("context_reduction")   # ② 特性开关

    section_texts = {                                             # ③ 先攒原始文本，不裁
        "prefix": str(getattr(self.agent, "prefix", "")),
        "memory": "Memory:\n- disabled" if not memory_enabled else str(self.agent.memory_text()),
        "history": "",
        CURRENT_REQUEST_SECTION: f"Current user request:\n{user_message}",
    }
    checkpoint_text = ""
    if hasattr(self.agent, "render_checkpoint_text"):
        checkpoint_text = str(self.agent.render_checkpoint_text() or "").strip()
    if checkpoint_text:
        section_texts["prefix"] = section_texts["prefix"] + "\n\n" + checkpoint_text   # ④ checkpoint 挂进 prefix

    selected_notes = []
    if memory_enabled and relevant_memory_enabled and hasattr(self.agent, "memory") and hasattr(self.agent.memory, "retrieval_candidates"):
        selected_notes = self.agent.memory.retrieval_candidates(user_message, limit=RELEVANT_MEMORY_LIMIT)  # ⑤ 先召回，后裁剪

    if not context_reduction_enabled:                             # ⑥ 关掉治理 → 无裁剪快路径
        rendered = self._render_sections_without_reduction(section_texts, selected_notes=selected_notes)
        prompt = self._assemble_prompt(rendered)
        metadata = self._metadata(...)
        return prompt, metadata

    budgets = dict(self.section_budgets)
    rendered = self._render_sections(section_texts, budgets, selected_notes=selected_notes)   # ⑦ 闸门一：局部
    prompt = self._assemble_prompt(rendered)
    reduction_log = []

    # 如果 prompt 超预算，就按固定顺序不断压缩。
    # 这里的顺序体现了平台偏好：
    # 先牺牲 relevant_memory，再牺牲 history，然后才动 memory 和 prefix。
    # 最新用户请求永远不裁剪，因为那是本轮最重要的输入。
    while len(prompt) > self.total_budget:                        # ⑧ 闸门二：全局
        overflow = len(prompt) - self.total_budget
        reduced = False
        for section in self.reduction_order:
            floor = int(self.section_floors.get(section, 0))
            current_budget = int(budgets.get(section, 0))
            if current_budget <= floor:
                continue                                          # ⑨ 到底线就跳过
            new_budget = max(floor, current_budget - overflow)     # ⑩ 精确到"刚好够"
            if new_budget >= current_budget:
                continue
            reduction_log.append({
                "section": section,
                "before_chars": current_budget,
                "after_chars": new_budget,
                "overflow_chars": overflow,
            })
            budgets[section] = new_budget
            rendered = self._render_sections(section_texts, budgets, selected_notes=selected_notes)  # ⑪ 立即重渲染
            prompt = self._assemble_prompt(rendered)
            reduced = True
            break                                                 # ⑫ 一次只砍一段
        if not reduced:
            break

    metadata = self._metadata(...)
    return prompt, metadata
```


### 逐点备注

**① 底线每轮重算。** `section_budgets` 可以被外部改（评测里就这么干），底线必须跟着走，否则底线会高于上限，`max(floor, current - overflow)` 会把预算**越压越大**。

**② 三个特性开关独立。**

- `memory` 关 → memory 段退化成 `"Memory:\n- disabled"`（保留占位，不删段，保证 prompt 结构稳定）
- `relevant_memory` 关 → 不召回，`selected_notes` 为空
- `context_reduction` 关 → 走 ⑥ 的无裁剪快路径

**Ablation 实验就是靠这三个开关做对照的**：同一个任务，开着跑一遍、关着跑一遍，比长度、比正确率。这也是为什么代码里到处是 `if xxx_enabled` 而不是直接删功能。

**③ 先攒原始文本，统一命名成 `section_texts`。** 这一步完全不裁，全是"意愿"。裁剪发生在下一步。**"意图"和"结果"分开存**，才能算出压缩率（`raw_chars` vs `rendered_chars`）。

**④ checkpoint 不是独立一层，而是挂在 prefix 末尾。**  
设计取舍：checkpoint 是"跨轮稳定 + 必须靠前"的信息，和 prefix 同性质；单独开一层会增加预算表的复杂度，收益不大。代价是 checkpoint 变长时会直接吃 prefix 的额度。

**⑤ 先召回，后裁剪——顺序很重要。**  
如果反过来（先按预算裁、再召回），就会出现"预算已经被历史吃光，召回的笔记无处可放"。现在是先确定**带什么**，再确定**带多少**。副作用是：召回选出的 3 条笔记可能随后被压到几乎看不见（见 `_render_relevant_memory`）。

**⑨ `current_budget <= floor: continue`** —— 到底线就不砍了，继续看下一段。这就是"底线"的作用：宁可总长超标，也不把某一段压成空壳。

**⑩ `new_budget = max(floor, current_budget - overflow)`** —— 精确计算"差多少砍多少"，不是"砍一半"。所以这套机制**不会过度压缩**：第一次收缩基本能刚好压到预算内。

**⑪ 砍完一段立刻重渲染 + 重测长度。**  
为什么不批量砍？因为"字符预算"和"渲染后字符数"不是线性的——历史段的渲染逻辑（`reversed` 逐条塞）对预算变化非常敏感，砍 100 字符可能只缩短 40 字符。**必须实测才知道够不够。**

**⑫ `break` + `if not reduced: break`** ——典型的**贪心收敛循环**：

- 每次只动一段（贪心）
- 动完重新测量，不够再继续
- `reduced=False` 说明所有段都到底线了，认输退出

**代价**：贪心不保证全局最优，可能多压了一点点；**收益**：实现 20 行、行为可预测、每一步都有 `reduction_log` 可解释。这是刻意的取舍——可观测性 > 理论最优。

---

## 4. 闸门一：局部裁剪的三个渲染器

### 4.1 基础工具 `_tail_clip`（`context_manager.py:33-41`）

```python
def _tail_clip(text, limit):
    text = str(text)
    if limit <= 0:
        return ""
    if len(text) <= limit:
        return text
    if limit <= 3:
        return text[:limit]
    return text[: limit - 3] + "..."
```

**备注：名字叫 `tail_clip`，行为却是"留头 + 省略号"。**  
它保留的是字符串**开头** `limit-3` 个字符，不是结尾。命名和行为不一致，是这份代码最容易误读的地方。

真实影响：工具输出（比如 `run_shell` 的报错）被它裁时，**丢掉的是尾部**——而 Python traceback 的关键帧恰恰在尾部。这是这套机制里最值得拿出来讨论的一个设计缺陷（见第 7 节）。

（对比：`workspace.py` 里另有一个 `middle()` 函数，是"留头留尾、砍中间"，那个才是适合日志/输出的裁法。）


### 4.2 分发器 `_render_sections`（`context_manager.py:226-241`）

```python
def _render_sections(self, section_texts, budgets, selected_notes=None):
    rendered = {}
    for section in SECTION_ORDER:
        budget = budgets.get(section)
        if section == CURRENT_REQUEST_SECTION:
            # 当前请求：原样输出，budget 固定 0，且根本不参与预算表
            raw = section_texts[section]
            rendered[section] = SectionRender(raw=raw, budget=0, rendered=raw, details={})
        elif section == "relevant_memory":
            rendered[section] = self._render_relevant_memory(selected_notes or [], int(budget or 0))
        elif section == "history":
            rendered[section] = self._render_history_section(int(budget or 0))
        else:
            # prefix / memory：简单粗暴，按预算留头截断
            raw = section_texts[section]
            rendered_text = _tail_clip(raw, int(budget)) if budget is not None else raw
            rendered[section] = SectionRender(raw=raw, budget=int(budget) if budget is not None else 0, rendered=rendered_text, details={})
    return rendered
```

**备注：三类段，三种裁剪策略。**

| 段 | 策略 | 为什么 |
| --- | --- | --- |
| current_request | 不裁 | 唯一不可替代的输入 |
| relevant_memory / history | **结构化裁剪**（按条、逐条塞） | 是"多条独立记录"，可以牺牲部分保全部 |
| prefix / memory | **整段截断**（`_tail_clip`） | 是"一整块连贯文本"，只能整体处理 |

`SectionRender` 同时记 `raw` / `budget` / `rendered` 三个值，这就是压缩率的原始数据来源。


### 4.3 relevant_memory：按条均分预算（`context_manager.py:243-295`）

```python
def _render_relevant_memory(self, selected_notes, budget):
    header = "Relevant memory:"
    note_texts = [str(note.get("text", "")) for note in selected_notes if str(note.get("text", "")).strip()]
    raw_lines = [header] + [f"- {text}" for text in note_texts]
    raw = "\n".join(raw_lines) if note_texts else "\n".join([header, "- none"])
    if not note_texts:
        ...  # 空召回："Relevant memory:\n- none"
        return SectionRender(...)

    per_note_budget = self._per_note_budget(budget, len(note_texts), header)
    rendered_notes = []
    while True:
        # 让每条 note 平分这一段的预算，避免一条超长笔记把其他笔记都挤掉。
        rendered_notes = [_tail_clip(text, per_note_budget) for text in note_texts]
        rendered = "\n".join([header] + [f"- {text}" for text in rendered_notes])
        if len(rendered) <= budget or per_note_budget <= 1:
            break
        per_note_budget -= 1

    if len(rendered) > budget and budget > 0:
        rendered = _tail_clip(raw, budget)      # 兜底：均分也塞不下，就整体截断
        rendered_notes = [rendered]

    return SectionRender(raw=raw, budget=budget, rendered=rendered, details={...})

def _per_note_budget(self, budget, note_count, header):
    if note_count <= 0:
        return 0
    overhead = len(header) + 3 * note_count    # header + 每条 "- " 前缀
    usable = max(0, budget - overhead)
    return max(1, usable // note_count)
```

**备注：为什么要"均分"而不是"整体截断"。**
`Relevant memory:` 下面挂的是最多 3 条**互相独立**的笔记。

- 整体截断 → 第一条长笔记吃掉全部预算，后两条直接消失。召回 3 条等于只召回 1 条。
- 均分 → 3 条各自留一点，**信息多样性**保住了，代价是每条都不完整。

这个取舍在检索场景下是对的：宁可拿到 3 个"半条线索"，也不要 1 个"完整线索"——因为用户请求里的关键词可能只和其中一条真正相关，被砍掉的那条往往才是有用的。

**备注：`while True` 是确定性收敛，不是盲目搜索。**
每轮把 `per_note_budget` 减 1，重新渲染测长度，直到塞得下或降到 1。上限是初始 `per_note_budget` 次迭代，不会死循环。

**`overhead` 把 `"- "`（2 字符）+ 换行（1 字符）算进去了**，所以预算是真的按"渲染后字符数"算的，不是按纯文本长度。


### 4.4 history：从最新往回塞（`context_manager.py:297-359`）

```python
def _render_history_section(self, budget):
    history = list(getattr(self.agent, "session", {}).get("history", []))
    raw = self._raw_history_text(history)
    if not history:
        return SectionRender(raw=raw, budget=budget, rendered="Transcript:\n- empty", details={...})

    # 优先保留最近的历史，因为下一步决策通常最依赖刚刚发生的工具结果。
    recent_window = 6
    recent_start = max(0, len(history) - recent_window)
    history_entries, history_details = self._compressed_history_entries(history, recent_start)

    rendered_entries = []
    for entry in reversed(history_entries):            # ① 从最新一条开始往前拼
        recent = bool(entry.get("recent", False))
        candidate_lines = list(entry.get("lines", []))
        candidate_entries = candidate_lines + rendered_entries
        candidate_rendered = "\n".join(["Transcript:", *candidate_entries])
        if len(candidate_rendered) <= budget:
            rendered_entries = candidate_entries       # ② 塞得下就收下
            continue
        if recent:                                     # ③ 最近 6 条：塞不下也抢救一下
            available = budget - len("Transcript:")
            if rendered_entries:
                available -= sum(len(line) + 1 for line in rendered_entries)
            available = max(20, available - 1)
            candidate_lines = [_tail_clip(line, available) for line in candidate_lines]
            candidate_entries = candidate_lines + rendered_entries
            candidate_rendered = "\n".join(["Transcript:", *candidate_entries])
            if len(candidate_rendered) <= budget:
                rendered_entries = candidate_entries
        else:                                          # ④ 更旧的条目：压到 20 字符
            smaller_lines = [_tail_clip(line, 20) for line in candidate_lines]
            smaller_entries = smaller_lines + rendered_entries
            smaller_rendered = "\n".join(["Transcript:", *smaller_entries])
            if len(smaller_rendered) <= budget:
                rendered_entries = smaller_entries
    rendered = "\n".join(["Transcript:", *rendered_entries])

    if len(rendered) > budget and budget > 0:
        rendered = _tail_clip(raw, budget)             # ⑤ 最后兜底

    return SectionRender(raw=raw, budget=budget, rendered=rendered, details={...})
```

**备注：`reversed()` 是关键——从最新一条往旧的方向填。**
预算优先分给"刚发生的事"，历史越旧越早被牺牲。填完再整体反转回来（因为 `rendered_entries` 是倒着累加的，最终拼出来仍是时间正序）。

**备注：旧条目不是被"丢弃"，而是被"降级"了四次。**
这个 `if/elif` 级联实际上是同一个条目在四个压缩等级里逐级下探：

| 等级 | 处理 | 位置 |
| --- | --- | --- |
| L0 完整 | 原文整条塞入 | `if len(...) <= budget` |
| L1 轻度截断 | 最近 6 条：按剩余空间截断 | `if recent:` |
| L2 重度截断 | 更旧条目：每条压到 20 字符 | `else:` |
| L3 完全折叠 | 在 `_compressed_history_entries` 里就没进列表 | 见 4.5 |

**备注：`recent_window = 6` 硬编码。** 没有做成配置项，因为 6 条刚好覆盖"上一次工具调用 + 上上次的上下文"，再多会挤占预算。


### 4.5 历史折叠：三级降级（`context_manager.py:361-423`）

```python
def _compressed_history_entries(self, history, recent_start):
    entries = []
    seen_older_reads = set()
    details = {"older_entries_count": 0, "collapsed_duplicate_reads": 0,
               "reused_file_summary_count": 0, "summarized_tool_count": 0}

    for index, item in enumerate(history):
        recent = index >= recent_start
        if recent:
            line_limit = 900                                    # 最近 6 条：给足 900 字符
            entries.append({"recent": True, "lines": self._render_history_item(item, line_limit)})
            continue

        if item["role"] == "tool" and item["name"] == "read_file":
            path = str(item["args"].get("path", "")).strip()
            if path in seen_older_reads:                        # 降级 1：重复读同一文件 → 直接丢
                details["collapsed_duplicate_reads"] += 1
                continue
            seen_older_reads.add(path)
            summary = self._reusable_file_summary(path)
            if summary:                                         # 降级 2：换成 memory 里的短摘要
                entries.append({"recent": False, "lines": [f"{path} -> {summary}"]})
                details["older_entries_count"] += 1
                details["reused_file_summary_count"] += 1
                continue

        if item["role"] == "tool":                              # 降级 3：所有旧工具调用 → 一行摘要
            summary_line = self._summarize_old_tool_item(item)
            entries.append({"recent": False, "lines": [summary_line]})
            details["older_entries_count"] += 1
            details["summarized_tool_count"] += 1
            continue

        entries.append({"recent": False, "lines": self._render_history_item(item, 60)})  # 旧对话 → 60 字符

    return entries, details

def _reusable_file_summary(self, path):
    memory = getattr(self.agent, "memory", None)
    if memory is None or not hasattr(memory, "to_dict"):
        return ""
    snapshot = memory.to_dict()
    summary = snapshot.get("file_summaries", {}).get(str(path), {})
    if not summary:
        return ""
    return str(summary.get("summary", "")).strip()

def _summarize_old_tool_item(self, item):
    if item["name"] == "run_shell":
        command = str(item["args"].get("command", "")).strip() or "shell"
        lines = [line.strip() for line in str(item.get("content", "")).splitlines() if line.strip()]
        summary = " | ".join(lines[:3]) if lines else "(empty)"
        return f"{command} -> {summary}"
    return self._render_history_item(item, 60)[0]
```

**备注：这是"分层"最实在的体现——同一个文件读两次，成本不一样。**

| 历史位置 | `read_file` 同一文件的处理 | 长度量级 |
| --- | --- | --- |
| 最近 6 条 | 原文（最多 900 字符） | ~900 |
| 更旧、首次出现 | `path -> memory 里的文件摘要` | ~100 |
| 更旧、重复出现 | **直接删掉** | 0 |

**降级 2 是跨模块协作的关键**：`memory` 层已经存过这个文件的摘要，历史层就不必重放全文，只放一个指针 `path -> summary`。历史层和 memory 层在这里形成"目录—正文"的分工。

**降级 3 只对 `run_shell` 做了特殊摘要**（取前 3 行非空行，拼成 `command -> line1 | line2 | line3`），其它工具退化成通用的 60 字符截断。是有意为之的"性价比选择"：shell 输出最容易膨胀，优先照顾。

---

## 5. 闸门二补充：预算表与底线（`context_manager.py:218-224`）

```python
def _compute_section_floors(self):
    floors = {
        section: max(20, int(budget) // 4)      # 默认：不超过上限的 1/4
        for section, budget in self.section_budgets.items()
    }
    floors.update(self._section_floor_overrides)  # 外部可覆盖
    return floors
```

**备注：底线是"上限的 1/4，且不低于 20"。**
为什么是 1/4？—— 上限的 1/4 是"这段还能读出意思"的经验值。`max(20, ...)` 是兜底，防止上限本身很小（评测里会改成 120）时底线算成 0。

**底线的意义：宁可 prompt 超标，也不让某段变成空壳。**
如果 `prefix` 被压成 0，模型就看不到工具怎么调；如果 `memory` 被压成 0，模型会忘掉"任务是啥"。**底线防的是"结构崩坏"，不是"超预算"**。

---

## 6. 拼接与记账

### 6.1 拼接顺序（`context_manager.py:444-454`）

```python
def _assemble_prompt(self, rendered):
    # 顺序是刻意设计的：稳定规则放前面，最新请求放最后。
    return "\n\n".join([
        rendered["prefix"].rendered,
        rendered["memory"].rendered,
        rendered["relevant_memory"].rendered,
        rendered["history"].rendered,
        rendered[CURRENT_REQUEST_SECTION].rendered,
    ]).strip()
```

**备注：位置不是随便排的，服务两个目的。**
1. **前缀缓存**：`prefix` 在最前且跨轮稳定 → `prompt_cache_key = prefix_hash` 能命中，省 token 省延迟
2. **近因效应**：`current_request` 在最末，紧贴模型生成位置，指令遵循率最高

中间三段（memory / relevant_memory / history）是"越靠前越稳定、越靠后越易变"，符合从"长期规则"到"临时事实"的渐变。


### 6.2 记账 `_metadata`（`context_manager.py:456-509`）

```python
def _metadata(self, prompt, rendered, budgets, reduction_log, selected_notes, user_message, section_texts):
    section_metadata = {}
    for section in SECTION_ORDER[:-1]:
        section_metadata[section] = {
            "raw_chars": rendered[section].raw_chars,        # 没裁之前多长
            "budget_chars": int(budgets.get(section, 0)),    # 这轮给了多少额度
            "rendered_chars": rendered[section].rendered_chars,  # 实际输出多长
        }
    section_metadata[CURRENT_REQUEST_SECTION] = {
        "raw_chars": len(section_texts[CURRENT_REQUEST_SECTION]),
        "budget_chars": None,                                # ← 没有额度，即"不参与预算"
        "rendered_chars": len(rendered[CURRENT_REQUEST_SECTION].rendered),
    }
    return {
        "prompt_chars": len(prompt),
        "prompt_budget_chars": self.total_budget,
        "prompt_over_budget": len(prompt) > self.total_budget,
        "section_order": list(SECTION_ORDER),
        "section_budgets": {
            section: (None if section == CURRENT_REQUEST_SECTION else int(budgets.get(section, 0)))
            for section in SECTION_ORDER
        },
        "sections": section_metadata,
        "budget_reductions": reduction_log,          # ← 每一次收缩的 before/after/overflow
        "reduction_order": list(self.reduction_order),
        "relevant_memory": {...},                    # selected_count / rendered_count / note_budget
        "history": {
            "raw_chars": ..., "rendered_chars": ...,
            "older_entries_count": ...,              # 降级成摘要的旧条目数
            "collapsed_duplicate_reads": ...,        # 被折叠掉的重复读次数
            "reused_file_summary_count": ...,        # 复用了 memory 摘要的次数
            "summarized_tool_count": ...,            # 被压成一行的工具调用数
        },
        "current_request": {
            "text": user_message,
            "raw_chars": len(user_message),
            "rendered_chars": len(user_message),     # ← 恒定相等，即"没被裁"
            "section_chars": len(rendered[CURRENT_REQUEST_SECTION].rendered),
        },
    }
```

**备注：`raw_chars` / `budget_chars` / `rendered_chars` 三列是整套机制的"可解释性"基础。**
压缩率 = `1 - rendered_chars / raw_chars`；`budget_reductions` 记录**每一段被砍了多少、为什么砍**（`overflow_chars`）。没有这层记账，压缩就变成黑盒，出问题无法归因。

**备注：`current_request` 的 `rendered_chars == raw_chars` 是恒等式。**
这是"当前请求不被裁坏"的可验证证据——不是靠信任，而是这段代码在结构上不可能不相等（`_render_sections` 里它直接 `rendered=raw`）。

---

## 7. 当前请求为什么不会被裁坏：三道结构性保险

| # | 保险 | 位置 |
| --- | --- | --- |
| 1 | `DEFAULT_REDUCTION_ORDER` 里**没有** `current_request`，收缩循环遍历不到它 | `context_manager.py:27` |
| 2 | `_render_sections` 中它是特判分支，`rendered = raw`，`budget` 恒为 0 且不读预算表 | `context_manager.py:230-232` |
| 3 | `_metadata` 里它 `budget_chars = None`，`rendered_chars` 直接取 `len(user_message)` | `context_manager.py:464-468` |

所以：**超长请求不会裁掉自己，而是触发其它段收缩给它腾位置**——这正好解释了实测里"当前请求保留率 100%"这个指标是怎么来的。

---

## 8. 设计思路总结（口语化版）

**比喻：分层配额 ≈ 分格行李箱，收缩循环 ≈ 搬家时的舍弃顺序。**

- 打包（分层）时先给每格定容量：证件格（prefix）、随身物品格（memory）、参考书格（relevant_memory）、换洗衣物格（history）。相机（current_request）**不占格子，抱在手上**。
- 箱子关不上（超预算）时，**按固定顺序往外拿**：先丢参考书（relevant_memory），再丢换洗衣物（history），再丢随身物品（memory），最后才动证件（prefix）。相机一直抱在手里，永远不丢。
- 每格都有"最低保留量"（底线）：证件格就算再挤，也不能一本都不留——不然出了机场寸步难行。

**四条核心思路：**

| # | 思路 | 一句话 |
| --- | --- | --- |
| 1 | **先分层，再裁剪** | 不分层就只能"整段 prompt 一刀切"，分了层才知道哪段能砍、砍到多少、砍了会丢什么 |
| 2 | **上限 + 底线 + 牺牲顺序，三件套** | 上限防超预算，底线防结构崩坏，顺序防误伤关键信息 |
| 3 | **贪心逐段收缩，不做全局最优** | 代码短、行为可预测、每步有日志；代价是可能多压一点、不是理论最短 |
| 4 | **全程可观测** | 每段记 raw/budget/rendered 三长度，每次收缩记 before/after/overflow——这是能拿来做 ablation 对照实验的前提 |

**两个已知的粗糙处（诚实记录，不改）：**

- `_tail_clip` 名字说 tail，行为是留头。工具输出的关键信息常在尾部（报错栈），这里会丢。
- `DEFAULT_SECTION_FLOORS` 是死常量，实际底线走 `budget // 4`，两处数值对不上（prefix 1200↔900、history 1500↔1300）。

---


## 9. 面试问答（口语化）

**Q：长上下文治理的核心是什么？**
A：两块。**分层**决定"带什么"，**预算裁剪**决定"带多少"。分层是给每类信息一个固定位置和固定额度，裁剪是总额超了就按优先级往回砍。关键点是当前请求永远不参与裁剪，长请求会挤别人，不会挤自己。

**Q：为什么先砍 relevant_memory，最后才砍 prefix？**
A：按"信息的替代成本"排的。relevant_memory 是参考笔记，砍了下一轮还能重新召回；prefix 是工具调用规则和仓库状态，砍了模型就不会调工具了，任务直接崩。history 在中间，因为它虽然不可再生，但旧的可以降级成摘要——所以它是第二个被砍的，而且砍法是渐进的，不是一刀切。

**Q：怎么保证当前请求不被裁坏？**
A：三处结构性保证。一是收缩顺序表里根本没有它；二是渲染时它是特判分支，直接原样输出；三是元数据里它没有预算额度。也就是说这不是靠"小心别裁错了"，而是代码结构上就进不了裁剪路径。长请求的后果是别的段收缩，不是它自己变短。

**Q：为什么用贪心逐段收缩，而不是一次性算好每段该留多少？**
A：因为"预算字符数"和"渲染后字符数"不是线性关系——历史段是按最新优先逐条塞的，砍 100 字符可能只缩短 40。所以必须砍一段、重新渲染、重新量。贪心换来了代码短和可解释。代价是不保证最优解，可能多压了一点。

**Q：这套机制怎么验证有效性？**
A：做 ablation。用同一个任务跑开/关两个变体，比 prompt 字符数和任务正确率。压缩率是 `1 - after/before` 逐配置算出来的；同时检查"当前请求保留"这个不变量。每一段的 raw / budget / rendered 三个长度都落在元数据里，收缩日志记了是哪一段、砍了多少、当时超了多少。

**Q：这套设计有什么问题？**
A：两个。第一，字符数当预算单位，但 token 数和字符数不是线性关系（中文、代码、JSON 密度差很多），严格说应该按 token 估。第二，`_tail_clip` 留头丢尾，对工具输出不友好——Python 报错栈在尾部，砍掉尾部等于把最有用的信息砍了，更适合的做法是留头留尾砍中间。这两个都是"知道但暂时不修"的程度——按字符算是零依赖的代价，`_tail_clip` 换成中间截断会影响所有段的压缩率基线，得连 ablation 一起重跑。
