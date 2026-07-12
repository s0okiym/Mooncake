# Mooncake 核心问题代码解析

> 针对学习 Mooncake 时容易"懂原理但缺细节"的四个核心问题，结合源码逐层解析。

---

## 问题一：Segment 怎么注册？生命周期谁管？卸载时怎么保证没未完成传输？

### 1. Segment 注册到 Transfer Engine

Transfer Engine 没有独立的 `registerSegment` 概念，而是通过 **注册本地内存（local memory）** 到 `LOCAL_SEGMENT_ID`，并把描述符发布到 metadata server。

**公共 API**

- 文件：`mooncake-transfer-engine/include/transfer_engine.h:101-114`
- 函数：`TransferEngine::registerLocalMemory` / `registerLocalMemoryBatch` / `openSegment`

**实现入口**

- 文件：`mooncake-transfer-engine/src/transfer_engine_impl.cpp:586-608`
- 函数：`TransferEngineImpl::registerLocalMemory`
- 逻辑：遍历所有已安装的 transport，调用 `Transport::registerLocalMemory`，然后把区域记录到 `local_memory_regions_`（受 `std::shared_mutex` 保护）。

**RDMA 层具体注册**

- 文件：`mooncake-transfer-engine/src/transport/rdma_transport/rdma_transport.cpp:210-331`
- 函数：`RdmaTransport::registerLocalMemoryInternal`
- 逻辑：在每个 `RdmaContext` 上注册 MR，收集 `lkey`/`rkey`，构建 `BufferDesc`，再调用 `TransferMetadata::addLocalMemoryBuffer` 加入本地 segment。

**metadata 层写入**

- 文件：`mooncake-transfer-engine/src/transfer_metadata.cpp:1117-1129`
- 函数：`TransferMetadata::addLocalMemoryBuffer`
- 逻辑：把 buffer 追加到 `LOCAL_SEGMENT_ID` 描述符；若 `update_metadata` 为 true，调用 `updateLocalSegmentDesc` → `updateSegmentDesc`（`:468-487`），把 segment JSON 写入 metadata store。

**本地 segment 创建**

- 文件：`mooncake-transfer-engine/src/transport/rdma_transport/rdma_transport.cpp:138-152`
- 函数：`RdmaTransport::install`
- 逻辑：分配本地 segment ID 并发布描述符。

**打开远端 segment**

- 文件：`mooncake-transfer-engine/src/transfer_engine_impl.cpp:528-551`
- 函数：`TransferEngineImpl::openSegment`
- 逻辑：通过 `TransferMetadata::getSegmentID` 把 segment 名字解析为 `SegmentID`。

---

### 2. Segment 生命周期管理

生命周期被拆成两部分：

- **Store Master**：负责权威的分配器/索引状态
- **Transfer Engine / Client**：负责内存注册和 metadata 描述符

#### Store Master 侧

- 文件：`mooncake-store/src/segment.cpp`
- 挂载：`ScopedSegmentAccess::MountSegment`（`:112-219`），创建分配器和索引
- 准备卸载：
  - `PrepareUnmountSegment`（`:266-301`）：移除分配器、移除 host 索引、重置 `buf_allocator`、状态置为 `UNMOUNTING`
  - `PrepareGracefulUnmountSegment`（`:304-344`）：同上，但状态置为 `GRACEFULLY_UNMOUNTING`
- 提交卸载：`CommitUnmountSegment`（`:346-397`）：从 `client_segments_`、`mounted_segments_`、名字索引中移除

MasterService 编排这些步骤：

- `MasterService::MountSegment`：`:782-828`
- `MasterService::UnmountSegment`：`:1961-1998`（prepare → `ClearInvalidHandles` → commit）
- `MasterService::GracefulUnmountSegment`：`:2000-2036`（校验所有权、准备 graceful 状态、调度真正卸载）

#### Client 侧

