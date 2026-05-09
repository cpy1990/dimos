# DimOS 架构文档对齐 main v2 — 设计规格

- **日期**：2026-05-09（v2，v1 已撤销）
- **状态**：Draft，待 review
- **前置**：分支 `docs/architecture` 已 rebase 到 `origin/main`，工作树 clean，0 commits behind main
- **方法论**：每条具体主张都必须有 `git ls-tree origin/main` / `git grep origin/main` 的 evidence，**不引用任何外部 audit / reviewer 结论**
- **目标产出位置**：5 份 `docs/architecture/*.md` 的多批次更新

## 0. v1 为什么撤销

v1 有 4 类系统性缺陷：

1. **4 个一等公民子系统零覆盖**：`dimos/hardware/whole_body/`、`dimos/navigation/patrolling/`、`dimos/simulation/unity/`、`dimos/robot/catalog/+config.py+model_parser.py`
2. **fan-out 不彻底**：README §3 散文（`deploy_parallel`/`ModuleProxy` 旧模型）、README §4 Agent 类表（line 414 仍列 `dimos/agents/agent.py`）、README §6 开头"13 个一级子系统"计数（line 679）、README §6 流程图 `MEM[memory/]`（line 694）、README §1280 锚点表 `agents.agent`、agent-stack:568 旧指南——批 C 只动 agent-stack 段，漏了 README 这些
3. **批 A 错位**：`control/blueprints.py → blueprints/` 是代码块里的树状结构改写，不是字串级替换；`SpatialPerception` 在 data-flow.md 早已是 `SpatialMemory`（no-op）；`teleop/blueprints.py` 旧路径其实 docs 已对；`moondream_hosted.py` 已在 README/subsystems 里
4. **4 处事实错误**：`ModuleCoordinator.load_blueprint/load_module/unload_module` API 漏了；`RunEntry.rpyc_port` 漏了；`types/manipulation.py`+`robot_capabilities.py`（OpenArm 随行）漏了；`skills/manipulation/` 6 个新 skill 漏了；`robot/drone/` 整个子系统从未进入文档

v2 换方法论：**先做 main-side inventory，再倒推每个文档位置的 patch**。

## 0.5 v2 self-review 后修正点（2026-05-09 当日 self-review 发现）

对 v2 自身的校正，evidence 已全部 git 核实。在后续章节原地修改，此处汇总：

1. **`DDSTransport` 不在 `core/transport.py`**：实际 main 只有 8 concrete Transport（pLCM/LCM/JpegLcm/pSHM/SHM/JpegShm/ROS/Zenoh），`DDSTransport` 在 `protocol/` 栈而非 `Transport` 子类——§1.20 已改
2. **manipulation/planning/monitor/ 非改名关系**：实际 3 文件 `world_monitor.py` + `world_obstacle_monitor.py` + `robot_state_monitor.py` 并存——§1.11 已改
3. **README §4 Agent 类表必须保留 WebInput / ollama_agent 行 + 命名重叠 3 避坑段**：`ollama_agent.py` 仍是"仅两函数不是类"（evidence：`grep "^(class|def)" origin/main -- ollama_agent.py` 只返回 2 个 def）；`agents_deprecated/` 仍在——批 C 已改为"保留避坑注，只替 Agent/VLMAgent 行"
4. **E.1 whole_body 位置歧义（双写隐患）**：锁定 robot-platforms.md §5.5，subsystems.md 不写——已改
5. **E.3 unity 位置歧义**：锁定 robot-platforms.md §6.6，subsystems.md 不写；§6.1 对比表由 4 后端改为 5 后端——已改
6. **F.4 drone 位置错记**：drone 在 robot-platforms.md **§3**（line 170 已存在占位），非"新增 §1.3"——已改
7. **`dimos/__init__.py` evidence 补录**：`__init__.py` 只 export `Dimos`（`__getattr__` lazy import），§1.3 已标明
8. **teleop/keyboard/ 已有文档**：README:1125/1137 已讲 `keyboard_teleop_module.py`（含"通用 vs unitree 专用"避坑）——批 A 不碰这段

## 0.6 Round 2 subagent 质疑 — 已 git 核实后整合

派 3 个 adversarial Explore subagent 独立读 v2 + `origin/main`，我对每一条结论再跑一遍 `git ls-tree` / `git grep` 核实（不盲采）。确认真实的 gap 与错误如下。**Subagent 的误报已剔除**（`GlobalConfig` 字段数 29、`WorkerManager` 行号等确是 spec 对）。

### A. 系统性路径错误（v1 residue，必须整改）

这组错误我自己的 spec §1.6 + 批 F.1/F.2 反复出现：

| 错路径 | 正确路径 |
|---|---|
| `go2/connection.py` | `robot/unitree/go2/connection.py` |
| `go2/cli/` | `robot/unitree/go2/cli/` |
| `go2/connection_spec.py` | `robot/unitree/go2/connection_spec.py` |
| `go2/blueprints/unitree_go2_webrtc_rage_keyboard_teleop.py` | `robot/unitree/go2/blueprints/basic/unitree_go2_webrtc_rage_keyboard_teleop.py` |
| `go2/blueprints/agentic/_huggingface.py` | `robot/unitree/go2/blueprints/agentic/unitree_go2_agentic_huggingface.py` |
| `go2/blueprints/agentic/_ollama.py` | `robot/unitree/go2/blueprints/agentic/unitree_go2_agentic_ollama.py` |
| `g1/connection.py` / `g1/wholebody_connection.py` | `robot/unitree/g1/...`（全部同理） |
| "新蓝图 `unitree_g1_coordinator.py`"（无路径） | `robot/unitree/g1/blueprints/basic/unitree_g1_coordinator.py` |

**处理**：§1.6 inventory 内就地修正（本 0.6 节下方），批 F.1/F.2 文档段落动笔前按正确路径落笔。

### B. 3 条 README fan-out gap（v2 漏）

1. **README.md:172 代码块** `from dimos.core.blueprints import autoconnect` — dead path。批 A 漏（只列 README §3 + agent-stack §4）→ 加入批 A 清单
2. **README.md:645 simulation 表** `| manipulators/ | 仿真内机械臂接口（sim_module.py、sim_manip_interface.py） |` — 目录已删。批 F.7 只动 robot-platforms.md → 加入批 G
3. **subsystems.md:206,226 §4 manipulation mermaid + 散文** `MON[WorldMonitor<br/>planning/monitor/]` + "`WorldMonitor`…监控实时世界状态" — 实际 3 文件 3 类（`world_monitor.py` + `world_obstacle_monitor.py` + `robot_state_monitor.py`）。v2 §1.11 注意到但无批次动 subsystems §4 mermaid → 加入批 G

### C. 3 个漏的代码表面（非子系统级但需点名）

4. **`navigation/visual/query.py`**: `get_object_bbox_from_image(vl_model, image, object_description)`（:21）— VLM→bbox 桥接，被 `skills/visual_navigation_skills.py` 使用 → 加入批 G（subsystems §3 navigation 补一句）
5. **`manipulation/control/servo_control/`**: `cartesian_motion_controller.py:87` `CartesianMotionController(Module)` + `:53` `CartesianMotionControllerConfig` — 与 `trajectory_controller/` 并列的伺服控制子包，子系统 §4 只把 `servo_control/` 作为树节点未展开 → 加入批 G
6. **`manipulation/planning/spec/models.py`** 是新文件（非 `types.py → models.py` 改名）：实际 `planning/spec/` 含 `config.py` / `enums.py` / `models.py` / `protocols.py` 四文件（不是单个 `types.py`）——v2 §1.11 已标"未独立核实"；现核实：非改名，是结构化 → §1.11 修正

### D. perception extra 跨子系统破坏面远大于 spec 标注

v2 §1.10 说"`perception` 从基础安装移出为可选 extra"，只让批 A 加一行标签。实测 `git grep -l "^from dimos\.perception" origin/main -- dimos/` 返回 **62 个文件**（非 perception 本身），分布：`agents/skills/navigation.py` / `manipulation/`（多处）/ `models/vl/*` / `navigation/visual_servoing/` / `memory2/vis/utils.py` / `experimental/security_demo/` 等。**这意味着若用户不装 extra 就会大面积 ImportError**。

**处理**：批 G 新增一节「perception 为可选 extra 的跨子系统影响」，在 subsystems §2 或 README §6 perception 格子里落一张"硬依赖 / 软依赖"清单（列出 top-level 受影响子系统：agents.skills / manipulation / models.vl / navigation.visual_servoing / memory2.vis / experimental.security_demo）。**不要**试图列出 62 个文件名（既不必要也不稳定），只列受影响的子系统。

### E. 微调（under-specification）

- §1.1 `coordination/ 17 文件` → 改"17 .py 文件（含 6 个 test_\*.py，11 个生产文件）"
- §批 B 的 runtime-model 批次未加 "protocol/rpc/pubsubrpc.py 仍存在但 de-link" 的明确交代，easy 遗漏 → 加到批 B §1.4 末尾一句

### F. `dimos/agents/skills/` 多文件（判定：非 gap，但可能值得一句）

