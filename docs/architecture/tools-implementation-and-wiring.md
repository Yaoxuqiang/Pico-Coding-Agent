# 六个工具的源码实现与接入链路

> 涉及文件：
> `pico/tools.py`（283 行，工具定义 + 校验 + 实现）
> `pico/tool_context.py`（21 行，工具能看到的"窄接口"）
> `pico/tool_executor.py`（160 行，执行闸门）
> `pico/runtime.py`（装配 + 解析 + 审批 + 记忆沉淀 + 路径解析）
> `pico/prompt_prefix.py`（把 schema 渲染成文字给模型看）
> `pico/security.py`（run_shell 的环境变量白名单）

---

## 0. 全景：一次工具调用的完整链路

```
① 声明   tools.py  BASE_TOOL_SPECS         ← 工具是"显式注册"，不是动态发现
② 注册   tools.py  build_tool_registry()   ← 把 ToolContext 用 partial 绑进去
③ 暴露   prompt_prefix.py                  ← schema 渲染成文字，写进 prefix
         ─────────────────────────────────
④ 模型返回  <tool>...</tool> 或 XML 风格
         ─────────────────────────────────
⑤ 解析   runtime.py  parse()               ← 形态识别 + JSON 校验
⑥ 闸门   tool_executor.py  execute()       ← 七道关卡，全在这里
          └ 允许清单 → 工具存在 → 参数校验 → 重复拦截 → 审批 → 快照 → 执行
⑦ 执行   tools.py  tool_*(context, args)   ← 真正的 IO
⑧ 记账   clip + diff + 状态判定 + 记忆沉淀 + trace/report
```

**一句话：模型只负责"申请"，平台负责"能不能做、怎么做、做完记什么"。**

---

# 第一部分：六个工具的源码

## 1. 只读组（`risky: False`，不需要审批）

### 1.1 `list_files` — 看目录

```python
def tool_list_files(context, args):
    path = context.path(args.get("path", "."))
    if not path.is_dir():
        raise ValueError("path is not a directory")
    entries = [
        item for item in sorted(path.iterdir(), key=lambda item: (item.is_file(), item.name.lower()))
        if item.name not in IGNORED_PATH_NAMES
    ]
    lines = []
    for entry in entries[:200]:
        kind = "[D]" if entry.is_dir() else "[F]"
        lines.append(f"{kind} {entry.relative_to(context.root)}")
    return "\n".join(lines) or "(empty)"
```

**备注 1：`sorted(key=(item.is_file(), item.name.lower()))` —— 目录排前面。**
`is_file()` 对目录是 `False`（=0），对文件是 `True`（=1），`False < True`，所以目录天然排在文件前面；同类再按名字小写排序。一行代码同时完成"目录优先 + 大小写不敏感"，不用自定义比较器。

**备注 2：只列一层，不递归。**
`path.iterdir()` 只拿直接子项。深目录要靠 `search` 补。这是刻意的——递归列目录在稍大的仓库里就能把上下文炸掉。

**备注 3：两处过滤重复了。**
列表推导里过滤了一次 `IGNORED_PATH_NAMES`，但 `iterdir()` 根本不返回 `.git` 这种子目录**内部**的东西 —— 它只返回顶层条目。所以过滤是为了防"工作区根目录恰好叫 `.git`"这类边界，实际生效场景很少。

**备注 4：硬上限 200 条 + `"(empty)"` 兜底。**
`entries[:200]` 防止目录爆量；`or "(empty)"` 保证空目录时返回非空字符串（模型拿到空串会困惑）。

**备注 5：用 `[D]` / `[F]` 而不是 emoji。**
agent 的 prompt 是要喂给模型的，纯 ASCII 标记更省 token、也不会有编码问题。

---

### 1.2 `read_file` — 读文件

```python
def tool_read_file(context, args):
    path = context.path(args["path"])
    if not path.is_file():
        raise ValueError("path is not a file")
    start = int(args.get("start", 1))
    end = int(args.get("end", 200))
    if start < 1 or end < start:
        raise ValueError("invalid line range")
    lines = path.read_text(encoding="utf-8", errors="replace").splitlines()
    body = "\n".join(f"{number:>4}: {line}" for number, line in enumerate(lines[start - 1:end], start=start))
    return f"# {path.relative_to(context.root)}\n{body}"
```

**备注 1：默认只读 200 行，强制按行号区间读。**
不给"读整个文件"这个选项。这是防上下文爆炸的第一道闸——模型必须自己说明要哪一段。

**备注 2：`errors="replace"` 是必须的。**
读到二进制文件（图片、`.pt` 权重）时不会抛异常，坏字节替换成 `\ufffd`。如果没有这个参数，agent 一次误读就会让整轮崩掉。

**备注 3：`enumerate(..., start=start)` + `{number:>4}` —— 输出自带行号。**
右对齐宽度 4（足够覆盖 9999 行），这样模型拿到内容后可以精确说出"第 47 行有问题"，后续 `patch_file` 的 `old_text` 才有依据。

**备注 4：返回值第一行是 `# 相对路径`，这和记忆层有耦合。**
`memory.py` 的 `summarize_read_result()` 里有一句：

```python
if lines[0].startswith("# "):
    lines = lines[1:]      # 丢掉 "# path" 这一行，只留正文
```

也就是说，**这个 `# path` 头不是给人看的，是给记忆层做摘要时精确剥离用的**。如果哪天把返回格式改成 `path\n---\nbody`，这段摘要逻辑就会静默失效（把路径当成正文第一行拼进摘要）。这是跨模块的隐式契约，属于"改之前必须搜一遍"的地方。

---

### 1.3 `search` — 搜字符串

```python
def tool_search(context, args):
    pattern = str(args.get("pattern", "")).strip()
    if not pattern:
        raise ValueError("pattern must not be empty")
    path = context.path(args.get("path", "."))

    if shutil.which("rg"):
        # 优先用 rg，因为搜索会非常频繁，搜索延迟会直接影响 agent 控制循环。
        result = subprocess.run(
            ["rg", "-n", "--smart-case", "--max-count", "200", pattern, str(path)],
            cwd=context.root,
            capture_output=True,
            text=True,
        )
        return result.stdout.strip() or result.stderr.strip() or "(no matches)"

    matches = []
    files = [path] if path.is_file() else [
        item for item in path.rglob("*")
        if item.is_file() and not any(part in IGNORED_PATH_NAMES for part in item.relative_to(context.root).parts)
    ]
    for file_path in files:
        for number, line in enumerate(file_path.read_text(encoding="utf-8", errors="replace").splitlines(), start=1):
            if pattern.lower() in line.lower():
                matches.append(f"{file_path.relative_to(context.root)}:{number}:{line}")
                if len(matches) >= 200:
                    return "\n".join(matches)
    return "\n".join(matches) or "(no matches)"
```