- 挂载：`Client::MountSegmentAndGetId`（`client_service.cpp:2781-2843`）先向 Transfer Engine 注册内存，再调用 `MasterClient::MountSegment`。
- 立即卸载：`Client::UnmountSegmentImpl`（`client_service.cpp:2733-2758`）请求 master 卸载，然后调用 `TransferEngine::unregisterLocalMemory` 并删除本地记录。
- 优雅卸载：`Client::UnmountSegmentById`（`client_service.cpp:2845-2964`）带 grace period，轮询 `QuerySegmentStatusById`，等 master 报告 segment 已移除或 `UNDEFINED` 后才 unregister TE 内存。

#### Transfer Engine 清理

- 移除本地映射：`TransferMetadata::removeLocalSegment`（`transfer_metadata.cpp:1107-1115`）
- `TransferEngineImpl::closeSegment` 当前是空实现（`:568-570`）

---

### 3. 未完成传输的保护机制

**系统并没有全局等待所有 in-flight transfer**。保护分两层：

#### Store 层优雅卸载（best-effort）

- `GracefulUnmountSegment` 移除分配器（新对象/副本不能再落在此 segment），状态置为 `GRACEFULLY_UNMOUNTING`，然后调度 `grace_period_ms` 后真正卸载（`master_service.cpp:2000-2036`、`segment.cpp:304-344`）。
- Client 端 `StartGracefulUnmountTimer` 等待 `grace_period_ms + 10s`，轮询 `QuerySegmentStatusById`，确认 segment 已移除后才调用 `TransferEngine::unregisterLocalMemory`（`client_service.cpp:2876-2964`）。
- `ClearInvalidHandles` 清理指向无效/已卸载 segment 的 object handle（`master_service.cpp:1904-1936`、`master_service.cpp:4797-4827`）。

#### 立即卸载无等待

- `Client::UnmountSegmentImpl` 在 master 卸载后直接调用 `unregisterLocalMemory`（`client_service.cpp:2733-2758`）。

#### RDMA endpoint 两阶段销毁

- 进入销毁：`RdmaEndPoint::beginDestroy`（`rdma_endpoint.cpp:215-238`）把 QP 状态切到 `IBV_QPS_ERR`，使 in-flight WR 被 flush 到 CQ。
- 完成销毁：`RdmaEndPoint::finishDestroy`（`rdma_endpoint.cpp:240-300`）等待 `wr_depth_list_[*] == 0`（或 30 秒超时）后销毁 QP。
- Endpoint store 驱动：`endpoint_store.cpp:78-140`

**重要注意点**：`RdmaContext::unregisterMemoryRegion` 会立即 deregister MR（`rdma_context.cpp:538-550`），不会等 endpoint WR。因此安全释放内存依赖上层的 graceful unmount 延时或外部保证传输完成。

---

### 4. 传输提交与 segment 拆解的同步

- `TransferEngineImpl` 用 `std::shared_mutex mutex_` 保护 `local_memory_regions_`：
  - `registerLocalMemory` / `unregisterLocalMemory` 加写锁
  - `checkOverlap` 加读锁（`transfer_engine_impl.cpp:581-584`、`606`、`618`、`829-864`）
  - 提交路径不持有该锁，依赖 slice 里已解析的 key
- `RdmaContext::memory_regions_lock_` 保护 MR 注册和 `lkey`/`rkey` 查询（`rdma_context.cpp:533-578`）
- `RdmaEndPoint::submitPostSend` 加 endpoint `lock_`，状态非 `CONNECTED` 时拒绝新 WR（`rdma_endpoint.cpp:766-774`）
- Worker pool 轮询 CQ 原子更新 `wr_depth_list_` 和 `cq_outstanding_`，`finishDestroy` 据此判断
- Store master segment 状态变更由 `ScopedSegmentAccess` 串行化；优雅卸载调度由 `DeadlineScheduler<GracefulUnmountDeadlineRecord>` 处理（`master_service.h:1494-1500`、`master_service.cpp:181-192`）

---

## 问题二：RDMA transport 里 QP 怎么创建和复用？WR 怎么组织？重连时未完成请求怎么办？

