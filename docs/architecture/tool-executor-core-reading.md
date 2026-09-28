# tool_executor.py 核心逻辑精读（提炼版）

> 对象：`pico/tool_executor.py`（160 行）
> 定位：提炼版。讲骨架、设计理由、实测证据，不逐行走读。
> 同系列：`workspace-core-reading.md` / `tool-context-core-reading.md` / `tools-core-reading.md`
> **与 `tools-implementation-and-wiring.md` 的关系**：那份的"七道闸门"是从接入链路角度概述的；这份把 `execute` 的 116 行逐段拆开，实测每种拒绝路径的 metadata。
> 所有数字均为本机实测（Windows + Python 3.13.12），脚手架见文末。

---

## 1. 一句话概括

**160 行、1 个 frozen dataclass、1 个工厂函数、1 个类、1 个方法。这个文件是工具的"海关"——所有工具调用都必须从这里过，它决定放行还是拒绝，并把这次调用记成一份结构化档案。**

模块 docstring 只有一句：`"""Structured tool execution for the agent runtime."""`

**结构极度不均**：

| 部分 | 行数 | 占比 |
|---|---|---|
| `ToolExecutionResult` | 4 | 2% |
| `_metadata` 工厂 | 24 | 15% |
| `ToolExecutor.__init__` | 3 | 2% |
| **`ToolExecutor.execute`** | **116** | **73%** |

**整个文件就是一个方法。** 读懂 `execute` 就等于读懂了这个文件。

---

## 2. 行区间分布

| 行区间 | 内容 | 行数 |
|---|---|---|
| 1–6 | docstring + import | 6 |
| 9–12 | `ToolExecutionResult`（frozen dataclass） | 4 |
| 15–38 | `_metadata` 工厂 | 24 |
| 41–43 | `ToolExecutor.__init__` | 3 |
| 45–56 | 闸门 ①：`allowed_tools` 白名单 | 12 |
| 58–68 | 闸门 ②：工具是否存在 | 11 |
| 70–87 | 闸门 ③：参数校验 | 18 |
| 89–98 | 闸门 ④：重复调用 | 10 |
| 100–110 | 闸门 ⑤：审批 | 11 |
| 112–113 | 快照捕获（before） | 2 |
| 114–142 | **执行 + 成功路径记账** | **29** |
| 143–160 | **异常路径记账** | **18** |

---

## 3. 必须分清的概念

### 3.1 `content` 和 `metadata` 是两个不同的消费者

```python
@dataclass(frozen=True)
class ToolExecutionResult:
    content: str
    metadata: dict
```

| 字段 | 谁消费 | 用来干什么 |
|---|---|---|
| `content` | **模型** | 进 history，是模型看到的工具返回值 |
| `metadata` | **runtime / trace / 评测** | 判定状态、写 checkpoint、算指标 |

**两者内容完全不同，而且**——关键——**`content` 里没有 `metadata` 的任何信息**。

模型只看到 `"error: invalid arguments for read_file: path is not a file"`，看不到 `tool_error_code="invalid_arguments"`、`risk_level="low"`、`security_event_type=""`。**所有结构化的判定结果只对程序可见。**

这个划分是对的（模型不需要 JSON 元数据），但它带来一个后果：**如果 `content` 写得不清楚，模型无法从 metadata 里补救**。所以 `content` 的措辞必须自己扛住全部信息量。

### 3.2 `tool_status` 的四个取值

| 值 | 含义 | 出现位置 |
|---|---|---|
| `rejected` | 调用被拦下，工具**没执行** | 闸门 ①–⑤ |
| `ok` | 执行成功（或 run_shell 退出码为 0） | 成功路径 |
| `error` | 执行失败，且工作区没变 | 异常路径 / run_shell 非 0 退出且无改动 |
| `partial_success` | **执行失败但工作区真的变了** | 异常路径 / run_shell 非 0 退出但有改动 |

**`rejected` 和 `error` 的区别很重要**：前者是"没让它跑"，后者是"跑了但失败了"。

### 3.3 `risk_level` 说的是**工具**，不是**这次调用**

实测八种拒绝路径的 risk_level：

| 场景 | risk_level | read_only | security_event_type |
|---|---|---|---|
| `allowed_tools` 不含该工具 | `high` | `False` | —— |
| **工具名不存在** | **`high`** | `False` | —— |
| 参数校验失败（读工具） | `low` | `True` | —— |
| 参数校验失败（写工具） | `high` | `False` | —— |
| **路径逃逸（读工具）** | **`low`** | **`True`** | **`path_escape`** |
| 重复调用（读工具） | `low` | `True` | —— |
| 审批被拒（写工具） | `high` | `False` | `approval_denied` |
| 审批被拒（只读模式） | `high` | `False` | `read_only_block` |