**备注 1：双实现策略 —— 有 `rg` 用 `rg`，没有就纯 Python 兜底。**
`shutil.which("rg")` 探测一次。理由写在注释里：搜索在 agent 循环里频率最高，延迟直接拖慢整个控制循环。用 `rg` 是几十毫秒，纯 Python 遍历可能是几秒。

**备注 2：`--max-count 200` 是"每个文件最多 200 条"，不是"总共 200 条"。**
这是个容易看错的坑。真正兜总上限的是外层 `ToolExecutor` 里的 `clip(..., 4000)`。

**备注 3：兜底实现的 `--max-count` 语义不同。**
Python 版本里 `if len(matches) >= 200` 是**总的**匹配数上限。所以两条路径的输出长度特征不一样：`rg` 路径在超大仓库里可能返回很长（多文件 × 200 行），靠 executor 的 4000 字符兜。**这是两条路径行为不一致的地方**，属于已知的粗糙点。

**备注 4：兜底路径没有单行长度限制。**
如果匹配到一行压缩过的 `min.js`，`matches.append(f"...{line}")` 会把整行塞进去。同样靠 executor 层的 4000 字符兜住。

**备注 5：兜底路径的忽略规则比 `rg` 严格。**
Python 版显式跳过 `IGNORED_PATH_NAMES`（`.git`/`.pico`/`__pycache__`/`.venv` 等）；`rg` 版**没有传 `--glob` 排除规则**，靠 `rg` 自带的 `.gitignore` 遵守。在 `.gitignore` 没覆盖的场景（比如 `__pycache__` 没写进 ignore），两条路径的过滤结果会不一样。

**备注 6：无匹配时返回 `"(no matches)"` 而不是空串。**
空串会被模型当成"工具没返回"；显式文案能让模型明确知道"搜过了，没有"。

---

### 1.4 `run_shell` — 跑命令（唯一同时"只读态也能被禁"的工具）

```python
def tool_run_shell(context, args):
    command = str(args.get("command", "")).strip()
    if not command:
        raise ValueError("command must not be empty")
    timeout = int(args.get("timeout", 20))
    if timeout < 1 or timeout > 120:
        raise ValueError("timeout must be in [1, 120]")
    result = subprocess.run(
        command,
        cwd=context.root,
        shell=True,
        capture_output=True,
        text=True,
        timeout=timeout,
        # 这里传入的是过滤后的环境变量，而不是直接继承整个父 shell 环境，
        # 目的是减少敏感信息被意外带进命令执行环境的风险。
        env=context.shell_env(),
    )
    return textwrap.dedent(
        f"""\
        exit_code: {result.returncode}
        stdout:
        {result.stdout.strip() or "(empty)"}
        stderr:
        {result.stderr.strip() or "(empty)"}
        """
    ).strip()
```

**备注 1：`env=context.shell_env()` 是这里最关键的一行。**
不继承父进程环境，只白名单透传。`security.shell_env()` 的实现：

```python
DEFAULT_SHELL_ENV_ALLOWLIST = ("HOME", "LANG", "LC_ALL", "LC_CTYPE", "LOGNAME",
    "PATH", "PWD", "SHELL", "TERM", "TMPDIR", "TMP", "TEMP", "USER")

def shell_env(env=None, allowlist=(), root="."):
    env = os.environ if env is None else env
    filtered = {name: env[name] for name in allowlist if name in env}
    filtered["PWD"] = str(root)
    if "PATH" not in filtered and env.get("PATH"):
        filtered["PATH"] = env["PATH"]
    return filtered
```

**没有 `*_API_KEY`、没有 `*_TOKEN`、没有 `AWS_*`、没有 `GITHUB_TOKEN`。** 所以模型即使写了 `curl -H "Authorization: $OPENAI_API_KEY"`，也拿不到值（变量不存在）。这比"执行后再脱敏"强一个量级——**从源头就不可能泄露**。

对比另一条防线：`redact_text()` 是在工具结果进 trace/report 之前做字符串替换。那层是"事后擦除"，这层是"事前隔离"。两层都要有——因为模型完全可能自己把密钥拼进 `echo` 里。

**备注 2：`PWD` 被强制覆写成 `context.root`。**
环境变量里的 `PWD` 可能是父进程的旧值，不覆写会让脚本里的 `$PWD` 指向错误目录。

**备注 3：`timeout` 卡在 `[1, 120]`。**
防止模型传 `timeout=99999` 把一个 run 挂死。上限 120 秒是"单条命令够用，但卡不死整个 run"的经验值。

**备注 4：输出统一成 `exit_code / stdout / stderr` 三段式。**
这个格式有下游依赖——`ToolExecutor` 里用正则从里面抠 exit code：

```python
match = re.search(r"exit_code:\s*(-?\d+)", content)
```

也就是 **"工具的输出格式"和"执行器的状态判定"是通过字符串正则耦合的**。改这里的输出文案，会静默改掉状态判定逻辑（`partial_success` / `error`）。这是全项目最脆的一处隐式契约。

**备注 5：`shell=True` + 白名单 env，边界在哪。**
白名单挡住的是"密钥泄露"，挡不住"命令本身能干什么"（`rm -rf`、`curl` 外联都能跑）。命令级的安全靠另外两道：`risky: True` 触发审批，以及 `read_only` 模式下审批直接返回 `False`（见第二部分第 5 节）。**读文件工具靠路径沙箱，跑命令工具靠审批——两种不同的安全模型。**

---

## 2. 写入组（`risky: True`，必须过审批 + 快照）

### 2.1 `write_file` — 写文件

```python
def tool_write_file(context, args):
    path = context.path(args["path"])
    content = str(args["content"])
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(content, encoding="utf-8")
    return f"wrote {path.relative_to(context.root)} ({len(content)} chars)"
```

**备注 1：`mkdir(parents=True, exist_ok=True)` —— 自动建父目录。**
模型说"在 `pico/features/x.py` 写"，即使 `features/` 不存在也能成。省掉一轮"先建目录再写"的往返。

**备注 2：全量覆盖，没有追加模式。**
`write_text` 直接覆盖。追加/局部改的职责归 `patch_file`。**一个工具一个语义**，不做参数开关式的多功能工具。

**备注 3：返回值很短，但带了两个关键信息。**
`wrote 相对路径 (N chars)` —— 路径让模型确认写对了地方，字符数让模型确认写全了（比如内容被截断成 0 字符时能立刻发现）。

**备注 4：`encoding="utf-8"` 无例外。**
Windows 上默认编码是 GBK，不写死 utf-8 就会出现"中文注释写进去变乱码"的问题。项目里所有读写都显式指定 utf-8。

