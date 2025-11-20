---
date: "2025-11-20T23:42:29+08:00"
draft: false
title: "Virtio Queue || DMA Coherent"
---

# Virtio Queue 和 DMA Coherent

# DMA-coherent 决定设备 DMA 的 coherence 由硬件还是软件维护

在 ARM Linux 启动时，内核会解析 `.dtb` (Device Tree Blob) 文件。这是判定一致性模式的**第一权威来源**。

### `dma-coherent`

dma-coherent 是 Device Tree (DT) 中的一个布尔属性，用于描述 CPU 缓存（Cache） 与 DMA 设备（Device） 之间的内存数据一致性维护方式。它决定了操作系统内核在进行 DMA 操作时，是依赖硬件自动同步，还是需要通过软件指令手动干预。

### （1）**设备节点或其父总线节点中包含 dma-coherent—>HardwareMaintained**

**硬件机制：**硬件嗅探 (Hardware Snooping) 固件告知内核，系统采用了支持硬件一致性的互联总线（如 ARM CCI、CCN 或 CMN）。当设备发起 DMA 读写请求时，总线会自动嗅探（Snoop）CPU 的 Cache。

**设备读（TO_DEVICE）：** 如果数据在 CPU Cache 中（Dirty），总线会直接从 Cache 获取最新数据给设备，无需写回内存。

**设备写（FROM_DEVICE）：** 设备写入数据后，总线会自动让 CPU Cache 中的旧数据失效（Invalidate）。

**内核行为：**零开销 (Zero-Overhead) 内核通过 DMA API（如 dma_sync_single_for_cpu/device）管理内存时，识别到硬件支持一致性，这些函数将退化为空操作 (No-Op) 或仅包含极其轻微的屏障指令。

优势： 极大减少 CPU 指令开销，显著提升吞吐量。

### （2）**设备节点或其父总线节点中不包含 dma-coherent—>SoftwareMaintained**

**硬件机制：**内核默认假设**硬件不具备一致性能力**。CPU 的 Cache 和设备直接访问的主存（DRAM）是两个独立的数据域。CPU 写在 Cache 里的数据，设备在 DRAM 里看不到；设备写在 DRAM 里的数据，CPU 在 Cache 里读不到（读到旧值）。
**设备读（TO_DEVICE）：**设备将 DMA 读取数据之前，内核执行 **Clean** 操作（如 `DC CVAC`），将 Cache 中的脏数据强制刷入 DRAM。
**设备写（FROM_DEVICE）：**设备 DMA 写完数据之后，内核执行 **Invalidate** 操作（如 `DC IVAC`），废弃 Cache 中的旧数据，强迫 CPU 下次读取时从 DRAM 重新加载。
**内核行为：**内核必须在每次 DMA 传输前后，调用`dma_sync_*`强制执行 Cache 维护操作（Cache Maintenance Operations, CMO），如上述提到的**Clean 和 Invalidate。**
**劣势：** 频繁的 CMO 指令会阻塞 CPU 流水线，大幅增加延迟，严重降低 I/O 性能。

| **维度**         | **存在 dma-coherent (Hardware)** | **不存在 dma-coherent (Software)**        |
| ---------------- | -------------------------------- | ----------------------------------------- |
| **一致性维护者** | **硬件互联总线** (CCI/CMN/DSU)   | **操作系统内核** (Software Driver)        |
| **同步机制**     | 自动嗅探 (Snooping)              | 手动刷洗 (Flushing / Invalidating)        |
| **CPU 汇编指令** | 无特殊指令 (Standard Load/Store) | `DC CVAC` (Clean), `DC IVAC` (Invalidate) |

