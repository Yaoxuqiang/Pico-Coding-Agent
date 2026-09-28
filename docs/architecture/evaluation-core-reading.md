# Pico 评测层核心精读：evaluator.py + metrics.py

> 阅读对象：`pico/evaluation/evaluator.py`（627 行）、`pico/evaluation/metrics.py`（1689 行）  
> 阅读视角：资深 Agent 平台工程师 —— 关注"这套评测能不能被信任、能不能进 CI、出事时能不能 5 分钟内定位"  
> 一句话定位：**evaluator 负责把一次 Agent 运行变成可判定的 pass/fail；metrics 负责把一批运行变成可引用的数字。**



---

## 0. 先看链路：架构设计链路图

### 0.1 分层结构

```
┌──────────────────────────── 编排层 scripts/ ────────────────────────────┐
│  run_large_scale_experiments.py    collect_resume_metrics.py            │
│  run_provider_experiments.py                                            │
│  （argparse → 调用 pico.evaluation.* → 落盘 JSON / Markdown）            │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────┐
│ 数据面 evaluator.py  BenchmarkEvaluator                                 │
│                                                                          │
│  load_benchmark ──► validate_benchmark ──► run_task (× N) ──► summarize  │
│       │                    │                    │                  │     │
│       │  schema_version=1  │ 8 类契约校验        │ 沙箱/四条件        │ 写 JSON│
│       │  tasks 非空列表    │ fixture 存在性      │ verifier 子进程    │       │
│       │                    │ tool 白名单         │ artifact digest    │       │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │ 复用 Pico / WorkspaceContext /
                                 │ SessionStore / RunStore
┌────────────────────────────────▼────────────────────────────────────────┐
│ 度量面 metrics.py                                                       │
│                                                                          │
│  合成消融（零网络）        真实模型实验            恢复消融               │
│  · context 12 配置矩阵     · provider gpt/claude   · 10 任务 × 2 变体    │
│  · memory 12 任务 × 3 变体 · real context/memory   · 4 类 resume 状态    │
│  · security 10 场景        · real security 11 场景 · false_accept 检测   │
│                              │                                          │
│                    聚合 aggregate_*  ──► 渲染 render_* / write_core_rep │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────┐
│ 产物层                                                                  │
│  artifacts/*-v2.json（四层）  .pico/runs/*/{report.json,trace.jsonl}    │
│  docs/metrics/pico-benchmark-core-report.md                             │
└─────────────────────────────────────────────────────────────────────────┘
```

### 0.2 一次任务的执行链路（`BenchmarkEvaluator.run_task`）

```
task(dict)
  │
  ├─1 fixture 隔离    fixture_copy_root = workspace_root/<task_id>/<fixture_name>
  │                   exists ? rmtree : pass  →  copytree   （幂等重跑）
  ├─2 工作区锚定      WorkspaceContext.build(copy_root, repo_root_override=copy_root)
  │                   SessionStore(copy/.pico/sessions)  RunStore(copy/.pico/runs)
  ├─3 模型注入        model_client_factory(task, workspace)  或  FakeModelClient(脚本化输出)
  ├─4 智能体装配      Pico(approval_policy="auto", max_steps=step_budget, allowed_tools=白名单)
  ├─5 场景预置        _apply_task_setup → context_reduction | freshness_mismatch | workspace_mismatch
  ├─6 基线采样        initial_history_empty / memory_empty / task_summary_empty / episodic_notes_empty
  ├─7 执行            final_answer = agent.ask(prompt)
  │                      └─► agent_loop ─► context_manager ─► model ─► tool_executor ─► run_store
  ├─8 结果取证        artifact_exists / artifact_digest / report / task_state
  ├─9 外部裁决        subprocess.run(verifier, cwd=copy_root, shell=True)
  └─10 四条件判定     within_budget ∧ verifier_passed ∧ artifact_exists ∧ non_failure_stop_reason
                      → passed  → _failure_category 定性 → 40+ 字段行
```

### 0.3 消融轴（贯穿全部实验）

| feature flag        | 默认   | 关掉后用于证明        |
| ------------------- | ---- | -------------- |
| `memory`            | True | 跨轮笔记是否减少了重复读文件 |
| `relevant_memory`   | True | 相关召回段是否有独立贡献   |
| `context_reduction` | True | 分层预算裁剪到底省了多少字符 |
| `prompt_cache`      | True | 前缀复用是否真的命中     |

改写方式统一走 `_temporary_feature_flags`（metrics.py:158）：上下文管理器改写 → `try/finally` 还原，**保证同一 agent 上连续跑多组对照不会互相污染**。

---

## 1. 核心设计

### 1.1 三层职责分离（编排 / 数据面 / 度量面）

| 层   | 文件             | 只做一件事                  | 不做的事         |
| --- | -------------- | ---------------------- | ------------ |
| 编排  | `scripts/*.py` | 解析参数、拼路径、落盘            | 不写任何判定逻辑     |
| 数据面 | `evaluator.py` | 跑任务、判 pass/fail、记复现元数据 | 不聚合、不渲染      |
| 度量面 | `metrics.py`   | 组织对照实验、聚合、渲染报告         | 不定义单个任务的判分规则 |

收益：判分口径只有一份（evaluator），换报告格式不需要动判分；`metrics.py` 里再复杂的实验组合也不会把 pass 口径改坏。

### 1.2 契约在前：benchmark 是带 schema 的资产，不是一份 JSON

`validate_benchmark`（evaluator.py:164）在跑任何东西之前做 8 类硬校验：

