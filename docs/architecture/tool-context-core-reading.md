# tool_context.py 核心逻辑精读（提炼版）

> 对象：`pico/tool_context.py`（21 行）
> 定位：提炼版。**这是本系列里最短的一个文件，所以结构做了压缩**——不硬凑段落，重点放在"它不暴露什么"上。
> 同系列：`workspace-core-reading.md` / `tools-core-reading.md` / `tool-executor-core-reading.md`
> 所有数字均为本机实测（Windows + Python 3.13.12），脚手架见文末。

---

## 1. 一句话概括

**21 行、1 个 dataclass、6 个字段、2 个方法。这个文件的价值不在它写了什么，而在它没写什么。**

它的全部内容就是：

```python
@dataclass
class ToolContext:
    root: Path
    path_resolver: Callable[[str], Path]
    shell_env_provider: Callable[[], dict]
    depth: int
    max_depth: int
    spawn_delegate: Callable[[dict], str]

    def path(self, raw_path):
        return self.path_resolver(str(raw_path))

    def shell_env(self):
        return self.shell_env_provider()
```

模块 docstring 一句话说清了定位：**"Narrow context passed from runtime into tool functions."**（从 runtime 传进工具函数的**窄**上下文。）

实测对比：

```
ToolContext 字段数:        6
Pico 对外可访问成员数:     70
```

**这个 6:70 的比例就是整个文件的设计意图。** 工具函数拿不到 `session`、拿不到 `model_client`、拿不到 `tools`、拿不到 `approve`、拿不到 `memory`——**它想"越权"也没有入口**。

---

## 2. 行区间分布

| 行区间 | 内容 | 行数 |
|---|---|---|
| 1 | docstring | 1 |
| 3–5 | `dataclass` / `Path` / `Callable` 导入 | 3 |
| 8–15 | 类声明 + 6 个字段 | 8 |
| 17–18 | `path()` | 2 |
| 20–21 | `shell_env()` | 2 |
| 空行 | 2, 6, 7, 16, 19 | 5 |

**没有任何逻辑。** 构造函数是 dataclass 自动生成的，两个方法各一行转发。所以这个文件的正确读法是**看字段清单**，不是读代码。

---

## 3. 必须分清的概念

### 3.1 两个 `path` 不是一个东西

```python
path_resolver: Callable[[str], Path]      # 字段名
def path(self, raw_path):                 # 方法名
```

`path_resolver` 是**注入进来的闭包**（runtime 里是 `self.path`），`path()` 是**对它的包装**。同名不同层，和 `session_store.py` 里 `path` / `Path` / `path` 三处同名是同一类问题。

### 3.2 沙箱不在这个文件里

`ToolContext.path()` 只有一行：

```python
return self.path_resolver(str(raw_path))
```

**它不做任何校验。** 所有路径安全检查都在 runtime 注入的那个 resolver 里。实测用一个会拒绝 `..` 的 resolver：

```
c3.path('a.py')             -> a.py
c3.path('../etc/passwd')    -> ValueError: path escapes workspace
```

**这个文件只是把闭包转发出去。** 想找沙箱逻辑，得去 `runtime.py` 看 `path_resolver=self.path` 那个 `self.path` 的实现。

**为什么这样设计？** 因为"什么叫合法路径"是 **runtime 的策略**，不是工具层的策略。同一个工具函数在 CLI 里和评测 harness 里可能面对不同的沙箱规则，把策略注入进来比写死在这里灵活。

**代价**：读 `tool_context.py` 完全看不出任何安全保证。必须顺着 `path_resolver` 的注入点跳到 runtime 才能确认"到底安不安全"。

### 3.3 `root` 和 `path_resolver` 是两个东西

- `root`：工作区根目录，**工具用它做 `relative_to`**（生成给模型看的短路径）
- `path_resolver`：把用户给的字符串**解析 + 校验**成绝对 `Path`

**实测 `tools.py` 里的用法分工**：

