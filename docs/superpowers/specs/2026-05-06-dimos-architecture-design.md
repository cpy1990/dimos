# DimOS 系统架构文档 — 设计规格

- **日期**：2026-05-06
- **状态**：Draft，待 review
- **目标产出位置**：`docs/architecture/`

## 1. 背景

DimOS 仓库当前已有以下文档分类：

| 路径 | 主题 |
|------|------|
| `docs/usage/` | 模块、蓝图、配置、数据流、传输、CLI 等使用指南 |
| `docs/development/` | 测试、Docker、profiling、大文件管理等开发指南 |
| `docs/capabilities/` | 操控、导航、感知、agents 能力分类 |
| `docs/platforms/` | 具体硬件平台（Go2、G1）配置 |
| `docs/installation/` | 各 OS 安装 |
| `docs/agents/` | 智能体文档系统（meta） |
| `AGENTS.md` | AI agent onboarding（仓库根） |

**缺失**：一份**贯通整体的系统架构文档**——把 Module / Blueprint / Skill 三件抽象与 Agent 栈、传输栈、平台层、能力子系统串成一张图、讲清数据如何流动、进程如何隔离。新工程师目前需要在 6 个文档目录间跳转拼凑全貌。

本规格定义新文件夹 `docs/architecture/` 的结构、每份文件的覆盖范围、写作约束。

## 2. 目标与非目标

### 目标

1. 新工程师**只读 `docs/architecture/README.md` 一份文档**就能完整建立 DimOS 整体心智模型（预计阅读 1.5 小时）
2. 每个一级子系统在 README 中都有**实质性介绍**（核心子系统 ≥350 字 / 辅助子系统 ≥150 字 + 关键文件路径 + 设计取舍）；不能仅以"一句话+跳转链接"呈现
3. README 之外的 5 份专题文档供"想深入某个细节"的读者使用，**不是**新人理解整体的必经之路
4. 全部中文
5. Mermaid（全景图、类图、时序图、进程关系图）+ ASCII（段落内小图）混合

### 非目标

- 不重复 `docs/usage/`、`docs/capabilities/`、`docs/platforms/` 已有的"怎么用"教程
- 不替代 `AGENTS.md`，但补它在"系统全景"上的短板（AGENTS.md 偏 quick-start 与 cheat sheet）
- 不写入大量代码示例，最多片段化示意；以概念图、关系表、设计取舍说明为主

## 3. 文件夹与文件

```
docs/architecture/
├── README.md            # 主文档：完整自包含全景  (~10000-14000 字)
├── runtime-model.md     # 专题 1：进程/Stream/Transport/Protocol/Config/日志  (~2500 字)
├── agent-stack.md       # 专题 2：Agent/Skill 双体系/MCP/Spec/系统提示词  (~2500 字)
├── robot-platforms.md   # 专题 3：robot/ + hardware/ + simulation/ 硬件栈  (~2500 字)
├── subsystems.md        # 专题 4：内部架构 / 依赖 / 扩展点（与 README § 6 互补，不重复）  (~5500-7000 字)
└── data-flow.md         # 专题 5：端到端数据流 trace  (~2000 字)
```

字数为目标量，实际允许 ±20% 浮动。

## 4. README.md 章节细则

主目标：**通读后无需跳转就能理解整个系统**。任何一段都不能是"详见某文档"占位。

### § 0. DimOS 是什么（约 400-600 字 + 顶层全景图 + 与其他文档关系小框）

- 一句话定位：「面向通用机器人的智能体操作系统」
- 解决什么问题（机器人系统耦合 → 模块化 + 智能体）
- 一张顶层 Mermaid 全景图：硬件 ↔ Modules（流水线） ↔ Stream（数据通道） ↔ Agent（决策） ↔ User（自然语言/MCP/Web/Teleop）
- **本文档与 `AGENTS.md` / `CLAUDE.md` 的分工**（一段小框）：
  - `AGENTS.md` = quick-start cheat-sheet 与必踩坑清单（命令速查、blueprint 表）
  - `CLAUDE.md` = AI 代理工作护栏（指向 `AGENTS.md`，加少量 Claude 专用规则）
  - 本架构文档 = 系统全景与设计取舍
  - 三者交叉引用，不重复

