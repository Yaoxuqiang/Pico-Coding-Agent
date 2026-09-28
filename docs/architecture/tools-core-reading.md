# tools.py 核心逻辑精读（提炼版）

> 对象：`pico/tools.py`（283 行）  
> 定位：提炼版。讲骨架、设计理由、实测证据，不逐行走读。  
> 同系列：`workspace-core-reading.md` / `tool-context-core-reading.md` / `tool-executor-core-reading.md`  
> **与 `tools-implementation-and-wiring.md` 的关系**：那份讲的是"六个工具横跨五个文件怎么接入"，是横向的链路图；这份是 `tools.py` 单个文件的纵向深挖，实测更细，不重复链路部分。  
> 所有数字均为本机实测（Windows + Python 3.13.12），脚手架见文末。

---

## 1. 一句话概括

**283 行、9 个顶层函数、4 个数据表、7 个工具实现。这个文件是 agent 的能力白名单——模型能申请哪些动作、参数怎么校验、最终怎么执行，全在这里。**

模块 docstring：

> 可以把这个文件看成 agent 的能力白名单：模型能申请哪些动作、这些动作如何做参数校验，以及最终如何执行，都是在这里定义的。

**结构上分四块**：

| 块  | 内容                                       | 作用                    |
| -- | ---------------------------------------- | --------------------- |
| 声明 | `BASE_TOOL_SPECS` / `DELEGATE_TOOL_SPEC` | 工具是什么（schema、风险等级、描述） |
| 注册 | `build_tool_registry`                    | 把工具和执行上下文绑在一起         |
| 校验 | `validate_tool`（67 行）                    | 参数合法性                 |
| 执行 | 7 个 `tool_*` 函数                          | 真正干活                  |

**复杂度集中在 `validate_tool`（67 行，占全文 24%）**，它是全文件最长的函数。其余六个工具实现加起来 107 行。

---

## 2. 行区间分布

| 行区间        | 内容                       | 行数     |
| ---------- | ------------------------ | ------ |
| 1–12       | docstring + import       | 12     |
| 14–45      | `BASE_TOOL_SPECS`（6 个工具） | 32     |
| 47–51      | `DELEGATE_TOOL_SPEC`     | 5      |
| 54–55      | `legal_tool_names`       | 2      |
| 57–65      | `TOOL_EXAMPLES`（7 条）     | 9      |
| 68–79      | `build_tool_registry`    | 12     |
| 82–83      | `tool_example`           | 2      |
| **86–152** | **`validate_tool`**      | **67** |
| 155–167    | `tool_list_files`        | 13     |
| 170–180    | `tool_read_file`         | 11     |
| 183–210    | `tool_search`            | 28     |
| 213–239    | `tool_run_shell`         | 27     |
| 242–247    | `tool_write_file`        | 6      |
| 250–264    | `tool_patch_file`        | 15     |
| 267–273    | `tool_delegate`          | 7      |
| 276–283    | `_TOOL_RUNNERS`          | 8      |

---

## 3. 必须分清的概念

### 3.1 `schema` 不是 JSON Schema，是给模型看的字符串 DSL

```python
"read_file": {"schema": {"path": "str", "start": "int=1", "end": "int=200"}, ...}
```

`"str='.'"` / `"int=20"` 这种写法看起来像有类型系统，**但全项目没有任何解析器**。实测唯一的两处使用：

```
prompt_prefix.py:29   "schema": tool["schema"]                              # 进 tool_signature（哈希用）
prompt_prefix.py:40   fields = ", ".join(f"{key}: {value}" ...)              # 拼成 "path: str, start: int=1"
```

**然后原样进 prompt 给模型看。**

所以 `"int=1"` 里的默认值语义**完全靠模型理解**——代码从不读取它，`validate_tool` 里的默认值是各自硬编码的（`args.get("start", 1)`）。**同一份默认值在两个地方各写一遍，没有任何机制保证它们一致。**

实测 `build_tool_registry` 返回的每个工具只含 4 个键：`description` / `risky` / `run` / `schema`。

### 3.2 `legal_tool_names()` 是静态全集，不知道 depth 限制

```python
def legal_tool_names():
    return set(BASE_TOOL_SPECS) | {"delegate"}
```

实测：

```
legal_tool_names():       7 个（含 delegate）
depth=0/max_depth=1 注册: 7 个（含 delegate）
depth=1/max_depth=1 注册: 6 个（delegate 消失）
```

**`legal_tool_names()` 永远返回 7 个。** 深度耗尽时 `build_tool_registry` 不注册 `delegate`，但 `legal_tool_names()` 不知道这件事。

**它用在哪？** 在 `runtime._normalize_allowed_tools` 里做参数合法性检查（用户传的 `--allow-tools` 必须在合法名单里）。所以是"配置校验"用途，不参与运行期判断——**这个不一致当前无害，但两个函数对"合法工具"的定义不同，读代码时容易误判。**

### 3.3 `TOOL_EXAMPLES` 里两种格式并存

实测：

| 工具           | 格式                                                      | 例子开头                                             |
| ------------ | ------------------------------------------------------- | ------------------------------------------------ |
| `read_file`  | `<tool>{JSON}</tool>`                                   | `<tool>{"name":"read_file","args":{...}}</tool>` |
| `write_file` | `<tool name=... path=...><content>...</content></tool>` | XML-ish                                          |
| `patch_file` | 同上                                                      | XML-ish                                          |

