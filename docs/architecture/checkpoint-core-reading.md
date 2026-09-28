# checkpoint.py 核心逻辑精读（提炼版）

> 对象：`pico/checkpoint.py`（176 行）  
> 定位：提炼版。讲骨架、设计理由、实测证据，不逐行走读。  
> 所有数字均为本机实测（Windows + Python 3.13.12），脚手架见文末。

---

## 1. 一句话概括

**176 行、7 个函数、6 个常量。这个模块回答一个问题：这个会话还能不能接着上次继续干，如果不能，是哪里对不上了。**

它干三件事：

1. **写**：在一次任务的关键节点拍一张"进度快照"（`create_checkpoint`）
2. **判**：下次启动时判断这张快照还作不作数，输出五种状态之一（`evaluate_resume_state`）
3. **说**：把快照翻译成一小段文本，塞进 prompt 告诉模型"你上次干到哪了"（`render_checkpoint_text`）

**复杂度分布极不均匀**：`evaluate_resume_state`（48 行）+ `create_checkpoint`（31 行）= 79 行，占全文 **45%**。其余五个函数全是十几行的辅助。

---

## 2. 行区间分布

| 行区间         | 内容                            | 行数     | 承担什么             |
| ----------- | ----------------------------- | ------ | ---------------- |
| 1–6         | docstring + import            | 6      | ——               |
| 8–13        | 5 个状态常量                       | 6      | 定义状态空间           |
| 15–27       | `RUNTIME_IDENTITY_KEYS`（11 项） | 13     | **决定对比什么**       |
| 30–44       | `current_runtime_identity`    | 15     | 采集 12 个环境字段      |
| 47–49       | `checkpoint_state`            | 3      | 取 checkpoints 容器 |
| 52–57       | `current_checkpoint`          | 6      | 取当前快照            |
| **60–107**  | **`evaluate_resume_state`**   | **48** | **五态判定（核心）**     |
| 110–132     | `render_checkpoint_text`      | 23     | 快照 → prompt 文本   |
| 135–142     | `infer_next_step`             | 8      | 猜下一步             |
| **145–175** | **`create_checkpoint`**       | **31** | **拍快照（核心）**      |

按"该看哪几段"排：`evaluate_resume_state` 是唯一的算法，`create_checkpoint` 是唯一的写路径，`RUNTIME_IDENTITY_KEYS` 是一个**配置清单**——它的内容比它的代码重要得多。

---

## 3. 必须分清的概念

### 3.1 checkpoint 不是文件，是 session 里的一个字典

`checkpoint.py` 里没有任何文件操作。快照存在 **`session["checkpoints"]["items"][checkpoint_id]`** 里，由 `session_store` 统一落盘到 `.pico/sessions/<id>.json`。

```
session.json
└── checkpoints
    ├── current_id: "ckpt_a1b2c3d4"        ← 指针，指向"当前"那一张
    └── items
        ├── ckpt_a1b2c3d4: {...}           ← 完整快照
        ├── ckpt_99887766: {...}           ← 更早的快照
        └── ...                            ← 全部保留，永不清理
```

**所以 checkpoint 的体积问题 = session 文件的体积问题。**

### 3.2 五张快照 vs 一个"当前"

`items` 每次 `create_checkpoint` 都往里**追加**，但 `current_id` 只指向最新那张。`current_checkpoint()` 永远只返回 `current_id` 那一张。

**其余的历史快照没有任何函数读取。** 唯一指向它们的是每张快照里的 `parent_checkpoint_id`——而那个字段全项目没有读者（见 §10.6）。

### 3.3 `resume_state` 有两个副本

| 位置                              | 谁写                                                                    | 谁读                                          |
| ------------------------------- | --------------------------------------------------------------------- | ------------------------------------------- |
| `agent.session["resume_state"]` | `evaluate_resume_state`（60–107）                                       | 落盘用；下次构造时当 `previous_resume_state`          |
| `agent.resume_state`            | `Pico.__init__`（`runtime.py:107`）和 `refresh_prefix`（`runtime.py:310`） | `render_checkpoint_text`、`agent_loop`、trace |

**同一个值存两处。** `evaluate_resume_state` 只写 session 里的那份，然后用返回值让 runtime 去更新属性那份。如果某次调用忘了接返回值，两份就会不一致——`render_checkpoint_text` 会渲染出过期的状态。

### 3.4 "stale"（失效）有两个来源

`evaluate_resume_state` 里的 `stale_paths` 是从两处合并来的：

| 来源         | 机制                                                               | 代码位置                  |
| ---------- | ---------------------------------------------------------------- | --------------------- |
| 记忆层摘要失效    | `invalidate_stale_memory()` —— 比对记忆里存的 freshness 和文件当前 freshness | `checkpoint.py:62`    |
| 快照里的关键文件失效 | 逐条比对 `checkpoint["key_files"]` 的 `freshness` 字段                  | `checkpoint.py:71–78` |

两处可能报同一个文件，靠 `path not in stale_paths` 去重（实测确认，见 §7.9）。

### 3.5 identity 有 12 个字段，但只比 11 个

`current_runtime_identity()` 返回 **12** 个键，`RUNTIME_IDENTITY_KEYS` 里只有 **11** 个。多出来的是 `session_id`。

它被采集、被存进快照、被写进 session，**但从不参与比对**（实测确认，见 §7.5）。

**为什么排除它？** 一个会话文件里的所有快照，`session_id` 必然相同，比了永远是"相等"，没有信息量。所以这是**刻意的冗余**——留着是为了让快照自带"我是哪个会话的"这个上下文。

### 3.6 `clip` 与 `current_goal` 的关系

`create_checkpoint` 里，同一段用户输入存了两份：

| 字段             | 处理               | 实测长度（2880 字符的输入） |
| -------------- | ---------------- | ---------------- |
| `current_goal` | **原样存**          | 2880 字符          |
| `summary`      | `clip(msg, 120)` | 160 字符           |

而 `render_checkpoint_text` 输出的是 **`current_goal`**（不裁剪），`summary` 反而不进 prompt。