### 1. QP 创建与管理

**没有 QP 池，每个 endpoint 独占一组 QP**

- `RdmaEndPoint::qp_list_`：`std::vector<ibv_qp *>`（`rdma_endpoint.h:187`）
- 创建：`RdmaEndPoint::construct()`（`rdma_endpoint.cpp:96-137`），按 `num_qp_per_ep` 配置创建 QP，共享同一个 CQ
- QP caps：
  - `max_send_wr = max_recv_wr = max_wr_depth`
  - `max_send_sge = max_recv_sge = max_sge_per_wr`
  - `max_inline_data = max_inline`
  - 见 `rdma_endpoint.cpp:125-127`

**Endpoint 级缓存（不是 QP 复用）**

- `RdmaContext` 用 `EndpointStore`（FIFO 或 SIEVE）按 `peer_nic_path` 缓存 endpoint
- 入口：`RdmaContext::endpoint()`（`rdma_context.cpp:602-622`）
- 实现：`endpoint_store.cpp:30-365`
- 同一远端 NIC 路径会复用同一个 endpoint 对象，从而复用其 QP

**QP 销毁是两阶段**：见问题一中的 `beginDestroy` / `finishDestroy`。

---

### 2. WR 组织方式

**无接收 WR 路径**：该 transport 仅用 RC QP 做 RDMA READ/WRITE。

**发送 WR 准备**

- 文件：`mooncake-transfer-engine/src/transport/rdma_transport/rdma_endpoint.cpp:766-856`
- 函数：`RdmaEndPoint::submitPostSend`
- 输入：一个 peer endpoint 对应的 `slice_list`
- 分配 `wr_list` 和 `sge_list`，大小为 `max_postable_per_qp`（`:785-788`）
- 每个 slice 对应一个 `ibv_send_wr`、一个 `ibv_sge`
- WR 通过 `wr.next` 链成链（`:824`）
- 每个 WR 都是 `IBV_SEND_SIGNALED`，opcode 为 `IBV_WR_RDMA_READ` 或 `IBV_WR_RDMA_WRITE`
- `wr_id = (uint64_t)slice`，remote addr/rkey 来自 slice（`:815-826`）

**QP 选择**

- 同一 endpoint 内按 round-robin 把 slice 分片到 `qp_list_`（`:792-851`）
- 计算每个 QP 可用深度：`qp_avail = max_wr_depth_ - wr_depth_list_[qp_index]`
- 用 `__sync_fetch_and_add/sub` 原子更新 `wr_depth_list_` 和 `cq_outstanding_`

**上层 WorkerPool 队列**

- 文件：`mooncake-transfer-engine/src/transport/rdma_transport/worker_pool.cpp:241-254`
- 按 `(target_id * 10007 + device_id) % kShardCount` 分 shard
- 每个 worker 的 `performPostSend()` 把同一 `peer_nic_path` 的 slice 收集到 `collective_slice_queue_[thread_id]`，再调用 `endpoint->submitPostSend()`（`:274-390`）

---

### 3. 连接重建与未完成请求

**重连触发条件**

- 在主动/被动握手时，若 peer QP number 列表不匹配，会触发重连
- 主动：`setupConnectionsByActive()`（`rdma_endpoint.cpp:314-559`）
- 被动：`setupConnectionsByPassive()`（`rdma_endpoint.cpp:561-685`）
- 当已 `CONNECTED` 但 `peer_qp_num_list_ != peer_desc.qp_num` 时，调用 `resetConnection()`（`:451-460`、`:573-576`）

**关键策略：一旦 `has_connected_` 为 true，endpoint 会被 retire 而不是复用**

- `resetConnection()`（`:730-750`）检查 `has_connected_`：
  - 若为 false（pre-connected）：`disconnectUnlocked()` 把 QP reset
  - 若为 true：日志告警并调用 `beginDestroyLocked()`，返回 `ERR_ENDPOINT`；新连接必须创建新 endpoint

**未完成 WR 的处理**

