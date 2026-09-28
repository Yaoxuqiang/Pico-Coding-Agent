# memory.py 核心逻辑精读（提炼版）

> 对象：`pico/features/memory.py`（659 行）  
> 定位：提炼版。讲骨架、设计理由、实测证据，不逐行走读。  
> 所有数字均为本机实测（Windows + Python 3.13.12），脚手架见文末。

---

## 1. 一句话概括

**659 行、12 个顶层函数、2 个类。这个模块管"工作记忆"——不是完整历史，是一份被反复提炼过的小抄。**

模块 docstring 把定位写得很清楚：

> session history 负责保存完整事件流；这个模块只保存更小的一层工作集：当前任务摘要、最近接触的文件、文件短摘要，以及少量跨轮笔记。**这样下一轮 prompt 还能接上上一轮，但不会被整段历史塞满。**

四层结构，前三层在 session JSON 里，第四层是独立的 markdown 文件：

| 层                | 存什么                         | 上限           |
| ---------------- | --------------------------- | ------------ |
| `working`        | 任务摘要 + 最近文件                 | 300 字符 / 8 个 |
| `file_summaries` | 每个文件的短摘要 + 内容哈希             | **无上限**      |
| `episodic_notes` | 会话内笔记                       | 12 条         |
| `durable`（磁盘）    | `MEMORY.md` + `topics/*.md` | **无上限**      |

**复杂度集中在两处**：`DurableMemoryStore`（166 行）+ `normalize_memory_state`（91 行）= 257 行，占全文 **39%**。其余十九个东西都是十几行的小工具。

---


## 2. 行区间分布

| 行区间         | 内容                                         | 行数      | 承担什么                                                                       |
| ----------- | ------------------------------------------ | ------- | -------------------------------------------------------------------------- |
| 1–13        | docstring + import                         | 12      | ——                                                                         |
| 15–17       | 3 个上限常量                                    | 3       | `WORKING_FILE_LIMIT=8` / `EPISODIC_NOTE_LIMIT=12` / `FILE_SUMMARY_LIMIT=6` |
| 19–41       | `DURABLE_TOPIC_DEFAULTS`                   | 23      | 四个固定主题的元数据                                                                 |
| 43–56       | `default_memory_state`                     | 14      | 空状态（**7 个键**）                                                              |
| **59–224**  | **`DurableMemoryStore`**                   | **166** | **跨会话记忆的读写（markdown）**                                                     |
| 227–247     | `_ensure_list` / `_dedupe_preserve_order`  | 21      | 小工具                                                                        |
| 250–292     | 路径工具 ×3 + `_tokenize` + `_parse_timestamp` | 43      | 路径规范化与哈希                                                                   |
| 295–331     | `_normalize_note`                          | 37      | 单条笔记的归一化                                                                   |
| **334–424** | **`normalize_memory_state`**               | **91**  | **整个模块的枢纽**                                                                |
| 427–502     | 6 个 mutator                                | 76      | 改状态的入口                                                                     |
| 505–516     | `summarize_read_result`                    | 12      | 工具结果 → 短摘要                                                                 |
| 519–558     | `retrieval_candidates` / `retrieval_view`  | 40      | 召回                                                                         |
| 561–586     | `render_memory_text`                       | 26      | 记忆 → prompt 文本                                                             |
| 589–596     | `is_effectively_empty`                     | 8       | 空判定                                                                        |
| 599–659     | `LayeredMemory`                            | 61      | 面向 runtime 的门面                                                             |

**该看哪几段**：`normalize_memory_state`（唯一的枢纽，所有入口都过它）、`DurableMemoryStore.promote`（唯一有"智能"的地方）、`render_memory_text`（唯一决定模型看到什么的函数）。

---

## 3. 必须分清的概念


### 3.1 状态里有三套并存的表示

`default_memory_state()` 返回 **7 个键**，但表达的只有 **4 件事**：

```python
{
    "working": {"task_summary": "", "recent_files": []},   # ① 新结构
    "episodic_notes": [],                                  # ② 新结构
    "file_summaries": {},
    "task": "",                                            # ① 的旧扁平镜像
    "files": [],                                           # ① 的旧扁平镜像
    "notes": [],                                           # ② 的旧扁平镜像
    "next_note_index": 0,
}
```

而 `normalize_memory_state` 末尾（418–420 行）**每次往两边都写一遍**：

```python
state["task"]  = working["task_summary"]
state["files"] = list(working["recent_files"])
state["notes"] = [note["text"] for note in episodic_notes]
```

实测三组镜像恒等：

```
working.task_summary : '给配置模块加超时参数'
task                 : '给配置模块加超时参数'
working.recent_files : ['a.py']
files                : ['a.py']
episodic_notes[0].text: '项目约定用 4 空格缩进'
notes                : ['项目约定用 4 空格缩进']
```

**为什么要留旧的？** 看 357–366 和 372–377 行——旧字段是**读入时的兼容路径**：

```python
if not str(working["task_summary"]).strip() and state.get("task"):
    working["task_summary"] = clip(str(state.get("task", "")).strip(), 300)
```

旧版本的记忆状态里只有扁平字段，新版本改成了嵌套。归一化层负责把旧的"拉"到新的位置。

**代价**：拉完之后**旧的还留着并持续同步**。所以一份数据在文件里占两倍空间，而且任何改动都要同步两个地方（写漏一处就不一致）。这属于"迁移做了一半"——正确的做法是拉过来之后把旧键删掉。

### 3.2 `clip` 结果长度 **不等于** limit

`clip` 来自 `workspace.py`，实现是：

```python
return text[:limit] + f"\n...[truncated {len(text) - limit} chars]"
```

实测：

```
clip(text, 300) -> 实际长度 326   结尾 'xx\n...[truncated 2700 chars]'
clip(text, 500) -> 实际长度 526   结尾 'xx\n...[truncated 2500 chars]'
```