**结果：一段长请求会以完整长度挤进 prefix。**（实测见 §7.7）

---

## 4. 主流程骨架

### 写路径

```
create_checkpoint(agent, task_state, user_message, trigger)
  ├─ checkpoint_state(agent)                    # 拿到 items 容器
  ├─ current_checkpoint(agent)                  # 拿到"上一张"，用来记父子关系
  ├─ checkpoint_id = "ckpt_" + uuid4()[:8]      # 新 id
  ├─ for path in memory.recent_files:           # 关键文件 = 记忆里的最近文件
  │     freshness[path] = sha256(文件内容)       # ★ 逐个文件做哈希
  │     key_files.append({path, freshness})
  ├─ checkpoint = { ...13 个字段... }            # 组装
  ├─ state["items"][checkpoint_id] = checkpoint # ★ 追加，不覆盖
  ├─ state["current_id"] = checkpoint_id        # 指针前移
  ├─ task_state.checkpoint_id = checkpoint_id
  ├─ session["runtime_identity"] = ...          # 顺手更新 session 级身份
  └─ session_store.save(session)                # ★ 全量落盘
```

**两个 ★ 是性能与体积的关键**：每个检查点都要对每个最近文件做 sha256，然后整份 session 重新写盘。

### 读路径

```
evaluate_resume_state(agent)
  ├─ previous_resume_state = session["resume_state"]     # 上一轮的计数，用于累加
  ├─ invalidated = agent.invalidate_stale_memory()       # ★ 有副作用：清记忆摘要
  ├─ checkpoint = current_checkpoint(agent)
  ├─ status = "no-checkpoint"                            # 默认值
  ├─ if checkpoint:
  │     ① schema 不对  → "schema-mismatch"   并结束
  │     ② 有文件失效   → "partial-stale"
  │     ③ 有环境差异   → "workspace-mismatch"
  │     ④ 都没有       → "full-valid"
  ├─ session["resume_state"] = {...}
  └─ session["runtime_identity"] = current_runtime_identity(agent)   # ★ 第二次采集
```

**顺序是刻意的**：越靠前的判定越"根本"。schema 不对说明**读字段这件事本身都不可信**，后面的检查全无意义——实测确认它确实会短路后面两个检查（见 §7.3）。

---

## 5. 核心函数逐个拆


### 5.1 `current_runtime_identity`（30–44）—— 环境的指纹

```python
def current_runtime_identity(agent):
    return {
        "session_id": agent.session.get("id", ""),
        "cwd": str(agent.root),
        "model": str(getattr(agent.model_client, "model", "")),
        "model_client": agent.model_client.__class__.__name__,
        "approval_policy": agent.approval_policy,
        "read_only": bool(agent.read_only),
        "max_steps": int(agent.max_steps),
        "max_new_tokens": int(agent.max_new_tokens),
        "feature_flags": dict(agent.feature_flags),
        "shell_env_allowlist": list(agent.shell_env_allowlist),
        "workspace_fingerprint": getattr(getattr(agent, "prefix_state", None), "workspace_fingerprint", agent.workspace.fingerprint()),
        "tool_signature": agent.tool_signature(),
    }
```

**备注一：这 12 个字段是"这次运行的配置"。**

改动其中任何一个，都会让旧快照里的判断失去前提。举几个真实的例子：

| 字段变了                    | 为什么旧快照不可信                          |
| ----------------------- | ---------------------------------- |
| `model`                 | 换了模型，之前那套上下文假设可能不适用                |
| `approval_policy`       | 从 `ask` 改成 `never`，之前"等用户确认"的阻塞点没了 |
| `feature_flags`         | 关掉 `memory` 后，快照里依赖记忆的推理就断了        |
| `workspace_fingerprint` | 仓库状态变了（分支、status、最近提交）             |
| `tool_signature`        | 工具集变了，之前用某个工具做的计划可能做不了了            |

**备注二：最后一个字段有个隐蔽的求值顺序问题。**

```python
getattr(getattr(agent, "prefix_state", None), "workspace_fingerprint", agent.workspace.fingerprint())
```

三层嵌套 `getattr`，字面意思是"先看 `prefix_state.workspace_fingerprint`，没有就现算一个"。

**但 Python 会先把所有参数求值再传进去**，所以 `agent.workspace.fingerprint()` **每次都会执行**——即使 `prefix_state` 里有现成的缓存值。

实测确认：

```
调用前 fingerprint 次数: 0 -> 调用后: 1
prefix_state 里明明有现成的值: 'cached-fp'
identity 里最终用的值      : 'cached-fp'      ← 缓存值被采用了
```

缓存值被采用了，但那次计算已经白做了。

**成本多少？** `WorkspaceContext.fingerprint()` 的实现是 `json.dumps(payload, sort_keys=True)` + `sha256`，payload 包含 `status`（git status 全文）、`recent_commits`、`project_docs`。按 30KB 的 git status 估算：

```
一次真实 fingerprint 的成本: 0.065 ms
```

**0.065 ms 本身可忽略。** 但要看它被调用几次——实测：

```
evaluate_resume_state 一次 -> fingerprint 调用 1 次
create_checkpoint   一次 -> 累计 2 次
再 evaluate 一次         -> 累计 4 次（增加 2）
```

**注意第三次是 +2**：因为 `evaluate_resume_state` 在 `if checkpoint:` 分支里调一次（第 80 行），在函数末尾又调一次（第 106 行）。**有 checkpoint 时每次评估采集两遍身份。**

所以一个"拍快照 → 评估"的周期里，`fingerprint` 跑了 **3 次**，全是 `json.dumps + sha256`，且每次都丢弃了本可以复用的缓存值。

**取舍**：`getattr` 的默认值写法让"降级路径"变得简洁，代价是降级永远会被执行。要避免就得写成显式的 `if`。这里真正的教训不是"0.065ms 很重要"，而&#x662F;**"看起来像惰性降级的代码，在 Python 里是急切求值"**。


### 5.2 `checkpoint_state` / `current_checkpoint`（47–57）

