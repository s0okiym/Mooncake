# Mooncake 异构部署方案：华为 NPU 推理节点 + NVIDIA GPU KV Cache 池

## 1. 概述

本文档分析将 Mooncake 的 Prefill/Decode 推理节点部署在华为 Ascend NPU 机器上，而 KV Cache 池部署在 NVIDIA GPU RDMA 机器上的异构部署方案。涵盖可行性、数据路径、关键限制及变通方案。

### 1.1 目标架构

| 组件 | 部署位置 | 硬件 | 角色 |
|------|---------|------|------|
| P-node (Prefill) | 华为 NPU 机器 | Ascend 910B/910C | KV Cache 生产者 |
| D-node (Decode) | 华为 NPU 机器 | Ascend 910B/910C | KV Cache 消费者 |
| KV Cache Pool | NVIDIA GPU RDMA 机器 | H20/A100/H100 + ConnectX | KV Cache 存储 |

### 1.2 数据流向

```
┌─────────────────┐    WRITE    ┌───────────────────┐    READ    ┌─────────────────┐
│  P-node (910B)  │ ──────────▶ │  KV Cache Pool    │ ─────────▶ │  D-node (910B)  │
│  Ascend NPU     │             │  NVIDIA GPU VRAM  │            │  Ascend NPU     │
└─────────────────┘             └───────────────────┘            └─────────────────┘
         ✓ 当前支持                      存储层                       ✗ 当前不支持
```

**核心结论**：P-node → KV Cache 方向可行，KV Cache → D-node 方向因 READ 语义未实现而不可行。

---

## 2. 关键组件：HeterogeneousRdmaTransport

Mooncake 提供了 `HeterogeneousRdmaTransport`（异构 RDMA 传输层），专门用于 Ascend NPU 与 NVIDIA GPU 之间的跨设备数据传输。

### 2.1 代码位置

- **头文件**：`mooncake-transfer-engine/include/transport/ascend_transport/heterogeneous_rdma_transport.h`
- **实现**：`mooncake-transfer-engine/src/transport/ascend_transport/heterogeneous_rdma_transport/heterogeneous_rdma_transport.cpp`
- **设计文档**：`docs/source/design/transfer-engine/heterogeneous_ascend.md`
- **示例代码**：`mooncake-transfer-engine/example/transfer_engine_heterogeneous_ascend_perf_initiator.cpp`

### 2.2 设计原理

`HeterogeneousRdmaTransport` 是一个装饰器模式（Decorator），内部包装了标准 `RdmaTransport`：

- `getName()` 返回 `"ascend"`，使其在 `MultiTransport::selectTransport()` 中被选中
- 拦截传输请求，根据源地址是否为 NPU 设备内存决定数据路径
- NPU 设备内存需经过 CPU Staging Buffer 中转，再通过标准 RDMA WRITE 发送
- CPU 内存源直接委托给内部 `RdmaTransport`

### 2.3 编译要求

| 侧 | CMake 编译选项 | 说明 |
|----|---------------|------|
| P-node (910B NPU) | `-DUSE_ASCEND_HETEROGENEOUS=ON` | 启用异构传输，链接 Ascend ACL |
| KV Cache (H20 GPU) | `-DUSE_CUDA=ON` | 启用 GPU Direct RDMA |

---

## 3. 数据路径详解

### 3.1 P-node → KV Cache（WRITE 路径，当前支持）

```
NPU HBM ──aclrtMemcpy──▶ CPU Staging Buffer (3GB) ──RDMA WRITE (hns NIC)──▶ Mellanox ConnectX ──DMA──▶ NVIDIA GPU VRAM
```

根据传输大小，存在两条子路径：

#### 3.1.1 小传输路径（< 2MB，aggTransport）