**所以 `clip(x, 300)` 之后可能是 326 字符。** 那个"300 上限"是"正文上限"，尾巴上还挂着一句提示。

对照 `context_manager.py` 里的 `_tail_clip`——那个**严格等于 limit**（`text[:limit-3] + "..."`）。**同一个项目里两个名字很像的裁剪函数，语义完全不同。**

在 memory 里，`clip` 的结果会进 prompt（`render_memory_text` → prefix 的 memory 段），所以"300 上限"实际是"300 + 截断提示"。

### 3.3 `episodic_notes` 和 `durable` 是两套独立的东西

|    | episodic                         | durable                          |
| -- | -------------------------------- | -------------------------------- |
| 存哪 | session JSON 里的 `episodic_notes` | 磁盘上的 `MEMORY.md` + `topics/*.md` |
| 寿命 | 随会话，超 12 条淘汰                     | **跨会话，永久**                       |
| 谁写 | `append_note`                    | `promote`（只允许四个固定主题）             |
| 格式 | 结构化 dict                         | **人可读的 markdown**                |
| 召回 | 参与 `retrieval_candidates`        | 同一套打分，`note_index` 固定为 `-1`      |

**两者互不知情**：`is_effectively_empty` 只看前者；`render_memory_text` 只输出 durable 的**主题名列表**，不输出内容。

### 3.4 `separate`：`_subject_key` 到底能认什么

durable 层的去重完全依赖 `_subject_key`——把一句话压成"主语键"，主语相同就认为是同一件事的不同说法，用新的替换旧的。

它只认 6 种句式：

```python
r"^(.+?)\s+is\s+.+$"      r"^(.+?)\s+are\s+.+$"
r"^(.+?)\s+uses?\s+.+$"   r"^(.+?)\s+should\s+.+$"
r"^(.+?)是.+$"            r"^(.+?)使用.+$"
```

而且拿到主语后还要过一遍 `_tokenize`：

```python
def _tokenize(text):
    return {token.lower() for token in re.findall(r"[A-Za-z0-9_]+", str(text))}
```

**这个正则只匹配 ASCII 字母数字下划线。**（实测影响见 §7.14）

### 3.5 `normalize_memory_state` 会读磁盘

它末尾（421–423 行）做了这件事：

```python
durable_root = Path(workspace_root) / ".pico" / "memory" if workspace_root is not None else None
durable_store = DurableMemoryStore(durable_root) if durable_root is not None else None
state["durable_topics"] = durable_store.topic_slugs() if durable_store is not None else []
```

`topic_slugs()` → `load_index()` → `read_text()`。**一个名字叫"归一化"的函数里有文件 IO。**

所有 13 个调用点都走它，包括 `render_memory_text`、`is_effectively_empty`、`retrieval_candidates` 这些看起来纯读的函数。（实测调用次数见 §7.7）

---

## 4. 主流程骨架

### 写入路径

```
runtime 每轮：ask(user_message)
  └─ agent.memory.set_task_summary(user_message)          # working.task_summary
     
工具执行后（runtime.py:405-418）：
  read_file   → remember_file(path) + set_file_summary(path, summarize_read_result(result))
                + append_note(summary, tags=(path,))
  write/patch → remember_file(path) + invalidate_file_summary(path)   # 内容变了，摘要作废
  
任务结束：promote_durable_memory(user_message, final_answer)
  └─ extract_durable_promotions()      # 从最终答案里按固定句式抠出 (topic, note)
       └─ 先看用户消息里有没有"记住/记下来"的意图，没有就整个跳过
  └─ DurableMemoryStore.promote()      # 写 markdown
```

**注意 `read_file` 的三个动作**：记文件 + 存摘要 + 追加笔记。**一次读文件在记忆层留下三条痕迹**，其中笔记那条的 `tags` 就是文件路径——这就是后面召回时"读过的文件"能被语义匹配到的原因。

### 读取路径

```
写 prompt 前：
  context_manager.build()
    ├─ agent.memory_text()  → render_memory_text()        # 仪表盘，进 memory 段
    └─ agent.memory.retrieval_candidates(user_message, 3) # 召回，进 relevant_memory 段

恢复会话时：
  agent.invalidate_stale_memory() → invalidate_stale_file_summaries()
       # 比对每个 file_summary 的哈希和现实，不一致就删掉
```

**两条路径对应 prompt 的两段**：`render_memory_text` 是"我知道什么"（低带宽、固定位置），`retrieval_candidates` 是"这轮用得上的"（高带宽、按需）。

### 归一化的位置

```
任何 mutator / 任何 render / 任何召回 / LayeredMemory 构造 / LayeredMemory.to_dict
        ↓
   normalize_memory_state(state, workspace_root)   ← 一切都从这过
        ↓
   顺手读一次 .pico/memory/MEMORY.md
```

---

## 5. 核心函数逐个拆


### 5.1 `normalize_memory_state`（334–424）—— 枢纽

**备注一：它是破坏性的。**

```python
def normalize_memory_state(state, workspace_root=None):
    if state is None:
        state = default_memory_state()
    elif not isinstance(state, dict):
        raise TypeError("memory state must be a mapping")
    ...
    state["working"] = working        # ← 原地改传入的 dict
    ...
    return state
```

**不是返回一个新对象，是改传入的那个。** 所以 `LayeredMemory.to_dict()` 里那句 `self.state = normalize_memory_state(self.state, ...)` 的赋值其实是冗余的——对象没换。

**为什么要这样？** 因为它是"归一化"而不是"转换"：目的是让传进来的东西符合当前预期。原地改省掉一次深拷贝，代价是**没有纯函数保证**——任何持有 state 引用的地方都会看到变化。

**备注二：五个字段各自的归一化策略不同。**