| 校验                                                                                             | 失败信息                                                  |
| ---------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| 顶层必须是 mapping                                                                                  | `benchmark must be a mapping`                         |
| 必填键 `schema_version`/`tasks`                                                                   | `benchmark is missing required keys: ...`             |
| `schema_version == 1`                                                                          | `unsupported benchmark schema_version`                |
| `tasks` 非空列表                                                                                   | `benchmark tasks must be a non-empty list`            |
| 任务必填 8 键（id/prompt/fixture_repo/allowed_tools/step_budget/expected_artifact/verifier/category） | `benchmark task {id!r} is missing required keys: ...` |
| id 非空 + 全局唯一                                                                                   | `duplicate benchmark task id: ...`                    |
| fixture 目录真实存在                                                                                 | `fixture repo does not exist: ...`                    |
| `allowed_tools` 在白名单内（`legal_tool_names()`）                                                    | `unknown allowed_tools entry: ...`                    |
| `step_budget >= 1`                                                                             | `step_budget must be positive`                        |

校验通过后返回**归一化副本**（`strip()` 字符串、int 化预算、list 化工具），下游不再需要做类型防御。

### 1.3 pass 的唯一定义：四条件 AND

```python
# evaluator.py:500-503
within_budget            = task_state.tool_steps <= int(task["step_budget"])
verifier_passed          = verifier.returncode == 0
non_failure_stop_reason  = task_state.stop_reason == "final_answer_returned"
passed = within_budget and verifier_passed and expected_artifact_exists and non_failure_stop_reason
```

四个条件各自回答一个问题：

| 条件                         | 回答的问题           | 数据来源                                              |
| -------------------------- | --------------- | ------------------------------------------------- |
| `expected_artifact_exists` | 说好的产物到底有没有落盘    | `fixture_copy_root / TASK_FIXTURE_ARTIFACTS[...]` |
| `within_budget`            | 是不是用光了步数硬凑出来的   | `task_state.tool_steps <= step_budget`            |
| `verifier_passed`          | 内容对不对           | 外部 shell 脚本 returncode                            |
| `non_failure_stop_reason`  | 是正常收尾还是崩了/超时/被拒 | `stop_reason == final_answer_returned`            |

### 1.4 失败定性：把 4 个布尔压缩成 1 个可行动标签

`_failure_category`（evaluator.py:550）按**固定优先级**返回单一定性：

```
missing_artifact  >  budget_exceeded  >  verifier_failed  >  failure_stop_reason  >  unknown
```

优先级不是随便排的：外层现象（文件没产出）比内层现象（断言没过）更靠近根因，先报外层能少走一层弯路。`summarize_rows` 再把它聚成 `failure_category_counts`，一眼看出本轮失败集中在哪一类。

### 1.5 复现元数据：让数字"可被追溯"

`run()`（evaluator.py:407）在产物里固定写入：

| 字段                             | 来源                                                    | 作用         |
| ------------------------------ | ----------------------------------------------------- | ---------- |
| `commit_sha` / `branch`        | `_git_value`（超时 5s）                                   | 结果对应哪份代码   |
| `fixture_snapshot_id`          | `_fixture_snapshot_id`（sha256 遍历 fixture 树，路径+内容都入哈希） | 结果对应哪份测试资产 |
| `model_name` / `model_version` | `FakeModelClient` / `scripted-deterministic`          | 结果对应哪个模型   |
| `decoding`                     | temperature=0.0 / top_p=1.0 / max_new_tokens=64       | 解码参数       |
| `timezone` / `locale`          | `Asia/Shanghai` / `locale.setlocale`                  | 环境上下文      |

关键点：**脚本化输出 + 固定 `created_at`（如 `_checkpoint_payload` 写死 `2026-04-15T08:00:00+00:00`）**，把随机性全部挤出去，理论上同一 commit 重跑结果字节级一致。

### 1.6 双轨实验：synthetic 进 CI，real 做背书

`collect_resume_metrics`（metrics.py:1082）用 `experiment_mode` 一个开关切两条轨：

|          | synthetic（默认）                  | real                                       |
| -------- | ------------------------------ | ------------------------------------------ |
| 模型       | `FakeModelClient` + 脚本化/状态机客户端 | `OpenAICompatible` / `AnthropicCompatible` |
| 网络       | 不需要                            | 需要 API key                                 |
| 用途       | 每次提交都能跑、结果确定                   | 证明"换真模型结论不塌"                               |
| provider | 无                              | gpt / claude / deepseek，逐个隔离               |

### 1.7 四层消融 + 一份核心报告

| 实验                 | 入口                          | 规模                         | 产物                           |
| ------------------ | --------------------------- | -------------------------- | ---------------------------- |
| Harness regression | `run_harness_regression_v2` | 12 固定任务                    | `harness-regression-v2.json` |
| Context ablation   | `run_context_ablation_v2`   | 12 配置 × 5 轮                | `context-ablation-v2.json`   |
| Memory ablation    | `run_memory_ablation_v2`    | 12 任务 × 5 轮 × 3 变体 = 180 次 | `memory-ablation-v2.json`    |
| Recovery ablation  | `run_recovery_ablation_v2`  | 10 任务 × 3 轮 × 2 变体 = 60 次  | `recovery-ablation-v2.json`  |

`write_benchmark_core_report`（metrics.py:1618）只读这四份 artifact，产出 `docs/metrics/pico-benchmark-core-report.md`，并且**显式分了两张清单**：

- 「可以安全写进简历的指标」：`avg_prompt_compression_ratio`、`repeated_reads`、`resume_success_rate`、`workspace_drift_detection_rate`、`resume_false_accept_rate` …
- 「只适合放文档/面试展开的指标」：`current_request_preserved_rate`、`memory_hit_rate`、`stale_reanchor_rate`、`failure_category_counts`

