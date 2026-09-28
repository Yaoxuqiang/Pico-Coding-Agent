# `context_manager.py` 核心逻辑精读

> 对象：`pico/context_manager.py`（509 行，md5 `6326417b312f6689321f87b5116897c2`）  
> 定位：**提炼版**。只讲核心逻辑骨架、设计理由和实测证据；逐行的完整源码走读见 `context-layered-budget-trimming.md`。  
> 本文所有数据来自真实运行，不是估算。

---

## 0. 一句话概括

509 行里真正承载逻辑的只有 **6 个函数**，其余是常量、记账和两段文本渲染。核心就三件事：

| # | 内容                                                                           |
| - | ---------------------------------------------------------------------------- |
| 1 | **两级裁剪**：段内裁剪（每段按自己的额度截）+ 段间收缩（总长超了按优先级砍）                                    |
| 2 | **三条不变量**：可裁段不超额度、当前请求恒等、结束时要么达标要么全部触底                                       |
| 3 | **一份账本**：每段的 `raw / budget / rendered` 三长度，每次收缩的 `before / after / overflow` |



---

## 1. 509 行的分布：逻辑只占三分之一

| 行区间     | 内容                              | 性质     |
| ------- | ------------------------------- | ------ |
| 13–30   | 常量策略表（预算、底线、两个顺序）               | 策略     |
| 33–41   | `_tail_clip`                    | 工具     |
| 44–57   | `SectionRender` 数据类             | 结构     |
| 78–182  | **`build()` 主流程**               | **核心** |
| 184–216 | 无裁剪快路径                          | 旁路     |
| 218–224 | `_compute_section_floors()`     | **核心** |
| 226–241 | `_render_sections()` 分发         | **核心** |
| 243–295 | `_render_relevant_memory()`     | **核心** |
| 297–359 | `_render_history_section()`     | **核心** |
| 361–423 | `_compressed_history_entries()` | **核心** |
| 425–435 | `_raw_history_text()`           | 辅助     |
| 437–442 | `_render_history_item()`        | 辅助     |
| 444–454 | `_assemble_prompt()`            | 拼接     |
| 456–509 | `_metadata()`                   | 记账     |

**关键：真正复杂的是 `history` 那一段（297–423，共 127 行，占全文 25%）。** 其它四处裁剪都是一行 `_tail_clip` 解决。

---

## 2. 五个必须分清的概念

### 2.1 段（section）：五层，但要按"能不能裁"分成两类

| 类别          | 段                           | 裁剪方式               |
| ----------- | --------------------------- | ------------------ |
| **多条独立记录**  | `relevant_memory`、`history` | 结构化裁剪（按条塞，牺牲部分保全部） |
| **一整块连贯文本** | `prefix`、`memory`           | 整体截断（`_tail_clip`） |
| **不可裁**     | `current_request`           | 原样输出               |

这个分类是全部裁剪逻辑的分水岭。**多条记录可以"砍几条"，一整块文本只能"砍掉尾巴"。**

### 2.2 两条顺序：拼接序 ≠ 牺牲序

```python
DEFAULT_REDUCTION_ORDER = ("relevant_memory", "history", "memory", "prefix")   # 牺牲序
SECTION_ORDER = ("prefix", "memory", "relevant_memory", "history", "current_request")  # 拼接序
```

两者**没有任何元素对应关系**，是独立设计：

- **拼接序**服务两个目标：`prefix` 在最前（跨轮稳定 → prompt cache 命中）；`current_request` 在最后（近因效应 → 指令遵循率最高）
- **牺牲序**服务一个目标：按"信息替代成本"从低到高排队。`relevant_memory` 砍了下一轮能重新召回；`prefix` 砍了模型不会调工具，任务直接崩

### 2.3 预算三元组

| 概念                   | 默认值                       | 作用                        |
| -------------------- | ------------------------- | ------------------------- |
| `total_budget`       | 12000                     | 触发第二级裁剪的阈值                |
| `section_budgets[段]` | 3600 / 1600 / 1200 / 5200 | 第一级裁剪的额度，**且是第二级裁剪的操作对象** |
| `section_floors[段]`  | 900 / 400 / 300 / 1300    | 第二级裁剪的下界                  |