| 字段                     | 策略                                                     | 代码           |
| ---------------------- | ------------------------------------------------------ | ------------ |
| `working.task_summary` | `clip(..., 300)`，空则回退到旧字段 `task`                       | 347, 357–358 |
| `working.recent_files` | `canonicalize_path` 逐个规范化 → 去重保序 → **只留最后 8 个**        | 348–354      |
| `episodic_notes`       | 空则从旧字段 `notes` 转换；否则逐条 `_normalize_note`；**只留最后 12 条** | 368–386      |
| `file_summaries`       | 逐条规范化；**空文本或空路径直接丢弃**                                  | 388–410      |
| `next_note_index`      | `max(旧值, 现存笔记最大索引 + 1)`                                | 412–416      |

**`recent_files` 的 `[-WORKING_FILE_LIMIT:]` 是"留最新的"不是"留最旧的"**——因为 `remember_file` 每次把新文件 append 到末尾，所以列表尾部总是最新。**列表顺序本身就是时间序**，不需要额外的时间戳字段。

**`next_note_index` 的 `max` 那行很关键**：

```python
max_index = max([note["note_index"] for note in episodic_notes], default=-1)
state["next_note_index"] = max(next_note_index, max_index + 1)
```

它保证新笔记的 `note_index` 一定大于所有现存笔记。如果从旧数据加载时 `next_note_index` 丢了（默认 0），这行会把它修正到"现存的编号之上"，**避免新笔记和旧笔记撞号**。

**备注三：`file_summaries` 没有数量上限。**

`FILE_SUMMARY_LIMIT = 6` 只在 `render_memory_text` 里用作切片，**存储层完全不限**。实测写入 100 个文件摘要后，100 条全留着。

而 `file_summaries` 是 6 个键里唯一"只增不减"的（`episodic_notes` 有 12 条上限、`recent_files` 有 8 个上限）。**只有 `invalidate_stale_file_summaries` 会删它，而且只删"哈希对不上"的。**

所以：**一个会话里读过 500 个不同的文件，`file_summaries` 就留 500 条**，每条最多 500 字符 + 64 字符哈希。


### 5.2 `DurableMemoryStore`（59–224）—— 唯一有人工"智能"的地方

**备注一：它读写的是人能直接编辑的 markdown。**

```
.pico/memory/
├── MEMORY.md                  # 索引
└── topics/
    ├── project-conventions.md
    ├── key-decisions.md
    ├── dependency-facts.md
    └── user-preferences.md
```

实测 `promote` 之后 `MEMORY.md` 的真实内容：

```markdown
# Durable Memory Index

- [project-conventions](topics/project-conventions.md): Project Conventions
  - summary: Stable repository conventions.
  - tags: convention
```

topic 文件的格式（`_write_topic`，171–186）：

```markdown
# Key Decisions

- topic: key-decisions
- summary: Long-lived decisions and rationale anchors.
- tags: decision
- updated_at: 2026-09-17T03:11:46.056246+00:00

## Notes
- Retry budget is 3 attempts
- Timeout value is 30 seconds
```

**这个设计是刻意的**：durable memory 是"项目级的长期知识"，用户可能想**手动编辑**——加一条约定、改一条决策。用 markdown 而不是 JSON，就是为了这个。

**代价**：解析靠正则（`load_index` 的 `re.match(r"- \[([^\]]+)\]\([^)]+\):\s*(.+)", line)`）。手写一个格式不对的行，那条主题就会**静默消失**。

**备注二：`promote` 的三层去重。**

```python
for topic, note_text in promotions:
    ...
    existing = topic_notes.setdefault(topic, [])
    if note_text in existing:                              # ① 完全相同 → 跳过
        continue
    new_subject = self._subject_key(note_text)
    replaced = False
    if new_subject:
        for index, old_text in enumerate(list(existing)):
            if self._subject_key(old_text) == new_subject: # ② 同主语 → 替换
                superseded.append(...)
                existing[index] = note_text
                replaced = True
                break                                      # ③ 只替换第一个
    if not replaced:
        existing.append(note_text)                         # ④ 否则追加
```

实测三层都有效，但**第 ③ 行的 `break` 有个后果**：如果历史里已经积累了多条同主语的笔记，只会替换掉**第一条**，剩下的原样留着。（实测见 §7.13）

**备注三：`promote` 遇到没预定义的主题会崩。**

```python
meta = DURABLE_TOPIC_DEFAULTS[topic]     # 196 行，直接下标取值
```

实测 `promote([("build-system", "...")])` → **`KeyError: 'build-system'`**。

为什么现实中没出问题？因为 topic 不是自由输入——`runtime.extract_durable_promotions`（`runtime.py:469`）用**固定的正则表** `DURABLE_MEMORY_LINE_PATTERNS` 从最终答案里抠，抠出来的 topic 必然是那四个之一。

**但这意味着扩展性被锁死了**：想加一个新主题，必须同时改 `DURABLE_TOPIC_DEFAULTS`（本文件）和 `DURABLE_MEMORY_LINE_PATTERNS`（runtime），**两处漏一处就是崩溃或静默丢失**。

**备注四：`_write_*` 是全量重写。**

`promote` 结尾：

```python
self._write_index([topics[slug] for slug in sorted(topics)])
for topic, notes in topic_notes.items():
    self._write_topic(topic, notes)
```

**哪怕只动了一个主题，所有主题文件都会被重写一遍。** 和 `SessionStore.save` 是同一个手法。在这个规模下（4 个主题、几十条笔记）没问题，但形态上是 O(主题数) 而不是 O(改动数)。

**备注五：durable 笔记的 `created_at` 全都一样。**

`load_topic_notes`（110–111, 120）从文件头读 `- updated_at:`，然后**给该文件里所有笔记都盖上这个时间戳**。

实测：两条在不同时间 promote 的笔记，读回来时间戳完全相同。

**后果**：召回排序里 `recency` 这一档对 durable 笔记**完全失效**（所有笔记并列），实际退化成按 `note_index` 排——而 durable 的 `note_index` 固定是 `-1`，所以**也并列**。最终 durable 笔记之间的顺序取决于 `sort` 的稳定性 + 加载顺序（即文件里的书写顺序）。