**两个反直觉的点**：

**① 一次路径逃逸的 `risk_level` 是 `low`。** 因为它是 `validate_tool` 对 **`read_file`** 抛的异常，而 `read_file` 不 risky，所以走的是 `risk_level="high" if tool["risky"] else "low"` 分支。

**`risk_level` 描述的是"这个工具危不危险"，不是"这次调用危不危险"。** 真正的安全标记在 `security_event_type="path_escape"` 里。**两个维度需要一起看，只看 `risk_level` 会漏掉安全事件。**

**② 一个不存在的工具被标成 `risk_level="high"`。** 这条规则是代码里硬写的（`_metadata("rejected", ..., risk_level="high", read_only=False)`，第 63–67 行），理由大概是"未知即危险"。

**而"重复调用"这种明确的坏行为反而只有 `low`**（因为 `read_file` 不 risky）。**同一个字段在不同分支上的语义并不统一** ——有的按"这次调用有多可疑"，有的按"这个工具本身有多危险"。

### 3.4 `read_only` 字段的语义是"这次操作有没有写权限"

- 读工具（`risky=False`）→ `read_only=True`
- 写工具（`risky=True`）→ `read_only=False`
- **所有 `rejected` 且拒因是"工具不存在 / 不在白名单 / 审批被拒"的 → 一律 `False`**

**注意它和 `agent.read_only` 不是一回事**：后者是"整个会话是不是只读模式"。metadata 里的 `read_only` 是"**这个工具**在正常情况下是不是只写的"。

实测那两行能看出区别：

```
审批被拒（可写）    -> sec=approval_denied   risk=high  ro=False
审批被拒（只读模式） -> sec=read_only_block   risk=high  ro=False
```

**两种情况下的 `risk_level` 和 `read_only` 完全相同**，唯一区别是 `security_event_type`。所以**要区分"用户拒绝了"和"会话是只读模式"只能靠 `security_event_type`**。

### 3.5 `_metadata` 的键集**不稳定**

```python
def _metadata(tool_status, ..., workspace_fingerprint="", diff_summary=None):
    result = { ...8 个固定键... }
    if workspace_fingerprint:                       # ← 条件字段
        result["workspace_fingerprint"] = workspace_fingerprint
    return result
```

实测：

```
被拒绝时 keys: 8 个
成功时   keys: 9 个
差异: ['workspace_fingerprint']
```

**`workspace_fingerprint` 只在实际执行过工具时才存在。** 任何 `for key in metadata` 或者 `metadata["workspace_fingerprint"]` 的写法都会对不同的结果崩或漏。

**为什么用条件字段而不是写空字符串？** 因为空字符串会进 `tool_signature` 和 trace，让"没执行"和"执行了但指纹是空"混在一起。**用"键不存在"表达"这个字段不适用"是更干净的做法**——但代价是键集不固定。

---

## 4. 主流程骨架

八道闸门，顺序是刻意的：

```
execute(name, args)
  ① allowed_tools 白名单      → 不在名单  → rejected / tool_not_allowed
  ② 工具是否已注册             → 不存在    → rejected / unknown_tool
  ③ validate_tool 参数校验     → 抛异常    → rejected / invalid_arguments
                                               ↑ 可能带 path_escape
  ④ repeated_tool_call        → 重复      → rejected / repeated_identical_call
  ⑤ approve（仅 risky）        → 拒绝      → rejected / approval_denied
                                               ↑ 只读模式下记为 read_only_block
  ── 到这里为止：没有任何副作用 ──
  ⑥ capture_workspace_snapshot（仅 risky）  ← 第一次碰文件系统
  ⑦ tool["run"](args)  +  clip
  ⑧ 第二次快照 + diff + 记账
```

**顺序上最重要的设计：所有拒绝都发生在拍快照之前。**

闸门 ①–⑤ 全是纯内存判断（`validate_tool` 会读文件，但不写）。**一个被拒绝的调用不会在工作区留下任何痕迹**，也不会白拍两次快照（实测 risky 工具的快照次数是 **2 次**，非 risky 是 0 次）。