**为什么两种？** 看代码就能明白：`write_file` 和 `patch_file` 的参数里含**多行文本**（`content` / `old_text` / `new_text`）。把这些塞进 JSON 字符串要处理大量 `\n` 转义，模型容易写错；用 XML 式的标签包裹就能直接写多行。

**代价**：模型要在两种格式之间切换，解析端（`runtime.parse`）也要同时支持两种。这是用**解析复杂度换模型的输出准确率**。

### 3.4 路径分隔符：工具输出用平台的，记忆层用 POSIX

实测同一个嵌套文件在两处的表示：

| 位置                         | 表示                              |
| -------------------------- | ------------------------------- |
| `read_file` 返回的第一行         | `'# pkg\\mod.py'`               |
| `write_file` 的返回值          | `'wrote pkg\\new.py (1 chars)'` |
| `list_files` 的条目           | `'[F] pkg\\mod.py'`             |
| `memory.canonicalize_path` | `'pkg/mod.py'`                  |
| `memory.file_summaries` 的键 | `'pkg/mod.py'`                  |

**工具输出用 `Path.relative_to()`（Windows 上是反斜杠），记忆层用 `.as_posix()`（正斜杠）。**

模型看到的是 `pkg\mod.py`，它在下一轮把这段路径原样传给 `read_file` 也能工作（因为 `Path` 会规范化）。**但模型看不到记忆层的键，所以它无从知道两者是同一个东西。**

这是"给机器看的表示"和"给人/模型看的表示"没有统一导致的，功能上不致命，但让"路径"这个概念的表示在系统里不唯一。

### 3.5 `_TOOL_RUNNERS` 里没有 `delegate`

```python
_TOOL_RUNNERS = {
    "list_files": ..., "read_file": ..., "search": ...,
    "run_shell": ..., "write_file": ..., "patch_file": ...,
}
```

六个基础工具在这里，**`delegate` 不在这里**——它在 `build_tool_registry` 里被单独注册：

```python
if context.depth < context.max_depth:
    tools["delegate"] = {**DELEGATE_TOOL_SPEC, "run": partial(tool_delegate, context)}
```

**这个分裂是刻意的**：`_TOOL_RUNNERS` 是"永远可用的集合"，`delegate` 是"条件可用的"。如果把它放进字典，`build_tool_registry` 就得先注册再删掉，不如直接分开。

**代价**：`grep _TOOL_RUNNERS` 找不到 `delegate` 的实现，新人要找一会儿。

---

## 4. 主流程骨架

```
声明阶段（模块加载时）
  BASE_TOOL_SPECS     6 个工具 × {schema, risky, description}
  DELEGATE_TOOL_SPEC  1 个（单独放）
  TOOL_EXAMPLES       7 条示例

注册阶段（每轮构造 Pico 时）
  build_tool_registry(context)
    ├─ 6 个基础工具: {**spec, "run": partial(_TOOL_RUNNERS[name], context)}
    └─ if depth < max_depth: 再加 delegate
       ★ partial 把 context 冻进 runner，之后调用只需传 args

执行阶段（每次工具调用）
  ToolExecutor.execute(name, args)
    ├─ agent.validate_tool(name, args)   ← 本文件的 validate_tool
    ├─ ...六道闸门...
    └─ tool["run"](args)                 ← 本文件的 tool_* 函数
```

**关键：`validate_tool` 和 `tool_*` 是两套独立的校验代码**（见 §10.1）。

---

## 5. 核心函数逐个拆


### 5.1 `validate_tool`（86–152）—— 24% 的代码量

**结构：一连串 `if name == ...`，每个分支检查若干条件后 `return`。**

```python
def validate_tool(context, name, args):
    args = args or {}
    if name == "list_files":
        path = context.path(args.get("path", "."))
        if not path.is_dir():
            raise ValueError("path is not a directory")
        return
    if name == "read_file":
        path = context.path(args["path"])          # ← 直接下标
        ...
```

**备注一：`args = args or {}` 把 `None` 和 `{}` 统一了。**

模型可能不传 args（`{"name":"list_files"}`），`args.get(...)` 会崩。这一行把 `None` 变成 `{}`。

**注意它也会把 `0`、`""`、`[]` 变成 `{}`** ——因为 `or` 用的是真值判断。但 `args` 按契约是 dict，所以无害。

**备注二：三个工具用 `args["path"]` 直接下标，四个用 `args.get(...)`。** 实测后果：

```
read_file  {}  -> KeyError: 'path'
write_file {}  -> KeyError: 'path'
patch_file {}  -> KeyError: 'path'
list_files {}  -> 通过（有默认值 "."）
```

**`KeyError` 会被 `ToolExecutor` 的 `except Exception` 兜住**，转成 `invalid_arguments` 错误并附上示例。所以对模型来说，看到的是同一种错误信息——**但错误消息里会带 `KeyError: 'path'` 这种 Python 术语**，不如 `ValueError("missing path")` 清楚。

**这是"缺参数"和"参数不合法"两种情况的错误信息质量不一致**：`write_file` 缺 `content` 时给的是 `"missing content"`（手写），缺 `path` 时给的是 `KeyError: 'path'`（Python 异常）。同一个函数里两种风格。