### 5.3 `retrieval_candidates`（519–547）—— 召回

**备注一：打分是三元组，没有 embedding。**

```python
```

排序规则是**字典序优先级**：先比 tag 命中，再比关键词数，再比时间，最后比序号。

**注释里说得很直白**：

> 召回逻辑故意保持简单透明：先看 tag 精确命中，再看关键词重叠，最后看新旧程度。**这里不引入 embedding。**

**取舍**：没有模型依赖、没有网络、结果可解释（能说清"为什么这条被召回了"）、快。代价是**只认字面重叠**——"缓存"和"cache"是两个不同的词，永远匹配不上。

**备注二：token 化的正则决定了召回的边界。**

`_tokenize` 用 `r"[A-Za-z0-9_]+"`——**中文被完全丢弃**。

实测影响：`"缓存穿透"` 和 `"缓存击穿"` 的 token 集合都是 `{'cache'}` 之外的东西……等下，实际上中文部分被丢光，只剩 ASCII 部分。所以：

- 查询 `"改一下 timeout 配置"` → tokens = `{'timeout'}`
- 笔记 `read_file 摘要: timeout = 30` → tokens 含 `timeout` → **能召回** ✓
- 查询 `"改一下超时配置"` → tokens = `{}`（中文全丢）→ **召回不到任何东西** ✗

**这是中文场景下召回失效的根因**，和 §3.4 的 `_subject_key` 是同一个根因。

**备注三：durable 的召回在函数内部重新构造了 store。**

```python
if workspace_root is not None:
    durable_store = DurableMemoryStore(Path(workspace_root) / ".pico" / "memory")
    for note in durable_store.retrieval_candidates(query, limit=limit):
```

注意它调的是 `DurableMemoryStore.retrieval_candidates`（144–159 行）——**和模块级的同名函数是两套逻辑**，durable 那版只比 `(exact_tag_match, keyword_overlap, recency)`，**没有 `note_index`**。所以两边的元组长度不同（4 元 vs 3 元），但它们**在同一个列表里排序**：

```python
ranked.append(((exact_tag_match, keyword_overlap, recency, -1), note))   # durable 补了个 -1
```

**这是刻意补齐的**——durable 那版返回的 note 会被重新打分并补上 `-1`，保证元组等长可比。`-1` 比任何真实序号都小，所以**同分时 durable 排在 episodic 之后**，也就是"会话内的新鲜事优先于长期知识"。这个选择合理，但代码里没有注释说明。

**备注四：`ranked[:limit]` 是硬截断。**

`limit` 由调用方给。`context_manager` 传的是 `RELEVANT_MEMORY_LIMIT = 3`。所以**durable 和 episodic 在抢同样 3 个位置**。


### 5.4 `render_memory_text`（561–586）—— 模型看到的仪表盘

```python
lines = [
    "Memory:",
    f"- task: {state['working']['task_summary'] or '-'}",
    f"- recent_files: {', '.join(state['working']['recent_files']) or '-'}",
]
summaries = []
for path in state["working"]["recent_files"][:FILE_SUMMARY_LIMIT]:
    summary = state["file_summaries"].get(path, {})
    current_freshness = file_freshness(path, workspace_root)
    if summary.get("summary", "") and summary.get("freshness") == current_freshness:
        summaries.append(f"- {path}: {summary['summary']}")
...
lines.append(f"- episodic_notes: {len(state['episodic_notes'])}")
lines.append(f"- durable_topics: {', '.join(durable_topics) or '-'}")
```

**备注一：这里做了第三次哈希校验。**

`summary.get("freshness") == current_freshness` —— 渲染时**又算了一遍文件的 sha256**（前两次是 `invalidate_stale_file_summaries` 和 checkpoint 的 `key_files`）。

**这是"双保险"还是"重复劳动"？** 我认为是**必要的**：因为 `invalidate_stale_file_summaries` 只在 `evaluate_resume_state` 时跑（会话启动），而渲染是每轮都跑。

**但代价很实在**：每轮渲染都对前 6 个最近文件做一次完整 `read_bytes + sha256`。而这 6 个文件里可能有刚被大文件读过的。

**备注二：只渲染前 6 个摘要，而 `recent_files` 有 8 个。**

实测：`recent_files` 装满 8 个时，只有 **6 个**摘要能进 prompt，**最后 2 个（也就是最新的 2 个！）被截掉**。

**这是个可疑的取舍**：列表尾部是最新文件，而切片 `[:6]` 取的是**最旧的 6 个**。如果意图是"最新优先"，应该是 `[-6:]`。

对照 `WORKING_FILE_LIMIT=8` 和 `FILE_SUMMARY_LIMIT=6` 两个常量，这看起来像是故意的"少渲染一点"，但方向反了。

**备注三：`episodic_notes` 只输出数量，不输出内容。**

```python
lines.append(f"- episodic_notes: {len(state['episodic_notes'])}")
```

实测渲染结果里有 `- episodic_notes: 0` 这样的行。

**这是刻意的**（注释 563–564 行）：

> 这里渲染的是给模型看的紧凑"仪表盘"，不是完整回放。笔记正文默认不展开，**只有在相关召回时才按需拿出来。**

**这是分层的关键设计**：笔记有 12 条 × 500 字符 = 最多 6000 字符。全塞进固定位置的 memory 段会挤爆预算；只在召回时拿 3 条，就把带宽花在刀刃上。

**代价**：模型知道"我有 12 条笔记"但看不到内容，**它不知道这 12 条是什么，也就不知道要不要主动去问**。仪表盘上显示"有 12 条笔记"是个信息，但不足以行动。

### 5.5 `LayeredMemory`（599–659）—— 门面

**备注：它把模块级函数包成方法，并统一传 `workspace_root`。**

```python
def append_note(self, text, tags=(), source="", created_at=None, kind="episodic"):
    self.state = append_note(self.state, text, tags=tags, source=source,
                             created_at=created_at, workspace_root=self.workspace_root,
                             kind=kind)
    return self
```