`agents/skills/` 有 16 文件（`demo_calculator_skill.py` / `demo_google_maps_skill.py` / `demo_gps_nav.py` / `demo_robot.py` / `demo_skill.py` / `google_maps_skill_container.py` / `gps_nav_skill.py` / `navigation.py` / `osm.py` / `person_follow.py` / `speak_skill.py` / `speak_skill_spec.py` + 4 test）。agent-stack §3 的 skills 章节覆盖双 skills 体系（`dimos/skills/` + `dimos/agents/skills/`）——这就是 skills 子目录。现有文档已有 "命名重叠 4：两套 skills"（README §4），不算 gap。不加批次。

## 1. Main-side inventory（已 git 验证）

以下每一条都跑过 `git ls-tree origin/main` / `git grep origin/main` 确认。

### 1.1 核心协调层 `dimos/core/` + `dimos/core/coordination/`

`coordination/` 子目录（17 文件）：
- `module_coordinator.py` — 持 `dict[str, WorkerManager]`（按 `deployment_identifier` 索引：`"python"` / `"docker"`）；动态 API：`load_blueprint`（:279）/ `load_module`（:341）/ `unload_module`（:348）
- `worker_manager.py` — `WorkerManager` Protocol（:29，`deployment_identifier: str`）
- `worker_manager_python.py` — `WorkerManagerPython`（:34 `deployment_identifier = "python"`，forkserver + `n_workers`）
- `worker_manager_docker.py` — `WorkerManagerDocker`（:33 `deployment_identifier = "docker"`）
- `python_worker.py` — `PythonWorker`（原 `core/worker.py`）
- `worker_messages.py` — `WorkerRequest` union + `WorkerResponse`（PR #1767）
- `rpyc_server.py` — `RpycServer` + `ThreadedServer` 服务端
- `rpyc_services.py` — rpyc 服务定义
- `watchdog_main.py` — 父进程死后清理 sidecar
- `process_lifecycle.py` — `spawn_watchdog` 封装
- `blueprints.py` — `Blueprint` frozen dataclass + `autoconnect()` + `BlueprintAtom`

`core/` 顶层（21 文件，节选非测试）：
- `rpc_client.py` — `RPCClient`（:97）+ `ModuleProxyProtocol`（:88），rpyc **客户端侧**
- `docker_module.py` — `DockerModuleProxy`（与 WorkerManagerDocker 配套）
- `native_module.py` — `NativeModule`（C++/Rust 子进程委托，PR #1794）
- `run_registry.py` — `RunEntry`（`grpc_port: int = 9877` legacy @ :47；`rpyc_port: int = 0` @ :48；`get_most_recent_rpyc_port` @ :194）
- `global_config.py` — `GlobalConfig` pydantic-settings，29 字段，`env_file=".env"`，无 `env_prefix`
- `library_config.py` — 仅含 `apply_library_config()`，内部 `cv2.setNumThreads(2)`，**不是**配置系统（v1 描述过度）
- `daemon.py` — `daemonize()` 双 fork + `install_signal_handlers()`（`health_check` 已移到 coordinator）
- `log_viewer.py` — CLI 日志查看器
- `core.py` / `module.py` / `stream.py` / `transport.py` — 核心抽象
- `o3dpickle.py` — Open3D 跨进程序列化
- `introspection/`、`resource_monitor/`、`resource.py` — 资源管控
- `run_registry.py`、6 个 `test_async_module_*.py`（PR #1920 async modules）

### 1.2 Agents `dimos/agents/`

- `agent.py` **已删除**（原 `Agent` 类消失）
- `agent_spec.py:22` — `AgentSpec(Spec, Protocol)` 契约（`add_message` / `dispatch_continuation`）
- `vlm_agent_spec.py` — VLM Spec 孪生体
- `demo_agent.py` — `demo_agent` + `demo_agent_camera` 两个纯 MCP 蓝图
- `vlm_agent.py` / `ollama_agent.py` — 具体实现者
- `mcp/tool_stream.py` — `ToolStream` + `TOOL_STREAM_TOPIC = "/tool_streams"`（PR #1713）
- `web_human_input.py` / `system_prompt.py` / `utils.py` / `testing.py`

### 1.3 Porcelain `dimos/porcelain/`（PR #1779，纯新增）

- `dimos.py` — `Dimos` 类；入口方法 `.run()` / `.connect()` / `.skills` / `.peek_stream()` / `.restart()` / `.stop()` / `.__getattr__()`
- `module_source.py` / `local_module_source.py` / `remote_module_source.py` — `ModuleSource` Protocol 双实现（本地也走 rpyc）
- `skills_proxy.py` — `SkillsProxy` lazy cache，复用 `SkillInfo`
- `run_registry.py` 驱动 `Dimos.connect()` 自动发现
- **`dimos/__init__.py`** — 只暴露 `Dimos`（`__getattr__` lazy import `from dimos.porcelain.dimos import Dimos`）；无其他 public symbol

### 1.4 Memory2 `dimos/memory2/`（PRs #1682/#1769/#2004，纯新增）

- `module.py` — `StreamModule`（:69 `Module, Generic[TIn, TOut]`）、`MemoryModule`（:170）、`SemanticSearch`（:196）、`Recorder`（:247）同住一文件
- `stream.py` — `Stream[T]` lazy pull（`.after()` / `.near()` / `.search()` / `.live()` / `.save()` / `.drain()` / `.observable()`）
- `transform.py` — `Transformer[T,R]` `Iterator→Iterator` 基类 + `FnTransformer` / `FnIterTransformer` / `Batch` / `QualityWindow` / `downsample` / `throttle` / `peaks` / `significant` / `smooth` / `normalize`
- `embed.py:40` — `EmbedImages(Transformer[Any, Any])`；`:61` — `EmbedText(Transformer[Any, Any])`
- `store/` — `Store` ABC + `MemoryStore` / `SqliteStore`（WAL，FTS5，sqlite-vec 即 vec0）/ `NullStore`
- `vectorstore/` — `VectorStore` ABC + `memory.py` 暴力实现 + `sqlite.py:70` `USING vec0(...)` ANN 实现
- `observationstore/` — `ObservationStore` Protocol + `ListObservationStore` / `SqliteObservationStore`
- `blobstore/` — 大 payload 文件/SQLite 两后端
- `codecs/` — `JpegCodec`（turbojpeg 10–20x）/ `LcmCodec` / `LZ4Codec` / `PickleCodec`，`codec_for()` 自动选
- `notifier/` — `Notifier` ABC + `SubjectNotifier` RxPY
- `vis/` — `color.py` + `plot/`（`rerun.py` / `svg.py` / `elements.py` / `plot.py` 时序图）+ `space/`（`rerun.py` / `svg.py` / `space.py` 3D 空间）+ `utils.py`
- `backend.py` — `Backend` 组合 `ObservationStore + BlobStore + VectorStore + Notifier`
- `registry.py` — `RegistryStore` 跨 run 流 registry
- `type/` — `Observation[T]` / `EmbeddedObservation[T]` / `Filter` / `StreamQuery`

### 1.5 Memory `dimos/memory/`（仅 timeseries 有价值）

- `timeseries/` — `TimeSeriesStore` ABC + 5 后端（`InMemory` / `Sqlite` / `PickleDir` / `Postgres` / `LegacyPickle`）；被 `tf.py`、`Timestamped`、测试固件使用
- `embedding.py` — `EmbeddingMemory`（未完成原型，零非测试调用，`_store_spatial_entry` no-op），**已被 `memory2.SemanticSearch` 替代**

### 1.6 Robot 平台 `dimos/robot/`

顶层新增：
- `config.py:50` — `RobotConfig(BaseModel)` 统一配置（URDF/MJCF 单一真相源，生成 `RobotModelConfig` / `HardwareComponent` / `TaskConfig`）；`:32` `GripperConfig`
- `model_parser.py:29/:43` — `JointDescription` + `ModelDescription` + `parse_model()` 函数
- `catalog/` — `openarm.py` / `panda.py` / `piper.py` / `ufactory.py` 四个机器人模型定义
- `all_blueprints.py`（更新，新增 OpenArm 蓝图）

Unitree Go2 新增（全在 `dimos/robot/unitree/go2/`）：
- `cli/` 新子目录（PR #1990）
- `connection.py` 新枚举值 `Go2Mode.RAGE`（枚举值行 :61；`class Go2Mode` 声明行 :59）
- `connection_spec.py` 新 Spec Protocol
- `blueprints/basic/unitree_go2_webrtc_rage_keyboard_teleop.py` 蓝图（PR #1903，**在 basic/ 下**不在 blueprints/ 顶层）
- `blueprints/agentic/` 下 5 个硬件绑定 MCP 蓝图：`unitree_go2_agentic.py` + `unitree_go2_agentic_huggingface.py` + `unitree_go2_agentic_ollama.py` + `unitree_go2_security.py` + `unitree_go2_temporal_memory.py`（另有 `_common_agentic.py` 共享辅助）

Unitree G1 新增（全在 `dimos/robot/unitree/g1/`）：
- `connection.py`（主 entry，**仍存**）
- `wholebody_connection.py` 新增（追加低层全身控制通道，PR #1954；**不取代** connection.py）
- `connection_spec.py` 新 Spec Protocol
- `sim.py` / `skill_container.py` / `system_prompt.py`
- 新蓝图 `robot/unitree/g1/blueprints/basic/unitree_g1_coordinator.py`（在 basic/ 下）
- G1 blueprints 子目录结构：`agentic/` / `basic/` / `perceptive/` / `primitive/`

