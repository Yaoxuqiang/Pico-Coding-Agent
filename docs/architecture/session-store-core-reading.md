# session_store.py 核心逻辑精读（提炼版）

> 对象：`pico/session_store.py`（33 行）  
> 定位：提炼版。讲骨架、设计理由、实测证据，不逐行走读。  
> 所有数字均为本机实测（Windows + Python 3.13.12），脚手架见文末。



---

## 1. 一句话概括

**33 行、1 个类、4 个方法。整个会话持久化层只做一件事：把一个 Python 字典按 JSON 存到磁盘、再读回来。**

真正承载逻辑的代码只有 4 行：`path` 的文件名拼接、`save` 的序列化与写入、`load` 的读取解析、`latest` 的排序取值。其余全是空行、导入和 docstring。

**它不是数据库，不是 ORM，也不是带索引的存储层。它是"一个字典 ↔ 一个文件"的最薄封装。**

---

## 2. 行区间分布

| 行区间   | 内容                                         | 行数 | 承担什么                 |
| ----- | ------------------------------------------ | -- | -------------------- |
| 1     | 模块 docstring                               | 1  | ——                   |
| 3–4   | `import json` / `from pathlib import Path` | 2  | ——                   |
| 7     | `class SessionStore:`                      | 1  | ——                   |
| 8–10  | `__init__`                                 | 3  | 建目录，把 root 转成 `Path` |
| 12–13 | `path`                                     | 2  | **唯一的位置决策点**         |
| 15–18 | `save`                                     | 4  | 序列化 + 覆盖写            |
| 20–21 | `load`                                     | 2  | 读取 + 解析              |
| 23–25 | `latest`                                   | 3  | 找最近一次会话              |
| 空行    | 2, 5, 6, 11, 14, 19, 22                    | 7  | ——                   |

按"每个方法几行"看：`__init__` 3 行、`path` 2 行、`save` 4 行、`load` 2 行、`latest` 3 行。

**唯一带算法的是 `latest()`**（排序取最大）。其余三个都是"一行调用标准库"。所以读这个文件的时间应该花在 `latest()` 的边界条件上，以及**为什么敢写得这么薄**。

---

## 3. 必须分清的概念

这几处是本文件最容易读混的地方。

### 3.1 `session_id`（参数）与 `session["id"]`（字典字段）

```python
def save(self, session):
    path = self.path(session["id"])          # 用的是字典里的 id
def load(self, session_id):
    return json.loads(self.path(session_id)...)   # 用的是传进来的参数
```

**两个不同的东西，靠"调用方保证它们相等"来维系。**

- 新建会话：`runtime.py:90` 生成 `id`，写进 session，`save` 用它当文件名。二者天然一致。
- 恢复会话：`from_session` 先用 `session_id` 找到文件，读出来的字典里 `"id"` 也就是这个值。一致。

**但一旦有人改了 `session["id"]`，`save` 就会写到新文件，旧文件原地残留。**

实测：

```
s3 = {"id": "name-A", "history": []}
store.save(s3)          # 写出 name-A.json
s3["id"] = "name-B"     # 只改内存
store.save(s3)          # 写出 name-B.json
→ 目录内容: ['name-A.json', 'name-B.json']   旧文件残留
```

### 3.2 `path`（方法名）与 `Path`（类型名）

同一个文件里，`path` 是小写的**方法名**，`Path` 是大写的**标准库类型**。`latest()` 的 `lambda path: path.stat().st_mtime` 里，`path` 又是**循环变量**。三处 `path` 三个意思，读的时候要注意作用域。

### 3.3 `root` 是 sessions 目录，不是 workspace root

`SessionStore` 的 `root` 指向 `<workspace>/.pico/sessions/`，不是项目根。装配点在 `cli.py:246`：

```python
store = SessionStore(workspace.repo_root + "/.pico/sessions")
```

`.pico` 是 pico 的私有目录，会被工具的目录扫描显式排除（`search` 的 Python 兜底实现里有这条）。所以**会话文件不会污染模型看到的文件列表**——这是 `SessionStore` 敢把 JSON 直接扔在工作区里的前提。

### 3.4 `latest()` 的"最新"是 mtime 最新

不是文件名里的时间戳，不是 `created_at` 字段，不是 `history` 的最后一条。就是**文件系统上的修改时间**。