**备注三：`search` 的校验最宽松。**

```python
if name == "search":
    pattern = str(args.get("pattern", "")).strip()
    if not pattern:
        raise ValueError("pattern must not be empty")
    context.path(args.get("path", "."))       # ← 只调用，不检查返回值
    return
```

实测对比：

| 工具           | `path="pkg"`（是目录）                | `path="不存在"`                          |
| ------------ | -------------------------------- | ------------------------------------- |
| `read_file`  | `ValueError: path is not a file` | `ValueError: path is not a file`      |
| `list_files` | 通过                               | `ValueError: path is not a directory` |
| `search`     | 通过                               | **通过**                                |

**`search` 只要求路径不逃逸，不检查它是否存在、是文件还是目录。** 那么执行时会发生什么？看 `tool_search` 的兜底实现：

```python
files = [path] if path.is_file() else [item for item in path.rglob("*") if item.is_file() and ...]
```

路径不存在时 `path.is_file()` 是 False，`rglob` 返回空 → `matches` 为空 → 返回 `"(no matches)"`。

**所以搜错目录会得到 `(no matches)`，而这个输出和"目录存在但没有匹配"完全一样。** 模型无法区分"这里真的没有"和"你找错地方了"——这是个会误导 agent 决策的静默失败。

**备注四：`run_shell` 的 timeout 校验是唯一有范围约束的。**

```python
timeout = int(args.get("timeout", 20))
if timeout < 1 or timeout > 120:
    raise ValueError("timeout must be in [1, 120]")
```

其余所有参数都没有上界（`read_file` 的 `end` 可以传 9999，实测不报错，只是静默忽略超出部分）。


### 5.2 `tool_read_file`（170–180）

```python
lines = path.read_text(encoding="utf-8", errors="replace").splitlines()
body = "\n".join(f"{number:>4}: {line}" for number, line in enumerate(lines[start - 1:end], start=start))
return f"# {path.relative_to(context.root)}\n{body}"
```

**备注一：`errors="replace"` 是必须的。**

实测读一个含二进制字节的文件：

```
'# bin.dat\n   1: \x00\ufffd\ufffd bad bytes'
```

`\ufffd` 是替换字符。如果不加 `errors="replace"`，读二进制文件会直接抛 `UnicodeDecodeError`，**而模型经常会读错文件**（比如 `list_files` 列出了 `bin.dat`，模型试着读一下）。**这个参数把"崩溃"变成了"读到乱码"**，后者是可恢复的。

**备注二：行号格式 `{number:>4}` 是给模型看的坐标。**

输出长这样：

```
# a.py
   1: def binary_search(nums, target):
   2:     return -1
```

**这个格式让模型能引用具体行号**（"第 2 行有问题"），也让 `patch_file` 的 `old_text` 更容易定位。

**备注三：第一行 `# 路径` 不是给人看的。**

它存在的原因是 `memory.summarize_read_result`（`memory.py:511`）：

```python
if lines[0].startswith("# "):
    lines = lines[1:]
```

**记忆层靠 `# ` 前缀精确剥离这一行**，否则摘要里会把路径当成正文第一条拼进去。

**这是一个跨模块的隐式契约**：改 `read_file` 的返回格式（比如把 `# ` 换成 `// `）会让记忆摘要静默变脏，**而且不会报错**。

**备注四：超范围的行号被静默吞掉。**

实测：

```
文件 10 行，请求 start=5, end=9999 -> 返回 7 行（含标题行）
文件 10 行，请求 start=500, end=600 -> 返回 '# big.py\n'  （只有标题行）
```

**`start` 超出文件长度时，模型得到的是一个只有路径头的两行文本。** 它无法区分"文件是空的"和"我要的行号超范围了"。


### 5.3 `tool_search`（183–210）—— 两条实现路径

```python
if shutil.which("rg"):
    result = subprocess.run(["rg", "-n", "--smart-case", "--max-count", "200", pattern, str(path)],
                            cwd=context.root, capture_output=True, text=True)
    return result.stdout.strip() or result.stderr.strip() or "(no matches)"
# 兜底：纯 Python 逐文件逐行
```

**备注一：本机没有 rg，走的是兜底路径。**

实测 `shutil.which("rg")` → `False`。所以在这台机器上**永远是 Python 兜底**在跑。

**备注二：两条路径的"200"语义不同。**

|                                           | rg 版                      | Python 兜底版                                 |
| ----------------------------------------- | ------------------------- | ------------------------------------------ |
| `--max-count 200` / `len(matches) >= 200` | **每个文件** 200 条            | **总共** 200 条                               |
| `.pico` 排除                                | **不排除**（没有 `--glob`）      | 排除                                         |
| 大小写                                       | `--smart-case`（全小写则忽略大小写） | `pattern.lower() in line.lower()`（永远忽略大小写） |
| 返回中路径                                     | rg 自己格式化的                 | `relative_to(context.root)`                |

**同一个工具，装上 rg 和不装 rg，结果集不一样。** 这不是 bug（两条路径都"能用"），但**评测结果不可复现**——换一台机器可能得到不同的召回。

**备注三：`or` 链让 stderr 可能被当成结果返回。**