Manipulators 新增：
- `robot/manipulators/openarm/`（PR #1897 OpenArm Integration）——`robot/manipulators/` 3 家平台：piper / xarm / openarm（**无 mock**，mock 只在 hardware 层）

Drone（文档现状：**从未进架构文档**，但确实是主干子系统）：
- `robot/drone/`——`connection_module.py` / `mavlink_connection.py`（MAVLink 协议驱动）/ `drone_tracking_module.py` / `drone_visual_servoing_controller.py` / `dji_video_stream.py` / `camera_module.py` + `blueprints/` + `README.md`

### 1.7 Hardware `dimos/hardware/`

- `drive_trains/unitree_go2/adapter.py` 新增（Go2 硬件抽象）
- `drive_trains/transport/adapter.py` 新增
- `drive_trains/mock/` / `flowbase/` 保持
- `manipulators/` 5 个实现：`mock/` / `piper/` / `xarm/` / `openarm/`（新）/ `sim/`（新，`adapter.py` + `test_shm_adapter.py`，shm 与 MuJoCo 同步）
- **`whole_body/` 新子包（纯新增，v1 零覆盖）**：
  - `spec.py:28` `MotorCommand`（dataclass q/dq/kp/kd/tau）、`:39` `MotorState`、`:48` `IMUState`、`:58` `WholeBodyAdapter(Protocol)`
  - `registry.py` 适配器 registry
  - `transport/adapter.py` 传输级适配器
- `sensors/camera/realsense/__init__.py` 已删除（包路径改导入方式，非目录消失）

### 1.8 Simulation `dimos/simulation/`

- `engines/genesis.py` / `isaac.py` / `mujoco.py` 原有
- `engines/mujoco_shm.py` 新（shm backend）
- `engines/mujoco_sim_module.py` 新
- `engines/registry.py` 新
- `mujoco/depth_camera.py` 新（`MujocoCamera`，PR #1694）
- **`unity/` 新子包（纯新增，v1 零覆盖）**：
  - `module.py:222` `UnityBridgeModule`（TCP 桥连 VLA Challenge Unity sim，ROS-TCP-Endpoint 二进制协议，**无** ROS 依赖）
  - `module.py:152` `UnityBridgeConfig`
  - `blueprint.py` 蓝图
- `simulation/manipulators/` 已删除（移到 `hardware/manipulators/sim/`）
- 新增 `--simulation` flag（PR #2027）；MuJoCo xarm+piper 遥操作（PR #1958）

### 1.9 Navigation `dimos/navigation/`

- `navigation_spec.py` 新（Protocol）
- `replanning_a_star/module_spec.py` 新
- `visual/query.py:21` `get_object_bbox_from_image(vl_model, image, object_description)` — VLM→bbox 桥接函数，被 `skills/visual_navigation_skills.py` 使用；文档里只有 "navigation/visual/" 树节点，无类/函数点名
- **`patrolling/` 新子系统（纯新增，v1 零覆盖）**：
  - `module.py:37` — `PatrollingModule(Module)`（async Module，监听 `odom` / `global_costmap` / `goal_reached` streams）
  - `patrolling_module_spec.py:24` — `PatrollingModuleSpec(Spec, Protocol)`
  - `create_patrol_router.py` — factory
  - `routers/` 6 文件：`base_patrol_router.py:27` `BasePatrolRouter(ABC)`、`patrol_router.py:21` `PatrolRouter(Protocol)`、`coverage_patrol_router.py` `CoveragePatrolRouter`、`frontier_patrol_router.py` `FrontierPatrolRouter`、`random_patrol_router.py` `RandomPatrolRouter`、`visitation_history.py:26` `VisitationHistory`
  - `utilities.py` / `constants.py` / `test_create_patrol_router.py`
  - e2e 测试：`e2e_tests/test_patrol_and_follow.py` 证明生产级

### 1.10 Perception `dimos/perception/`

- 从基础安装移出为 **可选 extra**（PR #1888）——架构文档里所有"总是可用"假设需要调整
- `spatial_perception.py:71` — 类名改 `SpatialMemory`（从 `SpatialPerception`）
- `spatial_memory_spec.py` 新 Spec Protocol
- `object_scene_registration_spec.py` 新
- `object_tracking_spec.py` 新
- `dimos/spec/perception.py` **仍存在**（"drop perception" PR 只删了 pip 依赖，没删 Spec 文件）

### 1.11 Mapping / Manipulation

Mapping：
- `mapping/types.py` → `mapping/models.py` **已改名**
- `mapping/google_maps/types.py` → `models.py` **已改名**
- `VoxelGridMapper`（`voxels.py:253` `StreamModule[PointCloud2, PointCloud2]`）—— 用 memory2 的 `StreamModule` 基类 + `VoxelMapTransformer`

Manipulation：
- `control/arm_driver_spec.py` 新 Protocol
- `control/servo_control/cartesian_motion_controller.py:87` `CartesianMotionController(Module)` + `:53` `CartesianMotionControllerConfig`（伺服/Cartesian 控制路径，与 `trajectory_controller/` 关节空间路径并列）
- `grasping/grasp_gen_spec.py` 新
- `planning/monitor/` 3 文件 **并存**（非改名）：`world_monitor.py` + `world_obstacle_monitor.py` + `robot_state_monitor.py`
- `planning/spec/` 结构化拆分为 4 文件：`config.py` / `enums.py` / `models.py` / `protocols.py`（**非** `types.py → models.py` 单改名）

### 1.12 Control `dimos/control/`

- `blueprints.py`（单文件）→ `blueprints/` 目录（4 文件：`basic.py` / `dual.py` / `mobile.py` / `teleop.py`）——**文档树里需要改结构，不是字串替换**
- `components.py` / `coordinator.py` / `hardware_interface.py` 变更（随 G1 wholebody PR #1954 做了改动）

### 1.13 Skills `dimos/skills/`

- `skills.py`（`SkillLibrary` + `AbstractSkill`）
- `kill_skill.py` / `speak.py` / `visual_navigation_skills.py`
- **`manipulation/` 新子包（纯新增，v1 零覆盖）**：`abstract_manipulation_skill.py` + `force_constraint_skill.py` + `manipulate_skill.py` + `pick_and_place.py` + `rotation_constraint_skill.py` + `translation_constraint_skill.py`
- `unitree/unitree_speak.py` / `rest/rest.py` 保持

### 1.14 Types `dimos/types/`

原有 `timestamped.py` / `vector.py` / `weaklist.py` / `robot_location.py` 保持
新增：
- `manipulation.py`（OpenArm 随行）——8 类：`ConstraintType` / `AbstractConstraint` / `TranslationConstraint` / `RotationConstraint` / `ForceConstraint` / `ObjectData` / `ManipulationMetadata` / `ManipulationTask`
- `robot_capabilities.py` — `RobotCapability` enum
- `ros_polyfill.py` / `sample.py` / `constants.py`

### 1.15 Msgs `dimos/msgs/`

- `sensor_msgs/` 新增：`JointCommand.py` / `MotorCommandArray.py` / `RobotState.py` / `image_impls/`（子目录）
- 其他 `sensor_msgs/` 原有：`CameraInfo.py` / `Image.py` / `Imu.py` / `JointState.py` / `Joy.py` / `PointCloud2.py`
- `trajectory_msgs/`：`JointTrajectory.py` / `TrajectoryPoint.py` / `TrajectoryStatus.py`

### 1.16 Models `dimos/models/`

- `vl/video_query.py` **不存在**——实际在 `models/qwen/video_query.py`
- `vl/` 子树：`base.py` / `create.py` / `florence.py` / `moondream.py` / `moondream_hosted.py`（已存在，非新增）/ `openai.py` / `qwen.py` / `types.py` / `README.md`
- `qwen/` 子树：`video_query.py` / `bbox.py`
- `embedding/` / `segmentation/` 保持

### 1.17 Visualization `dimos/visualization/rerun/`

- 类名改为 `RerunBridgeModule`（从 `RerunBridge`）
- Protocol 改为 `RerunConvertible`（从 `HasToRerun`）
- `init.py` 新增

### 1.18 Experimental `dimos/experimental/`

- `security_demo/` 新（`depth_estimator.py` + `security_module.py` + `conftest.py` + `test_security_module.py`）
- `temporal_memory/` 保持

### 1.19 Stream `dimos/stream/`（顶层包，非 `core/stream.py`）

- `audio/` / `data_provider.py` / `frame_processor.py` / `video_provider.py` / `rtsp_video_provider.py` / `ros_video_provider.py` / `stream_merger.py` / `video_operators.py`
- **与 `core/stream.py` 不是一回事**（`core/stream.py` 是 Module 间数据通道，`stream/` 是应用层媒体流工具）

### 1.20 Protocol 与 Transport