**闸门顺序的另一处刻意**：`approve` 排在 `repeated_tool_call` **之后**。所以一个重复调用不会弹审批窗——用户不会被无意义的弹窗打扰。

---

## 5. 核心函数逐个拆

### 5.1 `_metadata`（15–38）—— 唯一的元数据工厂

```python
def _metadata(tool_status, tool_error_code="", security_event_type="", risk_level="low",
              read_only=True, affected_paths=None, workspace_changed=False,
              workspace_fingerprint="", diff_summary=None):
```

**备注一：默认值选择传达了"最保守"的假设。**

`risk_level="low"` 和 `read_only=True` 是默认——**但这不是安全的默认**。看调用点：两条"未知工具"的拒绝路径都**显式**传了 `risk_level="high", read_only=False`，而参数校验失败那条是按 `tool["risky"]` 算的。

所以默认值只被成功路径用到（那里 `risk_level` 也是显式传的）。**默认值实际上从未生效** ——所有调用点都显式传了这两个参数。

**备注二：`list(affected_paths or [])` 做了两层防御。**

`affected_paths=None` 时变成 `[]`，传了列表时复制一份。**复制很重要**——`diff_workspace_snapshots` 返回的列表如果被外部改动，metadata 里的记录会跟着变。frozen dataclass 冻的是字段引用，冻不住里面的 dict 和 list。

**所以 `ToolExecutionResult` 是"浅冻结"的**：`result.metadata["affected_paths"].append(...)` 是允许的。

### 5.2 `execute` 的八道闸门（45–160）

**闸门 ①：白名单（47–56）**

```python
if agent.allowed_tools is not None and name not in agent.allowed_tools:
```

**`is not None` 是必须的**：`allowed_tools=None` 表示"不限制"，而 `[]` 表示"一个都不许"。**用 `None` 和空集合表达两种不同的语义**，这在配置接口里是个常见但容易写错的地方（写成 `if agent.allowed_tools and ...` 就会让 `[]` 变成"不限制"，正好反了）。

**闸门 ②：工具存在（58–68）**

`agent.tools.get(name)` 返回 `None` 就拒绝。**工具没注册**（比如深度耗尽时的 `delegate`）会走到这里。

**闸门 ③：参数校验（70–87）**

```python
try:
    agent.validate_tool(name, args)
except Exception as exc:
    example = agent.tool_example(name)
    message = f"error: invalid arguments for {name}: {exc}"
    if example:
        message += f"\nexample: {example}"
    security_event_type = "path_escape" if "path escapes workspace" in str(exc) else ""
```

**备注一：错误信息里附上示例。**

`tool_example(name)` 返回 `TOOL_EXAMPLES` 里那条。**这是把"你错了"变成"你错了，正确写法是这样"** ——对模型来说，示例比描述有效得多。

**备注二：`except Exception` 吞掉了 `KeyError`。**

`validate_tool` 里三个工具用 `args["path"]` 直接下标，缺参数会抛 `KeyError`。**这一层把它转成了 `invalid_arguments`**，所以模型看到的是统一的错误格式，而不是 `KeyError: 'path'` 的裸异常。

**但 `{:exc}` 会把 `'path'` 拼进去**，所以消息末尾还是带着 Python 味儿的 `KeyError: 'path'`。

**闸门 ④：重复调用（89–98）**

`agent.repeated_tool_call(name, args)` 由 runtime 实现。拒绝信息很直接：

> `error: repeated identical tool call for {name}; choose a different tool or return a final answer`

**这句话是在教模型怎么办**，不只是说"你错了"。这很重要——一个只会说 "repeated call" 的错误信息会让模型反复重试。

**闸门 ⑤：审批（100–110）**

```python
if tool["risky"] and not agent.approve(name, args):
```

**只有 risky 工具走审批。** 拒绝时的 `security_event_type` 依 `agent.read_only` 二选一：`read_only_block` 或 `approval_denied`。

**（跨模块）** `approve` 的实现里，`read_only` 检查**排在 `approval_policy` 之前** ——所以子 agent 的 `read_only=True` 是硬锁，改审批策略也绕不过去。

**快照（112–113）**

```python
before_snapshot = agent.capture_workspace_snapshot() if tool["risky"] else {}
after_snapshot = before_snapshot
```

**`after_snapshot = before_snapshot` 这个赋值是给非 risky 路径兜底的**：非 risky 工具不重拍，`diff(before, before)` 会返回空改动。**所以 `workspace_changed` 对非 risky 工具恒为 False。**

实测快照次数：

