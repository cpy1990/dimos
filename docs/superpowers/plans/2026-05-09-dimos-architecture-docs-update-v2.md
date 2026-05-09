# DimOS 架构文档对齐 main v2 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 `docs/architecture/` 5 份文档对齐到 `origin/main` 当前状态（~60 commits drift），零残留旧代码、零漏主干子系统、每条主张有 git evidence。

**Architecture:** 7 批独立 commit，每批改 1–3 份文档；每条 edit 前跑 `git ls-tree` / `git grep` 核实新值存在、旧值位置，edit 后跑反向 grep 确认清除；7 批全部本地完成后**一次 push**（一次 CI ≈ 1h）。不改代码，只改 `docs/architecture/*.md`。

**Tech Stack:** Markdown + Mermaid；git 作 evidence 源；pre-commit hooks（trailing-whitespace / markdownlint 等，需 `source .venv/bin/activate`）。

**Spec:** `docs/superpowers/specs/2026-05-09-dimos-architecture-docs-update-v2-design.md`（权威来源；本 plan 不重复 inventory，遇 "见 spec §X" 请跳转）。

---

## 文件结构（改动范围）

改动仅限 5 份文档，**不**新增文件。每份文件的职责（改后应如此）：

- `docs/architecture/README.md` — 全景 / 13→14 子系统目录 / §3 进程模型散文 / §4 Agent 类表 / §4.5 Porcelain 新增 / §6 mermaid + 计数 / §1280 锚点表。批 A/B/C/D/E/G 都有动。
- `docs/architecture/runtime-model.md` — §1 全重写（9 小节）/ §6.2 `RunEntry` 表。批 A/B 动。
- `docs/architecture/agent-stack.md` — §1 / §3 skills / §5 MCP / §6.1 / :568 指引。批 A/C/G 动。
- `docs/architecture/robot-platforms.md` — §0 统一配置层新增 / §1.1 Go2 扩 / §1.2 G1 扩 / §3 drone 扩 / §4 机械臂扩 / §5 hardware 扩（§5.5 whole_body 新增）/ §6 simulation 扩（§6.1 五后端表、§6.6 Unity 新增）。批 E/F 动。
- `docs/architecture/subsystems.md` — §1 control 代码块结构改 / §3 navigation 加 patrolling + visual/query / §4 manipulation mermaid + servo_control / §6 memory 全重写 / §7 models 补 / §11 msgs 补 / §12 types 补 / §14 stream/ 新增 / 附录 experimental/。批 A/D/E/G 动。

每条 edit 的精确位置与 before/after 在各 Task 内列出。

---

## 前置约定

- 工作目录：`/Users/perry/workspace/dimos`，分支 `docs/architecture`
- Python venv 已激活（`source .venv/bin/activate`）——pre-commit 需要
- 所有 `git ls-tree` / `git grep` 的目标 ref 一律 `origin/main`（不是 HEAD；我们要对齐 main）
- 每批一个 commit；commit message 用 `docs(architecture): <batch-letter> <short topic>` 格式（与近期 commits 风格一致）
- **不 amend**，pre-commit hook 失败 → 修复 → 新 commit
- **不 push** 直到批 A–G 全绿 + Task H 验收通过

---

## Task A: 散点字串修正（5 份文档字串级）

覆盖 spec v2 §3 批 A 全表 + §0.6.B.1（README:172 dead import）。纯字串替换，不改代码块结构。

**Files:**
- Modify: `docs/architecture/README.md`
- Modify: `docs/architecture/runtime-model.md`
- Modify: `docs/architecture/agent-stack.md`
- Modify: `docs/architecture/subsystems.md`
- Modify: `docs/architecture/data-flow.md`（只动 `agents.agent` 锚点）

### Step A.0: 预检 — 确认新值全部存在于 main

- [ ] **Run 并确认所有命令退出码 0 且输出非空：**

```bash
git ls-tree origin/main -- dimos/core/coordination/module_coordinator.py
git ls-tree origin/main -- dimos/core/coordination/worker_manager.py
git ls-tree origin/main -- dimos/core/coordination/python_worker.py
git ls-tree origin/main -- dimos/core/coordination/blueprints.py
git ls-tree origin/main -- dimos/models/qwen/video_query.py
git ls-tree origin/main -- dimos/mapping/models.py
git ls-tree origin/main -- dimos/mapping/google_maps/models.py
git grep -n "class SpatialMemory" origin/main -- dimos/perception/spatial_perception.py
git grep -n "class RerunBridgeModule" origin/main -- dimos/visualization/
git grep -n "class RerunConvertible" origin/main -- dimos/visualization/
```

任意一条空返回 → 停工汇报（spec inventory 与 main 脱节，不继续批 A）。

### Step A.1: README.md — core 包重组 + 死 import

- [ ] **定位目标：**

```bash
grep -n "dimos/core/module_coordinator\.py\|dimos/core/worker_manager\.py\|dimos/core/worker\.py\|dimos/core/blueprints\.py\|from dimos.core.blueprints import autoconnect" docs/architecture/README.md
```

记录每行行号。

- [ ] **Edits（每处 Edit 工具调用）：**

| 旧 | 新 |
|---|---|
| `dimos/core/module_coordinator.py` | `dimos/core/coordination/module_coordinator.py` |
| `dimos/core/worker_manager.py` | `dimos/core/coordination/worker_manager.py` |
| `dimos/core/worker.py` | `dimos/core/coordination/python_worker.py` |
| `dimos/core/blueprints.py` | `dimos/core/coordination/blueprints.py` |
| `from dimos.core.blueprints import autoconnect` | `from dimos.core.coordination.blueprints import autoconnect` |

README:1100 / README:1105 的 `RerunBridge` → `RerunBridgeModule`（仅加 Module 后缀，不改上下文）。

- [ ] **Verify README:**

```bash
grep -nE "dimos/core/(module_coordinator|worker_manager|worker|blueprints)\.py|from dimos\.core\.blueprints import|\bRerunBridge\b(?!Module)" docs/architecture/README.md
```

Expected: 0 行输出（除非上下文明确 "已删除"）。

### Step A.2: runtime-model.md — logger 名 + 路径

- [ ] **Edits:**

| 旧 | 新 |
|---|---|
| `dimos.core.module_coordinator`（logger literal） | `dimos.core.coordination.module_coordinator` |
| `dimos/core/module_coordinator.py` | `dimos/core/coordination/module_coordinator.py` |
| `dimos/core/worker_manager.py` | `dimos/core/coordination/worker_manager.py` |
| `dimos/core/worker.py` | `dimos/core/coordination/python_worker.py` |

注：runtime-model.md §1 整段将在批 B 重写，批 A 只做字串改，不改结构。

- [ ] **Verify:**

```bash
grep -nE "dimos/core/(module_coordinator|worker_manager|worker)\.py|dimos\.core\.module_coordinator[^.]" docs/architecture/runtime-model.md
```

Expected: 0 行。

### Step A.3: agent-stack.md — coordination/blueprints.py

- [ ] **Edit:** `dimos/core/blueprints.py` → `dimos/core/coordination/blueprints.py`

- [ ] **Verify:**

```bash
grep -nE "dimos/core/blueprints\.py" docs/architecture/agent-stack.md
```

Expected: 0 行。

### Step A.4: subsystems.md — perception/mapping/visualization 类名 + 路径

- [ ] **Edits:**

| 旧 | 新 | 预计位置 |
|---|---|---|
| `SpatialPerception` | `SpatialMemory` | :89, :108（§2 perception） |
| `RerunBridge`（不带 Module） | `RerunBridgeModule` | :490, :494, :498, :514, :590, :592（§9 visualization） |
| `HasToRerun` | `RerunConvertible` | :500, :592 |
| `vl/video_query.py` | `models/qwen/video_query.py` | §7 models |
| `mapping/types.py` | `mapping/models.py` | §5 mapping |
| `google_maps/types.py` | `google_maps/models.py` | §5 mapping |

行号仅供定位；若 grep 出多于预计位置也一并改。

- [ ] **加 perception extra 注**（单处 inline 增一句）：在 subsystems.md §2 perception 首段末尾加：

> "**安装注意**：`perception` 子系统自 PR #1888 起从基础安装移出为 pip extra，按需 `uv sync --extra perception` 装入；详细硬依赖清单见批 G.10。"

- [ ] **Verify:**

```bash
grep -nE "SpatialPerception|\bRerunBridge\b(?!Module)|\bHasToRerun\b|vl/video_query\.py|mapping/types\.py|google_maps/types\.py" docs/architecture/subsystems.md
```

Expected: 0 行。

### Step A.5: data-flow.md — agents.agent 锚点

- [ ] **Edit:** 把 `agents.agent`（§1280 锚点表内字串）→ `agents.agent_spec` + `agents.demo_agent`（两行或一行 `+`）。

- [ ] **Verify:**

```bash
grep -nE "\bagents\.agent\b(?!_spec|_send)" docs/architecture/data-flow.md
```

Expected: 0 行。

### Step A.6: README — perception extra 标签

- [ ] **Edit:** README §6 perception 格子中加一行：

> "**可选 extra**（PR #1888）：安装时用 `uv sync --extra perception`；跨子系统硬依赖清单见批 G。"

### Step A.7: pre-commit + commit

- [ ] **Run:**

```bash
source .venv/bin/activate
pre-commit run --files docs/architecture/README.md docs/architecture/runtime-model.md docs/architecture/agent-stack.md docs/architecture/subsystems.md docs/architecture/data-flow.md
```

Expected: all hooks pass（或自动 fix 后重跑通过）。

- [ ] **Commit:**

```bash
git add docs/architecture/README.md docs/architecture/runtime-model.md docs/architecture/agent-stack.md docs/architecture/subsystems.md docs/architecture/data-flow.md
git commit -m "docs(architecture): batch A — path/class literal fixes for core/coordination, perception rename, rerun rename, mapping rename"
```

- [ ] **Gate:**

```bash
grep -rnE "dimos/core/(module_coordinator|worker_manager|worker|blueprints)\.py|SpatialPerception\b|\bHasToRerun\b|vl/video_query\.py|mapping/types\.py" docs/architecture/
```

