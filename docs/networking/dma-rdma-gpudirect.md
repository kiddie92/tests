# DMA、RDMA 与 GPUDirect RDMA：数据到底怎么走？

> 面向 LLM 系统学习的基础笔记。以常见 PCIe 网卡和独立 GPU 为例，区分概念、数据路径与实际启用条件。本文没有在具体 GPU 集群上运行基准；硬件与驱动支持须以部署版本为准。

## 1. 一句话理解 DMA

DMA（Direct Memory Access，直接内存访问）让设备搬运数据，省去 CPU 核心逐字节执行复制指令的工作。

可以把 CPU 理解为安排任务的人，DMA 引擎是搬运工。但这个比喻只解释分工：CPU 通常仍要准备缓冲区、描述符、地址映射、通知与完成处理。“不经过 CPU”通常指 **CPU 核心不负责复制有效载荷**，并不表示不经过 CPU 所在封装的 PCIe 根复合体、内存控制器或系统互连。

DMA 不是一个网络协议，也不保证端到端零拷贝。

## 2. 普通网卡本来就在使用 DMA

典型收包路径：

```text
网络 → 网卡 → DMA 写入主机接收缓冲区 → 协议栈 → 应用
```

驱动先提供可供网卡使用的缓冲区和描述符；网卡收到数据后写入内存，再通过完成状态、中断或轮询让软件知道可以处理了。传统 socket 路径中，协议处理和内核到用户缓冲区的复制仍可能消耗 CPU。

典型发包路径则相反：

```text
应用 / 协议栈准备数据 → 网卡 DMA 读取主机发送缓冲区 → 网络
```

所以，“网卡会 DMA”不等于“这是一张支持 RDMA 的网卡”，更不等于“它能直接访问 GPU 显存”。

## 3. 网卡拿到的到底是什么地址？

必须区分三个层次：

- **CPU 虚拟地址**：进程或内核代码使用的指针。
- **CPU 物理地址**：CPU 地址翻译后对应的物理地址。
- **设备 DMA 地址**：设备执行 DMA 时使用的地址，Linux 中常用 `dma_addr_t` 表示。

```text
CPU 虚拟地址 ── CPU MMU ──→ 主机物理内存
设备 DMA 地址 ─ IOMMU 等 ─→ 主机物理内存
```

启用相应 IOMMU 映射时，设备可以使用 I/O 虚拟地址（IOVA），由 IOMMU 翻译并限制访问范围。没有这层翻译时，也不能通用地假设设备地址一定等于 CPU 物理地址，平台和主桥还可能引入差异。

驱动应通过 DMA API 获取设备可用地址，不应把任意用户指针或物理地址直接填给网卡。缓冲区生命周期、缓存一致性、内存屏障和 DMA 完成后的复用也必须正确处理。

