# 华为云 NPU 部署 Mooncake 网络最佳实践

本文档基于 Mooncake 源码深度分析和华为昇腾生态技术规范，系统梳理在华为云 NPU 环境部署 Mooncake 时，网络层面的稳定性与性能最佳实践。

---

## 目录

1. [华为昇腾网络架构总览](#1-华为昇腾网络架构总览)
2. [传输方式选型与编译配置](#2-传输方式选型与编译配置)
3. [HCCS 域内通信（同节点/同域）](#3-hccs-域内通信同节点同域)
4. [RoCE RDMA 跨节点通信](#4-roce-rdma-跨节点通信)
5. [无损网络配置（稳定性核心）](#5-无损网络配置稳定性核心)
6. [Traffic Class 与 Service Level 配置](#6-traffic-class-与-service-level-配置)
7. [ADXL 直连传输部署](#7-adxl-直连传输部署)
8. [HCCL Transport 部署](#8-hccl-transport-部署)
9. [UBShmem 节点内传输](#9-ubshmem-节点内传输)
10. [异构 RDMA 传输（910B→H20）](#10-异构-rdma-传输910bh20)
11. [Fabric Memory 与内存管理](#11-fabric-memory-与内存管理)
12. [元数据服务与 Master 组网](#12-元数据服务与-master-组网)
13. [容器化部署网络要点](#13-容器化部署网络要点)
14. [端口规划](#14-端口规划)
15. [性能调优参数全表](#15-性能调优参数全表)
16. [稳定性保障机制](#16-稳定性保障机制)
17. [故障排查清单](#17-故障排查清单)
18. [典型部署拓扑](#18-典型部署拓扑)

---

## 1. 华为昇腾网络架构总览

### 硬件平台

| 服务器型号 | NPU 配置 | 域内互联 | 跨节点互联 | 典型场景 |
|-----------|---------|---------|-----------|---------|
| Atlas 800T A2 (910B) | 8×Ascend 910B | HCCS (392GB/s) | RoCEv2 (100GbE/200GbE) | 训练 + 推理 |
| Atlas 800I A2 (910B) | 8×Ascend 910B | HCCS | RoCEv2 | 推理 |
| Atlas 800T A3 | 8×Ascend 910C | HCCS | RoCEv2 + Fabric Memory | 训练 |
| Atlas 800I A3 | 8×Ascend 910C | HCCS + Fabric | RoCEv2 + Fabric Memory | 推理 |

### 网络层次

```
┌──────────────────────────────────────────────────────────┐
│                   跨节点网络 (RoCEv2)                      │
│              100GbE / 200GbE 以太网交换机                   │
│         ┌──────┐    ┌──────┐    ┌──────┐                 │
│         │ NPU0 │    │ NPU0 │    │ NPU0 │                 │
│         │ NIC  │←──>│ NIC  │←──>│ NIC  │                 │
│         └──┬───┘    └──┬───┘    └──┬───┘                 │
│  ┌───────┴──────────────┴──────────────┴────────┐        │
│  │              HCCS 域内互联 (392GB/s)            │        │
│  │  NPU0 ↔ NPU1 ↔ ... ↔ NPU7                    │        │
│  │  每 8 个 NPU 为一个 HCCS 域                     │        │
│  └───────────────────────────────────────────────┘        │
│         Node A              Node B              Node C     │
└──────────────────────────────────────────────────────────┘
```

**关键概念：**

- **HCCS (Huawei Cache Coherence System)**：华为自研高速互联总线，连接同一节点内的 NPU，带宽 392GB/s。A2 服务器中每 8 个 NPU 构成一个 HCCS 域。
- **RoCEv2**：跨节点通信协议，运行在标准以太网上，**必须配置无损网络**。
- **Fabric Memory**：A3 平台新增特性，允许跨进程/跨节点直接访问远程 Host 内存，需要 CANN 9.0+ 和 HDK 26.0+。

### Mooncake 传输方式映射

| 通信场景 | Mooncake Transport | 底层协议 | 带宽 |
|---------|-------------------|---------|------|
| 同 HCCS 域内 NPU 间 | HCCL (IPC) / UBShmem / ADXL (HCCS) | HCCS/VNIC | 392 GB/s |
| 跨 HCCS 域、同节点 | HCCL (RoCE) / ADXL (RoCE) | RoCEv2 (VNIC) | ~25 GB/s |
| 跨节点 NPU 间 | HCCL (RoCE) / ADXL (RoCE) | RoCEv2 (物理 NIC) | ~25 GB/s/NIC |
| NPU → 远端 Host (RDMA) | Heterogeneous RDMA | NPU→Host暂存→RDMA | 受限于 HBM→DRAM |
| 节点内进程间共享 | UBShmem (Fabric/IPC) | Fabric Memory / IPC | 取决于模式 |

---

## 2. 传输方式选型与编译配置

### 决策树

```
部署场景是什么?
├── 纯昇腾 NPU 集群 (910B/910C)
│   ├── A3 平台 (支持 Fabric Memory)
│   │   └── 推荐: ADXL (ascend_direct) + Fabric Memory
│   ├── A2 平台
│   │   └── 推荐: ADXL (ascend_direct)
│   └── 需要 HCCL 兼容
│       └── HCCL Transport (hccl) — 即将废弃，仅兼容用
├── 异构集群 (910B NPU + H20 GPU)
│   └── Heterogeneous RDMA (ascend) — NPU→GPU 场景
├── 节点内 NPU 内存共享 (Mooncake Store)
│   └── UBShmem (ubshmem)
└── 昆鹏 CPU + UB 设备
    └── UB Transport (ub) — 需要 URMA SDK
```

### 编译选项

```bash
# ADXL 直连传输（推荐，A2/A3 均可）
cmake .. -DUSE_ASCEND_DIRECT=ON

# HCCL Transport（兼容旧版，即将废弃）
cmake .. -DUSE_ASCEND=ON

# 异构 RDMA（910B→H20 场景）
cmake .. -DUSE_ASCEND_HETEROGENEOUS=ON -DUSE_CUDA=ON

# UBShmem（节点内内存共享）
cmake .. -DUSE_UBSHMEM=ON

# 昆鹏 UB 传输
cmake .. -DUSE_UB=ON

# 多协议组合（常见生产配置）
cmake .. -DUSE_ASCEND_DIRECT=ON -DUSE_UBSHMEM=ON -DUSE_HTTP=ON -DUSE_ETCD=ON
```

### Transport 名称与 Python 配置

| Transport | protocol 参数 | device_name | server_name 格式 |
|-----------|-------------|-------------|-----------------|
| HCCL | `hccl` | `npu_X` | `ip:port:npu_X` |
| ADXL | `ascend_direct` | (auto) | `ip:port` |
| 异构 RDMA | `ascend` | `mlx5_X` | `ip:port` |
| UBShmem | `ubshmem` | (auto) | `ip:port` |

---

## 3. HCCS 域内通信（同节点/同域）

### HCCS 域判定逻辑

Mooncake 代码中的 HCCS 域判定（`hccl_transport_mem_c.cpp`）：

```cpp
// 每 8 个 NPU 构成一个 HCCS 域 (A2 系列)
bool is_cross_hccs = !(same_host && (local_devicePhyId / 8) == (remote_devicePhyId / 8));
```

- `devicePhyId / 8` 相同 → 同 HCCS 域 → 使用 IPC/VNIC
- 不同 → 跨 HCCS 域 → 使用 RoCE

### 同域通信路径

**HCCL Transport（IPC 模式）：**
1. 通过 `hrtRaGetSingleSocketVnicIpInfo()` 获取 VNIC IP
2. 调用 `P2PMgmtPub::EnableP2P()` + `WaitP2PEnabled()` 启用 P2P 访问
3. 创建 TransportMem（类型：IPC）
4. 通过 `HcclMemGrant()` 授权对端进程访问

**ADXL（HCCS 模式，默认）：**
- A2/A3 服务器内默认使用 HCCS 协议
- 无需额外配置

**强制使用 RoCE 替代 HCCS：**
```bash
export HCCL_INTRA_ROCE_ENABLE=1
```

> **注意**：强制使用 RoCE 会降低同域带宽（392GB/s → ~25GB/s），仅在调试或特定场景使用。

### 稳定性要点

1. **P2P 访问授权**：HCCL IPC 模式需要 `HcclMemGrant` 成功，确保 NPU 间 P2P 通道正常
2. **VNIC IP 可达**：同域通信依赖 VNIC 虚拟网卡，确保 VNIC IP 可解析
3. **内存对齐**：HCCS 协议要求注册内存地址 2MB 对齐（NPU Device Memory）
4. **Fabric Memory 模式**（A3）：通过 `ASCEND_ENABLE_USE_FABRIC_MEM=1` 启用，可跨进程直接访问内存

---

## 4. RoCE RDMA 跨节点通信

### 架构

```
┌─────────────┐     RoCEv2 (UDP 4791)     ┌─────────────┐
│  Node A     │◄──────────────────────────►│  Node B     │
│  NPU0-7     │    交换机 (PFC/ECN/DSCP)    │  NPU0-7     │
│  每卡独立   │                             │  每卡独立   │
│  RDMA NIC   │                             │  RDMA NIC   │
└─────────────┘                             └─────────────┘
```

**关键特性：**
- 每张 NPU 卡拥有独立的参数面 NIC（从 `/etc/hccn.conf` 读取 `address_X`）
- HCCL Transport 使用 `DEVICE_NIC_TYPE` 通过 `HcclNetOpenDev()` 打开物理 NIC
- ADXL 通过 `AdxlEngine::Connect()` 建立直连

### hccn.conf 配置

`/etc/hccn.conf` 是昇腾 NPU 网络配置的核心文件，格式：

```
address_0=10.0.0.1    # NPU 0 的 NIC IP
address_1=10.0.0.2    # NPU 1 的 NIC IP
...
address_7=10.0.0.8    # NPU 7 的 NIC IP
```

**部署检查：**
```bash
# 确认 hccn.conf 存在且配置正确
cat /etc/hccn.conf

# 确认每张卡的 NIC IP 可达
ping -c 3 <address_X>

# 确认 RDMA 设备可见
hccn_tool -i 0 -ip -s   # 查看 NPU 0 的 IP 配置
```

**容器环境必须挂载：**
```bash
docker run -v /etc/hccn.conf:/etc/hccn.conf:ro ...
```

### HCCL Transport 的 RoCE 连接流程

1. **TCP 控制面**：`initControlSocket()` 在 host 端口 `10000 + devicePhyId` 建立监听
2. **NIC 数据面**：`initServerNetSocket()` 打开物理 NIC，创建 RoCE Server Socket
3. **连接建立**：`createTransportMem()` 创建 TransportMem（类型：ROCE），设置 Traffic Class 和 Service Level
4. **内存注册**：`HcclMemReg → HcclMemExport → ExchangeMemDesc → EnableMemAccess`

---

## 5. 无损网络配置（稳定性核心）

> **这是昇腾 NPU 部署中最重要的稳定性保障措施。** RoCEv2 在标准以太网上运行 RDMA，依赖无损网络避免丢包导致的重传和性能劣化。

### 为什么必须配置无损网络

RDMA 协议假设底层网络不丢包。如果 RoCEv2 数据包被交换机丢弃：
- RDMA NIC 会触发重传（`HCCL_RDMA_RETRY_CNT`），增加延迟
- 重传超时（`HCCL_RDMA_TIMEOUT`）可能导致传输失败
- 严重的丢包会导致连接断开，触发 `clearTransportMems()` 全量重建连接
- 在 KVCache 传输场景中，丢包导致 prefill→decode 延迟暴涨，直接影响推理 TTFT

### 交换机配置

#### 1. PFC (Priority Flow Control)

在 RDMA 流量对应的优先级上启用 PFC，实现逐优先级的流量控制：

```
# 华为 CloudEngine 交换机配置示例
#
# 1. 配置 DSCP 到 802.1p 优先级映射
# 默认昇腾 HCCL_RDMA_TC=132 → DSCP 132 & 0x3F = DSCP 4
# 对应 802.1p CoS 4
#
interface 10GE1/0/1
  trust dscp
  qos dscp 4 local-precedence 4   # 昇腾 RDMA 流量映射到队列 4
#
# 2. 在队列 4 上启用 PFC
#
interface 10GE1/0/1
  priority-flow-control enable
  priority-flow-control priority 4 enable
#
# 3. 配置 PFC 阈值（防止 PFC 死锁）
#
priority-flow-control deadlock-detect interval 100
priority-flow-control deadlock-recover interval 1000
```

**关键参数：**
- PFC 阈值：建议动态阈值，确保缓冲区不会溢出
- PFC 死锁检测：必须启用，防止 PFC 风暴导致全网络阻塞
- 仅在 RDMA 优先级上启用 PFC，其他优先级不启用

#### 2. ECN (Explicit Congestion Notification)

在交换机上启用 ECN 标记，让发送端在拥塞时主动降速：

```
# 华为 CloudEngine 交换机配置示例
#
qos ecn-profile rdma_ecn
  ecn-per-queue enable queue 4
  ecn-per-queue threshold min 150KB max 3MB probability 100
#
interface 10GE1/0/1
  qos ecn-profile rdma_ecn inbound
  qos ecn-profile rdma_ecn outbound
```

**ECN 阈值建议：**
- 最小阈值：150KB-500KB（较早标记，避免队列积压）
- 最大阈值：2MB-5MB
- 概率：100%（简单配置）或使用概率 ECN

#### 3. 全局暂停禁用

```
# 禁用全局暂停（与 PFC 不兼容）
interface 10GE1/0/1
  undo flow-control
```

### NIC 侧配置

```bash
# 查看 NPU NIC 的 DCQCN 参数
hccn_tool -i 0 -roce -g

# 设置 NPU NIC 信任模式为 DSCP（而非 802.1p）
hccn_tool -i 0 -roce -s --trust_dscp 1

# 配置 DSCP 到 SL 的映射
# HCCL_RDMA_TC=132 → DSCP 4, HCCL_RDMA_SL=4
hccn_tool -i 0 -roce -s --dscp2sl 4:4

# 启用 ECN
hccn_tool -i 0 -roce -s --ecn 1

# 配置 RoCEv2 目标端口号（默认 4791）
hccn_tool -i 0 -roce -s --dport 4791
```

### 验证无损网络

```bash
# 1. 检查 PFC 统计
ethtool -S <nic> | grep pfc

# 2. 检查 ECN 标记计数
ethtool -S <nic> | grep ecn

# 3. 检查丢包
ethtool -S <nic> | grep -i drop

# 4. RDMA 带宽测试
# Server
hccn_tool -i 0 -ib_write_bw -s
# Client
hccn_tool -i 0 -ib_write_bw -d <server_ip>
```

---

## 6. Traffic Class 与 Service Level 配置

### 参数对应关系

| 参数 | 环境变量 | 默认值 | 作用 |
|------|---------|--------|------|
| Traffic Class | `HCCL_RDMA_TC` / `ASCEND_RDMA_TC` | 132 | 控制数据包的 DSCP 标记，交换机据此识别 RDMA 流量 |
| Service Level | `HCCL_RDMA_SL` / `ASCEND_RDMA_SL` | 4 | 控制数据包的优先级队列映射 |

### TC 值到 DSCP 的映射

`HCCL_RDMA_TC=132` 的含义：
- 高 2 位 (bit 7-6)：ECN 标记位，通常为 `10`（ECN capable）
- 低 6 位 (bit 5-0)：DSCP 值，`132 & 0x3F = 4`
- 因此 DSCP=4，对应 802.1p CoS=4

### 配置建议

```bash
# HCCL Transport 使用
export HCCL_RDMA_TC=132    # DSCP 4 + ECN capable
export HCCL_RDMA_SL=4      # 优先级队列 4

# ADXL Transport 使用（若不设置则回退到 HCCL_*）
export ASCEND_RDMA_TC=132
export ASCEND_RDMA_SL=4
```

**交换机侧必须配合**：确保 DSCP 4 的流量被映射到启用了 PFC 和 ECN 的队列。

### 自定义 TC 场景

如果网络规划使用了不同的 DSCP 值（例如 DSCP 26 用于存储流量，DSCP 48 用于计算流量），需要同步调整：

```bash
# 映射到 DSCP 26 (TC = 0b10_011010 = 154)
export HCCL_RDMA_TC=154
export HCCL_RDMA_SL=3      # 对应交换机队列 3

# 交换机侧配合
# qos dscp 26 local-precedence 3
# priority-flow-control priority 3 enable
```

---

## 7. ADXL 直连传输部署

> **推荐生产环境使用 ADXL（ascend_direct）**，HCCL Transport 已计划废弃。

### Python API 配置

```python
from mooncake import TransferEngine

engine = TransferEngine()
engine.initialize(
    hostname="node1",
    metadata_server="etcd://10.0.0.1:2379",
    protocol="ascend_direct",
    device_name=""
)
```

### 关键环境变量

```bash
# 传输超时（毫秒）
export ASCEND_CONNECT_TIMEOUT=10000    # 建链超时，默认 10000
export ASCEND_TRANSFER_TIMEOUT=10000   # 传输超时，默认 10000

# RDMA 重传配置
export HCCL_RDMA_TIMEOUT=14           # 重传超时系数，实际超时 = 4.096us × 2^14 ≈ 67ms
export HCCL_RDMA_RETRY_CNT=7          # 重传次数

# 传输模式
export ASCEND_USE_ASYNC_TRANSFER=1    # 异步传输模式（需 CANN 8.5+）
export ASCEND_AUTO_CONNECT=1          # 自动建连/断连（需 CANN 9.0+）

# 线程池
export ASCEND_THREAD_POOL_SIZE=8      # Worker 线程数，默认 8，最大 16

# RoCE 模式（强制同域也使用 RoCE）
export HCCL_INTRA_ROCE_ENABLE=0       # 默认关闭，同域使用 HCCS

# 端口范围
export ASCEND_BASE_PORT=20000         # ADXL 引擎监听端口基数
                                      # 实际端口 = base + phy_dev_id * 100 + offset
```

### 超时参数调优原则

**必须满足**：`ASCEND_TRANSFER_TIMEOUT > 重传超时 × 重传次数`

```
HCCL_RDMA_TIMEOUT=14 → 重传超时 ≈ 4.096us × 2^14 ≈ 67ms
HCCL_RDMA_RETRY_CNT=7 → 总重传时间 ≈ 67ms × 7 ≈ 470ms

因此 ASCEND_TRANSFER_TIMEOUT 至少应 > 500ms
推荐: ASCEND_TRANSFER_TIMEOUT=10000 (10s)
```

### 性能优化配置

```bash
# 大规模集群推荐配置
export ASCEND_USE_ASYNC_TRANSFER=1    # 异步模式，提升并发
export ASCEND_THREAD_POOL_SIZE=16     # 最大线程池
export ASCEND_AUTO_CONNECT=1          # 自动连接管理
export ASCEND_CONNECT_TIMEOUT=30000   # 建链超时适当放大
export ASCEND_TRANSFER_TIMEOUT=30000  # 传输超时适当放大
```

### A3 平台 Fabric Memory 模式

```bash
# A3 专属：启用 Fabric Memory（需 CANN 9.0+ HDK 26.0+）
export ASCEND_ENABLE_USE_FABRIC_MEM=1

# Fabric Memory 全局资源配置
export ASCEND_GLOBAL_RESOURCE_CONFIG='{"fabric_memory.max_capacity":32}'
```

**Fabric Memory 的优势：**
- 跨进程直接访问远程 Host 内存，无需 IPC key
- 支持 Mooncake Store 场景下的零拷贝传输
- 通过 ACL VMM API 管理（`aclrtMallocPhysical` + `aclrtMapMem`）

### 短连接模式

```bash
# 每次传输后断开连接（适用于长空闲间隔场景）
export ASCEND_USE_SHORT_CONNECTION=1
```

> **注意**：短连接模式会增加建连开销，不适合高频传输场景。

### 缓冲池模式（受限场景）

当设备内存不足时，可通过中间缓冲池实现传输：

```bash
# 格式: BUFFER_NUM:BUFFER_SIZE_MB
export ASCEND_BUFFER_POOL=4:8    # 4 个 8MB 缓冲区
```

> **限制**：缓冲池模式与异步传输不兼容。

---

## 8. HCCL Transport 部署

> HCCL Transport 即将废弃，推荐迁移到 ADXL。此处仅列出兼容性参考。

### server_name 格式

**必须使用 `ip:port:npu_X` 格式**，其中 X 为物理 NPU 设备 ID：

```python
# 错误格式
engine.initialize(..., local_server_name="10.0.0.1:12345")

# 正确格式
engine.initialize(..., local_server_name="10.0.0.1:12345:npu_2")
```

### 关键环境变量

```bash
export ASCEND_TRANSPORT_TRANSFER_MAX_RETRY_COUNT=2     # 最大重试次数
export ASCEND_TRANSPORT_TRANSFER_TIMEOUT=20000         # 传输超时 ms
export ASCEND_TRANSPORT_TRANSFER_ENABLE_FAST_RECOVERY=1 # 快速恢复（清除所有连接）

# 调试
export ASCEND_TRANSPORT_PRINT=1    # 打印传输计时
```

### 快速恢复 vs 选择性恢复

- `ASCEND_TRANSPORT_TRANSFER_ENABLE_FAST_RECOVERY=1`：传输失败时清除**所有**连接，重建干净状态
- 默认（不设置或 0）：仅清除失败远端的连接，保留其他连接

**生产推荐**：启用快速恢复。虽然重建连接有开销，但可避免脏连接导致的级联失败。

---

## 9. UBShmem 节点内传输

### 适用场景

- 同一节点内多个进程间的 NPU 内存共享
- Mooncake Store 场景下，Worker 进程共享 KVCache 缓冲区

### 两种模式

| 模式 | 环境变量 | 适用平台 | 特点 |
|------|---------|---------|------|
| Fabric Memory | 默认 | A3 (CANN 9.0+) | 高性能，跨进程直接访问 |
| IPC | `MC_USE_UBSHMEM_IPC=1` | A2/A3 | 兼容性好，使用 IPC key |

### 配置

```bash
# Fabric Memory 模式（默认，A3 推荐）
# 无需额外配置

# IPC 模式（A2 或旧 CANN 版本）
export MC_USE_UBSHMEM_IPC=1

# 流和线程配置
export MC_UBSHMEM_STREAMS_PER_TRANSFER=4   # 每次传输使用的 ACL 流数
export MC_UBSHMEM_MAX_STREAMS=32            # 最大流池大小
export MC_UBSHMEM_THREAD_POOL_SIZE=8        # 线程池大小 (1-64)
export MC_UBSHMEM_GET_STREAM_TIMEOUT=3000   # 获取流超时 ms
```

### 内存注册

- **Fabric 模式**：`aclrtMemExportToShareableHandleV2()` + `ACL_RT_VMM_EXPORT_FLAG_DISABLE_PID_VALIDATION`
- **IPC 模式**：`aclrtIpcMemGetExportKey()` + `ACL_RT_IPC_MEM_EXPORT_FLAG_DISABLE_PID_VALIDATION`

远端导入：
- **Fabric**：`aclrtMemImportFromShareableHandleV2()` + `aclrtReserveMemAddress()` + `aclrtMapMem()`
- **IPC**：`aclrtIpcMemImportByKey()` + `ACL_RT_IPC_MEM_IMPORT_FLAG_ENABLE_PEER_ACCESS`

---

## 10. 异构 RDMA 传输（910B→H20）

### 架构

```
┌─────────────┐                    ┌─────────────┐
│ 910B NPU    │   HBM→DRAM→RDMA    │ H20 GPU     │
│ (Prefill)   │ ──────────────────>│ (Decode)    │
│             │   NPU暂存 → RDMA   │             │
└─────────────┘                    └─────────────┘
```

### 数据流路径

1. **大块传输（≥2MB）**：`aclrtMemcpyAsync(DEVICE_TO_HOST)` → Host 暂存 → RDMA 发送
2. **小块传输（<2MB）**：`aclrtMemcpyAsync(DEVICE_TO_DEVICE)` → Device 暂存块(8MB) → 后台线程批量 `DEVICE_TO_HOST` → RDMA

### 暂存缓冲区配置

| 缓冲区 | 大小 | 说明 |
|--------|------|------|
| Host 暂存 | 3 GB | `HUGE_HOST_SIZE = 3GB`，用于大块传输 |
| Device 暂存 | 4×8 MB | `HUGE_DEVICE_NUM=4, HUGE_DEVICE_SIZE=8MB`，用于小块聚合 |
| 聚合阈值 | 2 MB | `AGGREGATE_SIZE_LIMIT = 2MB`，大于此值跳过聚合 |

### 编译与部署

```bash
# Prefill 端 (910B)
cmake .. -DUSE_ASCEND_HETEROGENEOUS=ON

# Decode 端 (H20)
cmake .. -DUSE_CUDA=ON
```

### 稳定性要点

1. **Host 暂存缓冲区大小**：3GB 固定，需确保 Host 有足够 DRAM
2. **Device 暂存块竞争**：仅 4 个 8MB 块，高并发时可能成为瓶颈
3. **aclrtSynchronizeStream**：每次暂存拷贝后同步 stream，超时由 `ASCEND_TRANSFER_TIMEOUT` 控制
4. **仅支持 WRITE 语义**：当前版本不支持 READ

---

## 11. Fabric Memory 与内存管理

### ACL VMM 内存分配流程

```
1. aclrtMallocPhysical()     → 分配物理内存
2. aclrtReserveMemAddress()  → 预留虚拟地址空间
3. aclrtMapMem()             → 映射物理内存到虚拟地址
```

### NUMA 感知

Mooncake 的 Ascend Allocator 实现 NUMA 感知映射：

```
Device 0-3 → NUMA Node 0
Device 4-7 → NUMA Node 2
```

内存分配策略：
1. 首选 `ACL_MEM_P2P_HUGE1G`（1GB 大页 P2P 内存）
2. 回退到 `ACL_MEM_P2P_HUGE`（普通大页 P2P 内存）
3. 最终回退到 `ACL_MEM_LOCATION_TYPE_HOST`（Host 内存）

### Fabric Memory 启用条件

| 条件 | 要求 |
|------|------|
| 硬件 | Atlas A3 超节点 |
| CANN 版本 | ≥ 9.0 |
| HDK 版本 | ≥ 26.0 |
| 环境变量 | `ASCEND_ENABLE_USE_FABRIC_MEM=1` |
| 编译时检测 | `ASCEND_SUPPORT_FABRIC_MEM` 宏（由 CMake 从 `aclrt.h` 头文件自动检测） |

---

## 12. 元数据服务与 Master 组网

### 元数据服务选型

| 后端 | 连接串 | 昇腾推荐度 | 说明 |
|------|--------|-----------|------|
| etcd | `etcd://10.0.0.1:2379` | ★★★★★ | 生产首选，强一致 |
| HTTP | `http://master:8080` | ★★★★ | Master 内嵌，简化部署 |
| Redis | `redis://10.0.0.1:6379` | ★★★ | 低延迟，需额外部署 |
| P2P | `P2PHANDSHAKE` | ★★ | 仅测试 |

### Master 启动

```bash
mooncake_master \
  --rpc_port=50051 \
  --rpc_address=0.0.0.0 \
  --rpc_thread_num=64 \
  --enable_http_metadata_server=true \
  --http_metadata_server_port=8080 \
  --allocation_strategy=free_ratio_first \
  --config_path=mooncake-store/conf/master.yaml
```

### Transfer Engine 初始化

```python
# 生产环境推荐
engine.initialize(
    hostname="node1",                        # 本节点主机名
    metadata_server="etcd://10.0.0.1:2379",  # 元数据服务
    protocol="ascend_direct",                # ADXL 传输
    device_name=""                           # 自动发现
)
```

---

## 13. 容器化部署网络要点

### 必须挂载的文件和设备

```bash
docker run -it \
  --network=host \
  --device /dev/davinci0 \
  --device /dev/davinci1 \
  --device /dev/davinci2 \
  --device /dev/davinci3 \
  --device /dev/davinci4 \
  --device /dev/davinci5 \
  --device /dev/davinci6 \
  --device /dev/davinci7 \
  --device /dev/davinci_manager \
  --device /dev/devmm_svm \
  --device /dev/hisi_hdc \
  -v /usr/local/Ascend/driver:/usr/local/Ascend/driver \
  -v /usr/local/Ascend/add-ons:/usr/local/Ascend/add-ons \
  -v /usr/local/Ascend/ascend-toolkit:/usr/local/Ascend/ascend-toolkit \
  -v /etc/hccn.conf:/etc/hccn.conf:ro \
  -v /usr/local/dcmi:/usr/local/dcmi \
  -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi \
  mooncake-ascend:latest
```

### 关键注意事项

1. **`/etc/hccn.conf` 必须挂载**：HCCL 和 ADXL 都依赖此文件获取 NPU NIC IP，缺失会导致连接建立失败
2. **`--network=host`**：昇腾 RDMA 通信必须使用 host 网络，bridge 模式不支持
3. **设备映射**：每张 NPU 卡对应 `/dev/davinciX`，需要全部映射
4. **驱动和 CANN 挂载**：容器内需要访问 Ascend 驱动和 CANN 运行时
5. **NPU SMI 工具**：`npu-smi` 用于设备状态检查，需要映射

### 验证容器环境

```bash
# 在容器内执行
npu-smi info                    # 检查 NPU 设备可见性
cat /etc/hccn.conf              # 检查网络配置
hccn_tool -i 0 -ip -g          # 检查 NPU 0 的 IP
python3 -c "import acl; print('CANN OK')"  # 检查 CANN 可用性
```

---

## 14. 端口规划

### 端口清单

| 服务 | 端口 | 说明 |
|------|------|------|
| Master RPC | 50051 | Client-Master 控制面 |
| HTTP Metadata | 8080 | 元数据注册/发现 |
| Metrics | 9003 | Prometheus 指标 |
| RDMA Handshake | 12001 | Mooncake TE Handshake |
| TE RPC | 15000-17000 | TE 元数据 RPC |
| HCCL TCP 控制面 | 10000+phyId | HCCL Transport Host 侧 TCP |
| ADXL Engine | 20000+phyId×100 | ADXL 监听端口范围 |
| etcd | 2379 | 元数据存储 |

### 防火墙规则

```bash
# 管理网络
iptables -A INPUT -p tcp --dport 50051 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 8080 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 2379 -s 10.0.0.0/24 -j ACCEPT

# RDMA 数据面（RoCEv2 封装在 UDP 中）
iptables -A INPUT -p udp --dport 4791 -s 10.0.0.0/24 -j ACCEPT

# Handshake 和 RPC 端口范围
iptables -A INPUT -p tcp --dport 12001 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 15000:17000 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 10000:10008 -s 10.0.0.0/24 -j ACCEPT  # HCCL
iptables -A INPUT -p tcp --dport 20000:20800 -s 10.0.0.0/24 -j ACCEPT  # ADXL
```

### 双平面部署

```
平面 1 — 管理网络 (1GbE/10GbE 以太网):
  Master RPC, HTTP Metadata, etcd, Metrics, HCCL TCP 控制面

平面 2 — 数据网络 (100GbE/200GbE RoCEv2):
  NPU RDMA 数据传输, RoCEv2 流量
```

---

## 15. 性能调优参数全表

### HCCL Transport

| 变量 | 默认 | 推荐值 | 说明 |
|------|------|--------|------|
| `ASCEND_TRANSPORT_TRANSFER_MAX_RETRY_COUNT` | 2 | 3-5 | 传输重试次数 |
| `ASCEND_TRANSPORT_TRANSFER_TIMEOUT` | 20000 | 20000-30000 | 传输超时 ms |
| `ASCEND_TRANSPORT_TRANSFER_ENABLE_FAST_RECOVERY` | 0 | 1 | 快速恢复模式 |
| `HCCL_RDMA_TC` | 132 | 132 | RDMA Traffic Class |
| `HCCL_RDMA_SL` | 4 | 4 | RDMA Service Level |
| `ASCEND_TRANSPORT_MAX_REG_MEMORY_NUM` | 8192 | 8192 | 最大内存注册数 |
| `Ascend_TCP_TIMEOUT` | 30 | 30-60 | TCP 连接超时秒 |
| `Ascend_HCCL_SOCKET_TIMEOUT` | 30 | 30 | HCCL Socket 超时 |
| `Ascend_TRANSPORT_MEM_TIMEOUT` | 120 | 120 | TransportMem 连接超时 |

### ADXL Transport

| 变量 | 默认 | 推荐值 | 说明 |
|------|------|--------|------|
| `ASCEND_CONNECT_TIMEOUT` | 10000 | 10000-30000 | 建链超时 ms |
| `ASCEND_TRANSFER_TIMEOUT` | 10000 | 10000-30000 | 传输超时 ms |
| `ASCEND_USE_ASYNC_TRANSFER` | 0 | 1 | 异步传输（CANN 8.5+） |
| `ASCEND_AUTO_CONNECT` | 0 | 1 | 自动建连（CANN 9.0+） |
| `ASCEND_THREAD_POOL_SIZE` | 8 | 8-16 | Worker 线程数 |
| `ASCEND_USE_SHORT_CONNECTION` | 0 | 0 | 短连接模式 |
| `ASCEND_BUFFER_POOL` | 0:0 | 按需 | 中间缓冲池 |
| `ASCEND_RDMA_TC` | 回退 HCCL | 132 | ADXL RDMA TC |
| `ASCEND_RDMA_SL` | 回退 HCCL | 4 | ADXL RDMA SL |
| `HCCL_INTRA_ROCE_ENABLE` | 0 | 0 | 同域 RoCE（调试用） |
| `ASCEND_BASE_PORT` | 20000 | 20000 | ADXL 端口基数 |
| `HCCL_RDMA_TIMEOUT` | - | 14 | RDMA 重传超时系数 |
| `HCCL_RDMA_RETRY_CNT` | - | 7 | RDMA 重传次数 |
| `ASCEND_ENABLE_USE_FABRIC_MEM` | 0 | 1(A3) | Fabric Memory |
| `ASCEND_GLOBAL_RESOURCE_CONFIG` | - | 按需 | 全局资源配置 |

### UBShmem Transport

| 变量 | 默认 | 推荐值 | 说明 |
|------|------|--------|------|
| `MC_USE_UBSHMEM_IPC` | 0 | 0(A3)/1(A2) | IPC 模式 |
| `MC_UBSHMEM_STREAMS_PER_TRANSFER` | 4 | 4-8 | 每传输流数 |
| `MC_UBSHMEM_MAX_STREAMS` | 32 | 32-64 | 流池上限 |
| `MC_UBSHMEM_THREAD_POOL_SIZE` | 8 | 8-16 | 线程池大小 |
| `MC_UBSHMEM_GET_STREAM_TIMEOUT` | 3000 | 3000-5000 | 获取流超时 ms |

---

## 16. 稳定性保障机制

### 连接级容错

| 机制 | 实现 | 配置 |
|------|------|------|
| 传输重试 | 失败后重试，可刷新元数据 | `ASCEND_TRANSPORT_TRANSFER_MAX_RETRY_COUNT` / `kTransferRetryTimes=2` |
| 流中止恢复 | `aclrtStreamAbort()` + 重新同步 | 自动 |
| 选择性连接清理 | 仅清理失败远端连接 | `ASCEND_TRANSPORT_TRANSFER_ENABLE_FAST_RECOVERY=0` |
| 全量连接清理 | 清除所有连接重建 | `ASCEND_TRANSPORT_TRANSFER_ENABLE_FAST_RECOVERY=1` |
| 自动断连 | 对端异常离线时自动断开 | `ASCEND_AUTO_CONNECT=1` |
| 短连接模式 | 每次传输后断开 | `ASCEND_USE_SHORT_CONNECTION=1` |

### 超时链路保障

```
TCP 控制: Ascend_TCP_TIMEOUT (30s)
  → HCCL Socket: Ascend_HCCL_SOCKET_TIMEOUT (30s)
    → TransportMem 建连: Ascend_TRANSPORT_MEM_TIMEOUT (120s)
      → RDMA 传输: ASCEND_TRANSFER_TIMEOUT (10s)
        → RDMA 重传: HCCL_RDMA_TIMEOUT × HCCL_RDMA_RETRY_CNT
```

**原则**：外层超时 > 内层超时 × 重试次数

### RDMA 重传与超时关系

```
实际重传超时 = 4.096μs × 2^HCCL_RDMA_TIMEOUT
总重传时间 = 实际重传超时 × HCCL_RDMA_RETRY_CNT

例: HCCL_RDMA_TIMEOUT=14, HCCL_RDMA_RETRY_CNT=7
  → 单次超时 ≈ 67ms
  → 总重传时间 ≈ 470ms
  → ASCEND_TRANSFER_TIMEOUT 应 > 500ms
  → 推荐 ASCEND_TRANSFER_TIMEOUT=10000 (10s)
```

### 网络级稳定性

1. **PFC 死锁防护**：交换机必须启用 PFC 死锁检测和恢复
2. **ECN 拥塞控制**：交换机 ECN 标记触发 DCQCN 降速，避免 PFC 风暴
3. **流量隔离**：RDMA 流量与存储/管理流量使用不同 DSCP 优先级
4. **网卡 Watchdog**：HCCL 的 `epoll` 监控机制检测 NIC 状态变化
5. **连接重建**：Mooncake 在连接失败后自动重建，无需人工干预

---

## 17. 故障排查清单

### NPU 网络基础检查

```bash
# 1. NPU 设备可见性
npu-smi info

# 2. hccn.conf 配置
cat /etc/hccn.conf

# 3. NPU NIC IP 可达性
for i in $(seq 0 7); do
    ip=$(grep "address_$i" /etc/hccn.conf | cut -d= -f2)
    echo "NPU $i: $ip"
    ping -c 1 -W 1 $ip
done

# 4. RDMA 设备检查
hccn_tool -i 0 -roce -g

# 5. 带宽测试
# Server
hccn_tool -i 0 -ib_write_bw -s
# Client
hccn_tool -i 0 -ib_write_bw -d <server_ip>
```

### Mooncake 连接问题

```bash
# 1. 检查日志
export MC_LOG_LEVEL=INFO
export ASCEND_TRANSPORT_PRINT=1

# 2. HCCL 日志
# 位于 /root/Ascend/log/debug/plog

# 3. 检查端口占用
ss -tlnp | grep -E '1000[0-8]|200[0-9]{2}|15000|12001'

# 4. 检查内存注册
export ASCEND_TRANSPORT_MAX_REG_MEMORY_NUM=8192
```

### 常见错误

| 错误 | 原因 | 解决方案 |
|------|------|---------|
| `hccn.conf not found` | 容器内缺少配置文件 | 挂载 `/etc/hccn.conf` |
| `aclrtMemcpyAsync failed` | NPU 内存未注册或未对齐 | 确认 2MB 对齐，检查内存注册 |
| `TransportMem connect timeout` | RoCE 网络不通 | 检查 NIC IP、交换机 PFC/ECN |
| `RDMA retry count exceeded` | 丢包导致重传耗尽 | 检查无损网络配置 |
| `ACL_ERROR_RT_STREAM_ABORT` | 流超时被中止 | 增大 `ASCEND_TRANSFER_TIMEOUT` |
| `Failed to open device` | NPU 设备不可访问 | 检查容器设备映射 |
| `Port already in use` | HCCL/ADXL 端口冲突 | 调整 `ASCEND_BASE_PORT` 或 HCCL 端口 |

### 性能低于预期

```bash
# 1. 确认传输模式
# 查看日志：是否使用 HCCS 还是 RoCE
# 同域默认 HCCS (392GB/s)，跨节点 RoCE (~25GB/s/NIC)

# 2. 确认异步模式
echo $ASCEND_USE_ASYNC_TRANSFER   # 应为 1

# 3. 确认无损网络
ethtool -S <nic> | grep -E "pfc|ecn|drop"

# 4. 确认 TC/SL 配置
echo $HCCL_RDMA_TC  # 应为 132
echo $HCCL_RDMA_SL  # 应为 4

# 5. 确认 Fabric Memory（A3）
echo $ASCEND_ENABLE_USE_FABRIC_MEM  # 应为 1
```

---

## 18. 典型部署拓扑

### Atlas 800T A2 (910B) 集群

```
┌─────────────────────────────────────────────────────────┐
│              管理网络 (10GbE Spine-Leaf)                  │
│  ┌───────┐  ┌───────┐  ┌───────┐                      │
│  │ etcd  │  │ etcd  │  │ etcd  │  (3节点集群)           │
│  └───────┘  └───────┘  └───────┘                      │
│  ┌──────────┐  ┌──────────┐                             │
│  │ Master A │  │ Master B │  (HA 主备)                  │
│  └──────────┘  └──────────┘                             │
├─────────────────────────────────────────────────────────┤
│            数据网络 (200GbE RoCEv2 Spine-Leaf)            │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Prefill A    │  │ Prefill B    │  │ Decode A     │  │
│  │ 8×910B       │  │ 8×910B       │  │ 8×910B       │  │
│  │ ADXL+RoCE   │  │ ADXL+RoCE   │  │ ADXL+RoCE   │  │
│  │ HCCL_TC=132  │  │ HCCL_TC=132  │  │ HCCL_TC=132  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**配置：**
```bash
# 所有 NPU 节点
export ASCEND_USE_ASYNC_TRANSFER=1
export ASCEND_AUTO_CONNECT=1
export ASCEND_THREAD_POOL_SIZE=16
export HCCL_RDMA_TC=132
export HCCL_RDMA_SL=4
export HCCL_RDMA_TIMEOUT=14
export HCCL_RDMA_RETRY_CNT=7
export ASCEND_TRANSFER_TIMEOUT=30000
export ASCEND_CONNECT_TIMEOUT=30000
export MC_METADATA_SERVER=etcd://etcd1:2379
```

### Atlas 800T A3 (910C) 集群 + Fabric Memory

```
┌─────────────────────────────────────────────────────────┐
│              管理网络 (10GbE)                             │
│  ┌───────┐  ┌──────────┐                                │
│  │ etcd  │  │ Master   │  (HA + Snapshot)               │
│  └───────┘  └──────────┘                                │
├─────────────────────────────────────────────────────────┤
│         数据网络 (200GbE RoCEv2 + Fabric Memory)          │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐                    │
│  │ Prefill      │  │ Decode       │                    │
│  │ 8×910C       │  │ 8×910C       │                    │
│  │ ADXL+Fabric  │  │ ADXL+Fabric  │                    │
│  │ UBShmem      │  │ UBShmem      │                    │
│  │ (跨进程共享) │  │ (跨进程共享) │                    │
│  └──────────────┘  └──────────────┘                    │
└─────────────────────────────────────────────────────────┘
```

**额外配置：**
```bash
export ASCEND_ENABLE_USE_FABRIC_MEM=1
export ASCEND_GLOBAL_RESOURCE_CONFIG='{"fabric_memory.max_capacity":32}'
```

### 异构集群 (910B + H20)

```
┌──────────────┐    RDMA (RoCE)    ┌──────────────┐
│ Prefill      │ ────────────────> │ Decode       │
│ 8×910B NPU   │  NPU→Host→RDMA   │ 8×H20 GPU    │
│ Ascend Heter.│                   │ RDMA+GDR     │
└──────────────┘                   └──────────────┘
```

**Prefill 端 (910B)：**
```bash
export MC_METADATA_SERVER=etcd://etcd1:2379
# protocol = "ascend"
```

**Decode 端 (H20)：**
```bash
export MC_METADATA_SERVER=etcd://etcd1:2379
# protocol = "rdma"
export MC_FORCE_HCA=true
```

---

## 附录：重要事项速查清单

### 部署前必检（Stability）

- [ ] `/etc/hccn.conf` 已正确配置且在容器内可见
- [ ] 交换机已配置 PFC（RDMA 优先级）、ECN、禁用全局暂停
- [ ] NPU NIC 的 DSCP/TC/SL 与交换机队列映射一致
- [ ] 防火墙已放行 RoCEv2 UDP 4791、TCP 控制面端口
- [ ] `ASCEND_TRANSFER_TIMEOUT > 重传超时 × 重传次数`
- [ ] HCCL 日志路径 `/root/Ascend/log/debug/plog` 可写
- [ ] 容器使用 `--network=host`，映射所有 davinci 设备

### 部署后验证（Performance）

- [ ] `npu-smi info` 显示所有 NPU 正常
- [ ] `hccn_tool -i X -ib_write_bw` 带宽达到预期
- [ ] Mooncake 传输日志显示使用 HCCS/RoCE 正确模式
- [ ] `ASCEND_USE_ASYNC_TRANSFER=1` 已启用
- [ ] A3 平台已启用 `ASCEND_ENABLE_USE_FABRIC_MEM=1`
- [ ] PFC/ECN 计数器正常（无异常 pause frame）
- [ ] 端到端 KVCache 传输延迟符合预期