这个区别在"文件被外部动过"时会暴露（见 §10.4）。

### 3.5 `save()` 返回的是 `Path` 对象

不是 bool，不是错误码。调用方直接当路径用：`agent.session_path = agent.session_store.save(agent.session)`。

**没有任何返回值能表示"保存失败了"——失败就是抛异常。**

---

## 4. 主流程骨架

三条路径，共用同一个 `path()`：

```
save(session)
  ├─ path(session["id"])                       # 决定写到哪
  ├─ json.dumps(session, indent=2)             # 全量序列化
  └─ path.write_text(..., encoding="utf-8")    # 覆盖写，非原子
     └─ return path

load(session_id)
  └─ json.loads(path(session_id).read_text(encoding="utf-8"))
     # 没有 try，没有兜底

latest()
  ├─ root.glob("*.json")                        # 只按扩展名筛选
  ├─ sorted(key=st_mtime)                       # 按修改时间排
  └─ files[-1].stem if files else None          # 取最后一个的文件名（不含 .json）
```

**顺序上刻意的设计只有一处**：`save` 先算 `path` 再 `dumps`，不是先 `dumps` 再算 `path`。看起来无所谓，但它意味着**文件位置的决策和内容的序列化是解耦的**——将来要换存储位置，只需要动 `path`。

### 落盘时机（跨模块）

`save` 本身不知道什么时候被调用。实际触发点有两类：

| 时机              | 位置                  |
| --------------- | ------------------- |
| 构造 Pico 对象时     | `runtime.py:106`    |
| 每建一个 checkpoint | `checkpoint.py:174` |

checkpoint 的触发点有 7 种（`agent_loop.py`）：`model_error`、`run_finished`、`freshness_mismatch`、`workspace_mismatch`、`context_reduction`、`tool_executed`、`run_stopped`。

**其中 `tool_executed` 是高频的**——每执行一次工具就存一次盘。所以"全量重写"不是每次对话一次，而是**每个工具调用一次**。

---

## 5. 核心函数逐个拆

### 5.1 `__init__`（8–10）

```python
def __init__(self, root):
    self.root = Path(root)
    self.root.mkdir(parents=True, exist_ok=True)
```

**备注：构造函数有文件系统副作用。**

`mkdir(parents=True, exist_ok=True)` 意味着"光是把对象建出来，磁盘上就多了一层目录"。两个参数都不能省：

- `parents=True`：`<ws>/.pico/sessions/` 的两级父目录都可能不存在，要一次建全
- `exist_ok=True`：`load` 场景下目录早就有了，不该报错

**取舍**：这让调用方不用关心"目录在不在"，代价是**构造对象不再是纯内存操作**——单元测试里 `SessionStore(tmp_path)` 这句话就会碰磁盘。所有测试都得传 `tmp_path`，不能传假对象。


### 5.2 `path`（12–13）—— 唯一的位置决策点

```python
def path(self, session_id):
    return self.root / f"{session_id}.json"
```

**备注：全文件最重要的一行。**

`save`、`load` 都走它。改存储位置、改扩展名、加子目录分区，只需要改这里一处。这是**单点收敛**的设计——把"文件放哪"这个决策从三个方法里抽出来，只留一份。

**取舍**：它同时是**唯一没有输入校验的地方**。`session_id` 直接拼进路径字符串，不做任何消毒。

实测：

```
path('../escaped')  → C:\...\pico_ss_xxx\.pico\sessions\..\escaped.json
path('')            → C:\...\pico_ss_xxx\.pico\sessions\.json
path('a/b/c')       → C:\...\pico_ss_xxx\.pico\sessions\a\b\c.json
```

第一条真的会逃出去：

```
session["id"] = "../escaped-session"
store.save(session)
→ 实际落盘: ...\.pico\sessions\..\escaped-session.json
→ 父目录内容: ['escaped-session.json', 'sessions']      ← 文件跑到了 .pico/ 下
```

**是不是漏洞？取决于 `session_id` 从哪来。** 追一下调用链：

- 新建：`runtime.py:90` 用 `时间戳 + uuid4[:6]` 生成，**不可控，不可能含 `..`**
- 恢复：`cli.py:248` 来自 `args.resume`，也就是**用户自己敲的命令行参数**