来源：[Linux Dynamic DMA mapping Guide](https://docs.kernel.org/core-api/dma-api-howto.html)。

## 4. RDMA 把什么能力扩展到了网络另一端？

RDMA（Remote Direct Memory Access）允许支持该能力的网卡配合软件，在网络两端的已注册内存之间传输数据，减少传统内核网络数据路径和中间复制的开销。

以 RDMA Write 为例：

```text
机器 A 主机内存
  → A 网卡读取
  → 网络传输
  → B 网卡写入
  → 机器 B 已注册主机内存
```

远端内存不是任意可写的。应用需要注册内存区域（MR），配置访问权限，并按操作类型提供地址和 key；`lkey` 用于本地缓冲区访问，`rkey` 用于远端 RDMA / 原子操作授权校验。MR 地址也不应直接等同于裸物理地址。

常见操作有：

- **Send / Receive**：接收端需要预先投递接收缓冲区。
- **RDMA Write / Read**：单边操作，远端 CPU 不必逐条参与数据搬运，但仍需要提前建立资源与权限，并由应用协议安排通知和同步。

“注册内存”与 IOMMU 映射是不同层次。固定页面是常见实现方式，但存在按需分页等机制，不能把所有 MR 都理解成永久锁定整块内存。

来源：[rdma-core：ibv_reg_mr 手册](https://github.com/linux-rdma/rdma-core/blob/master/libibverbs/man/ibv_reg_mr.3)。

## 5. GPUDirect RDMA 为什么对 GPU 通信有用？

若网络传输依赖主机内存中转，一个典型 GPU 到 GPU 路径是：

```text
GPU A 显存 → 主机 A 内存 → 网卡 A
          → 网络 →
网卡 B → 主机 B 内存 → GPU B 显存
```

在硬件、驱动、映射与拓扑都支持的情况下，GPUDirect RDMA 可形成：

```text
GPU A 显存 ↔ 网卡 A ↔ 网络 ↔ 网卡 B ↔ GPU B 显存
```

关键变化是减少主机内存 staging（中转缓冲）及相关复制。网络链路、网卡缓冲和 PCIe 传输仍然存在。

这里的“直接”有条件：

- 网卡和 GPU 必须兼容，软件必须能注册、映射对应 GPU 内存。
- PCIe 拓扑和平台支持影响可达性及性能。
- GPU 生产数据、网卡读取、远端写入、GPU 消费之间要建立正确的先后关系。不能默认网卡写入时，正在运行的 GPU kernel 就能安全读取同一缓冲区。
- 这条路径本身不保证 CPU 完全退出控制面，也不表示所有 GPU 通信都在使用它。

来源：[NVIDIA GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/)。

## 6. 实际部署时最容易混淆的边界

**普通主机 DMA 的 IOMMU 支持，不能直接套到 GPU peer-to-peer 路径上。** 后者需核对具体平台、裸机 / 虚拟机模式和驱动文档；ACS 路由也可能改变路径。不要为了追求带宽，直接照抄关闭 IOMMU 或 ACS 的命令，这些设置涉及隔离和系统安全。

GPU 内存注册可涉及 `nvidia-peermem`，也可能使用受支持的 DMA-BUF 路径。缺少前者不一定说明 GPUDirect RDMA 不可用；反过来，加载了它也不能证明应用真的走了直连路径。

来源：[NCCL GPU troubleshooting](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/troubleshooting/gpu_troubleshooting.html)。

另外：

- **DMA**：设备访问内存的数据搬运机制。
- **RDMA**：跨网络访问已授权内存的通信能力。
- **GPUDirect RDMA**：让 GPU 内存参与兼容设备的直接数据路径。
- **NVLink**：另一类 GPU 互连，不是 RDMA 的同义词。
- **NCCL**：集合通信库。AllReduce 等操作的具体路径由配置、拓扑与实现决定，不能从调用了 NCCL 就推断每一段传输都使用 GPUDirect RDMA。

## 7. 放到 LLM 训练 / 推理里怎么思考？

把一块待发送的张量想象成一个包裹，逐段问：

1. 数据当前在主机内存，还是 GPU 显存？
2. 哪个引擎负责每一段搬运？CPU 核心、GPU copy engine，还是网卡？
3. 网卡看到什么地址，谁负责映射和权限？
4. 是否经过主机中转缓冲？是哪一层引入的？
5. 生产者什么时候写完，消费者什么时候可以读？
6. 实测瓶颈在网络、PCIe、内存、同步等待，还是计算与通信无法重叠？

例如跨节点传输张量、梯度或 KV cache 时，减少主机中转可能有帮助，但收益必须结合消息大小、传输频率、GPU / NIC 亲和性和并发负载测量。不能只凭理论带宽或“支持 RDMA”判断性能。

## 8. 后续实践记录模板

本篇只建立概念。做实验时可按以下结构补充记录：

- 环境：GPU、NIC、PCIe / NUMA 拓扑，内核、GPU 驱动、通信库版本。
- 路径证据：实际传输后端、GPU 内存注册方式、日志和拓扑。
- 对照：主机内存中转与受支持的 GPU 直连路径；固定消息大小、次数和并发。
- 正确性：接收数据校验、完成事件与 GPU 同步，先确认正确再看速度。
- 指标：端到端延迟、有效带宽、CPU 开销和整项任务耗时。
- 结论边界：只描述当前配置的结果，不直接推广到其他 GPU 架构或平台。

### 延伸资料

- [NVIDIA DOCA DMA 技能与资料入口](https://github.com/NVIDIA/skills/blob/main/skills/doca-dma/SKILL.md)：侧重 BlueField / ConnectX 的 DMA 实践；跨网络搬运应查对应 RDMA 资料。
- [MIT HAN Lab KernelWiki](https://github.com/mit-han-lab/KernelWiki)：社区 GPU kernel 优化知识库，适合进一步学习 Hopper / Blackwell 的计算与片上搬运；不是本篇跨节点 DMA / RDMA 语义的规范来源。