```python
path = context.path(args["path"])                      # 解析 + 校验
... f"{kind} {entry.relative_to(context.root)}"        # 用 root 生成显示路径
```

所以 `root` 不只是"给 resolver 用的数据"，它自己也是个工具。**如果 `path_resolver` 返回的路径不在 `root` 下面，`relative_to` 会抛 `ValueError`** —— 这是两个字段之间一个隐式的、没有文档的契约。

### 3.4 `spawn_delegate` 是唯一的"逃逸通道"

`ToolContext` 里只有一个字段是**函数**且**能做 IO 之外的事**：

```python
spawn_delegate: Callable[[dict], str]
```

它转发到 `Pico.spawn_delegate`（`runtime.py:588`），那里会**新建一个完整的子 `Pico` 对象**：

```python
child = Pico(
    model_client=self.model_client, workspace=self.workspace, session_store=self.session_store,
    approval_policy="never", max_steps=int(args.get("max_steps", 3)),
    depth=self.depth + 1, max_depth=self.max_depth, read_only=True,
    ...
)
```

**这是工具层唯一能"启动新事务"的入口。** 它的安全边界靠三层锁串起来：

| 锁 | 位置 |
|---|---|
| `approval_policy="never"` | 子 agent 不会弹审批 |
| `read_only=True` | 写操作被硬锁（此检查在 `approve()` 里排在 `approval_policy` 之前） |
| `depth=self.depth + 1` | 深度耗尽时 `build_tool_registry` 干脆不注册 `delegate` |

**注意 `spawn_delegate` 恰好也是 `Pico` 的一个方法名**——实测对比字段名和 Pico 成员，唯一重合的就是它。这是有意的：字段名直接照搬被注入的方法名，减少心智负担。

---

## 4. 主流程骨架

这个文件没有"流程"，只有一条注入链：

```
runtime.Pico.tool_context()                    # runtime.py:578-586
  └─ ToolContext(
         root               = self.root
         path_resolver      = self.path              ← 沙箱在这里
         shell_env_provider = self.shell_env         ← 环境白名单在这里
         depth              = self.depth
         max_depth          = self.max_depth
         spawn_delegate     = self.spawn_delegate    ← 子 agent 在这里
     )
       ↓
tools.build_tool_registry(context)             # 用 partial 把 context 冻进每个 runner
       ↓
tool["run"] = partial(_TOOL_RUNNERS[name], context)
       ↓
工具函数签名统一为 (context, args)
```

**关键点**：`tool_context()` **每次调用都新建一个 `ToolContext`**（`runtime.py:578` 是普通方法，不是属性缓存）。所以它是个**快照**，不是活引用——运行期改 `ToolContext.depth` 不会影响 runtime。

---

## 5. 核心内容逐个拆

### 5.1 字段：为什么是这 6 个

| 字段 | 给了什么能力 | 为什么必须给 |
|---|---|---|
| `root` | 生成相对的显示路径 | 工具输出里要写 `a.py` 而不是 `C:\Users\...\a.py` |
| `path_resolver` | 把字符串变安全路径 | 所有文件工具的第一步 |
| `shell_env_provider` | 拿到过滤后的环境变量 | `run_shell` 要用 |
| `depth` / `max_depth` | 知道自己处在递归第几层 | `delegate` 要判断能不能再分 |
| `spawn_delegate` | 启动子 agent | `delegate` 工具的实现 |

**反过来看没给的**：没有 `session`（读不到对话历史）、没有 `memory`（读不到工作记忆）、没有 `model_client`（不能自己调模型）、没有 `workspace`（读不到仓库快照）、没有 `approve`（不能自己批准自己）。

**这就是"窄上下文"的全部含义**：工具的能力面 = 6 个字段能表达的操作，**没有钻空子的余地**。

### 5.2 `path()`（17–18）

```python
def path(self, raw_path):
    return self.path_resolver(str(raw_path))
```

**备注：那个 `str()` 是必要的防御。**

模型给的参数来自 JSON 解析，正常是 `str`，但理论上可能是数字（`read_file` 传 `path: 123`）。`str()` 保证 resolver 拿到的永远是字符串。