用于聚合小数据块，分摊 HBM→DRAM 拷贝的开销：

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐     ┌──────────────┐
│ NPU HBM      │     │ Device Buffer    │     │ Host Staging     │     │ Remote GPU   │
│ (source)     │ ──▶ │ 4 x 8MB blocks  │ ──▶ │ 3GB pinned       │ ──▶ │ VRAM (target)│
│              │     │ ACL_MEMCPY_D2D   │     │ ACL_MEMCPY_D2H   │     │ RDMA WRITE   │
└──────────────┘     └──────────────────┘     └──────────────────┘     └──────────────┘
```

1. **NPU → Device Buffer**：多个小传输通过 `aclrtMemcpyAsync(ACL_MEMCPY_DEVICE_TO_DEVICE)` 聚合到 8MB 设备缓冲块
2. **Device Buffer → Host Staging**：后台 `transferLoop` 线程将满块通过 `ACL_MEMCPY_DEVICE_TO_HOST` 拷贝到 CPU pinned buffer
3. **Host Staging → Remote GPU**：重写 `request.source` 指针为 host buffer 偏移，委托 `RdmaTransport::submitTransferTask()` 通过 RDMA WRITE 发送

#### 3.1.2 大传输路径（>= 2MB，noAggTransport）

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│ NPU HBM      │     │ Host Staging     │     │ Remote GPU   │
│ (source)     │ ──▶ │ 3GB pinned       │ ──▶ │ VRAM (target)│
│              │     │ ACL_MEMCPY_D2H   │     │ RDMA WRITE   │
└──────────────┘     └──────────────────┘     └──────────────┘
```

直接从 NPU HBM 拷贝到 CPU Staging Buffer，无需中间设备缓冲。

#### 3.1.3 CPU 内存源路径

当源地址为 CPU 内存时（`isCpuMemory()` 返回 true），直接委托给内部 `RdmaTransport`，不经过 ACL staging。

### 3.2 KV Cache → D-node（READ 路径，当前不支持）

**这是异构部署的核心阻塞点。**

`HeterogeneousRdmaTransport` 当前**仅支持 WRITE 语义**，READ 语义尚未实现：

> "Current version only supports WRITE semantics. READ semantics will be implemented in future releases."

这意味着 D-node 无法通过标准的 `submitTransfer(READ)` 操作从 NVIDIA GPU 端拉取 KV Cache 数据。

---

## 4. 跨厂商 RDMA 可行性分析

### 4.1 Huawei hns NIC → Mellanox ConnectX NIC 的 RDMA 通信

**结论：技术上可行。**

RDMA WRITE 的 rkey 由目标端 NIC（Mellanox ConnectX）消费，发起端 NIC（Huawei hns）不需要理解 rkey 格式：

1. 发起端（hns NIC）通过 RoCEv2 发送 RDMA WRITE 报文，报文中携带目标端的 rkey
2. 目标端（ConnectX NIC）接收报文，用 rkey 查找已注册的内存区域
3. ConnectX 通过 DMA-BUF/GPU Direct 将数据直接写入 GPU VRAM

### 4.2 NVIDIA GPU 端内存注册

NVIDIA GPU VRAM 通过以下方式注册为 RDMA 可访问区域：

| 编译模式 | 注册方式 | 要求 |
|---------|---------|------|
| `WITH_NVIDIA_PEERMEM` | `ibv_reg_mr()` | 需要 `nvidia-peermem` 内核模块 |
| `USE_CUDA`（无 PEERMEM） | `ibv_reg_dmabuf_mr()` | 不需要内核模块，DMA-BUF 方式 |

注册流程（`rdma_context.cpp` 中 `registerMemoryRegionInternal`）：

1. `cuPointerGetAttribute` 检测地址类型（HOST/DEVICE）
2. 对于 GPU VRAM：`cuMemGetAddressRange` 获取真实分配基址
3. `cuMemGetHandleForAddressRange` 获取 DMA-BUF fd
4. `ibv_reg_dmabuf_mr()` 注册，生成 rkey

### 4.3 网络要求

- 两端 NIC 必须在同一 RoCEv2 或 InfiniBand 网络上可达
- Huawei hns NIC 支持 RoCEv2，可与 Mellanox ConnectX 通过以太网互联
- 需正确配置 RoCEv2 无损网络（PFC、ECN、DSCP）

### 4.4 Topology 配置

NIC 优先级矩阵必须正确映射 NPU/CPU 位置到 RDMA NIC。异构示例中的配置如下：

```json
{
  "cpu:0": [["hns_2"], []],
  "npu:0": [["hns_2"], []],
  "cuda:0": [["hns_2"], []]
}
```