```python
def checkpoint_state(agent):
    agent._ensure_session_shape()          # ★ 调用前先补全结构
    return agent.session["checkpoints"]

def current_checkpoint(agent):
    state = checkpoint_state(agent)
    checkpoint_id = str(state.get("current_id", "")).strip()
    if not checkpoint_id:
        return None
    return state.get("items", {}).get(checkpoint_id)
```

**备注：`checkpoint_state` 每次都调 `_ensure_session_shape()`。**

这是个"防御性初始化"——因为 `load` 出来的老 session 可能根本没有 `checkpoints` 这个键。`_ensure_session_shape` 定义在 `runtime.py:132`，用一串 `setdefault` 补齐结构。

**这解释了一个跨模块分工**：`checkpoint.py` 不做版本迁移，迁移责任在 runtime。`checkpoint_state` 只是一行保险，保证自己不会因为键缺失而崩。

**代价**：`_ensure_session_shape` 会因 `current_checkpoint` 被间接调用而反复执行。实测在 200 次 `create_checkpoint` 里，它被调了 400 次以上（`create_checkpoint` 各调一次 + `evaluate_resume_state` 调一次）。因为它全是 `setdefault`（幂等且廉价），没有实测到可测量的开销，但这是个"防御性代码放在热路径上"的形态。

`current_checkpoint` 里那句 `.strip()` 值得注意：`current_id` 理论上永远是 id 或空串，但 `str(...).strip()` 保证即使加载回来的是 `" "` 也当作"没有"。


### 5.3 `evaluate_resume_state`（60–107）—— 核心

拆成四段看。

**第一段：采集（61–66）**

```python
previous_resume_state = dict(agent.session.get("resume_state", {}) or {})
invalidated = agent.invalidate_stale_memory()
checkpoint = current_checkpoint(agent)
status = CHECKPOINT_NONE_STATUS
stale_paths = list(invalidated)
mismatch_fields = []
```

**这里有个设计上的别扭处**：函数名叫"评估状态"（`evaluate`），但它**调用了有副作用的 `invalidate_stale_memory()`**——后者会把失效的文件摘要从记忆里**删掉**并同步回 session。

**为什么必须在这删？** 因为"哪些文件变了"这个信息只能通过"比对摘要和现实"得到。如果只读不删，下次评估还会把同一个文件报成失效，永远不会收敛。

**代价**：`evaluate_resume_state` 不再是个查询函数。它在 `Pico.__init__` 里被调用，意味着**光是把 agent 构造出来，记忆层就被改动了**。这是一处"纯读名字做了写事情"的命名债。

**第二段：schema 闸门（67–69）**

```python
if checkpoint:
    if checkpoint.get("schema_version") != CHECKPOINT_SCHEMA_VERSION:
        status = CHECKPOINT_SCHEMA_MISMATCH_STATUS
    else:
        ... 后面的检查全在这个 else 里 ...
```

**这是整个函数最重要的一行设计。** `if/else` 而不是"出错继续查"——schema 版本不对，意味着**快照里字段的含义可能已经变了**，此时再去读 `key_files`、`runtime_identity` 就是拿旧格式的假设去解释新格式的数据。

实测短路效果（文件改了 + 环境改了 + schema 不对）：

```
状态 = schema-mismatch
此时 stale_paths = []   mismatch 字段 = []
```

**两个检查都没跑。** 这是对的：与其报一堆可能误读的差异，不如只报"格式对不上"这一个准确的结论。

**第三段：两个检查（71–90）**

```python
for item in checkpoint.get("key_files", []):
    path = str(item.get("path", "")).strip()
    if not path:
        continue                                          # 空路径跳过
    expected = item.get("freshness")
    current = memorylib.file_freshness(path, agent.root)
    if expected != current and path not in stale_paths:
        stale_paths.append(path)                          # ★ 去重

saved_identity = dict(checkpoint.get("runtime_identity", {}) or agent.session.get("runtime_identity", {}) or {})
current_identity = current_runtime_identity(agent)
for key in RUNTIME_IDENTITY_KEYS:
    if key not in saved_identity:
        continue                                          # ★ 缺席的字段不报差异
    if saved_identity.get(key) != current_identity.get(key):
        mismatch_fields.append(key)
mismatch_fields.sort()                                    # ★ 排序保证可比
```

三个值得注意的细节：

**① `expected != current` 里，`expected` 可能是 `None`。** `file_freshness` 在文件不存在或不是文件时返回 `None`；快照里存的就是这个 `None`。所以"文件当时不存在，现在存在了"也会被判为失效——这是正确的（事实变了）。

**② `if key not in saved_identity: continue` 是向前兼容。** 老快照里没有的字段（比如新版本才加的 `tool_signature`）不会被报成差异。**这个设计让"新增身份字段"不会让所有旧快照集体失效**——如果去掉这一行，加一个新字段会导致所有历史会话都变成 `workspace-mismatch`。

代价：如果某个字段是**必须**匹配的，它缺席时会静默通过。这个"宽进"的策略在安全场景下要小心，在这里合理。

**③ `mismatch_fields.sort()`** 保证输出顺序稳定。这看起来无关紧要，但它让"同一组差异"产生**逐字节相同的 resume_state**——这对快照比对、评测断言、diff 都是必要的。同样的考虑在 `current_runtime_identity` 里没有做（那个 dict 的键序是稳定的，因为字面量顺序固定）。

**第四段：优先级（87–92）**

```python
if stale_paths:
    status = CHECKPOINT_PARTIAL_STALE_STATUS
elif mismatch_fields:
    status = CHECKPOINT_WORKSPACE_MISMATCH_STATUS
else:
    status = CHECKPOINT_FULL_VALID_STATUS
```

**`stale` 优先于 `mismatch`。** 实测两种差异同时存在时：

```
文件变了 + 环境也变了 -> partial-stale | stale: ['a.py'] | mismatch: ['max_steps']
```

两者都被记录了（`stale_paths` 和 `mismatch_fields` 都不为空），但 `status` 取 `partial-stale`。

**为什么 stale 优先？** 因为两者对模型的**可行动性**不同：