```
非 risky 工具 snapshot 次数: 0
risky 工具    snapshot 次数: 2
```

**成功路径（114–142）**

```python
content = clip(tool["run"](args))                                  # ★ 唯一的输出裁剪点
after_snapshot = agent.capture_workspace_snapshot() if tool["risky"] else before_snapshot
affected_paths, diff_summary = agent.diff_workspace_snapshots(before_snapshot, after_snapshot)
workspace_changed = bool(affected_paths)
tool_status = "ok"
if name == "run_shell":
    match = re.search(r"exit_code:\s*(-?\d+)", content)
    exit_code = int(match.group(1)) if match else 0
    if exit_code != 0 and workspace_changed:
        tool_status = "partial_success"; tool_error_code = "tool_partial_success"
    elif exit_code != 0:
        tool_status = "error"; tool_error_code = "tool_failed"
agent.update_memory_after_tool(name, args, content)
metadata = _metadata(...)
agent.record_process_note_for_tool(name, metadata)
return ToolExecutionResult(content=content, metadata=metadata)
```

**备注一：`clip(tool["run"](args))` 是唯一的输出裁剪点。**

`clip` 的默认上限是 `workspace.MAX_TOOL_OUTPUT = 4000`。**所有工具的输出都在这里被统一裁到 4000 字符**（实际会略超，因为 `clip` 会加截断提示）。

**这个位置选得很好**：在工具外面裁，工具自己不用管长度。**代价是所有工具共享一个上限**——一个需要看 5000 行的场景和一次 `list_files` 用的是同一个数字。

**备注二：退出码是从输出文本里正则抠出来的。**

```python
match = re.search(r"exit_code:\s*(-?\d+)", content)
exit_code = int(match.group(1)) if match else 0
```

**这是全文件最脆的一处。** `tool_run_shell` 的输出模板和这里的正则是**一对隐式契约**：改模板的措辞会静默改变状态判定。

**注意 `if match else 0`**：匹配不到时**默认当成 0（成功）**。实测：

```
run_shell 输出带 'exit_code: 1'  -> status=error
run_shell 输出里没有 exit_code 行 -> status=ok     ← 失败被报成成功
```

**这是"匹配失败时选最乐观的值"**。改成"匹配不到时报错"更安全，但显然作者认为"格式是我自己写的，不会不匹配"。

**备注三：`partial_success` 的判定是"退出码非 0 + 工作区变了"。**

```python
if exit_code != 0 and workspace_changed:
    tool_status = "partial_success"
```

**为什么单独标一个状态？** 因为如果统一报 `error`，模型会以为什么都没发生然后重试——而第二次可能把已经改好的部分又改坏。`partial_success` + 一条带行动指令的记忆（在 `update_memory_after_tool` 里写 `inspect diff before retry`）是在说"**有东西变了，先看看再决定**"。

**备注四：`update_memory_after_tool` 在成功路径有，异常路径没有。**

见 §10.3。

**异常路径（143–160）**

```python
except Exception as exc:
    after_snapshot = agent.capture_workspace_snapshot() if tool["risky"] else before_snapshot
    affected_paths, diff_summary = agent.diff_workspace_snapshots(before_snapshot, after_snapshot)
    workspace_changed = bool(affected_paths)
    security_event_type = "path_escape" if "path escapes workspace" in str(exc) else ""
    metadata = _metadata("partial_success" if workspace_changed else "error", ...)
    agent.record_process_note_for_tool(name, metadata)
    return ToolExecutionResult(content=f"error: tool {name} failed: {exc}", metadata=metadata)
```

**备注一：异常路径也拍快照、也算 diff、也记 process note。**

**这是这个文件里最值得抄的设计**：工具中途抛异常时，**工作区可能已经被改了一半**。如果不检查，就会把"改了一半"报成"完全失败"。

所以异常分支做的事和成功分支**几乎一样**（快照、diff、状态判定、记账），只有三处不同：
1. `tool_status` 是 `partial_success` / `error` 而不是 `ok`
2. 错误码是 `tool_partial_success` / `tool_failed`
3. **不调 `update_memory_after_tool`**

**备注二：异常路径的 `security_event_type` 只识别 `path_escape`。**

一个工具在运行期抛 `path escapes workspace`（而不是在校验期）会被正确分类。但其他安全相关的异常没有分类——**只有这一句话是安全事件的判据**。

实测这句话的脆弱性：