Expected: 0 行。有则回到对应 Step 修。

---

## Task B: runtime-model §1 重写 + §6.2 RunEntry 表 + README §3 散文同步

覆盖 spec v2 §3 批 B 全部 + §0.6.E.2（pubsubrpc de-link 交代）。

**Files:**
- Modify: `docs/architecture/runtime-model.md`（§1 全重写 ~60→~220 行；§6.2 加 `rpyc_port` 行）
- Modify: `docs/architecture/README.md`（§3 散文同步，约 line 301-310）

### Step B.0: 预读源码（避免盲写）

- [ ] **读 rpyc 双侧实现**（spec §5 风险缓解要求）：

```bash
wc -l dimos/core/coordination/rpyc_server.py dimos/core/coordination/rpyc_services.py dimos/core/rpc_client.py
```

用 Read 工具读以上 3 文件全文（若任一 > 500 行则读前 500 行再按 offset 追读），把握：
1. 服务端 `RpycServer` 绑定端口 + `ThreadedServer` 语义
2. 客户端 `RPCClient` / `ModuleProxyProtocol` 的 session / proxy 建立方式
3. `rpyc_services.py` 暴露的服务接口

**产出物**：在本 Task 的草稿 scratch buffer 中写出 3–5 句描述，这就是 §1.4 正文素材。

### Step B.1: 核实 §1 所有 inventory 事实

- [ ] **Run（全部必须非空/命中）：**

```bash
git grep -n "deployment_identifier *= *\"python\"" origin/main -- dimos/core/coordination/worker_manager_python.py
git grep -n "deployment_identifier *= *\"docker\"" origin/main -- dimos/core/coordination/worker_manager_docker.py
git grep -n "class WorkerManager\b" origin/main -- dimos/core/coordination/worker_manager.py
git grep -n "class RpycServer\b" origin/main -- dimos/core/coordination/rpyc_server.py
git grep -n "class RPCClient\b" origin/main -- dimos/core/rpc_client.py
git grep -n "class ModuleProxyProtocol" origin/main -- dimos/core/rpc_client.py
git grep -nE "def (load_blueprint|load_module|unload_module)\b" origin/main -- dimos/core/coordination/module_coordinator.py
git grep -n "rpyc_port" origin/main -- dimos/core/run_registry.py
git grep -n "grpc_port" origin/main -- dimos/core/run_registry.py
git grep -n "def get_most_recent_rpyc_port\|get_most_recent_rpyc_port" origin/main -- dimos/core/run_registry.py
git grep -n "class NativeModule\b" origin/main -- dimos/core/native_module.py
git grep -n "class DockerModuleProxy" origin/main -- dimos/core/docker_module.py
git grep -nE "WorkerRequest|WorkerResponse" origin/main -- dimos/core/coordination/worker_messages.py
git grep -n "def spawn_watchdog\|spawn_watchdog" origin/main -- dimos/core/coordination/process_lifecycle.py
git grep -n "cv2\.setNumThreads" origin/main -- dimos/core/library_config.py
git ls-tree origin/main -- dimos/core/daemon.py dimos/core/log_viewer.py dimos/core/o3dpickle.py dimos/core/introspection/ dimos/core/resource_monitor/ dimos/core/resource.py
```

记录每条输出（行号/路径）到 scratch buffer——正文要写正确的行号。

### Step B.2: runtime-model §1 重写 — 写 9 个小节

- [ ] **先 Read:**

```bash
grep -n "^##\|^###" docs/architecture/runtime-model.md
```

记录现有 §1 的起止行号。

- [ ] **Edit（删现有 §1 整段）：** 用 Read 把 §1 整块读出后，Edit 替换为下面的新 §1（注意单次 Edit ≤50 行，超出分多次；实操中可先写 `§1.1–§1.4` 一次、`§1.5–§1.9` 一次、`§1 标题 + 概述` 一次）。

**§1 结构与每节内容要点**（spec §3 批 B 全文，此处凝练成每节的 skeleton，engineer 按此填文案）：

1. **§1 总述**（6–8 行）：DimOS 运行时 = 协调层（coordination/）+ 双 worker path（Python / Docker）+ rpyc IPC + watchdog + 动态 blueprint 装卸。一张 mermaid flowchart：ModuleCoordinator → {WorkerManagerPython, WorkerManagerDocker} → {PythonWorker forkserver, DockerModuleProxy} ← rpyc（RpycServer ↔ RPCClient）。
2. **§1.1 协调层双轨**（15–25 行）：`ModuleCoordinator`（文件路径 `dimos/core/coordination/module_coordinator.py`，持 `dict[str, WorkerManager]`，key = `deployment_identifier` 字面 `"python"` / `"docker"`，对应 `worker_manager_python.py:<L>` / `worker_manager_docker.py:<L>`）；`WorkerManager` Protocol（`worker_manager.py:<L>`，字段 `deployment_identifier: str`）；双实现职责分工。<L> 用 Step B.1 grep 到的行号填。
3. **§1.2 Python 路径**（10–15 行）：`PythonWorker`（`python_worker.py`，forkserver context 惰性初始化）+ `worker_messages.py` 的 `WorkerRequest`/`WorkerResponse`（PR #1767，进程间消息协议）。
4. **§1.3 Docker 路径**（6–10 行）：`WorkerManagerDocker` + `core/docker_module.py` 的 `DockerModuleProxy`（住 `core/` 顶层**非** `coordination/`）。
5. **§1.4 rpyc IPC 双侧**（25–40 行，Step B.0 读出的内容灌这里）：
   - 服务端 `coordination/rpyc_server.py` 的 `RpycServer`（持 `ThreadedServer`）+ `coordination/rpyc_services.py`
   - 客户端 `core/rpc_client.py` 的 `RPCClient:<L>` + `ModuleProxyProtocol:<L>`
   - 取代旧 `Actor` + `Pipe`（两词须不复出现；见 Gate）
   - 本地/远程同管道（porcelain 也走这条；批 D 会讲 porcelain facade）
   - **末尾一句**：`protocol/rpc/pubsubrpc.py` 仍存在但已从 coordination 路径 de-link（PR #9d7806615），rpyc 为主 IPC；旧 pubsubrpc 路径只作 reference 存在，不在启动路径触发。
6. **§1.5 动态 blueprint 装卸**（10–15 行）：`ModuleCoordinator.load_blueprint` / `load_module` / `unload_module` 三 API（行号填 Step B.1 grep 值）——支持启动后热装卸（PR #1744）。
7. **§1.6 watchdog 生命周期**（8–12 行）：`watchdog_main.py` sidecar + `process_lifecycle.py` 的 `spawn_watchdog` 封装 + `core/run_registry.py` 跨 run 注册。
8. **§1.7 Native / Rust 模块**（6–10 行）：`core/native_module.py` 的 `NativeModule`（CLI 传 LCM topic 名，C/C++/Rust subprocess 委托；rust native modules PR #1794）。
9. **§1.8 Async 模块**（8–12 行）：PR #1920；patrol 示例（前向引用 §E「批 E.2 patrolling」）；6 个 `test_async_module_*.py`；点出 async dispatch / handles / process observables / RPC / sync-to-async 五条能力。
10. **§1.9 core/ 顶层其余**（8–12 行）：`introspection/` / `resource_monitor/` / `resource.py`（资源管控）；`o3dpickle.py`（Open3D 跨进程序列化）；`library_config.py`（仅 `cv2.setNumThreads(2)` 启动副作用，**不是**配置系统）；`core.py` / `daemon.py`（`daemonize()` 双 fork）/ `log_viewer.py`（CLI 日志）。

**不再出现关键词**（Step B.5 Gate 检）：`Actor\b`（除非明确"已删除"上下文）、`MethodCallProxy`、`dimos/core/worker\.py`（原单文件，批 A 应已清；此处是兜底）、旧 logger 字面 `dimos.core.module_coordinator`。

### Step B.3: runtime-model §6.2 RunEntry 表扩展

- [ ] **定位：**

```bash
grep -n "RunEntry\|grpc_port\|rpyc_port" docs/architecture/runtime-model.md
```

- [ ] **Edit RunEntry 表:** 在原有字段列表中追加两行（依现有表格式渲染）：
  - `rpyc_port: int = 0` — 新增（daemon 用此端口，当前主 IPC）
  - `grpc_port: int = 9877` — 标注为 **legacy**（`check_port_conflicts` 仍在用，不再是运行时 IPC）
  - 末尾加一句：`get_most_recent_rpyc_port()`（`run_registry.py:<L>`）按 run registry 取最新 rpyc 端口。

### Step B.4: README §3 散文同步

- [ ] **定位:**

```bash
grep -n "deploy_parallel\|ModuleProxy\|ThreadPoolExecutor\|Actor\b\|worker_manager\|module_coordinator" docs/architecture/README.md
```

记录 §3 内这些散文的现有行号。

- [ ] **Edit §3（line ≈ 301–310）:** 两件事：
  1. 把 `Actor` / `MethodCallProxy` 的概念散文替换为 "rpyc 为当前 IPC，详见 runtime-model.md §1.4"——**不展开**（README §3 是总览，展开在 runtime-model）。
  2. `deploy_parallel` / `ThreadPoolExecutor` / `ModuleProxy` 等旧词如仍在需标注 "**已被 rpyc 取代**，本节展开见 runtime-model §1"。**保留**散文骨架，不删整段。

### Step B.5: 验收 + pre-commit + commit

- [ ] **Gate（本批完成后整文档 0 命中）：**

```bash
grep -rnE "\bActor\b|MethodCallProxy|dimos/core/worker\.py|dimos\.core\.module_coordinator[^.]" docs/architecture/runtime-model.md docs/architecture/README.md
```

Expected: 0。若 `Actor` 在"已删除"或"legacy"上下文中出现可保留——逐条看上下文判断。

- [ ] **Gate（新词至少各 1 命中）：**

```bash
grep -cE "coordination/|WorkerManagerPython|WorkerManagerDocker|RpycServer|RPCClient|ModuleProxyProtocol|watchdog_main|NativeModule|load_blueprint|rpyc_port|WorkerResponse|pubsubrpc" docs/architecture/runtime-model.md
```