所以这不是"外部可注入的漏洞"，而是"**用户能把自己的会话文件写到工作区任意位置**"的钝器。危害等级低，但也是真的没有防护。

再看 `path('a/b/c')` 的后果——`save` 会直接抛异常：

```
save 抛异常 -> FileNotFoundError
```

因为 `path()` 只管拼字符串，**不会给子目录做 mkdir**。`__init__` 里建的只有 `root` 本身。所以带斜杠的 `session_id` 在 `save` 时是"要么报错、要么逃逸"，不会静默地建出一棵目录树。


### 5.3 `save`（15–18）

```python
def save(self, session):
    path = self.path(session["id"])
    path.write_text(json.dumps(session, indent=2), encoding="utf-8")
    return path
```

**备注：三行里有三个独立的决策。**

**决策一：`json.dumps` 没有传 `ensure_ascii=False`。**

`json.dumps` 默认 `ensure_ascii=True`，所以所有非 ASCII 字符会被转成 `\uXXXX`。实测磁盘上的真实内容：

```json
{
  "id": "sess-a",
  "history": [
    {
      "role": "user",
      "content": "\u5e2e\u6211\u770b\u4e00\u4e0b src/mod_0.p
```

`帮我看一下` 全部变成了转义序列。

**后果是什么？** 两个：

1. **文件人眼不可读**。想直接 `cat` 会话文件看发生了什么，看到的是转义码。
2. **体积变大**。一个中文字符 UTF-8 占 3 字节，转义成 `\uXXXX` 后占 6 字节。

实测体积（100 条 history，内容含中文）：

| 序列化方式                                  | 字节数    | 相对当前实现    |
| -------------------------------------- | ------ | --------- |
| `indent=2` + `ensure_ascii=True`（当前）   | 43,448 | 100.0%    |
| `indent=2` + `ensure_ascii=False`      | 38,048 | 87.6%     |
| 紧凑 `separators` + `ensure_ascii=False` | 31,292 | **72.0%** |

**但是有一个必须说清的点：这不会导致数据损坏。** `json.loads` 会把 `\uXXXX` 正确还原。实测往返：

```
load 结果 == 原始 session : True
```

所以这是纯粹的**可读性和体积**问题，不是正确性问题。

**决策二：`indent=2`。**

选择可读性，付出体积（和一点点 CPU）。会话文件的主要消费者是**人和调试工具**，不是高频读写的数据库。这个取舍是对的——**但它和 `ensure_ascii=True` 是矛盾的**：一边用 `indent=2` 追求可读，一边把中文转义掉。两个决策各自合理，合在一起互相抵消。

**决策三：`path.write_text()` 直接覆盖，不是原子写。**

`write_text` 的语义是 `open(path, "w")` 然后 `write`。`"w"` 模式会**先把文件截断为 0 字节**，再写入新内容。

这意味着写入过程中存在一个窗口：**文件已经是空的或者只有一半，但旧内容已经没了。**

正确的原子写法是先写临时文件再 `os.replace`，这样任何时刻读者看到的要么是完整旧版本、要么是完整新版本。当前实现没有这层保护。

实测这个窗口（58.6 MB 的会话文件，一个线程持续 `save`，另一个线程持续 `load`）：

```
写入次数 17  读取次数 9
读取失败分布: {'JSONDecodeError': 7}
失败率: 77.78%
循环结束后文件本身可正常解析: True
```

**9 次读取里 7 次拿到的是写了一半的文件。** 注意最后一行——**写完之后文件是完整的**，失败只发生在"写入过程中恰好去读"的窗口里。

在单进程 CLI 场景下这个窗口基本碰不到（没人会在自己写文件时去读）。真正会出事的是两种场景：

1. 两个终端同时 `pico --resume latest`（第二个进程可能读到半截）
2. **写入过程中进程被杀**（Ctrl+C、崩溃、断电）——这时文件永久停留在半截状态，**而旧内容已经被 `"w"` 截断掉了，无法恢复**

这才是"非原子写"最实在的代价：**它不是并发问题，是数据丢失问题。** 一次崩溃 = 整个会话历史归零。

**取舍**：不做原子写换来的是 3 行代码。做成原子写需要"写临时文件 → flush → fsync → os.replace"四步，加上异常清理。对一个本地开发工具来说，写 3 行还是写 12 行，是个真实的取舍；但要说清楚**代价是崩溃时全部丢失，不是性能**。