```
'path escapes workspace'               -> path_escape
'path escapes workspace: /etc/passwd'  -> path_escape
'Path Escapes Workspace'               -> (未分类)      ← 大小写敏感
'路径逃逸'                              -> (未分类)
```

**`in` 是大小写敏感的**，所以把异常消息改成大写、或者改成中文，**安全事件分类会静默失效**。

---

## 6. 设计理由

### 6.1 为什么所有拒绝都在拍快照之前？

**好处**：一个被拒绝的调用**不留任何痕迹**。不拍快照（省两次全仓库哈希遍历）、不弹审批（如果是重复调用）、不写记忆。

实测非 risky 工具的快照次数是 **0**，risky 工具是 **2**。如果一个非法参数能走到快照那一步，每次错误调用都要白跑两遍 `capture_workspace_snapshot`——那是个 O(仓库文件数) 的 sha256 遍历。

**代价**：`validate_tool` 里 `patch_file` 会**读一次完整文件**来数出现次数（数完还要在执行时再读一次）。所以"零副作用"不完全成立——**它不写，但会读**。

### 6.2 为什么 metadata 这么多字段，而不是一个状态字符串？

**好处**：一次调用的完整画像。`tool_status` 说结果，`tool_error_code` 说原因，`security_event_type` 说安全面，`affected_paths` 说影响面，`diff_summary` 说改了什么。

**这些字段各有各的消费者**：trace 报告要看 `security_event_type`，评测要看 `tool_status`，`checkpoint` 要看 `affected_paths` 来判断哪些文件需要重新读。

**代价**：字段多了以后**很难保证一致**。实测 `risk_level` 在八个分支上有两种不同的语义（§3.3），`read_only` 在拒绝路径上被硬编码成 `False` 而不是照实描述工具。

### 6.3 为什么异常路径不做 `update_memory_after_tool`？

代码里没有解释。可能的理由是：**异常路径下 `content` 是一句 `error: tool x failed: ...`，这个字符串作为记忆摘要没有价值**——而 `update_memory_after_tool` 的逻辑（在 `runtime.py:405`）正是按工具名分派、对 `read_file` 的返回做摘要。

但**它同时也负责 `remember_file`** ——失败路径下工作区可能真的变了（`partial_success` 就是为此存在的），**而那个变化的文件不会进 `recent_files`**。

**这是一个真实的缺口**：metadata 说"工作区变了、这些文件受影响"，但记忆层不知道。模型下一轮问"我刚改了什么"时，记忆里没有。

### 6.4 为什么 `ToolExecutionResult` 是 frozen？

**好处**：工具执行的结果应该是**不可变的既成事实**。frozen 保证下游（历史记录、trace）拿到后不会被别人改掉。

**代价**：**只是浅冻结**。`metadata` 是个 dict，`result.metadata["tool_status"] = "ok"` 是允许的。**frozen 挡住了字段重绑定，没挡住内容修改。**

**取舍判断**：对一个内部结果对象，浅冻结够用了。但**不要把它误当成"完全不可变"** ——如果 metadata 会跨线程或跨轮次复用，那还是要 `copy.deepcopy`。

---

## 7. 实测验证

用桩 agent 枚举每条路径。

### 7.1 八种拒绝路径的 metadata

| 场景 | `tool_status` | `tool_error_code` | `security_event_type` | `risk_level` | `read_only` |
|---|---|---|---|---|---|
| 不在白名单 | `rejected` | `tool_not_allowed` | —— | `high` | `False` |
| 工具不存在 | `rejected` | `unknown_tool` | —— | `high` | `False` |
| 校验失败（读工具） | `rejected` | `invalid_arguments` | —— | `low` | `True` |
| 校验失败（写工具） | `rejected` | `invalid_arguments` | —— | `high` | `False` |
| **路径逃逸（读工具）** | `rejected` | `invalid_arguments` | **`path_escape`** | **`low`** | **`True`** |
| 重复调用（读工具） | `rejected` | `repeated_identical_call` | —— | `low` | `True` |
| 审批被拒 | `rejected` | `approval_denied` | `approval_denied` | `high` | `False` |
| 审批被拒（只读模式） | `rejected` | `approval_denied` | `read_only_block` | `high` | `False` |

**注意最后两行：除了 `security_event_type` 完全一样。**

### 7.2 metadata 键集不稳定

```
被拒绝时 keys: 8 个（无 workspace_fingerprint）
成功时   keys: 9 个（含 workspace_fingerprint）
```

### 7.3 成功路径 vs 异常路径的调用轨迹