---

### 2.2 `patch_file` — 精确改文件

```python
def tool_patch_file(context, args):
    path = context.path(args["path"])
    if not path.is_file():
        raise ValueError("path is not a file")
    old_text = str(args.get("old_text", ""))
    if not old_text:
        raise ValueError("old_text must not be empty")
    if "new_text" not in args:
        raise ValueError("missing new_text")
    text = path.read_text(encoding="utf-8")
    count = text.count(old_text)
    if count != 1:
        raise ValueError(f"old_text must occur exactly once, found {count}")
    path.write_text(text.replace(old_text, str(args["new_text"]), 1), encoding="utf-8")
    return f"patched {path.relative_to(context.root)}"
```

**备注 1：`count != 1` 直接拒绝 —— 这是整个工具最重要的设计。**
设计意图原话在 `validate_tool` 里：

> `patch_file` 故意做得很严格：`old_text` 必须精确命中且只能出现一次，这样修改行为才是确定的，失败原因也更容易解释。

三选一的哲学：**如果 `old_text` 出现 0 次 → 说明模型记错了文件内容；出现 2 次以上 → 说明改哪个是不确定的。两种情况都不猜，直接报错让模型重新读文件。**

对比"模糊匹配 / 自动纠错"的方案：那种方案在模型记错内容时能"猜着改"，代价是**可能改错位置且不报错**。在写代码这个场景下，静默改错比报错失败严重得多。

**备注 2：这个校验做了两遍（`validate_tool` 一遍、`tool_patch_file` 一遍）。**
看代码像是重复，实际是**刻意的分层**：

| 层 | 时机 | 作用 |
| --- | --- | --- |
| `validate_tool()` | 审批**之前** | 提前拦截非法参数，**避免用户为一个注定失败的补丁点确认** |
| `tool_patch_file()` | 审批之后、执行时 | 兜住"审批期间文件被外部改动"的窗口（TOCTOU） |

第二遍不是冗余——两遍之间存在时间窗口。

**备注 3：`replace(old_text, new_text, 1)` 的 `1` 是第三道保险。**
既然已经确认 `count == 1`，`1` 看似多余。它的价值在于**语义自解释**："我只替换第一处，且我知道只有一处"。如果有人后来放宽了 `count != 1` 的检查，这个 `1` 还能挡住"全部替换"的连锁破坏。

**备注 4：没有备份，但外层有快照兜底。**
`ToolExecutor` 在 `risky` 工具执行前后各拍一次工作区快照。这是"外层补偿而非内层防御"——工具本身不管回滚，回滚能力由执行器统一提供。

---

# 第二部分：工具是怎么被接入的（六步装配）

## 第 1 步：声明 —— 工具是一张白名单表（`tools.py:14-51`）

```python
BASE_TOOL_SPECS = {
    "list_files": {
        "schema": {"path": "str='.'"},
        "risky": False,
        "description": "List files in the workspace.",
    },
    "read_file": {
        "schema": {"path": "str", "start": "int=1", "end": "int=200"},
        "risky": False,
        "description": "Read a UTF-8 file by line range.",
    },
    "search": {
        "schema": {"pattern": "str", "path": "str='.'"},
        "risky": False,
        "description": "Search the workspace with rg or a simple fallback.",
    },
    "run_shell": {
        "schema": {"command": "str", "timeout": "int=20"},
        "risky": True,
        "description": "Run a shell command in the repo root.",
    },
    "write_file": {
        "schema": {"path": "str", "content": "str"},
        "risky": True,
        "description": "Write a text file.",
    },
    "patch_file": {
        "schema": {"path": "str", "old_text": "str", "new_text": "str"},
        "risky": True,
        "description": "Replace one exact text block in a file.",
    },
}

DELEGATE_TOOL_SPEC = {
    "schema": {"task": "str", "max_steps": "int=3"},
    "risky": False,
    "description": "Ask a bounded read-only child agent to investigate.",
}

def legal_tool_names():
    return set(BASE_TOOL_SPECS) | {"delegate"}
```

**备注 1：`schema` 是一个**手写的迷你 DSL**，不是 JSON Schema。**

| 写法 | 含义 |
| --- | --- |
| `"str"` | 必填字符串 |
| `"str='.'"` | 字符串，默认 `"."` |
| `"int=200"` | 整数，默认 `200` |

用这种极简格式而不是标准 JSON Schema，是为了一条链路服务：**第 3 步要把它渲染成文字塞进 prompt**。JSON Schema 渲染出来又长又啰嗦，`{"path": "str='.'"}` 一行就够。

**备注 2：`risky` 标志是整个安全分层的开关，只影响"是否要审批 + 是否拍快照"。**

| 工具 | risky | 审批 | 快照 | 理由 |
| --- | --- | --- | --- | --- |
| list_files | False | 否 | 否 | 只读 |
| read_file | False | 否 | 否 | 只读 |
| search | False | 否 | 否 | 只读 |
| run_shell | **True** | 是 | 是 | 能改文件、能外联 |
| write_file | **True** | 是 | 是 | 能改文件 |
| patch_file | **True** | 是 | 是 | 能改文件 |

`run_shell` 是唯一的"只读工具也 risky"的例外——它本身不写文件（`list_files`/`read_file` 语义上不产生副作用），但命令能做的事没有边界。

**备注 3：`delegate` 单独放一张表。**
因为它有**注册条件**（见下一步），不是无条件暴露的。

---

## 第 2 步：注册 —— 用 `partial` 把依赖注入进去（`tools.py:68-79`）

```python
def build_tool_registry(context):
    # 工具不是动态发现的，而是显式注册的。
    # 这样模型看到的是一个有边界、可审计的动作集合。
    tools = {
        name: {**spec, "run": partial(_TOOL_RUNNERS[name], context)}
        for name, spec in BASE_TOOL_SPECS.items()
    }
    # 子 agent 是刻意做成受限能力的：一旦深度耗尽，
    # 就连 delegate 这个工具都不再暴露给模型。
    if context.depth < context.max_depth:
        tools["delegate"] = {**DELEGATE_TOOL_SPEC, "run": partial(tool_delegate, context)}
    return tools

_TOOL_RUNNERS = {
    "list_files": tool_list_files,
    "read_file": tool_read_file,
    "search": tool_search,
    "run_shell": tool_run_shell,
    "write_file": tool_write_file,
    "patch_file": tool_patch_file,
}
```

**备注 1：`partial(_TOOL_RUNNERS[name], context)` —— 依赖注入的极简做法。**
`tool_list_files(context, args)` 是两参数函数，`partial` 把 `context` 先绑死，得到的 `run` 只需要一个 `args`。
所以调用方写的是 `tool["run"](args)`（见 `tool_executor.py:115`），**执行器完全不知道 `context` 的存在**。

