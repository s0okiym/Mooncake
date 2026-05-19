# Mooncake RDMA Transport 数据路径与控制路径详解

本文档基于源码逐行追踪 Mooncake RDMA Transport 的完整数据路径和控制路径，涵盖从应用程序调用到 RDMA 硬件操作的全过程。

---

## 1. 典型应用场景

一个典型的 RDMA 传输场景：**节点 A 向节点 B 的已注册内存写入数据**。

```
节点 A (Initiator)                          节点 B (Target)
┌──────────────────────┐                   ┌──────────────────────┐
│  TransferEngine       │                   │  TransferEngine       │
│  ├─ RDMA Transport   │  RDMA Write       │  ├─ RDMA Transport   │
│  ├─ 内存区域 (src)    │ ────────────────> │  ├─ 内存区域 (dst)    │
│  └─ Batch 管理       │                   │  └─ 被动接收          │
└──────────────────────┘                   └──────────────────────┘
         │                                          │
    元数据存储 (etcd/Redis/HTTP)  <──段描述符发布/查询──>
```

完整生命周期分为以下几个阶段：

1. **初始化与资源准备**（控制路径）
2. **内存注册与段描述符发布**（控制路径）
3. **连接建立**（控制路径）
4. **数据传输**（数据路径）
5. **状态查询与完成通知**（数据路径 + 控制路径）
6. **错误处理与重试**（控制路径）
7. **资源销毁**（控制路径）

---

## 2. 阶段一：初始化与资源准备

### 2.1 TransferEngine 初始化

**调用链**: `TransferEngine::init()` → `TransferEngineImpl::init()`

```mermaid
flowchart TD
    A["TransferEngine::init(<br/>metadata_conn_string,<br/>local_server_name, ...)"] --> B["TransferEngineImpl::init()"]
    B --> C["setFilesLimit():<br/>设置文件描述符上限"]
    C --> D["解析 local_server_name<br/>为 host:port"]
    D --> E{"连接模式?"}
    E -->|"legacy/P2P"| F["使用指定 IP:Port"]
    E -->|"new RPC"| G["自动发现 IP,<br/>随机选择可用端口"]
    F --> H["创建 TransferMetadata<br/>(解析 conn_string 协议)"]
    G --> H
    H --> I["创建 MultiTransport"]
    I --> J["metadata_->addRpcMetaEntry()<br/>注册本地 RPC 元信息"]
    J --> K["拓扑发现:<br/>local_topology_->discover()"]
    K --> L{"存在 HCA 且<br/>未强制 TCP?"}
    L -->|"是"| M["installTransport('rdma')"]
    L -->|"否"| N["installTransport('tcp')"]
    M --> O["RdmaTransport::install()"]
    N --> P["TcpTransport::install()"]
```

**源码关键行** (`transfer_engine_impl.cpp`):

- 行 84-88: `setFilesLimit()` 设置 fd 上限
- 行 114: `parseHostNameWithPort()` 解析 `local_server_name`
- 行 120-180: RPC 绑定方式选择（legacy/P2P vs 随机端口）
- 行 192: `TransferMetadata` 创建，解析连接字符串协议
- 行 199: `MultiTransport` 创建
- 行 202: `addRpcMetaEntry()` 注册本地 RPC 元信息到元数据存储
- 行 236-251: 拓扑发现（自动或加载自定义 JSON）
- 行 305-330: 安装 RDMA Transport（当有 HCA 时）

### 2.2 RdmaTransport 安装

**调用链**: `MultiTransport::installTransport("rdma")` → `RdmaTransport::install()`

```mermaid
flowchart TD
    A["installTransport('rdma', topo)"] --> B["new RdmaTransport()"]
    B --> C["RdmaTransport::install()"]
    C --> D["initializeRdmaResources()"]
    D --> E["对拓扑中每个 HCA:<br/>创建 RdmaContext"]
    E --> F["RdmaContext::construct():<br/>ibv_open_device → ibv_alloc_pd →<br/>创建 CQ → 创建 WorkerPool"]
    F --> G["allocateLocalSegmentID():<br/>创建 SegmentDesc(protocol='rdma')<br/>填充 devices(lid, gid)"]
    G --> H["startHandshakeDaemon():<br/>注册 onSetupRdmaConnections 回调<br/>启动 TCP 握手守护进程"]
    H --> I["updateLocalSegmentDesc():<br/>发布段描述符到元数据存储"]
```

**源码关键行** (`rdma_transport.cpp`):

- 行 60-80: 构造函数检查 `ibv_reg_mr_iova2` 符号，判断是否支持 IBV Relaxed Ordering
- 行 92-130: `install()` 四步初始化
- 行 651-672: `initializeRdmaResources()` — 遍历 HCA 列表创建 Context，失败的设备被 `disableDevice()` 排除
- 行 360-381: `allocateLocalSegmentID()` — 构建 SegmentDesc，包含所有设备的 lid/gid
- 行 674-679: `startHandshakeDaemon()` — 将 `onSetupRdmaConnections` 绑定为握手回调

### 2.3 RdmaContext 构造

**调用链**: `RdmaContext::construct()`

