# workspace.py 核心逻辑精读（提炼版）

> 对象：`pico/workspace.py`（134 行）  
> 定位：提炼版。讲骨架、设计理由、实测证据，不逐行走读。  
> 同系列：`tools-core-reading.md` / `tool-context-core-reading.md` / `tool-executor-core-reading.md`  
> 所有数字均为本机实测（Windows + Python 3.13.12），脚手架见文末。

---

## 1. 一句话概括

**134 行、6 个顶层名字、1 个类。这个模块做两件事：给 agent 一份便宜的"仓库第一印象"，以及提供项目里唯一一套文本裁剪工具。**

模块 docstring 把定位写得很清楚：

> 这个模块负责在 agent 按需读文件之前，先给它一份便宜的"仓库第一印象"。这份快照刻意保持**小而稳定**：主要包含 Git 事实和少量白名单项目文档。

它是整条工具链的**最底层**——`tools.py` 从它拿 `IGNORED_PATH_NAMES`，`runtime.py` 从它拿 `MAX_HISTORY` 和 `clip`，`tool_executor.py` 从它拿 `clip`，`memory.py` 从它拿 `clip` 和 `now`。

**复杂度集中在 `WorkspaceContext.build`（47 行）**，占全文 35%。其余五个东西都是十行以内的小工具。

---

## 2. 行区间分布

| 行区间        | 内容                                           | 行数     | 承担什么                  |
| ---------- | -------------------------------------------- | ------ | --------------------- |
| 1–12       | docstring + import                           | 12     | ——                    |
| 14–15      | `MAX_TOOL_OUTPUT=4000` / `MAX_HISTORY=12000` | 2      | 两个跨模块共用的预算常量          |
| 16–18      | `DOC_NAMES`（4 个文件名）                          | 3      | 预加载白名单                |
| 19         | `IGNORED_PATH_NAMES`（7 个名字）                  | 1      | 遍历排除集                 |
| 22–23      | `now()`                                      | 2      | UTC ISO 时间戳           |
| 26–30      | `clip()`                                     | 5      | **留头 + 截断提示**         |
| 33–41      | `middle()`                                   | 9      | **留头留尾**              |
| 45–52      | `__init__`                                   | 8      | 7 个字段的朴素赋值            |
| **54–100** | **`build`**                                  | **47** | **5 次 git 调用 + 文档扫描** |
| 102–120    | `text()`                                     | 19     | 快照 → prompt 文本        |
| 122–134    | `fingerprint()`                              | 13     | 快照 → sha256           |

**该看哪几段**：`build`（唯一有 IO 和失败处理的地方）、`clip` 与 `middle`（项目里两种裁剪语义的来源）。

---

## 3. 必须分清的概念

### 3.1 `clip` 和 `middle` 是两种**完全不同**的裁剪

|          | `clip(text, limit)`                                    | `middle(text, limit)` |
| -------- | ------------------------------------------------------ | --------------------- |
| 保留       | **头部**                                                 | **头 + 尾**             |
| 结果长度     | `limit + len("\n...[truncated N chars]")`，**超出 limit** | **严格等于 limit**        |
| 换行       | 原样保留                                                   | **全部替换成空格**           |
| 谁说清被砍了多少 | 说（`truncated N chars`）                                 | 不说（只放 `...`）          |
| 谁在用      | 工具输出、history、memory、workspace status                   | **只有 CLI 的终端显示**      |

实测：

```
clip(t, 100)   -> 长度 126  结尾 'xxxx\n...[truncated 1909 chars]'
middle(t, 100) -> 长度 100  内容 '行1 行2 行3 xxxxxx...' ... 'xxxxxxxxxx'
```

**而项目里还有第三种**——`context_manager._tail_clip`，它留头、严格等于 limit、不留截断提示。**三个名字相近的函数，三种语义**，混用会出静默的错误长度。

### 3.2 `MAX_TOOL_OUTPUT` 不是"工具输出上限"，是 `clip` 的默认参数

```python
MAX_TOOL_OUTPUT = 4000
def clip(text, limit=MAX_TOOL_OUTPUT):
```

它被用作默认值，所以 `tool_executor.py:115` 那句裸调用 `clip(tool["run"](args))` 实际就是 `clip(..., 4000)`。**名字里的 "TOOL_OUTPUT" 描述的是"最典型的用法"，不是约束。**