**注意：分段预算之和 = 11600 < 总预算 12000。**  
多出来的 400 是留给分隔符和当前请求的。**总预算没有给当前请求单独留额度** —— 这是"长请求会挤别人、不挤自己"的机制来源。

### 2.4 `SectionRender`：三个长度

```python
@dataclass
class SectionRender:
    raw: str          # 没裁之前的原文
    budget: int       # 这轮给它的额度
    rendered: str     # 实际输出的内容

    @property
    def raw_chars(self): return len(self.raw)
    @property
    def rendered_chars(self): return len(self.rendered)
```

**三个长度是这套机制的"仪表盘"。** 压缩率 = `1 - rendered/raw`；超没超额度 = `rendered vs budget`。后续做 ablation 实验时，所有指标都从这三个数里算。

### 2.5 底线不是配置，是算出来的

```python
def _compute_section_floors(self):
    floors = {section: max(20, int(budget) // 4) for section, budget in self.section_budgets.items()}
    floors.update(self._section_floor_overrides)
    return floors
```

**底线 = 上限的 1/4，但不低于 20。** `max(20, ...)` 是兜底：评测里会把上限改成 120，此时 `120//4 = 30` 还够，但如果上限改成 50，`50//4 = 12` 就低于 20，会被抬到 20。

**为什么是上限的 1/4？** 这是"这段还能读出意思"的经验值。底线防的不是超预算，是**结构崩坏** —— 宁可 prompt 整体超标，也不能让某一段变成空壳。

---

## 3. 主流程骨架

```
build(user_message):
  │
  ├─ 1. 重算底线            ← 因为 section_budgets 可能被外部改过（评测就这么干）
  ├─ 2. 读三个特性开关
  │       context_reduction 关 → 走无裁剪快路径，直接返回
  ├─ 3. 攒原文（完全不裁）    ← "意图"
  │       prefix / memory / current_request 各就各位
  │       history 先留空占位
  ├─ 4. checkpoint 文本并入 prefix
  ├─ 5. 先召回，后裁剪        ← 召回 limit=3，决定"带什么"
  │
  ├─ 6. 【第一级】_render_sections(budgets)   ← 段内裁剪，决定"带多少"
  ├─ 7. _assemble_prompt()
  │
  ├─ 8. 【第二级】while len(prompt) > total_budget:
  │         overflow = len(prompt) - total_budget
  │         for section in reduction_order:          ← 按牺牲序
  │             if budget[section] <= floor: continue
  │             budget[section] = max(floor, budget[section] - overflow)
  │             重渲染 + 重新测长
  │             break                                ← 一次只砍一段
  │         如果一段都没砍动 → break（认输）
  │
  └─ 9. _metadata()  记账
```

**顺序上的两个刻意设计：**

1. **先召回再裁剪**（第 5 步在第 6 步之前）。反过来的话，预算会被历史吃光，召回的笔记无处可放。
2. **第 3 步攒原文时刻意不裁。** "意图"和"结果"分开存，才能算出压缩率。

---

## 4. 第一级：段内裁剪的三种策略

### 4.1 整体截断 —— `prefix` / `memory`

```python
rendered_text = _tail_clip(raw, int(budget)) if budget is not None else raw
```

`_tail_clip` 的行为：

```python
def _tail_clip(text, limit):
    text = str(text)
    if limit <= 0:
        return ""
    if len(text) <= limit:
        return text
    if limit <= 3:
        return text[:limit]
    return text[: limit - 3] + "..."     # 留头 limit-3 字符 + "..."
```

**名字叫 `tail`，行为却是留头。** 这个命名与实现不一致，是全文最容易读错的一处。

### 4.2 按条均分 —— `relevant_memory`

```python
per_note_budget = self._per_note_budget(budget, len(note_texts), header)
while True:
    rendered_notes = [_tail_clip(text, per_note_budget) for text in note_texts]
    rendered = "\n".join([header] + [f"- {text}" for text in rendered_notes])
    if len(rendered) <= budget or per_note_budget <= 1:
        break
    per_note_budget -= 1
```