外加一段「口径边界」：**Harness regression 只证明 runtime 合同稳定，不证明 provider 上限；context/memory/recovery 三层只证明模块收益，不和 provider benchmark 混写。**

---

## 2. 核心亮点

### 亮点 1：把"模型不确定性"从回归测试里物理摘除

`FakeModelClient` + `SCRIPTED_MODEL_OUTPUTS`（evaluator.py:47）把模型输出写成确定性脚本：

```python
"invalid_patch_recovery": [
    '<tool>{"name":"patch_file","args":{"path":"README.md","old_text":"..."}}</tool>',  # 故意缺 new_text
    '<tool name="patch_file" ...><old_text>...</old_text><new_text>...</new_text></tool>',  # 修正后重试
    "<final>Done.</final>",
]
```

含义：这一条任务**不是在测模型会不会写 patch，而是在测 harness 收到缺字段的参数后会不会正确拒绝、并把错误信息回灌给模型让它自我修正**。回归失败 = harness 有 bug，不会出现"模型今天心情不好"的伪失败。

### 亮点 2：一个任务一个沙箱，且幂等

```python
# evaluator.py:446-450
fixture_copy_root = self.workspace_root / task["id"] / fixture_source.name
if fixture_copy_root.exists():
    shutil.rmtree(fixture_copy_root)
shutil.copytree(fixture_source, fixture_copy_root)
```

- 任务之间零共享路径，天然并行安全；
- `rmtree` 前置让**重复跑同一任务得到干净起点**，不会因为上一轮残留的 `.pico/` 让 resume 判定串味；
- `repo_root_override=fixture_copy_root` 把 `WorkspaceContext` 的边界钉死在副本里 —— 测试写坏文件只脏副本。

### 亮点 3：结果取证与裁决分离

`expected_artifact_exists` 只查"文件在不在"，内容正确性 100% 交给外部 `verifier` 子进程。

```python
# evaluator.py:492-498
verifier = subprocess.run(task["verifier"], cwd=fixture_copy_root, shell=True, capture_output=True, text=True)
```

好处：加新任务不用改 Python 代码，只写一条 shell 断言；而且 `verifier_stdout/stderr` 原样进产物 —— **失败现场自带证据**。

更狠的是后 5 条任务（recovery 3 + durable-contract 2）的 verifier 读的是 `.pico/runs/*/report.json`：

```bash
python3 -c "... report['durable_rejections'] == ['dependency-facts:secret_shaped', 'key-decisions:transient_task_state']; ..."
```

即**验证的是 harness 的内部状态机（该拒的拒了、该收的收了），而不是工作区文件**。评测对象从"代码有没有改对"升级到"治理策略有没有生效"。

### 亮点 4：消融有对照组，不是自说自话

| 证明对象    | 实验组              | 对照组                    | 实际数据                                  |
| ------- | ---------------- | ---------------------- | ------------------------------------- |
| 记忆有用    | `memory_on`      | `memory_off`           | 重复读次数下降（见 `memory-ablation-v2.json`）  |
| 恢复机制有用  | `resume_enabled` | `resume_disabled`      | 对照组 resume_success_rate = 0.0，实验组 0.9 |
| 上下文裁剪有用 | `full`           | `no_context_reduction` | 12 配置矩阵的压缩率                           |


对照组全 0 是这套实验最有说服力的地方：**它排除了"这行代码本来就没用但恰好也过了"的可能**。

### 亮点 5：`_RecoveryScenarioModelClient` —— 用模型当探针

```python
# metrics.py:1274-1287
if all(fragment in prompt_lower for fragment in self.required_fragments):
    return f"<final>{self.success_answer}</final>"
return "<final>missing recovery state.</final>"
```

这个"模型"不生成内容，**只做断言**：把 `["resume status: partial-stale", "stale paths: sample.txt"]` 这类关键片段写成 `required_fragments`，片段全出现在 prompt 里才算恢复成功。

等价于：把"prompt 里到底有没有把恢复上下文塞进去"变成了一个可自动判定的布尔值，比人肉读 prompt 可靠得多。

### 亮点 6：指标层自带"能不能上简历"的元判断

`write_benchmark_core_report` 的最后三段不是技术输出，是**对外口径管理**。它承认了：

- `within_budget_rate` 这类指标会因为实现方式恒真（见坑点 2），所以不放进"可安全引用"清单；
- `memory_hit_rate`、`stale_reanchor_rate` 依赖具体任务构造，换一套任务就变，不适合当卖点；
- provider benchmark 和 harness regression 回答的是两个不同问题，绝不能混着说。

在真实工程里，"知道自己哪些数字能说、哪些不能说"往往比多跑几个实验更重要。

---

## 3. 生产级异常处理与兜底

### 3.1 总原则：契约层 fail-fast，运行时 fail-soft

| 层 | 策略 | 理由 |
|---|---|---|
| 输入契约（benchmark JSON） | **fail-fast**，显式 `ValueError` | 配置写错必须在跑之前炸，不能跑完才告诉你测错了 |
| 运行时采集（git/locale/时间/均值） | **fail-soft**，返回安全默认值 | 一个元数据取不到，不该让整轮实验白跑 |
| 外部依赖（provider） | **隔离 + 降级可观测**，标 `status` | 一个 provider 挂了不能拖死另外两个 |

### 3.2 兜底清单（逐处可查）

