# Mooncake 代码仓库概览

## 典型应用场景

Mooncake 是面向大语言模型推理的 **KV Cache-centric 分离式（disaggregated）推理服务平台**，也就是把 LLM 推理中的 **Prefill（计算/生成前缀）** 和 **Decode（逐 token 生成）** 两个阶段拆分到不同节点执行，并在节点之间高效传输中间状态 KV Cache。简单说：它在集群里把"算"和"存/搬"拆开，并通过高速网络把 KV Cache 在节点间快速倒来倒去，从而提升整体吞吐和 GPU 利用率。

这是 Moonshot AI / Kimi 实际使用的 serving 平台的开源版本。

## 两大核心组件

### 1. Transfer Engine（数据传输引擎）

负责在不同介质和不同节点之间批量搬运数据，包括：

- **介质**：DRAM、VRAM（GPU 显存）、NVMe SSD、CXL 等
- **网络**：TCP、RDMA/InfiniBand/RoCE/eRDMA、AWS EFA、NVLink、NVMe-oF、华为 Ascend HCCL/Direct/UBShmem、AMD HIP、Barex 等

它把底层五花八门的硬件/网络协议统一成一个上层 API：`transfer() / read() / write()` 式的批量数据移动接口。

### 2. Mooncake Store（分布式 KV Cache 存储引擎）

基于 Transfer Engine 构建，为 XpYd（多 Prefill 节点 + 多 Decode 节点）分离式推理提供分布式键值缓存存储，支持：

- 层级化 KV Cache 池化（内存 + SSD 卸载）
- 对象到 segment 的映射管理
- 端到端点对点数据传输

## 模块间沟通方式

```
┌─────────────────────────────────────────────────────────────┐
│                     上层应用 / vLLM / PyTorch                 │
└───────────────────────────┬─────────────────────────────────┘
                            │ Python bindings
        ┌───────────────────┴───────────────────┐
        ▼                                       ▼
┌───────────────┐                   ┌───────────────────────┐
│ mooncake-store│                   │ mooncake-transfer-    │
│ (store.so)    │                   │ engine (engine.so)    │
│ - 客户端 API    │                   │ - 传输 API            │
│ - 对象 put/get  │◄─────────────────│ - segment 注册/发现    │
└───────┬───────┘   调用 Transfer     └───────────┬───────────┘
        │                                         │
        │         ┌───────────────────────────────┘
        │         │ 底层传输实现
        ▼         ▼
┌──────────────────────────────────────────────────────────┐
│  Transport implementations: TCP / RDMA / NVLink / EFA /   │
│  CXL / HCCL / UBShmem / HIP ...                           │
└──────────────────────────────────────────────────────────┘
                            ▲
                            │ peer discovery
        ┌───────────────────┴───────────────────┐
        ▼                                       ▼
┌─────────────────────┐             ┌─────────────────────────┐
│  Mooncake Master    │             │  Metadata Server        │
│  (mooncake_master)  │             │  - HTTP server          │
│  - 对象-段映射       │             │  - etcd                 │
│  - 集群资源/调度      │             │  - Redis                │
│  - 控制平面          │             │  - P2PHANDSHAKE         │
└─────────────────────┘             └─────────────────────────┘
         Store 控制平面                Transfer Engine 名字发现
```

## 最核心的机制

### 1. Segment / Batch Transfer 模型

Transfer Engine 把内存/显存抽象成 **Segment**，一次传输是一个 **Batch**，里面多个 **Transfer Request**。核心调用类似：

```cpp
transfer(batch_desc);  // 发起一批读写请求
```

每个请求包含：源/目标地址、长度、本地/远端 segment、opcode（read/write）等。上层不用关心底层是 RDMA 还是 TCP。

### 2. 元数据服务做 peer 发现

Transfer Engine 需要知道"远端 segment 在哪台机器、什么地址、什么设备"。这个发现依赖一个独立的 **metadata server**，把 segment 信息注册上去，对端去查询。

注意：**metadata server 只做 peer 发现**，不做数据中转。数据走的是节点间直连（P2P）。

### 3. Mooncake Master 做 KV Cache 控制平面

Store 的 Master 管理对象到 segment 的映射、分配策略、副本/驱逐、SSD 卸载等。Client 通过 RPC 与 Master 通信，但实际 KV Cache 数据还是通过 Transfer Engine 在 client 之间直接搬。

### 4. 统一的内存/显存/网络抽象

不管数据在 CPU 内存、GPU 显存、SSD 还是远端机器，Transfer Engine 都试图用同一套 segment + batch 接口表达，屏蔽底层硬件差异。

## 一句话总结

Mooncake 让 LLM 推理可以把 Prefill 和 Decode 分开放到不同 GPU 节点上，并通过一个统一的 Transfer Engine 在节点间/显存间高速搬运 KV Cache，而 Mooncake Store 在此基础上提供分布式、可分层（内存+SSD）的 KV Cache 存储能力。