- `core/transport.py` 共 **8 个 concrete** Transport 类：`pLCMTransport`（:79）/ `LCMTransport`（:111）/ `JpegLcmTransport`（:142）/ `pSHMTransport`（:163）/ `SHMTransport`（:193）/ `JpegShmTransport`（:223）/ `ROSTransport`（:258，`DimosMsg` 泛型）/ `ZenohTransport`（:330，stub `...`）加上 `PubSubTransport` 基类（:65）共 **9 个**。现有文档"9 transports"成立。**注**：`DDSTransport` **不在** `core/transport.py`——DDS 只出现在 `protocol/` 栈（`protocol/pubsub/impl/` 下），不是 `Transport` 子类
- `protocol/rpc/pubsubrpc.py` 仍存在但已从 coordination 路径 de-link（PR #9d7806615）；rpyc 成为主 IPC
- `protocol/pubsub/impl/jpeg_lcm.py` 新增
- `protocol/pubsub/bridge.py` 已删
- 多个 `__init__.py` 删除（全仓 `__init__` 删除运动）

### 1.21 CLI / Daemon

CLI entrypoint `dimos.robot.cli.dimos:main` 暴露：
- `run` / `status` / `stop` / `restart` / `log` / `list` / `show-config` / `agent-send` / `rerun-bridge` / `lcmspy` / `agentspy` / `humancli` / `top`
- `mcp` 子 app：`list-tools` / `call` / `status` / `modules`
- `topic` 子 app：`echo` / `send`
- `go2tool` 子 app（通过 `main.add_typer`）

### 1.22 其他（纳入但不展开）

- `dimos/e2e_tests/` — 新增 `test_patrol_and_follow.py` / `test_security_module.py` / `test_scan_and_follow_person.py` / `test_spatial_memory.py` / `test_person_follow.py`；属 test churn，不动文档
- `dimos/project/` — 3 个 convention 测试文件（`test_get_logger.py` / `test_no_init_files.py` / `test_no_sections.py`），纯测试目录，不动文档
- `dimos/utils/` — 新 `prompt.py` / `ros1.py` / `safe_thread_map.py` / `workspace.py`（314 行，OpenArm 随行）/ `ament_prefix.py`；`utils/testing/` 结构化测试辅助；按原约定不展开，只在相关位置提 `utils/workspace.py`
- `dimos/agents_deprecated/` — 保持"严禁新代码使用"，不动文档
- `dimos/web/` — `websocket_vis_spec.py` 新 Spec；其他不变
- `dimos/exceptions/` — 仅 `__init__.py` 删除，无新类

## 2. 目标与非目标

### 目标

1. 5 份 `docs/architecture/*.md` 所有文件路径、类名、API、环境变量名对 `origin/main` 100% 正确——每条都能用 `git ls-tree` / `git grep` 验证
2. 已删代码不留痕：`dimos/core/worker.py` / `dimos/agents/agent.py` / `dimos/control/blueprints.py`（单文件）/ `dimos/mapping/types.py` / `SpatialPerception` / `RerunBridge`（不带 Module）/ `HasToRerun` / `Actor` / `MethodCallProxy` / `unitree-go2-agentic-mcp is唯一` 全部从文档清除或重写
3. 主干新系统 100% 覆盖：§1 inventory 里每个"新"或"纯新增"的子系统都在文档中有实质段落（不是一句带过）——包括 v1 漏的 4 个（whole_body / patrolling / unity / robot-catalog+config）+ drone（从未进文档）
4. fan-out 全做：§1 inventory 每条改动都要反查 5 份文档所有相关位置（而不是只动直接相关的那一节）
5. 不影响仍正确的章节

### 非目标

- 不翻译（全中文保持）
- 不改 `docs/usage/*` / `docs/capabilities/*` / `docs/platforms/*`（架构文档只指向，不重复）
- 不写 CI / 工作流 / 依赖更新
- 不覆盖 2026-05-06 原 spec / 原 plan（历史归档）
- 不展开 `dimos/utils/` 除非交叉依赖（`workspace.py` 单独提一句）
- 不展开 `dimos/agents_deprecated/`（维持"严禁新代码使用"）
- 不展开 `dimos/e2e_tests/` / `dimos/project/`（纯测试）

## 3. 批次规划

改动分 **7 批**（比 v1 多 2 批，因新主干子系统 + fan-out 工作量翻倍）。每批一个 commit。

| # | 主题 | 主要文件 | 估计行变 |
|---|---|---|---|
| A | 文件路径 + 类名 + 环境变量散点修正 | 5 份文档 | +60 / −40 |
| B | runtime-model §1 重写 + §6.2 RunEntry 表 + 动态 blueprint API | runtime-model.md、README §3 散文 | +220 / −80 |
| C | agent-stack §1 / §5 / §6.1 重写 + README §4 类表同步 + README §1280 锚点表 | agent-stack.md、README.md | +170 / −100 |
| D | memory2 主体重写（subsystems §6）+ README §4.5 Porcelain + README §6 流程图 + data-flow 补丁 | README.md、subsystems.md、data-flow.md | +280 / −100 |
| E | 4 个新主干子系统：whole_body、patrolling、unity、robot-catalog+config | README.md、subsystems.md、robot-platforms.md | +220 / −20 |
| F | 机器人平台尾收：Go2 rage/CLI、G1 wholebody、OpenArm、MujocoCamera、sim adapter、drone 首次进文档 | robot-platforms.md | +260 / −50 |
| G | 次级子系统补全：types/manipulation + robot_capabilities、skills/manipulation、stream/、experimental/security_demo、visualization rename、msgs 新增、models/vl 与 qwen 路径、mapping/models 改名 | subsystems.md、README.md | +140 / −40 |

**push 策略**：7 个 commit 全部本地完成 → 一次 push（一次 CI ~1h）。

### 批 A — 散点修正（字串级别，不动结构）

纯字串替换或简单行补。**不改代码块的树状结构**（那些放批 D/E/F/G）。清单（每行已经 git 核过）：

| 项 | 旧 | 新 | 位置 |
|---|---|---|---|
| core 包重组 | `dimos/core/module_coordinator.py` | `dimos/core/coordination/module_coordinator.py` | README §3、runtime-model §1 |
| 同上 | `dimos/core/worker_manager.py` | `dimos/core/coordination/worker_manager.py` | README §3、runtime-model §1 |
| 同上 | `dimos/core/worker.py` | `dimos/core/coordination/python_worker.py` | README §3、runtime-model §1（日志例） |
| 同上 | `dimos/core/blueprints.py` | `dimos/core/coordination/blueprints.py` | README §3、agent-stack §4 |
| logger 名 | `dimos.core.module_coordinator` | `dimos.core.coordination.module_coordinator` | runtime-model JSONL 例 |
| perception 类 | `SpatialPerception` | `SpatialMemory` | subsystems.md:89,108（仅这 2 处，data-flow 已是 SpatialMemory 跳过） |
| rerun 类（**替换所有出现**） | `RerunBridge`（不带 Module） | `RerunBridgeModule` | subsystems:490,494,498,514,590,592 + README:1100,1105 |
| rerun protocol | `HasToRerun` | `RerunConvertible` | subsystems:500,592 |
| VL 路径 | `vl/video_query.py` | `models/qwen/video_query.py` | subsystems §7（正文行） |
| Mapping types | `mapping/types.py` | `mapping/models.py` | subsystems §5 |
| Mapping google_maps | `google_maps/types.py` | `google_maps/models.py` | subsystems §5 |
| Perception 安装 | 隐含"始终可用" | 明标"perception 为 pip extra（PR #1888），按需安装" | README §6、subsystems §2 |
| **README §1 代码块** | `from dimos.core.blueprints import autoconnect` | `from dimos.core.coordination.blueprints import autoconnect` | README.md:172（round 2 发现的 fan-out gap） |

**批 A 不包括**（这些是结构性改，批 D/E/F/G 处理）：
- `control/blueprints.py` 单文件 → `blueprints/` 目录树（批 G）
- Go2/G1 目录树扩展、hardware/manipulators 新平台、mujoco depth_camera、drive_trains 新条目（批 F）
- 所有新主干子系统（批 E）
- msgs 新类型、types/manipulation、skills/manipulation（批 G）
- MCP blueprint "唯一" 段落重写（批 C）

**验收**：对每行跑 `git ls-tree origin/main -- <新路径>` 或 `git grep origin/main <新类名>` 确认新值存在；对旧值关键词全文搜应只剩批 A 未覆盖的"预定残留"（例如批 C 里将要重写的 agent-stack §5 MCP 段落里的 `unitree-go2-agentic-mcp`）。

### 批 B — runtime-model 重写 + README §3 散文同步

**Scope**：runtime-model.md §1 全重写（60 行→~200 行）+ §6.2 `RunEntry` 表 + README §3 散文（line 301-310 `WorkerManager`/`ModuleCoordinator` 描述）同步。