每个方法都 `return self`，支持链式调用。设计上是个标准的"状态持有者 + 方法转发"门面。

**关键价值在于它消除了 `workspace_root` 的漏传风险**——runtime 只需要 `agent.memory.append_note(...)`，不用每次都记得传 root。

**但这个风险没有被完全消除**（见 §7.12）：模块级的 `normalize_memory_state(state)` 不带 root 会让 `durable_topics` 被清空，而 `evaluator.py:476` 就有一个不带 root 的调用。

---

## 6. 设计理由

### 6.1 为什么不直接把 session history 全塞进 prompt？

**好处**：记忆层是**有损压缩**。12 条笔记 × 500 字符 + 8 个路径 + 6 个摘要，总共几千字符封顶；而 history 可以无限长。而且它做的是**提炼**——`summarize_read_result` 把一次 read_file 的输出压成 3 行。

**代价**：丢信息，且**丢得不可逆**。`read_file` 的完整输出只留在 history 里，记忆层只有摘要。如果摘要丢了关键行，模型只能重新读文件。

**取舍判断**：这是对的。`context_manager` 给 history 的预算是 5200、给 memory 的只有 1600——**架构上已经决定了"历史的细节可以丢，记忆的结构不能丢"**。

### 6.2 为什么 `file_summaries` 存哈希而不是存"读过的版本"？

**好处**：64 字节 vs 全文。而且哈希是**单向可比**的——能判断"变没变"，但不需要保存原文。

**代价**：每次校验都要**重新读一遍文件**算哈希（`read_bytes` + `sha256`）。实测一次 `render_memory_text` 就对最多 6 个文件做这事，而 `retrieval`、`checkpoint` 各自还有一处。

**取舍判断**：对本地小项目可以接受。**但要注意这是个 O(文件大小) 的操作，且分布在三条不同的路径上**——大仓库要换"大小 + mtime"这种廉价近似。

### 6.3 为什么召回不引入 embedding？

代码注释里写的是"故意保持简单透明"。

**好处**：无模型依赖（pico 的定位是零第三方依赖）、无网络、无延迟、**结果完全可解释**——能说清"这条被召回是因为 tag 命中了 `convention`"。

**代价**：**只认字面重叠**。"缓存" 和 "cache"、"重试" 和 "retry" 永远匹配不上，而这两组词在中文开发场景里是高频混用的。

**取舍判断**：透明性在这个规模下比召回率重要——因为召回结果会进 prompt，**"为什么这条在这里"必须能解释**，否则调优就是瞎猜。但字面匹配的边界（§7.14）应该在文档里写明。

### 6.4 为什么 durable 层用 markdown 而不是 JSON？

**好处**：**人类可读可编辑**。项目级的长期知识（"这个仓库用 4 空格缩进"）本来就该让人能直接加一条。

**代价**：解析靠正则，格式脆弱。手写一行格式不对的 markdown，那条主题会静默消失（`load_index` 的 `re.match` 不匹配就跳过）。

**取舍判断**：合理，**但"静默消失"这个失败模式不好**——宁可报个警告说"第 N 行解析不了"。对一个人会碰的文件，宽容解析 + 明确告警比严格正则更合适。

### 6.5 为什么每次操作都要 `normalize` 一遍？

**好处**：**调用方永远不用关心状态是不是"干净"的**。从磁盘读来的旧状态、被外部改过的状态、刚创建的空状态，进任何函数都先被整理成统一形状。这让所有下游代码可以无条件相信结构。

**代价**：**每次 normalize 都读一次磁盘**（§3.5），而 normalize 在几乎每条路径上。一次 `retrieval_candidates` 就触发 2 次索引读取。

**取舍判断**：**这个取舍在"防止状态不一致"上是对的，但实现把无关的 IO 塞进了归一化。** 更好的做法是把 `durable_topics` 的获取从 `normalize_memory_state` 里拿出来——它不是"归一化"，是"查询"。

### 6.6 为什么要三套镜像字段？

**好处**：读旧数据时不丢信息。旧格式只有 `task`/`files`/`notes`，新代码从任意一种都能恢复。

**代价**：**文件体积翻倍 + 每次都要同步三处**。而且旧字段在新代码里**没有任何读者**（所有读取都走 `working`/`episodic_notes`）。

**取舍判断**：迁移做了一半。正确做法是"读入时拉平，之后删掉旧键"。现在这样相当于**为了兼容一次性迁移，永久付出了双倍存储**。

---

## 7. 实测验证

用真实 `LayeredMemory` + 真实临时工作区跑，不拉模型。

### 7.1 三套镜像字段恒等

```
working.task_summary : '给配置模块加超时参数'
task                 : '给配置模块加超时参数'
working.recent_files : ['a.py']
files                : ['a.py']
episodic_notes[0].text: '项目约定用 4 空格缩进'
notes                : ['项目约定用 4 空格缩进']
```

### 7.2 `clip` 结果长度超出 limit

```
clip(text, 300) -> 实际长度 326   结尾 'xx\n...[truncated 2700 chars]'
clip(text, 500) -> 实际长度 526   结尾 'xx\n...[truncated 2500 chars]'
```

### 7.3 `_subject_key` 的句式覆盖

| 输入                                          | subject              |
| ------------------------------------------- | -------------------- |
| `The build system uses Makefile`            | `'system the build'` |
| `Retry budget is 3 attempts`                | `'budget retry'`     |
| `Cache entries are invalidated on write`    | `'entries cache'`    |
| `Users should run make check before commit` | `'users'`            |
| `Config file is referenced by the loader`   | `'config file'`      |
| `构建系统使用 Makefile`                           | **`None`**           |
| `超时时间是 30 秒`                                | **`None`**           |
| `prefer tabs`                               | `None`               |
| `always run tests first`                    | `None`               |
| `Deploy needs SSH key`                      | `None`               |

**10 个里 5 个提取不出主语**，且**所有中文全部失败**。