### § 0.5. 仓库布局总览（约 250-350 字 + ASCII 树 + 顶级目录/文件速查表）

- ASCII 目录树：仓库根 1 级目录全景
- 表格速查（必须**完整**覆盖以下，不漏一项）：
  - **顶级目录**：`dimos/`（主包）、`bin/`（脚本）、`scripts/`（仓库脚本）、`data/`（数据/replay 数据集）、`docker/`（容器）、`examples/`（示例代码）、`assets/`（资源文件）、`docs/`（文档）
  - **顶级文件**：`pyproject.toml`、`setup.py`、`uv.lock`、`flake.nix` / `flake.lock`、`default.env`、`MANIFEST.in`、`LICENSE`、`CLA.md`、`README.md`、`CLAUDE.md`、`AGENTS.md`
- 一句话提示：`dimos/utils/` 是 30+ 个横切工具（logging_config / llm_utils / transform_utils / gpu_utils / threadpool / urdf 等），被几乎所有子系统依赖；本文档不展开，需要时直接读源码

### § 1. 三个核心抽象（约 1000-1200 字 + 关系图 + 最小示例）

- **Module**：自治子系统，运行在独立 forkserver worker 进程；声明类型化的 `In[T]/Out[T]` 流；用 `@rpc` 暴露调用面
- **Blueprint**：用 `autoconnect()` 把 Module 拼成可运行栈；流按 `(name, type)` 自动连接
- **Skill**：智能体可调用的物理动作；`@skill` 装饰器自动生成 LLM tool schema
- 三者关系图（Mermaid 类图）
- 一个最小可运行 Blueprint 片段（≤15 行代码）说明三者协作
- 设计取舍说明：为什么类型化 stream、为什么 forkserver、为什么 @skill 与 @rpc 分层

### § 2. 通信骨架（约 1000-1200 字 + Transport 选型表 + Stream/Transport/Protocol 三层 Mermaid 分层图）

- **Stream（`dimos/core/stream.py`）**：`In[T]/Out[T]` 类型化数据通道——架构骨架
- **Transport（`dimos/core/transport.py`）**：实际有 9 个类。主用 6 个：`LCMTransport` / `pLCMTransport` / `SHMTransport` / `pSHMTransport` / `ROSTransport` / `DDSTransport`；视频专用 JPEG 编码变体 2 个：`JpegLcmTransport` / `JpegShmTransport`（专为视频流降低 CPU 拷贝代价）；占位 1 个：`ZenohTransport`（尚未投入使用）。完整对比表（含跨语言/跨主机/零拷贝/适用场景列），主用 6 个详写，JPEG 变体一段说明，Zenoh 一句话标"占位"
- **Protocol（`dimos/protocol/`）**：比 transport 更底层的封装层——`encode/`、`pubsub/`（含 `impl/{lcmpubsub,shmpubsub,jpeg_shm,ddspubsub,rospubsub,redispubsub,memory}.py` 多个具体实现）、`rpc/`（含 `pubsubrpc.py`、`redisrpc.py`、`spec.py`、`rpc_utils.py`）、`service/`、`tf/`；解释 `core/transport.py` 是 stream-facing 的 Transport 抽象，`protocol/pubsub/impl/*` 是真正的 pub/sub 实现层
- **关键避坑（命名重叠 1）**：明确区分两种 Stream 含义
  - `dimos/core/stream.py`：模块间数据通道（架构骨架）
  - `dimos/stream/`：视频/音频 data provider（RTSP、ROS video、frame_processor、video_operators、audio）
  - 两者**只是名字相同**，不是同一概念