Expected: 每个关键词 ≥1（数字 ≥12）。

- [ ] **pre-commit + commit:**

```bash
source .venv/bin/activate
pre-commit run --files docs/architecture/runtime-model.md docs/architecture/README.md
git add docs/architecture/runtime-model.md docs/architecture/README.md
git commit -m "docs(architecture): batch B — runtime-model §1 rewrite (coordination/ + rpyc + dynamic blueprint API) + RunEntry.rpyc_port"
```

---

## Task C: agent-stack §1/§5/§6.1 + README §4 Agent 类表 + tool streams

覆盖 spec v2 §3 批 C 全部。**关键约束**：README §4 Agent 类表**只换 Agent/VLMAgent 行**，保留 WebInput + ollama_agent.py + 4 条命名重叠避坑段。

**Files:**
- Modify: `docs/architecture/agent-stack.md`（§1 / §5 / §6.1 / :568 指引 / §3 skills 新小节留到批 G）
- Modify: `docs/architecture/README.md`（§4 类表 / §4.5 MCP 短章 / §1280 锚点表）

### Step C.0: 预检 — 新 AgentSpec / demo_agent / ToolStream 确在

- [ ] **Run（全部非空）：**

```bash
git grep -n "class AgentSpec\b" origin/main -- dimos/agents/agent_spec.py
git grep -n "class VlmAgentSpec\|class VLMAgentSpec" origin/main -- dimos/agents/vlm_agent_spec.py
git grep -nE "^demo_agent\b|^demo_agent_camera\b|^def demo_agent\b" origin/main -- dimos/agents/demo_agent.py
git grep -n "class ToolStream\b\|TOOL_STREAM_TOPIC" origin/main -- dimos/agents/mcp/tool_stream.py
git grep -nE "^(class|def) " origin/main -- dimos/agents/ollama_agent.py
git ls-tree origin/main -- dimos/agents/agent.py
```

末条（agent.py）应为**空输出**（类已删）；其他必须有命中。

### Step C.1: agent-stack §1 重写（原讲已删 agent.py + LangGraph）

- [ ] **先 Read:** agent-stack.md §1 现有内容（记录行号）。

- [ ] **Edit §1 整段为新结构（4 小节）：**

1. **§1.1 AgentSpec 契约**（8–12 行）：`AgentSpec(Spec, Protocol)` @ `dimos/agents/agent_spec.py:22`，核心方法 `add_message` + `dispatch_continuation`；运行时是**任何实现者 + 任何 MCP tools 组合**，没有"唯一的 Agent 类"。
2. **§1.2 VLM 孪生**（5–8 行）：`VlmAgentSpec` @ `dimos/agents/vlm_agent_spec.py`，视觉 LLM 场景。
3. **§1.3 纯 MCP 蓝图模板**（8–12 行）：`demo_agent.py` 两个 `autoconnect(McpServer.blueprint(), McpClient.blueprint())` 蓝图 `demo_agent` / `demo_agent_camera`——**不绑硬件**，可作新蓝图模板。
4. **§1.4 实现者家族**（5–8 行）：`vlm_agent.py` / `ollama_agent.py`（**注明**：ollama_agent.py 仅暴露 `ensure_ollama_model` / `ollama_installed` 两个 `def`，**不是 class**——这是原避坑的 main 现状）。

**禁止出现**：`LangGraph` / `LangChain` / `create_agent` / `dimos/agents/agent.py` / "唯一的 Agent 类"类论断。

### Step C.2: agent-stack §5 MCP 章重写

- [ ] **定位:** `grep -n "unitree-go2-agentic-mcp\|McpServer\|McpClient\|McpAdapter" docs/architecture/agent-stack.md`

- [ ] **Edit §5：** 把"`unitree-go2-agentic-mcp` 是唯一支持 MCP 的 blueprint"论断**整段重写**为双类：

**硬件绑定 MCP 蓝图（5 个，全在 `dimos/robot/unitree/go2/blueprints/agentic/`）：**
- `unitree_go2_agentic.py`
- `unitree_go2_agentic_huggingface.py`
- `unitree_go2_agentic_ollama.py`
- `unitree_go2_security.py`
- `unitree_go2_temporal_memory.py`
（另有 `_common_agentic.py` 为共享辅助，不对用户暴露）

**纯 MCP 模板（2 个，不绑硬件，`dimos/agents/demo_agent.py`）：** `demo_agent`、`demo_agent_camera`。

- [ ] **新增 §5 子节「Tool Streams」（PR #1713）：**
  - 源：`dimos/agents/mcp/tool_stream.py` 的 `ToolStream` + `current_skill_context` 绑定 + `TOOL_STREAM_TOPIC = "/tool_streams"`
  - 事件流：`@skill` 内 → `pLCMTransport` → `McpServer` 订阅 → `notifications/message` + `notifications/progress` JSON-RPC → MCP SSE 下发客户端
  - 3–5 行 Mermaid sequenceDiagram 或 ASCII pipe 示意。

### Step C.3: agent-stack §6.1 + :568 指引

- [ ] **定位 §6.1:** `grep -n "^### \|^#### \|Agent.*dimos/agents/agent.py" docs/architecture/agent-stack.md`

- [ ] **Edit §6.1：** 删"`Agent`（`dimos/agents/agent.py`）——标准智能体"整章；替换为"**运行时组合**"小节——3–5 行，指向 §1 的 AgentSpec 讨论。

- [ ] **Edit :568 附近指引：** 原文 "**所有新代码请使用 `dimos.agents.agent.Agent`…**" → 改为：
  > "**所有新代码请实现 `dimos.agents.agent_spec.AgentSpec` Protocol，或直接以 `demo_agent` / `vlm_agent` 蓝图为模板。`dimos/agents_deprecated/` 仍保留做 legacy 兼容，严禁在新代码引用。**"

### Step C.4: README §4 Agent 类表 — **只替 Agent/VLMAgent 行，保留避坑**

- [ ] **定位:** `grep -n "^### §4\|dimos/agents/agent\.py\|WebInput\|ollama_agent\.py\|agents_deprecated\|两套 skills\|命名重叠" docs/architecture/README.md`

- [ ] **Edits in §4 类表（保守原则：只动 Agent / VLMAgent 两行 + mermaid 一行，其他全保留）：**
  - **删** `Agent` @ `dimos/agents/agent.py` 行
  - **加** `AgentSpec(Protocol)` @ `dimos/agents/agent_spec.py:22` 行
  - **加** `demo_agent` / `demo_agent_camera` @ `dimos/agents/demo_agent.py` 行
  - **改** `VLMAgent` 行 → `VlmAgent` + `VlmAgentSpec` 两行（路径更新）
  - **保留** `WebInput` 行及"类名为 WebInput"避坑
  - **保留** `ollama_agent.py` 行及"不是类，只暴露两 def"避坑
  - **保留** 命名重叠 3 整段（`agents_deprecated/` 避坑），但把其中"请使用 `dimos.agents.agent.Agent`"改为"请实现 `dimos.agents.agent_spec.AgentSpec` / 用 `demo_agent`"
  - **保留** 命名重叠 4 整段（两套 skills）

- [ ] **mermaid classDiagram 改写：** 原 `Module <|-- Agent/VLMAgent/WebInput` → 新：

```
AgentSpec <|.. demo_agent
AgentSpec <|.. VlmAgent
Module <|-- WebInput
```

### Step C.5: README §4.5 MCP 短章 + §1280 锚点表

- [ ] **§4.5 MCP 段:** 同 agent-stack §5 双类结构，简版 3–5 行 + "详见 agent-stack §5"链接。

- [ ] **§1280 锚点表（data-flow 小节交叉表）：** `agents.agent` → `agents.agent_spec` / `agents.demo_agent` 两行。

### Step C.6: 验收 + pre-commit + commit

- [ ] **0-命中 Gate:**

```bash
grep -rnE "dimos/agents/agent\.py|LangGraph|LangChain|create_agent|unitree-go2-agentic-mcp.*唯一|\bclass Agent\b" docs/architecture/agent-stack.md docs/architecture/README.md
```

Expected: 0（`class AgentSpec` 允许存在，正则用 `\bclass Agent\b` 严格 word boundary 不会误杀 `AgentSpec`）。

- [ ] **≥1-命中 Gate:**

```bash
grep -cE "AgentSpec|agent_spec\.py|demo_agent(_camera)?|ToolStream|TOOL_STREAM_TOPIC" docs/architecture/agent-stack.md docs/architecture/README.md
```

Expected: 每文件每关键词 ≥1。

- [ ] **commit:**

```bash
source .venv/bin/activate
pre-commit run --files docs/architecture/agent-stack.md docs/architecture/README.md
git add docs/architecture/agent-stack.md docs/architecture/README.md
git commit -m "docs(architecture): batch C — AgentSpec Protocol + demo_agent + dual MCP blueprint classes + tool streams"
```

- [ ] **派 Explore subagent 交叉验证**（spec §4.3 要求）：

Agent tool，subagent_type=Explore，prompt：
> "读 `docs/architecture/agent-stack.md` 与 `docs/architecture/README.md` 的 Agent 部分，对照 `origin/main` 的 `dimos/agents/` 代码，找出还有哪些过时描述（已删类、错路径、错字面值）。只做 read-only，200 字内报告。"

如发现问题：同批次修复（允许追加一个"hotfix" commit）。

---

## Task D: memory2 主体 + Porcelain + README §6 fan-out + data-flow 补丁

覆盖 spec v2 §3 批 D 全部。memory2 是 v1 欠账最重的区块之一。

**Files:**
- Modify: `docs/architecture/subsystems.md`（§6 memory/ 重写 ~70→~200 行）
- Modify: `docs/architecture/README.md`（§4.5 Porcelain 新章 ~60 行 + §6 fan-out）
- Modify: `docs/architecture/data-flow.md`（补 mapping trace + tool streams trace + 去残留行号）

### Step D.0: 预读 memory2 源码（spec §5 风险缓解要求）

- [ ] **Read 下列文件全文，并记录类 / 函数 / 行号：**