---

## 5. Transport 选择机制

### 5.1 MultiTransport::selectTransport() 逻辑

位于 `mooncake-transfer-engine/src/multi_transport.cpp`：

1. 根据目标 segment ID 查询 `SegmentDesc`
2. 读取 `SegmentDesc::protocol` 字段
3. 在 `USE_ASCEND_HETEROGENEOUS` 编译条件下，将 `"rdma"` 重映射为 `"ascend"`
4. 用协议字符串作为 key 在 `transport_map_` 中查找传输层实例

```cpp
Status MultiTransport::selectTransport(const TransferRequest& entry, Transport*& transport) {
    auto target_segment_desc = metadata_->getSegmentDescByID(entry.target_id);
    auto proto = target_segment_desc->protocol;
#ifdef USE_ASCEND_HETEROGENEOUS
    if (target_segment_desc->protocol == "rdma") {
        proto = "ascend";  // 重映射到异构传输层
    }
#endif
    transport = transport_map_[proto].get();
    return Status::OK();
}
```

### 5.2 元数据共享

不同传输引擎可以共享同一个元数据服务（etcd 或 P2P 握手）：

- NPU 机器注册 segment 时 `protocol = "ascend"`
- NVIDIA 机器注册 segment 时 `protocol = "rdma"`
- 每台机器可通过元数据服务发现对方的 segment 描述符
- 元数据服务本身是协议无关的，不解释 segment 描述符内容

---

## 6. Store 层兼容性

### 6.1 透明性

Mooncake Store 层对底层设备类型无感知：

- `Segment` 结构体仅包含 `id`、`name`、`base`、`size`、`protocol`，无设备类型字段
- `RealClient::TransferData()` 委托给 `TransferEngine::submitTransfer()`，只传递原始地址和大小
- 设备类型感知（CPU/NPU/GPU）完全由 Transfer Engine 的传输层实现负责

### 6.2 已知限制

- **ConfigDict 校验**：`setup_internal` 的 ConfigDict 校验只接受 `"tcp"` 或 `"rdma"` 协议，`"ascend"` 需通过 `setup_real()` 直接调用
- **Dummy Client 的 Ascend 支持**：`RealClient` 有 `ascend_shm_internal()` 和 `ascend_ipc_shm_internal()` 方法用于 NPU 内存映射，但这仅用于共享内存注册，不影响传输路由

---

## 7. 变通方案

### 7.1 方案 A：D-node 使用标准 RdmaTransport

```
D-node（不编译 USE_ASCEND_HETEROGENEOUS）:
  1. RdmaTransport READ: NVIDIA GPU VRAM ──RDMA──▶ D-node CPU Memory
  2. aclrtMemcpy: CPU Memory ──▶ NPU HBM
```

| 项目 | 说明 |
|------|------|
| 优点 | 技术可行，标准 RdmaTransport 支持 RDMA READ |
| 缺点 | 需要应用层手动处理 CPU→NPU 拷贝；额外 memcpy 增加延迟；P-node 和 D-node 编译配置不同 |
| 适用 | 快速验证，延迟敏感度不高的场景 |

### 7.2 方案 B：KV Cache 机器主动推送

```
NVIDIA KV Cache 机器: submitTransfer(WRITE) ──▶ D-node CPU Memory
D-node: aclrtMemcpy ──▶ NPU HBM
```

| 项目 | 说明 |
|------|------|
| 优点 | 利用 WRITE 语义，绕过 READ 限制 |
| 缺点 | 改变 Mooncake 的客户端发起传输架构；需要额外调度逻辑；D-node 仍需 CPU→NPU 拷贝 |
| 适用 | 可自定义调度逻辑的场景 |

### 7.3 方案 C：KV Cache 使用 CPU DRAM

```
P-node (910B): WRITE ──▶ NVIDIA 机器的 CPU DRAM（注册为 RDMA segment）
D-node (910B): READ ──▶ 本地 CPU Memory ──▶ NPU HBM
```