```
成功: ['validate', 'repeated_check', 'diff', 'update_memory', 'record_note']
异常: ['validate', 'repeated_check', 'approve', 'snapshot', 'snapshot', 'diff', 'record_note']
                                                              ↑ 异常路径独缺 'update_memory'
```

### 7.4 快照次数

```
非 risky 工具: 0 次
risky 工具  : 2 次（执行前、执行后各一次）
```

### 7.5 `run_shell` 的退出码判定

| 场景 | `tool_status` | `tool_error_code` |
|---|---|---|
| 退出码 0 | `ok` | （空） |
| 退出码 1，无改动 | `error` | `tool_failed` |
| 退出码 1，有改动 | `partial_success` | `tool_partial_success` |
| **输出里没有 `exit_code:` 行** | **`ok`** | （空） |

### 7.6 退出码正则的边界

| 输入 | 抠出的值 |
|---|---|
| `"exit_code: 1"` | `1` |
| `"exit_code: -9"` | `-9` |
| `"exit_code:1"` | `1` |
| `"exit_code:  12"` | `12` |
| `"stdout:\nexit_code: 7"` | `7`（取第一个匹配） |
| `"没有这一行"` | `None` → **兜底按 0 处理** |

### 7.7 安全事件分类的大小写敏感性

```
'path escapes workspace'               -> path_escape
'path escapes workspace: /etc/passwd'  -> path_escape
'Path Escapes Workspace'               -> (未分类)
'路径逃逸'                              -> (未分类)
```

### 7.8 `ToolExecutionResult` 是 frozen

```
r.content = "y"  ->  FrozenInstanceError: cannot assign to field 'content'
```

但 `metadata` 是可变 dict（浅冻结）。

---

## 8. 不变量

| # | 不变量 | 验证方式 | 实测 |
|---|---|---|---|
| I1 | 拒绝路径不拍快照 | 计数 | ✓（只有违规工具以下才拍） |
| I2 | 成功路径必调 `update_memory_after_tool` | 轨迹 | ✓ |
| I3 | 异常路径也拍快照并 diff | 轨迹 | ✓ |
| I4 | 异常路径也调 `record_process_note_for_tool` | 轨迹 | ✓ |
| I5 | 非 risky 工具不拍快照 | 计数 | ✓（0 次） |
| I6 | 每个结果都有 `tool_status` | 键集 | ✓ |
| I7 | metadata 键集固定 | 对比成功/拒绝 | ✗ **被推翻**，差一个 `workspace_fingerprint` |
| I8 | `risk_level` 反映这次调用的危险度 | 看路径逃逸 | ✗ **被推翻**，路径逃逸是 `low` |
| I9 | `run_shell` 失败必被标为 error | 去掉 exit_code 行 | ✗ **被推翻**，标成了 `ok` |
| I10 | 异常路径也更新工作记忆 | 轨迹 | ✗ **被推翻**，独缺 `update_memory` |
| I11 | `ToolExecutionResult` 完全不可变 | 改 metadata | ✗ **被推翻**，浅冻结 |

---

## 9. 精读顺序建议

1. **`ToolExecutionResult`（9–12）** —— 4 行。先建立"每次调用产出 = 给模型的文本 + 给程序的档案"这个二分。
2. **`_metadata`（15–38）** —— 看那 8 个固定字段，尤其是末尾的 `if workspace_fingerprint` 条件分支。
3. **`execute` 的前五道闸门（45–110）** —— 逐道看，每道都问"这次拒绝留下了什么副作用"。答案是"没有"。
4. **成功路径（112–142）** —— 三处重点：`clip` 的位置、退出码正则、`partial_success` 的判定条件。
5. **异常路径（143–160）** —— 和成功路径逐行对照，找出三个不同点（状态值、错误码、缺 `update_memory`）。
6. **回到调用方** —— `agent_loop` 里拿到 `ToolExecutionResult` 之后怎么处理 `content` 和 `metadata`（前者进 history，后者进 trace 和 checkpoint）。

---

## 10. 已知粗糙处

按约定**只报告，不修改**。

### 10.1 退出码依赖字符串匹配，失败时默认乐观

```python
match = re.search(r"exit_code:\s*(-?\d+)", content)
exit_code = int(match.group(1)) if match else 0
```

和 `tools.tool_run_shell` 的输出模板是隐式契约。**匹配不到时兜底成 `0`（成功）** ——实测一个不含 `exit_code:` 行的输出会被判成 `tool_status="ok"`。**失败被报成成功，是这套机制里最危险的单点。**