```mermaid
flowchart TD
    A["RdmaContext::construct(<br/>num_cq, num_comp_channels,<br/>port, gid_index, max_cqe, ...)"] --> B["创建 EndpointStore<br/>(FIFO 或 SIEVE)"]
    B --> C["ibv_open_device():<br/>打开 RDMA 设备"]
    C --> D["ibv_alloc_pd():<br/>分配保护域"]
    D --> E["查询端口属性<br/>ibv_query_port()"]
    E --> F["自动选择 GID 索引:<br/>优先选择有网络设备的 GID"]
    F --> G["创建完成通道<br/>ibv_create_comp_channel()"]
    G --> H["创建完成队列<br/>ibv_create_cq()"]
    H --> I["epoll 注册 async_fd"]
    I --> J["创建 WorkerPool<br/>启动 TransferWorker + MonitorWorker"]
```

**GID 索引自动选择逻辑**:
1. 遍历所有 GID 索引（0 到 gid_tbl_len-1）
2. 对每个 GID，检查其关联的网络设备（通过 sysfs 路径 `/sys/class/infiniband/<dev>/device/net/`）
3. 优先选择有网络设备的 GID（`GID_WITH_NETWORK`）
4. 如果用户指定了 `MC_GID_INDEX`，则使用指定值

---

## 3. 阶段二：内存注册与段描述符发布

### 3.1 内存注册

**调用链**: `TransferEngine::registerLocalMemory()` → `TransferEngineImpl::registerLocalMemory()` → `RdmaTransport::registerLocalMemoryInternal()`

```mermaid
flowchart TD
    A["TransferEngineImpl::<br/>registerLocalMemory(addr, length, location)"] --> B["checkOverlap():<br/>检查与已注册区域重叠"]
    B --> C["遍历所有 Transport:<br/>transport->registerLocalMemory()"]
    C --> D["RdmaTransport::<br/>registerLocalMemoryInternal()"]
    D --> E{"内存 >= 4GB 且<br/>CPU 核数 >= 4?"}
    E -->|"是"| F["preTouchMemory():<br/>多线程预触碰内存<br/>加速 MR 注册"]
    E -->|"否"| G["跳过预触碰"]
    F --> H{"并行注册?"}
    G --> H
    H -->|"并行"| I["多线程: 每个 Context<br/>独立调用 registerMemoryRegion()"]
    H -->|"串行"| J["逐个 Context 调用<br/>registerMemoryRegion()"]
    I --> K["收集所有 Context 的<br/>lkey/rkey 到 BufferDesc"]
    J --> K
    K --> L{"location == '*'?"}
    L -->|"是"| M["getMemoryLocation():<br/>自动检测 NUMA/GPU 位置"]
    L -->|"否"| N["使用指定 location"]
    M --> O["metadata_->addLocalMemoryBuffer():<br/>添加到本地段描述符<br/>+ 更新元数据存储"]
    N --> O
```

**源码关键行** (`rdma_transport.cpp` 行 182-303):

- 行 189-196: 访问权限 = `LOCAL_WRITE | REMOTE_WRITE | REMOTE_READ` + 可选 `RELAXED_ORDERING`
- 行 197-199: 预触碰条件: 内存 ≥ 4GB 且 CPU ≥ 4 核
- 行 217-223: 并行注册决策: `MC_ENABLE_PARALLEL_REG_MR` 或自动（多 Context + 预触碰完成时）
- 行 278-281: 收集每个 Context 的 lkey/rkey
- 行 285-293: 自动检测内存位置（通配符 `*` 时）

### 3.2 RdmaContext::registerMemoryRegion() 内部流程

```mermaid
flowchart TD
    A["registerMemoryRegion(addr, length, access)"] --> B["获取写锁<br/>(RWSpinlock)"]
    B --> C{"地址已注册?"}
    C -->|"是"| D["返回已有 MR 的 key"]
    C -->|"否"| E{"GPU 内存<br/>(无 nvidia-peermem)?"}
    E -->|"是"| F["ibv_reg_dmabuf_mr():<br/>通过 DMA-BUF 注册 GPU 内存"]
    E -->|"否"| G["ibv_reg_mr():<br/>注册普通内存"]
    F --> H["存入 memory_region_map_<br/>key = addr"]
    G --> H
    H --> I["释放写锁"]
```

### 3.3 段描述符发布与同步

内存注册完成后，`addLocalMemoryBuffer()` 会更新本地 SegmentDesc（添加 BufferDesc），并通过 `update_metadata=true` 参数将最新描述符写入元数据存储。

**远端节点查询段描述符**（`openSegment` 时触发）:
1. `openSegment(peer_name)` → `metadata_->getSegmentID(peer_name)`
2. `getSegmentID()` → `getSegmentDescByName()` — 先查本地缓存，未命中则从元数据存储获取
3. 获取的 SegmentDesc 包含远端的 devices（lid, gid）、buffers（addr, length, rkey 列表）、topology

---

## 4. 阶段三：连接建立（控制路径核心）

连接建立是按需（on-demand）的：首次向某个远端 NIC 发送数据时才建立 QP 连接。

### 4.1 完整连接建立流程