```bash
wc -l dimos/memory2/stream.py dimos/memory2/transform.py dimos/memory2/module.py dimos/memory2/embed.py dimos/memory2/backend.py dimos/memory2/registry.py
ls dimos/memory2/store/ dimos/memory2/vectorstore/ dimos/memory2/observationstore/ dimos/memory2/blobstore/ dimos/memory2/codecs/ dimos/memory2/notifier/ dimos/memory2/vis/plot/ dimos/memory2/vis/space/
```

读完后 scratch buffer 写出 10–15 行素材——§6 正文素材。

- [ ] **核实 inventory：**

```bash
git grep -n "class Stream\b" origin/main -- dimos/memory2/stream.py
git grep -nE "class (Transformer|FnTransformer|Batch|QualityWindow)\b" origin/main -- dimos/memory2/transform.py
git grep -nE "class (StreamModule|MemoryModule|SemanticSearch|Recorder)\b" origin/main -- dimos/memory2/module.py
git grep -nE "class (EmbedImages|EmbedText)\b" origin/main -- dimos/memory2/embed.py
git grep -n "USING vec0" origin/main -- dimos/memory2/vectorstore/sqlite.py
git grep -n "class VoxelGridMapper\b" origin/main -- dimos/mapping/voxels.py
git grep -n "class Dimos\b" origin/main -- dimos/porcelain/dimos.py
git grep -n "^from dimos.porcelain.dimos import Dimos\|__getattr__" origin/main -- dimos/__init__.py
```

全部必须命中。

### Step D.1: subsystems.md §6 memory/ 重写（~200 行）

- [ ] **Read & 定位现有 §6：** `grep -n "^## §6\|^### §6\|^## §7\|memory/" docs/architecture/subsystems.md`

- [ ] **Edit（整段替换，多次 Edit 调用，每次 ≤50 行）** — §6 新结构 6 小节：

1. **§6 开头一段**（5 行）：memory/ 子系统由 `memory2/`（主）+ `memory/timeseries/`（辅，通用时序 KV）构成；`memory/embedding.py` 是未完成原型（零非测试调用），**已被 memory2.SemanticSearch 替代**，不展开。

2. **§6.1 memory2 核心抽象**（~40 行）：
   - `Stream[T]` @ `memory2/stream.py` — lazy pull；方法 `.after()` / `.near()` / `.search()` / `.live()` / `.save()` / `.drain()` / `.observable()`
   - `Transformer[T,R]` @ `memory2/transform.py` — `Iterator→Iterator` 基类；实现：`FnTransformer` / `FnIterTransformer` / `Batch` / `QualityWindow` / `downsample` / `throttle` / `peaks` / `significant` / `smooth` / `normalize`
   - `StreamModule` @ `memory2/module.py:69` — `Module, Generic[TIn, TOut]`；用 `pipeline(stream)` 声明变换
   - 一张小 mermaid：`Stream → Transformer chain → StreamModule → 下游`

3. **§6.2 成品 module 三件套（同住 `memory2/module.py`）**（~15 行）：
   - `MemoryModule:170`
   - `SemanticSearch:196`
   - `Recorder:247`
   - `EmbedImages` @ `embed.py:40` / `EmbedText` @ `embed.py:61` — 均 `Transformer[Any, Any]`，作嵌入层插入 pipeline

4. **§6.3 存储层七件套**（~50 行，每件 2–3 句）：
   - `store/`：`Store` ABC + `SqliteStore`（WAL / FTS5 / sqlite-vec（vec0））+ `MemoryStore` + `NullStore`
   - `observationstore/`：`ObservationStore` Protocol + `ListObservationStore` / `SqliteObservationStore`
   - `vectorstore/`：`VectorStore` ABC + `memory.py`（暴力实现）+ `sqlite.py:70` `USING vec0(...)`（ANN）
   - `blobstore/`：大 payload 文件 / SQLite 两后端
   - `codecs/`：`JpegCodec`（turbojpeg 10–20×）/ `LcmCodec` / `LZ4Codec` / `PickleCodec`；`codec_for()` 自动选
   - `notifier/`：`Notifier` ABC + `SubjectNotifier`（RxPY）
   - `vis/`：`color.py` + `plot/`（时序图 `rerun.py` / `svg.py` / `elements.py` / `plot.py`）+ `space/`（3D `rerun.py` / `svg.py` / `space.py`）+ `utils.py`
   - `backend.py`：`Backend` 组合 `ObservationStore + BlobStore + VectorStore + Notifier`
   - `registry.py`：`RegistryStore`（跨 run 流 registry）

5. **§6.4 集成示例**（~30 行）：
   - `VoxelGridMapper` @ `dimos/mapping/voxels.py:253` — **住在 `mapping/`**（非 memory2 内），继承 `StreamModule[PointCloud2, PointCloud2]`，用 `VoxelMapTransformer` 做 pipeline；体现"memory2 作为基础设施被其他子系统使用"
   - `Recorder`（smart go2 blueprint 使用）
   - `SqliteStore`（replay / dtop / demo 大量使用）
   - `RegistryStore`（跨 run SQLite 流 registry）

6. **§6.5 memory/ 短注**（~10 行）：
   - `timeseries/`：通用键值时序（`InMemory` / `Sqlite` / `PickleDir` / `Postgres` / `LegacyPickle`）；被 `tf.py` / `Timestamped` / 测试固件使用；与 memory2 不重叠
   - `embedding.py` 的 `EmbeddingMemory`：原型，零非测试调用，`_store_spatial_entry` no-op，已被 `memory2.SemanticSearch` 替代，不展开

### Step D.2: README §4.5 Porcelain 新章（~60 行）

- [ ] **定位:** `grep -n "^## §4\|^## §5\|porcelain\|Dimos" docs/architecture/README.md`

- [ ] **Edit（插在 §4 Agent 体系 与 §5 平台层之间，新建 §4.5）**，8 个小段：

1. **定位**（4 行）：Porcelain = Blueprint / ModuleCoordinator / rpyc 之上的 facade；`Dimos` 类是 `dimos/__init__.py` 唯一 public symbol（`__getattr__` lazy import `from dimos.porcelain.dimos import Dimos`）
2. **两模式**（6 行）：
   - 本地：`Dimos(**overrides)`——进程内建 coordinator
   - 远程：`Dimos.connect(run_id, host, port)`——rpyc 接已运行 daemon；无参时通过 `run_registry` 自动发现
3. **核心 API**（8 行）：`.run(target)` / `.skills.<name>(...)` / `.peek_stream(name, timeout)` / `.restart(module, reload_source=True)` / `.stop()` / `.__getattr__(name)`（直通 rpyc 模块代理）
4. **SkillsProxy**（4 行）：复用 Agent 的 `SkillInfo`（同一 `get_skills()` 协议）；`dimos/porcelain/skills_proxy.py` 实现 lazy cache
5. **与 Agent 的区别**（3 行）：porcelain 面向**外部脚本驱动**；Agent 面向**进程内 LLM dispatch**
6. **为何本地也走 rpyc**（3 行）：local/remote 路径统一，避免特化
7. **扩展阅读指针**（2 行）：指向 `docs/usage/python-api.md`（用户教程），不重复
8. **与 §2 通信栈关系**（2 行）：不重构三层栈；porcelain 位于"之上"作 facade

### Step D.3: README §6 fan-out

- [ ] **定位:** `grep -n "13 个一级子系统\|MEM\[memory/\]\|^### §6\|stream/" docs/architecture/README.md`

- [ ] **Edits（3 处）：**
  1. "DimOS 现有 13 个一级子系统" → "**14 个一级子系统**"（stream/ 进入 §14，批 G 补格子）
  2. 流程图节点 `MEM[memory/]` → `MEM2[memory2/（主）<br/>+ memory/timeseries（辅）]`
  3. §6 子系统目录表 memory 那一格：改写（memory2 为主 + timeseries 短注 + embedding 废弃 + VoxelGridMapper 被 memory2 承接的说明）
  4. §6 子系统目录表新增 stream/ 一格（点名与 `core/stream.py` 不同，批 G 正文展开；此处占位）

### Step D.4: data-flow.md 补丁

- [ ] **定位:** `grep -n "agents\.agent\b\|SpatialPerception\|SpatialMemory\|VoxelGridMapper\|ToolStream\|:[0-9][0-9][0-9]-[0-9]" docs/architecture/data-flow.md | head -50`

- [ ] **Edits:**
  1. §1280 锚点表：`agents.agent` → `agents.agent_spec` + `agents.demo_agent`（若批 A 已做此处确认即可）
  2. perception trace：类名已是 `SpatialMemory`——确认不动（Step A.5 已验证）
  3. **补 mapping trace**（一段 + 小 sequenceDiagram）：`PointCloud2` stream → `VoxelGridMapper.pipeline()` → `VoxelMapTransformer` → `Stream[PointCloud2]` 出口——定位到 manipulation/navigation trace 后一段新增小节
  4. **补 tool streams 事件流**（一段）：`@skill` → `ToolStream.send()` → `pLCMTransport` → `McpServer` → MCP SSE `notifications/{message,progress}`
  5. **去残留行号**：全文扫 `xxx\.py:[0-9]+(-[0-9]+)?` 字面，除非能被 `git grep origin/main` 复核——都删行号，保留类/函数名

### Step D.5: 验收 + pre-commit + commit

- [ ] **≥1-命中 Gate:**

```bash
grep -cE "memory2|StreamModule|Transformer\[|VoxelMapTransformer|porcelain|\bDimos\b|SkillsProxy|Recorder|SemanticSearch|vec0|sqlite-vec" docs/architecture/subsystems.md docs/architecture/README.md docs/architecture/data-flow.md
```

Expected: 每关键词 ≥1。

- [ ] **0-命中 Gate:**

```bash
grep -rnE "EmbeddingMemory[^/]|^\| memory/ \||13 个一级子系统\|MEM\[memory/\]" docs/architecture/README.md docs/architecture/subsystems.md
```

Expected: 0（`EmbeddingMemory` 在 §6.5 "已废弃" 上下文中允许出现一次，手动排除）。

- [ ] **commit:**