`MAX_HISTORY = 12000` 更直接——**它在本文件里完全没被用到**，读者是 `runtime.py:263`（拼 history 文本时）。

### 3.3 `cwd` 和 `repo_root` 是两个不同的东西

- `cwd`：用户**在哪启动** agent 的目录
- `repo_root`：Git 仓库**根目录**（`git rev-parse --show-toplevel`）

两者可以不同：在 `repo/pkg/` 下启动时，`cwd` 是 `repo/pkg`，`repo_root` 是 `repo`。

`build` 里的文档扫描**同时看这两个目录**（80 行 `for base in (repo_root, cwd)`），理由写在第 78–79 行的注释里：

> 同时扫描 repo_root 和 cwd，这样在子目录启动时也能看到本地文档；但用相对路径做 key，避免同一份文档被重复收集。

实测（在 `repo/pkg` 下启动）：

```
project_docs 键: ['README.md', 'pkg\\README.md', 'pkg\\package.json']
```

`README.md` 只收了一份——先扫 `repo_root` 命中并记下 key，扫 `cwd` 时发现同一个 key 就跳过了。

### 3.4 `IGNORED_PATH_NAMES` 是"名字集合"，不是"路径前缀集合"

```python
IGNORED_PATH_NAMES = {".git", ".pico", "__pycache__", ".pytest_cache", ".ruff_cache", ".venv", "venv"}
```

它匹配的是**路径中的任意一段名字**，不是前缀。所以 `src/vendor/__pycache__/x.pyc` 会被排除（因为有一段叫 `__pycache__`），而 `src/.github/workflows/ci.yml` 不会被排除（`.github` 不在集合里）。

三个使用点对这一点的理解是一致的：

| 位置               | 用法                                                                 |
| ---------------- | ------------------------------------------------------------------ |
| `tools.py:161`   | `item.name not in IGNORED_PATH_NAMES`（只看当前层）                       |
| `tools.py:202`   | `any(part in IGNORED_PATH_NAMES for part in relative.parts)`（看每一段） |
| `runtime.py:355` | 同上（看每一段）                                                           |

**注意它只是"遍历时跳过"，不是"禁止访问"。** `.git` 会被 `list_files` 隐藏，但 `read_file(".git/config")` 仍然能读到——因为路径沙箱只检查"是否在 workspace 内"，不检查"是否在忽略名单里"。

---


## 4. 主流程骨架

```
WorkspaceContext.build(cwd, repo_root_override=None)
  ├─ cwd = Path(cwd).resolve()
  ├─ repo_root = override 或 git rev-parse --show-toplevel
  ├─ docs = {}
  │    for base in (repo_root, cwd):            # 两个根都扫
  │        for name in DOC_NAMES:              # 只扫 4 个固定文件名
  │            key = path.relative_to(repo_root)   # ★ 用相对路径去重
  │            docs[key] = clip(内容, 1200)
  └─ 返回 cls(
         branch         = git(["branch","--show-current"], "-")
         default_branch = git(["symbolic-ref",...], "origin/main") 去掉 origin/ 前缀
         status         = clip(git(["status","--short"], "clean"), 1500)
         recent_commits = git(["log","--oneline","-5"]).splitlines()
         project_docs   = docs
     )
```

**一共 5 次 git 子进程调用**，每次都有 5 秒超时，每次失败都返回 fallback。

`text()` 把快照拍平成 prompt 片段：

```
Workspace:
- cwd: ...
- repo_root: ...
- branch: ...
- default_branch: ...
- status:
<git status --short>
- recent_commits:
- <5 行>
- project_docs:
- README.md
<snippet>
```

`fingerprint()` 把同样的 7 个字段 `json.dumps(sort_keys=True)` 后 sha256。

---

## 5. 核心函数逐个拆

### 5.1 `clip`（26–30）

```python
def clip(text, limit=MAX_TOOL_OUTPUT):
    text = str(text)
    if len(text) <= limit:
        return text
    return text[:limit] + f"\n...[truncated {len(text) - limit} chars]"
```

**备注：它主动告诉调用方"被砍了多少"。**

这是它和另外两个裁剪函数最重要的区别。砍掉的不是一个静默的 `...`，而是一句具体的话：