### 5.4 `load`（20–21）

```python
def load(self, session_id):
    return json.loads(self.path(session_id).read_text(encoding="utf-8"))
```

**备注：一行里串了三个可能抛异常的操作，一个都没接。**

- `path(session_id)` 拼路径
- `.read_text(encoding="utf-8")` 读文件
- `json.loads(...)` 解析

实测两种失败形态：

| 场景    | 抛出的异常                                                                          |
| ----- | ------------------------------------------------------------------------------ |
| 文件不存在 | `FileNotFoundError`                                                            |
| 文件被截断 | `JSONDecodeError: Unterminated string starting at: line 1 column 18 (char 17)` |

**这是刻意的"不接"还是疏忽？** 从调用链看是**刻意的**：

- `cli.py:248` 里 `session_id = args.resume`，用户敲了个不存在的会话名 → 抛 `FileNotFoundError` → CLI 层给用户一个 traceback
- 恢复是**用户显式发起的动作**，失败就该明确失败，不该静默新建一个空会话把用户的历史"顶掉"

如果这里 catch 住返回 `None`，调用方就得判断 `None`，而"读到 None"和"文件损坏"是两种完全不同的状况，迟早会被混在一起处理。**让异常往上抛，是把"怎么处理"的决定权交给最了解上下文的那一层。**

代价是：CLI 目前确实只是把 traceback 打出来，没有任何友好提示。


### 5.5 `latest`（23–25）

```python
def latest(self):
    files = sorted(self.root.glob("*.json"), key=lambda path: path.stat().st_mtime)
    return files[-1].stem if files else None
```

**备注：全文件唯一带算法的三行，也是边界条件最多的三行。**

拆开看：

- `self.root.glob("*.json")` —— **只按扩展名筛选**，不看文件内容
- `sorted(..., key=st_mtime)` —— 按修改时间升序排，数值比较
- `files[-1].stem` —— 取最后一个，`.stem` 去掉 `.json` 后缀还原成 `session_id`
- `if files else None` —— 空目录返回 `None`，这是 `test_session_store.py:20` 覆盖的用例

四处边界条件，逐条实测：

**① 空目录** → 返回 `None` ✓（测试里有覆盖）

**② 目录里混了无关的 `.json`**

```
目录内容: real-session.json, notes.json (内容 {"随便一个 json": true})
latest() -> 'notes'
```

**它把一个跟会话毫无关系的 json 文件当成了"最近一次会话"。** 因为筛选条件只有扩展名。如果 `notes.json` 恰好 mtime 最新，`pico --resume latest` 就会去 `load("notes")`，然后把那个 json 当 session 用。

实际风险：`.pico/sessions/` 是私有目录，正常不会有别的 json。但备份脚本、同步工具、用户手动复制都可能往里放东西。

**③ mtime 并列**

`st_mtime` 是 float 秒，但文件系统的实际时间戳精度有限。实测连续保存 5 个文件：

```
sess-0: st_mtime = 1789547951.3126540
sess-1: st_mtime = 1789547951.3139658
sess-2: st_mtime = 1789547951.3139658     ← 与 sess-1 相同
sess-3: st_mtime = 1789547951.3153200
sess-4: st_mtime = 1789547951.3153200     ← 与 sess-3 相同
唯一 mtime 个数: 3 / 5
```

**5 个文件只有 3 个不同的 mtime。** Python 的 `sorted` 是稳定排序，mtime 相同时**保持输入顺序**，而输入顺序来自 `glob()`——在 Windows 上是目录枚举顺序，**不保证等于创建顺序**。

构造一个能暴露问题的例子（先存 `zzz-first`，再存 `aaa-second`）：

```
latest() -> 'zzz-first'     ← 返回了先保存的那个
```

**为什么实际使用中大概率不会出错？** 因为 session id 的格式是：

```
20260916-164000-a1b2c3
└─ 日期 ─┘└时间┘└uuid┘
```

**时间戳在前，所以字母序 ≈ 时间序。** 当 mtime 并列时，稳定排序保持 glob 顺序（通常是字母序），取最后一个恰好就是时间上最新的那个。

这是一个**隐性的设计依赖**：`latest()` 的正确性不只靠 mtime，还靠"session id 的命名格式恰好是时间可排序的"。改 id 生成规则会静默破坏这个函数。