```mermaid
sequenceDiagram
    participant APP as "应用程序"
    participant WP as "WorkerPool<br/>(节点A)"
    participant CTX_A as "RdmaContext<br/>(节点A, mlx5_0)"
    participant EP_A as "RdmaEndPoint<br/>(节点A)"
    participant MS as "元数据存储"
    participant HS as "TCP 握手<br/>(SocketHandShake)"
    participant EP_B as "RdmaEndPoint<br/>(节点B)"
    participant CTX_B as "RdmaContext<br/>(节点B, mlx5_0)"

    Note over APP,CTX_B: "首次传输时触发连接建立"

    WP->>CTX_A: "1. context_->endpoint(peer_nic_path)<br/>获取/创建 EndPoint"
    WP->>EP_A: "2. setupConnectionsByActive()"

    Note over EP_A: "3. 获取写锁"
    Note over EP_A: "4. status: UNCONNECTED → CONNECTING"
    Note over EP_A: "5. 准备 local_desc:<br/>local_nic_path, lid, gid, qp_num"

    EP_A->>MS: "6. 查询 peer RPC meta<br/>(获取 IP:Port)"
    MS-->>EP_A: "7. 返回 IP:Port"

    EP_A->>HS: "8. TCP RPC: 发送 HandShakeDesc<br/>(local_nic_path, lid, gid,<br/>peer_nic_path, qp_num)"

    HS->>CTX_B: "9. onSetupRdmaConnections()<br/>按 NIC 名查找 RdmaContext"
    CTX_B->>EP_B: "10. context_->endpoint(local_nic_path)<br/>获取/创建 EndPoint"

    EP_B->>EP_B: "11. setupConnectionsByPassive()"
    Note over EP_B: "12. 准备 local_desc 响应"

    EP_B->>EP_B: "13. doSetupConnection():<br/>QP: →RESET→INIT→RTR→RTS"

    EP_B-->>HS: "14. 返回 local_desc<br/>(lid, gid, qp_num)"
    HS-->>EP_A: "15. RPC 返回 peer_desc"

    Note over EP_A: "16. 重新获取写锁"
    EP_A->>EP_A: "17. doSetupConnection():<br/>QP: →RESET→INIT→RTR→RTS"

    Note over EP_A: "18. status: CONNECTING → CONNECTED"
    Note over EP_B: "19. status 已设为 CONNECTED"
```

### 4.2 QP 状态转换详解

`doSetupConnection()` (行 710-838) 对每个 QP 执行四步状态转换：

```mermaid
flowchart LR
    A["任意状态"] -->|"ibv_modify_qp(QPS_RESET)"| B["RESET"]
    B -->|"port_num, pkey_index,<br/>access_flags"| C["INIT"]
    C -->|"path_mtu, dest_qp_num,<br/>dgid, sgid_index, dlid,<br/>max_dest_rd_atomic=16,<br/>min_rnr_timer=12"| D["RTR"]
    D -->|"timeout=14, retry_cnt=7,<br/>rnr_retry=7, sq_psn=0,<br/>max_rd_atomic=16"| E["RTS"]
```

**各步骤详细参数**:

| 转换 | 关键参数 | 说明 |
|------|----------|------|
| →RESET | `IBV_QP_STATE` | 清除 QP 状态 |
| →INIT | `port_num`, `pkey_index=0`, `access_flags=LOCAL_WRITE\|REMOTE_READ\|REMOTE_WRITE\|REMOTE_ATOMIC` | 初始化端口和权限 |
| →RTR | `path_mtu`(受 `MC_MTU` 上限约束), `dest_qp_num`, `dgid`(对端 GID), `sgid_index`, `dlid`(对端 LID), `hop_limit=16`, `is_global=1`, `max_dest_rd_atomic=16`, `min_rnr_timer=12` | 配置接收路径 |
| →RTS | `timeout=14`, `retry_cnt=7`, `rnr_retry=7`, `sq_psn=0`, `max_rd_atomic=16` | 配置发送路径 |

**流量类别** (行 792-795): 如果 `MC_IB_TC >= 0`，设置 `ah_attr.grh.traffic_class`，用于 RoCEv2 的 DSCP/QoS 标记。

### 4.3 并发握手保护

当多个线程同时触发同一 Endpoint 的连接建立时：

```mermaid
flowchart TD
    A["线程1: setupConnectionsByActive()"] --> B["获取写锁"]
    B --> C{"status == UNCONNECTED?"}
    C -->|"是"| D["status → CONNECTING<br/>设置 do_rpc = true"]
    C -->|"否 (CONNECTING)"| E["释放锁，自旋等待<br/>指数退避(1μs→1ms)"]
    D --> F["释放锁，执行 RPC"]
    E --> G{"等待超时(10s)?"}
    G -->|"是"| H["重置连接<br/>返回 ERR_ENDPOINT"]
    G -->|"否"| I{"status == CONNECTED?"}
    I -->|"是"| J["返回成功"]
    I -->|"否"| H
```

**同时打开 (Simultaneous Open)**: 如果 A→B 和 B→A 的握手同时发生，被动方（先收到 RPC 的一方）会先完成 `setupConnectionsByPassive()`，将 Endpoint 设为 CONNECTED。主动方 RPC 返回后，发现已经 CONNECTED，且 QP 号一致时复用现有连接。

---

## 5. 阶段四：数据传输（数据路径核心）

### 5.1 完整数据路径总览

```mermaid
flowchart TD
    A["应用: submitTransfer(batch_id, requests)"] --> B["TransferEngineImpl::<br/>multi_transports_->submitTransfer()"]
    B --> C["MultiTransport::submitTransfer()"]
    C --> D["selectTransport():<br/>查询目标段 protocol → 'rdma'<br/>获取 RdmaTransport*"]
    D --> E["RdmaTransport::<br/>submitTransferTask(task_list)"]
    E --> F["切片 + 设备选择<br/>(见 5.2)"]
    F --> G["context->submitPostSend(slices)<br/>(见 5.3)"]
    G --> H["WorkerPool::submitPostSend()<br/>解析远端 rkey + peer_nic_path<br/>(见 5.4)"]
    H --> I["TransferWorker:<br/>performPostSend() + performPollCq()<br/>(见 5.5, 5.6)"]
    I --> J["应用: getTransferStatus()<br/>(见 6.1)"]
```