```python
def _per_note_budget(self, budget, note_count, header):
    overhead = len(header) + 3 * note_count      # header + 每条 "- " 和换行
    usable = max(0, budget - overhead)
    return max(1, usable // note_count)
```

**为什么均分而不是整体截断？**  
`Relevant memory:` 下面挂的是最多 3 条互相独立的笔记。整体截断 → 第一条长笔记吃掉全部预算，后面两条直接消失，**召回 3 条等于只召回 1 条**。均分 → 3 条各留一点，**多样性**保住了。

**取舍很明确：宁可 3 个半条线索，也不要 1 个完整线索。** 因为用户请求里的关键词可能只和其中一条真正相关，被砍掉的那条往往才是有用的。

`while True` 是确定性收敛：每轮 `per_note_budget -= 1`，最多迭代初始值次，不死循环。


### 4.3 从新往旧填 + 四级降级 —— `history`（最复杂的一段）

**填充逻辑：**

```python
for entry in reversed(history_entries):     # ← 从最新一条开始
    candidate_lines = list(entry.get("lines", []))
    candidate_entries = candidate_lines + rendered_entries
    if len("\n".join(["Transcript:", *candidate_entries])) <= budget:
        rendered_entries = candidate_entries
        continue
    ...
```

**注意 `reversed()` + 前置拼接的组合效果**：迭代顺序是最新→最旧，但每条新的（更旧的）候选被放在 `rendered_entries` **前面**，所以最终输出仍是时间正序。

推演（A 最旧、C 最新）：
```
history_entries = [A, B, C]
reversed → C, B, A
  迭代 C: [C]
  迭代 B: [B, C]
  迭代 A: [A, B, C]   ← 最终时间正序
```

**四级降级：** 同一个条目在四个压缩等级里逐级下探。

| 等级 | 处理 | 触发条件 |
| --- | --- | --- |
| L0 完整 | 原文整条 | 塞得下 |
| L1 轻度截断 | 最近 6 条按剩余空间截断，`available = max(20, available - 1)` | 塞不下且是 recent |
| L2 重度截断 | 更旧条目每条压到 20 字符 | 塞不下且非 recent |
| L3 完全折叠 | 在 `_compressed_history_entries` 里就没进列表 | 见下 |

**L3 的三级折叠（`_compressed_history_entries`）：**

```python
if item["role"] == "tool" and item["name"] == "read_file":
    path = str(item["args"].get("path", "")).strip()
    if path in seen_older_reads:          # 折叠 1：重复读同一文件 → 直接丢
        details["collapsed_duplicate_reads"] += 1
        continue
    seen_older_reads.add(path)
    summary = self._reusable_file_summary(path)
    if summary:                            # 折叠 2：换成 memory 里的短摘要
        entries.append({"recent": False, "lines": [f"{path} -> {summary}"]})
        continue

if item["role"] == "tool":                 # 折叠 3：旧工具调用压成一行
    entries.append({"recent": False, "lines": [self._summarize_old_tool_item(item)]})
    continue
```

同一个文件的 `read_file`，成本随位置三档递减：

| 位置 | 处理 | 长度量级 |
| --- | --- | --- |
| 最近 6 条 | 原文 | ~900 |
| 更旧、首次出现 | `path -> memory 里的文件摘要` | ~100 |
| 更旧、重复出现 | 直接删 | 0 |

**折叠 2 是跨模块协作的关键**：`memory` 层已经存过这个文件的摘要，历史层就只放一个指针 `path -> summary`，不重放全文。历史层和 memory 层在这里形成"目录—正文"的分工。

**折叠 3 对 `run_shell` 特殊照顾：** 取前 3 行非空行拼成 `command -> line1 | line2 | line3`；其它工具退化成 `_render_history_item(item, 60)[0]` —— 也就是**只保留 `[tool:name] {args}` 这一行前缀，输出内容整段丢弃**。

---

## 5. 第二级：全局收缩