```
...[truncated 1909 chars]
```

**为什么这个细节重要？** 因为 clip 的结果会**进 prompt 给模型看**。模型看到 "truncated 1909 chars" 就知道"这里还有东西我没看到，如果需要得换个方式取"；只看到一个 `...` 的话，模型可能以为那就是全部。

**代价：结果长度超出 limit。** 名义 1500 的 `status` 实测输出 **1526 字符**。这个超出量不固定（取决于被砍了多少字符，`truncated 1909 chars` 比 `truncated 9 chars` 长），所以**无法预算**——调用方如果拿 `len(clip(x, n)) <= n` 做断言，会失败。


### 5.2 `middle`（33–41）

```python
def middle(text, limit):
    text = str(text).replace("\n", " ")
    if len(text) <= limit:
        return text
    if limit <= 3:
        return text[:limit]
    left = (limit - 3) // 2
    right = limit - 3 - left
    return text[:left] + "..." + text[-right:]
```

**备注一：第一行就把换行压平了。**

```python
text = str(text).replace("\n", " ")
```

实测：

```
输入 : '第一行内容\n第二行内容\n第三行内容'
输出 : '第一行内容 第二行内容 第三行内容'
```

**这是刻意的**——它唯一的用途是 CLI 终端里显示单行。

**但这是个危险的性质**：如果把这个函数用到工具输出上（比如报错栈、表格），**结构会被压成一行**。它之所以安全，是因为调用点全在 `cli.py` 的展示层（188/195/199/216 行），**没有任何 prompt 或数据处理逻辑用它**。

**备注二：左右分配是"左少右多"。**

```python
left = (limit - 3) // 2      # 向下取整 → 左边少
right = limit - 3 - left     # 右边补足
```

`limit=10` 时 `left=3, right=4`。**为什么右边多给一个？** 因为尾部通常更有信息量（错误信息、文件扩展名、路径的最后一段）。这是个没写进注释的细节。

**备注三：边界值里藏了一个反直觉的行为。**

实测：

```
middle('ABCDEFGHIJ', 100) -> 'ABCDEFGHIJ'
middle('ABCDEFGHIJ',  10) -> 'ABCDEFGHIJ'
middle('ABCDEFGHIJ',   4) -> '...J'          ← left=0，只剩省略号和最后一个字符
middle('ABCDEFGHIJ',   3) -> 'ABC'           ← 放不下省略号，硬切头部
middle('ABCDEFGHIJ',   2) -> 'AB'
middle('ABCDEFGHIJ',   0) -> ''
middle('ABCDEFGHIJ',  -1) -> 'ABCDEFGHI'     ← 负数：变成"去掉最后 1 个字符"
```

**`limit = -1` 时返回 `text[:-1]`** ——因为 `len(text) <= limit` 为假（10 ≤ -1 不成立），`limit <= 3` 为真，于是 `text[:-1]` 把"负索引"当成了"从末尾数"。

这正是 `clip` 里要显式写 `if limit <= 0: return ""` 的原因——**`middle` 缺了这个守卫**。实际调用点传的都是终端宽度（正数），所以没暴露。

**备注四：`middle(text, 4)` 返回 `'...J'` 也值得注意。**

`left = (4-3)//2 = 0`，`right = 4-3-0 = 1`，结果是 `'' + '...' + 'J'`。**头部内容一个字符都不留**，只剩最后一个字符。对"路径截断"这种场景是合理的（保尾部），但对"看开头就知道是什么"的内容就反了。


### 5.3 `build`（54–100）—— 唯一的 IO 与失败处理

**备注一：`git()` 是个"吞掉一切"的包装。**

```python
def git(args, fallback=""):
    try:
        result = subprocess.run(["git", *args], cwd=cwd, capture_output=True,
                                text=True, check=True, timeout=5)
        return result.stdout.strip() or fallback
    except Exception:
        return fallback
```

`except Exception` **捕获所有异常**：`FileNotFoundError`（没装 git）、`CalledProcessError`（不是仓库 / 命令失败）、`TimeoutExpired`（git 卡住 5 秒）。

**为什么这么写？** 因为 pico 要能在**任何目录**启动——包括不是 git 仓库的目录、没装 git 的机器。快照是"锦上添花"，不该因为它失败就让 agent 起不来。