### 5.2 切片与设备选择

**源码**: `rdma_transport.cpp` 行 456-573 `submitTransferTask()`

```mermaid
flowchart TD
    A["submitTransferTask(task_list)"] --> B["对每个 task/request:"]
    B --> C["selectDevice(local_segment_desc,<br/>source_addr, length,<br/>buffer_id, device_id)"]
    C --> D["遍历本地段 buffers:<br/>地址范围匹配 source_addr"]
    D --> E["解析 buffer.name 为 location<br/>(如 'segments:4096:0,1' → 'cpu:N')"]
    E --> F["topology.selectDevice(location):<br/>NUMA 感知选择 HCA"]
    F --> G["获得 device_id → context_list_[device_id]"]

    G --> H["按 kBlockSize(65536) 切片:<br/>offset = 0 → length, step = kBlockSize"]
    H --> I["每个 Slice 填充:<br/>source_addr = source + offset<br/>length = min(kBlockSize, remaining)<br/>dest_addr = target_offset + offset<br/>opcode, target_id, retry_cnt, max_retry_cnt"]
    I --> J["slice->rdma.source_lkey =<br/>local_segment_desc->buffers[buffer_id].lkey[device_id]"]
    J --> K["按 Context 分组:<br/>slices_to_post[context].push_back(slice)"]
    K --> L["__sync_fetch_and_add(&task.slice_count, 1)"]
    L --> M{"达到水位线?<br/>(max_wr * num_qp_per_ep)"}
    M -->|"是"| N["context->submitPostSend(slices)"]
    M -->|"否"| O["继续切片"]
    N --> O
    O --> P["最终: 提交剩余 slices"]
```

**selectDevice 详解** (行 684-727):

1. 遍历 SegmentDesc 的 buffers 数组
2. 对每个 buffer，检查 `offset` 是否在 `[buffer.addr, buffer.addr + buffer.length)` 范围内
3. 解析 buffer.name：
   - 普通格式 `"cpu:0"` → 直接使用
   - 分段格式 `"segments:4096:0,1"` → 调用 `resolveSegmentsLocation()` 根据偏移计算实际 NUMA 节点
4. 调用 `topology.selectDevice(location, retry_count)`:
   - `retry_count=0`: 从 preferred_hca 随机选择（或 `MC_PATH_ROUNDROBIN` 轮询）
   - `retry_count>0`: 确定性循环遍历 preferred+available（用于故障重试）
5. 回退: 若指定 location 无设备，尝试通配符 `"*"` location

**碎片合并优化** (行 491-492): 当剩余数据 ≤ `kBlockSize + kFragmentSize` 时，合并为一个 Slice，避免产生极小的尾 Slice。

### 5.3 Context 级提交

**源码**: `rdma_transport.cpp` 行 571-572

```cpp
for (auto &entry : slices_to_post)
    if (!entry.second.empty()) entry.first->submitPostSend(entry.second);
```

直接调用 `RdmaContext::submitPostSend()`，委托给 `WorkerPool::submitPostSend()`。

### 5.4 WorkerPool 提交 — 解析远端信息

**源码**: `worker_pool.cpp` 行 60-171

```mermaid
flowchart TD
    A["WorkerPool::submitPostSend(slice_list)"] --> B["查询目标段 SegmentDesc:<br/>getSegmentDescByID(target_id)"]
    B --> C["对每个 Slice:"]
    C --> D["selectDevice(peer_segment_desc,<br/>dest_addr, length, hint,<br/>buffer_id, device_id)"]
    D --> E["slice->rdma.dest_rkey =<br/>peer_segment_desc->buffers[buffer_id].rkey[device_id]"]
    E --> F["slice->peer_nic_path =<br/>MakeNicPath(segment_name,<br/>devices[device_id].name)"]
    F --> G["按 shard_id 分发到<br/>8 个分片队列:<br/>shard_id = (target_id*10007+device_id) % 8"]
    G --> H["通知休眠的 TransferWorker"]
```

**关键数据转换**:

| Slice 字段 | 来源 | 说明 |
|------------|------|------|
| `source_addr` | `request.source + offset` | 本地源地址 |
| `rdma.source_lkey` | `local_segment_desc->buffers[buf_id].lkey[dev_id]` | 本地 MR 的 lkey |
| `rdma.dest_addr` | `request.target_offset + offset` | 远端目标地址 |
| `rdma.dest_rkey` | `peer_segment_desc->buffers[buf_id].rkey[dev_id]` | 远端 MR 的 rkey |
| `peer_nic_path` | `MakeNicPath(peer_name, peer_dev_name)` | 对端 NIC 标识 |
| `rdma.retry_cnt` | `request.advise_retry_cnt` | 当前重试计数 |
| `rdma.max_retry_cnt` | `globalConfig().retry_cnt` (默认9) | 最大重试次数 |

### 5.5 TransferWorker — Post Send

**源码**: `worker_pool.cpp` 行 173-267 `performPostSend()`