- `partial-stale` 是"**这几个文件你读过的内容已经不对了，重新读一遍**"——这是一条具体的行动指令
- `workspace-mismatch` 是"**运行环境跟你上次不一样了，自己小心**"——这是一个模糊的警告

前者更可执行，所以状态里体现它。而且两者**信息都没丢**——`stale_paths` 和 `mismatch_fields` 是并列的两个字段，`status` 只是个"主要矛盾"的标签。

**第五段：计数（94–104）**

```python
"stale_summary_invalidations": max(
    len(invalidated),
    int(previous_resume_state.get("stale_summary_invalidations", 0))
    if status == CHECKPOINT_PARTIAL_STALE_STATUS
    else 0,
),
```

**这段的逻辑意图是"在 partial-stale 状态下让计数只增不减"**，避免"这一轮没有新失效"就把历史累计清零。

**但它不是单调的。** 实测：

```
第1次  无 checkpoint          status=no-checkpoint    invalidated=1  计数=1
第2次  有失效、有 checkpoint   status=partial-stale    invalidated=1  计数=1
第3次  失效没了                status=full-valid       invalidated=0  计数=0   ← 从 1 掉回 0
```

**只有 `partial-stale` 时才继承历史值**，其余四种状态每次都归零。所以这个"累计计数"在状态切换时会跳变。如果评测指标依赖它做"累计失效次数"，跨状态求和会偏小。


### 5.4 `render_checkpoint_text`（110–132）—— 快照 → prompt

```python
lines = [
    "Task checkpoint:",
    f"- Resume status: {...}",
    f"- Current goal: {checkpoint.get('current_goal', '-') or '-'}",
    f"- Current blocker: {...}",
    f"- Next step: {...}",
]
key_files = [...]
lines.append(f"- Key files: {', '.join(key_files) or '-'}")
if checkpoint.get("completed"):  lines.append("- Completed: " + ...)
if checkpoint.get("excluded"):   lines.append("- Excluded: " + ...)      # ★ 永远不执行
if agent.resume_state.get("stale_paths"): lines.append("- Stale paths: " + ...)
summary = str(checkpoint.get("summary", "")).strip()
if summary: lines.append(f"- Summary: {summary}")
```

**备注一：输出是"八行有七行可选"的结构。**

必出的四行：`Resume status` / `Current goal` / `Current blocker` / `Next step` / `Key files`。条件出的是 `Completed` / `Excluded` / `Stale paths` / `Summary`。

用 `- ` 前缀 + 换行的格式，是为了让模型能逐行读；`or '-'` 保证空值也有一行占位，**结构稳定比内容完整更重要**——模型能学会"这一行是空的"，但如果某些行时有时无，行与行的对应关系就断了。

**备注二：`Excluded` 那行永远不执行。**

`create_checkpoint` 里 `"excluded": []` 是硬编码空列表，全项目没有任何地方往里面写东西。所以 `if checkpoint.get("excluded")` **永远为假**。

这是"预留了字段和渲染逻辑，但功能没做"的典型形态——**删了不损失任何功能**。

**备注三：这里的 `stale_paths` 是从 `agent.resume_state` 读的，不是从 session 读的。**

呼应 §3.3 的双副本问题——**如果 runtime 忘了把返回值赋给 `agent.resume_state`，这一行就会渲染出过期的失效文件列表**（甚至可能是空的）。

**备注四：mismatch 信息不进 prompt。**

`runtime_identity_mismatch_fields` 被记录进 `resume_state`、被写进 trace（`runtime.py:334`），但 **`render_checkpoint_text` 完全不输出它**。

**所以模型只能看到"运行环境变了"这个状态标签，不知道具体哪个字段变了。** 从"让模型做决策"的角度看，这是个信息缺口——知道"是 `approval_policy` 变了"和知道"是 `model` 变了"，应对方式完全不同（前者可能意味着某些操作不再需要确认，后者意味着上下文假设可能变）。

**备注五：`current_goal` 不裁剪，直接进 prefix。**

实测：2880 字符的用户输入 → `current_goal` 2880 字符 → 渲染出的 `Current goal` 那一行 **2896 字符**，整个 checkpoint 文本 **3208 字符**。

而这段文本最终会被拼进 **prefix**（`context_manager.py:114–118`），prefix 的预算是 3600、底线 900。**一段长请求能一口气吃掉 prefix 的绝大部分额度。**

### 5.5 `infer_next_step`（135–142）—— 四分支

```python
if task_state.status == "completed":       return "No next step recorded."
if task_state.stop_reason == "step_limit_reached": return "Resume from the latest checkpoint and continue the task."
if task_state.last_tool:                   return f"Decide the next action after {task_state.last_tool}."
return "Continue the task from the latest checkpoint."
```

**备注：顺序决定优先级。**

`status == "completed"` 排第一——**任务已经完成时，"下一步"这个问题本身就不成立**，所以先短路。

`stop_reason == "step_limit_reached"` 排第二——这是**唯一的"是因为预算耗尽才停下"的情况**，需要明确告诉模型"你不是干完了，是被打断了"。

剩下两个分支是纯粹的信息量递减：知道上一个工具是什么（能推断出流程到哪一步）> 什么都不知道（只能泛泛地"继续"）。

**取舍**：这四个分支全是**模板字符串**，没有真正的推理。因为"猜下一步"这件事，让模型拿着上下文自己判断，比在这里写死规则更靠谱。这个函数的价值不是"给出答案"，而是**在模型还没看到历史的时候，给它一个不空洞的起点**。


### 5.6 `create_checkpoint`（145–175）—— 拍快照

