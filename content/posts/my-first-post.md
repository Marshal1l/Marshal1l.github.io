---
date: "2025-11-20T21:42:29+08:00"
draft: false
title: "VirtioCoherence"
---

# Virtio Queue 和 DMA Coherent

# DMA-coherent 决定设备 DMA 的 coherence 由硬件还是软件维护

在 ARM Linux 启动时，内核会解析 `.dtb` (Device Tree Blob) 文件。这是判定一致性模式的**第一权威来源**。

### `dma-coherent`

dma-coherent 是 Device Tree (DT) 中的一个布尔属性，用于描述 CPU 缓存（Cache） 与 DMA 设备（Device） 之间的内存数据一致性维护方式。它决定了作系统内核在进行 DMA 操作时，是依赖硬件自动同步，还是需要通过软件指令手动干预。

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

**2.传输：**驱动将 DMA 地址发送给设备，设备对 data buf