```mermaid
flowchart TD
    A["performPostSend(thread_id)"] --> B["排空分片队列到本地队列:<br/>shard_id = thread_id, thread_id+k, ..."]
    B --> C{"有 redispatch 标记?"}
    C -->|"是"| D["redispatch(): 重新选择设备<br/>重新填充 dest_rkey, peer_nic_path"]
    C -->|"否"| E["对本地队列每个 peer_nic_path:"]
    E --> F["context_.endpoint(peer_nic_path)<br/>获取/创建 RdmaEndPoint"]
    F --> G{"endpoint 活跃?"}
    G -->|"否"| H["加入 failed_slice_list"]
    G -->|"是"| I{"endpoint 已连接?"}
    I -->|"否"| J["setupConnectionsByActive()<br/>触发连接建立"]
    J --> K{"连接成功?"}
    K -->|"否"| L["标记 inactive,<br/>加入 failed_slice_list"]
    K -->|"是"| M["endpoint->submitPostSend()"]
    I -->|"是"| M
    M --> N["处理 failed_slice_list:<br/>retry_cnt++ → redispatch()"]
```

### 5.6 RdmaEndPoint — 构建 ibv_send_wr 并提交

**源码**: `rdma_endpoint.cpp` 行 578-661 `submitPostSend()`

```mermaid
flowchart TD
    A["RdmaEndPoint::submitPostSend(<br/>slice_list, failed_slice_list)"] --> B["获取写锁"]
    B --> C["计算可用资源:<br/>cq_remaining = max_cqe - cq_outstanding<br/>qp_avail = max_wr_depth - wr_depth[qp]"]
    C --> D["按 QP 轮询分配 Slice:<br/>chunk = ceil(remaining_slices / remaining_qps)<br/>wr_count = min(chunk, qp_avail, cq_remaining)"]
    D --> E["对每个 Slice 构建 ibv_send_wr:"]
    E --> F["sge.addr = slice->source_addr<br/>sge.length = slice->length<br/>sge.lkey = slice->rdma.source_lkey"]
    F --> G["wr.wr_id = (uint64_t)slice<br/>wr.opcode = READ→IBV_WR_RDMA_READ<br/>         WRITE→IBV_WR_RDMA_WRITE<br/>wr.send_flags = IBV_SEND_SIGNALED<br/>wr.wr.rdma.remote_addr = slice->rdma.dest_addr<br/>wr.wr.rdma.rkey = slice->rdma.dest_rkey"]
    G --> H["slice->ts = getCurrentTimeInNano()<br/>slice->status = POSTED<br/>slice->rdma.qp_depth = &wr_depth_list_[qp_index]"]
    H --> I["__sync_fetch_and_add(wr_depth, wr_count)<br/>__sync_fetch_and_add(cq_outstanding, wr_count)"]
    I --> J["ibv_post_send(qp, wr_list, &bad_wr)"]
    J --> K{"成功?"}
    K -->|"否"| L["遍历 bad_wr 链:<br/>调整 wr_depth, cq_outstanding<br/>加入 failed_slice_list"]
    K -->|"是"| M["返回成功"]
```

**ibv_send_wr 关键映射**:

```
Slice 字段            →  ibv_send_wr / ibv_sge 字段
─────────────────────────────────────────────────
source_addr           →  sge.addr
length                →  sge.length
rdma.source_lkey      →  sge.lkey
opcode (READ/WRITE)   →  wr.opcode (IBV_WR_RDMA_READ/IBV_WR_RDMA_WRITE)
rdma.dest_addr        →  wr.wr.rdma.remote_addr
rdma.dest_rkey        →  wr.wr.rdma.rkey
(slice 指针)          →  wr.wr_id (完成时用于回查)
```

### 5.7 TransferWorker — Poll CQ 完成处理

**源码**: `worker_pool.cpp` 行 269-350 `performPollCq()`

```mermaid
flowchart TD
    A["performPollCq(thread_id)"] --> B["轮询分配给本线程的 CQ:<br/>ibv_poll_cq(kPollCount=64)"]
    B --> C{"完成状态?"}
    C -->|"IBV_WC_SUCCESS"| D["slice->markSuccess():<br/>原子递增 task->transferred_bytes<br/>原子递增 task->success_slice_count<br/>check_batch_completion()"]
    C -->|"IBV_WC_WR_FLUSH_ERR"| E["slice->markFailed():<br/>(QP→ERR 期间正常)<br/>不重试, 不计入 RNIC 错误"]
    C -->|"其他错误"| F["failed_nr_polls++"]
    F --> G{"连续失败 > 32<br/>且无成功?"}
    G -->|"是"| H["标记 Context inactive"]
    G -->|"否"| I["context_.deleteEndpoint()"]
    I --> J["slice->rdma.retry_cnt++"]
    J --> K{"retry_cnt >=<br/>max_retry_cnt?"}
    K -->|"是"| L["slice->markFailed()"]
    K -->|"否"| M["redispatch():<br/>重新选择设备, 重新入队"]
    D --> N["__sync_fetch_and_sub(wr_depth, 1)<br/>__sync_fetch_and_sub(cq_outstanding, 1)"]
    E --> N
```

### 5.8 markSuccess/markFailed — 原子状态更新

**源码**: `transport.h` Slice 内联方法

```mermaid
flowchart TD
    A["slice->markSuccess()"] --> B["status = SUCCESS"]
    B --> C["__sync_fetch_and_add(<br/>&task->transferred_bytes, slice->length)"]
    C --> D["__sync_fetch_and_add(<br/>&task->success_slice_count, 1)"]
    D --> E["check_batch_completion(false)"]
    E --> F{"USE_EVENT_DRIVEN_COMPLETION?"}
    F -->|"是"| G["原子递增 completed_slice_count<br/>最后一个 slice 完成时:<br/>task->is_finished = true<br/>原子递增 batch_desc.finished_task_count<br/>最后一个 task 完成时:<br/>通知 completion_cv"]
    F -->|"否"| H["返回"]
```