```python
state = checkpoint_state(agent)
current = current_checkpoint(agent)
checkpoint_id = "ckpt_" + uuid.uuid4().hex[:8]
key_files = []
freshness = {}
for path in agent.memory.to_dict()["working"]["recent_files"]:
    file_freshness = memorylib.file_freshness(path, agent.root)   # ★ sha256
    freshness[path] = file_freshness
    key_files.append({"path": path, "freshness": file_freshness})
checkpoint = {
    "checkpoint_id": checkpoint_id,
    "parent_checkpoint_id": current.get("checkpoint_id", "") if current else "",
    "schema_version": CHECKPOINT_SCHEMA_VERSION,
    "created_at": now(),
    "current_goal": str(user_message),                            # ★ 不裁剪
    "completed": [task_state.final_answer] if task_state.final_answer else [],
    "excluded": [],                                               # ★ 永远是空的
    "current_blocker": "" if str(task_state.stop_reason or "") in ("", "final_answer_returned") else str(task_state.stop_reason),
    "next_step": infer_next_step(task_state),
    "key_files": key_files,
    "freshness": freshness,                                       # ★ 与 key_files 重复
    "summary": f"{trigger}: {clip(str(user_message), 120)}",
    "runtime_identity": current_runtime_identity(agent),          # ★ 又一次采集
}
state["items"][checkpoint_id] = checkpoint                        # ★ 只增不减
state["current_id"] = checkpoint_id
task_state.checkpoint_id = checkpoint_id
agent.session["runtime_identity"] = checkpoint["runtime_identity"]
agent.session_path = agent.session_store.save(agent.session)      # ★ 全量落盘
```

**备注一：关键文件从记忆层的"最近文件"来，不是从对话里现算。**

```python
for path in agent.memory.to_dict()["working"]["recent_files"]
```

**这是关键的跨模块复用。** 记忆层已经在维护"这个会话碰过哪些文件"（有上限、会去重），checkpoint 直接拿来用，**不需要自己再统计一遍**。

代价是：如果记忆层的 `recent_files` 被清空或上限太小，checkpoint 的 `key_files` 就会是空的——**而 `key_files` 为空意味着"文件失效"这个检查永远不会触发**，快照永远显示 `full-valid`。

**备注二：`freshness` 字段和 `key_files` 存的是同一份数据。**

两个人都存了 `{path: hash}`，但形式不同：

- `key_files`：`[{"path": p, "freshness": h}, ...]`（列表，保序）
- `freshness`：`{p: h}`（字典，可查）

**但顶层 `freshness` 全项目没有任何读者**（静态检查确认，见 §10.5）。`evaluate_resume_state` 读的是 `item.get("freshness")`，也就是从 `key_files` 里取。

所以这是**一份数据的冗余副本，且副本是死的**。它让每个 checkpoint 的体积翻倍。

**备注三：`parent_checkpoint_id` 连成链，但没人走这条链。**

每张快照记录上一张的 id，理论上可以回溯整条时间线。但全项目只有**写**没有**读**（静态检查确认）。

**这是个"为将来预留"的字段**——如果要做"回滚到某个历史快照"，这条链就是现成的索引。但现在它只贡献体积。

**备注四：`completed` 只在任务真的给了最终答案时才有内容。**

`[task_state.final_answer] if task_state.final_answer else []`

注意它是个**列表**但最多一个元素。设计上像是"完成事项清单"，实际只装最终答案。

**备注五：`current_blocker` 用了一个特殊的排除条件。**

```python
"" if str(task_state.stop_reason or "") in ("", "final_answer_returned")
   else str(task_state.stop_reason)
```

`final_answer_returned` 是"正常结束"，不算卡点。其余任何 `stop_reason`（工具错误、步数耗尽、工作区变了……）都会被当成卡点报给模型。

**这个判断是白名单式的**：只有明确知道的两个值不算卡点，其他一律算。**这样新加的 `stop_reason` 会自动被当作卡点**——保守但安全（宁可多报卡点，不可漏报）。

**备注六：最后一行把整份 session 写盘。**

这一行通过 `session_store.save()`，把**整个** session（历史 + 记忆 + 全部 checkpoint）重新序列化写一遍。

结合 §10.4 的实测：**每个 checkpoint 体积约 1.2 KB，且永不清理**。所以第 N 次保存要写大约 `1.2N KB` 的数据，累计写入量是 `O(N²)`。

---

## 6. 设计理由

### 6.1 为什么用 sha256 存 freshness，而不是存文件内容的副本？

**好处**：一个哈希 64 字节，文件内容可能几十 KB。而且哈希是**单向可比**的——只要哈希一样，内容就一样，不需要保留原文。

**代价**：每次拍快照都要把每个最近文件**完整读一遍**做哈希。`file_freshness` 是 `hashlib.sha256(resolved.read_bytes()).hexdigest()`——全量读。如果最近文件里有几个大文件，拍快照就变成了一次批量磁盘读。

对一个本地小工具，这个成本可以接受（文件通常几 KB）。但在大仓库里这会是个问题。

### 6.2 为什么状态是 5 个而不是"能不能恢复"（布尔）？

**好处**：不同状态对应**不同的恢复动作**。

| 状态                   | 恢复动作           |
| -------------------- | -------------- |
| `no-checkpoint`      | 全新会话，正常开始      |
| `full-valid`         | 直接用，什么都不用做     |
| `partial-stale`      | **重读那几个文件**再继续 |
| `workspace-mismatch` | 继续，但对环境假设保持怀疑  |
| `schema-mismatch`    | 快照不可用，当新会话处理   |

**代价**：状态机变复杂，`stale_summary_invalidations` 那种需要跨状态维护的计数器就会出现（并且实测证明它没做对）。

**取舍判断**：5 个状态是划算的。因为 `full-valid` 和 `partial-stale` 的差别是"直接跑"和"先重读文件"，这个差别必须让模型知道——否则它会拿着过期内容继续推理。

### 6.3 为什么 schema 检查放最前且短路？

**好处**：避免**用错误的假设解读数据**。schema 版本是"后面那些字段该怎么读"的契约。契约不对，读出来的东西没有意义。

**代价**：schema 不匹配时，用户拿不到"到底哪里变了"的细节——只能看到一句"格式对不上"，然后整个快照被丢弃。**老版本产生的所有会话都会集体失效。**

**取舍判断**：合理。schema 变更本来就是**一次性的、全量的**，这时候"精确诊断每个快照"没有价值，不如快速失败。

### 6.4 为什么 `if key not in saved_identity: continue`？

**好处**：**新增身份字段不会让所有旧快照集体失效**。