好处：执行器只关心"名字 + 参数 → 结果"，工具函数只关心"上下文 + 参数 → 结果"，两边解耦。

**备注 2：`{**spec, "run": ...}` 是浅拷贝展开，不是原地修改。**
`BASE_TOOL_SPECS` 是模块级常量，被 `from . import tools as toolkit` 共享。如果写成 `spec["run"] = ...`，会把 `run` 污染进常量，第二次构建时 `BASE_TOOL_SPECS` 里就已经带着上一轮的 `context` 了。**这里必须拷贝。**

**备注 3：`delegate` 是"按深度条件注册"，不是"注册后拒绝调用"。**
`context.depth < context.max_depth` —— 深度用尽时，`delegate` **根本不出现在 `tools` 字典里**，也就不在 prefix 的工具说明里。模型看不到这个工具，自然不会想着调它。

对比另一种做法（注册但调用时报错）：那会让模型反复尝试一个注定失败的工具，白白浪费步数。**"不给看到"比"看到了再拒绝"更省 token、更省步数。**

（`validate_tool` 和 `tool_delegate` 里确实还有 `depth >= max_depth` 的检查——那是给外部直接调用 `run_tool("delegate", ...)` 留的后门防线，正常链路走不到。）

**备注 4：`build_tool_registry` 每次调用都新建一份 `ToolContext`。**
`runtime.py:177-178`：

```python
def build_tools(self):
    return toolkit.build_tool_registry(self.tool_context())
```

而 `tool_context()` 每次都 `ToolContext(...)` 新建对象。这是"不可变上下文"的做法——工具持有的 `context` 在本次 run 内是快照，不会被后续状态变化篡改。

---

## 第 3 步：暴露 —— schema 渲染成文字写进 prefix（`prompt_prefix.py:37-43`）

```python
def build_prompt_prefix(workspace, tools, built_at=None):
    tool_lines = []
    for name, tool in tools.items():
        fields = ", ".join(f"{key}: {value}" for key, value in tool["schema"].items())
        risk = "approval required" if tool["risky"] else "safe"
        tool_lines.append(f"- {name}({fields}) [{risk}] {tool['description']}")
    tool_text = "\n".join(tool_lines)
```

渲染出来是这样（真实输出）：

```
- list_files(path: str='.') [safe] List files in the workspace.
- read_file(path: str, start: int=1, end: int=200) [safe] Read a UTF-8 file by line range.
- search(pattern: str, path: str='.') [safe] Search the workspace with rg or a simple fallback.
- run_shell(command: str, timeout: int=20) [approval required] Run a shell command in the repo root.
- write_file(path: str, content: str) [approval required] Write a text file.
- patch_file(path: str, old_text: str, new_text: str) [approval required] Replace one exact text block in a file.
```

**备注 1：`[safe]` / `[approval required]` 直接写进 prompt。**
让模型提前知道"这个动作会被拦下来问用户"，能显著减少无效的 risky 调用。**这是把平台的约束"翻译"成模型能理解的语言**，而不是让模型去试错。

**备注 2：`[approval required]` 里连 `"` 都没有，用的是纯 ASCII。**
省 token，且不受终端/字体影响。

**备注 3：这段文本的 hash 是 prompt cache key 的一部分（`prompt_prefix.py:22-34`）。**

```python
def tool_signature(tools):
    payload = []
    for name in sorted(tools):
        tool = tools[name]
        payload.append({
            "name": name,
            "schema": tool["schema"],
            "risky": tool["risky"],
            "description": tool["description"],
        })
    return hashlib.sha256(json.dumps(payload, sort_keys=True).encode("utf-8")).hexdigest()
```

**注意 `payload` 里没有 `run`。** `partial` 对象不可 JSON 序列化，所以 `tool_signature` 只取可序列化的四个字段。

**副作用（重要）：改任何一个工具的 `description`、`schema` 或 `risky`，`tool_signature` 就变了 → `prefix` 变了 → `prompt_cache_key` 变了 → 缓存全部失效。** 所以"改一句工具说明"的代价不是改一行文案，而是丢掉所有已缓存的 prompt。这个隐式关联在上一份文档（长上下文治理）里也有对应——`prompt_cache_key = prefix_state.hash`。

**备注 4：`TOOL_EXAMPLES` 是给模型的"格式示范"。**
`tools.py:57-65` 为每个工具准备了一条真实调用样例，`build_prompt_prefix` 只挑了 6 条（list_files / read_file / write_file / patch_file / run_shell / final）写进 `Valid response examples`。

**这里有个不一致可以留意**：`TOOL_EXAMPLES` 字典里定义了 `search` 和 `delegate` 的样例，但 `build_prompt_prefix` 的 examples 是**硬编码的 6 行**，没有用 `TOOL_EXAMPLES`。所以 `search` 和 `delegate` 的样例只在一处起作用——`ToolExecutor` 报错时把它拼进错误信息：

```python
example = agent.tool_example(name)
message = f"error: invalid arguments for {name}: {exc}"
if example:
    message += f"\nexample: {example}"
```

也就是说 `TOOL_EXAMPLES` 的两个消费点分工不同：**prefix 里那 6 条是"提前示范"，executor 里的错误提示是"事后纠正"。**

---

## 第 4 步：解析 —— 两种调用格式（`runtime.py:644-736`）

```python
@staticmethod
def parse(raw):
    raw = str(raw)
    # 这里支持两种工具格式：
    # 1. <tool>...</tool> 里包 JSON，适合简短调用
    # 2. XML 风格属性/子标签，适合写文件这类多行内容
    if "<tool>" in raw and ("<final>" not in raw or raw.find("<tool>") < raw.find("<final>")):
        body = Pico.extract(raw, "tool")
        try:
            payload = json.loads(body)
        except Exception:
            return "retry", Pico.retry_notice("model returned malformed tool JSON")
        if not isinstance(payload, dict):
            return "retry", Pico.retry_notice("tool payload must be a JSON object")
        if not str(payload.get("name", "")).strip():
            return "retry", Pico.retry_notice("tool payload is missing a tool name")
        args = payload.get("args", {})
        if args is None:
            payload["args"] = {}
        elif not isinstance(args, dict):
            return "retry", Pico.retry_notice()
        return "tool", payload
    if "<tool" in raw and ("<final>" not in raw or raw.find("<tool") < raw.find("<final>")):
        payload = Pico.parse_xml_tool(raw)
        if payload is not None:
            return "tool", payload
        return "retry", Pico.retry_notice()
    if "<final>" in raw:
        final = Pico.extract(raw, "final").strip()
        if final:
            return "final", final
        return "retry", Pico.retry_notice("model returned an empty <final> answer")
    raw = raw.strip()
    if raw:
        return "final", raw
    return "retry", Pico.retry_notice("model returned an empty response")
```