| **内核 API 行为**
(`dma_sync_*`) | **空操作 (No-Op)** | **执行繁重的 Cache 维护代码** |
| **内存访问路径** | CPU—Cache—Device | CPU—Cache—DRAM—Device |
| **性能影响** | **极高** (低延迟，高吞吐) | **较低** (高 CPU 开销，高延迟) |
| **典型应用场景** | 服务器、高性能 SoC、虚拟化 (Virtio) | 嵌入式设备、遗留硬件、无互联总线的 IP 核 |
| **SWIOTLB 配合** | 仅需 `memcpy`，不需要 Flush | 需要 `memcpy` **加上** 昂贵的 Flush 操作 |

# Virtio Queue

| **维度**     | **Virtio Queue (vring)**             | **Payload (Data Buffer)**              |
| ------------ | ------------------------------------ | -------------------------------------- |
| **API**      | **`dma_alloc_coherent`**             | **`dma_map_single`**                   |
| **本质**     | **分配**新内存 映射 DMA 地址         | **映射已有**内存 到 映射 DMA 地址      |
| **生命周期** | **静态长期** (Driver Load -> Unload) | **动态瞬时** (Packet TX/RX)            |
| **数据方向** | **双向频繁交互** (CPU <-> Device)    | **单向传输** (To Device / From Device) |

| **软件如何维护
coherence** | **禁用 Cache (NC)**
避免手动刷 Cache | **开启 Cache (WB) + 手动 Flush**
利用 Cache 加速数据填充 |

## (1).Virtio queue

使用`dma_alloc_coherent` 为`virtio queue`分配一致性内存。

### **A.支持 DMA-coherent(Hardware Coherent)**

**CPU Stage 1 属性:** **Normal Write-Back (WB)**

**机制:** CPU 读写 Cache，硬件总线会自动把数据同步给设备。

### B.不支持 DMA-coherent(Software Coherent / Non-Coherent HW)

**CPU Stage 1 属性:** **Normal Non-Cacheable (NC)**

**机制:** 内核通过 `pgprot_dmacoherent` 修改页表让内存属性为**Non-Cacheable**，**强制禁用** 这块内存的 Cache。
**原因:** 如果这里用 WB，CPU 每次改一个 bit（比如更新 `avail_idx`），都需要调用昂贵的 Cache Flush 指令。对于高频交互的 Ring Buffer，这太慢了。不如直接关掉 Cache，虽然访问 DRAM 慢点，但省去了显式 Flush 的巨大开销。

## (2).payload (data buffer)

将一段由 CPU 使用的虚拟地址（VA）内存（通常是 Payload，如网络包或磁盘数据块），临时转换为设备可访问的 DMA 地址（DMA Address / IOVA），让设备可以对这块内存进行 IO 操作。

### payload 传输的步骤

**1. 映射(map)：**将 CPU 使用的虚拟地址（VA）内存映射为设备可以访问的 DMA 地址，将内存所有权交给 Device，映射之后 CPU 不能再读写这一片内存，直到内存被 unmap。

`dma_addr_t dma_handle = dma_map_single(dev, cpu_ptr, size, direction);`

| **步骤**                                               | **Hardware Coherent (最佳)**       | **Software Maintained (无一致性)** | **SWIOTLB (ARM CCA / 受限 DMA)**                                      |
| ------------------------------------------------------ | ---------------------------------- | ---------------------------------- | --------------------------------------------------------------------- |
| **1. 地址翻译**                                        | `virt_to_phys(ptr)` 拿到 PA。      | `virt_to_phys(ptr)` 拿到 PA。      | `virt_to_phys` 拿到 PA，发现是 **Private** (或地址过高设备无法访问)。 |
| **2. 缓冲区搬运**                                      | **无 (Zero Copy)**。直接原地使用。 | **无 (Zero Copy)**。直接原地使用。 | **申请 SWIOTLB 槽位 (Shared)**。                                      |
| 如果是 `TO_DEVICE`，执行 `memcpy(Private -> Shared)`。 |
| **3. 缓存维护**                                        | **无 (No-Op)**。                   |
| 硬件保证 Cache 中的脏数据会被设备嗅探到。              | **Clean (Flush)**。                |