### § 3. 运行时模型（约 800-1000 字 + 进程关系图）

- **forkserver worker** 模型：为什么不是 fork，子模块隔离的原因
- **WorkerManager / ModuleCoordinator**：谁负责 worker 生命周期、谁负责 stream 接线
- **GlobalConfig cascade**：默认 → `.env` → 环境变量（`DIMOS_*`） → Blueprint 覆写 → CLI 标志
- **Daemon / RunRegistry**：`~/.local/state/dimos/runs/<run-id>.json`、日志路径 `~/.local/state/dimos/logs/<run-id>/main.jsonl`
- **CLI 命令族总览**：run / status / log / stop / restart / list / show-config / mcp / agent-send / topic echo / topic send / lcmspy / agentspy / humancli / top / rerun-bridge
- **关键避坑（命名重叠 2）**：`agent-send`（CLI 命令名，连字符）与 `agent_send`（底层 MCP 工具名，下划线）是同一动作的两层入口，写文档/代码时区分

### § 4. Agent 系统（约 1500-1800 字 + 类图 + 双 Skills 对比表）

- **Agent 体系实际形态**（按代码核实）：
  - `Agent`（`dimos/agents/agent.py`）：LangGraph + LangChain `create_agent`；默认 `model="gpt-4o"`；通过 `model.startswith("ollama:")` 路由到本地 Ollama，**Ollama 不是独立子类，是模型选择**
  - `VLMAgent`（`dimos/agents/vlm_agent.py`）：独立 `Module` 类，专为视频流推理；与 `Agent` 平级
  - `WebHumanInput`（`dimos/agents/web_human_input.py`）：人类介入输入面，独立 Module
  - `dimos/agents/ollama_agent.py`：仅暴露 `ensure_ollama_model` / `ollama_installed` 工具函数，不是 Agent 类
- **关键避坑（命名重叠 3）：legacy 包 `dimos/agents_deprecated/`**
  - 是上一代 OpenAI/Claude agent 实现（`agent.py`、`claude_agent.py`、`memory/`、`modules/`、`prompt_builder/`、`tokenizer/`）
  - 当前仍有 `dimos/web/dimos_interface/api/README.md` 的 `from dimos.agents_deprecated.agent import OpenAIAgent` 过渡引用
  - **不要在新代码引用**；新人 grep `class Agent` 会撞两个包，要分清
- **关键避坑（命名重叠 4）：两套 Skills 体系**

  | 路径 | 实际内容 | 结论 |
  |------|---------|-----|
  | `dimos/skills/` | `kill_skill`、`speak`、`visual_navigation_skills`、`manipulation/`、`rest/`、`unitree/`（含 `unitree_speak.py`） | 历史 / 平台无关 / 平台特化都有 |
  | `dimos/agents/skills/` | `person_follow`、`gps_nav_skill`、`navigation`、`speak_skill`、`google_maps_skill_container`、`osm` | 历史 / 第三方集成 / 通用都有 |

  **诚实结论**：两处分布是历史演进结果，不存在干净的"通用 vs 特化"对仗。加新 skill 时贴近其依赖位置即可；详细判断决策树留给 `agent-stack.md`
- **关键避坑（命名重叠 5）：`dimos/skills/manipulation/` ≠ `dimos/manipulation/`**
  - 前者是 LLM 调用面（`@skill` 装饰过的 6 个 `*_skill.py`：abstract / force_constraint / manipulate / pick_and_place / rotation_constraint / translation_constraint）
  - 后者是模块/算法层（subsystem）
  - 两者通过 RPC `Spec` 串联
- **关键避坑（命名重叠 6）：speak 在 3 个位置出现**
  - `dimos/skills/speak.py`、`dimos/skills/unitree/unitree_speak.py`、`dimos/agents/skills/speak_skill.py`——都是不同的 speak 实现，使用前确认意图