实测在非 git 目录下：

```
branch        : '-'
default_branch: 'main'
status        : 'clean'
recent_commits: []
project_docs  : ['README.md']
```

**五个字段全部有可用的占位值，agent 正常工作。**

**代价：三种失败无法区分。** "这里不是 git 仓库"和"git 挂了 5 秒超时"会得到完全一样的结果，而后者是需要用户知道的（仓库很大或磁盘有问题）。日志里没有任何痕迹。

**备注二：`default_branch` 那行是个内联 lambda。**

```python
default_branch=(
    lambda branch: branch[len("origin/"):] if branch.startswith("origin/") else branch
)(git(["symbolic-ref", "--short", "refs/remotes/origin/HEAD"], "origin/main") or "origin/main")
```

`git symbolic-ref --short refs/remotes/origin/HEAD` 的输出形如 `origin/main`，需要剥掉 `origin/` 前缀。用立即执行的 lambda 而不是单独函数，是为了不引入一个只在这里用一次的名字。

**三重兜底**：git 失败 → fallback `"origin/main"` → 仍然带 `origin/` 前缀 → lambda 剥掉 → `"main"`。所以最坏情况得到 `"main"`。

如果远端 HEAD 指向的不是 main（比如 `origin/master`），会正确得到 `"master"`。**这个链路是对的。**

**备注三：文档扫描在 cwd 不属于 repo_root 时会崩。**

```python
key = str(path.relative_to(repo_root))
```

实测：

```
cwd=<tmp>/a（有 README.md），repo_root=<tmp>/b
-> ValueError: '...\a\README.md' is not in the subpath of '...\b'
```

正常使用不会发生（`repo_root` 是从 `cwd` 用 `git rev-parse` 推出来的，`cwd` 必然在它下面）。但 `repo_root_override` 是公开参数——**评测 harness 就用它把工作区指到临时目录**。如果传了一个不包含 `cwd` 的路径，这里会直接抛 `ValueError`，而且**没有 try 包着**（不像 git 调用那样静默降级）。

这是一个"约定靠调用方遵守、代码不校验"的地方。

**备注四：`clip(..., 1200)` 和 `clip(..., 1500)` 是两个不同的文档/状态预算。**

| 内容                            | 上限   |
| ----------------------------- | ---- |
| 每个文档（`DOC_NAMES` 里最多 4×2=8 个） | 1200 |
| `git status --short`          | 1500 |

**理论上限**：8 × (1200 + 37) + 1500 + 37 ≈ 11400 字符。**这份快照最终会进 prompt prefix**，而 prefix 的预算是 3600、底线 900。所以**一份大仓库的快照能把 prefix 撑爆**——这正是 `context_manager` 里 prefix 段被 `_tail_clip` 处理的原因。

### 5.4 `text()`（102–120）

**备注：它把快照拍平成 YAML-ish 的文本。**

```python
commits = "\n".join(f"- {line}" for line in self.recent_commits) or "- none"
docs = "\n".join(f"- {path}\n{snippet}" for path, snippet in self.project_docs.items()) or "- none"
```

用 `- ` 前缀 + 缩进的格式，和 `memory.render_memory_text` / `checkpoint.render_checkpoint_text` 是同一套风格。**项目里三个"渲染给模型看的文本"的函数都用了这个格式**——统一风格对模型理解结构有帮助。

`or "- none"` 保证空值也占一行，**结构稳定比内容完整重要**。

**注意 `project_docs` 的渲染**：`- {path}` 单独一行，内容另起一段（不缩进）。所以文档内容里的 `- ` 开头行会和路径列表项**混在同一层级**。如果某个 README 里有 markdown 列表，渲染出来是这样的：

```
- project_docs:
- README.md
# 标题
- 这是 README 里的一个列表项     ← 看起来像是 project_docs 的直接子项
```

**这是一个真实的格式歧义**，但因为 prefix 是给模型看的（不是给解析器看的），影响有限。


### 5.5 `fingerprint()`（122–134）

```python
payload = {
    "cwd": ..., "repo_root": ..., "branch": ..., "default_branch": ...,
    "status": ..., "recent_commits": list(...), "project_docs": dict(...),
}
return hashlib.sha256(json.dumps(payload, sort_keys=True).encode("utf-8")).hexdigest()
```