**备注 1：三个返回值 `tool` / `final` / `retry`，对应控制循环的三条分支。**

| kind | 控制循环动作 | 代码位置 |
| --- | --- | --- |
| `tool` | 执行工具，`continue` 下一轮 | `agent_loop.py:215-254` |
| `final` | 结束本轮，写 report | `agent_loop.py:261-262` |
| `retry` | 把 notice 记进 history，`continue`（不消耗 tool 步数） | `agent_loop.py:256-259` |

**`retry` 不增加 `tool_steps`，只增加 `attempts`。** 所以模型可以"答错格式"好几次而不用消耗工具预算——这正是 `max_attempts = max(max_steps * 3, max_steps + 4)` 这个宽松上界存在的理由（`agent_loop.py:144`）。

**备注 2：`raw.find("<tool>") < raw.find("<final>")` —— 按出现位置判断意图。**
模型经常把说明和最终答案一起吐出来。这里取"谁先出现"来决定这是工具调用还是最终答案，避免把带示例的最终答案误判成工具调用。

**备注 3：`"<tool>"`（带右尖括号）和 `"<tool"`（不带）是两种不同的匹配。**
- `"<tool>"` 精确匹配 → 走 JSON 分支
- `"<tool"` 宽松匹配 → 走 XML 属性分支

顺序不能反。如果先判 `"<tool"`，`<tool>{"name":...}</tool>` 也会命中，被当成 XML 属性解析（解析不出 `name` 属性）→ 白白浪费一次 retry。

**备注 4：`parse_xml_tool` 解决了"多行内容塞不进 JSON"的问题。**

```python
@staticmethod
def parse_xml_tool(raw):
    match = re.search(r"<tool(?P<attrs>[^>]*)>(?P<body>.*?)</tool>", raw, re.S)
    if not match:
        return None
    attrs = Pico.parse_attrs(match.group("attrs"))
    name = str(attrs.pop("name", "")).strip()
    if not name:
        return None

    body = match.group("body")
    args = dict(attrs)
    for key in ("content", "old_text", "new_text", "command", "task", "pattern", "path"):
        if f"<{key}>" in body:
            args[key] = Pico.extract_raw(body, key)

    body_text = body.strip("\n")
    if name == "write_file" and "content" not in args and body_text:
        args["content"] = body_text
    if name == "delegate" and "task" not in args and body_text:
        args["task"] = body_text.strip()
    return {"name": name, "args": args}
```

关键在 `re.S`（让 `.` 匹配换行）和 `extract_raw`（**不做 `.strip()`**）：

```python
@staticmethod
def extract_raw(text, tag):
    ...
    return text[start:end]      # 注意：没有 .strip()
```

对比 `extract`（给 `<final>` 用，会 `.strip()`）——`extract_raw` 保留首尾空白，因为**文件内容里的缩进和末尾换行是有意义的**。如果用 `extract` 来抠 `content`，Python 文件的缩进会被 `.strip()` 吃掉，写出来的文件直接语法错误。

**备注 5：`body_text` 兜底 —— 模型只给 `<tool name="write_file" path="a.py">代码</tool>` 也能work。**
没有 `<content>` 子标签时，整个 body 当内容用。这是对模型偷懒的容错。

**备注 6：`parse_attrs` 只认双引号或单引号包裹的属性值。**

```python
def parse_attrs(text):
    attrs = {}
    for match in re.finditer(r"""([A-Za-z_][A-Za-z0-9_]*)\s*=\s*(?:"([^"]*)"|'([^']*)')""", text):
        attrs[match.group(1)] = match.group(2) if match.group(2) is not None else match.group(3)
    return attrs
```

所以 `<tool name=write_file ...>`（不加引号）解析不出 `name`，会退化成 retry。**格式约束是被强制执行的，不是"尽量兼容"。**

---

## 第 5 步：执行闸门 —— 七道关卡（`tool_executor.py:45-142`）

```python
def execute(self, name, args):
    agent = self.agent

    # 关卡 1：允许清单
    if agent.allowed_tools is not None and name not in agent.allowed_tools:
        return ToolExecutionResult(
            content=f"error: tool '{name}' is not allowed in this run",
            metadata=_metadata("rejected", tool_error_code="tool_not_allowed",
                               risk_level="high", read_only=False),
        )

    # 关卡 2：工具是否存在
    tool = agent.tools.get(name)
    if tool is None:
        return ToolExecutionResult(
            content=f"error: unknown tool '{name}'",
            metadata=_metadata("rejected", tool_error_code="unknown_tool",
                               risk_level="high", read_only=False),
        )

    # 关卡 3：参数校验
    try:
        agent.validate_tool(name, args)
    except Exception as exc:
        example = agent.tool_example(name)
        message = f"error: invalid arguments for {name}: {exc}"
        if example:
            message += f"\nexample: {example}"
        security_event_type = "path_escape" if "path escapes workspace" in str(exc) else ""
        return ToolExecutionResult(
            content=message,
            metadata=_metadata("rejected", tool_error_code="invalid_arguments",
                               security_event_type=security_event_type,
                               risk_level="high" if tool["risky"] else "low",
                               read_only=not tool["risky"]),
        )

    # 关卡 4：重复调用拦截
    if agent.repeated_tool_call(name, args):
        return ToolExecutionResult(
            content=f"error: repeated identical tool call for {name}; choose a different tool or return a final answer",
            metadata=_metadata("rejected", tool_error_code="repeated_identical_call", ...),
        )

    # 关卡 5：审批（只有 risky 才问）
    if tool["risky"] and not agent.approve(name, args):
        return ToolExecutionResult(
            content=f"error: approval denied for {name}",
            metadata=_metadata("rejected", tool_error_code="approval_denied", ...),
        )

    # 关卡 6：执行前快照（只有 risky 才拍）
    before_snapshot = agent.capture_workspace_snapshot() if tool["risky"] else {}
    after_snapshot = before_snapshot
    try:
        content = clip(tool["run"](args))
        after_snapshot = agent.capture_workspace_snapshot() if tool["risky"] else before_snapshot
        affected_paths, diff_summary = agent.diff_workspace_snapshots(before_snapshot, after_snapshot)
        workspace_changed = bool(affected_paths)
        tool_status = "ok"
        tool_error_code = ""

        # 关卡 7：run_shell 的状态判定
        if name == "run_shell":
            match = re.search(r"exit_code:\s*(-?\d+)", content)
            exit_code = int(match.group(1)) if match else 0
            if exit_code != 0 and workspace_changed:
                tool_status = "partial_success"
                tool_error_code = "tool_partial_success"
            elif exit_code != 0:
                tool_status = "error"
                tool_error_code = "tool_failed"

        agent.update_memory_after_tool(name, args, content)
        metadata = _metadata(...)
        agent.record_process_note_for_tool(name, metadata)
        return ToolExecutionResult(content=content, metadata=metadata)
    except Exception as exc:
        ...  # 异常路径同样拍快照、同样算 diff、同样记 process note
```

