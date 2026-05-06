# 专题：运行时模型（Runtime Model）

> 与 [README § 3](README.md#-3-运行时模型) 配套深入：进程/Stream/Transport/Protocol/Config/日志的实现层细节。

## 目录

- [1. 进程模型](#1-进程模型)
- [2. Stream 内部](#2-stream-内部)
- [3. 9 个 Transport 详细对比](#3-9-个-transport-详细对比)
- [4. Protocol 分层](#4-protocol-分层)
- [5. GlobalConfig cascade](#5-globalconfig-cascade)
- [6. Daemon 与 RunRegistry](#6-daemon-与-runregistry)
- [7. 日志结构](#7-日志结构)

<!-- TODO: Task 13 -->

---

## 扩展阅读

- 总览：[README](README.md)
- CLI：[`docs/usage/cli.md`](../usage/cli.md)
- 配置：[`docs/usage/configuration.md`](../usage/configuration.md)