### 5.9 数据路径完整时序图

```mermaid
sequenceDiagram
    participant APP as "应用线程"
    participant MT as "MultiTransport"
    participant RT as "RdmaTransport"
    participant WP as "WorkerPool"
    participant EP as "RdmaEndPoint"
    participant NIC as "RDMA NIC"
    participant CQ as "Completion Queue"

    rect rgb(230, 245, 255)
    Note over APP,CQ: "数据路径 — 提交阶段"
    APP->>MT: "1. submitTransfer(batch_id, requests)"
    MT->>MT: "2. selectTransport(): 查目标段 protocol→rdma"
    MT->>RT: "3. submitTransferTask(task_list)"
    RT->>RT: "4. 切片: request→Slices (kBlockSize=64KB)"
    RT->>RT: "5. selectDevice(): 地址范围匹配→buffer_id, NUMA→device_id"
    RT->>RT: "6. 填充 Slice: source_lkey, dest_addr, target_id"
    RT->>WP: "7. context->submitPostSend(slices)"
    WP->>WP: "8. 查远端段描述符→dest_rkey, peer_nic_path"
    WP->>WP: "9. 分发到 8 个分片队列, 通知 Worker"
    end

    rect rgb(255, 240, 230)
    Note over WP,CQ: "数据路径 — 工作线程阶段"
    WP->>EP: "10. performPostSend(): endpoint->submitPostSend()"
    EP->>EP: "11. 构建 ibv_send_wr 链:<br/>sge{addr, length, lkey}<br/>wr{opcode, remote_addr, rkey}"
    EP->>NIC: "12. ibv_post_send(qp, wr_list)<br/>[RDMA Write / RDMA Read]"
    NIC->>NIC: "13. 硬件执行零拷贝远程内存访问"
    NIC->>CQ: "14. 完成事件入队"
    WP->>CQ: "15. performPollCq(): ibv_poll_cq()"
    CQ-->>WP: "16. 返回 ibv_wc (wr_id=slice指针)"
    WP->>WP: "17. slice->markSuccess():<br/>原子更新 transferred_bytes"
    end

    rect rgb(230, 255, 230)
    Note over APP,WP: "数据路径 — 查询阶段"
    APP->>MT: "18. getTransferStatus(batch_id, task_id)"
    MT->>MT: "19. 检查 task: success+failed == slice_count?"
    MT-->>APP: "20. 返回 COMPLETED / WAITING / FAILED"
    end
```

---

## 6. 阶段五：状态查询与通知

### 6.1 状态查询

**调用链**: `TransferEngine::getTransferStatus()` → `TransferEngineImpl::getTransferStatus()` → `MultiTransport::getTransferStatus()`

```mermaid
flowchart TD
    A["getTransferStatus(batch_id, task_id)"] --> B["读取 task.transferred_bytes"]
    B --> C["success_count = task.success_slice_count<br/>failed_count = task.failed_slice_count"]
    C --> D{"success + failed<br/>== slice_count?"}
    D -->|"是, failed>0"| E["status = FAILED<br/>task.is_finished = true"]
    D -->|"是, failed==0"| F["status = COMPLETED<br/>task.is_finished = true"]
    D -->|"否"| G{"slice_timeout > 0<br/>且存在超时 Slice?"}
    G -->|"是"| H["status = TIMEOUT"]
    G -->|"否"| I["status = WAITING"]
```

**超时检测** (行 210-223): 遍历 task 的所有 Slice，检查 `slice->ts`（提交时间戳）与当前时间的差值是否超过 `MC_SLICE_TIMEOUT` 秒。

### 6.2 通知机制

**调用链**: `submitTransferWithNotify()` → 传输完成后自动发送通知

```mermaid
sequenceDiagram
    participant A as "发送方 (Initiator)"
    participant B as "接收方 (Target)"

    A->>A: "1. submitTransferWithNotify(<br/>batch_id, requests, notify_msg)"
    Note over A: "2. 存储 notifies_to_send_[batch_id] =<br/>{target_id, notify_msg}"

    A->>A: "3. 轮询 getTransferStatus() → COMPLETED"
    A->>A: "4. getBatchTransferStatus():<br/>所有 task 完成 → 发送 notify"

    A->>B: "5. metadata_->sendNotify():<br/>TCP RPC 发送 NotifyDesc<br/>{name, notify_msg}"
    B->>B: "6. metadata_->getNotifies():<br/>从本地 notifys 队列取出"
```

**源码** (`transfer_engine_impl.h` 行 133-159): `submitTransferWithNotify()` 在调用 `multi_transports_->submitTransfer()` 后，将 `{target_id, notify_msg}` 存入 `notifies_to_send_` 映射。当 `getTransferStatus()` 检测到 COMPLETED 时，调用 `sendNotifyByID()` 通过 TCP RPC 发送通知。

---

## 7. 阶段六：错误处理与重试

### 7.1 错误处理层级