- **`@skill` 装饰器**：4 条硬规则（docstring 必填、参数全注解、返回 `str`、不与 `@rpc` 叠加）；破坏后的失败模式
- **RPC Wiring**：
  - 推荐：`Spec` Protocol 类型注入（`dimos/spec/`：`control.py`、`mapping.py`、`nav.py`、`perception.py`、`utils.py`）
  - Legacy：`rpc_calls: list[str]` + `get_rpc_calls(...)`（运行时静默失败）
- **关键避坑（命名重叠 7）：`dimos/spec/` ≠ `dimos/protocol/rpc/spec.py`**
  - `dimos/spec/` 是用户面 Protocol 定义（写新 skill 时声明依赖用）
  - `dimos/protocol/rpc/spec.py` 是 RPC 实现层内部辅助
  - 命名巧合，作用不同
- **MCP 三件套**：`McpServer` + `McpClient` + `McpAdapter`，与 in-process Agent **互斥**；当前唯一 ship 的 MCP-enabled blueprint 是 `unitree-go2-agentic-mcp`
- **系统提示词**：Go2 默认 vs `G1_SYSTEM_PROMPT`；用错会导致 LLM 幻觉技能

### § 5. 机器人平台层（约 1000-1200 字 + 平台拓扑图）

- **`dimos/robot/` 树（按层级精确写）**：
  - `dimos/robot/unitree/{go2,g1,b1,modular}/`、`dimos/robot/unitree/{connection,mujoco_connection,keyboard_teleop,rosnav,demo_error_on_name_conflicts}.py`、`dimos/robot/unitree/{params,modular,testing,type}/`、`dimos/robot/unitree/unitree_skill_container.py`
  - `dimos/robot/unitree_webrtc/`（**注意**：是 `unitree/` 的兄弟目录，不是子目录）
  - `dimos/robot/drone/`：MAVLink + DJI 视频流 + 视觉伺服 + tracking + camera/connection 模块
  - `dimos/robot/manipulators/{piper,xarm}/`
  - `dimos/robot/cli/`：CLI 入口（`dimos.py`）
  - `dimos/robot/{foxglove_bridge,ros_command_queue,position_stream,robot,get_all_blueprints}.py`、`all_blueprints.py`（自动生成）
- **`dimos/hardware/`**：硬件抽象层
  - `drive_trains/`、`end_effectors/`、`manipulators/`、`sensors/`
- **`dimos/simulation/`**：多引擎仿真
  - `mujoco/`、`genesis/`、`isaac/`、`engines/`、`base/`、`manipulators/`、`utils/`、`sim_blueprints.py`
- 仿真 vs 真机切换：`--simulation`、`--replay`、`--robot-ip` 三个标志的语义
- 添加新机器人 / 新末端执行器 / 新传感器的入口路径
- 末尾"成熟度标注"（按 git log 实际状态）：
  - **稳定主线**：Go2、G1、xArm
  - **活跃演进**：drone（PR #1520 `feat(drone): modernize drone for CLI + Rerun + replay` 是最近重构）
  - **探针级别 / 未投入使用**：`dimos/robot/unitree/modular/`（仅 `detect.py` 一个文件，git log 仅 2 commit，最近 #1365 是 remove dask）
  - 不展开演进路线，避免文档与代码脱节

### § 6. 能力子系统全景（约 3500-5500 字，按复杂度分档）

每个子系统给：**职责一段** + **关键文件路径** + **发布/订阅的主要 stream** + **设计取舍** + **链到现有详细文档**

§ 6 起首一张 Mermaid 子系统拓扑图：13 个子系统的依赖与数据流向。

#### 核心子系统（每个 350-500 字）