假设某天给 `RUNTIME_IDENTITY_KEYS` 加了 `"python_version"`。所有历史快照的 `runtime_identity` 里都没有这个键。如果不加这个 `continue`，所有快照立刻变成 `workspace-mismatch`——**用户升级 pico 之后就再也接不上任何旧会话了。**

**代价**：字段"缺席"和字段"匹配"被当成了一回事。如果某个字段从"不记录"变成"必须匹配"，历史快照会静默通过检查。

**取舍判断**：在这个场景下（本地开发工具、快照的价值是"省去重读文件"而不是"安全保证"），宽进是对的。

### 6.5 为什么 stale 优先于 mismatch？

见 §5.3。核心是**可行动性**：stale 给出的是"重读这几个文件"这个具体动作，mismatch 只能给出"小心点"这个模糊警告。

而且两者信息都没丢——`status` 只是"主要矛盾"的标签，`stale_paths` 和 `mismatch_fields` 始终并存。

### 6.6 为什么 checkpoint 存在 session 里，不做独立文件？

**好处**：一次落盘就写完所有状态，不需要维护"session 和 checkpoint 的一致性"这一整类问题。

**代价**：**checkpoint 的膨胀直接变成 session 文件的膨胀**。实测 200 个 checkpoint → session 文件 230 KB，累计写入 22.6 MB（见 §7.6）。

而 checkpoint 本身是"只增不减"的，所以这个膨胀是**单调的、不可逆的**。

**取舍判断**：在短会话（几十个工具调用）下没问题。长会话会明显变慢——**而且慢的不是保存那个动作，是每次保存都要写越来越大的一坨。**

---

## 7. 实测验证

用桩对象 + 真实临时文件跑，不拉模型。脚手架见文末。

### 7.1 五态穷举

| #  | 输入                                      | 判定结果                                                     |
| -- | --------------------------------------- | -------------------------------------------------------- |
| ①  | 全新会话，无快照                                | `no-checkpoint`                                          |
| ②  | 快照完好，文件没动，环境没变                          | `full-valid`                                             |
| ②' | 同一份快照，`a.py` 被改过                        | `partial-stale`，`stale_paths = ['a.py']`                 |
| ④  | 只有 `approval_policy` 从 `ask` 改成 `never` | `workspace-mismatch`，`mismatch 字段 = ['approval_policy']` |
| ⑤  | `schema_version` 改成 `"phase0-v0"`       | `schema-mismatch`                                        |

### 7.2 优先级：stale 赢

```
文件变了 + 环境也变了
→ status = partial-stale
   stale:    ['a.py']
   mismatch: ['max_steps']
```

两者都被记录，`status` 取 stale。

### 7.3 schema-mismatch 短路

```
文件改了 + 环境改了 + schema 不对
→ status = schema-mismatch
   stale_paths = []        ← 检查没跑
   mismatch 字段 = []      ← 检查没跑
```

### 7.4 `getattr` 默认值被急切求值

```
调用前 fingerprint 次数: 0 -> 调用后: 1
prefix_state 里有缓存值 'cached-fp'，identity 里也用了 'cached-fp'
→ 但那一次 fingerprint() 已经执行了
```

单次成本（30 KB git status 负载）：**0.065 ms**

调用次数实测：

```
evaluate_resume_state 一次 → fingerprint 1 次
create_checkpoint   一次 → 累计 2 次
再 evaluate 一次         → 累计 4 次（+2，有快照时采集两遍身份）
```

### 7.5 `session_id` 采集但不比较

```
identity 有 session_id 字段: True
RUNTIME_IDENTITY_KEYS 含 session_id: False
对比字段数 11 / identity 实际字段数 12
把 session id 改成"完全不同的会话" → mismatch 字段 = []   ← 不报差异
```

### 7.6 `items` 只增不减 —— session 文件线性膨胀

```
checkpoint 数 : 200
    第   1 个 checkpoint 时 session 文件 =     1772 字节
    第  10 个 checkpoint 时 session 文件 =    12095 字节
    第  50 个 checkpoint 时 session 文件 =    58055 字节
    第 100 个 checkpoint 时 session 文件 =   115505 字节
    第 200 个 checkpoint 时 session 文件 =   230605 字节
累计写入总量  = 22681.7 KB （单个文件大小的累加）
当前 items 数量 = 200，全部保留，无任何清理
```

**每个 checkpoint 约 1.15 KB，文件随数量严格线性增长，累计写入是 O(N²)。**

结合 `tool_executed` 这个高频触发点：**每一次工具调用都要把整份 session 重写一遍**。

### 7.7 `current_goal` 不裁剪

```
用户输入长度            : 2880
checkpoint.current_goal  : 2880  (原样)
checkpoint.summary       : 160   (clip 到 120)
单个 checkpoint 体积      : 3771 字节
渲染进 prefix 的文本长度  : 3208
其中 "Current goal" 那一行 : 2896
```

3208 字符的 checkpoint 文本要挤进预算 3600、底线 900 的 prefix。

### 7.8 `stale_summary_invalidations` 非单调

```
第1次  无 checkpoint        status=no-checkpoint    invalidated=1  计数=1
第2次  有失效、有 checkpoint status=partial-stale    invalidated=1  计数=1
第3次  失效没了              status=full-valid       invalidated=0  计数=0   ← 掉回 0
```

### 7.9 `stale_paths` 去重正确

```
记忆层报 a.py 失效 + 快照的 key_files 也报 a.py 失效
→ 最终 stale_paths = ['a.py']，长度 1
```

去重靠 `if expected != current and path not in stale_paths`。

### 7.10 死字段（静态检查）

| 字段                     | 写入点                 | 读者                              | 结论          |
| ---------------------- | ------------------- | ------------------------------- | ----------- |
| 顶层 `freshness`         | `checkpoint.py:166` | **无**                           | 一份死副本，让体积翻倍 |
| `excluded`             | 永远写 `[]`            | 只在 `checkpoint.py:125` 的 `if` 里 | **该分支永不执行** |
| `parent_checkpoint_id` | `checkpoint.py:157` | **无**                           | 父子链建了没人走    |

---

## 8. 不变量