**备注 1：七道关卡全部返回字符串，不抛异常。**

这是全链路最重要的约定。`run_tool()` 的文档注释说得很清楚：

> 无论是成功结果还是错误信息，都会统一返回文本，这样模型下一轮都能继续消费这份反馈。

**"被拒绝"不是系统的失败，而是模型的可消费反馈。** 模型拿到 `error: approval denied for run_shell` 后，下一轮就知道"这条路走不通，换个方式"。如果抛异常，控制循环就得处理"工具报错怎么办"这个额外分支，复杂度和不确定性都上升。

**备注 2：关卡 4「重复调用拦截」的实现（`runtime.py:533-540`）。**

```python
def repeated_tool_call(self, name, args):
    # agent 很常见的一种坏循环，是在没有新信息的情况下反复发起同一调用。
    # 这里提前挡掉最简单的这种循环。
    tool_events = [item for item in self.session["history"] if item["role"] == "tool"]
    if len(tool_events) < 2:
        return False
    recent = tool_events[-2:]
    return all(item["name"] == name and item["args"] == args for item in recent)
```

只检查**最近 2 次**工具调用是否和当前完全一样。实现故意做得很窄：**不统计全历史频率、不做相似度判断**。因为更激进的去重会误伤合法场景（比如"改完文件再跑一次同样的测试"是完全合理的）。

**代价**：如果模型是 `A, B, A, B, A, B...` 这种交替循环，这里挡不住。这是"宁可漏拦，不可误伤"的取舍。

**备注 3：关卡 5 审批的三档策略（`runtime.py:631-642`）。**

```python
def approve(self, name, args):
    if self.read_only:
        return False                  # ← 只读模式：一律拒绝
    if self.approval_policy == "auto":
        return True
    if self.approval_policy == "never":
        return False
    try:
        answer = input(f"approve {name} {json.dumps(args, ensure_ascii=True)}? [y/N] ")
    except EOFError:
        return False                  # ← 非交互环境（管道/CI）默认拒绝
    return answer.strip().lower() in {"y", "yes"}
```

| 策略 | 行为 | 用在哪 |
| --- | --- | --- |
| `auto` | 全部放行 | 评测/回归测试 |
| `never` | 全部拒绝 | 子 agent（`spawn_delegate` 里写死 `approval_policy="never"`） |
| 其它（默认） | 交互式 `input()` 问用户 | CLI |

**`read_only` 的检查排在策略判断之前** —— 这是"只读模式不可被 approval_policy 覆盖"的保证。子 agent 是 `read_only=True` + `approval_policy="never"` 双重锁定：

```python
child = Pico(
    ...
    approval_policy="never",
    max_steps=int(args.get("max_steps", 3)),
    depth=self.depth + 1,
    max_depth=self.max_depth,
    read_only=True,
    ...
)
```

**即使有人把 `approval_policy` 改成 `auto`，`read_only=True` 那一关依然先返回 `False`。** 两把锁是串联的，不是并联的。委派出去的子 agent 只能调查、不能改——这才是安全的。

**备注 4：`EOFError` → 返回 False。**
在 CI 或管道里跑 agent 时，`input()` 会立刻 EOF。这里默认拒绝而不是崩溃，也不默认放行。**失败安全的方向是"拒绝"**。

**备注 5：关卡 6/7 的快照机制。**

快照本身很简单（`runtime.py:348-363`）：

```python
def capture_workspace_snapshot(self):
    snapshot = {}
    for path in self.root.rglob("*"):
        try:
            relative_parts = path.relative_to(self.root).parts
        except ValueError:
            continue
        if any(part in IGNORED_PATH_NAMES for part in relative_parts):
            continue
        if not path.is_file():
            continue
        try:
            snapshot[path.relative_to(self.root).as_posix()] = hashlib.sha256(path.read_bytes()).hexdigest()
        except Exception:
            continue
    return snapshot
```

**按内容 hash 而不是 mtime。** 理由是可靠性：`write_file` 写入相同内容时，mtime 会变但内容没变——用 mtime 会误报"文件被改了"。hash 判断的是"内容真的变了没有"。

差集判定（`runtime.py:365-380`）返回两类信息：

```python
@staticmethod
def diff_workspace_snapshots(before, after):
    changed_paths = []
    summaries = []
    for path in sorted(set(before) | set(after)):
        if before.get(path) == after.get(path):
            continue
        changed_paths.append(path)
        if path not in before:
            summaries.append(f"created:{path}")
        elif path not in after:
            summaries.append(f"deleted:{path}")
        else:
            summaries.append(f"modified:{path}")
    return changed_paths, summaries
```

`created:` / `deleted:` / `modified:` 三种标记，直接进 trace。

**注意：这里只做"有没有变"，不做"变了什么"。** 没有 diff 文本内容——那是另一个量级的工作量，而且会把 trace 撑爆。

**备注 6：关卡 7 —— 为什么只有 `run_shell` 需要状态判定。**
`write_file` / `patch_file` 失败会直接抛异常（进 `except` 分支）。只有 `run_shell` 是"进程跑完了但退出码非 0"——**这不是异常，是合法的返回**。

于是这里做了一次有价值的区分：

| 情况 | 状态 | 含义 |
| --- | --- | --- |
| exit 0 | `ok` | 正常 |
| exit ≠ 0 **且工作区变了** | `partial_success` | **命令失败但改了文件** ← 最危险 |
| exit ≠ 0 且工作区没变 | `error` | 干脆失败，没有副作用 |

`partial_success` 这个状态是整个设计里最有价值的一个。典型场景：模型跑 `mv a.py b.py && rm c.py`，中间某步失败了，但前面已经生效。**如果不区分这个状态，模型会以为"失败了，什么都没发生"，然后重试一遍——第二次可能把已经改好的东西又改坏。**

判定之后立刻沉淀进记忆（`runtime.py:425-439`）：