```python
while len(prompt) > self.total_budget:
    overflow = len(prompt) - self.total_budget
    reduced = False
    for section in self.reduction_order:
        floor = int(self.section_floors.get(section, 0))
        current_budget = int(budgets.get(section, 0))
        if current_budget <= floor:
            continue                                   # 触底就跳过，看下一段
        new_budget = max(floor, current_budget - overflow)
        if new_budget >= current_budget:
            continue
        reduction_log.append({...})
        budgets[section] = new_budget
        rendered = self._render_sections(section_texts, budgets, selected_notes=selected_notes)
        prompt = self._assemble_prompt(rendered)       # ← 立即重渲染 + 重测
        reduced = True
        break                                          # ← 一次只砍一段
    if not reduced:
        break
```

### 5.1 收敛性：为什么必然会停

**证明：**

1. 进入循环的前提是 `overflow > 0`
2. 循环体内选中的段一定满足 `current_budget > floor`（否则 `continue`）
3. 所以 `new_budget = max(floor, current_budget - overflow)`，其中 `current_budget - overflow < current_budget` 且 `floor < current_budget`
4. 因此 `new_budget < current_budget` —— **预算严格单调递减**
5. `Σbudget` 有下界 `Σfloor`，每次迭代至少减少 1
6. 所以最多迭代 `Σ(budget_i - floor_i)` 次

**默认参数下的上界：** `(3600-900) + (1600-400) + (1200-300) + (5200-1300) = 2700 + 1200 + 900 + 3900 = 8700` 次。实际远小于这个数（实测 3–4 次，见下节）。

### 5.2 退出时的两种状态

| 出口 | 条件 | 结果 |
| --- | --- | --- |
| 正常 | `len(prompt) <= total_budget` | 达标，`prompt_over_budget = False` |
| 认输 | 四段全部触底，`reduced` 保持 `False` | **`prompt_over_budget = True`，接受超预算** |

**第二个出口是刻意留的。** 宁可 prompt 超标，也不把某一段压成空壳 —— 这是"底线"概念存在的全部意义。

---

## 6. 实测验证

用桩对象直接跑 `ContextManager`（不依赖模型和网络），两组压力配置：

### case 1：长上下文但可救

输入：`prefix = 5000 字符`、`current_request = 3022 字符`、`history = 40 条（raw 22981 字符）`、3 条笔记。

```
prompt 12000 / budget 12000   over: False

reduction:
   relevant_memory  1200 →  300   (overflow 1038)
   history          5200 → 5062   (overflow  138)
   history          5062 → 5048   (overflow   14)

(raw, budget, rendered):
  prefix           (5000,  3600, 3600)
  memory           (  36,  1600,   36)
  relevant_memory  (1225,   300,  298)
  history         (22981,  5048, 5036)
  current_request (3022,  None, 3022)

current_request raw == rendered : True
prompt 以原始请求结尾           : True
```

**三个观察：**

1. **恰好收敛到 12000。** `prompt 12000 / budget 12000` —— 贪心循环的 `new_budget = current - overflow` 是精确的"差多少砍多少"，所以能刚好压到线上，不会过度。
2. **`history` 被砍了两次（1038 → 138 → 14）。** 这直接证明了"不能批量砍"：预设的 `overflow` 和"砍完后实际缩短的长度"**不是 1:1**，必须重渲染后重新量。
3. **`memory` 只用了 36 / 1600 的额度。** 看下面第 8 节的"预算不回收"。

### case 2：极端压力，必然认输

输入：`prefix = 30000`、`current_request = 20022`、`history = 200 条（raw 115001）`。

```
prompt 22564   over: True   reduction 次数: 4

(raw, budget, rendered):
  prefix           ( 30000,   900,   900)   ← 触底
  memory           (    36,   400,    36)
  relevant_memory  (  1225,   300,   298)   ← 触底
  history         (115001,  1300,  1300)    ← 触底
  current_request ( 20022,  None, 20022)    ← 完整保留

实际生效底线: {prefix: 900, memory: 400, relevant_memory: 300, history: 1300}
请求完整: True

history 折叠统计:
  older_entries_count        : 97
  collapsed_duplicate_reads  : 0
  reused_file_summary_count  : 0
  summarized_tool_count      : 97
```

**三个观察：**