| #   | 不变量                                    | 验证方式                                     | 实测                                      |
| --- | -------------------------------------- | ---------------------------------------- | --------------------------------------- |
| I1  | 判定结果必是五个状态之一                           | 穷举                                       | ✓ 五态全部复现                                |
| I2  | `schema-mismatch` 时另外两项检查不执行           | 构造三重失败                                   | ✓ `stale_paths=[]` `mismatch_fields=[]` |
| I3  | 同时 stale 与 mismatch 时 `status` 取 stale | 构造双重差异                                   | ✓                                       |
| I4  | `stale_paths` 内无重复元素                   | 两处同报同一文件                                 | ✓ 长度 1                                  |
| I5  | `mismatch_fields` 已排序                  | 检查输出                                     | ✓（`mismatch_fields.sort()`）             |
| I6  | `session_id` 永不出现在 `mismatch_fields`   | 改 session id                             | ✓                                       |
| I7  | 老快照缺字段不会造成 mismatch                    | `if key not in saved_identity: continue` | ✓ 逻辑保证                                  |
| I8  | `excluded` 恒为空列表                       | 全项目 grep                                 | ✓ 无写入点                                  |
| I9  | `items` 数量单调递增                         | 200 次创建                                  | ✓ 200 个全保留                              |
| I10 | `stale_summary_invalidations` 单调不减     | 状态切换                                     | ✗ **被推翻**，从 1 掉回 0                      |
| I11 | `current_goal` 长度 == 用户输入长度            | 2880 → 2880                              | ✓                                       |

**I10 是唯一被推翻的候选不变量。**

---

## 9. 精读顺序建议

1. **`RUNTIME_IDENTITY_KEYS`（15–27）** —— 先看这份清单。它是"什么算环境变了"的定义，比任何代码都重要。注意数一下：**11 项**。
2. **`current_runtime_identity`（30–44）** —— 对照清单看它采集了 12 个字段，找出多出来的那个（`session_id`），想想为什么不比它。
3. **`evaluate_resume_state`（60–107）** —— 唯一的算法，分四段读：采集 / schema 闸门 / 两个检查 / 优先级。重点看 `else:` 的括号范围——**schema 检查把后面全部包住了**。
4. **`create_checkpoint`（145–175）** —— 逐字段读那个 13 键的字典，对每个字段问"谁读它"。
5. **`render_checkpoint_text`（110–132）** —— 看哪些行是必出的、哪些是可选的，以及 `Excluded` 那行为什么永远不执行。
6. **回到调用方** —— `context_manager.py:114–118`（怎么进 prefix）、`agent_loop.py` 的 7 个 trigger、`runtime.py:132`（结构补全）。

跳过的：`checkpoint_state`、`current_checkpoint`、`infer_next_step`——它们各自只有 3–8 行，读到第 3 步自然会懂。

---

## 10. 已知粗糙处

按约定**只报告，不修改**。

### 10.1 `getattr` 默认值急切求值（性能）

```python
getattr(getattr(agent, "prefix_state", None), "workspace_fingerprint", agent.workspace.fingerprint())
```

`agent.workspace.fingerprint()` 每次必执行，结果被丢弃。单次 0.065 ms（30 KB 负载），一个周期内被触发 3 次。

改法：显式 `prefix_state = getattr(agent, "prefix_state", None)` 再 `if` 判断。

### 10.2 `current_runtime_identity` 在 `evaluate_resume_state` 里被调两次

第 80 行（做比对）和第 106 行（写回 session）。后者完全可以直接用第 80 行算出的 `current_identity`。

有快照时每次评估多跑一次 `json.dumps + sha256`（fingerprint）+ 一次 `tool_signature()`。

### 10.3 `current_goal` 不裁剪

2880 字符的输入原样进 checkpoint，再原样进 prefix（实测渲染 3208 字符）。而同一个字段的 `summary` 版本反而做了 `clip(120)` 且不进 prompt。

### 10.4 `items` 只增不减 → `O(N²)` 写入

200 个 checkpoint → 230 KB 文件，累计写入 22.6 MB。叠加 `tool_executed` 这个高频 trigger，长会话代价明显。

改法：保留近 N 张（比如 10 张），或用环形缓冲。但要注意 `parent_checkpoint_id` 的链会断。

### 10.5 顶层 `freshness` 是死副本

`checkpoint.py:166` 写入，全项目无读者。与 `key_files` 存同一份数据，让每个 checkpoint 体积接近翻倍。

### 10.6 `parent_checkpoint_id` 无读者

父子链完整维护，但没有任何函数遍历它。属于"为将来预留"的字段，当前只贡献体积。

### 10.7 `excluded` 永远为空

硬编码 `[]`，无写入点。`render_checkpoint_text` 里对应的分支永不执行。

### 10.8 `stale_summary_invalidations` 不是单调计数器

只有 `partial-stale` 时才继承历史值，其余状态归零。实测从 1 掉回 0。

如果它被当作"累计失效次数"用于评测，跨状态聚合会偏小。

### 10.9 `evaluate_resume_state` 名字是"评估"，实际有写副作用

它会调用 `invalidate_stale_memory()`（删除失效的记忆摘要）并修改 `session["resume_state"]`、`session["runtime_identity"]`。

而这个函数在 `Pico.__init__` 里被调用——**构造对象就会改动记忆层**。命名与行为不符，容易在读代码时误判为纯查询。

### 10.10 `resume_state` 双副本

`agent.session["resume_state"]`（落盘）和 `agent.resume_state`（内存属性）存同一个值。`evaluate_resume_state` 只写前者，靠 runtime 接返回值更新后者。

一旦某次调用忘了接返回值，`render_checkpoint_text` 会渲染过期状态。

### 10.11 mismatch 详情不进 prompt

`runtime_identity_mismatch_fields` 只进 `resume_state` 和 trace，`render_checkpoint_text` 不输出。模型只知道"环境变了"，不知道变了哪个字段。

### 10.12 `checkpoint_state` 每次调 `_ensure_session_shape`

防御性初始化放在被反复调用的路径上。因为 `setdefault` 幂等且廉价，实测无可测量开销，但形态上是"每次进门先检查一遍门框"。