```python
return result.stdout.strip() or result.stderr.strip() or "(no matches)"
```

如果 rg 报错（比如 pattern 是非法正则），stdout 为空，**stderr 的错误信息会被当成"搜索结果"返回给模型**。模型看到一段 rg 的错误输出，可能误以为那是匹配结果。

**备注四：`text=True` 不带 `encoding` —— 和 `run_shell` 有同样的问题。**

见 §10.2。


### 5.4 `tool_run_shell`（213–239）

```python
result = subprocess.run(command, cwd=context.root, shell=True, capture_output=True,
                        text=True, timeout=timeout, env=context.shell_env())
return textwrap.dedent(f"""\
    exit_code: {result.returncode}
    stdout:
    {result.stdout.strip() or "(empty)"}
    stderr:
    {result.stderr.strip() or "(empty)"}
    """).strip()
```

**备注一：输出格式是 `tool_executor` 的解析契约。**

`tool_executor.py:122` 用正则从这段文本里抠退出码：

```python
match = re.search(r"exit_code:\s*(-?\d+)", content)
```

**所以这段 `textwrap.dedent` 的模板不是一个普通的展示格式，是接口。** 改一个字都会静默改变状态判定。

**备注二：`env=context.shell_env()` 只给白名单环境变量。**

实测通过白名单传入的环境里，`API_KEY` / `TOKEN` 这类根本不存在——**它们在子进程里是"未设置"，不是"空字符串"**。

**备注三：它有 Windows 上的编码崩溃（实测复现）。**

`text=True` 不带 `encoding` / `errors`，当命令输出非 UTF-8 字节时：

```
输出非法 UTF-8 字节 -> AttributeError: 'NoneType' object has no attribute 'strip'
```

根因链：

```
subprocess 的 reader 线程按 locale 编码解码
  → 遇到非法字节抛 UnicodeDecodeError（在后台线程里）
  → 线程死掉，result.stdout / result.stderr 变成 None
  → f-string 里 result.stderr.strip() → AttributeError
```

**报错信息完全没有提到编码**，很难定位。详见 §10.2。

### 5.5 `tool_write_file`（242–247）

```python
path = context.path(args["path"])
content = str(args["content"])
path.parent.mkdir(parents=True, exist_ok=True)
path.write_text(content, encoding="utf-8")
return f"wrote {path.relative_to(context.root)} ({len(content)} chars)"
```

**备注一：`mkdir(parents=True, exist_ok=True)` 让模型不必先建目录。**

模型可以直接 `write_file("src/deep/nested/x.py", ...)`，父目录自动创建。**这减少了模型的步骤数**——否则它得先调 `run_shell mkdir`。

**备注二：返回值带字符数是给模型自查用的。**

`(9 chars)` 让模型能确认"我写的长度和我想写的一致"。如果模型输出被截断，这里能看出来。

**备注三：覆盖已有文件不做任何提示。**

实测：

```
write_file('exists.py')  -> 通过（静默覆盖）
write_file('adir')       -> ValueError: path is a directory
```

**覆盖是有风险的**：如果模型误判了文件名，旧内容直接没了。而且 —— 见 §10.6 —— **覆盖会让记忆层和 checkpoint 里存的 freshness 失效**，但那个失效是"下一次校验时才发现"的。


### 5.6 `tool_patch_file`（250–264）

```python
text = path.read_text(encoding="utf-8")
count = text.count(old_text)
if count != 1:
    raise ValueError(f"old_text must occur exactly once, found {count}")
path.write_text(text.replace(old_text, str(args["new_text"]), 1), encoding="utf-8")
```

**备注一：`count != 1` 直接拒绝，两种失败不区分处理。**

实测：

```
old_text='return -1'（文件里 1 次）  -> 成功
old_text='return -1'（文件里 2 次）  -> ValueError: found 2
old_text='不存在'                    -> ValueError: found 0
```

代码注释（130–131 行）解释了理由：

> patch_file 故意做得很严格：old_text 必须精确命中且只能出现一次，这样修改行为才是确定的，失败原因也更容易解释。

**`count == 0`** 说明模型记错了内容，**`count >= 2`** 说明目标不唯一。两种都不猜——从这个错误信息里，模型能看出是哪种情况（`found 0` vs `found 2`）。

**备注二：`replace(..., 1)` 里的 1 是冗余的。**

既然已经确认 `count == 1`，`replace` 不带 count 也只会替换一处。写上 `1` 是**防御性冗余**——万一将来放宽了 `count` 的检查，这里仍然只改第一处。

**备注三：没有 `errors="replace"`。**

`patch_file` 用 `read_text(encoding="utf-8")` **不带 `errors`**，读二进制文件会直接抛 `UnicodeDecodeError`。而 `read_file` 带了。**同一个文件工具族里两种处理方式**——`patch_file` 的行为更严格（读不了就报错，而不是改坏二进制文件），这个取舍合理，但两个函数的差异没有注释说明。

### 5.7 `tool_delegate`（267–273）

```python
def tool_delegate(context, args):
    if context.depth >= context.max_depth:
        raise ValueError("delegate depth exceeded")
    task = str(args.get("task", "")).strip()
    if not task:
        raise ValueError("task must not be empty")
    return context.spawn_delegate(args)
```