1. `dimos/control/`：低层控制循环框架（`tick_loop`、`coordinator`、`components`、`tasks/`、`hardware_interface`、`blueprints.py`）；与上层 Module 的关系
2. `dimos/perception/`：**子系统拆为两层**——
   - 检测核心：`dimos/perception/detection/`（`module2D.py`、`module3D.py`、`moduleDB.py`、`objectDB.py`、`person_tracker.py`、`detectors/`、`reid/`、`type/`）
   - 上层封装：`object_tracker_2d.py` / `object_tracker_3d.py` / `object_tracker.py`、`spatial_perception.py`、`object_scene_registration.py`、`perceive_loop_skill.py`
   - 实验性：`experimental/temporal_memory/`（PR #1511 引入的 spatio-temporal RAG）
3. `dimos/navigation/`：A* 重规划、前沿探索、视觉伺服、bbox 导航、ROS Nav 集成（`replanning_a_star`、`frontier_exploration`、`visual_servoing`、`bbox_navigation`、`rosnav`、`visual`）
4. `dimos/manipulation/`：抓取、规划、pick & place（`manipulation_module`、`pick_and_place_module`、`grasping/`、`planning/`、`control/`、`manipulation_interface`、`blueprints`）
5. `dimos/mapping/`：occupancy grid、voxels、costmapper、osm/google_maps、pointclouds
6. `dimos/memory/`：**只含**`embedding.py`（spatial entries） + `timeseries/`（pluggable backends：inmemory / pickledir / sqlite / postgres / legacy）
   - **关键避坑（命名重叠 8）**：temporal memory **不在**这里，在 `dimos/perception/experimental/temporal_memory/`（PR #1511 引入）；不要混淆
7. `dimos/models/`：ML 模型封装（`qwen`、`vl`、`segmentation`、`embedding`、`base.py`）；被 perception/agents 共用
8. `dimos/web/`：FastAPI + Flask + WebSocket + 命令中心扩展（`command-center-extension`、`dimos_interface`、`websocket_vis`、`edge_io`、`fastapi_server`、`flask_server`、`robot_web_interface`）

#### 辅助子系统（每个 150-250 字）

9. `dimos/visualization/rerun/`：Rerun bridge；与 `dimos/robot/foxglove_bridge.py` 的对比
10. `dimos/teleop/`：keyboard、phone、quest VR
    - 一句话提醒：`dimos/teleop/keyboard/keyboard_teleop_module.py`（通用）与 `dimos/robot/unitree/keyboard_teleop.py`（unitree 专用）是两份不同实现
11. `dimos/msgs/`：ROS 兼容消息类型（geometry、sensor、nav、vision、tf2、std、trajectory、visualization、foxglove）
12. `dimos/types/`：跨模块共享类型（`Vector`、`Timestamped`、`RobotLocation`、`RobotCapabilities` 等；含 `ros_polyfill` 用于无 ROS 环境）
13. `dimos/rxpy_backpressure/`：反应式背压（`drop`、`latest`、`function_runner`、`locks`、`observer`）；调试 stream 卡顿绕不开

### § 7. 端到端数据流（约 800-1000 字 + 时序图）

- 一条具体例子："说出'走到红色物体'"（明确选 perception.detection.module2D + replanning_a_star 这条最短路径）
- 简化时序：摄像头 → Perception（物体检测） → Memory（短时记忆） → Agent（LLM 决策） → Skill（导航） → Robot 控制 → 反馈 → TTS
- 涉及的关键 stream 名 / 类型 / transport（**只列锚点**，详细字段、Replay 模式差异、第二个 manipulation trace 全部留给 `data-flow.md`）
- README §7 末尾明示："详细字段级 trace、`--replay` 模式差异、manipulation trace 见 `data-flow.md`"

### § 8. 怎么继续读 + 常见踩坑（约 200-300 字）

- 五个专题文档（runtime-model / agent-stack / robot-platforms / subsystems / data-flow）— 何时翻
- `AGENTS.md`：cheat sheet 与"必踩的坑"清单（不在本文档重复，直接交叉引用）
- `docs/usage/`：使用教程
- `docs/capabilities/`：能力深度文档
- `docs/platforms/`：具体硬件平台
- `docs/installation/`：安装指引
- `docs/development/`：开发与测试