**④ mtime 会被外部操作改变**

`latest()` 判断"最近"的依据是文件系统时间戳。以下操作都会改 mtime 而不改内容：

- 编辑器里打开并保存
- `touch` 命令 / 备份工具的"恢复时间戳"
- 某些同步/复制工具（不保留元数据的复制会把 mtime 设为复制时间）

反过来说，`save` 每次都重写整个文件，所以**当前活跃的会话永远是 mtime 最新的**——这正是 `latest()` 想要的语义。

**取舍**：用 mtime 换来的是"不依赖文件内容"——不需要解析每个文件就能排序，`latest()` 是 O(文件数) 且不读内容。代价是准确性依赖文件系统，且对目录里的杂质零免疫。

正确做法应该是"读文件里的 `created_at` 或最后一条 history 的时间戳再排序"，但那样 `latest()` 就要解析所有会话文件——对一个纯本地工具，这个取舍是划算的。

---

## 6. 设计理由

每一条都配取舍。

### 6.1 为什么用"一个会话一个文件"，不用数据库？

**好处**：零依赖。pico 的定位是"零第三方依赖的本地代理"，`sqlite3` 虽然是标准库但会引入 schema 管理、迁移、并发锁一整套复杂度。文件方案让人可以**直接打开看**、直接拷走、直接删。

**代价**：没有查询能力。想找"标题含 X 的会话"只能遍历目录。而且**没有索引文件**——目录列表就是会话列表。

### 6.2 为什么 `session_id` 就是文件名？

**好处**：`load` 不需要搜索，一次路径拼接直达。`latest()` 也不需要读文件内容。