| 位置 | 手法 | 兜底值 |
|---|---|---|
| `_git_value` (evaluator.py:107) | `try/except` + `timeout=5` + `check=True` | `""` |
| `_current_locale` (:122) | `try/except` | `locale.getdefaultlocale()[0] or "C"` |
| `_safe_mean` (metrics.py:21) | 空集合判断 | `0.0` |
| `_safe_ratio` (:28) | 分母为 0 判断 | `0.0` |
| `_parse_iso8601` (:34) | `try/except` | `None` |
| `_infer_run_duration_ms` (:71) | **三级降级**：`run_finished.run_duration_ms` → 起止时间差 → `0.0` | `0.0` |
| `aggregate_run_artifacts` (:85) | `report.json` / `trace.jsonl` 不存在则跳过该文件 | 空列表 |
| `summarize_rows` (evaluator.py:245) | `total_tasks` 为 0 时不做除法 | 三个 rate 均 `0.0` |
| `_provider_profile` (metrics.py:680) | 多候选环境变量名依次回退；仍无 key 则**不抛异常** | `{"status": "blocked", "reason": "..."}` |
| `run_provider_experiments` (:751) | 每个 provider 单独 `try/except` | `{"status": "error", "reason": str(exc)}` |
| `_temporary_feature_flags` (:158) | `try/finally` 强制还原 | 原 `feature_flags` |
| `_apply_task_setup` (evaluator.py:301) | `setup` 为空直接 return | 不预置场景 |
| `_normalize_text` (metrics.py:744) | 循环剥离尾部 `.!?"'` | 归一化文本 |
| `run_task` 沙箱 | `rmtree` 前置保证幂等 | 干净起点 |

### 3.3 失败可观测性：错误不吞，全部落进产物

- `verifier_stdout` / `verifier_stderr` / `verifier_exit_code` 原样进 row；
- `security_event_type`（如 `path_escape`）+ `tool_error_code` 进实验 row 并聚合成计数；
- `stop_reason` 9 种取值全部落进 `report.json`，聚合层再出 `stop_reason_counts`；
- provider 失败不抛异常，写成 `status=blocked|error` + `reason`，让报告里能直接看到"哪个 provider 没跑成、为什么"。

### 3.4 明确的兜底缺口（当前实现没有的）

| 缺口 | 位置 | 后果 |
|---|---|---|
| verifier 无 `timeout`、无 `check`、无 `try/except` | evaluator.py:492 | verifier 挂死 → 整轮 benchmark 无限阻塞 |
| 无并发控制 | `run()` 用列表推导串行 | 12 任务串行，real 模式下耗时长 |
| 产物直写非原子 | `_write_artifact` (:567) | 中途崩溃可能留下半截 JSON |
| `run_real_*` 无 provider 级隔离 | metrics.py:852/918/1042 | 真实模型异常会中断整轮，与 `run_provider_experiments` 的策略不一致 |

---

## 4. 排查错误思路

**目标：从一份 artifact 走到一个根因，路径固定。**

### Step 1 —— 先看汇总，锁定失败类别

```bash
python -c "import json;d=json.load(open('benchmarks/results/<run>/harness-regression-v2.json',encoding='utf-8'));print(d['summary'])"
```

看 `summary.failure_category_counts`：空 `{}` = 全过；出现 `verifier_failed` = 结果不对；出现 `missing_artifact` = 产物没落盘。

### Step 2 —— 定位到具体任务，读 verifier 现场

在 `rows[]` 里找 `status == "fail"` 的行，**按这个顺序读字段**：

| 顺序 | 字段 | 读到什么就是什么 |
|---|---|---|
| 1 | `verifier_stderr` | 断言挂在哪一行、期望值 vs 实际值 |
| 2 | `verifier_exit_code` | 非 0 = 断言失败；非 0 且 stderr 是 traceback = verifier 自己写错了 |
| 3 | `stop_reason` | 不是 `final_answer_returned` → 去看是超步数、模型错、还是工具被拒 |
| 4 | `artifact_exists` / `artifact_digest` | False = 根本没写文件；digest 可跨轮比对内容是否变了 |
| 5 | `tool_steps` vs `step_budget` | 步数用满 = 大概率陷入重试循环 |
| 6 | `attempts` | 重试次数异常高 = 模型持续给非法参数 |

### Step 3 —— 进 run 目录看 trace，还原时间线

```
benchmarks/.../rows[].run_dir_relpath/report.json    ← 终态 + prompt_metadata
                                trace.jsonl          ← 9 类事件逐条时间线
```

trace 事件名固定 9 个，按需过滤即可：

```
run_started → prompt_built → model_requested → model_parsed
            → tool_executed → checkpoint_created → runtime_identity_mismatch
            → model_failed → run_finished
```

典型症状 → 对应事件：

| 症状 | 该看的事件 |
|---|---|
| prompt 太长/裁剪过头 | `prompt_built`（含 `duration_ms`、各段 `rendered_chars`、`budget_reductions`） |
| 工具被拒 | `tool_executed`（含 `tool_status`、`security_event_type`） |
| 恢复没生效 | `checkpoint_created`（看 `trigger`）、`runtime_identity_mismatch` |
| 模型输出解析失败 | `model_failed` |

### Step 4 —— 判"是不是环境漂移"

对比当前环境与 artifact 里的 `reproducibility` 段：

```bash
git rev-parse HEAD              # vs runtime.commit_sha
```

- `commit_sha` 变了 → 代码变了，结果不可比；
- `fixture_snapshot_id` 变了 → **测试资产被改过**，旧数字全部作废（这是最容易被忽略的一种"数据造假"）；
- `model_name` / `decoding` 变了 → 换了解码参数，pass 率不可直接对比。

### Step 5 —— 恢复类问题单独走一条路