- `beginDestroyLocked()`（`:220-238`）设置 `active_ = false`、`status_ = DESTROYING`，所有 QP 切到 `IBV_QPS_ERR`
- 硬件会把 in-flight WR flush 到 CQ，状态为 `IBV_WC_WR_FLUSH_ERR`
- `WorkerPool::performPollCq()`（`worker_pool.cpp:392-470`）对 `IBV_WC_WR_FLUSH_ERR` 特殊处理：标记 slice 失败，**不重试**（`:418-426`）
- 其他 WC 错误触发 `handlePathFailure()` + `redispatch()` 重试（`:431-449`）

**重试/重新调度**

- `WorkerPool::redispatch()`（`worker_pool.cpp:472-545`）增加 `slice->rdma.retry_cnt`，重新选择 peer device/rail，重新入队
- rail 失败 5 次后暂停 1 秒（`worker_pool.h:115-116`）

**异步致命事件**

- `IBV_EVENT_QP_FATAL`、`IBV_EVENT_DEVICE_FATAL` 等在 `WorkerPool::doProcessContextEvents()`（`worker_pool.cpp:597-667`）中处理：ack 事件后删除 endpoint 或标记整个 context inactive

---

### 4. Endpoint 生命周期与优雅关闭

状态机（`rdma_endpoint.h:40-48`）：

```
INITIALIZING → UNCONNECTED → CONNECTING → CONNECTED → DESTROYING → DESTROYED
```

**创建**

1. `RdmaContext::endpoint()` 查 EndpointStore（`rdma_context.cpp:602-622`）
2. `EndpointStore::insertEndpoint()` 构造 `RdmaEndPoint` 并调用 `construct()`（`endpoint_store.cpp:49-76`、`206-232`）
3. 握手进入 `CONNECTED`：`setupConnectionsByActive/Passive()`

**优雅/错误关闭**

1. `beginDestroy()` / `beginDestroyLocked()`：标记 inactive、状态 `DESTROYING`、QP 切 `IBV_QPS_ERR`（`rdma_endpoint.cpp:215-238`）
2. Endpoint 移入 `EndpointStore::waiting_list_`（`endpoint_store.cpp:78-95`、`234-255`）
3. `EndpointStore::reclaimEndpoint()` / `RdmaContext::reclaimEndpoints()` 周期性调用 `finishDestroy()`（`endpoint_store.cpp:131-140`、`309-318`）
4. `finishDestroy()`（`rdma_endpoint.cpp:240-300`）等待 `wr_depth_list_[*] == 0` 或 30 秒超时，然后 `deconstructLocked()` 销毁 QP，状态 `DESTROYED`

**Context / transport 销毁**

- `RdmaContext::deconstruct()`（`rdma_context.cpp:294-344`）：销毁 worker pool、EndpointStore QPs、MR、CQ、comp channel、PD、关闭设备
- `RdmaTransport::~RdmaTransport()`（`rdma_transport.cpp:95-103`）：清空 `context_list_`
- `WorkerPool::~WorkerPool()`（`worker_pool.cpp:106-112`）：停止 worker 并 join

---

## 问题三：metadata server 里 segment 信息怎么序列化？etcd/HTTP/Redis 三种后端有什么差异？

### 1. Segment 序列化/反序列化

Segment 在内存中是 `TransferMetadata::SegmentDesc`，存到 metadata store 时是 JSON。

- 编码：`TransferMetadata::encodeSegmentDesc()`（`mooncake-transfer-engine/src/transfer_metadata.cpp:287`）
- 解码：`TransferMetadata::decodeSegmentDesc()`（`mooncake-transfer-engine/src/transfer_metadata.cpp:634`）
- 写入：`TransferMetadata::updateSegmentDesc()` 调用 `storage_plugin_->set()`（`:468`）
- 读取：`TransferMetadata::getSegmentDesc()` 调用 `storage_plugin_->get()`（`:927`）

**JSON 是协议相关的**。公共字段包括 `name`、`protocol`、`tcp_data_port`、`timestamp`、可选 `rdma_server_name`。各协议的额外字段：