**代价**：id 的取值空间被文件系统规则约束了。带 `/`、`\`、`:`、`*` 的字符都不能用；大小写不敏感的文件系统上 `ABC` 和 `abc` 会指向同一个文件。而且正如 §5.2 实测的，**没有任何一层做校验**。

### 6.3 为什么全量重写，不做增量？

**好处**：`save` 永远是无状态的——拿到完整 session，写出完整文件。没有"要写哪几行"的记账逻辑，也就没有"记账和实际不一致"的 bug。崩溃时文件要么是旧的要么是新的（在原子写的前提下）。

**代价**：每次保存都是 O(总历史长度)。实测成本：

| history 条数 | 文件字节    | 单次 save 耗时  |
| ---------- | ------- | ----------- |
| 1          | 339     | 0.47 ms     |
| 10         | 3,004   | 0.37 ms     |
| 100        | 29,645  | 0.76 ms     |
| 500        | 148,045 | 1.06 ms     |
| 2,000      | 592,046 | **3.27 ms** |

**结论要说清楚：全量重写在当前规模下不是性能问题。** 2000 条历史（592 KB）单次 3.27 ms，即使每执行一次工具就保存一次，这个开销也是毫秒级、可忽略的。

**所以真正的风险不是性能，是 §5.3 说的原子性。** 不该拿"性能"当理由批评它。

### 6.4 为什么用 mtime 判断"最近"？

**好处**：不解析文件内容就能排序。`latest()` 不读任何 JSON。

**代价**：§5.5 实测的三条——不校验内容、mtime 并列、外部操作会改 mtime。

### 6.5 为什么没有 schema 版本号？

**好处**：文件格式就是内存字典的直接投影，改字段不需要改存储层。

**代价**：旧会话文件缺字段时，`load` 出来的字典是"残缺"的。

**这个代价由 runtime 层承接了**——`runtime.py:132` 的 `_ensure_session_shape`：

```python
self.session.setdefault("history", [])
self.session.setdefault("memory", memorylib.default_memory_state())
checkpoints = self.session.setdefault("checkpoints", {})
```

**这是一处重要的跨模块分工**：

| 层                               | 职责                          |
| ------------------------------- | --------------------------- |
| `session_store`                 | 只管"字典 ↔ 文件"，不关心字典长什么样       |
| `runtime._ensure_session_shape` | 用 `setdefault` 补全缺失字段，充当迁移层 |

**分工的代价**：迁移逻辑是"隐式向前兼容"而不是"显式版本升级"。如果某个字段的**语义**变了（不只是缺失），`setdefault` 救不了——它只能补空缺，不能改含义。

### 6.6 为什么没有锁？

**假设**：同一时刻只有一个人在用一个工作区。

pico 是本地 CLI 工具，这个假设基本成立。但假设不成立时（两个终端、并行评测），后果是 §5.3 实测的相互读到半截。

**取舍**：加锁需要跨平台的文件锁（Windows 的 `msvcrt.locking` 和 POSIX 的 `fcntl.flock` 完全不同），代码量会翻几倍。对一个开发工具，这个取舍合理——但要**写明假设**，而不是当它不存在。

---

## 7. 实测验证

全部用 `importlib` 直接加载单文件 + `tempfile` 隔离，不碰真实工作区。

### 7.1 数据完整性类

| # | 实验                       | 结果                                                             |
| - | ------------------------ | -------------------------------------------------------------- |
| 1 | `save` 后磁盘原文             | 中文被转义成 `\uXXXX`，`"帮我看一下"` → `"\u5e2e\u6211\u770b\u4e00\u4e0b"` |
| 2 | 体积对比（100 条中文 history）    | 43,448 / 38,048 / 31,292 字节（→ 87.6% / 72.0%）                   |
| 3 | 往返一致性                    | `load(save(s)) == s` 为 **True**                                |
| 9 | 改 `session["id"]` 再 save | 目录里出现 `['name-A.json', 'name-B.json']`，旧文件残留                   |

### 7.2 路径安全类

| # | 实验                                  | 结果                                                     |
| - | ----------------------------------- | ------------------------------------------------------ |
| 4 | `session_id = "../escaped-session"` | 落盘到 `.pico/../escaped-session.json`，**逃出 sessions 目录** |
| 8 | `session_id = "a/b/c"`              | `save` 抛 `FileNotFoundError`（不补建子目录）                   |

### 7.3 失败形态类

| # | 实验            | 结果                                                                   |
| - | ------------- | -------------------------------------------------------------------- |
| 5 | `load` 不存在的文件 | `FileNotFoundError`                                                  |
| 5 | `load` 被截断的文件 | `JSONDecodeError: Unterminated string starting at: line 1 column 18` |

### 7.4 `latest()` 边界类

| #  | 实验                          | 结果                                    |
| -- | --------------------------- | ------------------------------------- |
| 6  | 空目录                         | `None`                                |
| 6  | 目录含无关 `notes.json`          | `latest() -> 'notes'`（把无关 json 当成会话）  |
| 6  | 目录含被截断的 `broken.json`（最新写入） | `latest() -> 'broken'`（把损坏文件当成最新会话）   |
| 10 | 连续保存 5 个文件                  | 5 个文件只有 **3 个不同 mtime**               |
| 10 | 字母序与创建序相反的两次保存              | `latest() -> 'zzz-first'`（返回了先保存的）    |
| 10 | 上述场景连续调用 20 次               | 结果集合恒为 `{'sess-4'}`（本次恰好 glob 顺序=创建序） |

### 7.5 写入原子性类

| #  | 实验                    | 结果                                               |
| -- | --------------------- | ------------------------------------------------ |
| 7  | save 耗时随 history 增长   | 0.47 / 0.37 / 0.76 / 1.06 / **3.27** ms          |
| 11 | 58.6 MB 文件，一写一读并发 4 秒 | 写入 17 次，读取 9 次，**7 次 `JSONDecodeError`（77.78%）** |
| 11 | 并发结束后再读一次             | 正常解析（说明失败只发生在写入窗口内）                              |

---

## 8. 不变量

可以被输出或文件系统验证的断言。

| #  | 不变量                               | 验证方式          | 实测                                   |
| -- | --------------------------------- | ------------- | ------------------------------------ |
| I1 | `save(s)` 后 `load(s["id"]) == s`  | 往返比较          | ✓ True                               |
| I2 | `save(s)` 的返回值 == `path(s["id"])` | 路径比较          | ✓（测试用例覆盖）                            |
| I3 | 空目录时 `latest()` 返回 `None`         | 直接调用          | ✓                                    |
| I4 | `save` 不修改传入的 session 对象          | 调用前后比较        | ✓（未做局部修改）                            |
| I5 | `__init__` 后 `root` 目录一定存在        | 检查 `is_dir()` | ✓（`parents=True, exist_ok=True`）     |
| I6 | 写入完成后文件一定是合法 JSON                 | 写完立刻 load     | ✓（失败只在写入窗口内）                         |
| I7 | `latest()` 的返回值是某个既定文件的 stem      | 与目录列表对照       | ⚠️ **不成立**——`notes.json` 这类无关文件也会被选中 |

**I7 是唯一被实测推翻的候选不变量。** 它是这个文件里最值得记住的一条边界。

---

## 9. 精读顺序建议

按"从决策点到边界"的顺序，不要从第 1 行往下读。

1. **`path`（12–13）** —— 先看这一行。它决定了后面所有方法的形态：为什么 `save`/`load` 都只有两三行，就是因为位置决策被抽走了。顺便注意它没做校验。
2. **`save`（15–18）** —— 逐字读这一行：`path.write_text(json.dumps(session, indent=2), encoding="utf-8")`。三个决策点（`ensure_ascii` 缺席、`indent=2`、非原子写）全在这一行里。
3. **`load`（20–21）** —— 看它**没写什么**。没有 try/except，这就是它的设计选择。
4. **`latest`（23–25）** —— 唯一有算法的。三行里藏了四个边界条件，逐个想一遍。
5. **`__init__`（8–10）** —— 最后看。它只有一句话：建目录，但这解释了为什么所有测试都得传 `tmp_path`。
6. **回到调用方** —— `cli.py:246–252` 看 id 从哪来，`runtime.py:106` + `checkpoint.py:174` 看落盘时机，`runtime.py:132` 看缺字段怎么补。**这三个点决定了这个文件"薄"得合不合理。**

跳过的部分：`import`、docstring、空行。

---

## 10. 已知粗糙处

按约定**只报告，不修改**。

### 10.1 非原子写（最严重）

`write_text` 的 `"w"` 模式先截断再写。写入中崩溃 = 旧内容已丢、新内容不完整。实测读写并发失败率 77.78%。

改为"临时文件 + `os.replace`"即可，代价是代码从 1 行变 4 行。

### 10.2 `session_id` 无消毒

`path()` 直接字符串拼接。`../` 可逃出 sessions 目录（实测已复现），含 `/` 时报 `FileNotFoundError`。

因 `session_id` 来源是 `uuid` 生成或用户自己的命令行参数，**不是外部注入漏洞**，属于钝器而非利器。

### 10.3 `latest()` 不校验内容

只按 `*.json` 筛选。目录里任何 json 文件都会成为候选（实测：返回了 `notes`）。

### 10.4 `latest()` 在 mtime 并列时依赖 glob 顺序

实测 5 个文件只有 3 个不同 mtime。并列时结果由 `glob()` 的枚举顺序决定。

**当前之所以大体正确，是因为 session id 格式 `日期-时间-uuid` 恰好字母序≈时间序。这是一个未被写下来的隐性依赖。**

### 10.5 `save` 用 `session["id"]` 而非外部传入的 id

改内存里的 id 会写到新文件，旧文件残留，且没有任何清理。

### 10.6 `ensure_ascii=True` 与 `indent=2` 互相抵消

一个为了体积和可读性做缩进，另一个把中文转义得不可读。实测 `ensure_ascii=False` 能同时省 12.4% 体积并让文件可读。

### 10.7 `load` 零错误处理

`FileNotFoundError` 和 `JSONDecodeError` 直接透传到 CLI 层，用户看到的是原始 traceback。判断这是"刻意的让异常冒泡"还是"忘了接"，代码里没有注释说明。

### 10.8 构造函数有文件系统副作用

`SessionStore(root)` 就会建目录。这是"为什么所有测试都得传 `tmp_path`"的原因。

### 10.9 无锁

单进程假设没有被写下来。并行场景下两个进程可以互相覆盖。

### 10.10 `path` / `Path` / `path` 三处同名

方法名 `path`、类型名 `Path`、`latest()` 里的循环变量 `path`。读的时候容易看错作用域。

---


## 11. 面试问答

**Q：会话持久化你怎么做的？**

> 一个会话一个 JSON 文件，文件名就是会话 id，放在工作区的 `.pico/sessions/` 下面。整个存储层三十来行。核心就三个动作：存的时候按 id 拼出路径、把字典序列化写进去；读的时候拼出路径、读出来解析；列"最近一次会话"的时候扫目录、按文件修改时间排个序取最新的那个。

**Q：为什么不用数据库？**

> 因为没必要。这个工具是本地跑的，零第三方依赖是它的定位。用数据库就要引入建表、字段迁移、连接管理这一整套，而我的查询需求只有两个：「按 id 取一个」和「取最近的一个」。前者一次路径拼接就够，后者扫个目录就行。用文件还有个好处是人能直接打开看。

**Q：文件名当 id 有什么限制？**

> 三个。第一，id 里不能有路径分隔符和那些文件系统保留字符，不然会出问题。第二，大小写不敏感的文件系统上，两个只差大小写的 id 会指向同一个文件。第三，**如果 id 里带了 `..`，是可以跑到上层目录去的**——我的实现里没有做校验。不过实际上 id 是用时间戳加 uuid 生成的，用户改不了，恢复会话时虽然命令行能传任意字符串，但那是用户自己的机器，谈不上被攻击。

**Q：每次保存都重写整个文件，不慢吗？**

> 实测不慢。两千条历史记录、接近六百 KB，单次保存三点几毫秒。而且它只在建检查点的时候才存，不是每生成一个 token 就存。**全量重写换来的是无状态——不用记账「这次该写哪几行」，也就不会有记账和实际不一致的 bug。**

**但是有一个比性能重要的问题**：这个写入不是原子的。用的是覆盖写，会先把文件清空再写新内容。如果写到一半进程被杀，旧内容已经没了，新内容又不完整，这个会话就废了。我实测过写入过程中去读，九次里有七次解析失败。**正确的做法是先写临时文件再改名替换，这样任何时刻读到的要么是完整旧版本要么是完整新版本。**

**Q：怎么找"最近一次会话"？**

> 扫目录里所有 json，按文件的修改时间排序，取最新的那个。

**这里有三个我没处理好的边界**：
>
> 1. 只按后缀名筛选，不校验内容。目录里放一个跟会话无关的 json，它就会被当成最新的会话。
> 2. 修改时间会重复。我实测连续存五个文件，只有三个不同的时间戳。时间戳一样的时候，排序结果就取决于目录枚举的顺序，**不确定**。
> 3. 修改时间会被外部操作改——编辑器保存、同步工具、复制文件都可能动它。
>
> 之所以实际用起来没出事，是因为我的会话 id 格式是「日期-时间-随机串」，时间戳在前，**字母序恰好等于时间序**，所以即使排序不稳定也大概率是对的。但这是个隐性的依赖，没有写在任何地方。

**Q：读文件失败怎么办？**

> 不接。文件不存在就抛"文件找不到"，内容损坏就抛"解析失败"，让异常一路冒到命令行那层。

> 这是故意的，因为**恢复会话是用户显式敲的命令**。如果这里悄悄 catch 掉、返回一个空会话，用户会以为自己之前的记录还在，实际上已经被顶掉了。恢复失败就该明确地失败。代价是命令行现在只是把异常打出来，没有友好的提示，这一块可以补。

**Q：多个同时打开的终端会互相覆盖吗？**

> 会，我没有加锁。因为这工具的设计假设是「一个工作区同一时间一个人在用」，本地命令行工具这个假设基本成立。但假设没有被写在代码里，这是个缺陷。真要支持并发，得跨平台的文件锁——Windows 和 Linux 的锁机制完全不一样，代码量会翻几倍。

---

## 附：实测脚手架

```python
import importlib.util, sys, tempfile
from pathlib import Path

spec = importlib.util.spec_from_file_location("ss", r"E:\pico_agentharness\pico\pico\session_store.py")
ss = importlib.util.module_from_spec(spec)
sys.modules["ss"] = ss          # 必须！否则 dataclass 会报 NoneType.__dict__
spec.loader.exec_module(ss)
SessionStore = ss.SessionStore

root = Path(tempfile.mkdtemp(prefix="pico_ss_"))
store = SessionStore(root / ".pico" / "sessions")
```

注：`session_store.py` 本身没有 `@dataclass`，但同一套脚手架复用在 `context_manager.py` 上时必须保留 `sys.modules` 注册，所以统一写成这样。

Windows 托管 Python：`C:/Users/yxqyx/.workbuddy/binaries/python/versions/3.13.12/python.exe`