| 要看什么 | 字段 | 判读规则 |
|---|---|---|
| 恢复状态判定对不对 | `rows[].resume_status` | 与任务构造时预置的场景对比 |
| 有没有"假接受" | `false_accept` | 本该拒（partial-stale / workspace-mismatch / schema-mismatch）却报了 `full-valid` |
| stale 有没有真的重锚 | `stale_reanchored` | trace 里有 `checkpoint_created(trigger=freshness_mismatch)` |
| 漂移有没有被发现 | `workspace_drift_detected` | trace 里有 `runtime_identity_mismatch` |

### Step 6 —— 判"指标是不是虚高"

如果 12/12 全过但心里没底，检查两个**构造性恒真**的口子（见坑点 1、2）：

- `within_budget` 恒为 True；
- `expected_artifact_exists` 只查存在性，空文件也算存在。

真正的把关只有 `verifier_exit_code` 一项，所以**必须去读 verifier 脚本本身**，而不是只看 `pass_rate: 1.0`。

---

## 5. 易踩坑点

按"会不会让你得出错误结论"排序，越靠前越致命。

### 🔴 坑点 1：`within_budget` 构造性恒真，`budget_exceeded` 不可达

```python
# evaluator.py:468  max_steps = step_budget
Pico(..., max_steps=int(task["step_budget"]), ...)
# agent_loop.py:152  while tool_steps < agent.max_steps
# evaluator.py:500
within_budget = task_state.tool_steps <= int(task["step_budget"])
```

`tool_steps` 每轮只 +1，循环条件是 `< max_steps`，所以 `tool_steps <= step_budget` **数学上必然成立**。

后果：
- `within_budget_rate` 永远是 `100%`，**不能当指标讲**；
- `failure_category` 里的 `budget_exceeded` 永远走不到；
- 如果把 `budget-1` 写成 100% 简历，会被追问穿。

要真想测预算，得把 `max_steps` 设得比 `step_budget` 大，或直接改判定口径。

### 🔴 坑点 2：`expected_artifact_exists` 只查存在性，不查内容

`TASK_FIXTURE_ARTIFACTS = {"bench_repo_readme": "README.md", "bench_repo_patch": "sample.txt"}` 里写死的路径只要 `exists()` 就为真。一个 0 字节的 README.md 也能拿到这一分。内容正确性完全押在 verifier 上 —— 所以**写新任务时 verifier 的空值/边界断言不能省**。

### 🔴 坑点 3：`scripts/*.py` 的 import 路径已失效（实测报错）

三个脚本头部都是：

```python
from pico.metrics import collect_resume_metrics, render_resume_metrics_markdown
```

但实际模块是 `pico/evaluation/metrics.py`，仓库里**没有 `pico/metrics.py`**。实测：

```
FAIL pico.metrics -> ModuleNotFoundError No module named 'pico.metrics'
OK   pico.evaluation.metrics
```

即当前状态下 `python scripts/run_large_scale_experiments.py ...` 会直接 ImportError。要么补一个 shim，要么把三处 import 改成 `pico.evaluation.metrics`。

> 现状说明：仓库 `benchmarks/results/main-resume-repro-2026-06-07/` 下的四份 artifact 是完整有效的，说明当初是用别的方式（或在重构前）产出的；重构把 `metrics.py` 挪进 `evaluation/` 后漏改了这三处 import。

### 🟠 坑点 4：`scripts` 之外的第二个出口 —— `pico/metrics.py` 是否存在必须显式确认

打包/安装后（`pico.egg-info` 存在）`import pico.metrics` 同样失败，所以这不是"路径没加进 sys.path"的问题，是真的缺模块。CI 里如果有跑脚本的步骤，一定是红的。

### 🟠 坑点 5：`_apply_task_setup` 对未知 `kind` 静默忽略

```python
kind = str(setup.get("kind", "")).strip()
if kind == "context_reduction": ... return
if kind == "freshness_mismatch": ... return
if kind == "workspace_mismatch": ... return
# 没有 else → 未知 kind 直接掉出函数
```

`setup` 写错（比如写成 `context-reduction`）不会报错，任务会**在没有预置上下文的情况下"正常跑过"**，得到一个毫无意义的 pass。这类静默降级比直接报错危险得多。

### 🟠 坑点 6：`_failure_category` 的 `return "unknown"` 是不可达分支

`passed = A and B and C and D`。若 `passed` 为 False，则 A/B/C/D 至少一个为 False，函数内四个 `if not X` 必然命中一个。所以 `"unknown"` 永远不会被返回。它是防御性冗余，不是活代码 —— 看到 `failure_category_counts` 里出现 `unknown` 才应该警觉（说明 row 来自别的来源）。

### 🟠 坑点 7：`verifier` 子进程无超时，能把整轮实验卡死

```python
subprocess.run(task["verifier"], cwd=fixture_copy_root, shell=True, capture_output=True, text=True)
```

`_git_value` 很谨慎地加了 `timeout=5`，但**决定整轮成败的 verifier 反而没有超时**。一个写错的 verifier（例如等待 stdin）会让 12 任务串行跑到天亮。

### 🟡 坑点 8：`BenchmarkEvaluator` 默认 `mkdtemp` 不会自动清理

```python
self.workspace_root = Path(workspace_root) if workspace_root is not None else Path(tempfile.mkdtemp(prefix="pico-benchmark-"))
```

`tempfile.mkdtemp` 只在系统临时目录留一堆 `pico-benchmark-*`，不像 `TemporaryDirectory` 会自动删。对比 `metrics.py` 里到处用 `with tempfile.TemporaryDirectory(...)`，风格不统一 → 长期跑 CI 会堆临时目录。

### 🟡 坑点 9：新增 fixture 必须同步改 `TASK_FIXTURE_ARTIFACTS`

```python
def _artifact_path_for_task(task):
    fixture_repo_name = Path(str(task["fixture_repo"])).name
    if fixture_repo_name not in TASK_FIXTURE_ARTIFACTS:
        raise ValueError(f"unsupported fixture repo for artifact lookup: {fixture_repo_name}")
```