| 协议 | 额外字段 |
|------|---------|
| rdma / barex / efa / cxi | `devices[]`（name/lid/gid）、`buffers[]`（name/addr/length/rkey[]/lkey[]）、`priority_matrix` |
| ub | `devices[]`（name/eid）、`buffers[]`（name/addr/length/tseg[]）、`priority_matrix` |
| tcp | `buffers[]`（name/addr/length） |
| ascend | `devices[]`（name/lid）、`buffers[]`（name/addr/length）、`rank_info` |
| nvlink / nvlink_intra / hip / maca / ubshmem / sunrise_link | `buffers[]`（name/addr/length/shm_name） |
| cxl | `cxl_name`、`cxl_base_addr`、`buffers[]`（name/offset/length） |
| nvmeof | `nvmeof_buffers[]`（file_path/length/local_path_map） |

多协议模式（`ENABLE_MULTI_PROTOCOL`）支持同一段里组合 `cxl`/`tcp`/`rdma`/`hip`，buffer 带 `protocol` 标签（`:224`、`:511`）。

**SegmentDesc 结构**

- 文件：`mooncake-transfer-engine/include/transfer_metadata.h:96`
- 包含：`name`、`protocol`、`devices`、`topology`、`buffers`、`nvmeof_buffers`、`cxl_name`、`cxl_base_addr`、`timestamp`、`rank_info`、`tcp_data_port`、`rdma_server_name`

---

### 2. 后端差异

所有后端都实现抽象接口 `MetadataStoragePlugin`（`mooncake-transfer-engine/include/transfer_metadata_plugin.h:21`）：

```cpp
virtual bool get(const std::string &key, Json::Value &value) = 0;
virtual bool set(const std::string &key, const Json::Value &value) = 0;
virtual bool remove(const std::string &key) = 0;
```

工厂：`MetadataStoragePlugin::Create()`（`src/transfer_metadata_plugin.cpp:544`）

#### HTTP

- 文件：`mooncake-transfer-engine/src/transfer_metadata_plugin.cpp:210`
- 每线程使用 `libcurl`，PUT/GET/DELETE，key 做 URL 编码
- 超时硬编码：请求 3 秒、连接 1.5 秒
- 示例 server：`mooncake-transfer-engine/example/http-metadata-server/main.go`
  - 内存 `sync.Map`，`/metadata` 路由 GET/PUT/DELETE
  - 无持久化、复制、lease

#### etcd

- 文件：`mooncake-transfer-engine/src/transfer_metadata_plugin.cpp:397`
- 通过 `USE_ETCD_LEGACY` 选择实现：
  - 旧版 C++ client：`etcd::SyncClient`（`:399`）
  - 默认 Go wrapper via cgo（`:449`）
- Go wrapper：`mooncake-common/etcd/etcd_wrapper.go`
  - 全局 `clientv3.Client`，dial 超时 5 秒，消息大小 32 MiB（`:123`）
  - `EtcdPutWrapper` / `EtcdGetWrapper` / `EtcdDeleteWrapper`：单 key 操作，5 秒超时（`:166`、`:184`、`:207`）
  - 还支持 lease、transaction（CAS on `CreateRevision==0`）、prefix/range query、watch

#### Redis

- 文件：`mooncake-transfer-engine/src/transfer_metadata_plugin.cpp:71`
- 使用 `hiredis`；支持 `MC_REDIS_USERNAME`、`MC_REDIS_PASSWORD`、`MC_REDIS_DB_INDEX`
- 单个 `redisContext*`，受 `access_client_mutex_` 保护；原始 GET/SET/DEL
- `tent` 里还有一个较新的 Redis metastore（`mooncake-transfer-engine/tent/src/metastore/redis.cpp`），但当前 transfer engine 用上面的 plugin

#### P2PHANDSHAKE