| 项目 | 说明 |
|------|------|
| 优点 | RDMA 读写 CPU 内存完全没问题，无跨厂商 GPU Direct 问题 |
| 缺点 | NVIDIA GPU VRAM 未被充分利用；D-node 仍需 READ 语义和 CPU→NPU 拷贝 |
| 适用 | 对延迟不敏感，需要大容量 KV Cache 的场景 |

### 7.4 方案 D：为 HeterogeneousRdmaTransport 实现 READ 语义（推荐长期方案）

```
NVIDIA GPU VRAM ──RDMA READ──▶ CPU Staging Buffer ──aclrtMemcpy──▶ NPU HBM
```

需要开发的 READ 路径：

1. 在 `HeterogeneousRdmaTransport` 中实现 `READ` opcode 支持
2. RdmaTransport READ 从 NVIDIA GPU VRAM 读取到 CPU Staging Buffer
3. 通过 `aclrtMemcpyAsync(ACL_MEMCPY_HOST_TO_DEVICE)` 拷贝到 NPU HBM
4. 可复用现有的 3GB host staging buffer 和聚合优化策略

| 项目 | 说明 |
|------|------|
| 优点 | 架构完整，全链路支持；复用已有 staging 基础设施 |
| 缺点 | 需要开发工作；仍有 host staging 的性能开销 |
| 适用 | 生产部署，长期方案 |

---

## 8. 关键限制汇总

| 限制 | 详情 | 影响 |
|------|------|------|
| READ 语义未实现 | `HeterogeneousRdmaTransport` 仅支持 WRITE | D-node 无法拉取 KV Cache |
| Host Staging Buffer 固定 3GB | `HUGE_HOST_SIZE = 3GB`，回绕可能覆盖未完成传输 | 并发大传输有数据损坏风险 |
| Aggregation Buffer 有限 | 4 x 8MB = 32MB NPU 侧设备缓冲 | 限制小传输并发数 |
| 混合批次不支持 | `submitTransfer` 仅检查 entries[0] 的内存类型 | 混合 CPU/NPU 源批次会出错 |
| Store ConfigDict 不支持 "ascend" | 校验只接受 "tcp"/"rdma" | 需走 `setup_real()` 直接调用 |
| NPU 内存无法直接注册 RDMA | HeterogeneousRdmaTransport 只注册 CPU 内存 | NPU→RDMA 必须经过 host staging |
| 单向数据流 | 仅支持 NPU → NVIDIA GPU 方向 | 反向需额外开发 |

---

## 9. 部署配置参考

### 9.1 P-node 编译（910B NPU 侧）

```bash
mkdir build && cd build
cmake .. -DUSE_ASCEND=ON -DUSE_ASCEND_HETEROGENEOUS=ON -DUSE_HTTP=ON -DUSE_ETCD=ON
make -j$(nproc)
```

### 9.2 KV Cache Pool 编译（NVIDIA GPU 侧）

```bash
mkdir build && cd build
cmake .. -DUSE_CUDA=ON -DUSE_HTTP=ON -DUSE_ETCD=ON
make -j$(nproc)
```

### 9.3 P-node Transport 安装

```cpp
// P-node 安装异构传输层
auto* engine = new TransferEngine();
engine->installTransport("ascend", nullptr);  // HeterogeneousRdmaTransport
engine->registerLocalMemory(npu_buffer, size, "npu:0");
engine->registerLocalMemory(cpu_buffer, size, "cpu:0");
```

### 9.4 KV Cache Pool Transport 安装

```cpp
// NVIDIA GPU 侧安装标准 RDMA 传输层
auto* engine = new TransferEngine();
engine->installTransport("rdma", nullptr);
engine->registerLocalMemory(gpu_buffer, size, "cuda:0");  // 通过 ibv_reg_dmabuf_mr 注册
```

### 9.5 Topology 配置示例

```json
{
  "npu:0": [["hns_2"], []],
  "npu:1": [["hns_3"], []],
  "cpu:0": [["hns_2"], []],
  "cuda:0": [["hns_2"], []]
}
```

### 9.6 元数据服务配置

两端共享同一个 etcd 集群：

```bash
# P-node 和 KV Cache 机器均使用
export MC_METADATA_SERVER=etcd://10.0.0.100:2379
```

---

## 10. 性能考量

### 10.1 带宽瓶颈