**备注：深度检查写了两遍。**

`build_tool_registry` 里已经用 `if context.depth < context.max_depth` 决定**要不要注册这个工具**，这里又检查一遍。

**为什么重复？** 因为 `ToolContext` 是**可变对象**（见 `tool-context-core-reading.md` §10.1）。理论上注册之后 depth 可能被改。"注册时不给" 和 "执行时再拦" 是两道不同层级的保护——**深度限制不该只靠"不暴露工具"这一层**。

---

## 6. 设计理由

### 6.1 为什么工具是显式注册而不是动态发现？

`build_tool_registry` 的注释直接说了：

> 工具不是动态发现的，而是显式注册的。这样模型看到的是一个**有边界、可审计**的动作集合。

**好处**：模型能做的事是一个静态可枚举的集合。想知道"这个 agent 能干什么"，读 `BASE_TOOL_SPECS` 就够了，不需要扫描文件系统或插件目录。

**代价**：加一个新工具要改三个地方（`BASE_TOOL_SPECS`、`_TOOL_RUNNERS`、`TOOL_EXAMPLES`）+ `validate_tool` 里的分支。**漏掉 `_TOOL_RUNNERS` 会在注册时 `KeyError`**（因为字典推导式直接索引），这个失败是响亮的，不是静默的——**这是好的失败模式**。

**漏掉 `TOOL_EXAMPLES`** 则会让报错信息里没有示例（`tool_example` 返回 `""`，`ToolExecutor` 不加 `example:` 行），**静默降级**。

### 6.2 为什么校验和执行分成两套？

**好处**：`ToolExecutor` 可以在**不执行任何副作用**的前提下先验证参数。这在七道闸门里的位置很关键——**校验排在审批和快照之前**，意味着一个非法参数不会触发审批弹窗、不会拍工作区快照。

**代价**：**两套代码会漂移**。实测 `patch_file` 的"恰好一次"检查在两处都有（`tools.py:141-143` 和 `260-262`），措辞还不完全一样。

**更麻烦的是 `validate_tool` 会读文件**：

```python
text = path.read_text(encoding="utf-8")     # 141 行
count = text.count(old_text)
```

所以 `patch_file` 的校验阶段**已经做了一次完整读文件**，执行阶段**又读一次**。对一个大文件，这是双倍 IO 只为了数一个出现次数。

### 6.3 为什么 `search` 优先用 `rg`？

代码注释（190 行）：

> 优先用 rg，因为搜索会非常频繁，搜索延迟会直接影响 agent 控制循环。

**好处**：rg 是编译过的、多线程的、遵守 `.gitignore` 的。在一个大仓库里它和纯 Python 逐文件读的差距是数量级的。

**代价**：**两条路径行为不一致**（§5.3），导致"装没装 rg"影响 agent 的行为。而且本机实测**没有 rg**，所以正在跑的永远是慢的那条。

### 6.4 为什么 `patch_file` 要 `count != 1` 拒绝？

**好处**：修改行为是确定的。要么改，要么明确告诉模型为什么不能改。

**代价**：模型想把文件里所有 `return -1` 都换掉时做不到——它得一次改一处，或者改用 `write_file` 重写整个文件。

**取舍判断**：对的。**"错误地改了不该改的地方"比"改不了"代价大得多**，尤其是在一个会自动执行的 agent 里。

### 6.5 为什么 `write_file` 会静默覆盖？

**好处**：简单。模型要重写一个文件就是一次调用。

**代价**：误判文件名时旧内容直接丢失，**没有"文件已存在，是否覆盖"的确认**。

**取舍判断**：`write_file` 标记为 `risky: True`，所以会走审批流程（除非 `approval_policy="never"`）。**风险被转移到了审批环节**——这是合理的，因为"这个文件该不该被覆盖"是用户能判断的，代码判断不了。

---

## 7. 实测验证

### 7.1 `validate_tool` 的缺参数行为

```
read_file   {}  -> KeyError: 'path'
write_file  {}  -> KeyError: 'path'
patch_file  {}  -> KeyError: 'path'
list_files  {}  -> 通过（有默认值 "."）
```

### 7.2 `validate_tool` 对未知工具名静默通过

```
validate_tool('no_such_tool', {}) -> 返回 None，什么都没检查
```

（实践中到不了这一步，因为 `ToolExecutor` 先查 `agent.tools.get(name)`。）

### 7.3 `legal_tool_names` 与实际注册集不一致

```
legal_tool_names():       7 个
depth=0/max_depth=1 注册: 7 个
depth=1/max_depth=1 注册: 6 个   ← delegate 消失
```

### 7.4 `TOOL_EXAMPLES` 两种格式

| 工具           | 格式                                              |
| ------------ | ----------------------------------------------- |
| `read_file`  | JSON（`<tool>{"name":...}</tool>`）               |
| `write_file` | XML-ish（`<tool name=... path=...><content>...`） |
| `patch_file` | XML-ish                                         |

### 7.5 `read_file` 的容错与边界

```
读二进制文件      -> '# bin.dat\n   1: \x00\ufffd\ufffd bad bytes'（不崩）
正常文件          -> '# a.py' + 带行号的正文
end 超出文件行数   -> 静默忽略
start 超出文件行数 -> 只返回 '# big.py\n'
```