```bash
source .venv/bin/activate
pre-commit run --files docs/architecture/subsystems.md docs/architecture/README.md docs/architecture/data-flow.md
git add docs/architecture/subsystems.md docs/architecture/README.md docs/architecture/data-flow.md
git commit -m "docs(architecture): batch D — memory2 subsystem rewrite + Porcelain §4.5 + 14 subsystems fan-out + data-flow mapping/tool-stream traces"
```

---

## Task E: 4 个新主干子系统（v1 零覆盖）

覆盖 spec v2 §3 批 E 全部：`hardware/whole_body/`、`navigation/patrolling/`、`simulation/unity/`、`robot/catalog/ + config.py + model_parser.py`。

**Files:**
- Modify: `docs/architecture/robot-platforms.md`（§0 RobotConfig 新子节 + §5.5 whole_body 新增 + §6.6 Unity 新增 + §6.1 四→五后端对比表）
- Modify: `docs/architecture/subsystems.md`（§3 navigation 加 patrolling 子节）
- Modify: `docs/architecture/README.md`（§6 子系统目录表加 patrolling / unity / robot-config 三条）

**位置锁定（避免双写，spec §0.5 已核实）：**
- `whole_body/` 只在 robot-platforms.md §5.5，subsystems.md 不写
- `unity/` 只在 robot-platforms.md §6.6，subsystems.md 不写
- `patrolling/` 只在 subsystems.md §3（它是 navigation 子包，位置对）
- `robot-catalog + config + model_parser` 只在 robot-platforms.md §0

### Step E.0: 预检 4 个新子系统确在

- [ ] **Run（全部非空）：**

```bash
git grep -n "class WholeBodyAdapter\b\|class MotorCommand\b\|class MotorState\b\|class IMUState\b" origin/main -- dimos/hardware/whole_body/spec.py
git ls-tree origin/main -- dimos/hardware/whole_body/registry.py dimos/hardware/whole_body/transport/
git grep -n "class PatrollingModule\b" origin/main -- dimos/navigation/patrolling/module.py
git grep -n "class PatrollingModuleSpec\b" origin/main -- dimos/navigation/patrolling/patrolling_module_spec.py
git grep -nE "class (BasePatrolRouter|PatrolRouter|CoveragePatrolRouter|FrontierPatrolRouter|RandomPatrolRouter|VisitationHistory)\b" origin/main -- dimos/navigation/patrolling/routers/
git grep -n "class UnityBridgeModule\b\|class UnityBridgeConfig\b" origin/main -- dimos/simulation/unity/module.py
git grep -n "class RobotConfig\b\|class GripperConfig\b" origin/main -- dimos/robot/config.py
git grep -nE "class (JointDescription|ModelDescription)\b|def parse_model\b" origin/main -- dimos/robot/model_parser.py
git ls-tree origin/main -- dimos/robot/catalog/
```

### Step E.1: robot-platforms.md §5.5 whole_body/ 新增

- [ ] **定位 §5 hardware/:**

```bash
grep -n "^## \|^### \|§5\|drive_trains/\|end_effectors/\|manipulators/\|sensors/" docs/architecture/robot-platforms.md
```

确认现有子节 §5.1 drive_trains / §5.2 end_effectors / §5.3 manipulators / §5.4 sensors（实际 §号以文档为准；下方按"§5.X hardware/whole_body/"相对编号落笔）。

- [ ] **Edit（在 §5.4 之后插入 §5.5，~40 行）：**

```markdown
### §5.5 `hardware/whole_body/` — 全身多关节控制 HAL（新增）

`hardware/` 第五类抽象，与 drive_trains / end_effectors / manipulators / sensors 并列。HAL 层**仅定义 Protocol + 数据契约**，具体实现在 `dimos/robot/unitree/g1/wholebody_connection.py`（G1 专用，见 §1.2）。

**Protocol + 数据结构**（全部来自 `dimos/hardware/whole_body/spec.py`）：

- `WholeBodyAdapter(Protocol)` @ spec.py — 提供 `send_command(cmd: MotorCommand)` / `get_state() -> (MotorState, IMUState)` 等接口
- `MotorCommand` — dataclass（`q / dq / kp / kd / tau` per motor）
- `MotorState` — 电机状态回读
- `IMUState` — IMU 姿态/角速度回读

**其他文件：**
- `registry.py` — 适配器 registry（按 robot 名索引实现）
- `transport/adapter.py` — 传输级适配器

**与 `drive_trains/` / `manipulators/` 的区别：** 前者面向底盘/履带整体驱动，后者面向单臂/夹爪；**全身多关节控制**（双足 + 双臂 + 躯干 ≥20 DoF 同步）是独立 HAL 范畴。
```

### Step E.2: subsystems.md §3 navigation/patrolling 新增子节

- [ ] **定位:** `grep -n "^## \|^### \|§3\|navigation/\|patrolling" docs/architecture/subsystems.md`

- [ ] **Edit（在 §3 navigation 下加子节，~45 行）：**

```markdown
#### 自主巡逻 — `navigation/patrolling/`（新增）

生产级 async Module 示例（见 runtime-model §1.8），监听 `odom` / `global_costmap` / `goal_reached` 三条 stream 驱动巡逻循环。

**核心：**
- `PatrollingModule(Module)` @ `module.py`（async Module，e2e 测试 `dimos/e2e_tests/test_patrol_and_follow.py` 证生产级）
- `PatrollingModuleSpec(Spec, Protocol)` @ `patrolling_module_spec.py`
- `create_patrol_router.py` — router factory

**Router 插拔体系**（`routers/` 子目录）：
- `PatrolRouter(Protocol)` @ `routers/patrol_router.py` — 顶层协议
- `BasePatrolRouter(ABC)` @ `routers/base_patrol_router.py` — 两抽象方法基类
- 三策略实现：`CoveragePatrolRouter`（覆盖式）/ `FrontierPatrolRouter`（前沿式）/ `RandomPatrolRouter`（随机）
- `VisitationHistory` @ `routers/visitation_history.py` — 共享访问历史

**与其他 navigation 子节的关系：** patrolling 是**任务层**，复用 `replanning_a_star` / `global_planner` 生成路径；与 `visual_servoing` / `bbox_navigation` 平级。
```

### Step E.3: robot-platforms.md §6.6 Unity + §6.1 四→五后端

- [ ] **定位:** `grep -n "^## \|^### \|§6\|mujoco\|genesis\|isaac\|后端" docs/architecture/robot-platforms.md`

- [ ] **Edit §6.1 对比表:** 把"4 个仿真后端"改为"5 个仿真后端"；对比表加一列 Unity（其他列按已有格式：跨平台/GPU并行/物理真实度/依赖）。

- [ ] **Edit（在 §6.5 之后插入 §6.6，~30 行）：**

```markdown
### §6.6 Unity 后端（VLA Challenge，新增）

`dimos/simulation/unity/` 子包——为 VLA Challenge benchmark 提供基线环境。

**核心：**
- `UnityBridgeModule(Module)` @ `unity/module.py` — 走 ROS-TCP-Endpoint 二进制协议，TCP 桥接 Unity 仿真器
- `UnityBridgeConfig` @ `unity/module.py` — 配置 dataclass
- `unity/blueprint.py` — 运行入口蓝图

**关键点：** **无 ROS 依赖**——ROS-TCP-Endpoint 只是二进制协议，不需要运行 ROS master 或 rosdep 环境。

**与其他四后端对比：**
- MuJoCo — 深度真实的物理，主用 Go2/机械臂
- Genesis — 高吞吐场景生成
- Isaac — GPU 并行大规模训练
- Unity — VLA Challenge benchmark 基线
- engines/ — 后端抽象层 + registry
```

### Step E.4: robot-platforms.md §0 统一配置层

- [ ] **定位:** `grep -n "^## §0\|^## §1\|平台总览" docs/architecture/robot-platforms.md`

- [ ] **Edit（在 §0 之后、§1 之前插入新子节 "统一机器人配置层（RobotConfig）"，~30 行）：**

```markdown
### 统一机器人配置层 — `robot/config.py` + `robot/model_parser.py` + `robot/catalog/`

替代原有的"per-robot 硬编码配置"模式。**单一真相源** = URDF/MJCF。

**入口：**
- `RobotConfig(BaseModel)` @ `robot/config.py` — 从 URDF/MJCF 生成 `RobotModelConfig`（模型）+ `HardwareComponent`（硬件组件列表）+ `TaskConfig`（任务约束）
- `GripperConfig` @ `robot/config.py` — 夹爪子配置

**解析：**
- `JointDescription` / `ModelDescription` / `parse_model()` @ `robot/model_parser.py` — URDF + MJCF 双解析器

**Catalog：** `robot/catalog/` 四家机器人模型定义：`openarm.py` / `panda.py` / `piper.py` / `ufactory.py`

**新机器人接入流程：**
1. 写 `catalog/<name>.py` + 准备 URDF/MJCF 资源
2. `RobotConfig.from_urdf(...)` 自动生成 blueprint 组件
3. 按 blueprint 注册流程进入 `all_blueprints.py`（见 AGENTS.md）
```

### Step E.5: README §6 子系统目录补行

- [ ] **定位 §6 目录表:** `grep -n "navigation\||hardware\||robot/\|simulation/" docs/architecture/README.md | head -20`

- [ ] **Edits：**
  - navigation 那一格末尾加："…；新增子包 `patrolling/`（async Module 范式，生产级巡逻）"
  - simulation 那一格末尾加："…；新增 Unity 后端（VLA Challenge）"
  - robot 那一格末尾加："…；新增 `catalog/` + `config.py` + `model_parser.py` 统一配置层"
  - hardware 那一格末尾加："…；新增 `whole_body/` 全身多关节控制 HAL"

### Step E.6: 验收 + pre-commit + commit

- [ ] **≥1-命中 Gate:**

```bash
grep -cE "WholeBodyAdapter|MotorCommand|whole_body/|PatrollingModule|PatrolRouter|UnityBridgeModule|RobotConfig\b|model_parser|robot/catalog/" docs/architecture/robot-platforms.md docs/architecture/subsystems.md docs/architecture/README.md
```

Expected: 每关键词 ≥1。

- [ ] **commit:**

