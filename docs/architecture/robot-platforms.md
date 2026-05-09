# 专题：机器人平台（Robot Platforms）

> 与 [README § 5](/docs/architecture/README.md#-5-机器人平台层) 配套深入：unitree/unitree_webrtc/drone/manipulators/hardware/simulation 各层详解。
> 目标读者：正在接入新机器人平台、末端执行器、传感器或仿真后端的工程师。

## 目录

- [0. 统一机器人配置层](#0-统一机器人配置层)
- [1. unitree](#1-unitree)
- [2. unitree_webrtc](#2-unitree_webrtc)
- [3. drone](#3-drone)
- [4. manipulators](#4-manipulators)
- [5. hardware/](#5-hardware)
- [6. simulation/](#6-simulation)
- [7. 蓝图示例](#7-蓝图示例)
- [8. 添加新平台 / 末端 / 传感器](#8-添加新平台--末端--传感器)

---

## 0. 统一机器人配置层

`dimos/robot/config.py` + `dimos/robot/model_parser.py` + `dimos/robot/catalog/` 一起构成 DimOS 当前的**统一机器人配置层**——替代此前各平台子包里零散的"per-robot 硬编码配置"。**单一真相源 = URDF/MJCF 模型文件**，其余字段（关节拓扑、硬件组件清单、任务配置）由模型文件派生。

**入口层 `robot/config.py`**：

- `RobotConfig(BaseModel)` —— 顶层配置；通常由 `RobotConfig.from_urdf(...)` / `from_mjcf(...)` 从模型文件构造。内部组合 `RobotModelConfig`（解析结果）+ `HardwareComponent` 列表（夹爪、相机、传感器）+ `TaskConfig`（任务/技能开关）。
- `GripperConfig(BaseModel)` —— 夹爪子配置（型号名、开合量程、控制模式），作为 `HardwareComponent` 的一种挂到 `RobotConfig` 上。

**解析层 `robot/model_parser.py`**：

- `JointDescription` —— 单关节描述（名称、类型、父/子 link、axis、limits）。
- `ModelDescription` —— 整模型描述（关节列表 + 根 link + 元数据）。
- `parse_model(path) -> ModelDescription` —— 入口函数，按扩展名分派到 `_parse_urdf_string` / `_parse_mjcf_file`；URDF 走 xacro 展开 + ElementTree 解析，MJCF 走 `_walk_mjcf_bodies` 递归遍历。

**型号目录 `robot/catalog/`**：当前覆盖四家厂商——`openarm.py`、`panda.py`、`piper.py`、`ufactory.py`。每个文件导出该系列的默认 `RobotConfig` 构造器（URDF 路径 + 夹爪默认值 + 任务配置），上层直接 import 即可得到可跑的 `RobotConfig`。

**新机器人接入流程**：

1. 在 `dimos/robot/catalog/<name>.py` 写一个构造函数；准备对应 URDF / MJCF 文件。
2. 内部调用 `RobotConfig.from_urdf(path, gripper=GripperConfig(...), ...)`；`parse_model` 自动提取关节拓扑。
3. 在蓝图中注入该 `RobotConfig`，并按 `AGENTS.md` → "blueprint registry" 把新蓝图注册进 `dimos/robot/all_blueprints.py`（**自动生成**，触发 `pytest dimos/robot/test_all_blueprints_generation.py` 重建）。

---

## 1. unitree

`dimos/robot/unitree/` 是宇树科技四足机器人与人形机器人的核心集成层，整体结构如下：

```
dimos/robot/unitree/
├── go2/                   # Go2 四足犬型机器人
├── g1/                    # G1 人形机器人
├── b1/                    # B1 大型四足（试验中）
├── modular/               # 通用检测探针（★ 见成熟度提示）
├── params/                # 出厂传感器标定参数
├── testing/               # 测试辅助：Mock、actors、工具
├── type/                  # 传感器消息类型定义
├── connection.py          # WebRTC 实时连接核心
├── mujoco_connection.py   # MuJoCo 仿真共享内存适配层
├── keyboard_teleop.py     # Pygame 键盘遥控 Module
├── rosnav.py              # ROS 导航桥接（已标记 deprecated）
├── demo_error_on_name_conflicts.py  # 名称冲突演示
└── unitree_skill_container.py       # 宇树统一技能集合
```

### 1.1 go2/

Go2 是当前文档与示例最为丰富、运行时测试覆盖率最高的平台，也是默认代理系统提示所针对的型号。

```
go2/
├── blueprints/
│   ├── agentic/   # 包含 MCP、HuggingFace、Ollama、temporal-memory 等变种
│   ├── basic/     # unitree_go2_basic, unitree_go2_fleet
│   └── smart/     # 感知增强型（detection, ros, spatial, vlm_stream_test）
├── connection.py      # 继承 UnitreeWebRTCConnection，注入 Go2 专有参数
├── fleet_connection.py  # 多机群控连接管理
└── go2.urdf           # URDF 模型文件（供 MuJoCo 加载）
```

蓝图按功能深度分三层：`basic` 只做连接与遥控；`smart` 叠加感知与地图；`agentic` 进一步叠加 LLM 代理与技能调用。`unitree-go2-agentic-mcp` 是目前唯一内置 `McpServer` 的蓝图，是所有 `dimos mcp …` 命令的唯一有效目标。

### 1.2 g1/

G1 是双足人形机器人，其集成深度与 Go2 相当，但有几处关键差异：

```
g1/
├── blueprints/
│   ├── agentic/    # unitree_g1_agentic, unitree_g1_agentic_sim, unitree_g1_full
│   ├── basic/      # unitree_g1_basic, unitree_g1_basic_sim, unitree_g1_joystick
│   ├── perceptive/ # unitree_g1, unitree_g1_detection, unitree_g1_shm, unitree_g1_sim
│   └── primitive/  # uintree_g1_primitive_no_nav（无导航极简版）
├── connection.py      # G1ConnectionBase（抽象基类）+ 真实与仿真子类
├── sim.py             # G1 MuJoCo 专用场景加载
├── skill_container.py # G1 专属技能（含 speak 接口）
└── system_prompt.py   # G1_SYSTEM_PROMPT 常量
```

**注意**：G1 有独立的 `G1_SYSTEM_PROMPT`（`system_prompt.py`），描述了"Daneel"这一代理身份以及 G1 专用技能列表。如果在 G1 上使用默认 Go2 系统提示，LLM 会产生幻觉技能，造成运行时调用失败。启动 G1 代理蓝图时务必确认：

```python
# 正确：显式传入 G1 系统提示
from dimos.robot.unitree.g1.system_prompt import G1_SYSTEM_PROMPT
agent = UnitreeAgent(..., system_prompt=G1_SYSTEM_PROMPT)
```

### 1.3 b1/

B1 是宇树工业级四足机器人，当前处于基础集成阶段：

```
b1/
├── b1_command.py      # B1 专有指令封装
├── connection.py      # 连接 Module
├── joystick_module.py # UDP 手柄输入
├── joystick_server_udp.cpp  # C++ UDP 服务端（需独立编译）
├── README.md
├── test_connection.py
└── unitree_b1.py      # B1 顶层类
```

B1 使用 UDP 手柄服务器（`joystick_server_udp.cpp`），与 Go2/G1 基于 WebRTC 的连接路径不同。没有与之对应的 agentic 蓝图，当前主要满足低层遥控与连接验证需求。

### 1.4 modular/（★ 探针级，非生产就绪）

`modular/` 目前仅有一个文件 `detect.py`，其 git 历史显示只有两次提交，属于内部探针性质的早期代码，**尚未投入生产**。`detect.py` 演示了如何从录制的传感器数据（lidar、video、odom）中运行 2D 目标检测并广播至 LCM，是测试检测管线的单文件脚本，不代表通用的模块化机器人接口架构。

### 1.5 顶层公共文件

**`connection.py`**（`dimos/robot/unitree/connection.py`）

`UnitreeWebRTCConnection` 是所有宇树平台真实硬件连接的基础。它通过 `unitree_webrtc_connect` Python 包建立 WebRTC 通道，对外暴露以下 RxPY Observable 流：

- 视频流：`NDArray[uint8]`（H.264 解码后的 BGR 帧）
- 点云流：`PointCloud2`（来自内置激光雷达）
- 里程计：`Odometry`（融合惯量与运动学）
- 底层状态：`LowStateMsg`（电机温度、电压、关节力矩）

`connection.py` 还持有 `WebRTCConnectionMethod`（直连 / AP 模式 / 中继）的选择逻辑，以及内置的视频帧反序列化器 `SerializableVideoFrame`（Pickle 友好，用于跨进程传输）。

**`mujoco_connection.py`**

当平台在仿真中运行时，`mujoco_connection.py` 通过共享内存（ShmWriter/ShmReader）替代 WebRTC 通道，向外暴露与真实连接完全相同的 Observable 接口。仿真进程（MuJoCo subprocess）写入共享内存块，连接模块读取并转发，确保蓝图代码无需区分真实与仿真环境。

**`keyboard_teleop.py`**

`KeyboardTeleop` 是一个标准 `Module`，用 Pygame 捕获键盘事件，输出 `Twist` 消息到 `/cmd_vel`。使用时需要 X11 显示环境（已在代码内强制 `SDL_VIDEODRIVER=x11`）。通常与 `unitree-go2-basic` 或 `unitree-g1-basic` 蓝图组合使用。

**`rosnav.py`**

`NavigationModule` 已标记为 `deprecated`，是早期 ROS 导航适配层的遗留实现。新代码应使用 `dimos/navigation/` 下的原生导航实现。

**`demo_error_on_name_conflicts.py`**

演示文件，展示当两个 Module 的输出流类型相同时 `autoconnect` 如何抛出歧义错误，用于帮助开发者理解蓝图自动连接规则。

**`unitree_skill_container.py`**

定义了 `UNITREE_WEBRTC_CONTROLS` 常量（包含 `BalanceStand`、`StandUp`、`StandDown`、`RecoveryStand`、`Sit`、`RiseSit` 等约 20 种运动指令）以及对应的 `@skill` 包装函数，是代理系统调用宇树运动接口的统一入口。每个 `@skill` 函数必须满足：有 docstring、所有参数有类型注解、返回 `str`，否则启动时技能注册失败。

### 1.6 params/、testing/、type/

| 子目录 | 用途 |
|---|---|
| `params/` | YAML 格式的出厂传感器标定参数（`front_camera_720.yaml`、`sim_camera.yaml`） |
| `testing/` | 单元测试辅助（`helpers.py`、`mock.py`、`test_actors.py`、`test_tooling.py`） |
| `type/` | 平台专有消息类型：`lidar.py`（PointCloud2 转换）、`lowstate.py`（底层状态）、`map.py`（占据栅格）、`odometry.py`（里程计）、`timeseries.py`（时序索引）、`vector.py`（向量工具） |

`type/` 中的转换函数（如 `pointcloud2_from_webrtc_lidar`）将宇树 WebRTC 私有消息格式转为 dimos 标准 `sensor_msgs`，是跨平台数据流统一的关键。

---

## 2. unitree_webrtc

`dimos/robot/unitree_webrtc/` 是 `dimos/robot/unitree/` 的**兄弟目录，不是子目录**：

```
dimos/robot/unitree_webrtc/
├── README.md
└── type/
    ├── lidar.py
    ├── lowstate.py
    ├── map.py
    ├── odometry.py
    ├── timeseries.py
    └── vector.py
```

该目录仅因历史原因存在。其 `README.md` 明确说明：*"This directory only exists because some of the --replay tests depend on its existence (python pickle uses module names/paths so we would need to redo the pickle files)."* 即 Python Pickle 在反序列化时依赖模块路径字符串，若删除该目录，所有使用 `--replay` 模式重放的测试录制数据将无法加载。

实际类型定义已迁移至 `dimos/robot/unitree/type/`，`unitree_webrtc/type/` 中的文件是副本或符号引用。**新代码不应从 `unitree_webrtc/` 导入**，统一从 `dimos.robot.unitree.type` 导入。

---

## 3. drone

`dimos/robot/drone/` 提供基于 MAVLink 协议的无人机支持，于 PR #1520（`feat(drone): modernize drone for CLI + Rerun + replay`）完成现代化改造，接入 CLI 管理与 Rerun 可视化。

```
dimos/robot/drone/
├── blueprints/
│   ├── agentic/   # drone_agentic（含 LLM 代理）
│   └── basic/     # drone_basic（仅 MAVLink + 视频）
├── camera_module.py         # GStreamer / DJI 摄像头输入 Module
├── connection_module.py     # MAVLink 连接 Module 包装
├── dji_video_stream.py      # DJI 私有视频流解析
├── drone_tracking_module.py # 视觉跟踪控制 Module
├── drone_visual_servoing_controller.py  # 视觉伺服控制器
├── mavlink_connection.py    # MavlinkConnection：pymavlink 封装
├── README.md
└── test_drone.py
```

`MavlinkConnection`（`mavlink_connection.py`）通过 `pymavlink` 库建立 UDP 连接（默认 `udp:0.0.0.0:14550`），暴露 `Twist` 速度指令接口和 `PoseStamped` 位姿输出流。连接支持 `outdoor` 模式（GPS 定位）与室内模式（光流定位）切换。

`drone_visual_servoing_controller.py` 实现了基于图像的视觉伺服（IBVS）控制回路，结合 `drone_tracking_module.py` 中的目标跟踪算法，可实现对移动目标的自主跟随。

当前已发布蓝图：

| 蓝图名称 | 入口 | 功能 |
|---|---|---|
| `drone-basic` | `drone_basic` | MAVLink 连接 + 视频流 |
| `drone-agentic` | `drone_agentic` | 基础功能 + LLM 代理技能 |

---

## 4. manipulators

`dimos/robot/manipulators/` 下包含两款机械臂的驱动层：

```
dimos/robot/manipulators/
├── piper/
│   ├── __init__.py
│   └── blueprints.py
└── xarm/
    ├── __init__.py
    └── blueprints.py
```

### 4.1 piper/

AgileX Piper 六轴协作机械臂。`blueprints.py` 中定义的已发布蓝图包括：

- `arm-teleop-piper`：Quest 手柄遥控
- `coordinator-cartesian-ik-piper`：笛卡尔空间 IK 协调器
- `coordinator-piper`：基础关节空间协调器
- `coordinator-piper-xarm`：Piper + xArm 双臂协调
- `coordinator-teleop-piper`：遥控 + 协调器组合
- `keyboard-teleop-piper`：键盘遥控

### 4.2 xarm/

UFACTORY xArm 系列（xArm6 / xArm7）。`blueprints.py` 中定义的蓝图涵盖：

- `arm-teleop-xarm7`：Quest 手柄遥控
- `coordinator-xarm6` / `coordinator-xarm7`：关节空间协调器
- `coordinator-dual-xarm`：双 xArm 协同
- `coordinator-velocity-xarm6`：速度模式协调器
- `coordinator-teleop-xarm6` / `coordinator-teleop-xarm7`：遥控 + 协调组合
- `dual-xarm6-planner`：双臂轨迹规划器
- `keyboard-teleop-xarm6` / `keyboard-teleop-xarm7`：键盘遥控
- `xarm-perception` / `xarm-perception-agent`：感知视觉管线（含代理版）
- `xarm6-planner-only` / `xarm7-planner-coordinator` / `xarm7-planner-coordinator-agent`：规划器变种

机械臂驱动层在 `dimos/hardware/manipulators/` 中通过 `ManipulatorAdapter` 协议（Protocol）接口与上层 Module 解耦，详见第 5 节。

---

## 5. hardware/

`dimos/hardware/` 提供四类硬件抽象，每类都以 Python `Protocol`（`runtime_checkable`）或简单基类定义接口，支持 Mock 替换与单元测试：

```
dimos/hardware/
├── drive_trains/    # 移动底盘速度控制协议
├── end_effectors/   # 末端执行器基类
├── manipulators/    # 机械臂适配器协议 + 驱动实现
└── sensors/         # 传感器驱动（摄像头、激光雷达）
```

### 5.1 drive_trains/（驱动系统）

核心接口：`TwistBaseAdapter`（`drive_trains/spec.py`）

```python
@runtime_checkable
class TwistBaseAdapter(Protocol):
    def connect(self) -> bool: ...
    # 速度控制（SI 单位：m/s, rad/s）
```

适用于：四足机器人（Go2、G1）、无人机、RC 小车、全向轮底盘等一切接受 `Twist` 速度指令的平台。模拟实现位于 `drive_trains/mock/`，可在无硬件环境下通过类型检查（`isinstance(obj, TwistBaseAdapter)`）验证适配器正确性。`drive_trains/flowbase/` 提供基于 RxPY 的响应式包装。

### 5.2 end_effectors/（末端执行器）

核心类：`EndEffector`（`end_effectors/end_effector.py`）

```python
class EndEffector:
    def __init__(self, effector_type=None) -> None: ...
    def get_effector_type(self): ...
```

这是目前框架层最轻量的抽象——`EndEffector` 仅持有类型标识符，具体控制逻辑由 `manipulators/` 中的机械臂驱动器或专有夹爪驱动承担。`end_effectors/mock/` 提供测试用桩。注册自定义末端执行器类型通过 `end_effectors/registry.py` 完成。

### 5.3 manipulators/（机械臂）

核心接口：`ManipulatorAdapter`（`manipulators/spec.py`）

```python
@runtime_checkable
class ManipulatorAdapter(Protocol):
    # 关节状态读取、轨迹跟踪、末端位姿控制
    # DriverStatus 枚举：DISCONNECTED / CONNECTED
```

已实现驱动：
- `manipulators/piper/`：AgileX Piper 驱动
- `manipulators/xarm/`：UFACTORY xArm 驱动
- `manipulators/mock/`：单元测试 Mock 驱动

`manipulators/registry.py` 维护驱动名称到类的映射，Module 层通过名称字符串选择驱动，实现运行时可配置切换。

### 5.4 sensors/（传感器）

```
sensors/
├── camera/
│   ├── gstreamer/  # GStreamer 管线采集
│   ├── realsense/  # Intel RealSense D 系列
│   ├── webcam.py   # USB 摄像头
│   ├── zed/        # Stereolabs ZED 系列
│   ├── module.py   # 统一 CameraModule 基类
│   └── spec.py     # 摄像头接口协议
├── lidar/
│   ├── common/     # 通用激光雷达工具
│   ├── fastlio2/   # Fast-LIO2 融合 SLAM
│   └── livox/      # Livox Mid360 驱动（含 mid360 蓝图）
└── fake_zed_module.py  # ZED 仿真替代（仅 MuJoCo 环境）
```

激光雷达部分已有生产级蓝图：`mid360`（原始点云）、`mid360-fastlio`（Fast-LIO2 里程计）、`mid360-fastlio-voxels`（体素地图）、`mid360-fastlio-voxels-native`（本地进程版）。

摄像头传感器通过 `spec.py` 中的统一协议对上层模块暴露 `Image` 流，RealSense 同时输出 RGB 与深度图像，ZED 额外输出立体视差与空间坐标。

### 5.5 whole_body/（全身多关节控制 HAL）

`dimos/hardware/whole_body/` 是 `drive_trains/` 与 `manipulators/` 之外的第三类 HAL 范畴，服务于 **≥20 DoF 级双足 + 双臂 + 躯干同步控制**场景（典型平台：G1 人形）。

```
dimos/hardware/whole_body/
├── spec.py              # WholeBodyAdapter Protocol + 三类数据 dataclass
├── registry.py          # 适配器 registry（按名称选实现）
└── transport/
    └── adapter.py       # 传输级适配器（封装底层通道）
```

- `transport/adapter.py` — 传输级适配器基类（LCM / rpyc 两路注入 `MotorCommand` / `MotorState`；平台实现按传输差异化）。

核心接口：`WholeBodyAdapter`（`whole_body/spec.py`）

```python
class WholeBodyAdapter(Protocol):
    def send_command(self, cmd: MotorCommand) -> None: ...
    def get_state(self) -> tuple[MotorState, IMUState]: ...
```

数据三件套（均为 dataclass）：

- `MotorCommand`：每电机 `q / dq / kp / kd / tau`（位置/速度/刚度/阻尼/前馈力矩）——PD + 前馈的统一指令模型。
- `MotorState`：每电机当前 q / dq / 力矩反馈。
- `IMUState`：躯干姿态（quat / gyro / accel）。

**与兄弟节的区别**：

- `drive_trains/` = 底盘级 `Twist`（m/s + rad/s），单一速度向量驱动整机移动。
- `manipulators/` = 单臂 / 夹爪轨迹跟踪（≤7 DoF + 末端）。
- `whole_body/` = **一帧内同步下发全身所有关节**的 PD+前馈指令，对时序一致性要求最高；命令/状态打包粒度完全不同，无法用上述两者表达，因此独立成一层。

**G1 实现指针**：具体实现见 §1.2 G1 节（Batch F 扩写）——`dimos/robot/unitree/g1/wholebody_connection.py` 实现 `WholeBodyAdapter`，将 23/29 DoF 命令序列化为 Unitree SDK DDS 包。

---

## 6. simulation/

`dimos/simulation/` 包含**五个仿真后端**，它们既相互独立又共用同一个 `SimulatorBase` 抽象层：

```
dimos/simulation/
├── mujoco/          # MuJoCo（主力后端，最成熟）
├── genesis/         # Genesis（轻量级，GPU 加速）
├── isaac/           # Isaac Sim（NVIDIA，USD 场景）
├── unity/           # Unity（VLA Challenge 基线，TCP 桥接）
├── engines/         # 低层引擎适配（SimulationEngine ABC）
├── base/            # SimulatorBase 共享基类
├── manipulators/    # 机械臂仿真 Module（xarm7_trajectory_sim）
├── utils/           # MJCF XML 解析工具
└── sim_blueprints.py # 已发布的仿真蓝图
```

### 6.1 五后端横向对比

| 维度 | MuJoCo | Genesis | Isaac Sim | Unity | engines/ |
|---|---|---|---|---|---|
| **文件格式** | MJCF XML（`.xml`） | URDF / MJCF / primitive | USD（`.usd`） | Unity scene（外部工程） | MJCF XML |
| **GPU 需求** | 否（CPU 可运行）| 推荐（CUDA） | 必须（NVIDIA RTX）| 取决于 Unity 端 | 否 |
| **物理精度** | 高（接触力） | 中等（刚体主导） | 极高（PhysX 5） | Unity PhysX（基线级） | 高 |
| **渲染** | OpenGL（mujoco.viewer） | 内置 rasterizer | RTX 光线追踪 | Unity 原生 | headless 可选 |
| **共享内存** | 是（ShmWriter/ShmReader） | 否 | 否 | 否（TCP 桥接） | 否 |
| **主要用途** | Go2/G1 sim 蓝图、机械臂轨迹 | 快速原型 | 渲染真实感场景 | VLA Challenge benchmark | 机械臂 Module 层 |
| **成熟度** | 生产级 | 早期集成 | 占位级 | benchmark 基线 | 生产级（xarm） |

### 6.2 MuJoCo 后端（最成熟）

MuJoCo 通过两个机制接入 dimos：

1. **进程分离 + 共享内存**（`mujoco/mujoco_process.py` + `shared_memory.py`）：仿真进程独立运行，用 `ShmWriter` 将传感器数据写入共享内存，连接 Module（`mujoco_connection.py`）用 `ShmReader` 读取，对上层蓝图暴露与真实硬件相同的 Observable 流接口。这是 Go2 和 G1 仿真蓝图（`unitree-g1-basic-sim`、`unitree-g1-agentic-sim` 等）采用的架构。

2. **引擎层**（`engines/mujoco_engine.py`）：`MujocoEngine` 继承 `SimulationEngine` 抽象基类，在同进程内加载 MJCF XML，以线程方式步进物理模拟，接受 `JointState` 指令流。机械臂仿真（`xarm7_trajectory_sim`）采用此路径。

MuJoCo 仿真环境配置项（位于 `mujoco/constants.py`）：

```python
VIDEO_WIDTH = 1280
VIDEO_HEIGHT = 720
VIDEO_FPS = 30
LIDAR_FPS = 10
VIDEO_CAMERA_FOV = 60.0
DEPTH_CAMERA_FOV = 60.0
```

`mujoco/policy.py` 提供 ONNX 控制策略（`Go1OnnxController`、`G1OnnxController`），可直接加载 `.onnx` 格式的强化学习策略并以固定频率推理。

### 6.3 Genesis 后端

`genesis/simulator.py` 中的 `GenesisSimulator` 继承 `SimulatorBase`，初始化时调用 `gs.init()` 并创建带查看器的 `gs.Scene`。支持 `mesh`、`urdf`、`mjcf`、`primitive` 四种实体类型，通过 `entities` 列表配置场景。Genesis 的优势在于可直接加载 MJCF（兼容 MuJoCo 场景文件）同时提供 GPU 加速的并行仿真，适合强化学习数据采集。当前集成处于早期阶段，无已发布蓝图。

### 6.4 Isaac Sim 后端

`isaac/simulator.py` 中的 `IsaacSimulator` 在构造时调用 `SimulationApp({"headless": headless, "open_usd": open_usd})`，依赖 NVIDIA Omniverse 运行时。USD 场景路径通过 `open_usd` 参数传入，关闭时调用 `app.close()`。目前属于占位级接口，不具备完整的传感器流导出能力，适合与 Isaac Lab 进行联调验证。

### 6.5 engines/ 通用引擎层

`engines/base.py` 定义 `SimulationEngine` 抽象基类：

```python
class SimulationEngine(ABC):
    def __init__(self, config_path: Path, headless: bool) -> None: ...
    @abstractmethod
    def step(self, joint_state: JointState) -> JointState: ...
    @abstractmethod
    def reset(self) -> JointState: ...
    @abstractmethod
    def close(self) -> None: ...
```

`engines/mujoco_engine.py` 是唯一已实现的引擎。该层的价值在于：机械臂仿真 Module（`simulation/manipulators/sim_module.py`）通过 `engine="mujoco"` 字符串选择引擎，使业务逻辑与具体物理引擎解耦，为日后接入 Genesis 引擎做好了接口预留。

### 6.6 Unity 后端（VLA Challenge 基线）

`dimos/simulation/unity/` 是为 VLA Challenge benchmark 准备的第五个仿真后端；它不复用 `engines/` 抽象，而是直接实现为一个独立的 `Module`。

```
dimos/simulation/unity/
├── module.py       # UnityBridgeModule(Module) + UnityBridgeConfig(ModuleConfig)
└── blueprint.py    # 运行入口蓝图
```

核心：

- `UnityBridgeModule(Module)`（`unity/module.py`）—— 在 DimOS 侧作为标准 Module 运行；对外通过 [ROS-TCP-Endpoint](https://github.com/Unity-Technologies/ROS-TCP-Endpoint) 的**二进制协议**与运行在另一进程/另一主机上的 Unity 仿真器通信，解析 Unity 发来的传感器帧并把指令回写。
- `UnityBridgeConfig(ModuleConfig)`（同文件）—— 桥接配置（host / port / topic 映射等）。
- `unity/blueprint.py` —— autoconnect 装配的可运行蓝图，作为 VLA Challenge 基线入口。

**关键点**：**DimOS 侧无 ROS 依赖**。`ROS-TCP-Endpoint` 只是一套 TCP 二进制协议格式——它允许 Unity 端装 ROS package 来通信，但 DimOS 侧只需要按该协议编解码字节流，**不需要运行 ROS master、不需要 rosdep、不需要 `source setup.bash`**。这与 `unitree/rosnav.py` 真正依赖 ROS 2 的情况完全不同。

**与其他四后端对比**：MuJoCo 主打**物理真实**（接触力、共享内存时序）；Genesis 主打 GPU 加速**高吞吐**原型；Isaac Sim 主打 **GPU 并行 + USD 渲染**训练；Unity 主打 **VLA Challenge benchmark 基线**（外部仿真器，TCP 桥接，场景与物理都托付给 Unity 工程）；`engines/` 则是"后端抽象层 + registry"，目前只包机械臂 Module 使用。

---

## 7. 蓝图示例

`dimos/robot/all_blueprints.py` 由 `pytest dimos/robot/test_all_blueprints_generation.py` **自动生成**，不可手动编辑。以下按平台列出当前已注册的代表性蓝图（完整列表以该文件为准）：

### 宇树系列

| 蓝图名称 | 功能描述 |
|---|---|
| `unitree-go2-basic` | Go2 最小连接 + 键盘遥控 |
| `unitree-go2-agentic` | Go2 + LLM 代理 + 技能调用 |
| `unitree-go2-agentic-mcp` | Go2 + 代理 + MCP 服务器（唯一支持 `dimos mcp` 的蓝图）|
| `unitree-go2-temporal-memory` | Go2 + 时序记忆代理 |
| `unitree-go2-fleet` | Go2 多机群控 |
| `unitree-go2-ros` | Go2 + ROS 2 导航 |
| `unitree-go2-spatial` | Go2 + 空间感知（占据地图）|
| `unitree-g1-basic` | G1 最小连接 |
| `unitree-g1-basic-sim` | G1 MuJoCo 仿真（基础） |
| `unitree-g1-agentic` | G1 + LLM 代理（真实硬件） |
| `unitree-g1-agentic-sim` | G1 + LLM 代理（MuJoCo 仿真） |
| `unitree-g1-full` | G1 全功能（感知 + 代理 + 导航） |
| `unitree-g1-detection` | G1 + 目标检测感知 |
| `unitree-g1-joystick` | G1 + 手柄遥控 |

### 机械臂系列

| 蓝图名称 | 功能描述 |
|---|---|
| `xarm-perception` | xArm + 视觉感知管线 |
| `xarm-perception-agent` | xArm + 感知 + LLM 代理 |
| `xarm7-planner-coordinator-agent` | xArm7 规划器 + 协调器 + 代理 |
| `coordinator-dual-xarm` | 双 xArm 协同控制 |
| `coordinator-piper-xarm` | Piper + xArm 异构双臂协同 |
| `xarm7-trajectory-sim` | xArm7 MuJoCo 轨迹仿真（`sim_blueprints.py`） |

### 无人机系列

| 蓝图名称 | 功能描述 |
|---|---|
| `drone-basic` | MAVLink 连接 + 视频流 |
| `drone-agentic` | MAVLink + LLM 代理技能 |

### 传感器独立蓝图

| 蓝图名称 | 功能描述 |
|---|---|
| `mid360` | Livox Mid360 原始点云 |
| `mid360-fastlio` | Mid360 + Fast-LIO2 里程计 |
| `mid360-fastlio-voxels` | Mid360 + Fast-LIO2 + 体素地图 |

---

## 8. 添加新平台 / 末端 / 传感器

以下是每类集成任务的推荐路径，附具体文件路径示例。

### 8.1 添加新移动平台（四足 / 轮式 / 无人机）

1. **创建平台目录**

   ```
   dimos/robot/<platform>/
   ├── __init__.py
   ├── connection.py   # 实现 TwistBaseAdapter 协议
   ├── blueprints/
   │   ├── basic/
   │   └── agentic/
   └── type/           # 若有私有消息格式，放此处
   ```

2. **实现连接 Module**：继承 `dimos.core.module.Module`，对外暴露与平台 SDK 对应的 Observable 流（至少 `Twist` 输入、`Odometry` 输出）。

3. **遵守 `TwistBaseAdapter` 协议**（`dimos/hardware/drive_trains/spec.py`）：实现 `connect()`、速度控制等必要方法，用 `isinstance(obj, TwistBaseAdapter)` 验证。

4. **编写蓝图**：从 `basic` 开始，仅含连接 Module + 键盘遥控。

5. **注册蓝图**：在蓝图 Python 文件中确保蓝图对象在模块顶层可导入，然后运行：

   ```bash
   uv run pytest dimos/robot/test_all_blueprints_generation.py
   ```

   这会自动重新生成 `dimos/robot/all_blueprints.py`（**请勿手动编辑该文件**）。

6. **添加系统提示**（若需要代理）：参考 `dimos/robot/unitree/g1/system_prompt.py` 创建 `<platform>_SYSTEM_PROMPT`，列出平台专属技能。

7. **添加测试**：在 `dimos/robot/<platform>/` 下或 `dimos/robot/unitree/testing/` 模式下添加 `test_connection.py`。

### 8.2 添加新机械臂

1. **驱动层**：在 `dimos/hardware/manipulators/<armname>/` 下实现 `ManipulatorAdapter` 协议（`dimos/hardware/manipulators/spec.py`）。

2. **注册驱动**：在 `dimos/hardware/manipulators/registry.py` 中添加驱动名称 → 类的映射条目。

3. **Robot 层**：在 `dimos/robot/manipulators/<armname>/blueprints.py` 下添加蓝图（参考 `xarm/blueprints.py`）。

4. **仿真**（可选）：如需仿真，准备 MJCF XML（`.xml`）并在 `dimos/simulation/sim_blueprints.py` 中通过 `simulation(engine="mujoco", config_path=...)` 创建仿真蓝图。

5. **注册蓝图**：运行 `pytest dimos/robot/test_all_blueprints_generation.py` 重新生成 `all_blueprints.py`。

### 8.3 添加新末端执行器

1. **扩展 EndEffector 类型**：在 `dimos/hardware/end_effectors/end_effector.py` 中添加枚举值，或继承 `EndEffector` 创建专有子类。

2. **注册**：在 `dimos/hardware/end_effectors/registry.py` 中注册类型映射。

3. **添加 Mock**：在 `dimos/hardware/end_effectors/mock/` 下提供测试桩，确保单元测试无需真实硬件。

### 8.4 添加新传感器

**摄像头**：

```
dimos/hardware/sensors/camera/<vendor>/
├── __init__.py
├── module.py       # 继承 CameraModule 基类
└── driver.py       # SDK 封装
```

在 `dimos/hardware/sensors/camera/spec.py` 中确认接口，确保输出类型为 `Image`（`dimos.msgs.sensor_msgs`）。

**激光雷达**：

```
dimos/hardware/sensors/lidar/<vendor>/
├── __init__.py
├── <vendor>_blueprints.py  # 参考 livox_blueprints.py
└── driver.py
```

输出类型应为 `PointCloud2`。若需 SLAM，参考 `fastlio2/` 模式接入 Fast-LIO2 进程。

### 8.5 添加新仿真后端

1. 在 `dimos/simulation/<backend>/` 下创建 `simulator.py`，继承 `dimos/simulation/base/simulator_base.py` 中的 `SimulatorBase`。

2. 若需引擎层接入，在 `dimos/simulation/engines/` 下创建 `<backend>_engine.py`，继承 `SimulationEngine`，实现 `step()`、`reset()`、`close()`。

3. 在 `dimos/simulation/manipulators/sim_module.py` 的 `simulation()` 工厂中注册新后端名称，以便用 `engine="<backend>"` 字符串选择。

---

> **成熟度提示**
>
> `dimos/robot/unitree/modular/` 目前**仅有 `detect.py` 一个文件**，git 历史显示整个目录只有两次提交（最近一次：`ed35bd7 feat(dask): remove dask`）。它是内部探针代码，**尚未投入生产**，不代表通用的模块化机器人集成架构。请勿将其作为新平台集成的参考模板。
>
> 对比参考：Go2 蓝图目录下有 `agentic/`、`basic/`、`smart/` 三个子层共十余个蓝图，G1 同样具备完整的四层蓝图体系，是成熟平台集成的最佳参考范例。

---

## 扩展阅读

- 总览：[README](/docs/architecture/README.md)
- Module / Blueprint / GlobalConfig 架构：[`docs/usage/modules.md`](/docs/usage/modules.md)
- 代理系统与 `@skill`：[`docs/architecture/agent-stack.md`](/docs/architecture/agent-stack.md#L3)
- 运行时模型与传输层：[`docs/architecture/runtime-model.md`](/docs/architecture/runtime-model.md)
- 平台配置：[`docs/platforms/`](../platforms/)
