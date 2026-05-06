# DimOS 系统架构（System Architecture）

> 给新工程师的全景文档：通读后无需跳转即可建立完整心智模型（约 1.5 小时）。
> 想深入某个细节再翻同目录 5 份专题：[runtime-model](runtime-model.md) /
> [agent-stack](agent-stack.md) / [robot-platforms](robot-platforms.md) /
> [subsystems](subsystems.md) / [data-flow](data-flow.md)。

## 目录

- [§ 0. DimOS 是什么](#-0-dimos-是什么)
- [§ 0.5. 仓库布局总览](#-05-仓库布局总览)
- [§ 1. 三个核心抽象](#-1-三个核心抽象)
- [§ 2. 通信骨架](#-2-通信骨架)
- [§ 3. 运行时模型](#-3-运行时模型)
- [§ 4. Agent 系统](#-4-agent-系统)
- [§ 5. 机器人平台层](#-5-机器人平台层)
- [§ 6. 能力子系统全景](#-6-能力子系统全景)
- [§ 7. 端到端数据流](#-7-端到端数据流)
- [§ 8. 怎么继续读 + 常见踩坑](#-8-怎么继续读--常见踩坑)

---

### § 0. DimOS 是什么

DimOS 是面向通用机器人的智能体操作系统（The Agentive Operating System for Physical Space）。

传统机器人软件往往是一块铁板：感知、规划、控制、IO 被硬编码耦合在同一进程或同一框架里。换一个平台，换一个传感器，整套代码往往需要大幅重写。更棘手的是，机器人的"决策"通常藏在状态机或硬编码规则里，很难被外部指令灵活驱动。

DimOS 的核心思路是把这块铁板拆成两个正交维度：

**Module（模块）** 负责所有与硬件相关的运行时能力——感知流水线（摄像头、激光雷达、语音）、空间记忆与建图、导航与规划、运动控制与电机驱动。每个 Module 通过类型化 Stream 发布和消费数据；Stream 底层可以是 LCM、ROS2、DDS 或内存管道，对上层透明。多个 Module 被 Blueprint（蓝图）编排成可直接运行的机器人软件栈，一行 `dimos run <blueprint>` 即可启动。

**Agent（智能体）** 负责高层决策。它订阅 Module 暴露的 Stream、调用 Module 注册的 Skill（技能函数），用自然语言驱动机器人行为。用户可以通过 CLI（`dimos agent-send`）、Web UI 或 MCP 客户端向 Agent 下指令，由 Agent 翻译成 Skill 调用；Teleop 则绕过 Agent，直接以人手操作驱动 Module。无需修改底层 Module，只需更换或扩展 Skill，机器人的行为边界就随之改变。

```mermaid
flowchart LR
  H[硬件] -->|sensors| M[Modules<br/>Perception · Planning · Control]
  M -->|Stream In/Out| A[Agent<br/>LLM 决策]
  A -->|Skill 调用| M
  U1[CLI / agent-send] --> A
  U2[Web UI] --> A
  U3[MCP Client] --> A
  U4[Teleop] --> M
  M -->|control| H
```

> **本文档与 AGENTS.md / CLAUDE.md 的分工**
>
> - `AGENTS.md` — quick-start cheat-sheet 与必踩坑清单（命令速查、blueprint 表、`@skill` 规则、预提交钩子）；**事实之源**，有疑问先看这里。
> - `CLAUDE.md` — AI 代理工作护栏；指向 `AGENTS.md`，附加少量 Claude 专用约束（不重复 AGENTS.md 内容）。
> - 本架构文档 — 系统全景与设计取舍。
>
> 三者交叉引用，不重复。如需修改某条规则，只在其"主权"文档里改，其余文档只做引用。

### § 0.5. 仓库布局总览

```text
dimos/                    # 仓库根
├── dimos/                # 主 Python 包（所有运行时代码）
├── bin/                  # CI/开发辅助脚本（如 bin/pytest-slow）
├── scripts/              # 安装与运维辅助脚本
├── data/                 # 示例 / replay 数据集
├── docker/               # 容器构建上下文
├── examples/             # 跑得起来的示例代码
├── assets/               # 图标、静态资源
├── docs/                 # 文档（本文件所在）
├── pyproject.toml        # 项目元数据 + 依赖声明
├── setup.py              # 兼容 setuptools 入口（保留兼容）
├── uv.lock               # uv 锁定文件
├── flake.nix             # Nix flake 定义
├── flake.lock            # Nix 锁定
├── default.env           # 默认环境变量
├── MANIFEST.in           # 打包清单
├── LICENSE               # 许可证
├── CLA.md                # 贡献者许可协议
├── README.md             # 仓库主 README
├── CLAUDE.md             # Claude Code 工作护栏
└── AGENTS.md             # AI agent onboarding（事实之源）
```

| 顶级目录/文件 | 用途 |
|---|---|
| `dimos/` | 主 Python 包：所有运行时代码 |
| `bin/` | shell 包装（如 `bin/pytest-slow` 跑全套测试） |
| `scripts/` | 安装与运维辅助脚本（目前主要是 `install.sh`） |
| `data/` | 示例数据、replay 数据集 |
| `docker/` | Docker 构建文件 |
| `examples/` | 跑得起来的示例 |
| `assets/` | 图标、静态资源 |
| `docs/` | 文档：本文件所在 |
| `pyproject.toml` | 项目元数据 + 依赖声明（setuptools 构建后端，uv 管理依赖） |
| `setup.py` | 旧式 setuptools 入口（保留兼容） |
| `uv.lock` | uv 锁定 |
| `flake.nix` / `flake.lock` | Nix 复现构建 |
| `default.env` | 默认环境变量 |
| `MANIFEST.in` | 打包清单 |
| `LICENSE` | 许可证 |
| `CLA.md` | 贡献者许可协议 |
| `README.md` | 仓库主 README |
| `CLAUDE.md` | Claude Code 工作护栏 |
| `AGENTS.md` | AI agent onboarding（事实之源） |

> `dimos/utils/` 是 20 来个横切工具模块（`logging_config` / `llm_utils` /
> `transform_utils` / `gpu_utils` / `threadpool` / `urdf` 等），被几乎所有
> 子系统依赖。本文档不展开；需要时直接读源码。

### § 1. 三个核心抽象

DimOS 的所有运行代码都围绕三个抽象组装：**Module / Blueprint / Skill**。读懂这三者，整个仓库的 80% 代码读起来就有定位感。

#### Module —— 自治子系统

Module 是 DimOS 最基础的运行单元。每个 Module 是一个继承自 `Module` 基类的 Python 类，代表一个独立的、可组合的能力模块——例如摄像头驱动、目标检测、导航规划或运动控制。Module 运行在 forkserver 工作进程中，与主进程隔离；多个 Module 之间默认按 worker 池调度（详见 §3）；进程间通过 LCM 消息总线或共享内存进行通信。

Module 的数据接口通过类型注解声明：`In[T]` 表示输入流，`Out[T]` 表示输出流，`T` 是实际消息类型（如 `Image`、`PoseStamped`、`Twist`）。声明即接口——Blueprint 在构建时会读取这些注解，自动按 `(名称, 类型)` 匹配并连接各 Module 的流。流的底层传输是可替换的（LCM / ROS2 / DDS / 内存管道），Module 本身不感知传输细节。

Module 还通过 `@rpc` 装饰器暴露可远程调用的方法。`@rpc` 方法是 Module 的过程调用面：其他 Module 或框架代码可以跨进程调用它们，保留原始返回类型。所有 Module 天然拥有两个基础 `@rpc` 方法：`start()` 负责初始化订阅和定时器，`stop()` 负责安全关闭和资源释放。

#### Blueprint —— 用 autoconnect 拼装

Blueprint 是一张描述"哪些 Module 共同构成这台机器人软件栈"的配置图。它本身是不可变数据结构（冻结 dataclass），因此可以安全地在多处复用和组合。

`autoconnect(*blueprints)` 是拼装的核心：它接受多个 Module 级或 Blueprint 级蓝图，去重合并后返回一个新的 `Blueprint`。构建阶段（`.build()`）会为每个 Module 启动 forkserver 工作进程，然后扫描所有模块的 `In[T]` / `Out[T]` 注解，凡 `(名称, 类型)` 完全匹配的流对，自动共享同一个传输层实例——这就是"自动连线"。

若两个 Module 恰好有同名但含义不同的流，可以用 `.remappings([(ModuleA, "old_name", "new_name"), ...])` 解决命名冲突，将某个 Module 的特定流改名后再参与匹配。Module 之间的 RPC 引用则通过 `Spec` Protocol 注解在构建期（`.build()`）绑定，找不到匹配实现会在 build 阶段直接报错，而不是在运行时静默失败。

#### Skill —— 智能体可调用的动作

Skill 是 Agent 能调用的物理动作函数，定义在继承自 `Module` 的 Skill 容器类里。`@skill` 装饰器（`dimos/agents/annotation.py`）同时设置 `__rpc__ = True` 和 `__skill__ = True`：前者让该方法可以被跨进程 RPC 调用，后者让框架在注册时自动把函数签名和文档字符串转换成 LLM 工具调用所需的 JSON schema。

`@skill` 与 `@rpc` 在职责上是两个不同的层面：`@rpc` 是 Module 内部 / Module 间的过程调用面，保留原始 Python 返回类型；`@skill` 是 LLM 向外暴露的工具接口，**必须返回 `str`**，因为语言模型只消费文本。因此，不要将两者叠加使用——`@skill` 已经蕴含了 `@rpc`。`@skill` 函数还有几条硬性规则：必须有文档字符串、所有参数必须类型注解、返回值必须是 `str`；完整规则见 §4，违反任意一条都会导致模块注册失败或 LLM schema 缺失。

#### 关系图（类图）

```mermaid
classDiagram
  class Module {
    +In~T~ inputs
    +Out~T~ outputs
    +@rpc methods
    +start() void
    +stop() void
  }
  class Blueprint {
    +blueprints: tuple
    +autoconnect()
    +remappings()
    +build() ModuleCoordinator
  }
  class SkillContainer {
    «Module subclass»
    +@skill methods
    +returns str
    +auto JSON schema
  }
  class Agent {
    +Spec Protocol refs
    +LLM tool calls
  }
  Blueprint o-- Module : 包含并编排
  Module <|-- SkillContainer : Skill 容器是\nModule 子类
  Agent ..> SkillContainer : 通过 Spec Protocol\n引用 @skill 方法
```

#### 最小可运行 Blueprint 片段

```python
from dimos.core.module import Module
from dimos.core.stream import In, Out
from dimos.core.core import rpc
from dimos.agents.annotation import skill
from dimos.core.blueprints import autoconnect
from dimos.msgs.geometry_msgs import Twist
class DriveModule(Module):    # ① Module：类型化流 + @rpc 调用面
    cmd_vel: In[Twist]
    velocity_feedback: Out[Twist]
    @rpc
    def start(self) -> None:
        super().start()
        self.cmd_vel.subscribe(self._process)  # 实现略
class Skills(Module):         # ② Skill：@skill 向 LLM 暴露工具 schema
    @skill
    def move(self, speed: float = 0.5) -> str:
        """让机器人前进。Args: speed: m/s。"""
        return f"以 {speed} m/s 前进"
my_robot = autoconnect(DriveModule.blueprint(), Skills.blueprint())  # ③ Blueprint
```

#### 为什么这样分层

**类型化流（In[T] / Out[T]）** 的设计目标是把"哪个流连接到哪里"从运行时错误提前到构建期。流的 `(名称, 类型)` 双重匹配确保一个 `In[Image]` 不会意外接到 `Out[Twist]`——类型不符时 `autoconnect` 在 `.build()` 阶段就会拒绝连线，而不是等到消息在运行时被错误解码才爆出隐晦的 deserialization 错误。这也使得传输层可以无缝替换（LCM / ROS2 / DDS / 内存管道），因为上层代码只依赖类型，不依赖具体传输 API。

**forkserver 进程模型**的选择优先于 `fork`，原因有三：`fork` 会把父进程中已初始化的 CUDA 上下文、GPU 内存映射和打开的 socket 直接复制到子进程，几乎必然导致 CUDA 上下文损坏或文件描述符竞争；forkserver 则让每个工作进程从干净状态启动，只按需初始化自己所需的资源。相较于 `spawn`，forkserver 只需预分叉一次，后续 worker 启动更快，且可以预先加载共享库。每个 Module 运行在独立进程里，任一模块崩溃不会把整个机器人软件栈拖垮。

**`@skill` 与 `@rpc` 分层**的设计来自两个面向不同消费者的接口需求。`@rpc` 方法的调用者是其他 Module 或框架代码，它们是 Python 进程，可以直接处理 `bool`、`PoseStamped` 等原生类型，保留类型信息对编译器检查和序列化都有意义。`@skill` 方法的调用者是语言模型，它只能理解 JSON 参数和纯文本返回值——强制 `str` 返回确保 Agent 总能收到有意义的人类可读反馈，而不是 `None` 或二进制对象。两层分离还带来一个额外好处：一个 Module 可以有许多内部 `@rpc` 方法供其他 Module 调用，但只向 LLM 暴露经过精心设计、有完整文档的 `@skill` 子集。

### § 2. 通信骨架

DimOS 模块间通信采用三层栈：上层 **Stream**（架构语义层） → 中层 **Transport**（IPC 选型层） → 底层 **Protocol**（编码与 pub/sub 实现层）。三层解耦让同一个 Module 不改业务代码就能在 LCM、共享内存、ROS、DDS、Redis 等多种后端之间自由切换。§1 里提到的 `LCM / ROS2 / DDS / memory` 选项，正是由这一层栈来承接的。

#### Stream — 类型化的数据通道

`Stream`、`In[T]`、`Out[T]` 定义于 `dimos/core/stream.py`，是 DimOS 通信体系最高抽象层。`Out[T]` 代表一个模块对外发布的数据出口，`In[T]` 代表一个模块对外声明的数据入口。两者均以泛型类型参数 `T` 标注所携带的数据类型——例如 `Out[Image]` 只能连接到 `In[Image]`，类型不符时 `autoconnect` 在 `.build()` 阶段便会拒绝建链，而不是等到运行时产生难以追踪的反序列化错误。

每个 `Stream` 实例持有一个名称字段（`name`）和一个类型字段（`type`），`autoconnect` 依据"名称匹配且类型兼容"双重条件在蓝图拼装阶段自动连线；若需手动控制，也可在 Blueprint 里显式指定连接关系。每个 `Stream` 还关联一个 `Transport`，负责把数据实际搬运到发布/订阅后端——上层代码不感知传输细节，只调用 `Out.publish(msg)` 或订阅 `In.subscribe(callback)`。

`§1` 的最小代码示例里已经出现了 `In[str]` / `Out[str]`；本章的重点是它们背后的传输机制。

#### Transport — 9 个类，主用 6 个

`dimos/core/transport.py` 定义了 **9 个**功能性 Transport 类（外加一个内部基类 `PubSubTransport` 不计入）：**主用的六个**具体传输类（LCM、pLCM、SHM、pSHM、ROS、DDS）、**两个** JPEG 变体（JpegLcm、JpegShm）、**一个**占位 `ZenohTransport`。其中 `DDSTransport` 仅在编译时检测到 DDS 库才会启用（`if DDS_AVAILABLE:` 分支），其余 8 个无条件加载。

**主用六个传输类**各有不同的适用场景：

- **`LCMTransport`**：基于 LCM（Lightweight Communications and Marshalling）协议。LCM 使用 UDP 多播，天然跨主机、跨语言，序列化格式紧凑，是最通用的消息总线选择。需要额外指定消息类型（`type` 参数），底层通过 `lcmpubsub.py` 的 `LCM` 实例完成发布/订阅。
- **`pLCMTransport`**：在 `LCMTransport` 基础上改用 Pickle 序列化（`PickleLCM`），无需手写 LCM 消息定义，适合快速原型或 Python 内部对象的跨进程传递；代价是牺牲了跨语言兼容性。
- **`SHMTransport`**：基于 POSIX 共享内存（`BytesSharedMemory`），数据无需经过网络栈或内核 socket，同机进程间近似零拷贝传输，适合传输大对象（如原始图像帧）；天然不跨主机。
- **`pSHMTransport`**：SHM 的 Pickle 变体（`PickleSharedMemory`），同样适合同机大对象，但可以直接传递任意 Python 对象而无需手动字节化。
- **`ROSTransport`**：通过 `dimos/protocol/pubsub/impl/rospubsub.py` 桥接至 ROS2 话题，固定泛型类型为 `DimosMsg`，专为与 ROS 生态互通而设计。只要机器人已安装 ROS2，即可与现有 ROS 节点双向收发消息。
- **`DDSTransport`**：基于 CycloneDDS，在文件末尾以 `if DDS_AVAILABLE:` 条件编译定义（已计入上述 9 个功能性传输类中）。DDS 支持丰富的 QoS 策略，是 Unitree 真机产线的首选传输；未安装 `cyclonedds` 时该类不存在，代码依然可运行。

**JPEG 视频变体**：`JpegLcmTransport` 继承自 `LCMTransport`；`JpegShmTransport` 直接继承自基类 `PubSubTransport`（不是 `SHMTransport`），但同样基于共享内存路径。两者内部分别替换底层 pub/sub 实现为 `JpegLCM` 与 `JpegSharedMemory`，在传输前对图像帧做 JPEG 压缩编码，目的是降低跨进程传输视频流时的带宽占用与 CPU 拷贝开销。两者专为图像流场景设计，普通非图像消息应使用基础变体。

**占位类**：`ZenohTransport` 仅声明为 `class ZenohTransport(PubSubTransport[T]): ...`，方法体为空，尚未实现，当前不可用。Zenoh 协议预留于未来的去中心化边缘计算场景。

| 名称 | 跨语言 | 跨主机 | 零拷贝 | 适用场景 | 备注 |
|---|---|---|---|---|---|
| `LCMTransport` | ✓ | ✓（UDP 多播） | ✗ | 通用消息总线 | 需声明消息类型 |
| `pLCMTransport` | ✗（仅 Python） | ✓ | ✗ | 快速原型 / Python 对象 | Pickle 序列化 |
| `SHMTransport` | ✗（同机） | ✗ | ✓ | 同机大对象（原始帧） | `BytesSharedMemory` |
| `pSHMTransport` | ✗（同机） | ✗ | ✓ | 同机 Python 对象 | `PickleSharedMemory` |
| `ROSTransport` | ✓ | ✓ | ✗ | 与 ROS2 生态互通 | 固定 `DimosMsg` 类型 |
| `DDSTransport` | ✓ | ✓ | ✗（QoS 可控） | 真机产线（Unitree） | 需安装 `cyclonedds` |
| `JpegLcmTransport` | 同 LCMTransport | 同 LCMTransport | 部分 | 视频帧流传输（LCM） | JPEG 压缩降低带宽 |
| `JpegShmTransport` | ✗（同机） | ✗ | 部分 | 视频帧流传输（SHM） | 继承自 PubSubTransport，非 SHMTransport |
| `ZenohTransport` | — | — | — | 未来边缘场景 | 占位，尚未实现 |

#### Protocol — 比 Transport 更底层的封装层

`dimos/protocol/` 是 Transport 之下的实际编码与通信实现层，分为五个子目录。Transport 类是面向 Stream 的高层抽象，`protocol/` 里的各 impl 文件才是真正与 LCM daemon、共享内存段、DDS participant、ROS rclpy 节点等系统资源打交道的代码。两层的职责边界是：Transport 负责"数据如何被 Module 的流订阅/发布"，Protocol 负责"数据如何被序列化并搬运到底层 IPC 通道"。

```text
dimos/protocol/
├── encode/        # 序列化/反序列化（msgpack 等格式工具）
├── pubsub/        # 发布订阅核心
│   └── impl/
│       ├── lcmpubsub.py        # LCM / PickleLCM / JpegLCM 实现
│       ├── shmpubsub.py        # BytesSharedMemory / PickleSharedMemory
│       ├── jpeg_shm.py         # JpegSharedMemory（视频帧 JPEG 压缩共享内存）
│       ├── ddspubsub.py        # CycloneDDS pub/sub
│       ├── rospubsub.py        # ROS2 rclpy 订阅/发布
│       ├── redispubsub.py      # Redis 发布订阅（跨机低延迟）
│       └── memory.py           # 进程内内存通道（单元测试 / 无 IPC 场景）
├── rpc/           # 远程过程调用
│   ├── pubsubrpc.py            # 基于 pubsub 的 RPC 封装
│   ├── redisrpc.py             # 基于 Redis 的 RPC 实现
│   ├── spec.py                 # Spec Protocol —— 类型安全 RPC 声明（推荐新代码使用）
│   └── rpc_utils.py            # 公共工具函数
├── service/       # 请求/响应模式（LCM service / DDS service）
└── tf/            # 坐标变换（ROS tf2 桥接）
```

值得注意的是 `rpc/spec.py` 中的 `Spec` Protocol 模式：这是 AGENTS.md 推荐的新式 RPC 接线方式，以 Python `Protocol` 类型声明依赖，在构建期即可捕获接口不匹配，而不是等到运行时才报错。旧式的 `rpc_calls: list[str]` 方式仍可工作，但静默失败风险更高，新代码应优先使用 `Spec`。

`protocol/pubsub/impl/redispubsub.py` 对应的 Transport 层目前没有专门的 `RedisTransport` 类——Redis pub/sub 是通过更高层的 RPC 机制（`redisrpc.py`）使用，而非直接作为 Stream Transport 暴露。`memory.py` 提供进程内纯内存通道，主要用于单元测试和不需要任何外部 IPC 守护进程的场景。

#### 三层关系图

```mermaid
flowchart TB
  S["Stream<br/>dimos/core/stream.py<br/>In[T] / Out[T] 类型化数据通道"] --> T
  T["Transport<br/>dimos/core/transport.py<br/>9 个具名类（+DDSTransport 条件编译）"] --> P
  P["Protocol<br/>dimos/protocol/<br/>encode / pubsub / rpc / service / tf"] --> IPC
  IPC["底层 IPC<br/>LCM · SHM · DDS · ROS2 · Redis · in-memory"]
```

#### 关键避坑（命名重叠 1）：两种 Stream

DimOS 里有**两个**以 `stream` 命名的模块，**名字相近、用途完全不同**：

| 路径 | 角色 | 内容 |
|---|---|---|
| `dimos/core/stream.py` | 架构骨架：模块间类型化数据通道 | `Stream` / `In[T]` / `Out[T]` / `Transport` 基类 |
| `dimos/stream/` | 视频/音频 data provider（外部数据源接入） | `rtsp_video_provider.py` / `ros_video_provider.py` / `frame_processor.py` / `video_operators.py` / `audio/` 等 |

`dimos/core/stream.py` 里的 `Stream` 是抽象的通信信道——任何模块都用它来声明"我需要哪种数据、我输出哪种数据"。`dimos/stream/` 里的文件是具体的数据采集模块，负责从 RTSP 摄像头、ROS 话题、麦克风等外部设备抓取原始帧或音频，再通过 `Out[T]` 注入到系统的数据流中。两者在名称上的重叠是历史遗留，读代码时务必按完整 import 路径区分：`from dimos.core.stream import In` 与 `from dimos.stream.rtsp_video_provider import ...` 是截然不同的两件事。

### § 3. 运行时模型

#### forkserver 进程模型

DimOS 使用 Python 的 `forkserver` 启动上下文（`multiprocessing.get_context("forkserver")`）来生成每个 worker 进程，而不是更常见的 `fork`。这一选择有明确的工程理由，在 `dimos/core/worker.py` 的注释里有直接说明：

```python
# Global forkserver context. Using `forkserver` instead of `fork` because it
# avoids CUDA context corruption issues.
```

`fork` 在机器人/GPU 工程场景下有多个不可忽视的失效模式。第一，**CUDA 上下文污染**：父进程一旦初始化了 CUDA（哪怕只是加载了 PyTorch 或 OpenCV），`fork` 之后子进程会继承一个半初始化的 CUDA 上下文副本，后续 CUDA 调用几乎必然崩溃或返回不可预测结果。第二，**GPU 内存映射**：CUDA 的 unified memory 和 device buffer 映射是在父进程地址空间中建立的，`fork` 把整张地址空间的虚拟映射关系复制给子进程，但底层驱动状态无法同步，造成映射悬空。第三，**文件描述符继承**：`fork` 会把父进程所有打开的 socket、管道、LCM fd 原封不动地复制到子进程，导致多个进程同时持有同一个 LCM 多播 socket，收发行为混乱。第四，**GIL 锁状态**：在多线程父进程中调用 `fork` 时，子进程继承了 GIL 的内部锁状态，如果 `fork` 时某个线程正持有 GIL，子进程在单线程模式下永远无法获得 GIL，会立刻死锁。

`forkserver` 的做法是：主进程启动时预先 fork 出一个干净的"forkserver"守护进程，每次需要新 worker 时向它发请求，由它从自己的干净状态 fork 出子进程，再将模块代码和初始化参数通过 pipe 传入。由于 forkserver 本身从未初始化过 CUDA 或打开过业务 socket，每个 worker 都从真正干净的状态启动。相比 `spawn`（完全重新执行 Python 解释器），`forkserver` 只需在进程启动时预分叉一次，后续 worker 启动速度更快，且可以复用已预加载的共享库。任何一个 Module 崩溃或卡住，最坏结果是那个 worker 进程死掉，不会波及其余 worker 或主守护进程。

#### WorkerManager 与 ModuleCoordinator 分工

`WorkerManager`（`dimos/core/worker_manager.py`）与 `ModuleCoordinator`（`dimos/core/module_coordinator.py`）是 DimOS 进程管理层的两个责任分明的角色，不能混淆。

**WorkerManager** 是 worker 进程池的所有者，只关心进程生命周期：启动时创建 `n_workers`（默认值 `2`）个 worker 进程，每个进程运行一个 `_worker_entrypoint` 事件循环等待请求；它持有 `Worker` 对象列表，每次 `deploy()` 调用时选出当前 `module_count` 最小的 worker（最小负载优先分配），将 Module 类和初始化参数序列化后通过 pipe 发给该 worker 进程；worker 侧反序列化后实例化 Module、启动其内部线程。`WorkerManager` 还通过 `deploy_parallel()` 支持多个 Module 的并发部署——先顺序预分配 worker（保证计数准确），再用 `ThreadPoolExecutor` 并发执行跨 pipe 的实际 deploy 请求。

**ModuleCoordinator** 是更高层的编排者，持有 `WorkerManager` 引用并在其上构建 stream 接线逻辑。它读取 Blueprint 中各 Module 的 `In[T]` / `Out[T]` 注解，将匹配的流对连接到同一 Transport 实例，确保消息能在不同 worker 中的 Module 之间流动；同时维护所有已部署 Module 的 `ModuleProxy` 句柄供 `health_check()` 和 `stop()` 使用。

两者分工的关键含义是：**当 `n_workers` 小于 Module 总数时，多个 Module 会共享同一个 worker 进程**。WorkerManager 按最小负载将它们塞到同一个 `_worker_loop`，这些 Module 在该进程内串行处理 pipe 请求（各自有独立线程驱动内部逻辑，但 pipe 通信是串行的）。这在小型部署或测试场景下节省系统资源，但若某 Module CPU 密集，会对同 worker 内其他 Module 的响应延迟产生影响。

#### 进程关系图

```mermaid
flowchart TB
  CLI["dimos run<br/>CLI 入口"] --> D["Daemon<br/>double-fork 常驻进程"]
  D --> WM["WorkerManager<br/>worker 进程池"]
  D --> MC["ModuleCoordinator<br/>stream 接线 + 模块编排"]
  WM --> W1["worker 1<br/>Module A · Module B"]
  WM --> W2["worker 2<br/>Module C"]
  WM --> W3["worker N<br/>Module …"]
  MC -.stream connect.-> W1
  MC -.stream connect.-> W2
  W1 <-->|"Stream LCM / SHM / DDS"| W2
  W2 <-->|"Stream"| W3
```

CLI 执行 `dimos run <blueprint>` 后，若带 `--daemon` 标志则调用 `daemon.py` 的 `daemonize()` 做 double-fork 脱离终端，否则在前台运行。进程成为 daemon 后通过 RunRegistry 注册自身（写入 `run-id.json`），然后构建 Blueprint、启动 WorkerManager 和 ModuleCoordinator，最后进入 signal 等待循环。`SIGTERM` / `SIGINT` 触发 `coordinator.stop()` 并从 RunRegistry 中注销条目。

#### GlobalConfig 配置级联

`GlobalConfig`（`dimos/core/global_config.py`）是一个继承自 `pydantic-settings` `BaseSettings` 的 dataclass 单例。pydantic-settings 的级联机制决定了每个字段的最终取值，从低到高依次被覆盖：

```text
代码内 dataclass 字段默认值（如 n_workers = 2，mcp_port = 9990）
  ↓ 被覆盖
.env 文件（通过 SettingsConfigDict(env_file=".env") 加载）
  ↓ 被覆盖
DIMOS_* 环境变量（pydantic-settings 自动按 DIMOS_<FIELD_NAME> 格式识别）
  ↓ 被覆盖
Blueprint 显式调用 global_config.update(**overrides) 注入的覆写值
  ↓ 被覆盖
CLI 标志（dimos run --n-workers 4 等，typer 动态回调写入 global_config）
```

五层级联中，**CLI 标志优先级最高**，可以在不修改任何文件的情况下临时调整任何配置项，方便调试。Blueprint 层的覆写（`update()` 调用）则允许特定蓝图固定某些参数（如专用 MCP 端口），不受环境变量干扰。`.env` 文件适合把机器人 IP、API key 等部署级参数版本化但不提交（`.gitignore` 排除），团队成员各自维护本地 `.env`。所有 `GlobalConfig` 字段均对应一个 `DIMOS_<FIELD_NAME>` 环境变量：例如 `DIMOS_ROBOT_IP`、`DIMOS_N_WORKERS`、`DIMOS_SIMULATION`，CI/容器环境通过注入环境变量即可完成配置，无需改代码。

每次 `GlobalConfig()` 实例化时（程序启动时仅做一次），pydantic-settings 按此顺序解析并合并所有来源。`global_config` 是模块级单例，整个运行期共享同一实例；`update()` 方法允许 Blueprint 在 build 阶段原地修改字段，后续代码感知同一对象上的最新值。

#### Daemon 与 RunRegistry

Daemon 与 RunRegistry 共同提供 DimOS 的"进程生命周期账本"，使 `dimos status`、`dimos log`、`dimos stop` 等命令能定位并操控正在运行的实例，即使这些实例已脱离终端后台运行。

**run-id** 是每次运行的唯一标识符，格式为 `{YYYYMMDD-HHMMSS}-{blueprint-name}`（由 `run_registry.py` 的 `generate_run_id()` 生成）。run-id 同时作为状态文件名和日志目录名，将注册表条目与日志文件关联起来：

```text
~/.local/state/dimos/
├── runs/<run-id>.json        # 运行注册表条目（pid、blueprint、grpc_port 等）
└── logs/<run-id>/main.jsonl  # 单次运行的结构化日志（structlog JSON Lines）
```

**`daemon.py`** 是 double-fork 逻辑与 signal handler 的所在处。`daemonize(log_dir)` 执行标准 Unix double-fork：第一次 fork 脱离前台进程组，`setsid()` 新建会话，第二次 fork 确保进程永不重新获取控制终端；stdin/stdout/stderr 全部重定向到 `/dev/null`，实际日志通过 structlog 的 `FileHandler` 写入 `main.jsonl`。`install_signal_handlers(entry, coordinator)` 注册 `SIGTERM` 和 `SIGINT` 的处理函数：收到信号时先调用 `coordinator.stop()` 优雅关闭所有 worker，再调用 `entry.remove()` 从 RunRegistry 注销自身。

**`run_registry.py`** 实现注册表的 CRUD 操作。`RunEntry` dataclass 包含 `run_id`、`pid`、`blueprint`、`started_at`、`log_dir`、`grpc_port` 等字段，序列化为 JSON 落盘到 `runs/` 目录。`list_runs(alive_only=True)` 遍历所有 `.json` 条目，对已死亡的进程（`os.kill(pid, 0)` 失败）自动清理过期条目，避免积累僵尸记录。`stop_entry()` 先发 `SIGTERM` 等待 5 秒，超时则升级到 `SIGKILL`，实现 `dimos stop` 命令的优雅停止逻辑。

**`log_viewer.py`** 是 `dimos log` 命令的后端。它按 run-id 在 RunRegistry 中定位对应的 `log_dir`，读取 `main.jsonl`（structlog 写入的 JSON Lines 格式），解析每行 JSON 记录，格式化为 `HH:MM:SS [lvl] logger  event  k=v …` 的可读输出并加 ANSI 颜色（error 红、warning 黄、debug 灰）。`dimos log -f` 模式使用 `follow_log()` 实现 `tail -f` 风格的实时追踪。

#### CLI 命令族

| 命令 | 作用 |
|---|---|
| `dimos run <blueprint>` | 启动一次蓝图运行（前台或 `--daemon`） |
| `dimos status` | 列出当前活动 run（run-id、PID、蓝图名、运行时长、日志路径） |
| `dimos log [-f]` | 查看日志（`-f` 实时追踪，`-n N` 指定行数） |
| `dimos stop` / `restart` | 优雅停止（SIGTERM → SIGKILL） / 停止后重启 |
| `dimos list` | 列出所有可运行 blueprint |
| `dimos show-config` | 输出当前生效的 GlobalConfig 值 |
| `dimos mcp` | MCP server 交互子命令（list-tools / call / status / modules） |
| `dimos agent-send "<text>"` | 通过 LCM 向正在运行的 agent 发送消息 |
| `dimos topic echo <topic>` / `topic send <topic> <expr>` | LCM topic 订阅调试 / 发布调试 |
| `dimos lcmspy` | LCM 流量嗅探工具 |
| `dimos agentspy` | Agent 消息嗅探工具 |
| `dimos humancli` | 人工介入 CLI（远程操控机器人动作） |
| `dimos top` | 类 top 实时进程监控（dtop） |
| `dimos rerun-bridge` | 启动独立的 Rerun 可视化桥接进程 |

完整参数与示例见 [`AGENTS.md`](../../AGENTS.md) CLI Reference 段。

#### 关键避坑（命名重叠 2）：agent-send 与 agent_send

`dimos agent-send`（CLI 命令，连字符形式）是 shell 层的入口，定义于 `dimos/robot/cli/dimos.py`：

```python
@main.command("agent-send")
def agent_send_cmd(message: str) -> None:
    ...
    text = _get_adapter().call_tool_text("agent_send", {"message": message})
```

`agent_send`（下划线形式）是 MCP 工具层的函数名，定义于 `dimos/agents/mcp/mcp_server.py`：

```python
def agent_send(self, message: str) -> str:
    ...
```

**两个名字、两层入口、同一动作**：CLI 命令用连字符（shell 习惯），MCP 工具用下划线（Python 函数命名规范）。CLI 内部调用 `call_tool_text("agent_send", ...)` 把消息转发给 MCP 层，再由 MCP 层路由到 agent。写文档时按上下文区分：描述 shell 命令写 `agent-send`，描述 MCP 工具调用写 `agent_send`。新工程师常见错误是在代码里写 `agent-send`（含连字符的字符串），或在 shell 里写 `dimos agent_send`（下划线）——两者均无法工作。

### § 4. Agent 系统

<!-- TODO: Task 6 -->

### § 5. 机器人平台层

<!-- TODO: Task 7 -->

### § 6. 能力子系统全景

<!-- TODO: Tasks 8-10 -->

### § 7. 端到端数据流

<!-- TODO: Task 11 -->

### § 8. 怎么继续读 + 常见踩坑

<!-- TODO: Task 11 -->

---

## 扩展阅读

- 仓库速查与必踩坑：[`AGENTS.md`](../../AGENTS.md)
- 使用教程：[`docs/usage/`](../usage/)
- 能力深度文档：[`docs/capabilities/`](../capabilities/)
- 平台配置：[`docs/platforms/`](../platforms/)
- 安装：[`docs/installation/`](../installation/)
- 开发与测试：[`docs/development/`](../development/)