### 10.2 安全事件分类依赖一句异常措辞，且大小写敏感

```python
security_event_type = "path_escape" if "path escapes workspace" in str(exc) else ""
```

同一个表达式在成功路径外的两处（77 行、147 行）各写了一遍。改措辞或改成中文，分类会**静默失效**。

### 10.3 异常路径不做 `update_memory_after_tool`（两处，成功路径有）

实测轨迹：

```
成功: [..., 'diff', 'update_memory', 'record_note']
异常: [..., 'diff', 'record_note']          ← 缺 update_memory
```

**后果**：`partial_success` 场景下工作区确实变了（metadata 里 `affected_paths` 有值），但**那个文件不会进 `recent_files`，也不会有摘要**。模型下一轮问"我刚改了什么"时记忆里没有。

代码里没有注释解释这个不对称。

### 10.4 `_metadata` 的 `risk_level` 有两种语义

同一个字段，在"工具不存在"分支上是"未知即危险"（硬编码 `high`），在"参数校验失败"分支上是"这个工具本身危不危险"（按 `tool["risky"]`）。

**结果是"路径逃逸"这种明确的安全事件 `risk_level` 是 `low`。** 要识别它必须读 `security_event_type`。

### 10.5 `metadata` 键集不稳定

`workspace_fingerprint` 是条件字段，拒绝路径没有它。任何按固定键集遍历 metadata 的代码都要注意。

### 10.6 `read_only` 字段在拒绝路径被硬编码

"工具不存在"、"不在白名单"、"审批被拒"三条路径都传 `read_only=False`，**即使被拒的是 `read_file`**。字段名容易被误读成"这次操作是不是只读的"。

### 10.7 所有工具共享 `clip` 的 4000 上限

`clip(tool["run"](args))` 用默认上限。一次 `list_files` 和一次需要看长输出的场景用同一个数字，没有按工具区分。

### 10.8 `ToolExecutionResult` 是浅冻结

`metadata` 是可变 dict。frozen 只挡住字段重绑定。

### 10.9 成功路径和异常路径的记账代码高度重复

两段各 18–29 行，做的事几乎一样（拍快照、diff、算 `workspace_changed`、判定状态、记 process note），只是状态值和错误码不同。**将来加一个 `security_event_type` 的判断，很容易只改一边。**

### 10.10 `except Exception` 的范围过大

成功路径的 `try` 包住了 `tool["run"](args)`、`clip`、两次快照、diff、`update_memory_after_tool`、`_metadata`、`record_process_note_for_tool`。

**所以"工具执行失败"和"记账代码有 bug"会被报成同一种错误**（`tool_failed`），而后者其实是 pico 自己的缺陷，不是工具的问题。排查时会被误导。

---

## 11. 面试问答

**Q：工具调用是怎么被拦的？**

> 五道闸门，顺序是刻意的。

> 第一道看这个工具**在不在本次运行的白名单里**——用户可以用参数限制这次只允许哪些工具。第二道看**工具有没有被注册**——比如子 agent 深度耗尽时，连"派生子 agent"这个工具本身都不会被注册。

> 第三道做**参数校验**。第四道查**是不是重复调用**——同一个工具同样的参数连着叫两次，基本可以确定模型卡住了。第五道才是**审批**，而且只有高风险工具才走。

**关键设计是：所有拒绝都发生在拍工作区快照之前。**

**为什么重要？** 我的高风险操作会在执行前后各拍一次工作区快照——那是把整个仓库的文件都做一遍哈希。如果参数非法也要拍两次，每次错误调用都是纯浪费，而且**拒绝的调用不该在工作区留下任何痕迹**。

**Q：为什么审批排在重复调用检查的后面？**

> 因为一个重复调用根本不值得打扰用户。

> 如果顺序反了，模型卡在循环里的时候，用户会连续收到一堆一模一样的审批弹窗——**既没意义又让人烦躁**。先判断"这是个重复调用"，直接拒掉，用户体验好很多。

**Q：工具失败了怎么判断？**

> 分两种情况。

> **执行抛异常了**——这时候我会再拍一次工作区快照，和之前的比。**如果工作区真的变了，状态是"部分成功"，而不是"失败"。** 因为工具可能已经改了一半文件才出错。

> **`run_shell` 退出码非 0**——同样看工作区变没变：变了就是部分成功，没变才是失败。