---

## 5. 五份专题文档细则

### 5.1 `runtime-model.md`

- 进程模型：fork vs forkserver、worker 隔离、`module_coordinator.py` / `worker_manager.py` / `worker.py` 关系
- Stream 内部：`autoconnect` 的 `(name, type)` 匹配算法、remappings 解决冲突
- 9 个 Transport 详细对比（主用 6 + JPEG 编码变体 2 + Zenoh 占位 1；列消息大小、频率上限、跨语言、跨主机、零拷贝、适用场景）
- `dimos/protocol/{encode,pubsub,rpc,service,tf}` 详解：与 transport 的分层关系
- GlobalConfig cascade 完整链路 + Configurable 装饰器
- Daemon 实现：`dimos/core/daemon.py`（守护进程） + `run_registry.py`（运行注册表） + `log_viewer.py`（`dimos log` CLI 后端：按 run-id 解析 JSONL 日志并彩色输出）
- 每运行日志结构：JSONL schema

### 5.2 `agent-stack.md`

- Agent 内部循环：LangGraph + LangChain `create_agent`，默认 `gpt-4o`，`ollama:` 前缀路由本地 Ollama；state / tools / 决策步
- `@skill` schema 生成深入：`annotation.py` 实现、4 条硬规则的失败模式
- 两套 skills 体系深度对比 + 何时把 skill 放哪里的判断决策树（依赖第三方 SDK / 是否需要 langchain 工具上下文 / 是否平台无关）
- Spec Protocol 编译期类型检查 + remappings + 多匹配解决
- MCP 三件套（Server/Client/Adapter）协议层与 HTTP 端点
- agent 变种与"非变种"清单：`Agent`（`agent.py`）/ `VLMAgent`（`vlm_agent.py`）/ `WebHumanInput`（`web_human_input.py`）/ `dimos/agents/ollama_agent.py`（仅工具函数，非 Agent 类）/ `dimos/agents_deprecated/`（legacy，勿用）

### 5.3 `robot-platforms.md`

- `dimos/robot/unitree/`：`{connection,mujoco_connection,keyboard_teleop,rosnav}.py`、`{go2,g1,b1,modular,params,testing,type}/`、`unitree_skill_container.py`
- `dimos/robot/unitree_webrtc/`：注意是 `unitree/` 兄弟目录而非子目录
- `dimos/robot/drone/`：MAVLink、DJI 视频流、视觉伺服、跟踪、camera/connection 模块（PR #1520 重构后形态）
- `dimos/robot/manipulators/{piper,xarm}`
- `dimos/hardware/`：四类抽象（drive_trains / end_effectors / manipulators / sensors）
- `dimos/simulation/`：四种仿真后端深入对比（mujoco / genesis / isaac / engines）
- 蓝图示例：`unitree-g1-agentic-sim`、`xarm-perception-agent` 等
- 添加新机器人 / 末端执行器 / 传感器路径与样板
- 成熟度提示：`unitree/modular/` 仅 `detect.py`，非投入使用

### 5.4 `subsystems.md`（与 README § 6 职责分工，**不重复字数膨胀**）

README § 6 的职责：**做什么 + 关键文件路径 + 设计取舍**（功能视角，新人建立心智模型）。

`subsystems.md` 的职责（**互补**，不重复）：

- **内部架构**：子系统内部模块如何协作（每个核心子系统给一张内部架构小图）
- **依赖关系**：上游 / 下游子系统、共享的 `Spec` Protocol、共享的 `Stream` 名
- **扩展点**：在哪里、如何添加新检测器 / 新规划器 / 新末端执行器
- **与现有 `docs/capabilities/` 的交叉链接**：本文档负责架构层；具体 how-to 留给 `docs/capabilities/{manipulation,navigation,perception,agents}/`
- 每个核心子系统 ≥ 400 字内部架构 + 依赖 + 扩展点；辅助子系统 ≥ 250 字
- **不**复制 README § 6 的"做什么"叙述