```mermaid
flowchart TD
    subgraph "L1: Slice 级"
        S1["CQ 轮询错误"] --> S2["retry_cnt++"]
        S2 --> S3{"retry_cnt >= max_retry_cnt?"}
        S3 -->|"是"| S4["slice->markFailed()"]
        S3 -->|"否"| S5["redispatch():<br/>重新选择设备→重新入队"]
    end

    subgraph "L2: Endpoint 级"
        E1["IBV_EVENT_QP_FATAL"] --> E2["endpoint->set_active(false)"]
        E2 --> E3["后续 Slice 走 redispatch<br/>到其他 Endpoint"]
        E4["连续握手失败"] --> E5["endpoint->set_active(false)"]
    end

    subgraph "L3: Context 级"
        C1["IBV_EVENT_DEVICE_FATAL<br/>PORT_ERR / CQ_ERR / WQ_FATAL"] --> C2["context->set_active(false)"]
        C2 --> C3["disconnectAllEndpoints()"]
        C4["连续 CQ 错误 > 32<br/>(无成功)"] --> C5["context->set_active(false)"]
        C6["IBV_EVENT_PORT_ACTIVE"] --> C7["context->set_active(true)<br/>(自动恢复)"]
    end
```

### 7.2 Redispatch 流程

**源码**: `worker_pool.cpp` 行 352-389

```mermaid
flowchart TD
    A["redispatch(slice_list, thread_id)"] --> B["重新获取目标段描述符<br/>(force_update=true)"]
    B --> C["对每个 Slice:"]
    C --> D["selectDevice(peer_segment_desc,<br/>dest_addr, length,<br/>buffer_id, device_id,<br/>retry_cnt)"]
    D --> E["retry_cnt>0 时:<br/>确定性循环选择<br/>不同的 HCA 设备"]
    E --> F["更新 dest_rkey, peer_nic_path"]
    F --> G["重新入队到<br/>collective_slice_queue_"]
```

**关键**: `selectDevice()` 在 `retry_cnt > 0` 时采用确定性循环策略，确保重试时选择不同的 HCA，实现故障路径切换。

### 7.3 两阶段 Endpoint 销毁

```mermaid
flowchart TD
    A["Endpoint 被标记删除"] --> B["beginDestroy():<br/>active_=false, status_=DESTROYING<br/>QP → ERR 状态<br/>(硬件刷出 inflight WR)"]
    B --> C["performPollCq():<br/>收到 WR_FLUSH_ERR 完成<br/>slice->markFailed()"]
    C --> D["MonitorWorker 周期调用:<br/>context_->reclaimEndpoints()"]
    D --> E["finishDestroy():<br/>等待 wr_depth == 0<br/>(30s 超时)"]
    E --> F["deconstructLocked():<br/>ibv_destroy_qp()<br/>释放资源"]
    F --> G["status_ = DESTROYED"]
```

**死锁避免** (行 423-494): 在 `doProcessContextEvents()` 中，先 `ibv_ack_async_event()` 再调用 `set_active(false)` 或 `disconnectAllEndpoints()`。因为 `ibv_destroy_qp` 可能阻塞等待 event ack，而 `disconnect()` 需要获取 endpoint 锁，如果顺序反了会死锁。

---

## 8. 阶段七：资源销毁

### 8.1 Engine 释放

```mermaid
flowchart TD
    A["TransferEngineImpl::freeEngine()"] --> B["metadata_->removeRpcMetaEntry():<br/>从元数据存储移除 RPC 信息"]
    B --> C["metadata_.reset():<br/>销毁 TransferMetadata<br/>(关闭握手守护线程)"]
    C --> D["multi_transports_.reset():<br/>销毁 MultiTransport"]
    D --> E["销毁 TransportMap 中所有 Transport"]
    E --> F["RdmaTransport::~RdmaTransport():<br/>removeSegmentDesc()<br/>清空 context_list_"]
    F --> G["~RdmaContext():<br/>WorkerPool 停止<br/>销毁 QP, CQ, PD<br/>ibv_close_device()"]
```

### 8.2 内存注销

```mermaid
flowchart TD
    A["unregisterLocalMemory(addr)"] --> B["遍历所有 Transport:<br/>transport->unregisterLocalMemory()"]
    B --> C["RdmaTransport::<br/>unregisterLocalMemoryInternal()"]
    C --> D["metadata_->removeLocalMemoryBuffer():<br/>从段描述符移除 BufferDesc<br/>更新元数据存储"]
    D --> E["遍历所有 Context:<br/>unregisterMemoryRegion(addr)"]
    E --> F["ibv_dereg_mr(mr)"]
```

---

## 9. MonitorWorker — 后台守护

**源码**: `worker_pool.cpp` 行 497-527

MonitorWorker 以 1 秒为周期运行，执行两项任务：

```mermaid
flowchart TD
    A["monitorWorker()"] --> B["周期性任务 (1s):"]
    B --> C["context_.set_active(true):<br/>尝试重新激活 Context<br/>(恢复被临时标记 inactive 的设备)"]
    B --> D["context_.reclaimEndpoints():<br/>回收 waiting_list_ 中的 Endpoint:<br/>beginDestroy() → finishDestroy()"]
    B --> E["epoll_wait(async_fd, 100ms timeout)"]
    E --> F{"有异步事件?"}
    F -->|"是"| G["doProcessContextEvents()"]
    F -->|"否"| H["继续循环"]
    G --> I{"事件类型?"}
    I -->|"QP_FATAL"| J["endpoint->set_active(false)"]
    I -->|"DEVICE_FATAL /<br/>CQ_ERR / WQ_FATAL /<br/>PORT_ERR / LID_CHANGE"| K["context->set_active(false)<br/>disconnectAllEndpoints()"]
    I -->|"PORT_ACTIVE"| L["context->set_active(true)"]
```