**runtime-model §1 新结构**（9 小节）：
- **§1.1 协调层双轨**：`ModuleCoordinator`（`coordination/module_coordinator.py`）持 `dict[str, WorkerManager]`（按 `deployment_identifier` 索引，字面值 `"python"` 在 `worker_manager_python.py:34` / `"docker"` 在 `worker_manager_docker.py:33`）；`WorkerManager` Protocol（`worker_manager.py:29`）；双实现：`WorkerManagerPython`（forkserver + `n_workers`）/ `WorkerManagerDocker`（容器编排）
- **§1.2 Python 路径**：`PythonWorker`（`python_worker.py`）forkserver context 惰性初始化 + `worker_messages.py` 的 `WorkerRequest`/`WorkerResponse` 进程间消息协议（PR #1767）
- **§1.3 Docker 路径**：`WorkerManagerDocker` + `core/docker_module.py` 的 `DockerModuleProxy`（住 `core/` 顶层非 coordination/）
- **§1.4 rpyc IPC 双侧**：服务端 `coordination/rpyc_server.py`（`RpycServer` 持 `ThreadedServer`）+ `coordination/rpyc_services.py`；客户端 `core/rpc_client.py`（`RPCClient:97` + `ModuleProxyProtocol:88`）；取代旧 `Actor`+`Pipe`；本地/远程同管道（porcelain 也走这条）
- **§1.5 动态 blueprint 装卸**：`ModuleCoordinator.load_blueprint`（:279）/ `load_module`（:341）/ `unload_module`（:348）——支持启动后热装卸（PR #1744）
- **§1.6 watchdog 生命周期**：`watchdog_main.py` sidecar + `process_lifecycle.py` 的 `spawn_watchdog` + `core/run_registry.py` 跨 run 注册
- **§1.7 Native / Rust 模块**：`core/native_module.py` 的 `NativeModule`（CLI 传 LCM topic 名，C/C++/Rust subprocess 委托；rust native modules PR #1794）
- **§1.8 Async 模块**：PR #1920；patrol 示例（`navigation/patrolling/` + asyncio）；6 个 `test_async_module_*.py`；讲 async dispatch / handles / process observables / RPC / sync-to-async
- **§1.9 core/ 顶层其余**：`introspection/` / `resource_monitor/` / `resource.py`（资源管控）；`o3dpickle.py`（Open3D 跨进程序列化）；`library_config.py`（仅 `cv2.setNumThreads(2)` 启动副作用，**不是**配置系统）；`core.py` / `daemon.py`（`daemonize()` 双 fork）/ `log_viewer.py`（CLI 日志）

**runtime-model §6.2 RunEntry 表补**：
- 补 `rpyc_port: int = 0`（新字段，实际 daemon 用这个）
- 标 `grpc_port: int = 9877` 为 legacy（`check_port_conflicts` 还在用，运行流程不再用）
- 加 `get_most_recent_rpyc_port()` 查询函数（`run_registry.py:194`）

**README §3 同步**（line 301-310）：
- 散文里`deploy_parallel` + `ThreadPoolExecutor` + `ModuleProxy` 概念本节不删但需标注"本节展开在 runtime-model §1"，并把 rpyc 替换 Actor 的关系讲一句
- 路径 `dimos/core/worker_manager.py` → `dimos/core/coordination/worker_manager.py`（与批 A 联动；批 A 改字串，批 B 改周边散文不矛盾部分）

**不再出现**：`Actor`、`MethodCallProxy`、`dimos/core/worker.py`、旧 `core/blueprints.py` 路径、`dimos.core.module_coordinator` logger 名（改 `dimos.core.coordination.*`）

### 批 C — agent-stack + README 全面同步（Agent 体系重写）

**Scope**：agent-stack.md §1 / §5 / §6.1 全重写 + README §4 Agent 类表（line 414-445）+ README §1280 锚点表 + tool streams 新增。

**agent-stack §1 重写**（原 65 行讲已删除的 `agent.py` 内部 LangGraph 循环，按 main 重写）：
- `AgentSpec(Spec, Protocol)`（`agents/agent_spec.py:22`）契约：`add_message` + `dispatch_continuation`
- `VlmAgentSpec`（`vlm_agent_spec.py`）视觉 LLM 孪生
- `demo_agent.py`：`demo_agent = autoconnect(McpServer.blueprint(), McpClient.blueprint())` + `demo_agent_camera`——纯 MCP 蓝图，不绑硬件，可作模板
- 实现者：`vlm_agent.py` / `ollama_agent.py`
- Protocol 化之后没有"唯一的 Agent 类"——运行时是任何 `AgentSpec` 实现 + 任何 MCP tools 组合

**agent-stack §5 MCP 章**：
- 把"`unitree-go2-agentic-mcp` 是唯一支持 MCP 的 blueprint"论断**整段重写**为双类：
  - **硬件绑定**（5 个）：`agentic/unitree_go2_agentic.py` + `_huggingface.py` + `_ollama.py` + `unitree_go2_security.py` + `unitree_go2_temporal_memory.py`
  - **纯 MCP**（2 个）：`agents/demo_agent.py` 的 `demo_agent` + `demo_agent_camera`，不绑硬件
- 新增 tool streams 子节（PR #1713）：`dimos/agents/mcp/tool_stream.py` 的 `ToolStream` + `current_skill_context` 绑定 + `TOOL_STREAM_TOPIC = "/tool_streams"`；事件流：`@skill` 内 → `pLCMTransport` → `McpServer` 订阅 → `notifications/message` + `notifications/progress` JSON-RPC → MCP SSE

**agent-stack §6.1（line 499）**："`Agent`（`dimos/agents/agent.py`）——标准智能体"章节：删除整章，替换为"运行时组合"章节，指向 §1 的 AgentSpec 讨论

**agent-stack:568** 导读句："**所有新代码请使用 `dimos.agents.agent.Agent`…**" → 改为"**所有新代码请实现 `dimos.agents.agent_spec.AgentSpec` Protocol，或直接使用 `demo_agent` / `vlm_agent` 蓝图**"

**README §4 Agent 类表（line 414-445）整表重写**——**保留原有避坑注意事项，只替换 Agent/VLMAgent 行**：
- 删除 `Agent` @ `dimos/agents/agent.py` 行（该类已从 main 删除）
- 加 `AgentSpec(Protocol)` @ `dimos/agents/agent_spec.py` 行
- 加 `demo_agent` / `demo_agent_camera` @ `dimos/agents/demo_agent.py` 行
- `VLMAgent` 行 → `VlmAgent` + `VlmAgentSpec` 两行（路径均更新）
- **保留** `WebInput` 行（含"类名为 WebInput"避坑）
- **保留** `ollama_agent.py` 行（evidence：`grep "^(class|def)" origin/main -- dimos/agents/ollama_agent.py` 仅返回两个 `def ensure_ollama_model` + `def ollama_installed`；**确认仍是"不是类，只暴露两函数"**——避坑仍成立）
- **mermaid classDiagram 改写**：`Module <|-- Agent/VLMAgent/WebInput` → `AgentSpec <|.. demo_agent` + `AgentSpec <|.. VlmAgent` + `Module <|-- WebInput`
- **保留** "命名重叠 3：legacy 包 `dimos/agents_deprecated/`" 整段——该包仍在，避坑仍有效；但"新代码请使用 `dimos.agents.agent.Agent`"那条指引在其中改为"请实现 `dimos.agents.agent_spec.AgentSpec`"
- **保留** "命名重叠 4：两套 skills" 整段

**README §4:445 "新代码中严禁使用 `dimos/agents_deprecated/`" 段**：类名避坑（`grep "class Agent"` 命中新旧）保留；但提示里"`dimos.agents.agent.Agent`"改为"`dimos.agents.agent_spec.AgentSpec` / `demo_agent`"

**README §4.5 MCP 章**：同 agent-stack §5 的双类结构，简版（3-5 行 + 指向 agent-stack）

**README §1280 锚点表**（data-flow 小节交叉表）：`agents.agent` → `agents.agent_spec` / `agents.demo_agent`

**应消失关键词**（本批完后整文档全局 grep 应 0 命中）：
- `dimos/agents/agent.py`
- `class Agent\b`（作为独立段落标题——允许 `AgentSpec` 存在）
- `LangGraph` / `LangChain` / `create_agent`（agent-stack §1 原 LangGraph 实现描述）
- `unitree-go2-agentic-mcp` 作为"唯一"论断
- `on_system_modules` 作为"Agent 主入口"论断（是 Module 基础设施，不是 Agent 专属）

### 批 D — memory2 主体 + Porcelain + README §6 fan-out + data-flow

**Scope**：subsystems.md §6 memory/ 重写、README §4.5 Porcelain 新章、README §6 流程图与计数同步、data-flow.md 补丁。