```python
def record_process_note_for_tool(self, name, metadata):
    status = str(metadata.get("tool_status", "")).strip()
    if status not in {"partial_success", "error", "rejected"}:
        return
    affected_paths = [str(path).strip() for path in metadata.get("affected_paths", []) if str(path).strip()]
    path_text = ", ".join(affected_paths) or "workspace"
    if status == "partial_success":
        text = f"{name} partial_success on {path_text}; inspect diff before retry"
    elif status == "error":
        text = f"{name} error on {path_text}; check the failure before retry"
    else:
        text = f"{name} rejected; choose a different action before retry"
    tags = ["process", status, *affected_paths]
    self.memory.append_note(text, tags=tuple(tags), source=name, kind="process")
    self.session["memory"] = self.memory.to_dict()
```

**注意 `ok` 状态被直接 `return` 掉了。** 只有异常状态才值得占用记忆条目的额度。**记忆是稀缺资源，只记"教训"，不记"一切正常"。** 而且这三条文案都带了明确的行动指令（`inspect diff before retry` / `check the failure` / `choose a different action`）——写进记忆就是为了影响下一轮决策。

---

## 第 6 步：记账 —— 执行之后发生了什么（`agent_loop.py:215-254`）

```python
if kind == "tool":
    tool_steps += 1
    name = payload.get("name", "")
    args = payload.get("args", {})
    task_state.record_tool(name)
    tool_started_at = time.monotonic()
    tool_result = agent.execute_tool(name, args)
    result = tool_result.content
    agent.record({
        "role": "tool",
        "name": name,
        "args": args,
        "content": result,
        "created_at": now(),
    })
    agent.run_store.write_task_state(task_state)
    agent.emit_trace(
        task_state,
        "tool_executed",
        {
            "name": name,
            "args": args,
            "result": clip(result, 500),        # ← trace 里只留 500 字符
            "duration_ms": int((time.monotonic() - tool_started_at) * 1000),
            **dict(tool_result.metadata or {}),  # ← metadata 全部摊平进 trace
        },
    )
    checkpoint = agent.create_checkpoint(task_state, user_message, trigger="tool_executed")
    agent.run_store.write_task_state(task_state)
    agent.emit_trace(task_state, "checkpoint_created", {...})
    continue
```

**备注 1：三个不同的存储，三种不同的长度。**
同一次工具调用，结果被存了三份，长度各不相同：

| 存哪 | 长度 | 用途 |
| --- | --- | --- |
| `history`（进 prompt） | 原始长度 | 给模型看 |
| `trace.jsonl` | `clip(result, 500)` | 给人审计 |
| `tool["run"]` 返回值 | `clip(result)` = 最多 `MAX_TOOL_OUTPUT`（4000） | 工具层的第一道裁剪 |

**`clip` 被调了两次，是两个不同的闸门。** `tool_executor.py:115` 的 `clip(tool["run"](args))` 是 4000（`MAX_TOOL_OUTPUT`），`agent_loop` 里的 `clip(result, 500)` 是 trace 专用的 500。**`run_shell` 打印 10 万行输出时，模型看到 4000 字符，trace 里只有 500 字符，工具输出本身一个字都没丢进错误日志。**

**备注 2：每次工具执行都建一次 checkpoint。**
`trigger="tool_executed"`。所以 checkpoint 的粒度是"每个工具步"，不是"每个 run"。这就是恢复机制能做到"从上一个工具结果之后接着跑"的原因。

**备注 3：`**dict(tool_result.metadata or {})` 把 metadata 摊平进 trace。**
所以 `trace.jsonl` 里直接能看到 `tool_status`、`tool_error_code`、`affected_paths`、`workspace_changed`、`diff_summary`、`security_event_type`、`risk_level`、`read_only` —— **不需要看代码就能知道这一步做了什么、改了什么、有没有被拒绝。** 这是"可审计"的具体形态。

---

## 附：`ToolContext` —— 为什么工具函数拿不到 `agent`

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

`runtime.py` 里的装配：

```python
def tool_context(self):
    return ToolContext(
        root=self.root,
        path_resolver=self.path,
        shell_env_provider=self.shell_env,
        depth=self.depth,
        max_depth=self.max_depth,
        spawn_delegate=self.spawn_delegate,
    )
```

**备注 1：工具只能看到 6 个东西，而不是整个 agent。**
这是刻意的能力收窄。如果传 `agent` 进去，工具函数里就能写 `agent.session`、`agent.model_client`、`agent.run_store` —— 那么"工具能干什么"就变成"读代码才知道"，审计成本爆炸。

现在这 6 个字段是**声明式的能力清单**：能解析路径、能拿 shell 环境、知道自己在第几层、能派生一个子 agent。**别的都碰不到。**

**备注 2：`path_resolver` / `shell_env_provider` 是 Callable 而不是值。**
因为路径解析依赖 `self.root`（不变），但 shell env 每次可能要重新取。用 Callable 而不是直接传 `dict`，保持了"按需计算"的灵活性。

**备注 3：`ToolContext` 同时管着"能不能再委派"（`depth` / `max_depth`）。**
所以"委派深度"不是在工具函数里判断的，而是作为上下文的一部分传进去。同一个 `tool_delegate` 函数，在不同深度的 `context` 下行为不同：

```python
def tool_delegate(context, args):
    if context.depth >= context.max_depth:
        raise ValueError("delegate depth exceeded")
    ...
```

---

## 附：路径沙箱 —— 所有文件工具共用的一条线（`runtime.py:771-780`）

```python
def path(self, raw_path):
    path = Path(raw_path)
    path = path if path.is_absolute() else self.root / path
    resolved = path.resolve()
    # 所有文件类工具都被锚定在 workspace root 之下。
    # 这样既能防住 "../" 逃逸，也能防住符号链接解析后跳出仓库。
    if os.path.commonpath([str(self.root), str(resolved)]) != str(self.root):
        raise ValueError(f"path escapes workspace: {raw_path}")
    return resolved
```

**备注 1：五个文件类工具（list_files / read_file / search / write_file / patch_file）全都经过这里。**
因为每个工具的第一行都是 `context.path(...)`。**一个函数，一条边界，没有例外路径。**

**备注 2：`path.resolve()` 是防符号链接的关键。**

```
workspace/
  link -> /etc/passwd      ← 工作区里有个符号链接

context.path("link")  →  raw_path 是相对路径，先拼成 workspace/link
                      →  .resolve() 顺着符号链接解到 /etc/passwd
                      →  commonpath 检查失败 → 拒绝
```

**如果只在字符串层面判断 `".." in raw_path`，这个攻击完全挡不住。** `resolve()` 之后比较真实路径，才是"物理位置"级别的检查。

**备注 3：`os.path.commonpath` 而不是 `startswith`。**

```python
# 错误做法
if str(resolved).startswith(str(self.root)):   # ← /workspace-evil 也能通过
```