- 文件：`mooncake-transfer-engine/src/transfer_metadata.cpp:162`
- 常量：`P2PHANDSHAKE`（`include/transfer_metadata.h:41`）
- 不创建 `storage_plugin_`，设置 `p2p_handshake_mode_ = true`
- 通过 TCP socket 直接交换 metadata：`HandShakePlugin::exchangeMetadata()`
- `getSegmentDesc()` 把 segment_name 解析为 `ip:port`，向对方发送本地编码后的描述符（`:904`）
- 握手插件固定为 `SocketHandShakePlugin`（`src/transfer_metadata_plugin.cpp:1253`），负责连接/握手/通知/probe 消息

---

### 3. 一致性、缓存与过期

#### 客户端缓存

- `TransferMetadata` 维护：
  - `segment_id_to_desc_map_` 和 `segment_name_to_id_map_`，受 `segment_lock_` 保护
  - `rpc_meta_map_`，受 `rpc_meta_lock_` 保护
- 缓存行为由 `globalConfig().metacache` 控制，默认 `true`，可通过环境变量 `MC_DISABLE_METACACHE` 关闭（`src/config.cpp:315`、`include/config.h:62`）
- `getSegmentDescByName()`：若 `metacache` 开启且 `force_update` 为 false，直接返回缓存（`:999`）
- `getSegmentDescByID()`：`force_update` 为 true 或 `metacache` 关闭时从 storage 刷新（`:1037`）
- `syncSegmentCache()`：重新拉取所有非本地缓存 segment（`:961`）
- RPC meta 首次 lookup 后永久缓存（`:1234`）

#### 后端一致性

- **HTTP 示例 server**：单进程内存存储，无额外一致性协议；生产 HTTP 后端行为取决于外部实现
- **etcd**：使用 etcd 默认的线性化读写（5 秒超时）；可用 lease 和 CAS transaction 做更强的生命周期控制
- **Redis**：单节点 Redis 语义；segment metadata 没有乐观锁或事务
- **P2PHANDSHAKE**：无中心存储；一致性是点对点的，即"连接时读取对方当前状态"，没有超过 TCP 交换生命周期的过期跟踪

#### 过期风险

- `metacache` 开启后，segment 描述符首次拉取后缓存；owner 的更新**不会推送**给其他 client，consumer 只有在 `force_update` 或 `syncSegmentCache()` 后才能看到最新值
- RPC meta 同样首次 lookup 后缓存
- P2P 模式下 `removeSegmentDesc()` 只删本地 map 条目（`:489`）

---

## 问题四：一个 `store.get(key)` 从 Python API 到 Master 再到 Transfer Engine，经过多少层 RPC？

### 1. Python 入口

以低层字节 `get` 为例：

- 文件：`mooncake-wheel/mooncake/structured_object_store.py:3013`
- 函数：`_MooncakePayloadTransport.read_payload()` 最终调用 `self._store.get(chunk["key"])`
- `store` 对象就是 pybind 绑定的 C++ 类 `MooncakeDistributedStore`

### 2. 进入 C++ pybind

- 模块定义：`mooncake-integration/store/store_py.cpp:1806`（`PYBIND11_MODULE(store, m)`）
- 类绑定：`mooncake-integration/store/store_py.cpp:2100`（`py::class_<MooncakeStorePyWrapper>(m, "MooncakeDistributedStore")`）
- `get` 方法绑定：`:2213`（`.def("get", &mooncake::MooncakeStorePyWrapper::get)`）
- `MooncakeStorePyWrapper::get()`：`:472-506`，释放 GIL，调用 `store_->get_buffer(key)`，把返回的 `BufferHandle` 拷贝成 `pybind11::bytes`
- 底层 `store_` 是 `std::shared_ptr<PyClient>`，通过 `init_real_client()` 初始化为 `RealClient`（`:368`）

### 3. 到 Master 的 RPC

从 `RealClient::get_buffer_internal()` 开始：