```bash
source .venv/bin/activate
pre-commit run --files docs/architecture/robot-platforms.md docs/architecture/subsystems.md docs/architecture/README.md
git add docs/architecture/robot-platforms.md docs/architecture/subsystems.md docs/architecture/README.md
git commit -m "docs(architecture): batch E — 4 new mainline subsystems (whole_body, patrolling, unity, robot-config)"
```

- [ ] **派 Explore subagent 交叉验证**（spec §4.3 要求）：

> "读 `docs/architecture/robot-platforms.md` §0 与 §5.5 与 §6.6，以及 `docs/architecture/subsystems.md` §3 navigation/patrolling，对照 `origin/main` 的 `dimos/hardware/whole_body/`、`dimos/navigation/patrolling/`、`dimos/simulation/unity/`、`dimos/robot/{config,model_parser}.py` + `robot/catalog/` 找漏描述 / 错类名 / 错路径。只做 read-only，250 字内。"

---

## Task F: robot-platforms.md 机器人平台尾收

覆盖 spec v2 §3 批 F 全部 + §0.6.A 路径整改。**动笔时所有 Go2 路径前缀 `dimos/robot/unitree/go2/`，G1 前缀 `dimos/robot/unitree/g1/`**。

**Files:**
- Modify: `docs/architecture/robot-platforms.md`（§1.1 Go2 扩、§1.2 G1 扩、§3 drone 扩、§4 机械臂扩、§5 hardware 扩、§6 simulation 扩）

### Step F.0: 预检

- [ ] **Run（全部必须命中）：**

```bash
git grep -n "Go2Mode\b\|class Go2Mode\b\|RAGE" origin/main -- dimos/robot/unitree/go2/connection.py
git ls-tree origin/main -- dimos/robot/unitree/go2/cli/ dimos/robot/unitree/go2/connection_spec.py
git ls-tree origin/main -- dimos/robot/unitree/go2/blueprints/basic/unitree_go2_webrtc_rage_keyboard_teleop.py
git ls-tree origin/main -- dimos/robot/unitree/go2/blueprints/agentic/
git ls-tree origin/main -- dimos/robot/unitree/g1/wholebody_connection.py dimos/robot/unitree/g1/connection_spec.py
git ls-tree origin/main -- dimos/robot/unitree/g1/blueprints/basic/unitree_g1_coordinator.py
git ls-tree origin/main -- dimos/robot/unitree/g1/blueprints/
git ls-tree origin/main -- dimos/robot/manipulators/openarm/ dimos/hardware/manipulators/openarm/ dimos/hardware/manipulators/sim/
git ls-tree origin/main -- dimos/robot/drone/
git grep -n "class MujocoCamera\b" origin/main -- dimos/simulation/mujoco/depth_camera.py
git ls-tree origin/main -- dimos/simulation/engines/mujoco_shm.py dimos/simulation/engines/mujoco_sim_module.py dimos/simulation/engines/registry.py
git ls-tree origin/main -- dimos/hardware/drive_trains/unitree_go2/adapter.py dimos/hardware/drive_trains/transport/adapter.py
git ls-tree origin/main -- dimos/simulation/manipulators/ 2>&1 | head -3
```

末条应空（目录已删）。

### Step F.1: §1.1 Go2 扩

- [ ] **定位:** `grep -n "^## §1\.1\|^### §1\.1\|Go2\|unitree/go2" docs/architecture/robot-platforms.md`

- [ ] **目录树补**（若现有有 ASCII 树）：`cli/` + `connection_spec.py`；树结构里每条都要带 `dimos/robot/unitree/go2/` 完整前缀。

- [ ] **新子节 "Go2 rage mode"**（~15 行）：
  - `Go2Mode.RAGE` enum 值 @ `dimos/robot/unitree/go2/connection.py`（类声明行号 + 枚举行号通过 `git grep` 取，填入正文）
  - `enable_rage_mode()` 调用点
  - 专用蓝图：`dimos/robot/unitree/go2/blueprints/basic/unitree_go2_webrtc_rage_keyboard_teleop.py`（PR #1903，**在 `basic/` 下**，不在 `blueprints/` 顶层）

- [ ] **新子节 "Go2 CLI"**（~8 行）：`dimos/robot/unitree/go2/cli/` 工具（PR #1990）；与顶层 `dimos` CLI（§1.21 inventory）的关系——前者是 Go2 专属快捷命令，后者是跨平台统一入口。

- [ ] **Agentic MCP 蓝图 5 个全名列清单**（~10 行）：

```markdown
**硬件绑定 MCP 蓝图（5 个，均位于 `dimos/robot/unitree/go2/blueprints/agentic/`）：**
- `unitree_go2_agentic.py`
- `unitree_go2_agentic_huggingface.py`
- `unitree_go2_agentic_ollama.py`
- `unitree_go2_security.py`
- `unitree_go2_temporal_memory.py`

（`_common_agentic.py` 为共享辅助，不对外暴露。）
```

- [ ] **时间戳章节** 若存在：补 PR #1992（lidar 时间戳修复）+ PR #2021（自适应时间戳矫正）一句提及。

### Step F.2: §1.2 G1 扩

- [ ] **定位:** `grep -n "^## §1\.2\|G1\|unitree/g1\|wholebody_connection" docs/architecture/robot-platforms.md`

- [ ] **目录树补:** `wholebody_connection.py` + `connection_spec.py`；全条目前缀 `dimos/robot/unitree/g1/`。

- [ ] **新子节 "G1 全身控制分工"**（~12 行）：
  - `connection.py` **仍是主 entry**（高层运动命令）
  - `wholebody_connection.py` **追加**低层全身控制通道（PR #1954，**不取代** connection.py）
  - `connection_spec.py` — Spec Protocol
  - 交叉引用：`wholebody_connection.py` 实现的就是批 E.1 里 `dimos/hardware/whole_body/spec.py` 定义的 `WholeBodyAdapter(Protocol)`

- [ ] **新蓝图:** `dimos/robot/unitree/g1/blueprints/basic/unitree_g1_coordinator.py`（在 `basic/` 下）

- [ ] **G1 blueprints 子目录结构**（~5 行）：四子类别 `agentic/` / `basic/` / `perceptive/` / `primitive/`——每类 1 句职责。

### Step F.3: §4 机械臂（两层矩阵）

- [ ] **定位:** `grep -n "^## §4\|^### §4\|manipulators/\|piper\|xarm\|openarm" docs/architecture/robot-platforms.md`

- [ ] **Edit 分层描述：**
  - `dimos/robot/manipulators/` 3 家：`piper/` / `xarm/` / `openarm/`（**无 mock**，mock 只在 hardware 层）——**新增 openarm**（PR #1897）
  - `dimos/hardware/manipulators/` 5 实现：`mock/` / `piper/` / `xarm/` / `openarm/`（新）/ `sim/`（新）

- [ ] **新子节 "OpenArm Integration"**（~12 行）：
  - `dimos/robot/manipulators/openarm/`
  - `dimos/hardware/manipulators/openarm/{adapter, driver, test_driver}.py`
  - 随行新增：`dimos/types/manipulation.py`（8 类）+ `robot_capabilities.py`（批 G.3 展开）+ `dimos/skills/manipulation/` 6 个 skill（批 G.4 展开）+ `dimos/utils/workspace.py`（314 行 URDF/workspace 工具）

- [ ] **新子节 "Sim Adapter"**（~8 行）：
  - `dimos/hardware/manipulators/sim/adapter.py` + `test_shm_adapter.py`
  - sim-backed `ManipulatorAdapter`（shm 与 MuJoCo 同步）
  - 为何新：替代已删除的 `dimos/simulation/manipulators/` 顶级目录

- [ ] **对比表**（~8 行）：robot/ 3 列 × hardware/ 5 列两层矩阵（markdown 表格）。

### Step F.4: §3 drone 扩写（占位 → 实内容）

- [ ] **Read 现有 §3:** `grep -n "^## §3\|^### §3\|drone\|MAVLink\|DJI" docs/architecture/robot-platforms.md` 并 Read 前后 50 行确认占位内容。

- [ ] **读 drone/README.md:**

```bash
test -f dimos/robot/drone/README.md && wc -l dimos/robot/drone/README.md
```

用 Read 工具读全文——用于正文素材。

- [ ] **Edit §3（补实而非重写）** ~40 行：
  - `connection_module.py` — 主连接
  - `mavlink_connection.py` — MAVLink 协议驱动
  - `drone_tracking_module.py` — 跟踪
  - `drone_visual_servoing_controller.py` — 视觉伺服控制器
  - `dji_video_stream.py` — DJI 视频流
  - `camera_module.py` — 相机模块
  - `blueprints/` — 子目录
  - 底部链到 `robot/drone/README.md`

### Step F.5: §5 hardware drive_trains 扩 + §5.4 sensors 注

- [ ] **Edit drive_trains:**
  - `drive_trains/unitree_go2/adapter.py` + `README` — Go2 硬件抽象
  - `drive_trains/transport/adapter.py` — 传输适配器

- [ ] **Edit sensors（§5.4）** 加说明：
  > "`sensors/camera/realsense/__init__.py` **删除** ≠ 目录消失；`camera.py` + `handeyeout_xarm6/` 仍在。导入方式改为 `from dimos.hardware.sensors.camera.realsense.camera import ...`。"

### Step F.6: §6 simulation 扩

- [ ] **Edits:**
  - `mujoco/depth_camera.py` 的 `MujocoCamera`（PR #1694）——第一个 sim 内感知模块
  - MuJoCo xarm+piper 遥操作（PR #1958）
  - `engines/mujoco_shm.py` / `mujoco_sim_module.py` / `registry.py`
  - `--simulation` flag（PR #2027）
  - replay 内存泄漏修复（PR #2025）
  - 一句："`simulation/manipulators/` 已**删除**，仿真内机械臂接口移至 `hardware/manipulators/sim/`（见 §4）"
  - Unity 章由 E.3 已覆盖，此处放交叉引用

### Step F.7: 验收 + pre-commit + commit

- [ ] **≥1-命中 Gate:**