`commonpath` 按路径分隔符做**组件级**比较，不会把 `/tmp/workspace-evil` 误判成在 `/tmp/workspace` 下面。

**备注 4：抛的是 `ValueError`，会被关卡 3 接住并转成可消费的文本。**

```python
security_event_type = "path_escape" if "path escapes workspace" in str(exc) else ""
```

**靠异常消息字符串匹配来识别安全事件类型** —— 又是一处字符串耦合（和 `run_shell` 的 `exit_code` 正则同类）。改这句话的措辞会让安全事件分类静默失效。

---

# 设计思路总结

## 四条主线

| # | 思路 | 具体体现 |
| --- | --- | --- |
| 1 | **显式白名单，不做动态发现** | `BASE_TOOL_SPECS` 是写死的字典；`delegate` 还按深度条件注册。模型看到的是一个"有边界、可审计"的集合 |
| 2 | **错误是反馈，不是异常** | 七道关卡全部返回字符串；`retry` 不消耗工具预算。模型可以"试错"，但每一步都在平台划定的圈子里 |
| 3 | **依赖收窄** | `ToolContext` 只给 6 个字段，而不是整个 `agent`。工具能力的边界是**声明式**的，不用读实现就知道 |
| 4 | **两层安全模型** | 读文件靠**路径沙箱**（`resolve()` + `commonpath`）；跑命令靠**审批 + 环境白名单 + 只读锁定**。两类风险用两种手段，不混用 |

## 一个类比

把这套东西想成**公司门禁系统**：

- `BASE_TOOL_SPECS` = **员工手册里列出的允许事项**，不是"你能想到的都能干"
- `ToolContext` = **工牌**，上面只有"能开哪几扇门"，不含"公司全部权限"
- `ToolExecutor` 七道关卡 = **闸机**：卡是不是本公司的（允许清单）→ 门存不存在（工具存在）→ 有没有权限（参数校验）→ 是不是刚刷过（重复拦截）→ 要不要主管签字（审批）→ 进出前后录像（快照）
- `path()` 沙箱 = **围墙**：闸机拦的是"刷卡动作"，围墙拦的是"绕过去"
- `partial_success` = **保安的记事本**："这人进来了、东西动了但没干成，下次注意"——比"成功/失败"两分法信息量大得多

**核心是：模型提需求，平台定边界。模型永远拿不到"直接执行"的能力，只能拿到"申请执行"的通道。**

## 三个已知的粗糙处（只报告，不修改）

1. **`run_shell` 的输出格式与执行器的状态判定用正则耦合。**
   `ToolExecutor` 靠 `re.search(r"exit_code:\s*(-?\d+)", content)` 从工具返回的文本里抠退出码。改工具的输出文案会静默改掉状态判定。

2. **`path escapes workspace` 这句话是安全事件分类的唯一依据。**
   `security_event_type = "path_escape" if "path escapes workspace" in str(exc) else ""` —— 同样是字符串匹配。

3. **`search` 的两条路径行为不一致。**
   `rg` 版的 `--max-count 200` 是"每文件 200 条"且不显式排除 `IGNORED_PATH_NAMES`；Python 兜底版是"总共 200 条"且显式排除。同一个 `search` 调用，取决于环境里有没有装 `rg`，结果集和长度特征都不一样。

## 一处可留意的不一致

`TOOL_EXAMPLES` 里定义了 `search` 和 `delegate` 的样例，但 `build_prompt_prefix` 的 `Valid response examples` 段是硬编码的 6 行，**没有引用 `TOOL_EXAMPLES`**。所以这两个工具的样例会走 `ToolExecutor` 报错时拼进错误信息这条路（`example: ...`），而不是提前出现在 prefix 里。

---

# 面试问答（口语化）

**Q：这几个工具是怎么接进 agent 的？**
A：五步。先在 `tools.py` 里用一张声明表写清楚"有哪些工具、参数长什么样、要不要审批"；然后 `build_tool_registry` 把上下文用 `partial` 绑进每个工具函数；再把这张表渲染成文字写进 prompt 的 prefix，让模型知道有什么可调；模型输出后用 `parse` 转成结构化的名字+参数；最后所有调用都过一个统一的执行闸门。**关键是模型只能"申请"，从来没有直接执行的能力。**

**Q：为什么工具函数拿到的是 `ToolContext` 而不是整个 agent？**
A：能力收窄。`ToolContext` 只有 6 个字段：根目录、路径解析器、环境变量获取器、当前深度、最大深度、派生函数。工具想动 session、想直接调模型，做不到。好处是"工具能干什么"在一个 dataclass 里就看完了，不用读实现，审计成本直接降一个量级。

**Q：为什么被拒绝的调用返回字符串而不是抛异常？**
A：因为这是给模型的反馈，不是系统的故障。模型拿到 `error: approval denied` 下一轮就知道换路；抛异常的话控制循环还得多一个分支处理。而且 `retry` 这种"格式错误"不消耗工具步数，所以模型有试错空间，但试错范围是平台划定的。**错误消息本身也是 prompt 的一部分。**

**Q：读工具和写工具的安全策略有什么不一样？**
A：两套模型。读文件靠路径沙箱 —— 所有路径都过 `resolve()` 再和根目录比 `commonpath`，能挡住 `../` 逃逸和符号链接跳出去。写工具靠审批 —— `risky: True` 就会在动手前问用户，而且执行前后各拍一次工作区快照好算 diff。`run_shell` 比较特殊，它语义上不写文件但能力上什么都能干，所以它也标了 risky，同时环境变量走白名单，API key 这类根本不会传进去。

**Q：`patch_file` 为什么要求 `old_text` 只能出现一次？**
A：为了让"改哪一处"是确定的。出现 0 次说明模型记错了文件内容，出现 2 次以上说明改哪个都说得通 —— 这两种情况都不猜，直接报错让模型重新读文件。对比"模糊匹配自动纠错"的方案：那种方案能在模型记错时猜着改，代价是可能改错位置还不报错。写代码这个场景下，静默改错比报错失败严重得多。而且这个校验特意做了两遍：审批之前一遍是为了不让用户为一个注定失败的补丁点确认，执行时再一遍是为了挡住审批期间文件被外部改动的窗口。

**Q：工具执行失败怎么区分"什么都没发生"和"改了一半"？**
A：靠 `exit_code` 和工作区快照的组合判断。`run_shell` 退出码非 0 但工作区真变了，标成 `partial_success`；退出码非 0 且工作区没变，才是 `error`。这个区分很关键 —— 如果统一报"失败"，模型会以为什么都没发生然后重试，第二次可能把已经改好的部分又改坏。标成 `partial_success` 之后还会写一条带行动指令的记忆（"重试前先看 diff"），直接影响下一轮决策。