这个 dict 只有 2 个 key。加第 3 个 fixture 时忘改 → 12 个任务全挂 `missing_artifact`。

### 🟡 坑点 10：`load_benchmark` 的 `repo_root` 是"文件位置推测"，不是显式配置

```python
if repo_root is None:
    repo_root = path.resolve().parent.parent   # benchmarks/coding_tasks.json → 仓库根
```

只有 `benchmarks/xxx.json` 这个深度才对。把 benchmark 文件挪个位置，`repo_root` 静默指错，随后 `fixture_repo` 找不到 → 报的是"fixture 不存在"，而真实原因是"你挪了文件"。

### 🟡 坑点 11：孤儿数据与孤儿函数

| 类型 | 内容 | 说明 |
|---|---|---|
| 孤儿脚本化输出 | `SCRIPTED_MODEL_OUTPUTS` 里的 `readme_ordering_note`、`sample_placeholder_delta` | `coding_tasks.json` 12 条任务里没有这两个 id，属于历史遗留 |
| 孤儿场景函数 | `_scenario_empty_command`（metrics.py:554） | 定义了但没登记进 `SECURITY_SCENARIOS`，永远不会执行 |

只提不删：可能是故意留的扩展位，删之前需确认。

### 🟡 坑点 12：synthetic 与 real 的安全场景集**不完全对称**

| | synthetic（`SECURITY_SCENARIOS`） | real（`REAL_SECURITY_SCENARIOS`） |
|---|---|---|
| `read_only_write` | ✅ | ✅ |
| `read_only_patch` | ❌ **缺** | ✅ |
| `empty_command` | ❌（只定义了孤儿函数） | ❌ |
| 场景数 | 10 | 10 + `repeated_identical_call` = 11 |

拿两边的 `security_event_counts` 直接对齐做对比会错位，必须先按 `scenario_id` join。

### 🟡 坑点 13：`schema_mismatch_missing` 名实不符，是唯一失败源

```python
{"id": "schema_mismatch_missing", "category": "schema_mismatch", "setup": "no_checkpoint",
 "required_fragments": ["resume status: no-checkpoint"]}
```

category 声称测 schema 不匹配，`setup` 却用的是 `no_checkpoint`（根本没有 checkpoint）。

实测结果（`recovery-ablation-v2.json`）：`resume_enabled` 组 30 行里**只有这 3 行失败**，把 `resume_success_rate` 从 1.00 拉到 **0.90**。原因是 prompt 里没有出现 `resume status: no-checkpoint` 这个片段，探针返回 `"missing recovery state."`，而 `resume_status` 本身确实报了 `no-checkpoint` —— 说明 **no-checkpoint 时 prompt 的恢复段根本没有渲染 "resume status:" 这一行**，这个任务测的是"渲染分支缺失"，不是"schema 迁移"。


### 🟡 坑点 14：`false_accept` 的检测覆盖面比看上去窄

```python
invalid_resume = task["category"] in {"partial_stale", "workspace_mismatch", "schema_mismatch"}
"false_accept": invalid_resume and resume_status == "full-valid"
```

只有在 `resume_status` 恰好等于 `full-valid` 时才算"假接受"。而实测里 `checkpoint_resume_*` 这类本该 `full-valid` 的任务回落成了 `partial-stale`（见下表），于是分母里的 `partial_stale` 任务即使**误报成 partial-stale 也测不出来**。

实测 `resume_enabled` 的 status 分布：

| 任务 | 期望 | 实测 status |
|---|---|---|
| `checkpoint_resume_goal` | full-valid | **partial-stale** |
| `checkpoint_resume_files` | full-valid | **partial-stale** |
| `partial_success_shell/tool` | — | partial-stale |

结论：`resume_false_accept_rate = 0.00` 是"没有被检出"而不是"证明没有"。引用这个数时要说清口径。

### 🟡 坑点 15：`_provider_summary_from_artifact` 依赖手工注入的私有字段

```python
"artifact_path": payload.get("_artifact_path", ""),   # metrics.py:676
```

`_artifact_path` 是 `run_provider_experiments` 在 evaluator 返回后**手写补进 payload** 的（metrics.py:792），evaluator 本身从不产出这个字段。任何绕过 `run_provider_experiments` 直接调 `run_fixed_benchmark` 再喂给这个函数的路径都会得到空路径。

### 🟢 坑点 16：`datetime.utcnow()` 在 Python 3.12+ 已 deprecated

出现在 `run_context_ablation_v2` / `run_memory_ablation_v2` / `run_recovery_ablation_v2`（metrics.py:1572 / 1585 / 1605）。产出格式与其他地方的 `_now_in_timezone`（带时区偏移）也不一致，跨 artifact 对齐时间戳时要留意。

### 🟢 坑点 17：`_artifact_path_for_task` 只认 fixture 名，不认任务 id

`fixture_repo` 是同名 fixture 的不同任务会共用一条 artifact 路径。当前任务集里 `bench_repo_readme` 被 6 个任务共用、`bench_repo_patch` 被 6 个任务共用，因为每个任务跑在独立副本里所以没问题 —— 但**一旦改成共享工作区就会互相覆盖**。

---

## 6. 为什么这么设计

### 问题一：Agent 的输出天然不确定，怎么测？

**做法**：把模型换成 `FakeModelClient` + 脚本化输出，`SCRIPTED_MODEL_OUTPUTS` 里逐条写死工具调用序列。

**为什么**：回归测试要回答的问题是"**我的代码改坏了没有**"。如果模型也参与随机，一次失败你无法区分是代码 bug 还是模型抖动 —— 排障成本翻十倍。把模型钉死之后，**失败 100% 是 harness 的问题**，这是"可回归"的前提。

