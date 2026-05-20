# Mooncake 组网部署最佳实践

本文档基于 Mooncake 源码深度分析，系统阐述生产环境部署时的组网方案选型、关键配置、性能调优与运维考量。

---

## 目录

1. [架构总览与网络角色](#1-架构总览与网络角色)
2. [传输协议选型](#2-传输协议选型)
3. [RDMA 组网方案（推荐生产环境）](#3-rdma-组网方案推荐生产环境)
4. [TCP 组网方案](#4-tcp-组网方案)
5. [云上组网方案](#5-云上组网方案)
6. [异构加速器组网方案](#6-异构加速器组网方案)
7. [元数据服务组网](#7-元数据服务组网)
8. [Master 服务组网](#8-master-服务组网)
9. [HA 高可用组网](#9-ha-高可用组网)
10. [NUMA 拓扑感知与性能优化](#10-numa-拓扑感知与性能优化)
11. [端口规划与防火墙](#11-端口规划与防火墙)
12. [安全考量](#12-安全考量)
13. [监控与可观测性](#13-监控与可观测性)
14. [容器化部署网络注意事项](#14-容器化部署网络注意事项)
15. [环境变量速查表](#15-环境变量速查表)
16. [典型部署拓扑示例](#16-典型部署拓扑示例)
17. [故障排查清单](#17-故障排查清单)

---

## 1. 架构总览与网络角色

Mooncake 的网络通信涉及三类角色、四条关键数据/控制路径：

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Mooncake 网络通信全景                            │
│                                                                     │
│  ┌──────────┐    ① RPC 控制面     ┌──────────────┐                 │
│  │  Client   │ ──────────────────> │  Master       │                │
│  │ (Prefill/ │ <────────────────── │  Service      │                │
│  │  Decode)  │    RPC 响应         │  (mooncake_   │                │
│  └─────┬─────┘                     │   master)     │                │
│        │                           └──────┬────────┘                │
│        │ ② Transfer Engine 数据面         │ ③ 元数据注册/发现        │
│        │ (RDMA/TCP/EFA/...)              │ (etcd/Redis/HTTP)        │
│        v                                  v                         │
│  ┌──────────┐                     ┌──────────────┐                 │
│  │  Client   │  ④ RDMA Handshake  │  Metadata     │                │
│  │ (Worker   │ ────────────────── │  Storage      │                │
│  │  节点)    │   TCP 建连 + GID/LID│  (etcd/Redis/ │               │
│  └──────────┘                     │   HTTP Server)│                │
│                                   └──────────────┘                 │
└─────────────────────────────────────────────────────────────────────┘
```

| 路径 | 协议 | 端口 | 说明 |
|------|------|------|------|
| ① Client ↔ Master RPC | TCP (coro_rpc) | 50051 (默认) | 对象管理、段注册、缓冲区分配 |
| ② Client ↔ Client 数据传输 | RDMA/TCP/EFA/NVLink/... | 动态 (15000-17000) | KVCache 零拷贝传输，核心数据路径 |
| ③ Master/TE → 元数据存储 | etcd/Redis/HTTP | 2379/6379/8080 | 段描述符注册与发现 |
| ④ RDMA Handshake | TCP | 12001 (默认) | QP 信息交换、GID/LID 协商 |

**关键原则**：数据面（②）带宽直接决定系统吞吐，控制面（①③④）延迟影响故障恢复速度。生产环境必须确保数据面零丢包、低延迟。

---

## 2. 传输协议选型

### 决策树

```
是否有 RDMA 网卡 (ibv_devices 可见)?
├── 是 → 使用 RDMA (推荐)
│   ├── InfiniBand → IB 模式 (原生 RDMA)
│   ├── RoCEv2 → 需配置无损以太网 (PFC/ECN)
│   └── eRDMA (阿里云) → 自动适配 (CONFIG_ERDMA)
├── 否 → 是否在 AWS EFA 实例?
│   ├── 是 → 使用 EFA (需编译 -DUSE_EFA=ON)
│   └── 否 → 使用 TCP
└── GPU 直连需求?
    ├── NVIDIA MNNVL → NVLink 跨节点 (需编译 -DUSE_MNNVL=ON)
    ├── AMD ROCm → HIP + RDMA (需编译 -DUSE_HIP=ON)
    └── 华为 Ascend → ascend_direct / HCCL (需编译 -DUSE_ASCEND=ON)
```

### 协议性能对比

| 协议 | 单 NIC 带宽 | 延迟 | CPU 开销 | 适用场景 |
|------|-----------|------|---------|---------|
| **RDMA (IB/RoCE)** | ~25 GB/s (200 Gbps) | <1μs | 极低 | 生产环境首选 |
| **EFA (AWS)** | ~21 GB/s (~88% RoCE) | ~6-11μs | 中等 | AWS 云上首选 |
| **TCP** | ~10-12 GB/s | ~50-100μs | 高 | 开发/测试/无 RDMA 环境 |
| **NVLink (MNNVL)** | ~300 GB/s+ | <1μs | 极低 | GPU 直连场景 |
| **CXL** | 取决于 CXL 版本 | ~1μs | 极低 | 内存池化场景 |

### 自动降级机制

Transfer Engine 在初始化时自动检测 RDMA 设备：

```cpp
// transfer_engine_impl.cpp:314-316
if (local_topology_->getHcaList().size() > 0 && !getenv("MC_FORCE_TCP") ||
    getenv("MC_FORCE_HCA")) {
    // 安装 RDMA transport
} else {
    // 降级到 TCP transport
}
```

- 检测到 HCA → 自动启用 RDMA
- 设置 `MC_FORCE_TCP=true` → 强制 TCP（即使有 RDMA 网卡）
- 设置 `MC_FORCE_HCA=true` → 强制 RDMA（用于调试）

---

## 3. RDMA 组网方案（推荐生产环境）

### 3.1 网络拓扑设计

#### InfiniBand 组网

```
                    ┌─────────────┐
                    │  IB Switch  │
                    │  (子网管理器) │
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
           │               │               │
     ┌─────┴─────┐   ┌────┴─────┐   ┌────┴─────┐
     │ Prefill   │   │ Decode   │   │ Worker   │
     │ Node      │   │ Node     │   │ Node     │
     │ mlx5_0,1  │   │ mlx5_0,1 │   │ mlx5_0,1 │
     └───────────┘   └──────────┘   └──────────┘
```

**关键配置：**
- 确保子网管理器（OpenSM 或交换机内置）正常运行
- `MC_IB_PORT=1`（默认），多端口 HCA 需确认活跃端口
- GID 索引自动选择（`MC_GID_INDEX=-1`），优先 RoCEv2 IPv4-mapped GID
- MTU 设置：`MC_MTU=4096`（默认，必须不超过链路实际 MTU）

#### RoCEv2 组网

RoCEv2 在标准以太网上运行 RDMA，**必须配置无损网络**：

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│ Prefill  │───>│  ToR     │───>│ Decode   │
│ Node     │    │  Switch  │    │ Node     │
│ (RoCEv2) │<───│ (PFC/ECN)│<───│ (RoCEv2) │
└──────────┘    └──────────┘    └──────────┘
```

**必需的交换机配置：**

1. **PFC (Priority Flow Control)**：在 RDMA 流量对应的优先级上启用
   - 建议 DSCP 优先级映射：DSCP 26/28 → CoS 3/4
   - 配合 `MC_IB_TC` 设置 IB Traffic Class

2. **ECN (Explicit Congestion Notification)**：启用 ECN 标记
   - 阈值建议：最小阈值 150KB，最大阈值 3MB

3. **全局暂停 (Global Pause) 禁用**：PFC 和 Global Pause 不兼容

**网卡侧配置（Linux）：**
```bash
# 启用 RoCE 模式（Mellanox 网卡示例）
mlxconfig -d /dev/mst/mt4123_pciconf0 set LINK_TYPE_P1=2
# 设置信任模式为 DSCP
mlnx_qos -i <iface> --trust=dscp
# 设置 DSCP 到优先级映射
mlnx_qos -i <iface> --dscp2prio=26,3
```

### 3.2 多网卡 (Multi-NIC) 带宽聚合

Mooncake 原生支持多 RDMA NIC 的带宽聚合和负载均衡：

**工作机制：**
- 每个网卡创建独立的 `RdmaContext`（PD、CQ、WorkerPool）
- `Topology::discover()` 自动检测 NUMA 亲和性
- `selectDevice()` 优先选择同 NUMA 节点的 NIC
- 重试时轮询所有可用 NIC（`MC_PATH_ROUNDROBIN`）

**典型 8-NIC 服务器配置：**
```
NUMA Node 0          NUMA Node 1
├── mlx5_0 (preferred)  ├── mlx5_4 (preferred)
├── mlx5_1 (preferred)  ├── mlx5_5 (preferred)
├── mlx5_2 (avail)      ├── mlx5_6 (avail)
└── mlx5_3 (avail)      └── mlx5_7 (avail)

GPU 0 (NUMA 0)         GPU 1 (NUMA 1)
└── preferred: mlx5_0-3  └── preferred: mlx5_4-7
```

**最佳实践：**
1. **NIC 白名单**：使用 `MC_MS_FILTERS=mlx5_0,mlx5_2,mlx5_4,mlx5_6` 过滤只使用特定网卡
2. **目标设备亲和**：设置 `MC_ENABLE_DEST_DEVICE_AFFINITY=true`，使源端和目标端使用同名 NIC，避免跨 NIC 流量
3. **自定义拓扑**：通过 `MC_CUSTOM_TOPO_JSON` 指定 JSON 文件覆盖自动发现的拓扑
4. **拓扑生成工具**：使用 `scripts/generate_cluster_topology.py` 自动测量 NIC 间带宽/延迟，生成最优分区匹配

**自定义拓扑 JSON 格式：**
```json
{
  "cpu:0": [["mlx5_0", "mlx5_1"], ["mlx5_2", "mlx5_3"]],
  "gpu:0": [["mlx5_0"], ["mlx5_1", "mlx5_2"]]
}
```
- `preferred_hca`（第一个列表）：优先使用的 NIC
- `avail_hca`（第二个列表）：重试时使用的 NIC

### 3.3 RDMA 性能调优参数

| 参数 | 默认值 | 推荐生产值 | 说明 |
|------|--------|-----------|------|
| `MC_NUM_QP_PER_EP` | 2 | 2-4 | 每 endpoint 的 QP 数，增加可提升单连接并行度 |
| `MC_MAX_WR` | 256 | 256-512 | QP 深度，增加可提升 inflight ops |
| `MC_MAX_CQE_PER_CTX` | 4096 | 4096-16384 | CQ 容量，需 >= 所有 endpoint 的 inflight 总和 |
| `MC_WORKERS_PER_CTX` | 2 | 2-8 | 每 NIC 的 CQ 轮询线程数 |
| `MC_NUM_CQ_PER_CTX` | 1 | 1-4 | CQ 数量，多 CQ 分散中断负载 |
| `MC_SLICE_SIZE` | 65536 (64KB) | 65536-262144 | 切片大小，增大减少开销但降低并行度 |
| `MC_MAX_INLINE` | 64 | 64 | 内联数据阈值，小消息避免额外 DMA |
| `MC_MAX_SGE` | 4 | 4 | SGE 条目数 |
| `MC_MTU` | 4096 | 4096 | 路径 MTU |
| `MC_RETRY_CNT` | 9 | 7-13 | 切片重试次数 |
| `MC_IB_TC` | -1 | 浵网 | IB Traffic Class，配合交换机 QoS |
| `MC_IB_PCI_RELAXED_ORDERING` | 0 | 2 (auto) | PCI Relaxed Ordering，可提升 DMA 吞吐 |
| `MC_ENDPOINT_STORE_TYPE` | SIEVE | SIEVE | Endpoint 缓存淘汰策略，SIEVE 适合偏斜访问 |
| `MC_ENABLE_PARALLEL_REG_MR` | -1 (auto) | -1 | 并行 MR 注册，多 NIC + 大内存自动启用 |

**QP 状态转换关键参数（`doSetupConnection`）：**
```
RESET → INIT:  access_flags = LOCAL_WRITE | REMOTE_READ | REMOTE_WRITE | REMOTE_ATOMIC
INIT  → RTR:   path_mtu=min(config, active), max_dest_rd_atomic=16, min_rnr_timer=12
RTR   → RTS:   timeout=14, retry_cnt=7, rnr_retry=7, max_rd_atomic=16
```

### 3.4 GPUDirect RDMA

当 Transfer Engine 编译时启用 `-DUSE_CUDA=ON`，且网卡和驱动支持 GPUDirect RDMA（NVIDIA peermem），数据可直接在 GPU 内存和 RDMA NIC 间传输，**零拷贝跳过 CPU**。

**前提条件：**
1. GPU 和 NIC 在同一 NUMA 节点（或 PCI 拓扑距离最小）
2. 加载 `nvidia_peermem` 模块：`modprobe nvidia_peermem`
3. 编译时 `-DWITH_NVIDIA_PEERMEM=ON`（默认开启）
4. 不使用 peermem 时需编译 nvlink-allocator：`WITH_NVIDIA_PEERMEM=OFF`

**验证 GPUDirect RDMA：**
```bash
# 检查 peermem 模块
lsmod | grep nvidia_peermem
# 检查 RDMA 设备是否支持
ibv_devinfo -d mlx5_0 | grep "max_mr_size"
```

---

## 4. TCP 组网方案

### 适用场景
- 开发测试环境
- 无 RDMA 硬件的部署
- 跨数据中心/跨公网传输（RDMA 不适用时）

### 配置要点

```python
# Python API
engine.initialize(
    hostname="node1",
    metadata_server="http://master:8080",
    protocol="tcp",
    device_name=""
)
```

```bash
export MC_FORCE_TCP=true          # 强制 TCP
export MC_TCP_ENABLE_CONNECTION_POOL=1  # 启用连接池（推荐生产）
```

### TCP 连接池

**启用连接池** (`MC_TCP_ENABLE_CONNECTION_POOL=1`) 可避免频繁 TCP 握手：
- 连接空闲超时：60 秒（硬编码 `kConnectionIdleTimeout`）
- I/O 缓冲区：64KB（硬编码 `kDefaultBufferSize`）
- 支持多连接复用到同一目标

### 本地内存拷贝优化

当仅使用 TCP 传输时（`isTcpOnly() == true`），同进程内的传输自动使用 `memcpy` 替代 TCP loopback：

```
传输策略选择逻辑：
1. 非纯 TCP 模式 → 使用 Transfer Engine
2. 纯 TCP + 本地传输 → 使用 LOCAL_MEMCPY
3. 纯 TCP + 远程传输 → 使用 Transfer Engine (TCP)
```

可通过 `MC_STORE_MEMCPY=1` 强制启用，或 `MC_STORE_MEMCPY=0` 禁用。

### TCP 性能调优

**操作系统层面：**
```bash
# 增大 TCP 缓冲区
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"
sysctl -w net.ipv4.tcp_wmem="4096 65536 134217728"

# 启用 TCP Node_NoDelay（Mooncake 默认已启用 rpc_enable_tcp_no_delay=true）
# 减少 TIME_WAIT
sysctl -w net.ipv4.tcp_tw_reuse=1
```

---

## 5. 云上组网方案

### 5.1 AWS EFA

**适用实例**：p5e.48xlarge, p6-b200.48xlarge, p4d.24xlarge

**编译：**
```bash
cmake .. -DUSE_EFA=ON -DUSE_CUDA=ON
```

**配置：**
```python
engine.initialize(
    hostname="ip-10-0-0-1",
    metadata_server="etcd://10.0.0.1:2379",
    protocol="efa",
    device_name=""
)
```

**EFA 架构特点：**
- 使用 libfabric SRD 协议（非传统 ibverbs QP）
- 共享 Endpoint 模型：一个 `fid_ep` 每本地 NIC，peer 通过 Address Vector (容量 65536) 寻址
- RDMA Write 为软件模拟（CPU 开销高于原生 RDMA）
- 8 个 EFA 设备可达 ~170 GB/s 聚合带宽

**关键环境变量：**
- `MC_EFA_MAX_PTE_ENTRIES`：每 NIC 最大页表条目数（默认 22M）

### 5.2 阿里云 eRDMA

Mooncake 编译时自动启用 eRDMA 支持（`CONFIG_ERDMA` 始终定义），无需额外编译选项。

**eRDMA 特殊处理：**
- QP 重连时需完全重建（非标准 RTS→RESET→INIT→RTR→RTS 流程）
- 代码中 `resetConnection()` 方法专门处理 eRDMA 的 QP 状态机差异

### 5.3 通用云环境

无 RDMA 的云主机使用 TCP 传输。注意：
- 安全组需放行 RPC 端口 (50051)、Handshake 端口 (12001)、RPC 端口范围 (15000-17000)
- 建议使用 `MC_TCP_ENABLE_CONNECTION_POOL=1` 减少短连接开销
- 云网络延迟通常 50-100μs，适合延迟不敏感场景

---

## 6. 异构加速器组网方案

### 6.1 华为 Ascend NPU

| 传输方式 | CMake 选项 | Transport 名 | 特点 |
|---------|-----------|-------------|------|
| HCCL | `-DUSE_ASCEND=ON` | `hccl` | 基于 HCCL 集合通信库 |
| Ascend Direct (ADXL) | `-DUSE_ASCEND_DIRECT=ON` | `ascend_direct` | 直连传输，支持异步/同步 |
| 异构 RDMA | `-DUSE_ASCEND_HETEROGENEOUS=ON` | `ascend` | 包装 RdmaTransport + NPU 内存暂存 |
| UBShmem | `-DUSE_UBSHMEM=ON` | `ubshmem` | 基于 UB 共享内存 |

**Ascend 关键环境变量：**
- `ASCEND_TRANSPORT_TRANSFER_MAX_RETRY_COUNT`：最大重试次数
- `ASCEND_TRANSPORT_TRANSFER_TIMEOUT`：传输超时
- `ASCEND_AUTO_CONNECT`：自动建连
- `ASCEND_RDMA_TC`：RDMA 流量类别

**Server name 格式**（HCCL）：`ip:port:npu_X`（NPU 卡 ID 必填）

### 6.2 AMD ROCm (HIP)

```bash
cmake .. -DUSE_HIP=ON -DUSE_CUDA=OFF  # HIP 替代 CUDA
```

- HIP Transport 自动安装，与 RDMA/TCP 共存
- 处理节点内 GPU P2P（通过 XGMI/IPC）
- 节点间仍使用 RDMA/TCP

### 6.3 昆鹏 UB

```bash
cmake .. -DUSE_UB=ON
```

- 使用 URMA (Unified RDMA Middleware Architecture) 协议
- 需要 `liburma.so`（来自 openEuler URMA SDK）
- QP 类似结构为 Jetty，Endpoint Store 使用 SIEVE 算法
- 配置：`num_jfc_per_ctx`, `num_jetty_per_ep`, `eid_index`

### 6.4 NVIDIA MNNVL (多节点 NVLink)

```bash
cmake .. -DUSE_MNNVL=ON -DUSE_CUDA=ON
export MC_FORCE_MNNVL=true  # 强制使用 MNNVL（即使有 RDMA NIC）
```

- 需要 NVIDIA MNNVL 硬件
- 编译 nvlink-allocator：`mooncake-transfer-engine/nvlink-allocator/build.sh`
- 环境变量 `MC_USE_NVLINK_IPC` 禁用 fabric memory，改用 IPC 模式

---

## 7. 元数据服务组网

元数据服务是 Transfer Engine 发现远程段的关键基础设施，支持三种后端：

### 7.1 后端选型

| 后端 | 连接串格式 | 延迟 | 持久化 | 适用场景 |
|------|-----------|------|--------|---------|
| **etcd** | `etcd://host:2379` | ~5ms | 强一致 | 生产环境首选 |
| **Redis** | `redis://host:6379` | ~1ms | 可配置 | 低延迟需求 |
| **HTTP** | `http://host:8080` | ~10ms | 无 | 轻量部署（Master 内嵌） |
| **P2P** | `P2PHANDSHAKE` | 取决于网络 | 无 | 小集群/测试 |

### 7.2 etcd 部署

```bash
# 最小部署（单节点，仅测试）
etcd --listen-client-urls http://0.0.0.0:2379 \
     --advertise-client-urls http://10.0.0.1:2379

# 生产部署（3 节点集群）
etcd --name etcd1 \
     --listen-client-urls http://0.0.0.0:2379 \
     --advertise-client-urls http://10.0.0.1:2379 \
     --listen-peer-urls http://0.0.0.0:2380 \
     --initial-advertise-peer-urls http://10.0.0.1:2380 \
     --initial-cluster etcd1=http://10.0.0.1:2380,etcd2=http://10.0.0.2:2380,etcd3=http://10.0.0.3:2380 \
     --initial-cluster-token mooncake-cluster \
     --initial-cluster-state new
```

**注意事项：**
- etcd 3 节点集群容忍 1 节点故障
- 磁盘 IO 是瓶颈，建议 SSD
- 默认 2GB 空间限制，大集群需调大 `--quota-backend-bytes`

### 7.3 Redis 部署

```bash
# 启用 AOF 持久化
redis-server --appendonly yes --requirepass <password>

# 环境变量
export MC_REDIS_USERNAME=<user>
export MC_REDIS_PASSWORD=<password>
export MC_REDIS_DB_INDEX=0
```

### 7.4 HTTP 元数据服务器

Mooncake Master 内嵌 HTTP 元数据服务器，替代独立 etcd：

```bash
mooncake_master \
  --enable_http_metadata_server=true \
  --http_metadata_server_host=0.0.0.0 \
  --http_metadata_server_port=8080

# Transfer Engine 连接串
export MC_METADATA_SERVER=http://master:8080/metadata
```

**API 端点：**
- `GET /metadata?key=<key>` — 获取段描述符
- `PUT /metadata?key=<key>` — 注册段描述符
- `DELETE /metadata?key=<key>` — 删除
- `GET /health` — 健康检查

### 7.5 P2P 模式

当 `metadata_server="P2PHANDSHAKE"` 时，无需外部存储，节点间直接交换元数据：
- 每个 Transfer Engine 启动 Handshake Daemon（TCP 端口 12001）
- 元数据通过 `HandShakePlugin::exchangeMetadata()` 直接在 peer 间传递
- **仅适合小规模测试集群**，大规模集群请使用 etcd/Redis

### 7.6 元数据缓存

默认启用元数据缓存（`MC_DISABLE_METACACHE` 不设置时）：
- 段描述符在本地缓存，避免每次传输都查询存储
- 缓存通过 `metacache` 配置项控制
- 当段信息变更时（如节点重启），需要等待缓存过期或强制刷新

### 7.7 集群隔离

使用 `MC_METADATA_CLUSTER_ID` 实现多租户隔离：
```bash
export MC_METADATA_CLUSTER_ID="production-cluster-1"
# 所有元数据键前缀为 mooncake/production-cluster-1/
```

---

## 8. Master 服务组网

### 8.1 启动配置

```bash
mooncake_master \
  --rpc_port=50051 \
  --rpc_address=0.0.0.0 \
  --rpc_thread_num=64 \
  --rpc_enable_tcp_no_delay=true \
  --enable_http_metadata_server=true \
  --http_metadata_server_host=0.0.0.0 \
  --http_metadata_server_port=8080 \
  --metrics_port=9003 \
  --enable_metric_reporting=true \
  --allocation_strategy=free_ratio_first \
  --config_path=mooncake-store/conf/master.yaml
```

### 8.2 网络接口选择

**容器环境**中 `0.0.0.0` 可能绑定到错误的接口，使用 `--rpc_interface` 解析特定接口的 IP：

```bash
mooncake_master --rpc_interface=eth0
```

这会在启动时解析 `eth0` 的 IPv4 地址作为 RPC 绑定地址。

### 8.3 客户端连接 Master

Python 端配置：
```python
# 直接连接 Master
export MOONCAKE_MASTER="10.0.0.1:50051"

# 或通过配置文件
# mooncake_config.json
{
    "master_server_address": "10.0.0.1:50051",
    "local_hostname": "node1",
    "metadata_server": "http://10.0.0.1:8080",
    "protocol": "rdma",
    "device_name": "mlx5_0"
}
```

### 8.4 Master 分配策略

| 策略 | 参数值 | 特点 |
|------|--------|------|
| `random` | `--allocation_strategy=random` | 最快，随机分配，适合均匀负载 |
| `free_ratio_first` | `--allocation_strategy=free_ratio_first` | 采样多候选，选择空闲比最高的段，负载更均衡 |

**生产推荐**：`free_ratio_first`，尤其在多 Worker 节点不均衡时。

---

## 9. HA 高可用组网

### 9.1 架构概览

```
┌─────────────────────────────────────────────┐
│               etcd / Redis / K8s            │ ← Leader Election Backend
│                                             │
│  ┌──── Leader ────┐    ┌─── Standby ────┐  │
│  │  Master A      │    │  Master B      │  │
│  │  (Serving)     │───>│  (Replicating) │  │
│  │  OpLog Write   │    │  OpLog Apply   │  │
│  └────────────────┘    └────────────────┘  │
│         ↑ 故障                                 │
│         │                                     │
│  ┌──── 新 Leader ──┐                         │
│  │  Master B       │ ← 自动提升              │
│  │  (Promoted)     │                         │
│  └─────────────────┘                         │
└─────────────────────────────────────────────┘
```

### 9.2 HA 后端选型

| 后端 | 编译选项 | 连接串 | 适用场景 |
|------|---------|--------|---------|
| etcd | `-DSTORE_USE_ETCD=ON` | `http://etcd1:2379` | 通用生产环境 |
| Redis | `-DSTORE_USE_REDIS=ON` | `redis://redis1:6379` | 低延迟需求 |
| K8s Lease | `-DSTORE_USE_K8S_LEASE=ON` | (K8s 内部) | K8s 原生部署 |

### 9.3 HA Master 启动

```bash
mooncake_master \
  --enable_ha=true \
  --ha_backend_type=etcd \
  --etcd_endpoints="http://etcd1:2379;http://etcd2:2379;http://etcd3:2379" \
  --cluster_id="mooncake_prod" \
  --config_path=mooncake-store/conf/master.yaml
```

### 9.4 HA 状态机

```
kStarting → kStandby → kCandidate → kLeaderWarmup → kServing
                                                    ↓ (领导权丢失)
                                              kStandby (重新竞选)
```

- **Warmup 阶段**：获得领导权后持续续约一个完整的 lease TTL，确保稳定性
- **Serving 阶段**：启动 RPC 服务，同时监控领导权状态
- **Failover**：领导权丢失 → 停止 RPC → 释放领导权 → 进入 Standby → 重新竞选

### 9.5 客户端 HA 感知

客户端自动感知 Master 切换：
1. 连接时通过 LeaderCoordinator 读取当前 Leader 地址
2. 启动 `LeaderMonitorThreadMain` 定期检查领导权变更
3. 检测到新 Leader → `SwitchLeader()` 连接到新 Master
4. 版本保护：不会降级到更旧的 view_version

### 9.6 OpLog 复制

- PUT_END 操作：异步复制（最佳努力）
- REMOVE 操作：同步持久化（必须确保不丢数据）
- OpLogStore 后端：etcd 或本地文件系统
- 最大 OpLog 条目：100,000（内存有界队列）

### 9.7 快照/恢复

```bash
mooncake_master \
  --enable_snapshot=true \
  --snapshot_interval_seconds=600 \
  --snapshot_backend_type=local \
  --snapshot_retention_count=2 \
  --enable_snapshot_restore=true
```

- 快照存储后端：`local`（本地文件系统）或 `s3`
- 环境变量 `MOONCAKE_SNAPSHOT_LOCAL_PATH`（local 后端必填）
- 恢复时从最新快照加载 + 增量 OpLog 回放

---

## 10. NUMA 拓扑感知与性能优化

### 10.1 自动拓扑发现

Mooncake 自动检测 CPU/GPU 与 RDMA NIC 的 NUMA 亲和性：

**CPU 拓扑** (`discoverCpuTopology`)：
- 扫描 `/sys/devices/system/node/node*`
- 同 NUMA 节点的 NIC 为 `preferred`，其他为 `avail`

**GPU 拓扑** (`discoverCudaTopology`)：
- 通过 CUDA API 获取 GPU PCI Bus ID
- 首先筛选同 NUMA 节点的 NIC
- 然后选择 PCI 距离最小的 NIC 作为 `preferred`

**PCI 距离计算**（`getPciDistance`）：
- 解析 `/sys/bus/pci/devices/` 下的符号链接路径
- 路径组件数越少，PCI 拓扑距离越近

### 10.2 设备选择策略

`Topology::selectDevice()` 的选择逻辑：

| 场景 | 策略 |
|------|------|
| 首次尝试 (retry_count=0) | 从 preferred_hca 随机选择 |
| 首次无 preferred | 从 avail_hca 随机选择 |
| 重试 (retry_count>0) | 轮询所有 NIC (preferred + avail) |
| 有 hint (源端 NIC 名) | 查找同名 NIC |
| `MC_PATH_ROUNDROBIN` | 确定性轮询替代随机 |

### 10.3 Worker 线程 NUMA 绑定

- `WorkerPool` 创建线程时自动绑定到 NIC 的 NUMA Socket
- 通过 `bindToSocket()` 使用 `numa_node_to_cpus()` + `pthread_setaffinity_np()`
- 无需手动设置 CPU 亲和性

### 10.4 NUMA 感知内存位置

对于跨 NUMA 节点的大缓冲区，使用 "segments" 编码指定每段的 NUMA 位置：

```
segments:65536:0,0,1,1,0,0,1,1
```

`resolveSegmentsLocation()` 根据偏移量确定数据所在的 NUMA 节点，从而选择最优 NIC。

### 10.5 实践建议

1. **GPU-NIC 对齐**：每个 GPU 使用同 NUMA 节点的 NIC
2. **内存分配**：使用 `numactl --membind=<node>` 确保内存在正确节点
3. **Hugepages**：启用 hugepage 减少 TLB miss
   ```bash
   echo 2048 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
   ```
4. **内存预触**：大缓冲区（>=4GB）自动触发 `preTouchMemory()`，预先解决 page fault

---

## 11. 端口规划与防火墙

### 11.1 端口清单

| 服务 | 默认端口 | 环境变量 | 协议 | 说明 |
|------|---------|---------|------|------|
| Master RPC | 50051 | `--rpc_port` | TCP | Client-Master 控制面 |
| HTTP Metadata | 8080 | `--http_metadata_server_port` | HTTP | TE 元数据注册/发现 |
| Metrics | 9003 | `--metrics_port` | HTTP | Prometheus 指标 |
| RDMA Handshake | 12001 | `MC_HANDSHAKE_PORT` | TCP | QP 连接协商 |
| TE RPC | 15000-17000 | `MC_MIN_RPC_PORT`/`MC_MAX_RPC_PORT` | TCP | 元数据 RPC |
| Client RPC | 12300-14300 | `MC_STORE_CLIENT_MIN_PORT`/`MC_STORE_CLIENT_MAX_PORT` | TCP | Store Client |
| etcd Client | 2379 | - | HTTP/gRPC | 元数据存储 |
| etcd Peer | 2380 | - | HTTP/gRPC | etcd 集群内部 |
| Redis | 6379 | - | TCP | 元数据存储（可选） |
| vLLM Side Channel | 6557 | `VLLM_MOONCAKE_SIDE_CHANNEL_PORT` | TCP | vLLM 集成 |

### 11.2 防火墙规则

**RDMA 网络（数据面）：**
- IB 网络：无需防火墙（子网管理器控制）
- RoCEv2：需放行 RDMA 流量的 UDP 端口（RoCEv2 封装在 UDP 中，默认 4791）
- 建议 RDMA 流量在独立 VLAN/子网

**控制面网络：**
```bash
# iptables 示例：仅允许集群内网访问 Master
iptables -A INPUT -p tcp --dport 50051 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 8080 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 9003 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 12001 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 15000:17000 -s 10.0.0.0/24 -j ACCEPT
```

### 11.3 双网络平面

**最佳实践**：数据面和控制面使用不同的物理网络

```
平面 1 — 管理网络 (1GbE/10GbE):
  Master RPC, HTTP Metadata, etcd, Metrics, Handshake

平面 2 — 数据网络 (100GbE/200GbE IB/RoCE):
  RDMA 数据传输, GPUDirect RDMA
```

- 管理网络绑定到 `eth0`/管理网卡
- 数据网络绑定到 RDMA NIC (`mlx5_*`)
- 使用 `MC_TCP_BIND_ADDRESS` 指定 TCP 绑定的 IP 地址

---

## 12. 安全考量

### 12.1 当前安全状态

**重要**：Mooncake 当前没有内置认证/授权机制：

- RPC 服务器默认绑定 `0.0.0.0`，任何可达的进程都可以调用
- 无 TLS/SSL 支持
- 无 API 密钥或令牌验证
- 数据传输无加密

### 12.2 安全加固建议

1. **网络隔离**
   - Master 和元数据服务仅在内网可达
   - RDMA 流量在独立子网/VLAN
   - 使用 VPC / 安全组限制访问

2. **防火墙**
   - 白名单模式：只允许集群节点 IP
   - 阻断所有管理端口的外部访问

3. **etcd/Redis 加固**
   - 启用 etcd TLS：`--client-cert-auth --trusted-ca-file`
   - Redis 密码：`MC_REDIS_PASSWORD`
   - etcd RBAC

4. **操作系统层面**
   - RDMA 设备权限：`chmod 666 /dev/infiniband/uverbs*` 或配置 udev 规则
   - 限制 `ibv_devices` 访问权限
   - 使用非 root 运行 Master：需配置设备权限

5. **多租户隔离**
   - 使用 `MC_METADATA_CLUSTER_ID` 隔离不同租户的元数据
   - 不同租户使用不同的 Master 实例

---

## 13. 监控与可观测性

### 13.1 Master Metrics

```bash
# Prometheus 格式
curl http://master:9003/metrics

# 人类可读摘要
curl http://master:9003/metrics/summary
```

### 13.2 Transfer Engine Metrics

```bash
export MC_TE_METRIC=1
export MC_TE_METRIC_INTERVAL_SECONDS=5
```

### 13.3 Client Metrics

```bash
export MC_STORE_CLIENT_METRIC=1
export MC_STORE_CLIENT_METRIC_INTERVAL=10
```

### 13.4 Grafana 部署

使用内置 monitoring 栈：

```bash
cd monitoring
docker-compose up -d
# Prometheus: http://localhost:9090
# Grafana: http://localhost:3000 (admin/admin)
```

预置 Dashboard：`monitoring/grafana/dashboards/mooncake.json`

### 13.5 日志级别

```bash
# Transfer Engine
export MC_LOG_LEVEL=INFO  # TRACE/INFO/WARNING/ERROR

# yalantinglibs (coro_rpc/coro_http)
export MC_YLT_LOG_LEVEL=info  # trace/debug/info/warn/error/critical
```

---

## 14. 容器化部署网络注意事项

### 14.1 RDMA 设备透传

```bash
# Docker 运行时透传 RDMA 设备
docker run --rm -it \
  --device /dev/infiniband/uverbs0 \
  --device /dev/infiniband/rdma_cm \
  -v /sys/class/infiniband:/sys/class/infiniband \
  mooncake:latest
```

### 14.2 GPU 透传

```bash
docker run --rm -it \
  --gpus all \
  --device /dev/infiniband/uverbs0 \
  --device /dev/infiniband/rdma_cm \
  -v /sys/class/infiniband:/sys/class/infiniband \
  mooncake:latest
```

### 14.3 Hugepage 配置

```bash
# 宿主机预留 hugepage
sudo sysctl -w vm.nr_hugepages=262144

# Docker 挂载 hugepage
docker run --rm -it \
  -v /dev/hugepages:/dev/hugepages \
  mooncake:latest
```

### 14.4 网络模式选择

| 模式 | 命令 | RDMA 支持 | 适用场景 |
|------|------|-----------|---------|
| Host | `--network=host` | 完整 | 生产环境首选 |
| Macvlan | `--network=macvlan` | 需额外配置 | 需要 IP 隔离 |
| Bridge | 默认 | 不支持 | 不推荐 |

**生产推荐**：`--network=host` 避免 NAT 和 bridge 带来的性能损失和 RDMA 兼容性问题。

### 14.5 RPC 地址解析

容器中 IP 可能不固定，使用 `--rpc_interface` 绑定到稳定接口：

```bash
mooncake_master --rpc_interface=eth0
```

### 14.6 Docker 镜像

预构建镜像包含 RDMA 库：
```dockerfile
# 来自 docker/mooncake.Dockerfile
# 运行时依赖
RUN apt-get install -y ibverbs-providers rdma-core libibverbs1 librdmacm1 libnuma1
```

---

## 15. 环境变量速查表

### 核心传输配置

| 变量 | 默认 | 说明 |
|------|------|------|
| `MC_FORCE_TCP` | - | 强制 TCP 传输 |
| `MC_FORCE_HCA` | - | 强制 RDMA 传输 |
| `MC_METADATA_SERVER` | - | 元数据服务器连接串 |
| `MC_METADATA_CLUSTER_ID` | - | 集群隔离 ID |
| `MC_TCP_ENABLE_CONNECTION_POOL` | 0 | TCP 连接池 |
| `MC_TCP_BIND_ADDRESS` | 自动 | TCP 绑定 IP |
| `MC_USE_IPV6` | - | 启用 IPv6 |

### RDMA 性能调优

| 变量 | 默认 | 说明 |
|------|------|------|
| `MC_NUM_QP_PER_EP` | 2 | QP 数量/endpoint |
| `MC_MAX_WR` | 256 | QP 深度 |
| `MC_MAX_CQE_PER_CTX` | 4096 | CQ 大小 |
| `MC_WORKERS_PER_CTX` | 2 | Worker 线程数/NIC |
| `MC_NUM_CQ_PER_CTX` | 1 | CQ 数量/NIC |
| `MC_SLICE_SIZE` | 65536 | 切片大小 (bytes) |
| `MC_MAX_INLINE` | 64 | 内联数据上限 |
| `MC_MAX_SGE` | 4 | SGE 条目数 |
| `MC_MTU` | 4096 | 路径 MTU |
| `MC_IB_PORT` | 1 | IB 端口号 |
| `MC_GID_INDEX` | -1 (auto) | GID 索引 |
| `MC_IB_TC` | -1 | IB Traffic Class |
| `MC_IB_PCI_RELAXED_ORDERING` | 0 | PCI Relaxed Ordering |
| `MC_RETRY_CNT` | 9 | 重试次数 |
| `MC_ENABLE_DEST_DEVICE_AFFINITY` | - | 目标设备亲和 |
| `MC_ENABLE_PARALLEL_REG_MR` | -1 | 并行 MR 注册 |
| `MC_ENDPOINT_STORE_TYPE` | SIEVE | Endpoint 缓存策略 |
| `MC_MAX_EP_PER_CTX` | 65536 | 最大 endpoint 数 |

### 拓扑与发现

| 变量 | 默认 | 说明 |
|------|------|------|
| `MC_CUSTOM_TOPO_JSON` | - | 自定义拓扑 JSON 文件路径 |
| `MC_PATH_ROUNDROBIN` | - | 轮询路径选择 |
| `MC_MS_AUTO_DISC` | 1 | 自动拓扑发现 |
| `MC_MS_FILTERS` | - | NIC 白名单 |

### 端口配置

| 变量 | 默认 | 说明 |
|------|------|------|
| `MC_HANDSHAKE_PORT` | 12001 | Handshake 端口 |
| `MC_MIN_RPC_PORT` | 15000 | RPC 最小端口 |
| `MC_MAX_RPC_PORT` | 17000 | RPC 最大端口 |
| `MC_STORE_CLIENT_MIN_PORT` | 12300 | Client 最小端口 |
| `MC_STORE_CLIENT_MAX_PORT` | 14300 | Client 最大端口 |

### Store 相关

| 变量 | 默认 | 说明 |
|------|------|------|
| `MC_STORE_MEMCPY` | 0 | 本地 memcpy 优化 |
| `MC_STORE_USE_HUGEPAGE` | - | Hugepage 支持 |
| `MC_STORE_HUGEPAGE_SIZE` | 2MB | Hugepage 大小 |
| `MC_MMAP_ARENA_POOL_SIZE` | - | Arena 池大小 |
| `MC_STORE_CLIENT_METRIC` | 1 | Client 指标 |
| `MC_TE_METRIC` | 0 | TE 指标 |

### 日志

| 变量 | 默认 | 说明 |
|------|------|------|
| `MC_LOG_LEVEL` | INFO | 日志级别 |
| `MC_LOG_DIR` | stderr | 日志目录 |
| `MC_YLT_LOG_LEVEL` | warning | yalantinglibs 日志级别 |

---

## 16. 典型部署拓扑示例

### 16.1 小规模测试集群 (2-4 节点)

```
┌──────────────────────────────────────────┐
│           Node 1 (Master + Worker)        │
│  ┌──────────────┐  ┌──────────────────┐  │
│  │ mooncake_     │  │ Transfer Engine  │  │
│  │ master        │  │ (RDMA mlx5_0)    │  │
│  │ (HTTP Meta)   │  │                  │  │
│  │ :50051/:8080  │  │                  │  │
│  └──────────────┘  └──────────────────┘  │
└─────────────────────┬────────────────────┘
                      │ RDMA / TCP
┌─────────────────────┴────────────────────┐
│           Node 2 (Worker)                 │
│  ┌──────────────────┐                     │
│  │ Transfer Engine  │                     │
│  │ (RDMA mlx5_0)    │                     │
│  └──────────────────┘                     │
└──────────────────────────────────────────┘
```

**配置：**
```bash
# Node 1
mooncake_master \
  --enable_http_metadata_server=true \
  --http_metadata_server_port=8080 \
  --rpc_port=50051

# Node 1 & 2 的 Transfer Engine
export MC_METADATA_SERVER=http://node1:8080/metadata
```

### 16.2 中规模生产集群 (8-32 节点)

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  etcd 1  │  │  etcd 2  │  │  etcd 3  │
└──────────┘  └──────────┘  └──────────┘
      ↑              ↑             ↑
┌─────┴──────────────┴─────────────┴──────┐
│           管理网络 (10GbE)                │
├─────────────────────────────────────────┤
│  ┌──────────┐        ┌──────────┐       │
│  │ Master A │        │ Master B │       │
│  │ (Leader) │        │(Standby) │       │
│  └──────────┘        └──────────┘       │
├─────────────────────────────────────────┤
│           IB Switch (200GbE)             │
│  ┌───────┐ ┌───────┐ ┌───────┐         │
│  │Prefill│ │Prefill│ │Decode │ ...      │
│  │Node 1 │ │Node 2 │ │Node 1 │         │
│  │4xNIC  │ │4xNIC  │ │4xNIC  │         │
│  │4xGPU  │ │4xGPU  │ │4xGPU  │         │
│  └───────┘ └───────┘ └───────┘         │
└─────────────────────────────────────────┘
```

**配置：**
```bash
# Master (HA 模式)
mooncake_master \
  --enable_ha=true \
  --ha_backend_type=etcd \
  --etcd_endpoints="http://etcd1:2379;http://etcd2:2379;http://etcd3:2379" \
  --allocation_strategy=free_ratio_first \
  --enable_http_metadata_server=true \
  --http_metadata_server_port=8080 \
  --rpc_thread_num=64 \
  --enable_snapshot=true \
  --snapshot_backend_type=local \
  --snapshot_interval_seconds=600

# Transfer Engine
export MC_METADATA_SERVER=etcd://etcd1:2379
export MC_WORKERS_PER_CTX=4
export MC_NUM_QP_PER_EP=2
export MC_ENABLE_DEST_DEVICE_AFFINITY=true
```

### 16.3 大规模生产集群 (100+ 节点)

```
┌──────────────────────────────────────────────────┐
│                管理网络 (Spine-Leaf 10GbE)        │
│  ┌───────┐  ┌───────┐  ┌───────┐               │
│  │ etcd  │  │ etcd  │  │ etcd  │  (5节点集群)   │
│  └───────┘  └───────┘  └───────┘               │
│  ┌──────────┐  ┌──────────┐                     │
│  │ Master A │  │ Master B │  (HA 主备)           │
│  │(Leader)  │  │(Standby) │                     │
│  └──────────┘  └──────────┘                     │
│  ┌──────────────────────────┐                    │
│  │ Prometheus + Grafana     │                    │
│  └──────────────────────────┘                    │
├──────────────────────────────────────────────────┤
│            数据网络 (Spine-Leaf 200GbE IB)        │
│                                                  │
│  ┌─────────────────────────────────────────────┐ │
│  │ Prefill Pool          Decode Pool           │ │
│  │ ┌─────┐ ┌─────┐     ┌─────┐ ┌─────┐       │ │
│  │ │ P-1 │ │ P-2 │ ... │ D-1 │ │ D-2 │ ...   │ │
│  │ │8NIC │ │8NIC │     │8NIC │ │8NIC │       │ │
│  │ │8GPU │ │8GPU │     │8GPU │ │8GPU │       │ │
│  │ └─────┘ └─────┘     └─────┘ └─────┘       │ │
│  └─────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

**额外配置：**
```bash
# 自定义拓扑（多 NIC 亲和性优化）
export MC_CUSTOM_TOPO_JSON=/etc/mooncake/cluster-topology.json

# 使用 generate_cluster_topology.py 生成
python3 scripts/generate_cluster_topology.py \
  --src-host node1 --dst-host node2 \
  --file cluster-topology.json

# 增大 CQ 和 Worker
export MC_MAX_CQE_PER_CTX=16384
export MC_WORKERS_PER_CTX=8
export MC_NUM_CQ_PER_CTX=4

# 启用并行 MR 注册
export MC_ENABLE_PARALLEL_REG_MR=1

# 元数据集群隔离
export MC_METADATA_CLUSTER_ID="prod-cluster-1"
```

---

## 17. 故障排查清单

### RDMA 连接失败

```bash
# 1. 检查 RDMA 设备
ibv_devices
ibv_devinfo -d mlx5_0

# 2. 检查端口状态
ibv_devinfo -d mlx5_0 | grep "state:"  # 应为 PORT_ACTIVE

# 3. 检查 GID 表
ibv_devinfo -d mlx5_0 -v  # 查看 GID 列表

# 4. 测试 RDMA 连通性
ib_write_bw -d mlx5_0       # Server
ib_write_bw -d mlx5_0 <server_ip>  # Client

# 5. 检查设备权限
ls -la /dev/infiniband/uverbs*

# 6. 检查 NUMA 亲和性
cat /sys/class/infiniband/mlx5_0/device/numa_node
```

### 元数据连接失败

```bash
# etcd
etcdctl --endpoints=http://etcd1:2379 endpoint health

# Redis
redis-cli -h redis1 -p 6379 ping

# HTTP
curl http://master:8080/health
```

### Master 连接失败

```bash
# 检查 Master 进程
ps aux | grep mooncake_master

# 检查 RPC 端口
ss -tlnp | grep 50051

# 检查网络连通性
telnet master 50051
```

### 性能低于预期

```bash
# 1. 检查是否真正使用 RDMA
# 查看日志中的 transport 初始化信息
export MC_LOG_LEVEL=INFO

# 2. 检查 NUMA 绑定
numactl --hardware
cat /proc/<pid>/status | grep Cpus_allowed

# 3. 检查 RDMA 统计
ibstat -d mlx5_0
perfquery -d mlx5_0

# 4. 检查 GPU-NIC 亲和性
nvidia-smi topo -m

# 5. 调整切片大小
export MC_SLICE_SIZE=262144  # 256KB
```

### 常见错误码

| 错误 | 原因 | 解决方案 |
|------|------|---------|
| `NO_AVAILABLE_HANDLE` | 无可用缓冲区 | 增加 segment 大小或 Worker 节点 |
| `Error from etcd client` | etcd 不可达 | 检查 etcd 健康状态和网络连通性 |
| `No matched device found` | 无 RDMA NIC | 检查 `ibv_devices`，或设置 `MC_FORCE_TCP` |
| `Failed to register memory` | MR 注册失败 | 检查 `max_mr_size`、设备权限、内存锁定限制 |
| `RNR retry count exceeded` | 对端未 post receive | 检查对端 Worker 线程是否正常运行 |

---

## 附录 A：集群拓扑生成工具

`scripts/generate_cluster_topology.py` 可自动生成最优 NIC 分区匹配：

```bash
python3 scripts/generate_cluster_topology.py \
  --src-host node1 \
  --dst-host node2 \
  --file cluster-topology.json
```

该工具：
1. 通过 SSH 在两端执行 `ibv_devices` 发现 RDMA 设备
2. 使用 `ib_write_bw` 测量所有 NIC 对之间的带宽
3. 使用 `ib_read_lat` 测量延迟
4. 使用匈牙利算法（`scipy.optimize.linear_sum_assignment`）求解最优分区匹配
5. 输出 JSON 格式的拓扑描述（带宽/延迟/NUMA 亲和性）

---

## 附录 B：TENT 配置参考

TENT（下一代 Transfer Engine）使用 JSON 配置文件：

```json
{
    "metadata_type": "p2p",
    "metadata_servers": "127.0.0.1:2379",
    "topology": {
        "rdma_whitelist": ["mlx5_0", "mlx5_2"],
        "rdma_blacklist": []
    },
    "transports": {
        "rdma": {
            "enable": true,
            "device": { "max_cqe": 4096, "port": 1, "gid_index": 0 },
            "endpoint": {
                "max_qp_wr": 256, "max_sge": 4, "path_mtu": 4096,
                "max_rd_atomic": 16, "send_retry_count": 7
            },
            "workers": { "max_retry_count": 8, "block_size": 65536 }
        },
        "tcp": {
            "enable": true,
            "max_retry_count": 3,
            "max_concurrent_tasks": 16
        }
    }
}
```

启用 TENT：
```bash
export MC_USE_TENT=1
# 或
export MC_USE_TEV1=1
```