**备注一：`sort_keys=True` 是必须的。**

字典在 Python 里保序，但 `payload` 是字面量构造的，顺序本来稳定。**`sort_keys` 真正的价值是覆盖嵌套层**——`project_docs` 里的 key 顺序取决于扫描顺序，`sort_keys=True` 保证它不影响指纹。

**如果没有这个参数，两次内容相同的扫描可能算出不同的指纹**，导致 prefix 缓存无谓失效。

**备注二：它把 `status` 全文纳入指纹。**

`status` 是 `git status --short` 的输出，**任何文件的修改都会改变它**。所以"改了一个文件"就足以让指纹变化 → 触发 prefix 重建。

这是刻意的（122–124 行注释）：

> 这个指纹用来判断仓库状态是否发生了足够大的变化，从而决定是否需要重建缓存中的 prompt prefix。

**代价**：agent 自己在工作区里改文件（这正是它的主要工作）会不断让 prefix 缓存失效。实测在 400 个改动文件的仓库里，`status` 是 4688 字符（被 clip 到 1526），指纹每次都会变。

**备注三：`cwd` / `repo_root` 必须是 `str`，构造函数不做保证。**

实测：

```
传 str  : sha256 正常
传 Path : TypeError: Object of type WindowsPath is not JSON serializable
```

`build()` 内部传的是 `str(cwd)` / `str(repo_root)`，所以正常路径没问题。但直接 `WorkspaceContext(cwd=Path(...), ...)` 就会在 `fingerprint()` 时才炸——**错误发生在离原因很远的地方**。

**备注四：`text()` 和 `fingerprint()` 用的字段集完全相同，但没有任何机制保证它们同步。**

加了新字段（比如 `remote_url`）如果只加进 `text()` 忘了加进 `fingerprint()`，会出现"**这个字段变了但缓存不失效**"的静默错误——模型看到的是旧值。

---

## 6. 设计理由

### 6.1 为什么快照要"小而稳定"？

**好处**：它进 prompt 的 **prefix 段**，而 prefix 的作用是**稳定基线**——每轮都在同一位置、内容基本不变。这样模型的注意力有一个固定的锚点，而且 prefix 缓存（`refresh_prefix` 里那套 `workspace_changed` / `prefix_changed` 判断）才有意义。

**代价**：只有 4 个白名单文件 + 5 条提交 + git status。**agent 对仓库的理解非常浅**，要靠后续的 `list_files` / `search` / `read_file` 补。

**取舍判断**：对的。预加载整个仓库会让 prefix 变得又大又不稳定，得不偿失。

### 6.2 为什么 git 失败要静默降级？

**好处**：agent 能在任意目录启动。非 git 目录、没装 git、git 超时——三种情况都能跑。

**代价**：**用户无法知道快照是"真的干净"还是"没读到"**。`status: clean` 在"仓库确实没改动"和"git 命令挂了"两种情况下是同一个字符串。

**取舍判断**：对本地开发工具合理，但**至少该在 metadata 里留个标记**（比如 `git_available: false`）。现在的设计把"没有信息"伪装成了"信息是空的"。

### 6.3 为什么文档只扫 `repo_root` 和 `cwd` 两层？

**好处**：`rglob` 全仓库找 `README.md` 在 monorepo 里会扫出几十个，全部塞进 prefix 就是灾难。只扫两个点，最多 8 个文件。

**代价**：`repo/pkg-a/README.md` 在 `repo_root` 启动时**看不到**。

**取舍判断**：合理，而且和 §6.1 的"小而稳定"一致。代价是这个限制**没有写进 docstring**——用户不会知道子包文档被忽略了。

### 6.4 为什么 `clip` 要说"truncated N chars"？

**好处**：模型知道自己漏了什么，而且知道**漏了多少**——这决定了它是"再读一次"（漏了几百字符）还是"换个工具"（漏了几万字符）。这个信息对 agent 的决策质量有直接影响。

**代价**：结果长度超出 limit，调用方无法用 `len(result) <= limit` 做断言。**项目里三个裁剪函数语义各不相同，这是个持续的认知负担。**

**取舍判断**：信息量赢了。但**应该在文档里写清楚"返回长度可能超过 limit"**——现在只能靠读实现发现。

### 6.5 为什么 `middle` 只给 CLI 用？