**代价**：`str(123)` 会变成 `"123"`，然后 resolver 会把它当文件名——**得到"文件不存在"而不是"参数类型错误"**。更严格的做法是在这里做类型检查，但那会把校验责任从 `validate_tool` 拉到这里，破坏"校验统一在 `validate_tool`"的分工。

### 5.3 `shell_env()`（20–21）

```python
def shell_env(self):
    return self.shell_env_provider()
```

**备注：它每次调用都重新问 provider，没有缓存。**

实测：

```
三次调用: {'PATH': 'v1'} {'PATH': 'v2'} {'PATH': 'v3'}
provider 被调用 3 次
```

provider 是 `Pico.shell_env`（`runtime.py:301`）：

```python
def shell_env(self):
    return securitylib.shell_env(allowlist=self.shell_env_allowlist, root=self.root)
```

每次都会重新构造一份白名单环境字典。**`run_shell` 一次调用只问一次，所以开销可以忽略**。但这是个"每次都要重新算"的形态——如果哪天有人在循环里调它，就会变成热点。

**不缓存是对的选择**：环境变量是安全边界，缓存意味着"运行期改了 allowlist 不生效"。**宁可每次算，不可算错。**

---

## 6. 设计理由

### 6.1 为什么不用 `agent` 直接传进去？

**好处**：工具的"可做之事"变成了一个 6 行的 dataclass，**一眼看完**。审计"这个工具能碰什么"不需要读工具代码，只需要读这个文件。

**代价**：工具想做的事如果超出这 6 个字段，就只能改 `ToolContext` 本身。加一个字段意味着**所有工具都获得了这个能力**——粒度是粗糙的。

**取舍判断**：对的。对一个"希望模型只能做有限事情"的系统，**能力面的可见性和可审计性比扩展性重要**。

### 6.2 为什么用 `Callable` 注入而不是继承/参数传递？

**好处**：`ToolContext` 不知道沙箱怎么实现、不知道环境白名单怎么算。**策略和机制分离**——同一套工具函数可以配不同的 resolver（CLI 一套、评测 harness 一套）。

**代价**：读这个文件完全看不出安全保证。**"这里安全吗"这个问题在 `tool_context.py` 里无解**，必须跳到 runtime。

**取舍判断**：合理。但代价应该被缓解——比如在 docstring 里写一句"路径安全由注入的 `path_resolver` 负责，见 `runtime.Pico.path`"。现在只有一句 "Narrow context passed from runtime into tool functions."，读者不会知道要去哪找沙箱。

### 6.3 为什么是可变 dataclass 而不是 frozen？

**实测**：

```python
c.depth = 99      # 成功
```

它是普通 `@dataclass`，不是 `frozen=True`。**对比 `tool_executor.ToolExecutionResult` 是 frozen 的**——同一个模块族里两种选择。

**为什么这里不 frozen？** 没有明确理由。`ToolContext` 从创建到被 `partial` 冻进工具函数，中间没有任何地方需要改它。**`frozen=True` 应该也能工作，而且更安全**（防止工具函数在运行期改自己的 `depth` 来绕过限制）。

这是一个可以收紧但没收紧的地方。

### 6.4 为什么 `depth` / `max_depth` 是裸 `int`？

**好处**：简单。`tools.build_tool_registry` 里就是一句 `if context.depth < context.max_depth`。

**代价**：**没有范围约束**。实测：

```python
ToolContext(depth=-5, max_depth=-1, ...)     # 构造成功
# -5 < -1 为 True -> delegate 仍然会被注册
```

负数的深度会得到反直觉的结果。实际 runtime 传的都是非负值，所以没暴露。

---

## 7. 实测验证

### 7.1 隔离面：6 vs 70

```
ToolContext 字段: ['root', 'path_resolver', 'shell_env_provider', 'depth', 'max_depth', 'spawn_delegate']
字段数: 6
Pico 对外可访问成员数: 70
直接暴露给工具的 Pico 成员: ['spawn_delegate']
```