### 7.4 `subject_key` 跨进程不稳定

三种 `PYTHONHASHSEED` 下跑同一个句子：

```
seed 1 -> ['the system build', 'retry budget', 'config file']
seed 2 -> ['build system the', 'budget retry', 'config file']
seed 3 -> ['system the build', 'retry budget', 'file config']
```

根因：`" ".join(_tokenize(match.group(1)))` —— `_tokenize` 返回的是 **set**，`join` 遍历 set 的顺序依赖字符串哈希，而 Python 默认开启哈希随机化。

**重要澄清**：这个不稳定**当前不造成功能问题**——因为 `_subject_key` 的结果**只在进程内存活**（`promote` 里的 old/new 都在同一次调用中计算），从不落盘。所以"看起来会失效"和"实际会失效"是两回事，不能混为一谈。

**但它仍是个隐患**：如果哪天把这个 key 持久化（比如做成去重索引），立刻失效。改法很简单——`" ".join(sorted(_tokenize(...)))`。

### 7.5 `promote` 遇到未定义主题崩溃

```
promote([("build-system", "The build system uses Makefile")])
-> KeyError: 'build-system'
可选主题: ['project-conventions', 'key-decisions', 'dependency-facts', 'user-preferences']
```

`_write_topic` 同样：

```
_write_topic('custom-topic', ['x']) -> KeyError 'custom-topic'
```

### 7.6 容量上限的不对称

```
WORKING_FILE_LIMIT = 8   FILE_SUMMARY_LIMIT = 6
recent_files 数量: 8
渲染出来的摘要条数: 6   ← 最早写入的 6 个，最新的 2 个被截掉

写入 100 个文件摘要后 file_summaries 条数: 100   ← 存储层无上限
```

### 7.7 `normalize` 每次都读磁盘

```
构造 LayeredMemory           -> normalize 1 次, 读索引 1 次
append_note 一次             -> normalize 1 次, 读索引 1 次
to_dict 一次                 -> normalize 1 次, 读索引 1 次
render_memory_text 一次      -> normalize 1 次, 读索引 1 次
retrieval_candidates 一次    -> normalize 1 次, 读索引 2 次   ← 自己也构造了一个 store
```

### 7.8 durable 笔记时间戳全部相同

```
'Retry budget is 3 attempts'  created_at = 2026-09-17T03:11:46.056246+00:00
'Timeout value is 30 seconds' created_at = 2026-09-17T03:11:46.056246+00:00
```

两条笔记分别 promote，读回来时间戳一模一样（都取文件头的 `updated_at`）。**召回排序里的 `recency` 档对 durable 完全失效。**

### 7.9 `promote` 的正常替换路径有效

```
promote("Retry budget is 3 attempts")  -> 写入
promote("Retry budget is 5 attempts")  -> superseded: ['key-decisions: Retry budget is 5 attempts -> ...']
```

### 7.10 `is_effectively_empty` 返回 True

```
工作记忆全空，但 durable 里已有笔记 -> is_effectively_empty = True
render 输出: Memory: | - task: - | - recent_files: - | - file_summaries: - | - episodic_notes: 0 | - durable_topics: project-conventions
```

**判定为空，但 `render` 里明明列出了 durable 主题。** 两者对"空"的定义不一致。

而 `evaluator.py:476` 用这个函数判断 `initial_memory_empty` 作为评测基线。

### 7.11 不带 `workspace_root` 会清空 `durable_topics`

```
带 root  : durable_topics = ['project-conventions']
不带 root: []                                    ← 被重置成空
```

归一化函数（421–423 行）在 `workspace_root is None` 时直接把 `durable_store` 设为 `None`，于是写入空列表。

**`evaluator.py:476` 就是一次不带 root 的调用**：`memorylib.is_effectively_empty(initial_memory_state)`。

### 7.12 durable 路径一致性

三处构造 `DurableMemoryStore` 的路径一致，都是 `<workspace_root>/.pico/memory`：

- `normalize_memory_state`（421）
- `LayeredMemory.__init__`（603）
- `retrieval_candidates`（537）

实测写入位置：`<ws>/.pico/memory/MEMORY.md` ✓

### 7.13 supersede 只替换第一条

手工写入两条同主语笔记后 promote 一条新的：

```
写入前: 'Retry budget is 3 attempts'  subject = 'retry budget'
        'Retry budget is 7 attempts'  subject = 'retry budget'
promote('Retry budget is 9 attempts')
superseded = ['key-decisions: Retry budget is 3 attempts -> Retry budget is 9 attempts']
结果:  'Retry budget is 9 attempts'
       'Retry budget is 7 attempts'      ← 重复项原样保留
```

第 217 行的 `break` 让循环只替换第一个匹配。

### 7.14 中文笔记永远去重失败

```
中文  '构建系统使用 Makefile'   -> subject None
中文  '重试预算是 3 次'         -> subject None
中文  '超时时间是 30 秒'        -> subject None
中文  '用户偏好四个空格缩进'      -> subject None
英文  'Build system uses Makefile' -> subject 'system build'
英文  'Retry budget is 3 attempts' -> subject 'retry budget'
```

**根因**：句式正则是匹配的（`^(.+?)使用.+$` 确实匹配上了），但 `_tokenize` 用 `[A-Za-z0-9_]+` 提取主语时**把中文全丢了**，得到空集，于是 `subject or None` 返回 `None`。

**后果**：所有中文 durable 笔记永远不会互相替换，只会不断追加。

**同样影响召回**：查询换成纯中文（`"改一下超时配置"`）时 `query_tokens` 是空集，`exact_tag_match == 0 and keyword_overlap == 0` → **一条都召不回来**。

---

## 8. 不变量