**代价与对冲**：脚本化会掩盖真实模型下的行为差异。所以另开一条 `experiment_mode="real"` 轨道，用真模型跑同样的任务集做背书 —— 两条轨道回答两个不同问题，谁也不替代谁。

### 问题二：评测会写坏文件，怎么保证不污染仓库？

**做法**：一任务一 `copytree` 副本 + `WorkspaceContext.build(repo_root_override=copy_root)`。

**为什么**：Agent 的 `write_file` / `patch_file` 是真的会落盘的。评测跑在源 fixture 上，跑几次之后 fixture 就被改得面目全非，之后的 pass 全是假的。副本 + 边界重锚定让**测试资产永远是只读源**。

**代价与对冲**：磁盘占用上升；用 `rmtree` 前置保证幂等，让重跑开销可接受。

### 问题三：只看"verifier 过了"够不够？

**做法**：四条件 AND，verifier 只是其中一项。

**为什么**：verifier 只看**终态文件**，看不到过程。以下三种情况 verifier 都会返回 0，但都不是"任务完成"：

1. **超步数硬凑**：模型试了 20 次终于蒙对（`within_budget` 抓）；
2. **异常停止**：中途 `model_error` / `tool_timeout`，只是恰好文件已经写好了（`non_failure_stop_reason` 抓）；
3. **产物没落**：verifier 断言的是别的东西，真正该产出的文件压根没写（`expected_artifact_exists` 抓）。

只信 verifier 就是把"撞出来的正确"记成"能力"。

### 问题四：`failure_category` 为什么要定优先级，而不是返回 4 个布尔？

**做法**：`missing_artifact > budget_exceeded > verifier_failed > failure_stop_reason`。

**为什么**：4 个布尔里可能有 3 个同时为假，人看到之后还要自己判断"先修哪个"。给一个**按可行动性排序的单一标签**，`failure_category_counts` 就能直接告诉你"本轮 12 个失败里 9 个是 verifier 挂了"，排障从这个桶开始就行。

**优先级依据**：越靠"因果上游"越优先。文件没产出是上游，断言没过是下游 —— 修上游可能自动解决下游。

### 问题五：为什么要在产物里塞 `fixture_snapshot_id` 和 `commit_sha`？

**做法**：`_fixture_snapshot_id` 遍历 fixture 树，把相对路径 + 文件内容一起喂进 sha256。

**为什么**：这是防"数字脱锚"。真实事故形态是：改了 fixture 里的 README 文案 → 旧 verifier 断言失效 → 有人顺手调了 verifier → 旧报告里的 `pass_rate: 100%` 还挂在简历/文档上。`fixture_snapshot_id` 一变，任何引用旧数字的行为立刻可被质疑。

`commit_sha` 同理，防的是"代码改了但结论没重跑"。

### 问题六：为什么要做"对照组"而不是只看"打开功能时的表现"？

**做法**：`memory_on/memory_off`、`resume_enabled/resume_disabled`、`full/no_context_reduction`。

**为什么**：只有实验组数据，"记忆有用"就是一句无法证伪的话。有了对照组：

- `resume_disabled` 组 `resume_success_rate = 0.0` → 证明**成功 100% 来自 checkpoint 机制本身**，而不是任务太简单；
- `memory_off` 组重复读次数上升 → 证明收益来自笔记召回，不是"文件本来就读得快"。

**这是整套评测设计里最关键的一条**：它把"我写了 X 功能"变成"X 功能带来了可测量的差值"。

### 问题七：为什么一份报告要同时给"能上简历的指标"和"不能上简历的指标"？

**做法**：`write_benchmark_core_report` 显式分两张清单 + 一段「口径边界」。

**为什么**：指标的说服力取决于**它是否会被追问穿**。

- `avg_prompt_compression_ratio` 可复算、口径固定 → 能上简历；
- `within_budget_rate` 构造性恒真 → 一追问就穿 → 只配放文档；
- `memory_hit_rate` 强依赖任务集构造 → 换套任务就变 → 面试展开说过程，不要说成能力上限。

同时那段「口径边界」把 `harness regression`（证明 runtime 合同稳定）和 `provider benchmark`（讨论模型上限）划开 —— **防止把"我的框架没 bug"说成"我的模型很强"**。

---

## 7. 解决了什么

| 之前的状态 | 现在 | 靠什么机制 |
|---|---|---|
| "跑一遍看着对" | 12 个固定任务的确定 pass/fail | `validate_benchmark` + 四条件 + `FakeModelClient` |
| 失败只知道"没过" | 单一可行动的 failure_category + verifier 原始 stderr | `_failure_category` + `verifier_stdout/stderr` 入产物 |
| 数字无法追溯来源 | 结果绑定 `commit_sha` + `fixture_snapshot_id` + 解码参数 | `run()` 的 `reproducibility` 段 |
| 评测污染测试资产 | 一任务一副本，源 fixture 只读 | `copytree` + `repo_root_override` + `rmtree` 幂等 |
| "我加了记忆/裁剪/恢复" | 带对照组的量化差值 | `_temporary_feature_flags` + on/off 变体 |
| 长短上下文靠感觉 | 12 配置矩阵的压缩率 + 最新请求保留率 | `run_context_stress_matrix` |
| 安全边界靠口述 | 10 场景 × `security_event_type` 计数 | `SECURITY_SCENARIOS` + `run_security_experiment_suite` |
| "恢复能用" | 4 类 resume 状态 + `false_accept` 检测 | `RECOVERY_ABLATION_TASKS` + `_RecoveryScenarioModelClient` |
| 真实模型没验证过 | provider 轨道（gpt/claude/deepseek）独立隔离跑同一任务集 | `run_provider_experiments` + per-provider try/except |
| 指标混着说 | 四层 artifact + 一份带「口径边界」的核心报告 | `write_benchmark_core_report` |
| 一次 provider 挂掉拖死全程 | 标 `status=blocked/error` + `reason`，其余照跑 | `_provider_profile` 返回态 + 单 provider try/except |