异构路径的主要瓶颈在于 NPU HBM → CPU DRAM 的拷贝带宽：

| 路径段 | 典型带宽 | 瓶颈 |
|--------|---------|------|
| NPU HBM → CPU DRAM (aclrtMemcpy) | ~12-25 GB/s | PCIe 带宽限制 |
| CPU DRAM → RDMA WRITE | ~50-100 Gb/s | RoCEv2 网络带宽 |
| RDMA → GPU VRAM (GPUDirect) | ~50-100 Gb/s | 网络带宽 |

### 10.2 延迟优化

- **聚合优化**：小传输（< 2MB）使用 aggTransport 分摊 HBM→DRAM 拷贝开销
- **流水线**：transferLoop 后台线程异步拷贝，与 RDMA 传输重叠执行
- **NUMA 感知**：确保 hns NIC 和 CPU buffer 在同一 NUMA 节点

### 10.3 与同构方案对比

| 指标 | 同构 (NPU↔NPU ADXL) | 异构 (NPU↔GPU RDMA) |
|------|---------------------|---------------------|
| Prefill→Pool 带宽 | HCCS/RoCE 全带宽 | 受 host staging 限制 |
| Pool→Decode 带宽 | ADXL 直传 | 当前不支持 |
| 延迟 | 低 | 增加 host staging 延迟 |
| 部署复杂度 | 低 | 高（跨厂商、跨编译） |

---

## 11. 建议与路线图

### 短期（验证阶段）

1. 使用方案 A（D-node 用标准 RdmaTransport + 手动 aclrtMemcpy）进行功能验证
2. 确认跨厂商 RoCEv2 网络连通性和稳定性
3. 评估 host staging 对 Prefill 延迟的影响

### 中期（优化阶段）

1. 为 `HeterogeneousRdmaTransport` 实现 READ 语义（方案 D）
2. 优化 host staging buffer 管理（避免回绕、支持更大并发）
3. 支持混合 CPU/NPU 源的批次传输

### 长期（生产部署）

1. 探索 Ascend NPU 内存直接 RDMA 注册（类似 GPUDirect）的可能性
2. 评估 Fabric Memory（A3 平台，CANN 9.0+）是否支持跨厂商远程内存访问
3. 考虑统一传输层设计，将异构逻辑下沉到更底层

---

## 附录 A：关键源文件索引

| 文件 | 用途 |
|------|------|
| `mooncake-transfer-engine/include/transport/ascend_transport/heterogeneous_rdma_transport.h` | 异构传输层头文件 |
| `mooncake-transfer-engine/src/transport/ascend_transport/heterogeneous_rdma_transport/heterogeneous_rdma_transport.cpp` | 异构传输层实现 |
| `mooncake-transfer-engine/example/transfer_engine_heterogeneous_ascend_perf_initiator.cpp` | 异构性能测试示例 |
| `mooncake-transfer-engine/src/multi_transport.cpp` | Transport 选择逻辑（含 HETEROGENEOUS 重映射） |
| `mooncake-transfer-engine/src/transport/rdma_transport/rdma_context.cpp` | GPU VRAM 内存注册（ibv_reg_dmabuf_mr） |
| `mooncake-transfer-engine/src/transport/rdma_transport/rdma_transport.cpp` | RDMA 传输层实现 |
| `docs/source/design/transfer-engine/heterogeneous_ascend.md` | 异构传输设计文档 |
| `mooncake-common/common.cmake` | `USE_ASCEND_HETEROGENEOUS` 编译选项定义 |

## 附录 B：环境变量参考

| 环境变量 | 说明 | 默认值 |
|---------|------|--------|
| `MC_METADATA_SERVER` | 元数据服务连接串 | - |
| `MC_USE_TENT` | 启用 TENT 传输引擎 | - |
| `ASCEND_AUTO_CONNECT` | ADXL 自动连接 | 0 |
| `ASCEND_RDMA_TC` | Ascend RDMA Traffic Class | - |
| `ASCEND_RDMA_SL` | Ascend RDMA Service Level | - |
| `HCCL_RDMA_TIMEOUT` | RDMA 重传超时系数 | - |
| `HCCL_RDMA_RETRY_CNT` | RDMA 重传次数 | - |