执行 `DC CVAC`。
将 CPU Cache 中的脏数据强制刷入 DRAM，确保设备读到最新值。 | **视 SWIOTLB 槽位的一致性而定**。
如果槽位是 Non-Coherent，也需要 Flush SWIOTLB 对应的 Cache。 |
| **4. 返回地址** | 返回原始 PA (或 IOVA)。 | 返回原始 PA (或 IOVA)。 | 返回 **SWIOTLB 槽位的 PA (Shared)**。 |

**2.传输：**驱动将 DMA 地址发送给设备，设备对 data buffer 进行 IO 操作。

- **Coherent:** 设备读写请求经过总线，总线去 CPU Cache 里找数据。
- **Non-Coherent:** 设备直接读写 DRAM。
- **SWIOTLB:** 设备读写 SWIOTLB 里的 Shared 副本。

**3.解除映射(unmap)：**设备操作完成，将内存的**所有权**交还给 CPU，解除 swiotlb/iommu 等映射关系。

`dma_unmap_single(dev, dma_handle, size, direction);`

| **步骤**                                     | **路径 A: Hardware Coherent** | **路径 B: Software Maintained** | **路径 C: SWIOTLB (ARM CCA)** |
| -------------------------------------------- | ----------------------------- | ------------------------------- | ----------------------------- |
| **1. 缓存维护**                              | **无 (No-Op)**。              |
| CPU 直接读，硬件会保证读到设备刚写的新数据。 | **Invalidate**。              |

如果是 `FROM_DEVICE`，执行 `DC IVAC`。
废弃 CPU Cache 里的旧副本，强迫 CPU 下次读取时从 DRAM 拉取设备刚写的数据。 | **视 SWIOTLB 槽位的一致性而定**。
如有必要，Invalidate SWIOTLB 槽位的 Cache。 |
| **2. 缓冲区搬运** | **无**。 | **无**。 | **回拷 (Bounce Back)**。
如果是 `FROM_DEVICE`，执行 `memcpy(Shared -> Private)`。
把设备写的数据从 SWIOTLB 拷回 App 的缓冲区。 |
| **3. 资源释放** | 无。 | 无。 | 释放 SWIOTLB 槽位，归还给池子。 |

### payload 的缓存属性

### **A.支持 DMA-coherent(Hardware Coherent)**

**CPU Stage 1 属性:** **Write-Back (WB)**

**机制:** CPU 读写 Cache，硬件总线会自动把数据同步给设备。

### B.不支持 DMA-coherent(Software Coherent / Non-Coherent HW)

**CPU Stage 1 属性:** **Write-Back (WB)**

**机制:**

- 保持内存是 WB 属性。
- 在 DMA 开始前，内核调用`dma_map_single` —> `dma_sync_single_for_device` -> 执行 `DC CVAC` (Clean Cache) -> 把数据刷入 DRAM。
- 在 DMA 结束后，内核调用`dma_unmap_single` —> `dma_sync_single_for_cpu` -> 执行 `DC IVAC` (Invalidate Cache) -> 废弃 Cache 里的旧数据。

**原因:** 对于 data buffer 这样 IO 数据量大的内存，如果都设定为 NC，会严重影响 cpu 访存的速率，造成巨大开销。相比之下，每次 DMA 前后对缓存刷新和废弃更好。

| **典型用途**               | **场景类型**     | **是否支持硬件一致性**        | **Linux 内核 Stage 1 页表属性** | **如何保证一致性** |
| -------------------------- | ---------------- | ----------------------------- | ------------------------------- | ------------------ |
| Virtio Queue               | **一致性分配**   |
| (`dma_alloc_coherent`)     | **dma-coherent** | **Normal Write-Back (WB)**    | 硬件 (CCI/CMN)                  |
| Virtio Queue               | **一致性分配**   |
| (`dma_alloc_coherent`)     | **no-coherent**  | **Normal Non-Cacheable (NC)** | **页表属性 (Bypass Cache)**     |
|                            |                  |                               |                                 |                    |
| 网络包 Payload, 磁盘数据块 | **流式映射**     |
| (`dma_map_single`)         | **dma-coherent** | **Normal Write-Back (WB)**    | 硬件 (CCI/CMN)                  |
| 网络包 Payload, 磁盘数据块 | **流式映射**     |
| (`dma_map_single`)         | **no-coherent**  | **Normal Write-Back (WB)**    | **软件指令 (`DC C/IVAC`)**      |