| #   | 不变量                                             | 验证方式       | 实测                     |
| --- | ----------------------------------------------- | ---------- | ---------------------- |
| I1  | `task` ≡ `working.task_summary`                 | 改完后比对      | ✓                      |
| I2  | `files` ≡ `working.recent_files`                | 同上         | ✓                      |
| I3  | `notes` ≡ `[n["text"] for n in episodic_notes]` | 同上         | ✓                      |
| I4  | `len(working.recent_files) ≤ 8`                 | 塞 20 个     | ✓                      |
| I5  | `len(episodic_notes) ≤ 12`                      | 塞 30 条     | ✓                      |
| I6  | `next_note_index > max(note_index)`             | 检查         | ✓（`max(...)+1`）        |
| I7  | 非空路径且非空文本才进 `file_summaries`                    | 塞空值        | ✓（403–404 行过滤）         |
| I8  | `len(file_summaries)` 无上限                       | 塞 100 个    | ✗ **被推翻**，100 条全留      |
| I9  | `render_memory_text` 输出的摘要数 ≤ 6                 | 塞 8 个      | ✓                      |
| I10 | durable 主题必属四个预定义之一                             | 传自定义 topic | ✗ **被推翻**，`KeyError`   |
| I11 | `_subject_key` 对同一输入返回同一结果                      | 换进程重跑      | ✗ **被推翻**，三种 seed 三种结果 |
| I12 | 中文文本能提取出 `_subject_key`                         | 中文句子       | ✗ **被推翻**，全部 `None`    |
| I13 | 归一化是纯函数（不改传入对象）                                 | 检查引用       | ✗ **被推翻**，原地改          |
| I14 | `normalize(state)` 不产生副作用                       | 统计索引读取     | ✗ **被推翻**，每次都读         |

**14 条里 6 条被推翻**——这个模块的"看起来是"和"实际是"之间距离不小。

---

## 9. 精读顺序建议

1. **`default_memory_state`（43–56）** —— 先看这 7 个键。立刻会注意到有三对是镜像的，这时就该问"为什么"。
2. **`normalize_memory_state`（334–424）** —— 枢纽。分四段读：`working` / `episodic_notes` / `file_summaries` / 末尾的镜像写回 + 磁盘读取。**最后 7 行是全文件信息密度最高的地方。**
3. **`DurableMemoryStore.promote`（188–224）** —— 唯一有"智能"的地方。三层去重逐条看，注意 `break` 的位置。
4. **`_subject_key`（126–142）** —— 配合 `_tokenize`（282–283）一起读。**两个函数加起来 18 行，决定了 durable 层能不能去重。**
5. **`retrieval_candidates`（519–547）** —— 看四元组打分和 durable 的 `-1` 补齐。
6. **`render_memory_text`（561–586）** —— 唯一决定模型看到什么的函数。注意它**不输出笔记内容**。
7. **回到调用方** —— `runtime.py:405–418`（工具结果怎么写进记忆）、`runtime.py:469–497`（durable 提升）、`context_manager.py:120–121`（召回怎么用）。

跳过的：路径工具三件套（250–279）、`_ensure_list`/`_dedupe_preserve_order`、`LayeredMemory`（纯转发）。

---

## 10. 已知粗糙处

按约定**只报告，不修改**。

### 10.1 `_tokenize` 只认 ASCII，中文全面失效

`re.findall(r"[A-Za-z0-9_]+", ...)` 丢弃所有非 ASCII 字符。连锁后果有两处：

- `_subject_key` 对中文返回 `None` → **中文 durable 笔记永不去重**
- `retrieval_candidates` 的 `query_tokens` 为空 → **纯中文查询召回不到任何东西**

影响面：durable 去重 + 召回匹配。**这是本文件影响最大的一个缺陷。**

### 10.2 `file_summaries` 无数量上限

`FILE_SUMMARY_LIMIT = 6` 只用于渲染切片，不限制存储。实测 100 条全留。而它是唯一"只增不减"的层（另外两层都有硬上限）。

### 10.3 渲染取的是最旧的 6 个，不是最新的

`state["working"]["recent_files"][:FILE_SUMMARY_LIMIT]` —— 列表尾部才是最新文件，`[:6]` 取的是头部。实测 8 个文件时，**最新的 2 个摘要进不了 prompt**。

若意图是"最新优先"，应改为 `[-6:]`。

### 10.4 `_subject_key` 的 `" ".join(set)` 顺序不稳定

跨进程实测三种结果。当前不影响功能（key 不落盘），但写法本身不成立，且是持久化时的定时炸弹。改法：`" ".join(sorted(...))`。

### 10.5 `promote` / `_write_topic` 对未定义主题 `KeyError`

`DURABLE_TOPIC_DEFAULTS[topic]` 直接下标。加新主题必须同时改本文件和 `runtime.DURABLE_MEMORY_LINE_PATTERNS`，**两处漏一处即崩**。

### 10.6 supersede 只替换第一条

第 217 行的 `break`。历史里积累多条同主语笔记时，只替换第一条。

### 10.7 `normalize_memory_state` 有磁盘 IO 且是破坏性的

名字是"归一化"，实际：(a) 原地修改传入的 dict，(b) 每次读一次 `MEMORY.md`。13 个调用点全部继承这两个性质。

### 10.8 不带 `workspace_root` 会清空 `durable_topics`

`workspace_root is None` 时直接写入 `[]`。`evaluator.py:476` 存在这样的调用。

### 10.9 durable 笔记时间戳全部相同

`load_topic_notes` 把文件头的 `updated_at` 盖给该文件所有笔记。导致召回排序里的 `recency` 档对 durable 完全失效。

### 10.10 `is_effectively_empty` 与 `render_memory_text` 对"空"的定义不一致

前者不看 durable，后者输出 `durable_topics`。实测可以同时"判定为空"和"渲染出主题名"。

### 10.11 三套镜像字段

`task`/`files`/`notes` 与 `working.*`/`episodic_notes` 永久双写双存，体积翻倍，且旧键在新代码里无读者。属于"迁移做了一半"。

### 10.12 `clip` 的结果长度超出 limit