1. **历史从 115001 压到 1300（压缩率 98.9%），97 条旧工具调用全部降级成一行摘要。**
2. **`prefix` 被压到 900 —— 正好是底线，不是 0。** 底线机制生效的直接证据。
3. **prompt 最终 22564 > 12000，但当前请求 20022 字符一字未少。** 这就是"当前请求不被裁坏"的真实代价形态：**整体超标，而不是请求变短。**

---

## 7. 三条不变量（可从 metadata 直接验证）

| # | 不变量 | 验证方式 | 上面实测 |
| --- | --- | --- | --- |
| **I1** | 可裁段的 `rendered_chars ≤ budget_chars` | `meta["sections"]` | case1：3600≤3600、298≤300、5036≤5048 ✓ |
| **I2** | `current_request.rendered_chars == raw_chars`（恒等） | `meta["current_request"]` | 两组都是 `True` ✓ |
| **I3** | 结束时要么 `prompt ≤ total_budget`，要么**所有可裁段都在底线** | `prompt_over_budget` + 各段 `budget == floor` | case2：`over=True` 且四段全在底线 ✓ |

**I2 是三条结构性保证的结果，不是靠小心：**

| 保证 | 位置 |
| --- | --- |
| `DEFAULT_REDUCTION_ORDER` 里**没有**这个 key，收缩循环遍历不到 | 第 27 行 |
| `_render_sections` 里是特判分支，`rendered = raw`，`budget` 恒为 0 | 第 230–232 行 |
| `_metadata` 里 `budget_chars = None`，`rendered_chars` 直接取 `len(user_message)` | 第 464–468 行 |

---

## 8. 为什么这么设计

### 8.1 为什么"先分层，再裁剪"

只做裁剪不做分层，就只能对整段 prompt 一刀切 —— 但 prompt 里混着"工具调用规则"和"三条参考笔记"，它们的价值差了几个数量级。**分层的作用是让每类信息有独立的额度，裁剪才有抓手。**

### 8.2 为什么是贪心逐段收缩，而不是一次算好每段留多少

**因为"预算字符数"和"渲染后字符数"不是线性关系。**

`history` 的渲染是"从新往旧逐条塞"，预算减少 100 字符，可能只缩短 40 字符（因为最后一条可能刚好卡在边界上）。**实测证据**：case1 里 `history` 被砍了两次才收敛，`overflow` 从 138 降到 14。

如果一次算好，就得反推一个非线性的映射函数 —— 复杂度爆炸，还容易算错。贪心的代价是"可能多迭代几次"，换来的是：

- 代码 20 行
- 行为完全可预测
- 每一步都有 `reduction_log` 可解释

**可观测性 > 理论最优。**

### 8.3 为什么要有底线

防止某一段被压成 0。如果 `prefix` 变空，模型看不到工具怎么调；如果 `memory` 变空，模型会忘掉任务是什么。**底线防的是"结构崩坏"，代价是"宁可整体超标"。**

case2 就是这个策略的完整体现：四段全部压在底线，prompt 22564 远超 12000，**系统选择"超标交付"而不是"把 prefix 压没"**。

### 8.4 为什么 `relevant_memory` 第一个被牺牲

按**信息替代成本**排序：

| 段 | 砍了之后能怎么补救 | 位置 |
| --- | --- | --- |
| `relevant_memory` | 下一轮重新召回，**可再生** | 第 1 个砍 |
| `history` | 不可再生，但旧的能降级成摘要 | 第 2 个砍 |
| `memory` | 是"工作集"，丢了会忘掉任务主线 | 第 3 个砍 |
| `prefix` | 丢了模型不会调工具，**任务直接崩** | 最后才动 |

### 8.5 为什么历史要"从新往旧填 + 四级降级"

**从新往旧**：agent 的下一步决策最依赖"刚刚发生的工具结果"（近因效应）。预算优先给最近 6 条。

**四级降级而不是直接删**：直接删会让模型不知道"这个文件我读过"。降级成 `path -> summary` 之后，模型至少知道"读过这个文件、大概是什么"，需要细节时再读一遍。**保住了"发生过什么"的骨架，牺牲的是细节。**

