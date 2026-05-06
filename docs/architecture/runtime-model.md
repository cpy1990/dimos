# 专题：运行时模型（Runtime Model）

> 与 [README § 3](README.md#-3-运行时模型) 配套深入：进程/Stream/Transport/Protocol/Config/日志的实现层细节。
> 目标读者：正在调试 stream 不通、transport 选型、daemon 停不下来、日志找不到等问题的工程师。

## 目录

- [1. 进程模型](#1-进程模型)
- [2. Stream 内部](#2-stream-内部)
- [3. 9 个 Transport 详细对比](#3-9-个-transport-详细对比)
- [4. Protocol 分层](#4-protocol-分层)
- [5. GlobalConfig cascade](#5-globalconfig-cascade)
- [6. Daemon 与 RunRegistry](#6-daemon-与-runregistry)
- [7. 日志结构](#7-日志结构)

---

## 1. 进程模型

### 1.1 三层类关系

DimOS 的进程层由三个类协作完成——`ModuleCoordinator`、`WorkerManager`、`Worker`——职责严格分层：

```
ModuleCoordinator          ← 业务编排层（蓝图构建、模块生命周期）
  └─ WorkerManager         ← 进程池层（n 个 Worker 的分配与管理）
       └─ Worker × n       ← 单个工作进程（通过 Pipe 接收指令，托管多个 Module）
```

**ModuleCoordinator**（`dimos/core/module_coordinator.py`）是蓝图（Blueprint）调用的直接对象。它持有一个 `WorkerManager` 实例，并维护一张 `{Module 类 → ModuleProxy}` 字典，记录已部署的全部模块。核心方法 `deploy` / `deploy_parallel` 将模块类路由到合适的 Worker，并将返回的 `RPCClient` 包装为 `ModuleProxy` 对外暴露。蓝图 `build()` 完成后，调用 `health_check()` 验证所有 Worker 的 PID 仍然存活；`stop()` 则以反序优雅地停止每个模块，最后关闭整个进程池。

**WorkerManager**（`dimos/core/worker_manager.py`）管理固定数量的 `Worker`，默认 `n_workers=2`。`start()` 一次性启动全部 Worker 进程；`deploy()` / `deploy_parallel()` 使用最小负载（min by `module_count`）策略选择目标 Worker。`deploy_parallel` 先顺序分配（保证计数准确），再用 `ThreadPoolExecutor` 并发发送部署请求，显著缩短多模块蓝图的启动时间。

**Worker**（`dimos/core/worker.py`）每个实例对应一个独立的子进程。进程通信通道是 `multiprocessing.Pipe`（一对 `Connection` 对象）。Worker 内部维护 `{module_id → Actor}` 字典；每次收到 `deploy_module` 消息就实例化对应的 Module 类，并回传成功响应；后续 RPC 调用（`call_method`、`getattr`、`set_ref`）通过同一条 Pipe 串行传递，由 `threading.Lock` 保证请求-响应的一对一匹配。

### 1.2 forkserver vs fork

DimOS 固定使用 `forkserver` 上下文（`multiprocessing.get_context("forkserver")`），而非默认的 `fork`：

```python
# dimos/core/worker.py
_forkserver_ctx = multiprocessing.get_context("forkserver")
```

选择 `forkserver` 的原因：

| 维度 | `fork` | `forkserver` |
|---|---|---|
| CUDA 上下文 | 继承父进程 CUDA context，极易损坏 | 全新进程，不继承 GPU 状态 |
| 多线程安全 | fork 后子进程只有一个线程，持有的锁可能永久死锁 | 子进程从干净状态启动 |
| 文件描述符泄露 | 父进程所有 fd 均继承 | 只传递显式 Pipe |
| 启动速度 | 最快（写时复制） | 稍慢（需要序列化 module_class 并通过 Pickle 传送） |
| 适用场景 | 纯 Python、无 GPU、无复杂线程 | GPU / ROS / 多线程框架（DimOS 的选择） |

代价是 `forkserver` 要求传递给子进程的所有对象必须可 Pickle，因此 `Module` 类不能携带不可序列化的属性（如 `asyncio.Event`、锁等）——这也是 `Module.__getstate__` 手动剥离 `_disposables`、`_loop`、`_rpc` 等属性的原因。

### 1.3 Actor / MethodCallProxy 代理模式

`Actor` 是父进程持有的远程代理对象（Proxy）。它本身不包含任何 Module 状态，所有属性访问和方法调用都被 `__getattr__` 截获，通过 `Pipe.send()/recv()` 路由到子进程执行，再把结果同步返回。`MethodCallProxy` 是一个更轻量的包装，用于 `RemoteOut/RemoteIn` 的 `set_transport` 调用，让返回值符合 `ActorFuture.result()` 接口。

### 1.4 ModuleCoordinator 启动序列

```
blueprint.build(coordinator)
  ├─ coordinator.start()          → WorkerManager.start() → 启动 n 个 Worker 进程
  ├─ coordinator.deploy_parallel(module_specs)
  │    ├─ 为每个模块选 Worker（min module_count）
  │    └─ ThreadPoolExecutor → Worker.deploy_module() × n（并发）
  ├─ _connect_streams()           → 为每个 In/Out 设置 Transport
  ├─ coordinator.start_all_modules()
  │    └─ ThreadPoolExecutor → module.start() × n（并发）
  └─ coordinator.health_check()   → 检查所有 Worker PID 存活
```

---

## 2. Stream 内部

### 2.1 Stream 的四种状态

```python
class State(enum.Enum):
    UNBOUND   = "unbound"    # 描述符已定义，尚未绑定到 owner
    READY     = "ready"      # 已绑定 owner，等待 Transport
    CONNECTED = "connected"  # 已连接（In 连到 Out）
    FLOWING   = "flowing"    # 运行时：已观测到数据
```

模块类的类型注解（`Out[Image]`、`In[Twist]`）在 `Module.__init__` 中被 `get_type_hints()` 扫描，自动实例化为 `Out(inner_type, name, self)` / `In(inner_type, name, self)`。这一步发生在模块被 Worker 实例化时，因此 Stream 对象始终和 Module 实例同生命周期。

### 2.2 autoconnect 的 (name, type) 匹配算法

`autoconnect(*blueprints)` 是 DimOS stream 连接的核心函数（`dimos/core/blueprints.py`）。其匹配算法分四步：

**Step 1 — 收集所有 stream 的 (remapped_name, type)**

遍历所有 blueprint 声明的 `streams`（即模块类型注解中的 `In`/`Out`），对每个连接先查询 `remapping_map` 得到最终名字，然后以 `(remapped_name, stream_type)` 为键分组：

```python
streams[(remapped_name, conn.type)].append((blueprint.module, original_name))
```

**Step 2 — 名字唯一性检查**

若同一个名字对应多种不同类型，则抛出 `ValueError`，告知冲突的模块和类型。名字重复但类型相同是合法的——这正是"发布-订阅"的语义（多个 In/Out 共享同一个 Transport Topic）。

**Step 3 — 选择 Transport**

对每组 `(name, type)` 调用 `_get_transport_for(name, stream_type)`：
- 若 `stream_type` 带有 `lcm_encode` 方法（LCM 结构化消息），选 `LCMTransport`；
- 否则（任意 Python 对象），选 `pLCMTransport`（Pickle over LCM）；
- 若名字在整个图中唯一，topic 用 `/{name}`；否则用 `/{short_id()}` 避免冲突。

**Step 4 — 批量注入 Transport**

遍历该组所有 `(module, original_name)` 对，通过 `ModuleProxy.set_transport(original_name, transport)` 将 Transport 对象注入到 Worker 进程中的 Module 实例，完成 Stream 连接。

### 2.3 Remapping 解析

Remapping 用于两种场景：

1. **Stream 改名**：`autoconnect(...).remappings([(ModuleA, "video_out", "camera_feed")])`，将 `ModuleA.video_out` 的发布 topic 改名为 `camera_feed`，使其与订阅同名 stream 的其他模块匹配。

2. **模块引用替换**：`autoconnect(...).remappings([(ModuleA, "nav_ref", ConcreteNavModule)])`，将 `ModuleA` 对导航模块的引用明确指定为 `ConcreteNavModule`，用于消歧（多个模块满足同一 Spec 时）。

Remapping 存储在 `Blueprint.remapping_map`（`MappingProxyType`，不可变），多个蓝图 merge 时通过 `dict.update` 合并。**先声明的 remapping 会被后声明的覆盖**，这是调试多蓝图连接问题时需要注意的行为。

### 2.4 ReactiveX 背压

`In.subscribe()` 返回的 Observable 默认经过 `backpressure()` 包装。背压策略是"丢弃过旧的消息，只保留最新一帧"（`ops.sample` 语义），防止高频传感器数据（摄像头 30fps）堵塞慢速消费者。需要原始不丢帧流时，使用 `In.pure_observable()`。

---

## 3. 9 个 Transport 详细对比

以下表格是 README §2 简表的扩展版，增加了消息大小限制、跨主机能力、零拷贝、典型适用场景和已知约束列。

| Transport | 底层实现 | 消息大小 | 频率上限 | 跨语言 | 跨主机 | 零拷贝 | 典型用途 | 主要约束 |
|---|---|---|---|---|---|---|---|---|
| **LCMTransport** | LCM UDP 组播 + LCM 结构化消息 | ~65 KB（UDP 限制） | ~1 kHz | 是（C/C++/Python/Java） | 是（同子网） | 否 | 与 C++ 节点互通；低延迟传感器数据 | 消息类型须有 `lcm_encode`；组播需路由支持 |
| **pLCMTransport** | LCM UDP 组播 + Pickle | ~65 KB | ~1 kHz | 否（Python only） | 是（同子网） | 否 | 任意 Python 对象跨进程/主机；快速原型 | Pickle 不安全；大对象效率低 |
| **SHMTransport** | 共享内存 + LCM 信令（字节帧） | 默认 3.6 MB（可配置） | 硬件限制 | 否 | 否（本机） | 是（同进程可用 mmap） | 高分辨率图像、点云等大帧数据 | 仅限同机；需手动配置 `default_capacity` |
| **pSHMTransport** | 共享内存 + Pickle | 同上 | 硬件限制 | 否 | 否（本机） | 否（Pickle 拷贝） | 大型 Python 对象本机共享 | Pickle 序列化开销；仅限同机 |
| **JpegLcmTransport** | `JpegLCM`（JPEG 压缩 + LCM） | 网络带宽决定 | ~30 fps | 有限（JPEG 字节流） | 是 | 否 | 跨网络传输摄像头图像 | 有损压缩（quality 参数）；解码延迟 |
| **JpegShmTransport** | `JpegSharedMemory`（JPEG + SHM） | 同 SHM | 硬件限制 | 否 | 否（本机） | 部分（SHM 读） | 本机 JPEG 压缩传图（节省带宽） | 有损压缩；仅限同机 |
| **ROSTransport** | `rclpy`（ROS2 DDS 中间件） | ROS QoS 决定 | ROS 节点频率 | 是（全 ROS 生态） | 是（ROS 域内） | 否 | 与 ROS2 机器人生态系统互通 | 需安装 ROS2；消息类型须为 DimosMsg |
| **DDSTransport** | CycloneDDS（条件编译） | DDS QoS 决定 | DDS 硬件极限 | 是（DDS 标准） | 是（DDS 域内） | 部分（DDS loan API） | 工业/车规 DDS 部署；多机器人系统 | 仅在 `DDS_AVAILABLE`（`cyclonedds` 可导入）时编译 |
| **ZenohTransport** | Zenoh 协议（占位符） | 待定 | 待定 | 是（Zenoh 生态） | 是 | 待定 | 未来：边缘-云协同通信 | 当前实现为空类体（`...`），尚不可用 |

**关于 DDSTransport 条件编译**：`transport.py` 顶部尝试 `import cyclonedds`，若失败则 `DDS_AVAILABLE = False`，`DDSTransport` 类的整个定义块被跳过。如果在未安装 cyclonedds 的环境中引用 `DDSTransport`，将得到 `NameError`。安装方式：`uv sync --all-extras`（默认不包含 `dds` extra）或单独 `pip install cyclonedds`。

**JpegLcmTransport 与 JpegShmTransport** 均继承自对应的基类（`LCMTransport` / `PubSubTransport`），区别仅在于底层 pub/sub 实现替换为 JPEG 编码版本（`JpegLCM` / `JpegSharedMemory`），对上层模块完全透明。quality 参数（默认 75）在 `JpegShmTransport.__init__` 中暴露，可通过 Transport 构造参数调整。

---

## 4. Protocol 分层

`dimos/protocol/` 按通信关注点划分为五个子包，每个子包有独立的 `spec.py`（接口定义）和若干 `impl/` 实现。

### 4.1 `protocol/encode`

编码/解码工具层。`encoders.py` 定义 mixin：`LCMEncoderMixin`（将 Python 对象序列化为 LCM 字节帧）、`PickleEncoderMixin`（Pickle 序列化）、`JpegEncoderMixin`（PIL/numpy 图像→JPEG 字节）。这些 mixin 被 `pubsub/impl/` 中的各种 PubSub 实现组合使用——例如 `LCMPubSubBase` 混入 `LCMEncoderMixin`，`PickleLCM` 混入 `PickleEncoderMixin`。`encode/spec.py` 定义抽象 Encoder 接口，供需要替换编码策略的高级用法使用。`patterns.py` 实现 Glob 模式订阅（`topic/*`），允许消费者订阅整个 topic 命名空间。

### 4.2 `protocol/pubsub`

发布-订阅层。`spec.py` 定义 `PubSub[TopicT, MsgT]` 协议（抽象基类），规范 `publish(topic, msg)`、`subscribe(topic, callback) → unsubscribe_fn` 接口，以及 `sub()` 上下文管理器、`aiter()` 异步迭代器等语法糖。`impl/` 目录包含六种实现：

- `lcmpubsub.py`：LCM over UDP，支持结构化消息和 Pickle
- `shmpubsub.py`：共享内存，支持字节帧和 Pickle，通过 `CpuShmChannel` 实现环形缓冲
- `jpeg_shm.py`：JPEG 压缩版共享内存
- `ddspubsub.py`：CycloneDDS，仅在 `cyclonedds` 可导入时激活
- `rospubsub.py`：`rclpy` 包装，含 `ROS_AVAILABLE` 守卫
- `redispubsub.py`：Redis pub/sub（用于测试和远程调试场景）

`bridge.py` 提供在不同 PubSub 实现之间转发消息的桥接器，可将 LCM topic 实时转发到 Redis。

### 4.3 `protocol/rpc`

远程过程调用层。`spec.py` 定义 `RPCSpec`（服务端）和 `RPCClient`（客户端）两个 Protocol 类，规范 `call(name, args, cb)`、`call_sync(name, args)`、`call_nowait(name, args)` 三种调用语义（异步回调 / 同步阻塞 / fire-and-forget）。默认实现 `pubsubrpc.py` 将 RPC 请求/响应编码为 LCM 消息，通过 `pLCMTransport` 发送（request topic = `/rpc/{module_id}/{method}`，response topic 含调用者 UUID 防止串扰）。`redisrpc.py` 是备用实现，用于测试时不依赖 LCM 环境。`rpc_utils.py` 处理跨进程的异常序列化（`serialize_exception` / `deserialize_exception`），确保 Worker 端抛出的异常能以友好形式传回父进程。

### 4.4 `protocol/service`

服务配置层。`spec.py` 定义 `Configurable[ConfigT]` 泛型基类——持有一个 `default_config: type[ConfigT]`，`__init__` 时以 kwargs 实例化配置对象。`Service` 继承 `Configurable`，提供 `start()`/`stop()` 生命周期钩子。LCM 服务实现 `lcmservice.py` 封装了底层 LCM socket 的配置（组播地址、TTL、日志通道过滤等）；DDS 服务实现 `ddsservice.py` 封装 CycloneDDS 域配置。`system_configurator/` 子目录实现开机检查（如 LCM 组播路由是否就绪），在蓝图 build 时通过 `_run_configurators()` 自动执行，失败则提示用户并退出。

### 4.5 `protocol/tf`

坐标变换层（Transform Frame）。`tf.py` 定义 `TFSpec`（抽象接口）：`publish(*transforms)`、`publish_static(*transforms)`、`get(parent, child, time_point)` 等方法，以及 `TFConfig`（`buffer_size: float = 10.0` 秒、`rate_limit: float = 10.0` Hz）。默认实现 `LCMTF`（`tflcmcpp.py`）通过 LCM 与 C++ tf2 后端通信，复用 ROS2 的 tf2 时间缓冲语义。`Module.tf` 属性是懒加载的——首次访问时才实例化 `LCMTF()`，避免不使用坐标变换的模块引入不必要的 LCM 连接。

---

## 5. GlobalConfig cascade

`GlobalConfig` 继承自 `pydantic_settings.BaseSettings`，配置值按以下优先级**从低到高**级联覆盖（cascade）：

```
层级  来源                        示例
 1    代码默认值（字段默认）       n_workers=2, viewer="rerun"
 2    default.env 文件             ROBOT_IP=192.168.12.1
 3    .env 文件（工作目录）        SIMULATION=true
 4    环境变量（shell export）     N_WORKERS=4
 5    Blueprint 代码显式调用       cfg.update(n_workers=8)
```

**重要**：环境变量名与 `GlobalConfig` 字段名**完全一致，不加任何前缀**（如 `N_WORKERS`，而非 `DIMOS_N_WORKERS`）。这是 pydantic-settings 的默认行为（`env_prefix=""`），与许多框架使用项目前缀的惯例不同，初次使用容易混淆。

`model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8", extra="ignore")` 声明的 `extra="ignore"` 意味着 `.env` 中存在但 `GlobalConfig` 没有定义的键会被静默忽略，不会报错——这有助于同一 `.env` 文件同时服务多个工具，但也意味着拼写错误的键名不会有任何警告。

`Configurable[ConfigT]`（`protocol/service/spec.py`）是面向各 Service/Transport 实现的**局部配置**模式：每个 Service 类声明自己的 `ConfigT` 数据类作为 `default_config`，调用方通过 kwargs 覆盖局部参数，与全局 `GlobalConfig` 正交。例如 `LCMService(ttl=4)` 只影响该 LCM 实例的 TTL，不影响全局配置。

**蓝图如何注入 GlobalConfig**：`_deploy_all_modules()` 检查每个 Module 的 `__init__` 签名，若存在 `cfg` 参数则自动传入当前 `global_config` 实例，使模块可读取全局设置（如 `robot_ip`、`simulation`）而无需手动传参。

---

## 6. Daemon 与 RunRegistry

守护进程功能由三个模块协作实现，职责清晰分离：

### 6.1 `daemon.py`——进程守护实现

`daemonize(log_dir: Path)` 实现经典的 **Unix double-fork**：

```
CLI 主进程
  ├─ fork() → 中间进程（os._exit(0) 立即退出）
  │    └─ os.setsid()  → 创建新会话，脱离控制终端
  │         └─ fork() → 守护孙进程（真正的 daemon）
  │              ├─ stdin/stdout/stderr → /dev/null
  │              └─ 继续运行 blueprint
  └─ os._exit(0)
```

第一次 fork 后调用 `setsid()` 使进程成为新会话领导者，从而脱离终端控制。第二次 fork 确保 daemon 永远不能重新获取控制终端（因为它不是会话领导者）。stdio 重定向到 `/dev/null` 后，所有日志输出均通过 structlog 的 `RotatingFileHandler` 写入 `main.jsonl`，不会丢失。

`install_signal_handlers()` 为守护进程注册 `SIGTERM`/`SIGINT` 处理器：收到信号后先调用 `coordinator.stop()`（优雅停止所有模块），再调用 `entry.remove()`（从注册表删除记录），最后 `sys.exit(0)`。这确保即使 daemon 被 `kill` 终止，注册表也能得到清理（只要信号是 SIGTERM 而非 SIGKILL）。

### 6.2 `run_registry.py`——运行实例追踪

`RunEntry` 是注册表的基本单元（dataclass），字段包括：

| 字段 | 类型 | 含义 |
|---|---|---|
| `run_id` | `str` | `YYYYMMDD-HHMMSS-<blueprint>` 格式，人类可读 |
| `pid` | `int` | daemon 进程的 PID |
| `blueprint` | `str` | 蓝图名称 |
| `started_at` | `str` | ISO 8601 启动时间 |
| `log_dir` | `str` | 日志目录绝对路径 |
| `cli_args` | `list[str]` | 启动时传入的 CLI 参数 |
| `config_overrides` | `dict` | Blueprint 级别的配置覆盖 |
| `grpc_port` | `int` | gRPC 端口（默认 9877） |
| `original_argv` | `list[str]` | 完整的 `sys.argv` 快照 |

注册表文件保存路径遵循 XDG 规范：`~/.local/state/dimos/runs/<run-id>.json`（若设置了 `XDG_STATE_HOME` 则以它为根）。`list_runs(alive_only=True)` 遍历该目录，对每个 `.json` 文件调用 `is_pid_alive(pid)`（`os.kill(pid, 0)` 探针），自动清理僵尸记录。

### 6.3 `log_viewer.py`——日志查阅

`resolve_log_path(run_id="")` 实现三级回退：

1. 若提供了 `run_id`，在注册表中精确匹配；
2. 否则，返回当前存活进程的日志；
3. 再否则，返回最近一次已停止进程的日志。

`follow_log(path, stop)` 实现 `tail -f` 语义：seek 到文件末尾后在循环中 `readline()`，有新行则 yield，无新行则 `time.sleep(0.1)` 轮询，`stop()` 返回 True 时退出。`format_line(raw)` 将 JSONL 行解析为人类可读格式 `HH:MM:SS [lvl] logger  event  k=v …`，标准键（`timestamp`、`level`、`logger`、`event`、`func_name`、`lineno`）固定位置展示，其余 key-value 追加在末尾。

---

## 7. 日志结构

### 7.1 文件位置

DimOS 的 JSONL 日志路径取决于运行模式：

- **Daemon 模式**：`~/.local/state/dimos/logs/<run-id>/main.jsonl`
- **非 Daemon 前台模式**：`~/.local/state/dimos/logs/dimos_<YYYYMMDD_HHMMSS>_<pid>.jsonl`（fallback 路径）

日志文件使用 `RotatingFileHandler`，单文件大小上限约 10 MB，避免长时间运行导致磁盘耗尽。

### 7.2 JSONL 行结构

每行是一个独立的 JSON 对象，由 structlog 的 processor 链生成：

```jsonl
{"timestamp": "2026-05-06T14:23:01.456789Z", "level": "info", "logger": "dimos.core.module_coordinator", "event": "Worker pool started.", "n_workers": 2, "func_name": "start", "lineno": 93}
{"timestamp": "2026-05-06T14:23:02.100Z", "level": "info", "logger": "dimos.core.blueprints", "event": "Transport", "name": "video_out", "original_name": "video_out", "topic": "/video_out", "type": "dimos.msgs.image.Image", "module": "CameraModule", "transport": "pLCMTransport", "func_name": "_connect_streams", "lineno": 304}
{"timestamp": "2026-05-06T14:23:05.201Z", "level": "error", "logger": "dimos.core.worker", "event": "Worker error", "error": "RuntimeError: ...", "func_name": "deploy_module", "lineno": 220}
```

**固定字段**（`_STANDARD_KEYS`）：

| 字段 | 来源 | 说明 |
|---|---|---|
| `timestamp` | `structlog.processors.TimeStamper(fmt="iso")` | ISO 8601 UTC 时间戳 |
| `level` | `structlog.stdlib.add_log_level` | `debug`/`info`/`warning`/`error`/`critical` |
| `logger` | `structlog.stdlib.add_logger_name` | 调用 `setup_logger()` 的模块路径 |
| `event` | 日志调用第一个位置参数 | 人类可读的事件描述 |
| `func_name` | `CallsiteParameterAdder` | 调用日志的函数名 |
| `lineno` | `CallsiteParameterAdder` | 调用日志的行号 |

**附加字段**（key=value 键值对）：调用日志时传入的所有额外 kwargs，例如 `logger.info("Transport", name="video_out", module="CameraModule")`。这些字段可用于 `jq` 过滤、Loki 查询等结构化日志检索。

**异常记录**：`structlog.processors.format_exc_info` 将 `exc_info=True` 引用的异常格式化为字符串，写入 `exception` 字段（包含完整 traceback），不再是多行文本，便于结构化解析。

### 7.3 常用查询

```bash
# 查看最近 50 条日志（默认）
dimos log

# 实时追踪
dimos log -f

# 原始 JSONL（可管道给 jq）
dimos log --json | jq 'select(.level == "error")'

# 按模块过滤
dimos log --json | jq 'select(.logger | contains("camera"))'

# 查看指定 run 的日志
dimos log --run 20260506-142300-unitree-go2-agentic
```

---

## 扩展阅读

- 总览：[README](README.md)
- CLI：[`docs/usage/cli.md`](../usage/cli.md)
- 配置：[`docs/usage/configuration.md`](../usage/configuration.md)
- Agent 系统深入：[agent-stack.md](agent-stack.md)
- 数据流端到端：[data-flow.md](data-flow.md)