```bash
grep -cE "unitree/go2/cli/|Go2Mode\.RAGE|unitree_go2_agentic_huggingface|unitree_g1_coordinator|wholebody_connection|manipulators/openarm|hardware/manipulators/sim|MujocoCamera|unitree_go2/adapter\.py|drone/mavlink_connection" docs/architecture/robot-platforms.md
```

Expected: 每关键词 ≥1。

- [ ] **0-命中 Gate（v1 路径残留检查）：**

```bash
grep -nE "(^|[^/a-zA-Z_])go2/(connection|cli|blueprints|connection_spec)|(^|[^/a-zA-Z_])g1/(connection|wholebody|blueprints)|blueprints/agentic/_huggingface|blueprints/agentic/_ollama|simulation/manipulators/[a-z]" docs/architecture/robot-platforms.md
```

Expected: 0（所有 unitree 路径必须带 `robot/unitree/` 前缀）。

- [ ] **commit:**

```bash
source .venv/bin/activate
pre-commit run --files docs/architecture/robot-platforms.md
git add docs/architecture/robot-platforms.md
git commit -m "docs(architecture): batch F — robot platforms tail (Go2 rage/CLI, G1 wholebody, OpenArm, drone, MujocoCamera, sim adapter)"
```

---

## Task G: 次级子系统补全 + round-2 整合

覆盖 spec v2 §3 批 G 全部 + §0.6.B.2/B.3/C.4/C.5/D（5 项 round 2 整合）+ §0.6.E.1（coordination 文件数精化，如需回补）。

**Files:**
- Modify: `docs/architecture/subsystems.md`（§1 control 代码块结构改 / §3 navigation 加 visual/query / §4 manipulation mermaid + servo_control / §7 / §11 / §12 / §14 新增 / 附录 experimental）
- Modify: `docs/architecture/README.md`（§5 simulation 表删 manipulators 行 / §6 stream 格子 / perception 硬依赖清单）
- Modify: `docs/architecture/agent-stack.md`（§3 skills/manipulation + skills/unitree + skills/rest）

### Step G.0: 预检 — 新关键类/字面值确在

- [ ] **Run：**

```bash
git grep -nE "class CartesianMotionController\b|class CartesianMotionControllerConfig\b" origin/main -- dimos/manipulation/control/servo_control/cartesian_motion_controller.py
git grep -n "def get_object_bbox_from_image\b" origin/main -- dimos/navigation/visual/query.py
git ls-tree origin/main -- dimos/manipulation/planning/monitor/ dimos/manipulation/planning/spec/
git ls-tree origin/main -- dimos/skills/manipulation/
git ls-tree origin/main -- dimos/msgs/sensor_msgs/ dimos/stream/ dimos/experimental/security_demo/
git ls-tree origin/main -- dimos/types/manipulation.py dimos/types/robot_capabilities.py
git grep -n "class WebsocketVisSpec\|WebsocketVisSpec" origin/main -- dimos/web/websocket_vis_spec.py
git ls-tree origin/main -- dimos/control/blueprints/
```

### Step G.1: subsystems.md §14 新增 `dimos/stream/`（~30 行）

- [ ] **定位:** `grep -n "^## \|§13\|rxpy_backpressure\|扩展阅读" docs/architecture/subsystems.md`

- [ ] **Edit（§13 rxpy_backpressure 之后、§扩展阅读 之前插入 §14）：**

```markdown
## §14 `dimos/stream/` — 应用层媒体流（新增）

**与 `dimos/core/stream.py` 不是一回事**——后者是 Module 间类型化数据通道（架构骨架），本节是应用层媒体流工具包。

文件职责：
- `audio/` — 音频流
- `data_provider.py` — 通用数据 provider 基类
- `frame_processor.py` — 帧级处理
- `video_provider.py` / `rtsp_video_provider.py` / `ros_video_provider.py` — 三源视频
- `stream_merger.py` — 多流融合
- `video_operators.py` — 视频变换算子
```

### Step G.2: subsystems.md 附录 `experimental/security_demo/`（~20 行）

- [ ] **定位:** `grep -n "^## 扩展阅读\|experimental" docs/architecture/subsystems.md`

- [ ] **Edit（§扩展阅读 之前插入"附录：实验模块（experimental/）"）：**

```markdown
## 附录 A：实验模块（`dimos/experimental/`）

API **不稳定**、不进 blueprint registry、不编号——因不稳定。

**`security_demo/`：**
- `depth_estimator.py` — 深度估计
- `security_module.py` — 安防模块主体
- `conftest.py` + `test_security_module.py` — 测试固件

**`temporal_memory/`：** PR #1511 spatio-temporal RAG（保持原状态描述）。
```

### Step G.3: subsystems.md §12 types 补

- [ ] **定位:** `grep -n "^## §12\|types/\|Vector\|Timestamped" docs/architecture/subsystems.md`

- [ ] **Edit 追加段：**
  - `manipulation.py` — 8 类：`ConstraintType` / `AbstractConstraint` / `TranslationConstraint` / `RotationConstraint` / `ForceConstraint` / `ObjectData` / `ManipulationMetadata` / `ManipulationTask`（OpenArm 随行）
  - `robot_capabilities.py` — `RobotCapability` enum
  - `ros_polyfill.py` / `sample.py` / `constants.py` 各一句

### Step G.4: agent-stack.md §3 skills 新增 `skills/manipulation/` 小节

- [ ] **定位:** `grep -n "^## §3\|^### \|skills/\|manipulation/" docs/architecture/agent-stack.md`

- [ ] **Edit 新增小节（~20 行）：**

```markdown
#### `skills/manipulation/` — 约束化机械臂动作（新增）

6 个 LLM 可调用 `@skill`（OpenArm 随行）：
- `abstract_manipulation_skill.py` — 基类
- `force_constraint_skill.py` — 力约束
- `manipulate_skill.py` — 通用操作
- `pick_and_place.py`
- `rotation_constraint_skill.py` — 旋转约束
- `translation_constraint_skill.py` — 平移约束

**分层：** `dimos/types/manipulation.py`（见 subsystems §12）定义约束类型；本处是执行层。

**其他位置的 skill：**
- `skills/unitree/unitree_speak.py` — Unitree 平台专用 TTS
- `skills/rest/rest.py` — REST 客户端 skill
```

### Step G.5: subsystems.md §11 msgs 补

- [ ] **定位:** `grep -n "^## §11\|msgs/\|sensor_msgs" docs/architecture/subsystems.md`

- [ ] **Edit 追加:**
  - `sensor_msgs/` 新增：`JointCommand.py` / `MotorCommandArray.py`（G1 wholebody 载体，见 robot-platforms §1.2）/ `RobotState.py` / `image_impls/`
  - `trajectory_msgs/`：`JointTrajectory.py` / `TrajectoryPoint.py` / `TrajectoryStatus.py`

### Step G.6: subsystems.md §7 models 补

- [ ] **定位:** `grep -n "^## §7\|models/\|vl/\|qwen/" docs/architecture/subsystems.md`

- [ ] **Edit:**
  - 路径纠正：`vl/video_query.py` 描述 → `models/qwen/video_query.py`（批 A 已改字串，此处确认周边正文与字串一致）
  - `vl/` 子树：确认列出 `moondream_hosted.py`

### Step G.7: subsystems.md §1 control 代码块结构改写

- [ ] **定位:** `grep -n "^## §1\|control/\|blueprints" docs/architecture/subsystems.md`

- [ ] **Read §1 的代码块**（含 control 目录树的那块），记录起止行号。

- [ ] **Edit:** 把树节点 `control/blueprints.py`（单文件）重排为：

```
control/blueprints/
├── basic.py
├── dual.py
├── mobile.py
└── teleop.py
```

- [ ] **Edit 补 `components.py` / `coordinator.py` / `hardware_interface.py`** 随 G1 wholebody PR #1954 的改动——每项一句。

### Step G.8: round 2 subagent 发现整合（5 项）

- [ ] **G.8.1** — **subsystems.md:206,226 §4 manipulation mermaid + 散文：**

```bash
grep -n "WorldMonitor\|planning/monitor" docs/architecture/subsystems.md
```

把 mermaid 节点 `MON[WorldMonitor<br/>planning/monitor/]` → 3 节点：
- `WM[WorldMonitor<br/>world_monitor.py]`
- `WOM[WorldObstacleMonitor<br/>world_obstacle_monitor.py]`
- `RSM[RobotStateMonitor<br/>robot_state_monitor.py]`

散文 "`WorldMonitor`（`planning/monitor/`）监控实时世界状态" → 改为 3 类并存简介（每句不超过 20 字）。

- [ ] **G.8.2** — **README.md:645 simulation 表删 `manipulators/` 行：**

```bash
grep -n "^| manipulators/\|sim_module\.py\|sim_manip_interface" docs/architecture/README.md
```

删该行；在 simulation 表下加一句："仿真内机械臂接口**已迁至** `hardware/manipulators/sim/`，见 robot-platforms §4。"

- [ ] **G.8.3** — **subsystems.md §3 navigation 加 `visual/query.py`：**

```markdown
**VLM→bbox 桥接** — `navigation/visual/query.py` 暴露 `get_object_bbox_from_image(vl_model, image, object_description)`，被 `skills/visual_navigation_skills.py` 调用，将 VLM 输出转为可用的 bbox。
```

- [ ] **G.8.4** — **subsystems.md §4 manipulation 加 `servo_control/`：**

```markdown
**伺服控制（Cartesian 空间）** — `manipulation/control/servo_control/cartesian_motion_controller.py`：
- `CartesianMotionController(Module)` — 伺服循环
- `CartesianMotionControllerConfig` — 配置
与 `trajectory_controller/`（关节空间路径）并列。
```

- [ ] **G.8.5** — **perception 可选 extra 跨子系统硬依赖清单**（加在 subsystems §2 perception 末或 README §6 perception 格子，选一）：

