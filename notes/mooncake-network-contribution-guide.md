# Mooncake / KV Cache 项目网络通信方向贡献指南

> 面向负责底层网络高性能通信库的工程师，梳理要掌握的知识体系和实践路径。

## 一、先吃透 Mooncake 的架构和核心抽象

### 1. Transfer Engine 的 Segment + Batch 模型
- 重点文件：
  - `mooncake-transfer-engine/include/transfer_engine.h`
  - `mooncake-transfer-engine/include/transfer_engine_c.h`
- 核心概念：
  - `Segment`：内存/显存的一段连续区域
  - `BatchTransferRequest`：一次批量传输请求
  - `TransferOpcode`：read / write 等操作码
- 关键 API：`transfer()` / `submitTransfer()` / `getTransferStatus()` 异步批量传输接口
- 重要原则：**metadata server 只做 peer discovery，数据是 P2P 直连传输，不经过元数据服务中转**

### 2. 各 transport 实现
| Transport | 路径 | 学习建议 |
|-----------|------|---------|
| TCP | `mooncake-transfer-engine/src/transport/tcp_transport/` | 最简单，先看生命周期 |
| RDMA | `mooncake-transfer-engine/src/transport/rdma_transport/` | 核心，深入 wr/qp/cq、内存注册、GID 选择、重连 |
| NVLink / intra-node | `nvlink-allocator/`、`intranode_nvlink_transport` | 节点内 GPU 直连 |
| EFA / CXI / HCCL / UBShmem | 对应目录 | 按需了解，不用一开始就全看 |

### 3. Mooncake Store 的控制面与数据面分离
- 关键文件：
  - `mooncake-store/src/master_service.cpp`
  - `mooncake-store/src/master_client.cpp`
  - `mooncake-store/src/real_client.cpp`
- 理解要点：
  - Object → Segment 映射
  - 分配策略、副本、SSD 卸载
  - Client 与 Master 通过 RPC 通信
  - 实际 KV Cache 数据通过 Transfer Engine 在 client 之间 P2P 搬运

---

## 二、补 KV Cache 场景知识

### 1. 分离式推理（PD 分离）
- Prefill 节点产生 KV Cache，Decode 节点消费
- 流量特征：
  - 大对象传输
  - 一次性写、多次读
  - 延迟敏感
  - 突发批量 get
- 核心目标：低延迟 + 高吞吐，容忍网络抖动

### 2. KV Cache 的存储分层
```
GPU HBM → CPU DRAM → SSD/NVMe → 远端节点
```
- 各层带宽/延迟差异巨大
- 理解 Mooncake Store 的 eviction、promotion、offload 策略

### 3. 与上层框架集成
- vLLM、SGLang、LMDeploy 如何调用 Mooncake
- 参考：`docs/source/getting_started/examples/`
- 关键集成点：`MooncakeStoreConnector`、`HiCache`

---

## 三、硬技能清单

### C++ / 系统编程
- C++20 常用特性
- 多线程、无锁/低锁并发、内存对齐
- `mmap`、大页（hugepage）、DMA、内存注册（`ibv_reg_mr`）
- CMake、Ninja、动态库链接

### 网络协议和硬件
- RDMA verb 编程：`ibv_post_send/recv`、qp/cq、SRQ、RC/UD
- RoCE v2 / InfiniBand：GID、PFC/ECN、DSCP
- NVLink / GPUDirect RDMA
- 网络拓扑感知：NUMA、NIC-GPU affinity
- 可选：AWS EFA、CXL、华为 HCCL/UBShmem

### 调试和性能分析
- `perf`、`ib_write_bw`、`qperf`、`iperf`
- RDMA 相关：`rdma_cm`、mlx5 驱动日志
- GPU：`nccl-tests`、`nvidia-smi topo`
- 火焰图、bpftrace、eBPF

### 测试与交付
- GoogleTest、CI 流程（`.github/workflows/ci.yml`）
- 性能基准：`transfer_engine_bench`、`storage_benchmark_v1`
- 可复现的 regression test 设计

---

## 四、推荐阅读和实践顺序

### 第 1 周：把项目跑起来
```bash
bash dependencies.sh
mkdir build && cd build
cmake -G Ninja .. -DUSE_TCP=ON -DUSE_HTTP=ON -DBUILD_UNIT_TESTS=ON -DBUILD_EXAMPLES=ON
ninja
ctest --output-on-failure
```
- 目标：跑通 TCP 版本的单元测试和 benchmark