---

## 10. 控制路径与数据路径交互总览

```mermaid
flowchart TD
    subgraph "控制路径 (Control Plane)"
        INIT["初始化: init()"]
        TOPO["拓扑发现: discover()"]
        INSTALL["Transport 安装: install()"]
        REG["内存注册: registerLocalMemory()"]
        META_PUB["段描述符发布: updateLocalSegmentDesc()"]
        META_QUERY["段描述符查询: getSegmentDescByName()"]
        CONN["连接建立: setupConnectionsByActive/Passive()"]
        NOTIFY["通知: sendNotify/getNotifies()"]
        PROBE["探活: probePeerAliveByID()"]
        DESTROY["资源销毁: freeEngine()"]
    end

    subgraph "数据路径 (Data Plane)"
        SUBMIT["提交传输: submitTransfer()"]
        SLICE["切片: kBlockSize=64KB"]
        DEV_SEL["设备选择: selectDevice()"]
        POST["提交 WR: ibv_post_send()"]
        POLL["轮询完成: ibv_poll_cq()"]
        STATUS["状态更新: markSuccess/markFailed()"]
        QUERY["状态查询: getTransferStatus()"]
    end

    INIT --> TOPO --> INSTALL
    INSTALL --> REG --> META_PUB
    META_QUERY --> CONN
    CONN --> SUBMIT

    SUBMIT --> SLICE --> DEV_SEL --> POST --> POLL --> STATUS
    QUERY -.->|"读取 task 状态"| STATUS

    META_QUERY -.->|"为 POST 提供<br/>dest_rkey, peer_nic_path"| POST
    CONN -.->|"为 POST 提供<br/>QP 连接"| POST
    REG -.->|"为 POST 提供<br/>source_lkey"| POST
    META_PUB -.->|"远端查询<br/>段描述符"| META_QUERY

    STATUS -.->|"COMPLETED 触发"| NOTIFY
    POLL -.->|"错误触发<br/>redispatch"| DEV_SEL
```

---

## 11. 关键数据结构在路径中的流转

### 11.1 SegmentDesc 在各阶段的内容

```
初始化后 (allocateLocalSegmentID):
  name = "local_server_name"
  protocol = "rdma"
  devices = [{name: "mlx5_0", lid: 0x1234, gid: "fe80:..."}, ...]
  topology = {cpu:0 → preferred:[mlx5_0], ...}
  buffers = []  // 空，等待内存注册

内存注册后 (addLocalMemoryBuffer):
  buffers = [{addr: 0x7f000000, length: 1073741824,
              name: "cpu:0", lkey: [12345, 12346], rkey: [54321, 54322]}]
              // lkey/rkey 数组长度 = context_list_.size()

远端查询到:
  // 包含远端的 devices, buffers(含rkey), topology
  // 本地 WorkerPool 通过 rkey[device_id] 获取远端访问密钥
```

### 11.2 Slice 字段在各阶段的填充

| 阶段 | 填充的字段 |
|------|-----------|
| `submitTransferTask()` | `source_addr`, `length`, `opcode`, `target_id`, `rdma.dest_addr`, `rdma.retry_cnt`, `rdma.max_retry_cnt`, `task`, `status=PENDING` |
| `submitTransferTask()` | `rdma.source_lkey` (从本地段描述符的 `buffers[buffer_id].lkey[device_id]`) |
| `WorkerPool::submitPostSend()` | `rdma.dest_rkey` (从远端段描述符的 `buffers[buffer_id].rkey[device_id]`), `peer_nic_path` |
| `RdmaEndPoint::submitPostSend()` | `rdma.qp_depth` (指向 `wr_depth_list_[qp_index]`), `ts`, `status=POSTED` |
| `performPollCq()` | 通过 `markSuccess/markFailed` 更新 `task` 的原子计数器 |

---

## 12. 环境变量对数据路径的影响

| 环境变量 | 影响的数据路径阶段 | 效果 |
|----------|-------------------|------|
| `MC_SLICE_SIZE` | 切片 | 每个 Slice 的大小（默认 65536） |
| `MC_RETRY_CNT` | 错误重试 | 最大重试次数（默认 9） |
| `MC_MAX_WR` | WR 提交 | 每个 QP 的最大 WR 深度（默认 256） |
| `MC_NUM_QP_PER_EP` | 连接建立 | 每 Endpoint 的 QP 数量（默认 2） |
| `MC_WORKERS_PER_CTX` | 工作线程 | 每 Context 的 TransferWorker 数量（默认 2） |
| `MC_MAX_CQE_PER_CTX` | CQ 轮询 | CQ 容量，影响一次可提交的 WR 数量 |
| `MC_IB_TC` | QP 建立 | RoCEv2 流量类别/DSCP 标记 |
| `MC_ENDPOINT_STORE_TYPE` | Endpoint 管理 | FIFO 或 SIEVE（默认 SIEVE）淘汰策略 |
| `MC_SLICE_TIMEOUT` | 状态查询 | Slice 超时检测（秒） |
| `MC_ENABLE_DEST_DEVICE_AFFINITY` | 设备选择 | 启用目的端设备亲和性选择 |
| `MC_PATH_ROUNDROBIN` | 设备选择 | 首次选择使用轮询而非随机 |
| `MC_FORCE_TCP` | 初始化 | 强制使用 TCP 而非 RDMA |