**为什么要单独标"部分成功"？** 因为如果统一报"失败"，模型会以为什么都没发生，然后重试——**而第二次重试可能把已经改好的那部分又改坏**。所以我给它一个专门的状态，还往记忆里写一条带行动指令的提示："重试之前先看 diff"。

**Q：这个文件有什么坑？**

> 两个，都在 `run_shell` 的判定上。

> **第一个是退出码是从输出文本里正则抠出来的。** 工具的返回格式和这里的正则是**一对隐式契约**——谁改了输出模板的措辞，状态判定就会静默改变，不会报错。**而且匹配不到的时候我兜底成了 0，也就是"成功"。** 实测把输出里那行去掉，一个失败的命令会被报成 `ok`。这个兜底方向选错了，应该报错而不是当成成功。

> **第二个是安全事件的分类。** 我用 `"path escapes workspace" in str(exc)` 来判断这次是不是路径逃逸——**一句话的措辞就是安全分类的唯一依据，而且是大小写敏感的**。把异常消息改成大写，或者翻译成中文，分类就会静默失效，安全事件变成普通错误。

**Q：有没有你觉得该改但还没改的？**

> 有一个不对称：**成功路径会更新工作记忆，异常路径不会。**

> 这条记忆更新负责把"我刚碰过这个文件"记下来，供下一轮参考。但异常路径下——尤其是"部分成功"那种情况——工作区是真的变了，metadata 里也记了受影响的文件，**可记忆层完全不知道**。模型下一轮问"我刚改了什么"，答案里没有这个文件。

> 我怀疑当初是觉得"异常路径的输出是一句错误信息，拿它当摘要没价值"就跳过了，**但那个函数不只做摘要，它还负责记住文件**。这两件事被混在一个函数里，所以一起被跳掉了。

**Q：metadata 是给谁看的？**

> 不是给模型看的，是给**程序**看的——trace、报告、评测指标、还有检查点。

> 模型只拿到 `content`，也就是工具返回的那段文本。**它看不到状态码、看不到风险等级、看不到安全事件类型。** 这个划分是有意的——模型不需要 JSON 元数据。

> 但有个后果：**如果 `content` 写得不清楚，模型没有任何补救渠道**，因为结构化信息它拿不到。所以错误信息的措辞必须自己扛住全部信息量。这也是为什么我在被拒绝的时候会**把工具的正确用法示例一起附上**——让模型知道"你错了，而且知道怎么改"。

---

## 附：实测脚手架

```python
import sys
from pathlib import Path
sys.path.insert(0, r"E:\pico_agentharness\pico")
from pico import tool_executor as te

class Agent:
    def __init__(self, tools, allowed=None, read_only=False, approve=True, repeated=False,
                 changed=False, validate_raises=None):
        self.tools = tools; self.allowed_tools = allowed; self.read_only = read_only
        self._approve = approve; self._repeated = repeated
        self._changed = changed; self._validate_raises = validate_raises
        self.workspace = type("WS", (), {"fingerprint": lambda s: "FP"})()
        self.trace = []; self.snaps = 0
    def validate_tool(self, n, a):
        if self._validate_raises: raise self._validate_raises
    def tool_example(self, n): return "<tool>example</tool>"
    def repeated_tool_call(self, n, a): return self._repeated
    def approve(self, n, a): return self._approve
    def capture_workspace_snapshot(self):
        self.snaps += 1; return {"n": self.snaps}
    def diff_workspace_snapshots(self, a, b):
        return (["a.py"] if self._changed else []), (["a.py: +1 -1"] if self._changed else [])
    def update_memory_after_tool(self, n, a, c): self.trace.append("update_memory")
    def record_process_note_for_tool(self, n, m): self.trace.append("record_note")

def spec(runner, risky=False):
    return {"risky": risky, "run": runner, "schema": {}, "description": ""}

a = Agent({"read_file": spec(lambda args: "ok")})
r = te.ToolExecutor(a).execute("read_file", {})
print(r.metadata)
```

**想观察调用轨迹**：把桩 agent 的每个方法都改成 `self.trace.append("<方法名>")` 再返回，跑完打印 `trace`，就能看出成功/异常两条路径的差异（实测唯一差别是异常路径缺 `update_memory`）。

**测"无 exit_code 行会被判成功"**：让桩 runner 返回一个不含 `exit_code:` 的字符串，`name="run_shell"`。

Windows 托管 Python：`C:/Users/yxqyx/.workbuddy/binaries/python/versions/3.13.12/python.exe`