**D.1 subsystems.md §6 memory/ 重写**（原 ~70 行 → ~200 行）：
- 主题 memory2，memory/ 做短注
- **memory2 核心抽象**：`Stream[T]`（`memory2/stream.py`：`.after()` / `.near()` / `.search()` / `.live()` / `.save()` / `.drain()` / `.observable()`）、`Transformer[T,R]`（`memory2/transform.py`：`Iterator→Iterator`；`FnTransformer` / `Batch` / `QualityWindow` / `downsample` / `throttle` / `peaks` / `significant` / `smooth` / `normalize`）、`StreamModule`（`memory2/module.py:69` `Module, Generic[TIn, TOut]`，用 `pipeline(stream)` 声明变换）
- **成品 module**（三者同住 `memory2/module.py`）：`MemoryModule:170` / `SemanticSearch:196` / `Recorder:247`；`EmbedImages`/`EmbedText`（`embed.py:40`/`:61`，均 `Transformer[Any, Any]`）作为 Transformer 嵌入 pipeline
- **存储层七件套**（每项 2-3 句）：`store/`（`Store` ABC + `SqliteStore` WAL/FTS5/sqlite-vec + `MemoryStore` + `NullStore`）、`observationstore/`（`ObservationStore` 协议 + List/Sqlite）、`vectorstore/`（`VectorStore` ABC + `memory.py` 暴力 + `sqlite.py:70` `USING vec0(...)` ANN）、`blobstore/`（大 payload 文件/SQLite 两后端）、`codecs/`（`JpegCodec` turbojpeg 10–20× / `LcmCodec` / `LZ4Codec` / `PickleCodec`；`codec_for()` 自动选）、`notifier/`（`Notifier` ABC + `SubjectNotifier` RxPY Subject）、`vis/`（`color.py` + `plot/`（时序图 Rerun/SVG）+ `space/`（3D 空间 Rerun/SVG）+ `utils.py`）、`backend.py`（`Backend` 组合 `ObservationStore + BlobStore + VectorStore + Notifier`）、`registry.py`（`RegistryStore` 跨 run 流 registry）
- **集成示例**：
  - `VoxelGridMapper`（`dimos/mapping/voxels.py:253`——**住在 mapping/，非 memory2 内**；继承 `StreamModule[PointCloud2, PointCloud2]`；用 `VoxelMapTransformer` 做 pipeline；体现"memory2 作为基础设施被其他子系统使用"）
  - `Recorder`（smart go2 blueprint）
  - `SqliteStore` 被 replay / dtop / demo 大量使用
  - `RegistryStore` SQLite 跨 run 流 registry
- **memory/ 短注**（~10 行）：
  - `timeseries/` 继续存在——通用键值时序（`InMemory` / `Sqlite` / `PickleDir` / `Postgres` / `LegacyPickle`），被 `tf.py` / `Timestamped` / 测试固件使用，**与 memory2 不重叠**
  - `embedding.py` 的 `EmbeddingMemory` 是未完成原型，零非测试调用，已被 memory2.SemanticSearch 替代，不展开

**D.2 README §4.5 Porcelain 新章**（~60 行，位置：插在 §4 Agent 体系 与 §5 平台层 之间）：
- 定位：Blueprint / ModuleCoordinator / rpyc 之上的 facade；`Dimos` 类是 `dimos/__init__.py` 唯一 public symbol
- 两模式：本地 `Dimos(**overrides)`（进程内建 coordinator）/ 远程 `Dimos.connect(run_id, host, port)`（rpyc 接已运行 daemon；无参通过 `run_registry` 自动发现）
- 核心 API：`.run(target)` / `.skills.<name>(...)` / `.peek_stream(name, timeout)` / `.restart(module, reload_source=True)` / `.stop()` / `.__getattr__(name)`（直通 rpyc 模块代理）
- `SkillsProxy` 复用 Agent 的 `SkillInfo`（同一 `get_skills()` 协议）；与 Agent 的区别：porcelain 面向外部脚本驱动，Agent 面向进程内 LLM dispatch
- 为什么本地也走 rpyc：local/remote 路径统一，避免特化
- 指向 `docs/usage/python-api.md`（用户教程），不重复
- 与 §2 通信栈关系：不重构三层栈；porcelain 位于"之上"作为 facade

**D.3 README §6 fan-out**：
- line 679："DimOS 现有 13 个一级子系统" → "**14 个一级子系统**"（stream/ 进入 §14）
- line 694 流程图 `MEM[memory/]` → `MEM2[memory2/（主）+ memory/timeseries（辅）]`
- §6 子系统目录表：memory 那一格改写（memory2 为主 + timeseries 短注 + embedding 废弃说明）
- §6 子系统目录表新增 stream/ 一格
- §6 子系统目录表 memory 那一格已包含 VoxelGridMapper 被 memory2 承接的说明，避免遗漏

**D.4 data-flow.md 补丁**：
- line 1280 锚点表：`agents.agent` → `agents.agent_spec` + `agents.demo_agent`（与批 C 同步）
- perception trace：类名已是 `SpatialMemory`（已是正确，不动）
- **补 memory2-涉入的 mapping trace**：`PointCloud2` stream → `VoxelGridMapper.pipeline()` → `VoxelMapTransformer` → `Stream[PointCloud2]` 出口
- **补 tool streams 事件流**：`@skill` → `ToolStream.send()` → `pLCMTransport` → `McpServer` → MCP SSE `notifications/{message,progress}`
- 清除残留行号（60+ commits 漂移后所有 `xxx.py:123-456` 不再可靠）——只写 class / function 名
- 允许保留那些在本 rebase 后通过 `git grep` 复核确认仍对齐的行号，其余都去掉行号

### 批 E — 4 个新主干子系统（v1 零覆盖）

**Scope**：`hardware/whole_body/`、`navigation/patrolling/`、`simulation/unity/`、`robot/catalog/+config.py+model_parser.py`。每个都在 subsystems.md 新增实质段落 + README §6 目录表加行 + 对应 robot-platforms 交叉引用（仅 whole_body、robot-config）。

**E.1 `hardware/whole_body/` → robot-platforms.md §5 新增 §5.5**
- 位置锁定：robot-platforms.md §5 hardware/ 下现有 §5.1 drive_trains / §5.2 end_effectors / §5.3 manipulators / §5.4 sensors——新增 §5.5 whole_body/（**仅此一处**，subsystems.md 不双写）
- 内容：
  - `spec.py:58` `WholeBodyAdapter(Protocol)`——提供 `send_command(cmd: MotorCommand)` / `get_state() -> (MotorState, IMUState)` 等
  - `spec.py:28,39,48` `MotorCommand`（dataclass q/dq/kp/kd/tau per motor）/ `MotorState` / `IMUState`
  - `registry.py` 适配器 registry
  - `transport/adapter.py` 传输级适配器
- 明确它与 G1 `wholebody_connection.py` 的分工：hardware 层定义 Protocol + 数据契约，robot 层 `g1/wholebody_connection.py` 实现 Protocol
- 与现有 `drive_trains/` / `manipulators/` 的对比：三者并列的 HAL 类别，**全身多关节控制**是新的 HAL 范畴

**E.2 `navigation/patrolling/` 写进 subsystems.md §3 navigation**
- 位置：subsystems.md §3 navigation 下新增子节「自主巡逻（patrolling）」
- 内容：
  - `module.py:37` `PatrollingModule(Module)`——async Module，监听 `odom` / `global_costmap` / `goal_reached` streams
  - `patrolling_module_spec.py:24` `PatrollingModuleSpec(Spec, Protocol)`
  - `create_patrol_router.py` factory
  - Router 体系（插拔式）：`routers/patrol_router.py:21` `PatrolRouter(Protocol)` + `routers/base_patrol_router.py:27` `BasePatrolRouter(ABC)` 提供 2 抽象：随机 / 覆盖 / 前沿三策略：
    - `CoveragePatrolRouter`（覆盖式，走过再回补）
    - `FrontierPatrolRouter`（前沿式，趋向未知边界）
    - `RandomPatrolRouter`（随机）
    - `VisitationHistory`（`routers/visitation_history.py:26`）作为共享访问记录
  - `utilities.py` / `constants.py`
- 与 runtime-model §1.8 async modules 的关联：patrolling 是 async module 范式的主要生产级示例（e2e 测试 `test_patrol_and_follow.py` 在 `dimos/e2e_tests/`）
- 与 navigation 其他子节（`global_planner` / `replanning_a_star`）的关系：巡逻是一类**任务**，复用 planner 子系统生成路径

**E.3 `simulation/unity/` → robot-platforms.md §6 新增 §6.6 Unity 后端**
- 位置锁定：robot-platforms.md §6 simulation/ 下现有 §6.1 四后端横向对比 / §6.2 MuJoCo / §6.3 Genesis / §6.4 Isaac / §6.5 engines——新增 §6.6 Unity（VLA Challenge），同时更新 §6.1 的对比表把四后端改为五后端（**仅 robot-platforms.md**，subsystems.md 不涉及）
- 内容：
  - `unity/module.py:222` `UnityBridgeModule(Module)`——走 ROS-TCP-Endpoint 二进制协议 TCP 桥接 Unity 仿真器
  - `unity/module.py:152` `UnityBridgeConfig`
  - `unity/blueprint.py` 蓝图
- 关键点：**无 ROS 依赖**——ROS-TCP-Endpoint 协议只是二进制格式，不需要运行 ROS master
- 用途：VLA Challenge benchmark 基线；与已有 MuJoCo（深度真实的物理）/ Genesis（高吞吐场景生成）/ Isaac（GPU 并行）四者的对比

**E.4 `robot/catalog/` + `robot/config.py` + `robot/model_parser.py` 写进 robot-platforms.md §0（平台总览章节首）**
- 位置：robot-platforms.md §0 平台总览之后，新增"统一机器人配置层（RobotConfig）"子节
- 内容：
  - `robot/config.py:50` `RobotConfig(BaseModel)`——单一真相源：从 URDF/MJCF 文件生成 `RobotModelConfig`（模型描述）+ `HardwareComponent`（硬件组件列表）+ `TaskConfig`（任务约束）；`:32` `GripperConfig` 夹爪子配置
  - `robot/model_parser.py:29,43` `JointDescription` / `ModelDescription` + `parse_model()` 函数——URDF 与 MJCF 解析器
  - `robot/catalog/` 四个 catalog 文件：`openarm.py` / `panda.py` / `piper.py` / `ufactory.py`——各家模型定义