```markdown
**硬依赖子系统清单**（不装 `--extra perception` 会 ImportError）：
- `agents.skills.navigation`
- `manipulation/`（多个子模块）
- `models/vl/*`
- `navigation/visual_servoing/`
- `memory2/vis/utils.py`
- `experimental/security_demo/`

（全量文件 ≈ 62 个，此处仅列子系统；CI 可通过 `grep -l "^from dimos\.perception" dimos/` 枚举。）
```

### Step G.9: 其他小遗漏

- [ ] **`agents_deprecated/` 段**（agent-stack §6.5 / README §4 已经在批 C 处理；这里仅确认）：

```bash
grep -n "agents_deprecated\|dimos\.agents\.agent\.Agent" docs/architecture/agent-stack.md docs/architecture/README.md
```

若 `dimos.agents.agent.Agent` 仍作"推荐"出现 → 改为 `AgentSpec` / `demo_agent`（批 C 应已清，此处兜底）。

- [ ] **`web/websocket_vis_spec.py` 新 Spec**：subsystems.md §8 web 加一句。

- [ ] **`utils/workspace.py`**：批 F.3 "OpenArm Integration" 小节已提，此处不重写，只 grep 确认：

```bash
grep -n "utils/workspace\.py\|dimos/utils/workspace" docs/architecture/robot-platforms.md
```

Expected: ≥1 命中。

### Step G.10: 验收 + pre-commit + commit

- [ ] **≥1-命中 Gate:**

```bash
grep -cE "skills/manipulation/|types/manipulation\.py|robot_capabilities|MotorCommandArray|RobotStateMonitor|WorldObstacleMonitor|CartesianMotionController|get_object_bbox_from_image|control/blueprints/|dimos/stream/|security_demo|websocket_vis_spec" docs/architecture/subsystems.md docs/architecture/README.md docs/architecture/agent-stack.md
```

Expected: 每关键词 ≥1。

- [ ] **0-命中 Gate（batch G 残留）：**

```bash
grep -rnE "^\| manipulators/ \|\|planning/monitor/[^a-z]|control/blueprints\.py\b" docs/architecture/
```

Expected: 0。

- [ ] **commit:**

```bash
source .venv/bin/activate
pre-commit run --files docs/architecture/subsystems.md docs/architecture/README.md docs/architecture/agent-stack.md
git add docs/architecture/subsystems.md docs/architecture/README.md docs/architecture/agent-stack.md
git commit -m "docs(architecture): batch G — secondary subsystems (stream, security_demo, types/manipulation, skills/manipulation, msgs, control/blueprints/ tree, round-2 integrations)"
```

---

## Task H: 终极 0/≥1 命中双向验收 + push

覆盖 spec v2 §4.2 全部 gate。批 A–G 全绿后**一次**跑完所有 gate，通过再 push。

### Step H.1: 0-命中 gate（全 `docs/architecture/` 对以下 grep 必须 0）

- [ ] **Run（每条必须 0 行输出；`-E` 正则）：**

```bash
cd /Users/perry/workspace/dimos

# 旧路径
grep -rnE "dimos/core/(worker_manager|module_coordinator|blueprints|worker)\.py" docs/architecture/
grep -rnE "dimos/agents/agent\.py\b" docs/architecture/
grep -rnE "dimos/control/blueprints\.py\b" docs/architecture/
grep -rnE "dimos/mapping/types\.py\b" docs/architecture/
grep -rnE "vl/video_query\.py\b" docs/architecture/

# 旧类/接口名
grep -rnE "\bActor\b" docs/architecture/   # 允许 "legacy" / "已删除" 上下文
grep -rnE "\bMethodCallProxy\b" docs/architecture/
grep -rnE "\bSpatialPerception\b" docs/architecture/
grep -rnE "\bRerunBridge\b(?!Module)" docs/architecture/
grep -rnE "\bHasToRerun\b" docs/architecture/

# 旧 logger
grep -rnE "\bdimos\.core\.module_coordinator\b(?!\.)" docs/architecture/

# 旧论断
grep -rnE "unitree-go2-agentic-mcp.*唯一" docs/architecture/
grep -rnE "\bLangGraph\b|\bLangChain\b|\bcreate_agent\b" docs/architecture/

# 旧计数
grep -rnE "13 个一级子系统" docs/architecture/

# 旧 go2/g1 路径（必须带 robot/unitree/ 前缀）
grep -rnE "(^|[^/a-zA-Z_])go2/(connection|cli|blueprints|connection_spec)" docs/architecture/
grep -rnE "(^|[^/a-zA-Z_])g1/(connection|wholebody|blueprints)" docs/architecture/
grep -rnE "blueprints/agentic/_(huggingface|ollama)\.py" docs/architecture/
```

**任意一条非 0**：逐条定位，开"hotfix" commit 修——**不 push**。全部 0 再继续。

`\bActor\b` 允许的上下文：位于明确标注"legacy"/"已删除"/"已被 rpyc 取代"的段落内。逐行人工审；如有疑问保守删除。

### Step H.2: ≥1-命中 gate

- [ ] **Run（每个关键词全 `docs/architecture/` 内 ≥1 命中）：**

```bash
cd /Users/perry/workspace/dimos
for kw in \
  "coordination/" "AgentSpec" "agent_spec\.py" "demo_agent(_camera)?" \
  "ToolStream" "TOOL_STREAM_TOPIC" "porcelain" "\bDimos\b" \
  "memory2" "StreamModule" "Transformer\[" "VoxelMapTransformer" \
  "WorkerManagerPython" "WorkerManagerDocker" "RpycServer" "RPCClient" \
  "ModuleProxyProtocol" "watchdog_main" "NativeModule" "WorkerResponse" \
  "load_blueprint" "load_module" "rpyc_port" \
  "OpenArm|openarm" "Go2Mode\.RAGE" "wholebody_connection" "WholeBodyAdapter" \
  "MotorCommand" "PatrollingModule" "PatrolRouter" "UnityBridgeModule" \
  "\bRobotConfig\b" "model_parser" "MujocoCamera" "dimos/robot/drone" \
  "skills/manipulation/" "types/manipulation\.py" "robot_capabilities" \
  "RerunBridgeModule" "RerunConvertible" \
  ; do
  count=$(grep -rlE "$kw" docs/architecture/ | wc -l)
  if [ "$count" -eq 0 ]; then echo "MISSING: $kw"; fi
done
```

Expected: 无 `MISSING:` 输出。有 → 开 hotfix commit 补。

### Step H.3: 计数验收（14 一级子系统）

- [ ] **Run：**

```bash
grep -n "14 个一级子系统\|\*\*14\*\*" docs/architecture/README.md
grep -nE "^## §14\|^### §14\|dimos/stream/" docs/architecture/subsystems.md
```

Expected: 两条都有命中。

### Step H.4: 派 round-3 Explore subagent 做终审

- [ ] **Agent call（Explore，read-only）：**

> "你是终审 reviewer。目标 ref `origin/main`。读 `docs/architecture/` 5 份文档**全部**，对每份找出：(1) 与 origin/main 代码不符的类名/路径；(2) 遗漏的主干子系统；(3) 内部自相矛盾（批与批间、§之间）。不要引用任何外部来源，只用 `git ls-tree origin/main` / `git grep origin/main` 证。500 字内报告，按文件分段，每条含 evidence 命令。"

**若报告出具体 evidence 的错**：开 hotfix commit 修；修完回 Step H.1 重跑 gate。
**若报告仅是主观建议无 evidence**：忽略。

### Step H.5: 最终一次 push

- [ ] **Run：**

```bash
git log --oneline origin/docs/architecture..HEAD 2>/dev/null || git log --oneline origin/main..HEAD | head -20
```

确认 A–G 共 7 commits（+ 可能的 hotfix）都在本地。

- [ ] **Push:**

```bash
git push -u origin docs/architecture
```

Expected: 推成功。CI 启动约 1h。

- [ ] **等 CI 结果：** 用 Monitor 工具 tail PR 状态；CI 失败看 log 修，开 hotfix commit 再 push（**不 force push**，除非明确允许）。

---

## Self-Review 清单

- [ ] **Spec 覆盖：** 批 A/B/C/D/E/F/G 的每 Step 对应 spec v2 §3 哪一节？
  - A → spec §3 批 A 表 + §0.6.B.1
  - B → spec §3 批 B + §0.6.E.2
  - C → spec §3 批 C
  - D → spec §3 批 D
  - E → spec §3 批 E（4 新子系统）
  - F → spec §3 批 F + §0.6.A（路径整改）
  - G → spec §3 批 G + §0.6.B.2/B.3/C.4/C.5/D（round 2 整合）
  - H → spec §4.2 双向 gate + §4.3 subagent 验证 + §6 交付清单尾项
- [ ] **占位扫描**：无 TBD / TODO / "fill in" / "similar to Task N" / "implement later"——本 plan 每 Step 都有具体 grep/edit 指令。
- [ ] **类型一致性**：本 plan 不定义新类型，只引用 main 既有类；所有新关键词（`AgentSpec` / `StreamModule` / `WholeBodyAdapter` 等）在对应 Step 0 用 `git grep` 核实过。
- [ ] **单 Edit ≤50 行约束**：每个新段（§5.5 / §6.6 / §14 / 附录等）独立 Edit 一次，正文 ≤50 行；超过则拆两次 Edit（如 memory2 §6 全章 + Porcelain §4.5 + Drone §3）。

---

## 风险提醒（开工时重读）

1. **pre-commit 失败不 amend**：修 → 新 commit。如果批 A 因 markdownlint 失败，产生 "batch A" + "batch A: fix markdownlint" 两 commit 可接受；不要 amend。
2. **不 push 每批**：spec §6 明示"7 commit 全部本地完成 → 一次 push"。CI ≈ 1h，push 7 次 ≈ 7h。
3. **行号漂移**：60+ commits 后 `docs/architecture/` 自身行号也漂了；每批开工前重 `grep -n` 定位，**不要**引用本 plan 的"line ≈ 301-310"字面作 Edit 锚点。
4. **路径前缀规则**：所有 `unitree/go2/*` 路径必须带 `dimos/robot/` 前缀（同理 `unitree/g1/*`）；批 F 开工前读 spec §0.6.A 表再动笔。
5. **双写隐患**：`whole_body/` 只在 robot-platforms.md §5.5 写；`unity/` 只在 §6.6；不要在 subsystems.md 重写（spec §0.5 明示的位置锁定）。