**好处**：终端宽度是固定的，需要**严格**控制在 N 列内。`clip` 那种"超出 limit"的行为会把终端表格撑破。而且单行显示需要把换行压平。

**代价**：项目里出现了第三套裁剪语义。

**取舍判断**：需求确实不同（一个是给模型看的内容裁剪，一个是给终端看的显示裁剪），**但两者共用一个模块、命名风格相似（`clip` / `middle`），容易被拿错**。如果 `middle` 挪到 `cli.py` 里，这个风险就消失了。

---

## 7. 实测验证

### 7.1 `clip` 与 `middle` 对比

```
MAX_TOOL_OUTPUT = 4000   MAX_HISTORY = 12000
clip(t, 100)   -> 长度 126   结尾 'xxxx\n...[truncated 1909 chars]'
middle(t, 100) -> 长度 100   内容保留头尾
```

### 7.2 `middle` 压平换行

```
输入 : '第一行内容\n第二行内容\n第三行内容'
输出 : '第一行内容 第二行内容 第三行内容'
```

### 7.3 `middle` 边界值穷举

| limit  | 结果                | 说明                 |
| ------ | ----------------- | ------------------ |
| 100    | `'ABCDEFGHIJ'`    | 原文没超               |
| 10     | `'ABCDEFGHIJ'`    | 恰好等于               |
| 4      | `'...J'`          | left=0，头部一个字符不留    |
| 3      | `'ABC'`           | 放不下省略号，硬切          |
| 2      | `'AB'`            | ——                 |
| 0      | `''`              | ——                 |
| **-1** | **`'ABCDEFGHI'`** | **负数变成"去掉最后 1 个"** |

### 7.4 非 git 目录的降级值

```
branch='-'  default_branch='main'  status='clean'  recent_commits=[]  project_docs=['README.md']
```

### 7.5 `build` 在 cwd 不属于 repo_root 时崩溃

```
cwd=<tmp>/a, repo_root=<tmp>/b
-> ValueError: '...\a\README.md' is not in the subpath of '...\b'
```

### 7.6 文档扫描的双根与去重

```
在 repo/pkg 下启动 -> ['README.md', 'pkg\\README.md', 'pkg\\package.json']
```

### 7.7 `status` 名义上限 1500，实际超出

```
git status --short 原始长度: 4688
ctx.status 长度     : 1526   结尾 '...[truncated 3188 chars]'
```

### 7.8 `fingerprint` 的输入类型要求

```
传 str  : 64 位十六进制 ✓
传 Path : TypeError: Object of type WindowsPath is not JSON serializable
```

同一个对象两次调用结果一致；重新 `build()` 一次结果也一致（内容没变时）。

---

## 8. 不变量

| #   | 不变量                                                 | 验证方式              | 实测                        |
| --- | --------------------------------------------------- | ----------------- | ------------------------- |
| I1  | `clip` 结果以 `...[truncated N chars]` 结尾（当超限时）        | 直接看               | ✓                         |
| I2  | `len(clip(x, n)) <= n`                              | 拿长文本试             | ✗ **被推翻**，n=1500 时输出 1526 |
| I3  | `middle(text, limit)` 长度满足 `len ≤ limit`（limit ≥ 0） | 穷举                | ✓                         |
| I4  | `middle` 输入含换行时输出不含换行                               | 直接试               | ✓                         |
| I5  | `middle(text, 0) == ''`                             | 直接试               | ✓                         |
| I6  | `middle` 对负数 limit 也返回空                             | 试 -1              | ✗ **被推翻**，返回 `text[:-1]`  |
| I7  | `build()` 在任意目录都能返回对象                               | 非 git 目录          | ✓                         |
| I8  | `build()` 对任意 `repo_root_override` 都安全              | cwd 不在 override 下 | ✗ **被推翻**，`ValueError`    |
| I9  | `fingerprint()` 是纯函数                                | 两次调用比对            | ✓                         |
| I10 | `fingerprint()` 接受任意路径类型                            | 传 `Path`          | ✗ **被推翻**，`TypeError`     |
| I11 | `text()` 与 `fingerprint()` 覆盖同一字段集                  | 静态阅读              | ✓（7 个字段都覆盖，但无机制保证）        |

---

## 9. 精读顺序建议