- 关键点：替代原有的"per-robot 硬编码配置"模式；新机器人接入流程：写 catalog + 准备 URDF/MJCF → 自动生成 blueprint 组件
- 与 `dimos/robot/all_blueprints.py`（blueprint registry）的关系：`all_blueprints.py` 按 `catalog/` 条目注册硬件相关蓝图

**README §6 子系统目录更新**（与 D.3 协调）：
- 新增一行 `navigation/patrolling/` 或在 navigation 那一格里点名 patrolling 子包
- 新增一行 simulation 里的 unity 后端（或在 §5 平台层新增 Unity 小段）
- robot 那一格里提 `catalog/` + `config.py` 的统一配置层
- `hardware/whole_body/` 如果在 README §5 硬件层有总览，需要加一行；否则仅 robot-platforms.md 展开

### 批 F — 机器人平台尾收（robot-platforms.md 全面）

**Scope**：Go2 / G1 / 机械臂 / drone / simulation / sensors 六节扩充。

**F.1 §1.1 Go2（扩）**（**动笔时所有路径前缀一律 `dimos/robot/unitree/go2/`**）：
- 目录树补 `cli/`（PR #1990）、`connection_spec.py`
- 新子节「Go2 rage mode」：`Go2Mode.RAGE` enum（`connection.py` 枚举值行 :61，class 声明 :59）、`enable_rage_mode()`、`blueprints/basic/unitree_go2_webrtc_rage_keyboard_teleop.py` 蓝图（PR #1903，**在 `basic/` 下**不在 `blueprints/` 顶层）
- 新子节「Go2 CLI」：`cli/` 工具；与顶层 `dimos` CLI（§1.21 inventory）关系
- Agentic MCP 蓝图 5 个全名（agent-stack §5 / README §4 MCP 处引用时用全名）：`blueprints/agentic/unitree_go2_agentic.py` + `unitree_go2_agentic_huggingface.py` + `unitree_go2_agentic_ollama.py` + `unitree_go2_security.py` + `unitree_go2_temporal_memory.py`（还有 `_common_agentic.py` 是共享辅助）
- 时间戳章节补 PR #1992 lidar 时间戳修复 + PR #2021 自适应时间戳矫正

**F.2 §1.2 G1（扩）**（**动笔时所有路径前缀一律 `dimos/robot/unitree/g1/`**）：
- 目录树补 `wholebody_connection.py`（PR #1954）、`connection_spec.py`
- 分工：`connection.py` 仍是主 entry（高层运动命令），`wholebody_connection.py` **追加**低层全身控制通道，两者并存；`connection_spec.py` 是 Spec Protocol
- 新蓝图 `blueprints/basic/unitree_g1_coordinator.py`（**在 `basic/` 下**）
- G1 blueprints 子目录结构：`agentic/` / `basic/` / `perceptive/` / `primitive/` 四个子类别
- 与 §5 hardware/whole_body/（批 E.1）交叉引用：wholebody_connection.py 实现的就是 `WholeBodyAdapter(Protocol)`

**F.3 §4 机械臂（扩）——分层清楚讲**：
- `robot/manipulators/` 3 家：`piper/` / `xarm/` / `openarm/`（**无 mock**，mock 只在 hardware 层）；**新增 openarm**（PR #1897）
- `hardware/manipulators/` 5 实现：`mock/` / `piper/` / `xarm/` / `openarm/`（新）/ `sim/`（新）
- 新子节「OpenArm」：`robot/manipulators/openarm/` + `hardware/manipulators/openarm/{adapter,driver,test_driver}.py`
- 新子节「sim adapter」：`hardware/manipulators/sim/adapter.py` + `test_shm_adapter.py`——sim-backed ManipulatorAdapter（shm 与 MuJoCo 同步）
- 对比表：robot/ 3 列 × hardware/ 5 列两层矩阵

**F.4 §3 drone 扩写（当前文档已有占位 §3 drone，但可能空/浅）**：
- 位置锁定：robot-platforms.md **§3 drone**（现有 line 170，**已存在占位**，非"新增 §1.3"；v1 spec 写 §1.3 有误）
- 在原 §3 基础上按 `robot/drone/` inventory 扩实内容：
  - `connection_module.py` 主连接模块
  - `mavlink_connection.py` MAVLink 协议驱动
  - `drone_tracking_module.py` 跟踪模块
  - `drone_visual_servoing_controller.py` 视觉伺服控制器
  - `dji_video_stream.py` DJI 视频流
  - `camera_module.py` 相机模块
  - `blueprints/` 子目录
  - 链到 `robot/drone/README.md`
- 开工前 Read 现有 §3 的实际内容，按"补实"而非"重写"处理

**F.5 §5 hardware drive_trains（扩）**：
- 补 `drive_trains/unitree_go2/adapter.py` + README（Go2 硬件抽象）
- 补 `drive_trains/transport/adapter.py`

**F.6 §5 hardware manipulators / whole_body（已由 F.3 + E.1 覆盖）**：交叉引用即可

**F.7 §6 simulation（扩）**：
- 补 `mujoco/depth_camera.py`（`MujocoCamera`，PR #1694）——第一个 sim 内感知模块
- 补 MuJoCo 遥操作 xarm+piper（PR #1958）
- 补 `engines/mujoco_shm.py` / `mujoco_sim_module.py` / `registry.py`
- 补 `--simulation` flag（PR #2027）
- 补 replay 内存泄漏修复（PR #2025）
- `simulation/manipulators/` 已删除的说明（移到 `hardware/manipulators/sim/`）
- Unity 章见 E.3

**F.8 §5.4 sensors**：
- 说明 realsense `__init__.py` 删除 ≠ 目录消失；`camera.py` + `handeyeout_xarm6/` 仍在；导入方式需改为 `from dimos.hardware.sensors.camera.realsense.camera import ...`

### 批 G — 次级子系统补全 + 字串遗漏

**Scope**：subsystems.md 加 §14 stream/ + 附录 security_demo + §12 types 补 manipulation/robot_capabilities + agent-stack §3 skills 补 manipulation/ + msgs 新类型 + 路径改名 + `control/blueprints/` 结构改写 + visualization rename fan-out。

**G.1 subsystems.md 新增 §14 `dimos/stream/`（顶层包）**（~30 行）：
- 位置：subsystems.md §13 `rxpy_backpressure/` 之后（同属"数据流工具类"）
- 内容：`audio/` / `data_provider.py` / `frame_processor.py` / `video_provider.py` / `rtsp_video_provider.py` / `ros_video_provider.py` / `stream_merger.py` / `video_operators.py`；各一两句职责说明
- 关键点：与 `dimos/core/stream.py`（Module 间数据通道）**不是一回事**——应用层媒体流工具包

**G.2 subsystems.md 附录 `experimental/security_demo/`**（~20 行）：
- 位置：subsystems.md §扩展阅读 之前新增"附录：实验模块（experimental/）"
- 内容：`depth_estimator.py` + `security_module.py` + `conftest.py` + `test_security_module.py`
- 明确"experimental"：API 不稳定、不进 blueprint registry、不编号（因不稳定）

**G.3 subsystems.md §12 types 补**：
- `manipulation.py`（8 类 `ConstraintType` / `AbstractConstraint` / `TranslationConstraint` / `RotationConstraint` / `ForceConstraint` / `ObjectData` / `ManipulationMetadata` / `ManipulationTask`）——OpenArm 随行
- `robot_capabilities.py`（`RobotCapability` enum）
- `ros_polyfill.py` / `sample.py` / `constants.py`

**G.4 agent-stack.md §3 skills 补**：
- 新增「`skills/manipulation/` 子包」小节（6 类：`abstract_manipulation_skill.py` / `force_constraint_skill.py` / `manipulate_skill.py` / `pick_and_place.py` / `rotation_constraint_skill.py` / `translation_constraint_skill.py`）
- 与 `types/manipulation.py`（G.3）交叉引用——types 层定义约束类型，skills 层实现约束化动作
- `skills/unitree/unitree_speak.py` 和 `skills/rest/rest.py` 也补点名（避免漏）

**G.5 subsystems.md §11 msgs 补**：
- `sensor_msgs/` 新增：`JointCommand.py` / `MotorCommandArray.py` / `RobotState.py` / `image_impls/`（子目录）
- 说明 `MotorCommandArray` 是 G1 wholebody（批 F.2）的消息载体

**G.6 subsystems.md §7 models 补**：
- 路径修正：`vl/video_query.py` → `models/qwen/video_query.py`（批 A 已做字串，此处补周边描述）
- `vl/` 子树补 `moondream_hosted.py`（若 README 已有可跳过，但 subsystems.md §7 需确认有）

**G.7 subsystems.md §1 control 结构改写**：
- `control/blueprints.py` 单文件树节点 → `control/blueprints/` 目录含 `basic.py` / `dual.py` / `mobile.py` / `teleop.py`
- 因树状代码块不是字串替换，必须结构改写，归批 G（不归批 A）
- `control/components.py` / `coordinator.py` / `hardware_interface.py`（随 G1 wholebody PR 改动）补描述