### 8.6 为什么一切都要记长度

因为**不可观测的压缩就是黑盒**。出了问题是"prompt 变短了导致答错"还是"模型本来就答错"，没有数据无法归因。

`raw / budget / rendered` 三长度 + `reduction_log` 的 `before / after / overflow`，让每个问题都能回答：

- 这一轮压了多少？→ `1 - rendered/raw`
- 压的是哪一段？→ `budget_reductions[].section`
- 为什么压它？→ `reduction_order` + 各段触底状态
- 当时超了多少？→ `budget_reductions[].overflow_chars`

**这也是能做 ablation 对照实验的前提** —— 开/关 `context_reduction` 各跑一遍，比的就是这几个数。能算账才能做实验。

---

## 9. 精读顺序建议

| 顺序 | 位置 | 读什么 | 为什么先读它 |
| --- | --- | --- | --- |
| 1 | 13–30 行 | 常量表 | 三个顺序、三个数值，先建立"策略"的整体印象 |
| 2 | 78–182 行 `build()` | 主流程 | 读的时候只看结构，`_render_*` 先当黑盒 |
| 3 | 226–241 行 `_render_sections` | 分发 | 搞清"三类段三种策略" |
| 4 | 218–224 行 `_compute_section_floors` | 底线 | 一句话的事，但决定第二级裁剪的终止条件 |
| 5 | 297–423 行 `history` | 最复杂的一段 | 放到最后，前面四个都懂了这里才不绕 |
| 6 | 456–509 行 `_metadata` | 记账 | 回头对照，看哪些数被记下来了 |

**读的时候抓住一个问题就够了：这个数字（`raw` / `budget` / `rendered`）在整条链路里被谁改过？**

---


## 10. 已知的粗糙处（只记录，不改）

| # | 问题 | 影响 | 位置 |
| --- | --- | --- | --- |
| 1 | **`_tail_clip` 命名与行为不符** | 名字说 tail，实际是留头 + 省略号。工具输出的报错栈在尾部（Python traceback 关键帧在末尾），会被优先丢掉。`workspace.middle()` 才是"留头留尾"的正确裁法 | 33–41 |
| 2 | **`DEFAULT_SECTION_FLOORS` 是死常量** | 定义了但从未被读取，真实底线走 `budget // 4`。两处数值对不上：`prefix` 1200↔900、`history` 1500↔1300 | 20–25 |
| 3 | **预算不回收** | case1 里 `memory` 只用了 36 / 1600，闲置 1564 字符，同期 `history` 却被砍了 152 字符。没有"富余额度再分配"机制 | 137–171 |
| 4 | **`section_texts["history"]` 是空占位** | `build()` 里写成 `""`，`history` 的 raw 由 `_render_history_section` 内部从 `session` 现算。字段存在但语义不成对，读代码时容易误以为有个 `_raw_history_from(section_texts)` 的路径 | 111 |
| 5 | **两处重复构造 relevant 文本** | `_render_sections_without_reduction` 里手写了一遍 `relevant_raw` 的拼装，与 `_render_relevant_memory` 的逻辑重复 | 186–191 |
| 6 | **字符数当预算单位** | token 数和字符数不是线性关系（中文 / 代码 / JSON 密度差异很大）。严格说应该按 token 估 | 全局 |

**第 3 条是最有改进价值的一个。** 它的存在说明这套机制是"单向的"：只会在超预算时往下压，不会在有余量时往上抬。在 case1 里，如果把 `memory` 闲置的 1564 字符转给 `history`，`history` 本来不必被砍。

**为什么没改：** 改动会动到所有段的压缩率基线，前面那 12 组长上下文配置的实验结果得全部重跑。属于"知道但暂时不动"。

---

## 11. 一句话总结

> **分层给了裁剪抓手，底线给了崩溃保护，牺牲序给了判断依据，账本给了可解释性。**
> 整套机制的本质是一个**有下界的贪心收缩循环**：每次只动一段、动完立刻重测、精确到"差多少砍多少"；当前请求不在表里，所以长请求挤别人不挤自己；全部触底就认输超标，而不是把某段压成空壳。