1. **`MAX_TOOL_OUTPUT` / `MAX_HISTORY`（14–15）** —— 先看这两个常量的**读者**（`grep -rn MAX_HISTORY pico/`）。发现 `MAX_HISTORY` 在本文件里根本没用，是给 `runtime.py` 的。这一步建立"这个文件是别人的工具箱"的印象。
2. **`clip`（26–30）** —— 5 行。注意它**故意让结果超出 limit**。
3. **`middle`（33–41）** —— 9 行。注意第一行压平换行、`limit <= 3` 的硬切、以及**缺 `limit <= 0` 守卫**。
4. **`build`（54–100）** —— 唯一有 IO 的。重点看 `git()` 那个 `except Exception`，和文档扫描里 `relative_to` 没有保护这件事。
5. **`text()`（102–120）** —— 看它的输出格式，和 `memory` / `checkpoint` 里两个渲染函数对照。
6. **`fingerprint()`（122–134）** —— 13 行。注意 `sort_keys=True` 的必要性，和它要求 `str` 类型这件事。

跳过的：`now()`、`IGNORED_PATH_NAMES`（看集合内容就够了）。

---

## 10. 已知粗糙处

按约定**只报告，不修改**。

### 10.1 项目里三套裁剪语义，命名又很像

| 函数           | 位置                   | 留哪 | 结果长度           |
| ------------ | -------------------- | -- | -------------- |
| `clip`       | 本文件                  | 头  | **超出 limit**   |
| `middle`     | 本文件                  | 头尾 | 等于 limit（负数除外） |
| `_tail_clip` | `context_manager.py` | 头  | 等于 limit       |

`clip` 和 `_tail_clip` 名字都在说"裁"，行为却一个超限一个严格。**跨模块读代码时很容易拿错。**

### 10.2 `middle` 缺 `limit <= 0` 守卫

实测 `middle(x, -1)` 返回 `x[:-1]`。`clip` 有 `if limit <= 0: return ""`，`middle` 没有。当前调用点都传终端宽度（正数），所以没暴露。

### 10.3 `build()` 的 `relative_to` 没有保护

`key = str(path.relative_to(repo_root))`（85 行）在 cwd 不属于 repo_root 时抛 `ValueError`，而 `repo_root_override` 是公开参数。

### 10.4 `git()` 吞掉所有异常，无法区分失败原因

"不是 git 仓库"、"没装 git"、"git 超时 5 秒"三种情况得到完全相同的降级值。而 `status: clean` 会把"没读到"伪装成"没有改动"。

### 10.5 `middle` 压平换行（对非终端用途危险）

`replace("\n", " ")` 是给单行终端显示用的。这个函数放在通用工具模块里，被误用到工具输出上会破坏结构。当前调用点全在 `cli.py` 显示层，所以安全——**但这是靠"没人误用"而不是靠设计保证的**。

### 10.6 `fingerprint()` 不强制 `str` 类型

构造函数把 `cwd` / `repo_root` 原样存，`build()` 里 `str()` 过一次。直接构造传 `Path` 会在 `fingerprint()` 时才炸 `TypeError`。

### 10.7 `text()` 与 `fingerprint()` 的字段集靠人工同步

7 个字段两处都要写。新增字段漏掉 `fingerprint()` 会导致"字段变了但缓存不失效"的静默错误。

### 10.8 `project_docs` 的渲染结构有歧义

`- {path}` 单独一行、内容另起一段不缩进，导致文档内容里的 markdown 列表项和路径列表项在同一层级。

### 10.9 `clip` 的默认参数隐藏了耦合

`MAX_TOOL_OUTPUT = 4000` 作为默认值，意味着 `tool_executor.py` 那句裸 `clip(...)` 的上限**由本文件的常量决定**，而调用点看不出来。

### 10.10 `build()` 每次都跑 5 次子进程，没有缓存

`refresh_prefix` 每条路径都会调 `build`，每次都重新跑 5 次 git。虽然每次有 5 秒超时保护，但**正常仓库里这 5 次调用是纯开销**。

---


## 11. 面试问答

**Q：agent 启动时怎么知道自己在什么仓库里？**

> 起一个"仓库快照"：当前目录、仓库根、当前分支、默认分支、`git status` 的简版输出、最近 5 条提交，再加上几个白名单文档的内容——`README.md`、`AGENTS.md`、`pyproject.toml`、`package.json`。