### 7.2 沙箱完全外包

```
c3.path('a.py')             -> a.py            调用记录: ['a.py']
c3.path('../etc/passwd')    -> ValueError: path escapes workspace
```

`tool_context.py` 里没有任何校验逻辑，全部来自注入的 resolver。

### 7.3 可变性

```
c.depth = 99 -> 成功（frozen=False）
```

### 7.4 `depth` 无范围约束

```
ToolContext(depth=-5, max_depth=-1) 构造成功
-5 < -1 为 True -> delegate 仍会注册
```

### 7.5 `shell_env` 无缓存

```
三次调用 -> {'PATH': 'v1'} {'PATH': 'v2'} {'PATH': 'v3'}
provider 被调用 3 次
```

### 7.6 构造函数不做任何校验

六个字段全裸赋值，没有 `__post_init__`。传 `root="not a path"`、`depth="abc"`、`path_resolver=None` 都能构造出来，**错误会推迟到第一次使用**。

---

## 8. 不变量

| # | 不变量 | 验证方式 | 实测 |
|---|---|---|---|
| I1 | `path(s)` 等价于 `path_resolver(str(s))` | 传非字符串 | ✓ |
| I2 | `shell_env()` 每次调用 provider | 计数 | ✓（无缓存） |
| I3 | 工具拿不到 `session` / `model` / `approve` | 字段清单 | ✓ |
| I4 | `ToolContext` 不可变 | 赋值 | ✗ **被推翻**，可变 |
| I5 | `depth` 非负 | 传负数 | ✗ **被推翻**，任意 int 都接受 |
| I6 | 构造时校验字段类型 | 传错类型 | ✗ **被推翻**，无 `__post_init__` |
| I7 | `spawn_delegate` 是唯一能启动子事务的通道 | 字段清单 | ✓ |

---

## 9. 精读顺序建议

只有 21 行，但建议按这个顺序看：

1. **6 个字段（9–15）** —— 这是全部内容。逐个问"这个给了工具什么能力"。
2. **反向列一遍 Pico 上工具拿不到的东西**（`session` / `memory` / `model_client` / `workspace` / `approve`）。这一步比第 1 步更能说明设计意图。
3. **`path()` 和 `shell_env()`（17–21）** —— 看到它们只是一行转发，就该意识到**真正的逻辑在注入点**。
4. **跳到 `runtime.py:578-586` 看 `tool_context()` 怎么构造** —— 这一步是这个文件阅读的必要部分，否则你看不出沙箱在哪。
5. **再看 `tools.build_tool_registry`** —— 看 `partial` 怎么把 context 冻进每个 runner。

---

## 10. 已知粗糙处

按约定**只报告，不修改**。

### 10.1 不是 `frozen`

同一代码库里 `ToolExecutionResult` 用了 `frozen=True`，`ToolContext` 没有。工具函数理论上可以在运行期改 `context.depth` 绕过深度限制。当前的工具代码不会这么做，但类型上没挡住。

### 10.2 没有 `__post_init__` 校验

六个字段全裸赋值。`root="x"`、`depth="abc"`、`path_resolver=None` 都能构造成功，错误推迟到第一次使用。

### 10.3 `depth` / `max_depth` 是裸 `int`

负数会得到反直觉行为（`-5 < -1` 为 True，`delegate` 照样注册）。

### 10.4 docstring 没有指路

只有一句 "Narrow context passed from runtime into tool functions."。**读者不会知道路径安全要看 `path_resolver`、环境白名单要看 `shell_env_provider`**——而这正是这个文件最需要交代的两件事。

### 10.5 `root` 与 `path_resolver` 的隐式契约没有文档

`tools.py` 里多处 `path.relative_to(context.root)`，隐含假设是"resolver 返回的路径一定在 root 下面"。这个契约没有写在任何地方，违反了会在工具执行时才抛 `ValueError`。

### 10.6 `path()` 里的 `str()` 把类型错误变成了"文件不存在"