**G.8 visualization rename fan-out**：
- 批 A 已改关键词；批 G 确认 subsystems.md §9 正文描述、README §6 目录表、data-flow 中的所有 `RerunBridge`/`HasToRerun` 交叉引用已同步
- 最后一次全文 grep 确保 0 命中

**G.9 其他小遗漏**：
- `agents_deprecated/` 段落（agent-stack §6.5 / README §4）确认仍说"严禁新代码使用"（evidence：文件仍存在，但 README:445 + agent-stack:568 原文"请使用 `dimos.agents.agent.Agent`"需在批 C 改）
- `web/websocket_vis_spec.py` 新 Spec——补 §8 web 一句
- `utils/workspace.py`（314 行，OpenArm 随行）：在 manipulators 章节（批 F.3）提一句"OpenArm 随行的 `dimos/utils/workspace.py` 提供 URDF/workspace 工具"
- `perception` 从基础安装移出为可选 extra 的说明（批 A 已加）

**G.10 round 2 subagent 发现整合**：
- **subsystems.md:206,226 §4 manipulation mermaid + 散文**：`MON[WorldMonitor<br/>planning/monitor/]` 节点 + "`WorldMonitor`（`planning/monitor/`）监控实时世界状态"散文 → 改为 3 节点/3 类 `WorldMonitor(world_monitor.py)` + `WorldObstacleMonitor(world_obstacle_monitor.py)` + `RobotStateMonitor(robot_state_monitor.py)`
- **README.md:645 simulation 表**：删 `| manipulators/ | 仿真内机械臂接口... |` 行，改为一行"（已迁至 `hardware/manipulators/sim/`，见 robot-platforms §5.3）"
- **subsystems.md §3 navigation 补一句**：`navigation/visual/query.py:21` `get_object_bbox_from_image(vl_model, image, object_description)` — VLM→bbox 桥接，`visual_navigation_skills.py` 调用
- **subsystems.md §4 manipulation 补 `servo_control/`**：`control/servo_control/cartesian_motion_controller.py:87` `CartesianMotionController(Module)` 与 `trajectory_controller/` 关节空间路径并列的 Cartesian 伺服路径
- **perception 可选 extra 跨子系统影响**：在 subsystems §2 perception 或 README §6 perception 格子里加一段「硬依赖子系统」清单：`agents.skills.navigation` / `manipulation/`（多个模块）/ `models/vl/*` / `navigation/visual_servoing/` / `memory2/vis/utils.py` / `experimental/security_demo/` — 这 6 组在不装 perception extra 时会 ImportError（不列 62 个文件名，只列受影响子系统）
- **批 B §1.4 末尾加一句**：`protocol/rpc/pubsubrpc.py` 仍存在但已从 coordination 路径 de-link（PR #9d7806615），rpyc 是当前主 IPC；旧 pubsubrpc 路径可作 reference 读，不会在启动时被触发

## 4. 验证策略（每条都要能 git 证明）

### 4.1 每 commit 之前

对该批次引入的**每一条**新 path / class / env var / blueprint：
- 文件路径：`git ls-tree origin/main -- <path>` 必须非空
- 类名：`git grep -nE "^class <name>\b" origin/main -- dimos/` 必须有命中；如涉及行号引用，行号要对
- 环境变量字面值：`git grep -n "<literal>" origin/main -- dimos/core/` 验证
- 外部链接（`docs/usage/*` 等）：`git ls-tree origin/main -- <path>` 验在

### 4.2 每 commit 之后

**旧关键词应消失清单**（批 A 到 G 全部完成后，整个 `docs/architecture/` 对这些 grep 应为 0）：
- 路径：`dimos/core/worker_manager\.py`、`dimos/core/module_coordinator\.py`、`dimos/core/blueprints\.py`、`dimos/core/worker\.py`、`dimos/agents/agent\.py`、`dimos/control/blueprints\.py`（作单文件引用）、`dimos/mapping/types\.py`、`vl/video_query\.py`
- 类/接口：`\bActor\b`（除非上下文明确"已删除"）、`MethodCallProxy`、`SpatialPerception\b`、`RerunBridge\b`（单独出现，不带 Module）、`HasToRerun`
- Logger：`dimos\.core\.module_coordinator`（新为 `dimos.core.coordination.*`）
- 蓝图："`unitree-go2-agentic-mcp` 是唯一"类论断
- Agent 相关：`LangGraph`、`LangChain`、`create_agent`、"`on_system_modules` 是 Agent 核心"类论断
- 计数："13 个一级子系统"（新为 14）

**新关键词应至少出现一次**（批 A 到 G 完成后，整个 `docs/architecture/` 对这些 grep 应 ≥1）：
`coordination/`、`AgentSpec`、`agent_spec\.py`、`demo_agent(_camera)?`、`ToolStream`、`TOOL_STREAM_TOPIC`、`porcelain`、`\bDimos\b`、`memory2`、`StreamModule`、`Transformer\[`、`VoxelMapTransformer`、`WorkerManagerPython`、`WorkerManagerDocker`、`RpycServer`、`RPCClient`、`ModuleProxyProtocol`、`watchdog_main`、`NativeModule`、`WorkerResponse`、`load_blueprint`、`load_module`、`rpyc_port`、`OpenArm` / `openarm`、`Go2Mode\.RAGE`、`wholebody_connection`、`WholeBodyAdapter`、`MotorCommand`、`PatrollingModule`、`PatrolRouter`、`UnityBridgeModule`、`RobotConfig`、`model_parser`、`MujocoCamera`、`dimos/robot/drone`、`skills/manipulation/`、`types/manipulation\.py`、`robot_capabilities`、`RerunBridgeModule`、`RerunConvertible`

### 4.3 批 B / C / E 完成后派 Explore subagent 交叉验证

对每个这三批：派一个 Explore subagent，指令是"读 `docs/architecture/<相应文件>` 再对照 `origin/main` 找出还有没有错"。subagent 只做 read-only。发现问题在同批次 commit 前修，不留到下一批。

### 4.4 pre-commit

每 commit 单跑一次（不要批量）。失败修复再 commit（不 amend）。

## 5. 风险与缓解

| 风险 | 缓解 |
|---|---|
| 又有新 main commit 到来 | 开工前 `git fetch origin main`，若 `main HEAD ≠ merge base` 重新 rebase + 重跑本 spec §1 inventory |
| 外部 reviewer 的结论**未经 git 验证**导致文档改错（v1 教训：memory2/vis/space/ 等 3 处反被误改） | 所有改动只以本 spec §1 inventory 为依据；reviewer 建议必须能落到 §1 的一条 evidence 上，否则不采纳 |
| rpyc / rpyc_services / RPCClient 具体机制我没通读源码 | 批 B 开工前派 Explore 读 `coordination/rpyc_server.py` + `coordination/rpyc_services.py` + `core/rpc_client.py` 全文，把握 session / connection / proxy 语义后再动笔 |
| memory2 backend / notifier / vis/space 具体实现我没全看 | 批 D 开工前读 `memory2/backend.py` + `notifier/` + `vis/space/` 全文 |
| `control/blueprints/` 目录改写会破代码块格式 | 批 G 专门用 Read 把 subsystems.md §1 的代码块读出，手工重排，跑 pre-commit |
| drone 从未进文档，没有现成描述可改 | 批 F.4 前读 `robot/drone/README.md` + 主要模块头部注释，基于 README 首句重写 |
| data-flow.md 行号全漂移 | 全部去掉行号，只写 class/function 名；个别行号经 `git grep` 复核后才保留 |

## 6. 交付清单

- [ ] 批 A 散点 commit（5 份文档字串级修正）
- [ ] 批 B runtime-model §1 重写 + RunEntry 表 + README §3 散文同步
- [ ] 批 C agent-stack §1/§5/§6.1 重写 + README §4 类表 + §1280 锚点表 + tool streams
- [ ] 批 D memory2 主体 + README §4.5 Porcelain + README §6 流程图/计数 + data-flow 补丁
- [ ] 批 E 4 个新主干子系统（whole_body、patrolling、unity、robot-catalog+config）
- [ ] 批 F 机器人平台尾收（Go2 rage/CLI、G1 wholebody、OpenArm、MujocoCamera、sim adapter、drone 首次进文档）
- [ ] 批 G 次级子系统补全（stream/、security_demo、types/manipulation、skills/manipulation、msgs 新增、control/blueprints/ 目录、visualization rename fan-out 收口、**round 2 整合的 5 项**：subsystems §4 mermaid `WorldMonitor` 拆 3 类、README §5 simulation 表删 `manipulators/` 行、subsystems §3 加 `navigation/visual/query.py`、subsystems §4 加 `servo_control/CartesianMotionController`、perception extra 硬依赖子系统清单）
- [ ] 4.2 旧关键词 0 命中、新关键词 ≥1 命中验证通过
- [ ] 批 B / C / E 各派一次 Explore subagent 交叉验证
- [ ] 所有 commit pre-commit 通过
- [ ] 一次性 push，CI 通过

## 7. 与 2026-05-06 spec / 2026-05-09 v1 的关系

- 2026-05-06 spec + plan：第一轮（写文档）。归档不动
- 2026-05-09 v1 spec：第二轮 draft，已撤销（4 类系统性缺陷见 §0）
- 本 v2：第二轮最终版，方法论从"基于 audit + reviewer 反驳"改为"基于 main-side inventory"