### 7.6 `list_files` 的排序与过滤

```
[D] sub
[F] a.py
[F] b.py
[F] bin.dat
```

`.git` 被隐藏；目录排在文件前（`key=(item.is_file(), item.name.lower())`——`is_file()` 对目录返回 False，排前面）。

### 7.7 `search` 在无 rg 机器上走兜底

```
本机有 rg: False
search('return') -> ['a.py:2:    return -1', 'b.py:2:    return -1', 'b.py:3:    return -1']
```

### 7.8 `patch_file` 的 `count != 1` 拒绝

```
文件里 1 次 -> 成功
文件里 2 次 -> ValueError: old_text must occur exactly once, found 2
不存在      -> ValueError: old_text must occur exactly once, found 0
```

### 7.9 `write_file` 建父目录并返回字符数

```
write_file('deep/nest/x.py', 'print(1)\n') -> 'wrote deep\nest\x.py (9 chars)'
```

### 7.10 路径分隔符不一致

| 位置              | 表示                              |
| --------------- | ------------------------------- |
| `read_file` 首行  | `'# pkg\\mod.py'`               |
| `write_file` 返回 | `'wrote pkg\\new.py (1 chars)'` |
| `list_files`    | `'[F] pkg\\mod.py'`             |
| `patch_file` 返回 | `'patched pkg\\mod.py'`         |
| `memory` 的键     | `'pkg/mod.py'`                  |

### 7.11 **`run_shell` 在非 UTF-8 输出下崩溃**

```
纯 ASCII 命令        -> OK 首行 'exit_code: 0'
命令失败             -> OK 首行 'exit_code: 3'
输出非法 UTF-8 字节   -> AttributeError: 'NoneType' object has no attribute 'strip'
```

直接复现根因：

```
subprocess.run(..., text=True)                         -> stdout=None, 后台线程抛 UnicodeDecodeError
subprocess.run(..., text=True, errors="replace")       -> stdout='������\n' ✓
```

本机 `locale.getpreferredencoding(False)` = `utf-8`，所以是"命令输出的是 GBK 字节，按 UTF-8 解码失败"。

### 7.12 `search` 的路径校验最宽松

```
search(path='pkg')       -> 通过
search(path='不存在的目录') -> 通过
read_file(path='pkg')    -> ValueError: path is not a file
```

### 7.13 `write_file` 的覆盖行为

```
write_file('exists.py')   -> 通过（静默覆盖）
write_file('adir')        -> ValueError: path is a directory
write_file('new/deep/x.py') -> 通过
```

### 7.14 `schema` DSL 无解析器

```
build_tool_registry 返回的键: ['description', 'risky', 'run', 'schema']
prompt_prefix 里对 schema 的唯一处理: ", ".join(f"{key}: {value}" ...)
```

---

## 8. 不变量