# Force Write Back(FWB)

维护的核心是 **Host 与 Guest 之间因内存属性不匹配而可能产生的 coherence 问题**。

## 问题

对于虚拟机为设备分配的 DMA 内存，在 no-coherent 的情况下，virtio queue 分配的是 NC 属性的内存：

**Host 视角**：**Normal Memory, Write-Back (WB) Cacheable**。

**Guest 视角**：**Device Memory** 或 **Normal Memory, Non-Cacheable (NC)**。

因此 Host 优先从缓存读写数据，而 guest 从 DRAM 读写，出现不一致。

## FWB 的解决方法

无论 Guest 在 Stage-1 把这页内存标记为什么（即使 Guest 说是不可缓存的 Device Memory），CPU 都会**强制在 Stage-2**将最终属性视为 **Normal Memory, Write-Back，保证 Guest 和 Host 都从缓存中读写数据，保持 coherent。**

FWB 的主要目的是为了**消除为了保证 Host 和 guest 的 coherent，用软件手动刷 Cache 的开销**。

- **没有 FWB：** 为了保证数据一致性，Guest 发送数据给 Host 或者 Host 发送数据给 Guest 前，必须执行 `DC CIVAC`（Clean/Invalidate Cache）指令，把数据手动刷到 DRAM。这非常慢，严重影响性能。
- **有 FWB：** Hypervisor 强制都在 Cache 这一层读写。Host 写 Cache，Guest 读 Cache（通过 FWB 在 stage-2 的强制属性设置）。不需要手动刷内存，性能极高。

# OPENCCA 如何在没有 FWB 机制的情况下解决 Host 和 Guest 的 coherent 问题

## 1.在 guest mmu 未启动时

### 问题

在没有 FWB 的情况下，最大的风险在于 Guest 启动阶段（MMU/Cache 关闭，访问 DRAM）与 Host（开启 Cache）之间的数据不一致。

### 机制

RMM 在 `HCR_EL2` 寄存器中设置 `TVM` 位。这意味着当 Guest 尝试写 `SCTLR_EL1`（系统控制寄存器，控制 MMU 和 Cache 开关）、`TTBR`（页表基地址）等寄存器时，会触发异常陷入到 RMM。
Host 加载了 Kernel 镜像到 Host Cache 中。Guest 启动时是关 Cache 的，读 DRAM 读不到数据。当 Guest 决定开启 MMU/Cache 的一瞬间（写 `SCTLR_EL1`），陷入 RMM。RMM 执行 `flush_dcache_all`，把 Host 写的数据压进 DRAM。Guest 随后开启 MMU，就能正确读取数据了。

## 2.guest mmu 启用之后

Guest 成功开启了 MMU 和 Cache，RMM 认为 Guest 已经进入了正常的 Write-Back 模式（与 Host 一致）。

## 为什么出现之前错误

Guest：设备树中没有为 virtio 设备指定 DMA-coherent 属性，virtio 驱动分配 virtio queue 时分配为**Normal Non-Cacheable**属性。

Host：virtio 相关内存以**Normal Write-Back** 的属性读写。

在有 FWB 机制的情况下，FWB 强行让 guest 分配的 NC 类型的内存以 WB 方式读写，因此 Host 和 Guest 都是以 WB 格式读写。

在没有 FWB 机制下，必须让 virtio driver 以 WB 属性分配内存，指明 virtio 设备设定为 DMA-coherent。