- 文件：`mooncake-store/src/real_client.cpp:2551`
- 先调用 `client_->Query(key)`
- `Client::Query()`（`client_service.cpp:1055-1066`）调用 `master_client_.GetReplicaList(object_key)`
- `MasterClient::GetReplicaList()`（`master_client.cpp:529-538`）通过 `invoke_rpc<&WrappedMasterService::GetReplicaList, GetReplicaListResponse>(object_key, tenant_id)` 发 RPC
- RPC 声明：`mooncake-store/include/rpc_service.h:62-67`
- RPC 注册：`mooncake-store/src/rpc_service.cpp:1269-1285`（`RegisterRpcService()` 在 master coro_rpc server 注册 handler）

Master 返回：

- `MasterService::GetReplicaList()`（`master_service.cpp:2527-2599`）构造 `GetReplicaListResponse`
- 包含：
  - `replicas`：`Replica::Descriptor` 数组（memory / local_disk / disk / nof），只返回 `status == COMPLETE` 的
  - `lease_ttl_ms`：KV lease，防止读取期间对象被驱逐
- Client 包装成 `QueryResult`（`client_service.cpp:1063-1066`）

### 4. 定位远端 segment 并发起传输

回到 client：

- `RealClient::get_buffer_internal()`（`:2586-2593`）调用 `SelectBestReplica()` 选择最佳副本：本地 MEMORY > 任意 MEMORY > LOCAL_DISK > DISK
- `:2600-2643`：分配本地 buffer，用 `allocateSlices()` 构造 slices
- `:2643`：调用 `client_->Get(key, filtered_qr, slices)`

`Client::Get()` 内部：

- `client_service.cpp:1114-1172`：找到第一个 complete replica，可选走 local hot cache，然后调用 `TransferRead(replica, slices)`
- `TransferRead()`（`:3506-3531`）：校验大小，调用 `TransferData(..., TransferRequest::READ)`
- `:3464` / `:3489`：`transfer_submitter_->submit()` 或 `submitRangeRead()` 把请求提交给 Transfer Engine
- Transfer Engine 执行实际的 RDMA/TCP/CXL 等传输

### 5. LOCAL_DISK（SSD offload）副本的特殊路径

- `RealClient::get_buffer_internal()`（`:2610-2624`）路由到 `batch_get_into_offload_object_internal()`
- `:5538`：向 owner 的 offload RPC service 发 `client_requester_->batch_get_offload_object()` RPC
- `:5551`：然后调用 `client_->BatchGetOffloadObject()`，根据返回的 `transfer_engine_addr` 做 TE 传输

### 6. 典型 get 路径的网络跳数

| 副本类型 | 跳数 | 说明 |
|---------|------|------|
| MEMORY / NOF / disk | 2 | 1 次 Master `GetReplicaList` RPC + 1 次 Transfer Engine 数据传输 |
| LOCAL_DISK（SSD offload） | 3 | 1 次 Master RPC + 1 次远端 offload RPC + 1 次 TE 数据传输 |

Hot cache 路径可以在本地命中时跳过 TE 数据传输，但仍会先查 Master 的 `GetReplicaList`。

---

## 汇总：四个问题对应的必读文件

| 问题 | 必读文件 |
|------|---------|
| Segment 注册/生命周期 | `transfer_engine.h`、`transfer_engine_impl.cpp`、`rdma_transport.cpp`、`transfer_metadata.cpp`、`segment.cpp`、`client_service.cpp`、`master_service.cpp` |
| RDMA QP/WR/重连 | `rdma_endpoint.cpp`、`.h`、`rdma_context.cpp`、`.h`、`rdma_transport.cpp`、`worker_pool.cpp`、`.h`、`endpoint_store.cpp` |
| Metadata 序列化/后端 | `transfer_metadata.cpp`、`.h`、`transfer_metadata_plugin.cpp`、`.h`、`etcd_wrapper.go`、HTTP metadata server |
| store.get 完整链路 | `structured_object_store.py`、`store_py.cpp`、`real_client.cpp`、`client_service.cpp`、`master_client.cpp`、`master_service.cpp`、`rpc_service.cpp`、`.h` |