---


## 11. 面试问答

**Q：检查点是干什么的？**

> 就是给一次任务拍进度快照。存在三个时候：任务正常结束、工具报错、或者上下文被压缩了。每次拍的时候记一下"当前目标是什么、卡在哪、下一步该干嘛、碰过哪些文件"，连同当时的运行环境一起存下来。

**下次启动的时候**，拿这份快照跟现在的实际情况比一比，看还能不能接着用。

**Q：怎么判断快照还能不能用？**

> 四道关，按顺序过：

> 1. 有没有快照。没有就是全新会话。
> 2. 快照的 schema 版本对不对得上。对不上就直接判不可用，**后面两关都不查了**。
> 3. 快照里记的那几个关键文件，内容还跟当时一样吗？用哈希比。有一个变了就是"部分失效"。
> 4. 上次的运行环境和现在一样吗？模型、审批策略、功能开关、工作区状态这些，有一样不同就报"环境不一致"。

> 全过了才是"完好"。

**Q：为什么 schema 不对就不查后面了？**

> 因为 schema 决定的&#x662F;**"后面那些字段该怎么读"**。格式契约都不对了，再去比对内容就是拿旧格式的假设去解释新格式的数据，比出来的差异可能是假的。

> 与其报一堆可能是误读的差异，不如只报一句准确的"格式对不上"。**老版本产生的快照会集体失效**，这是代价，但 schema 变更本来就是一次性的、全量的，这时候不需要精确诊断。

**Q：文件失效和环境不一致同时发生，报哪个？**

> 报文件失效，但两个信息都记下来。

> 因为这两件事的**可行动性不一样**：文件失效能翻译成一句具体的指令——"你读过的这几个文件内容变了，重读一遍"；环境不一致只能给个模糊的警告——"环境跟你上次不一样了，小心点"。**状态标签给主要矛盾，细节字段两个都留着。**

**Q：比对的时候，快照里没有的字段怎么办？**

> 跳过，不当差异。

> 这是个刻意的选择。假设我加了新的一个身份字段，历史快照里都没有。如果不跳过，**所有旧会话立刻全部变成"环境不一致"，用户升级完 pico 就再也接不上任何以前的会话了**。

> 代价是"字段缺失"和"字段匹配"被当成一回事。但在这个场景里，快照的价值是省去重读文件，不是安全保证，**宽进是划算的**。

**Q：这块有什么你觉得做得不好的？**

> 三个。



> **第一是快照只增不减。** 每次拍快照都往一个字典里追加，`current_id` 指向最新的，但老的那些全留着。实测拍两百次之后，会话文件涨到 230 KB，累计写入量二十多兆——因为每次保存都是整份重写，所以总写入量是**数量的平方**。而拍快照的触发点里有一个是"每执行一次工具"，这个频率很高。



> **第二是判断逻辑里有几处白算。** 有个字段我写成了"优先用缓存，缓存没有就现算"，但 Python 会**先把现算那个表达式求值再传进去**，所以缓存命中时那次计算照样跑了。而且我算环境指纹的那个函数，在一次评估里被调了两遍——第一遍用来比对，第二遍用来写回。

> **第三是名字骗人。** 评估状态这个函数听起来是只读的，实际上它会删掉过期的记忆摘要、还会改会话里的字段。而且它在构造对象的时候就被调用，**等于构造一个对象就动了记忆层**。读代码的人很容易假设它是纯查询。

**Q：为什么用哈希比文件变没变，不直接存内容？**

> 因为哈希是 64 字节，文件可能是几十 KB。而且**单向可比**——哈希一样内容就一样，不需要保留原文。

> 代价是每次拍快照都要把最近碰过的文件完整读一遍算哈希。本地小项目文件都不大，可以接受；**换成大仓库这就是一次批量磁盘读了**，那时候得换成"比对修改时间和大小"这种更廉价的近似。

---


## 附：实测脚手架

`checkpoint.py` 用的是相对导入（`from .features import memory as memorylib`），所以不能像单文件模块那样用 `importlib` 直接加载，**要把项目根加进 `sys.path` 后按包导入**：

```python
import sys, tempfile
from pathlib import Path
sys.path.insert(0, r"E:\pico_agentharness\pico")
from pico import checkpoint as ck
```

桩对象需要提供的最小接口：

| 属性/方法                                                                                                      | 被谁用                                         |
| ---------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| `session`（含 `id` / `checkpoints` / `resume_state` / `runtime_identity`）                                    | 几乎全部                                        |
| `root`                                                                                                     | `current_runtime_identity`、`file_freshness` |
| `model_client`（要有 `.model`）                                                                                | `current_runtime_identity`                  |
| `approval_policy` / `read_only` / `max_steps` / `max_new_tokens` / `feature_flags` / `shell_env_allowlist` | `current_runtime_identity`                  |
| `prefix_state.workspace_fingerprint`（可选）                                                                   | `current_runtime_identity` 的降级分支            |
| `workspace.fingerprint()`                                                                                  | 同上                                          |
| `tool_signature()`                                                                                         | `current_runtime_identity`                  |
| `memory.to_dict()["working"]["recent_files"]`                                                              | `create_checkpoint`                         |
| `_ensure_session_shape()`                                                                                  | `checkpoint_state`                          |
| `invalidate_stale_memory()`                                                                                | `evaluate_resume_state`                     |
| `session_store.save()`                                                                                     | `create_checkpoint`                         |
| `resume_state`（**runtime 属性，不是 session 里的**）                                                               | `render_checkpoint_text`                    |

**最后一行最容易漏。** `render_checkpoint_text` 读的是 `agent.resume_state`，而这个属性由 `Pico.__init__` / `refresh_prefix` 赋值，不由 `evaluate_resume_state` 赋值——这就是 §3.3 说的双副本。

`task_state` 桩需要：`status` / `stop_reason` / `last_tool` / `final_answer` / `checkpoint_id`。

Windows 托管 Python：`C:/Users/yxqyx/.workbuddy/binaries/python/versions/3.13.12/python.exe`