| #   | 不变量                                   | 验证方式                 | 实测                         |
| --- | ------------------------------------- | -------------------- | -------------------------- |
| I1  | `build_tool_registry` 返回的每个工具都含 `run` | 检查键集                 | ✓（缺 runner 会 KeyError）     |
| I2  | 深度耗尽时 `delegate` 不注册                  | `depth >= max_depth` | ✓                          |
| I3  | `patch_file` 在 `count != 1` 时拒绝       | 三种输入                 | ✓                          |
| I4  | `read_file` 不因二进制内容崩溃                 | 读 `.dat`             | ✓（`errors="replace"`）      |
| I5  | `write_file` 自动创建父目录                  | 写深层路径                | ✓                          |
| I6  | `list_files` 隐藏 `IGNORED_PATH_NAMES`  | 建 `.git` 目录          | ✓                          |
| I7  | `validate_tool` 检查路径是否存在              | `search(path='不存在')` | ✗ **被推翻**，只有 `search` 不检查  |
| I8  | `run_shell` 对任意命令都返回结果                | 输出非 UTF-8            | ✗ **被推翻**，`AttributeError` |
| I9  | `legal_tool_names()` 等于实际可用集          | 对比注册结果               | ✗ **被推翻**，depth 耗尽时不符      |
| I10 | 工具输出与记忆层用同一种路径表示                      | 对比                   | ✗ **被推翻**，`\` vs `/`       |
| I11 | 缺参数报 `ValueError`                     | 四种工具传 `{}`           | ✗ **被推翻**，三个抛 `KeyError`   |
| I12 | `read_file` 的 `end` 有上界               | 传 9999               | ✗ **被推翻**，静默忽略             |

---

## 9. 精读顺序建议

1. **`BASE_TOOL_SPECS`（14–45）** —— 先看这 6 条声明。注意 `risky` 字段的分布：三个读工具 `False`，三个写工具 `True`。这个布尔值决定了后面所有安全流程。
2. **`build_tool_registry`（68–79）** —— 12 行，看 `partial` 怎么把 context 冻进去，和 `delegate` 的条件注册。
3. **`validate_tool`（86–152）** —— 全文件最长。逐分支看，重点注意**哪些检查了存在性、哪些没有**（`search` 是唯一的例外）。
4. **七个 `tool_*`** —— 按这个顺序读，难度递增：`tool_write_file`（6 行）→ `tool_read_file`（11）→ `tool_list_files`（13）→ `tool_patch_file`（15）→ `tool_run_shell`（27）→ `tool_search`（28）→ `tool_delegate`（7）。
5. **`TOOL_EXAMPLES`（57–65）** —— 最后看。这时你已经知道哪些工具有多行参数，就能理解为什么有两个用 XML 格式。
6. **跳到调用方** —— `tool_executor.execute`（看校验被放在第几道闸门）、`prompt_prefix.build_prompt_prefix`（看 `schema` 怎么进 prompt）。

---

## 10. 已知粗糙处

按约定**只报告，不修改**。

### 10.1 `validate_tool` 与 `tool_*` 是两套重复校验

`patch_file` 的"恰好出现一次"检查在两处都有：

- `validate_tool`：`tools.py:140-143`（`ValueError(f"old_text must occur exactly once, found {count}")`）
- `tool_patch_file`：`tools.py:259-262`（同样的话，**少了 "in this file"**）

**两处措辞不一致**，所以失败信息取决于哪一层先触发。而且校验层**已经完整读了一次文件**，执行层又读一次。

其余工具也都有类似的双份检查（`is_file` / `is_dir` / 参数非空）。这是"防御性重复"，代价是漂移风险 + 重复 IO。

### 10.2 **`run_shell` / `search` 在非 UTF-8 输出下崩溃**（本机可复现）

`subprocess.run(..., text=True)` 不带 `encoding` 和 `errors`。当子进程输出非 UTF-8 字节时：

1. subprocess 的后台 reader 线程抛 `UnicodeDecodeError`
2. 线程终止，`result.stdout` / `result.stderr` 变成 **`None`**
3. 调用处 `result.stderr.strip()` → **`AttributeError: 'NoneType' object has no attribute 'strip'`**

实测：

```
cmd /c "echo ±±±"  -> AttributeError: 'NoneType' object has no attribute 'strip'
```

**报错信息完全不提编码**，定位成本很高。在中文 Windows 上这是个高频场景（`cmd` 的内置命令默认输出 GBK）。

**`tool_search` 的 rg 分支、`workspace.py` 的 git 调用也是同样的写法**，全都有这个隐患。

修法：`text=True, encoding="utf-8", errors="replace"`。

### 10.3 `search` 不校验路径存在性，静默返回 `(no matches)`

`validate_tool` 的 `search` 分支只做两件事：pattern 非空、`context.path()` 不抛异常。**不检查存在性、不检查是文件还是目录。**

搜一个不存在的目录会得到 `(no matches)`，**和"目录存在但没匹配"无法区分**。这会误导模型以为"这里确实没有"。

### 10.4 `legal_tool_names()` 与实际可用集不一致

它是静态全集（永远 7 个），不感知 `depth < max_depth` 的条件注册。当前只用于配置校验，无害，但两个函数对"合法工具"的定义不同。

### 10.5 `schema` DSL 没有解析器，默认值写了两遍

`"int=1"` 里的 1 和 `validate_tool` 里 `args.get("start", 1)` 的 1 是同一个值，**在两处各写一遍**。改一处不改另一处会导致"prompt 里说默认 1，实际按 200 处理"。

### 10.6 `write_file` 静默覆盖，且会让 freshness 失效

覆盖不做提示（风险转移到审批环节）。另外覆盖会让记忆层和 checkpoint 里的 `freshness` 变旧——**但那个失效要等下一次校验才发现**，中间这段时间模型可能拿着过期摘要推理。

### 10.7 路径分隔符在系统里不统一

工具输出用 `relative_to()`（Windows 反斜杠），记忆层用 `.as_posix()`（正斜杠）。模型看到的和系统内部存的不是同一种表示。

### 10.8 `search` 的 `or` 链会把 stderr 当成结果

```python
return result.stdout.strip() or result.stderr.strip() or "(no matches)"
```

rg 报错时，**错误信息会作为"搜索结果"返回给模型**。

### 10.9 `search` 两条实现路径的行为不一致

`--max-count 200`（每文件）vs `len(matches) >= 200`（总计）；是否排除 `.pico`；大小写处理。**装没装 rg 影响结果集**，评测不可复现。

### 10.10 `read_file` 超范围行号静默忽略

`start` 超出文件长度时只返回 `'# 路径\n'`，模型无法区分"文件空"和"行号超范围"。`end` 也没有上界。

### 10.11 缺参数的报错风格不一致

`write_file` 缺 `content` → `"missing content"`（手写）；缺 `path` → `KeyError: 'path'`（Python 异常）。同一个函数两种风格。

### 10.12 `patch_file` 没有 `errors="replace"`

和 `read_file` 不一致——读二进制文件时 `patch_file` 直接抛异常。行为更安全，但差异没有注释说明。

### 10.13 `_TOOL_RUNNERS` 里没有 `delegate`

`grep _TOOL_RUNNERS` 找不到 `delegate` 的实现，它在 `build_tool_registry` 里单独注册。

---


## 11. 面试问答

**Q：工具是怎么定义的？**

> 三张表 + 一个注册函数。

> 第一张是"工具规格"，每个工具记三件事：**参数长什么样**、**是不是高风险操作**、**一句话描述**。第二张是参数示例，第三张是"工具名到执行函数"的映射。

> 注册的时候把上下文用 `partial` 冻进每个执行函数里，这样工具函数的签名就统一成"接收参数、返回字符串"。

**关键设计是：工具是显式注册的，不是动态发现的。** 这样"这个 agent 能干什么"是一个静态可枚举的集合——读一遍那张表就知道，不用扫文件系统。

**Q：为什么要分"高风险"？**

> 三个读操作不标，三个写操作标了。

> 这个布尔值决定了后面一整套流程：**要不要弹审批、要不要在前后拍工作区快照、错误时的默认风险等级**。



> 有意思的是 `delegate` 不标高风险——因为子 agent 被强制成只读加不弹审批，它本身就是安全的。

**Q：参数校验为什么要单独一层？**

> 因为**校验必须发生在任何副作用之前**。

> 我在执行器里把它排在第一道闸门——在审批弹窗和快照之前。这样模型传了一个非法参数，用户不会被无谓地打扰，也不会白拍一次快照。

**但有个代价我要说清楚**：校验层和执行层的检查是**两套代码**。像"改文件时原文必须恰好出现一次"这个检查，两边都写了，而且措辞还不一样。这种重复迟早会漂移。

**而且更浪费的是**，"恰好出现一次"这个检查需要**把文件完整读一遍**来数出现次数——校验时读一次，执行时又读一次。

**Q：文件工具怎么容错？**

> 有两个地方是刻意的。

> `read_file` 读文件时用了**替换模式**——遇到不合法的字节不报错，换成替换字符。因为模型经常读错文件，比如看见一个 `.dat` 也去读一下。这样它得到的是乱码，而不是一个异常把整轮对话打断。

> 但 `patch_file` 反过来，**不**用替换模式。因为它要改动文件内容，读到乱码还去替换就是灾难。**一个宽容一个严格，是按"读会不会造成伤害"分的。**

**Q：搜索为什么优先用 rg？**

> 因为搜索太频繁了，搜索的延迟直接拖慢整个控制循环。

> rg 是编译过的、多线程的、会遵守忽略规则，在大仓库里和纯 Python 逐文件读不是一个量级。

**但这里有个我做得不好的地方**：两条实现路径的行为不一致。rg 的 `--max-count 200` 是**每个文件** 200 条，我的兜底实现是**总共** 200 条；rg 那条不排除内部目录，兜底那条排除了。**结果是"这台机器装没装 rg"会改变 agent 的行为**，评测也没法复现。

**Q：`run_shell` 有什么坑？**

> 有一个我在中文 Windows 上实测出来的崩溃。

> 我调 subprocess 的时候开了文本模式但**没指定编码**。如果命令输出的是本地编码的字节（比如 `cmd` 内置命令输出 GBK），解码会在 subprocess 的后台线程里失败。线程一死，`stdout` 就变成了 `None`——然后我代码里紧接着调 `result.stderr.strip()`，直接抛 **`AttributeError: 'NoneType' object has no attribute 'strip'`**。

> **报错信息里完全没提编码**，只说 `None` 没有 `strip`，排查成本很高。

> 修法很简单，加 `encoding="utf-8", errors="replace"` 就行。**而且 `search` 的 rg 分支和另一个模块里调 git 的地方都是同样的写法**，都有这个隐患。

**Q：模式匹配的修改为什么要"恰好一次"？**

> 因为要保证修改是确定的。

> 出现零次说明模型记错了内容，出现两次说明目标不唯一——**两种都不猜**。而且我把出现次数写在错误信息里（`found 0` / `found 2`），模型能直接看出是哪一种情况。

> 代价是**没法一次替换多处**。但在这个场景里，"改错地方"比"改不了"代价大得多。

---


## 附：实测脚手架

`tools.py` 有相对导入，按包加载：

```python
import sys, tempfile
from pathlib import Path
sys.path.insert(0, r"E:\pico_agentharness\pico")
from pico import tools as tl
from pico.tool_context import ToolContext