`clip(x, 300)` 实测 326 字符。而这个结果会进 prompt，所以"300 上限"名不副实。与 `context_manager._tail_clip`（严格等于 limit）语义不同，两者容易被混用。

### 10.13 `_write_*` 全量重写

`promote` 一次会重写索引 + 所有 topic 文件，即使只动了一个主题。

---


## 11. 面试问答

**Q：agent 的记忆是怎么做的？**

> 分两层。

> 底下是**完整历史**，就是每轮的原始对话，存在 session 文件里。上面是**工作记忆**，是一份被反复提炼的小抄——当前任务摘要、最近碰过哪些文件、每个文件的短摘要、还有十几条会话内笔记。



> 还有个跨会话的长期层，存成 markdown 文件，放项目目录下的 `.pico/memory/` 里，分成"项目约定""关键决策""依赖事实""用户偏好"四个固定主题。

**设计上的关键取舍是**：记忆层是**有损**的。它不存文件全文，只存三行摘要加一个内容哈希。下一轮 prompt 里带的是这份小抄，不是完整历史。

**Q：为什么要存哈希？**

> 用来判断"这个文件我读过的内容还作不作数"。



> 每次要给模型拼 prompt 之前，拿记忆里存的哈希和文件现在的哈希比一下。不一样，说明文件被人改了，那条摘要就不能用了，要么重新读要么直接丢掉。

> **存哈希不存原文**，是因为哈希只有 64 字节，而文件可能几十 KB。而且哈希是单向可比的——只要哈希一样，内容就一定一样。

**代价是每次校验都要把文件完整读一遍算哈希**。本地小项目文件都不大，扛得住；**换成大仓库这就得换成"比大小和修改时间"这种廉价近似了**。

**Q：召回是怎么做的？**

> 打分，完全不用 embedding。

> 每条笔记算四个数：**标签有没有精确命中**、**关键词重叠了几个**、**什么时候写的**、**是第几条写的**。然后按这个顺序逐级比，取前三名。

> 代码注释里写得很清楚——"故意保持简单透明"。因为召回结果直接进 prompt，**必须能解释清楚"这条为什么在这儿"**，不然调优就是瞎猜。

> 代价是**只认字面重叠**。"缓存"和"cache"、"重试"和"retry"这种同义不同词的，永远匹配不上。

**Q：长期记忆怎么去重？**

> 把每句话压成一个"主语键"。比如"重试预算是 3 次"和"重试预算是 9 次"，主语都是"重试预算"，那就认为是同一件事，用新的替换旧的。

> 具体做法是先用六个正则抠出主语，再用关键词集合表示它。

**这里有两个我做得不好的地方**：

> 一是那个关键词提取**只认英文字母数字**，中文全被丢掉了。结果是**中文句子的主语永远是空的，永远不会互相去重**，只会一条条往上加。而 `_subject_key` 是从英文句式设计的，中文句式虽然写了正则，但走到 token 化那一步就断掉了。

> 二是生成主语键的时候把关键词集合直接拼成字符串，而集合是无序的——**换个进程跑同一个句子会得到不同的键**。现在没出事是因为这个键只在一次调用里比较、不落盘；但只要哪天想把它存下来做索引，立刻就废了。改法很简单，拼之前排个序就行。

**Q：为什么每次操作都要"归一化"一遍？**

> 因为状态的来源太多了——可能是刚创建的空状态、可能是从磁盘读回来的旧格式、可能被别的地方改过。让每个函数自己判断"我这个字段在不在、格式对不对"，代码会到处是防御性判断。

> 统一在一个地方整理，下游就能无条件相信结构。

**但我的实现有个问题**：归一化函数里**顺手读了一次磁盘**——因为要列出长期记忆的主题名。结果就是 `render`、`is_effectively_empty` 这些看起来纯读的函数，每次调用都会碰一次文件。**这两件事应该分开**：一个是真的归一化，一个是查询。

**Q：这套记忆有什么已知的坑？**

> 最实在的三个：

> 1. **文件摘要那一层没有数量上限。** 另外两层都有硬上限（文件 8 个、笔记 12 条），只有它只增不减。一个会话里读过五百个文件，就留五百条。
> 2. **渲染的时候取的是最旧的六个摘要，不是最新的六个**。文件列表是尾部最新，切片却从头部取，**最新的反而被截掉了**。
> 3. **中文召回失效。** 查询词做分词的时候只保留英文数字，纯中文的查询词集合是空的，一条笔记都匹配不上。这个和去重失效是同一个根因。

---


## 附：实测脚手架

`memory.py` 用相对导入（`from ..workspace import clip, now`），**不能**用 `importlib` 单文件加载：

```python
import sys, tempfile
from pathlib import Path
sys.path.insert(0, r"E:\pico_agentharness\pico")
from pico.features import memory as ml
```

**关键路径约定**：`workspace_root` 是**工作区根**，durable 层在 `<workspace_root>/.pico/memory/`。

```python
ws = Path(tempfile.mkdtemp()) / "proj"; ws.mkdir()
store = ml.DurableMemoryStore(ws / ".pico" / "memory")     # 索引在 .pico/memory/MEMORY.md
```

**最容易踩的坑**：给 `normalize_memory_state` 传了非根的路径（比如直接传 `.pico/memory`），代码会再拼一层 `.pico/memory`，导致读不到刚写的内容（§7.12 就是这个坑踩出来的）。

**统计调用次数**：直接替换模块属性即可，因为所有内部调用都走模块全局查找：

```python
_real = ml.normalize_memory_state
def counting(state, workspace_root=None):
    calls["n"] += 1
    return _real(state, workspace_root)
ml.normalize_memory_state = counting
```

**验证哈希随机化**：`PYTHONHASHSEED` 环境变量，在不同进程里读 `os.environ.get('PYTHONHASHSEED')`（注意别在 shell 里用 `$seed` 插值，会被引号吃掉）。

Windows 托管 Python：`C:/Users/yxqyx/.workbuddy/binaries/python/versions/3.13.12/python.exe`