`context.path(123)` 会得到路径 `"123"`，然后报"不是文件"而不是"参数类型错"。错误信息不够准确。

### 10.7 `path` 方法名与 `path_resolver` 字段名过近

同一行代码里 `self.path_resolver` 和 `def path` 并存，读的时候要注意作用域。

---

## 11. 面试问答

**Q：工具是怎么拿到 runtime 的能力的？**

> 通过一个只有 **6 个字段**的窄上下文对象。

> 里面是：工作区根目录、一个"把字符串解析成安全路径"的回调、一个"返回过滤后环境变量"的回调、当前递归深度和最大深度，还有一个"启动子 agent"的回调。

> 工具函数的签名统一是 `(context, args)`——**它除了这 6 样东西，什么都拿不到**。拿不到会话历史、拿不到工作记忆、拿不到模型客户端、也拿不到审批函数。

**Q：为什么要这么窄？**

> 为了让"工具能做什么"成为一个**可以一眼看完的东西**。

> 如果我直接把整个 agent 对象传进去，审查"这个工具会不会干坏事"就得把工具代码全读一遍，还得考虑它能碰到的每一个 agent 属性。现在只要读 21 行的字段清单就够了——**6 个字段表达不了的能力，工具就是做不到**。

> 代价是扩展性差。想给工具加一个新能力，就得改这个类，而一改就是**所有工具同时获得这个能力**，没法精细控制。

**Q：路径安全在哪里做的？**

> **不在这里。**

> `ToolContext.path()` 只有一行，就是把参数交给注入进来的 resolver。这个文件里没有任何校验逻辑。

> 这是刻意的——"什么叫合法路径"是 runtime 的策略，不是工具层的策略。同一套工具函数在命令行里和评测环境里可以配不同的沙箱规则。**机制和策略分离。**

> 但这带来一个可读性问题：**光看 `tool_context.py` 完全看不出安不安全**，必须顺着注入点跳到 runtime 去看那个 resolver 的实现。我觉得这个类的文档里应该写一句"路径安全由注入的 `path_resolver` 负责，见 runtime 的对应实现"，现在没写。

**Q：为什么它是可变的对象？**

> 没有好理由。同一个代码库里另一个结果对象我用了 `frozen=True`，这个没有。

> 实际上从创建到被冻进工具函数中间没有任何地方需要改它——用 `frozen=True` 应该也能工作，而且更安全，因为工具函数理论上可以在运行期改自己的深度值来绕过限制。**这是一个该收紧但没收紧的地方。**

**Q：环境变量为什么要现算，不缓存？**

> 因为环境变量是**安全边界**。缓存意味着"运行期改了白名单不生效"，那就成了一个静默的安全漏洞。

> 算一次的成本很低，而算错的代价很高。**宁可每次算，不可算错。**

---

## 附：实测脚手架

```python
import sys
from pathlib import Path
sys.path.insert(0, r"E:\pico_agentharness\pico")
from pico.tool_context import ToolContext
import dataclasses

# 看字段清单
print([f.name for f in dataclasses.fields(ToolContext)])

# 造一个带行为的 resolver，观察沙箱是否外包
calls = []
def resolver(p):
    calls.append(p)
    if ".." in p:
        raise ValueError("path escapes workspace")
    return Path(p)

ctx = ToolContext(root=Path("."), path_resolver=resolver, shell_env_provider=lambda: {"PATH": "x"},
                  depth=0, max_depth=1, spawn_delegate=lambda a: "x")
```

**对比隔离面的做法**：把 `ToolContext` 的字段名和 `Pico` 的公开成员名取交集，看工具"能碰到"哪些 runtime 能力：

```python
import pico.runtime as rt
fields = {f.name for f in dataclasses.fields(ToolContext)}
agent_attrs = {a for a in dir(rt.Pico) if not a.startswith("__")}
print(fields & agent_attrs)      # -> {'spawn_delegate'}
```

Windows 托管 Python：`C:/Users/yxqyx/.workbuddy/binaries/python/versions/3.13.12/python.exe`