tmp = Path(tempfile.mkdtemp()); root = tmp / "repo"; root.mkdir()

# 最小可用 context：resolver 直接拼路径即可（想测沙箱就自己加校验）
ctx = ToolContext(
    root=root,
    path_resolver=lambda p: (root / p).resolve(),
    shell_env_provider=lambda: {"PATH": __import__("os").environ.get("PATH", ""), "SYSTEMROOT": r"C:\Windows"},
    depth=0, max_depth=1,
    spawn_delegate=lambda a: "[delegate]",
)
```

**三个注意点**：

1. **`shell_env_provider` 要真的给 `PATH`**，否则 `cmd` / `python` 都找不到。Windows 上还要给 `SYSTEMROOT`，不然 `cmd` 起不来。
2. **不要用 heredoc 写含 `\n`、`\s` 的探针**——Git Bash 的 MSYS 路径转换会把反斜杠改成斜杠（实测把 `\n` 变成了 `/n`），导致正则失效。**写成临时 `.py` 文件再跑。**
3. **测 `run_shell` 的非 UTF-8 崩溃**：`cmd /c "echo ±±±"`，注意会往 stderr 打一大段后台线程的 traceback，但那不是真正的错误——真正的错误是主线程的 `AttributeError`。

Windows 托管 Python：`C:/Users/yxqyx/.workbuddy/binaries/python/versions/3.13.12/python.exe`