### 第 2–3 周：读核心代码
按顺序阅读：
1. `mooncake-transfer-engine/src/transfer_engine_impl.cpp`
2. `mooncake-transfer-engine/src/transport/tcp_transport/tcp_transport.cpp`
3. `mooncake-transfer-engine/src/transport/rdma_transport/rdma_context.cpp`
4. `mooncake-transfer-engine/src/transport/rdma_transport/rdma_endpoint.cpp`
5. `mooncake-store/src/real_client.cpp`
6. `mooncake-store/src/master_service.cpp`

### 第 4 周：动手小改造
- 给某个 transport 增加更详细的 metric/日志
- 优化一次 RDMA GID 选择逻辑
- 修复一个 flaky test
- 给 benchmark 增加延迟分布输出

---

## 五、独特贡献方向

| 方向 | 具体切入点 |
|------|-----------|
| **传输性能优化** | RDMA 批处理、QP 复用、零拷贝、自适应重连、拥塞控制 |
| **拓扑感知调度** | NIC-GPU affinity、NUMA-aware segment 分配、多路径传输 |
| **新硬件/协议支持** | CXI、EFA、华为 HCCL、Sunrise Link、CXL |
| **可靠性** | 网络抖动下的 failover、graceful shutdown、endpoint 重建 |
| **可观测性** | 传输延迟 histogram、带宽利用率、丢包/重传指标 |
| **框架深度集成** | 为 vLLM/SGLang 提供更贴合 PD 分离语义的 API |

---

## 六、从"懂原理"到"扣细节"

很多人会经历这个阶段：架构图看了很多，感觉自己懂了，但真到看代码、改代码、定位问题时，发现细节是空的。这很正常。

### 6.1 为什么会"飘"

| 你以为懂了 | 实际还没碰到的细节 |
|-----------|------------------|
| "Segment 就是一段内存" | Segment 怎么注册？生命周期谁管？卸载时怎么保证没未完成传输？ |
| "RDMA transport 做 P2P" | QP 怎么创建和复用？wr 怎么组织？重连时未完成请求怎么办？ |
| "metadata server 做 peer 发现" | segment 信息怎么序列化？etcd/HTTP/Redis 三种后端一致性差异？ |
| "Master 管理对象映射" | 一个 get 请求从 Python API 到 Master 再到 Transfer Engine，经过多少层 RPC？ |

### 6.2 三个方法把知识压到地面

**方法一：跟踪一条完整请求链路，逐行看**

不要泛泛读文件，选一条具体路径。例如 Python 端 `store.put(key, tensor)` 到底层 RDMA 的全流程：

```
mooncake-wheel/mooncake/structured_object_store.py
  → mooncake-integration/store/store_py.cpp
  → mooncake-store/src/real_client.cpp
  → mooncake-store/src/master_client.cpp (RPC to master)
  → mooncake-store/src/master_service.cpp
  → back to client
  → mooncake-transfer-engine/src/transfer_engine_impl.cpp
  → mooncake-transfer-engine/src/transport/rdma_transport/rdma_context.cpp
  → ibv_post_send()
```

每经过一层，问自己：
- 这一层输入输出是什么？
- 做了什么决策？
- 出错时怎么处理？
- 有没有锁/异步/回调？

**方法二：改一个小东西，然后跑测试验证**

- 在 `rdma_transport.cpp` 里加一行日志，打印每次 post_send 的 wr_id 和长度
- 把 TCP transport 的 buffer size 改大/改小，跑 `transfer_engine_bench` 看吞吐变化
- 在 `transfer_engine_impl.cpp` 的 `transfer()` 入口加一个延迟 histogram
- 故意制造一个 RDMA 连接断开场景，看重连逻辑是否触发

目标不是做优化，而是**让代码在你面前"动"起来**。

**方法三：给自己出题，用代码回答**

- 一个 Batch 里 100 个 request，其中 3 个失败，Transfer Engine 怎么告诉上层？
- metadata server 挂掉 10 秒，已有的 RDMA 连接还能传数据吗？
- SSD offload 时，数据是从 GPU 直接到 SSD，还是先拉回 CPU？

不要猜，去代码里找答案，然后把结论写成几行注释或文档。

### 6.3 本周最小行动

> **在 `rdma_transport.cpp` 或 `tcp_transport.cpp` 里加 5 行日志，打印每次传输的 size、opcode、status，然后跑一遍单元测试和 benchmark，把输出保存下来。**

这件事很小，但会让你从"我觉得传输是这样的"变成"我看到传输就是这样的"。

---

## 七、一句话总结

> **先把 TCP/RDMA transport 和 Transfer Engine 的 segment/batch 模型跑通、读透，再理解 Mooncake Store 控制面与数据面分离，最后结合公司实际网络硬件和 KV Cache 流量特征做针对性优化。**

网络背景让你天然适合从 **transport 层** 和 **性能优化** 切入。