### 5.5 `data-flow.md`

- 完整 trace："去红色物体"端到端，列出每一步涉及的：
  - Module
  - Stream（名字、类型）
  - Transport
  - 决策点
- Mermaid `sequenceDiagram`
- `--replay` 模式数据流变化（数据源切换 + 时间戳重放）
- 第二个 trace 示例："拿起杯子"（manipulation 路径）

## 6. 撰写约束

- 全部中文
- 每份文件开头放目录跳转锚点
- 每份文件末尾"扩展阅读"链接相关文档
- 主文件名 `README.md`（GitHub 默认渲染）
- 图：Mermaid 用于全景/类图/时序/进程关系；ASCII 用于段落内小图
- 代码片段：仅在确实清晰受益时使用，不超过 15 行
- 路径引用使用相对路径（`../../dimos/core/module.py` 等）

## 7. 实现顺序

1. 写 `README.md` 全文（最优先、最长、最关键）
2. 写 5 份专题文档（任意顺序，可后续补充）
3. 互相校对内链与术语一致
4. 自审核对："新人通读 README 即建立完整心智模型"硬目标 + § 8 验收清单

## 8. 验收标准

**正向条目**：

- [ ] `docs/architecture/README.md` 字数在 10000-14000 字之间（自包含；如确有内容可超出）
- [ ] § 6 覆盖 13 个一级子系统：核心 8 项每个 ≥ 350 字、辅助 5 项每个 ≥ 150 字（§ 6 总字数 3500-5500）
- [ ] § 4 明确区分两套 Skills 体系（含对比表，承认历史成因，不强行对仗）
- [ ] § 4 明示 `dimos/agents_deprecated/` 的存在与"勿用"立场
- [ ] § 2 明确区分两种 Stream 概念
- [ ] § 2 准确陈述 transport 数量为 9（主用 6 + JPEG 变体 2 + Zenoh 占位 1）
- [ ] § 0.5 给出仓库顶级目录与文件**完整**速查表（不漏 setup.py / uv.lock / flake.lock / LICENSE / CLA.md / README.md / CLAUDE.md / AGENTS.md）
- [ ] § 0 明示本文档与 `AGENTS.md` / `CLAUDE.md` 的分工
- [ ] 命名重叠避坑至少覆盖 8 组：core/stream vs dimos/stream、agent-send vs agent_send、agents vs agents_deprecated、两套 skills、skills/manipulation vs manipulation、speak 三处、spec vs protocol/rpc/spec、memory vs perception/experimental/temporal_memory
- [ ] 全文中文，无中英文混排错误
- [ ] Mermaid 图至少 8 张：顶层全景、Stream/Transport/Protocol 三层、Module-Blueprint-Skill 关系、进程模型、Agent 类图、平台拓扑、子系统拓扑、E2E 时序
- [ ] 4 份专题文档（runtime / agent / robot / data-flow）每份 ≥ 2000 字
- [ ] `subsystems.md` ≥ 5200 字（每核心子系统 ≥ 400 字内部架构+依赖+扩展点；辅助子系统 ≥ 250 字）；与 README § 6 互补不重复
- [ ] README 中每个跳转链接都可达

**反向条目（不应出现）**：

- [ ] § 6 内不出现"详见 X 文档"占位句
- [ ] 不重复 `AGENTS.md` 的 quick-start 命令、blueprint quick-reference 表
- [ ] 不重复 `docs/usage/configuration.md`、`docs/usage/blueprints.md` 的使用样例
- [ ] `subsystems.md` 不复制 README § 6 的"做什么"叙述

## 9. Open Questions

无，待用户 review。