> 只扫两个位置：仓库根和当前目录。**在子目录启动时也能看到那一层的文档**，但用相对仓库根的路径做 key 去重，所以同一份不会收两遍。

**Q：这个快照为什么要"小而稳定"？**

> 因为它进 prompt 的**最前面那一小段**，作用是"稳定的基线上下文"。

> 稳定有两个好处。一是模型有个固定的锚点，二是**前端可以缓存它**——我在 runtime 里算一个指纹，指纹没变就不重建这段 prompt。如果把整个仓库预加载进来，这段会又大又每轮都变，缓存就没有意义了。

**Q：git 不可用怎么办？**

> 静默降级。`git branch` 失败就用 `-`，`status` 失败就用 `clean`，提交列表失败就是空列表。**agent 在非 git 目录、甚至没装 git 的机器上都能跑。**

**但这个设计有个我不满意的地方**：`clean` 这个词同时表示"仓库确实干净"和"我没读到 git 状态"，两件事被混成了一个值。**更好的做法是加一个标记字段说明 git 到底可不可用**，现在的写法是把"没有信息"伪装成了"信息是空的"。

**Q：`clip` 和 `middle` 有什么区别？**

> 两个都是把长文本截短，但用途完全不同。

> `clip` 是**给模型看的**：保留开头，然后在末尾**明确写上"截掉了多少字符"**。这句话很重要——模型看到 "truncated 1909 chars" 就知道还有东西没看到，能自己判断是该重新读还是换个方式取。**代价是它的结果会超出给定的上限**，因为那句提示本身也要占字符。

> `middle` 是**给终端看的**：保留头和尾，中间用省略号，长度严格控制在 N 列以内，而且**首先把换行全换成空格**——因为是单行显示。

**这两个函数的差别大到不应该放在一起看名字猜**。我承认 `clip` 和另一个模块里的 `tail_clip` 命名太像了，行为却一个超限一个严格。

**Q：为什么文档只扫两个目录？**

> 因为在 monorepo 里全仓库找 `README.md` 能找出几十个，全塞进 prompt 就是灾难。扫仓库根加当前目录，最多 8 个文件，是可控的。

**不过这个限制没有写进文档字符串**，用户不会知道 `packages/foo/README.md` 被忽略了。

**Q：指纹里为什么包含 `git status`？**

> 因为"仓库里有哪些文件被改动了"是判断"这段 prompt 还能不能复用"的核心信号。文件一变，`status` 就变，指纹就变，缓存就失效。

**但这个选择有个副作用**：agent 的主要工作就是改文件，所以它一干活，缓存就失效。**指纹有效地退化成"每次工具调用后都失效"**。如果要优化，得把指纹和"prefix 实际用到的字段"对齐——现在的指纹比 prefix 需要的信号更敏感。

---

## 附：实测脚手架

`workspace.py` 只有标准库依赖（无相对导入），可以用 `importlib` 单文件加载，但既然测试其他三个文件时已经把手项目根加进 `sys.path` 了，统一按包导入更省事：

```python
import sys, tempfile
from pathlib import Path
sys.path.insert(0, r"E:\pico_agentharness\pico")
from pico import workspace as ws

tmp = Path(tempfile.mkdtemp())
plain = tmp / "plain"; plain.mkdir()
(plain / "README.md").write_text("# Hello", encoding="utf-8")
ctx = ws.WorkspaceContext.build(plain)          # 非 git 目录，验证全部降级值
```

**验证 `relative_to` 崩溃**：造两个平级目录 `a` 和 `b`，在 `a` 里放 `README.md`，然后 `build(a, repo_root_override=b)`。

**验证 `fingerprint` 的类型要求**：直接构造 `WorkspaceContext(root, root, ...)` 传 `Path` 而非 `str`。

**注意 shell 转义**：在 Git Bash 的 heredoc 里写含 `\n`、`\s` 的字符串会被 MSYS 的路径转换规则篡改（实测把 `\n` 变成了 `/n`）。**涉及正则或含反斜杠的探针，写成临时 `.py` 文件再跑**，不要用 heredoc。

Windows 托管 Python：`C:/Users/yxqyx/.workbuddy/binaries/python/versions/3.13.12/python.exe`