**一句话总结**：这套代码把"Agent 到底行不行"从一个主观判断，变成了一份**可复算、可追溯、带对照组、并且明确标注了哪些数字不能引用**的证据链。

---

## 8. 速查表

### 8.1 关键常量

| 常量 | 值 | 位置 |
|---|---|---|
| `BENCHMARK_SCHEMA_VERSION` | `1` | evaluator.py:19 |
| `METRICS_SCHEMA_VERSION` | `2` | metrics.py:13 |
| `TASK_FIXTURE_ARTIFACTS` | `{bench_repo_readme: README.md, bench_repo_patch: sample.txt}` | evaluator.py:42 |
| `SCRIPTED_MODEL_OUTPUTS` | 14 条（12 有任务 + 2 孤儿） | evaluator.py:47 |
| `MEMORY_EXPERIMENT_TASKS` | 12 条 / 3 类 | metrics.py:330 |
| `SECURITY_SCENARIOS` | 10 条（synthetic） | metrics.py:612 |
| `REAL_SECURITY_SCENARIOS` | 10 条（real，另加 repeated call） | metrics.py:993 |
| `RECOVERY_ABLATION_TASKS` | 10 条 / 5 类 | metrics.py:1290 |
| 默认解码 | temperature=0.0 / top_p=1.0 / max_new_tokens=64 | evaluator.py:25-27 |

### 8.2 12 个固定任务分布

| category | 数量 | 任务 id | artifact | 测什么 |
|---|---|---|---|---|
| documentation | 2 | `readme_intro_locked`、`readme_schema_note` | README.md | 精确文本替换 |
| text-edit | 2 | `sample_beta_locked`、`sample_gamma_locked` | sample.txt | 单 token 替换 |
| tool-boundary | 3 | `invalid_patch_recovery`、`path_escape_recovery`、`repeated_read_recovery` | README/sample | 被拒后能否自我修正 |
| recovery | 3 | `context_reduction_checkpoint`、`freshness_reanchor_resume`、`workspace_mismatch_resume` | README.md（**verifier 读 report.json**） | checkpoint 触发与恢复 |
| durable-contract | 2 | `durable_promotion_accept`、`durable_promotion_reject` | README.md（**verifier 读 report.json**） | 长期记忆该收的收、该拒的拒 |

注意最后 5 条：`allowed_tools` 只有 `read_file`，实测 `tool_steps = 0` —— **它们根本不改代码，验的是 harness 内部状态机**。

### 8.3 实验规模

| 实验 | 计算式 | 次数 |
|---|---|---|
| harness regression | 12 | 12 |
| context matrix | 12 配置 × 5 轮 | 60 |
| memory large | 12 任务 × 5 轮 × 3 变体 | 180 |
| recovery | 10 任务 × 3 轮 × 2 变体 | 60 |
| security（synthetic） | 10 场景 × 3 轮 | 30 |

### 8.4 实测基线（`benchmarks/results/main-resume-repro-2026-06-07/`）

| 指标 | 值 |
|---|---|
| harness `pass_rate` | `1.0`（12/12） |
| harness `within_budget_rate` / `verifier_pass_rate` | `1.0` / `1.0`（注意恒真，见坑点 1） |
| `failure_category_counts` | `{}` |
| `fixture_snapshot_id` | `sha256:61745154689175001a71064515a95f50abb7569cf344b436057ac61b61bbaf99` |
| recovery `resume_success_rate`（enabled） | `0.90`（27/30，唯一失败 = `schema_mismatch_missing`） |
| recovery `stale_reanchor_rate` | `1.00` |
| recovery `workspace_drift_detection_rate` | `1.00` |
| recovery `resume_false_accept_rate` | `0.00`（口径见坑点 14） |
| recovery（disabled 对照组） | 三个率全 `0.00`，证明机制本身有效 |

---

## 9. 待确认 / 建议动作

| 优先级 | 事项 | 建议 |
|---|---|---|
| P0 | `scripts/*.py` import 失效（`pico.metrics` 不存在） | 三处改为 `pico.evaluation.metrics`，或在 `pico/` 下补 shim；顺手在 CI 里加一条 import 冒烟 |
| P1 | verifier 无超时 | `subprocess.run(..., timeout=120)` + `try/except TimeoutExpired` 计入 `failure_stop_reason` |
| P1 | `within_budget` 恒真 | 要么让 `max_steps > step_budget`，要么在报告里注明该指标不具区分度 |
| P1 | `_apply_task_setup` 未知 kind 静默忽略 | 加 `else: raise ValueError(f"unknown setup kind: {kind}")` |
| P2 | `schema_mismatch_missing` 名实不符 | 改 id/category，或补一个真正的 schema 迁移场景；否则 `resume_success_rate = 0.90` 的解释成本很高 |
| P2 | `false_accept` 覆盖窄 | 把"期望 full-valid 却拿到 partial-stale"也计入异常检出 |
| P2 | `mkdtemp` 泄漏 | 统一走 `TemporaryDirectory`，或在 `run()` 结束时清理 |
| P3 | 孤儿数据/函数、`datetime.utcnow()`、`_artifact_path` 私有字段 | 清理 + 换成 `datetime.now(timezone.utc)` + 显式传参 |
