# GPGPU 虚拟加速器硬件手册（教学版）

> 来源：https://qemu.gevico.online/exercise/2026/stage1/gpu/gpu-datasheet/

**免责声明**：本手册描述的是用于 QEMU 训练营的**教学用虚拟 GPGPU 加速器**的规格与编程模型，并不对应任何实体芯片。训练营可能在不另行通知的情况下调整实现细节；若手册与实际实现不一致，以测题/参考实现为准。

| 项目 | 值 |
|------|-----|
| 目标读者 | 训练营学员、设备建模/驱动开发人员 |
| 文档状态 | Draft |
| 最后更新 | 2026-04-05 |

---

## 目录

- [1. 概述](#1-概述)
- [2. 文档约定](#2-文档约定)
- [3. 系统架构](#3-系统架构)
- [4. 地址空间与 BAR 映射](#4-地址空间与-bar-映射)
- [5. 设备信息寄存器 (0x0000)](#5-设备信息寄存器-0x0000)
- [6. 全局控制寄存器 (0x0100)](#6-全局控制寄存器-0x0100)
- [7. 中断控制寄存器 (0x0200)](#7-中断控制寄存器-0x0200)
- [8. 内核分发寄存器 (0x0300)](#8-内核分发寄存器-0x0300)
- [9. DMA 引擎寄存器 (0x0400)](#9-dma-引擎寄存器-0x0400)
- [10. SIMT 上下文寄存器 (0x1000)](#10-simt-上下文寄存器-0x1000)
- [11. 同步寄存器 (0x2000)](#11-同步寄存器-0x2000)
- [12. 执行模型](#12-执行模型)
- [13. 指令集](#13-指令集)
- [14. 限制与已知约束](#14-限制与已知约束)

---

## 1. 概述

GPGPU 是为 QEMU 训练营定制的虚拟 PCIe 3D 加速器设备。其设计目标是提供一个**覆盖 GPU 关键子系统**的硬件抽象，用于：

- PCIe 设备建模（BAR 映射、MSI-X 中断、配置空间）
- DMA 引擎建模（Host↔VRAM 双向传输）
- SIMT 执行模型（Warp 锁步、线程层次）
- RISC-V 指令集解释器（RV32I + RV32F + 低精度浮点扩展）

---

## 2. 文档约定

### 2.1 数值与端序

- 除非特别说明，所有数值均为十六进制，形如 `0x1001_8000`（下划线仅用于分组）。
- 地址均为**字节地址（byte addressable）**。
- 所有多字节寄存器访问均按**小端序（little-endian）** 解释。

### 2.2 术语

- **未定义（undefined）**：实现可以产生任意结果；软件不得依赖其行为。
- **实现定义（implementation-defined）**：实现必须在某处给出选择；若未给出，软件仍不得依赖。

---

## 3. 系统架构

### 3.1 顶层框图

```
+--------------------------------------------------------------------------+
|                      GPGPU Device Architecture                           |
|                                                                          |
|   Host CPU                                                               |
|      |                                                                   |
|      | PCIe Bus                                                          |
|      |                                                                   |
|   +--v-----------------------------------------------------------+       |
|   |                    GPGPU PCIe Device                         |       |
|   |  Vendor: 0x1234   Device: 0x1337   Class: 0x0302 (3D Ctrl)  |       |
|   |                                                              |       |
|   |  +------------+    +-------------+    +------------------+   |       |
|   |  |   BAR0     |    |    BAR2     |    |     BAR4         |   |       |
|   |  |  1 MB MMIO |    |  64 MB VRAM |    |  64 KB Doorbell  |   |       |
|   |  |  Ctrl Regs |    |  Video Mem  |    |  (Not Impl)      |   |       |
|   |  +-----+------+    +------+------+    +------------------+   |       |
|   |        |                  |                                  |       |
|   |  +-----v------------------v------+                           |       |
|   |  |       Execution Engine        |                           |       |
|   |  |  Grid → Block → Warp (32 lane)|                           |       |
|   |  |  RV32I + RV32F + LP-Float     |                           |       |
|   |  +-------------------------------+                           |       |
|   |                                                              |       |
|   |  +------------------+    +------------------+                |       |
|   |  |   DMA Engine     |    |   Interrupt Ctrl |                |       |
|   |  |  Host ↔ VRAM     |    |   MSI-X (4 vec)  |                |       |
|   |  +------------------+    +------------------+                |       |
|   +--------------------------------------------------------------+       |
+--------------------------------------------------------------------------+
```

### 3.2 PCI 配置

| 项目 | 值 |
|------|-----|
| Vendor ID | `0x1234` |
| Device ID | `0x1337` |
| Revision | `0x01` |
| Class Code | `0x0302` (PCI_CLASS_DISPLAY_3D) |
| 设备标识寄存器 | `0x4750_5055` ("GPPU") |
| 版本寄存器 | `0x0001_0000` (v1.0.0) |
| MSI-X | 4 向量 |

### 3.3 默认配置

| 参数 | 默认值 | QEMU 属性 |
|------|--------|-----------|
| Compute Units | 4 | `num_cus` |
| Warps / CU | 4 | `warps_per_cu` |
| Warp Size | 32 | `warp_size` |
| VRAM Size | 64 MiB | `vram_size` |

命令行示例：`-device gpgpu,num_cus=8,vram_size=128M`

---

## 4. 地址空间与 BAR 映射

### 4.1 BAR 总览

| BAR | 大小 | 类型 | 用途 |
|-----|------|------|------|
| BAR0 | 1 MiB | 64-bit MMIO | 控制/状态寄存器 |
| BAR2 | 64 MiB\* | 64-bit MMIO, 可预取 | VRAM（代码 + 数据） |
| BAR4 | 64 KiB | 32-bit MMIO | Doorbell（未实现） |

\*大小可通过 `vram_size` 属性配置。

### 4.2 BAR0 寄存器映射总览

| 区域 | Base | End | 说明 |
|------|------|-----|------|
| 设备信息 | `0x0000` | `0x00FF` | ID、版本、能力、VRAM 大小 |
| 全局控制 | `0x0100` | `0x01FF` | 使能、复位、状态、错误 |
| 中断控制 | `0x0200` | `0x02FF` | 中断使能、状态、确认 |
| 内核分发 | `0x0300` | `0x03FF` | 内核地址、Grid/Block 维度、DISPATCH |
| DMA 引擎 | `0x0400` | `0x04FF` | 源/目标地址、大小、控制、状态 |
| SIMT 上下文 | `0x1000` | `0x1FFF` | Thread/Block/Warp/Lane ID |
| 同步 | `0x2000` | `0x2FFF` | Barrier、Thread Mask |
| MSI-X 表 | `0xF_E000` | `0xF_EFFF` | MSI-X Table |
| MSI-X PBA | `0xF_F000` | `0xF_FFFF` | MSI-X PBA |

### 4.3 MMIO 访问规则

- BAR0 寄存器按 **32-bit** 对齐访问（min=4, max=4）。
- BAR2 VRAM 支持 1–8 字节访问。
- 对标记为 **Reserved** 的位：软件**必须写 0**，读出值为**实现定义**。

---

## 5. 设备信息寄存器 (0x0000)

只读。

| Offset | 寄存器 | 复位值 | 描述 |
|--------|--------|--------|------|
| `0x0000` | `DEV_ID` | `0x4750_5055` | 设备标识 "GPPU" |
| `0x0004` | `DEV_VERSION` | `0x0001_0000` | 版本 v1.0.0 |
| `0x0008` | `DEV_CAPS` | 见下 | 能力寄存器 |
| `0x000C` | `VRAM_SIZE_LO` | `0x0400_0000` | VRAM 大小低 32 位 |
| `0x0010` | `VRAM_SIZE_HI` | `0x0000_0000` | VRAM 大小高 32 位 |

### 5.1 `DEV_CAPS` (Offset `0x0008`) — 能力寄存器

| Bit | 名称 | 描述 |
|-----|------|------|
| `7:0` | `NUM_CUS` | Compute Unit 数量 |
| `15:8` | `WARPS_PER_CU` | 每 CU 的 Warp 数 |
| `23:16` | `WARP_SIZE` | Warp 宽度（线程数） |
| `31:24` | Reserved | 保留 |

---

## 6. 全局控制寄存器 (0x0100)

### 6.1 寄存器映射

| Offset | 寄存器 | 访问 | 复位值 | 描述 |
|--------|--------|------|--------|------|
| `0x0100` | `GLOBAL_CTRL` | R/W | `0x0000_0000` | 控制寄存器 |
| `0x0104` | `GLOBAL_STATUS` | R | `0x0000_0001` | 状态寄存器 |
| `0x0108` | `ERROR_STATUS` | R/W | `0x0000_0000` | 错误状态（W1C） |

### 6.2 `GLOBAL_CTRL` (Offset `0x0100`)

| Bit | 名称 | 访问 | 复位 | 描述 |
|-----|------|------|------|------|
| `31:2` | Reserved | - | `0` | 保留 |
| `1` | `RESET` | R/W | `0` | 软复位：写 `1` 触发复位，自动清除 |
| `0` | `ENABLE` | R/W | `0` | 设备使能 |

### 6.3 `GLOBAL_STATUS` (Offset `0x0104`)

| Bit | 名称 | 访问 | 复位 | 描述 |
|-----|------|------|------|------|
| `31:3` | Reserved | R | `0` | 保留 |
| `2` | `ERROR` | R | `0` | 错误标志 |
| `1` | `BUSY` | R | `0` | 内核执行中 |
| `0` | `READY` | R | `1` | 设备就绪 |

### 6.4 `ERROR_STATUS` (Offset `0x0108`)

| Bit | 名称 | 访问 | 复位 | 描述 |
|-----|------|------|------|------|
| `31:4` | Reserved | - | `0` | 保留 |
| `3` | `DMA_FAULT` | R/W | `0` | DMA 错误；写 `1` 清除 |
| `2` | `KERNEL_FAULT` | R/W | `0` | 内核执行错误；写 `1` 清除 |
| `1` | `VRAM_FAULT` | R/W | `0` | VRAM 越界访问；写 `1` 清除 |
| `0` | `INVALID_CMD` | R/W | `0` | 无效命令；写 `1` 清除 |

### 6.5 软复位行为

写 `GLOBAL_CTRL.RESET = 1` 后：
- `global_ctrl` 清零
- `global_status` 恢复 `READY`
- 所有错误/中断状态清零
- SIMT 上下文清零
- 内核参数、DMA 状态清零
- VRAM **不**清零

---

## 7. 中断控制寄存器 (0x0200)

### 7.1 寄存器映射

| Offset | 寄存器 | 访问 | 复位值 | 描述 |
|--------|--------|------|--------|------|
| `0x0200` | `IRQ_ENABLE` | R/W | `0x0000_0000` | 中断使能掩码 |
| `0x0204` | `IRQ_STATUS` | R | `0x0000_0000` | 中断挂起状态 |
| `0x0208` | `IRQ_ACK` | W | — | 写 `1` 清除对应中断 |

### 7.2 中断位定义

| Bit | 名称 | MSI-X 向量 | 描述 |
|-----|------|-----------|------|
| `0` | `KERNEL_DONE` | 0 | 内核执行完成 |
| `1` | `DMA_DONE` | 1 | DMA 传输完成 |
| `2` | `ERROR` | 2 | 错误条件 |

中断递送优先级：MSI-X > MSI > Legacy INTx。

---

## 8. 内核分发寄存器 (0x0300)

### 8.1 寄存器映射

| Offset | 寄存器 | 访问 | 复位值 | 描述 |
|--------|--------|------|--------|------|
| `0x0300` | `KERNEL_ADDR_LO` | R/W | `0` | 内核代码地址低 32 位（VRAM 内） |
| `0x0304` | `KERNEL_ADDR_HI` | R/W | `0` | 内核代码地址高 32 位 |
| `0x0308` | `KERNEL_ARGS_LO` | R/W | `0` | 内核参数地址低 32 位（预留） |
| `0x030C` | `KERNEL_ARGS_HI` | R/W | `0` | 内核参数地址高 32 位（预留） |
| `0x0310` | `GRID_DIM_X` | R/W | `0` | Grid X 维度 |
| `0x0314` | `GRID_DIM_Y` | R/W | `0` | Grid Y 维度 |
| `0x0318` | `GRID_DIM_Z` | R/W | `0` | Grid Z 维度 |
| `0x031C` | `BLOCK_DIM_X` | R/W | `0` | Block X 维度 |
| `0x0320` | `BLOCK_DIM_Y` | R/W | `0` | Block Y 维度 |
| `0x0324` | `BLOCK_DIM_Z` | R/W | `0` | Block Z 维度 |
| `0x0328` | `SHARED_MEM_SIZE` | R/W | `0` | 共享内存大小（预留，未实现） |
| `0x0330` | `DISPATCH` | W | — | 写任意值触发内核执行 |

### 8.2 编程模型

1. 通过 DMA 或直接 MMIO 将内核代码写入 VRAM。
2. 配置 `KERNEL_ADDR` 指向内核代码在 VRAM 中的偏移。
3. 配置 `GRID_DIM_*` 和 `BLOCK_DIM_*`（所有维度必须 > 0）。
4. 写 `DISPATCH` 寄存器（任意值）触发执行。
5. 设备设置 `BUSY`，顺序执行所有 Block 的所有 Warp。
6. 完成后清除 `BUSY`，置位 `IRQ_STATUS.KERNEL_DONE`。

**Note**：当设备未使能、正在忙、Grid/Block 维度为零、或内核地址越界时，DISPATCH 写入将被忽略并设置 `ERROR_STATUS.INVALID_CMD`。

---

## 9. DMA 引擎寄存器 (0x0400)

### 9.1 寄存器映射

| Offset | 寄存器 | 访问 | 复位值 | 描述 |
|--------|--------|------|--------|------|
| `0x0400` | `DMA_SRC_LO` | R/W | `0` | 源地址低 32 位 |
| `0x0404` | `DMA_SRC_HI` | R/W | `0` | 源地址高 32 位 |
| `0x0408` | `DMA_DST_LO` | R/W | `0` | 目标地址低 32 位 |
| `0x040C` | `DMA_DST_HI` | R/W | `0` | 目标地址高 32 位 |
| `0x0410` | `DMA_SIZE` | R/W | `0` | 传输大小（字节） |
| `0x0414` | `DMA_CTRL` | R/W | `0` | DMA 控制 |
| `0x0418` | `DMA_STATUS` | R | `0` | DMA 状态 |

### 9.2 `DMA_CTRL` (Offset `0x0414`)

| Bit | 名称 | 访问 | 复位 | 描述 |
|-----|------|------|------|------|
| `31:3` | Reserved | - | `0` | 保留 |
| `2` | `IRQ_EN` | R/W | `0` | 完成时产生中断 |
| `1` | `DIR` | R/W | `0` | 方向：`0` Host→VRAM；`1` VRAM→Host |
| `0` | `START` | R/W | `0` | 写 `1` 启动传输 |

### 9.3 `DMA_STATUS` (Offset `0x0418`)

| Bit | 名称 | 访问 | 复位 | 描述 |
|-----|------|------|------|------|
| `31:3` | Reserved | R | `0` | 保留 |
| `2` | `ERROR` | R | `0` | 传输错误 |
| `1` | `COMPLETE` | R | `0` | 传输完成 |
| `0` | `BUSY` | R | `0` | 传输进行中 |

### 9.4 编程模型

- **Host→VRAM**：`SRC` = 主机物理地址，`DST` = VRAM 偏移，`DIR = 0`。
- **VRAM→Host**：`SRC` = VRAM 偏移，`DST` = 主机物理地址，`DIR = 1`。
- 写 `DMA_CTRL.START = 1` 触发传输。数据同步拷贝，完成通过 1ms 虚拟定时器通知。

---

## 10. SIMT 上下文寄存器 (0x1000)

### 10.1 寄存器映射

| Offset | 寄存器 | 访问 | 描述 |
|--------|--------|------|------|
| `0x1000` | `THREAD_ID_X` | R/W | 线程索引 X |
| `0x1004` | `THREAD_ID_Y` | R/W | 线程索引 Y |
| `0x1008` | `THREAD_ID_Z` | R/W | 线程索引 Z |
| `0x1010` | `BLOCK_ID_X` | R/W | Block 索引 X |
| `0x1014` | `BLOCK_ID_Y` | R/W | Block 索引 Y |
| `0x1018` | `BLOCK_ID_Z` | R/W | Block 索引 Z |
| `0x1020` | `WARP_ID` | R/W | Warp 索引 |
| `0x1024` | `LANE_ID` | R/W | Lane 索引 (0–31) |

**Note**：主机侧通过 MMIO 可读写这些寄存器。GPU 核心通过内部 CTRL 地址空间（`0x8000_0000+`）只读访问。

---

## 11. 同步寄存器 (0x2000)

| Offset | 寄存器 | 访问 | 描述 |
|--------|--------|------|------|
| `0x2000` | `BARRIER` | W | 写任意值发出 Barrier 到达信号 |
| `0x2004` | `THREAD_MASK` | R/W | 32-bit 活跃线程掩码（每 bit 对应一个 lane） |

---

## 12. 执行模型

### 12.1 线程层次

```
Grid (grid_x × grid_y × grid_z 个 Block)
 └─ Block (block_x × block_y × block_z 个线程)
     └─ Warp (最多 32 个 Lane, 锁步执行)
         └─ Lane (独立寄存器文件 + PC)
```

### 12.2 每 Lane 寄存器

| 寄存器 | 数量 | 宽度 | 描述 |
|--------|------|------|------|
| `x0–x31` | 32 | 32-bit | 通用寄存器（x0 恒为 0） |
| `f0–f31` | 32 | 32-bit | 浮点寄存器（IEEE 754 单精度） |
| `pc` | 1 | 32-bit | 程序计数器 |
| `fcsr` | 1 | 8-bit | 浮点控制状态 |
| `mhartid` | 1 | 32-bit | Hart 标识（编码见下） |

### 12.3 mhartid 编码

```
Bit [31:13]  block_id_linear   (19 位，最多 524,287 个 Block)
Bit [12:5]   warp_id           (8 位，最多 256 个 Warp)
Bit [4:0]    thread_id (lane)  (5 位，0–31)
```

GPU 核心通过 `csrrs x6, mhartid, x0` 读取，用 `andi x6, x6, 0x1F` 提取 lane ID。

### 12.4 Warp 执行规则

- 所有活跃 Lane 执行同一条指令（SIMT 锁步）。
- Lane 遇到 `ebreak` 或 `ret`（ra=0）时退出，对应 `active_mask` 位清零。
- 当 `active_mask == 0` 时 Warp 执行结束。
- 每 Warp 最大 **100,000 周期**，超出则报 `KERNEL_FAULT`。

### 12.5 GPU 核心内存地址空间

| 地址范围 | 说明 |
|----------|------|
| `0x0000_0000` – `VRAM_SIZE-1` | VRAM 读写 |
| `0x8000_0000` + offset | CTRL 寄存器（只读：threadIdx, blockIdx, blockDim, gridDim） |

---

## 13. 指令集

### 13.1 RV32I 基础整数指令

| 类型 | 指令 | Opcode |
|------|------|--------|
| 上立即数 | LUI, AUIPC | `0x37`, `0x17` |
| 跳转 | JAL, JALR | `0x6F`, `0x67` |
| 分支 | BEQ, BNE, BLT, BGE, BLTU, BGEU | `0x63` |
| 访存 | LB, LH, LW, LBU, LHU | `0x03` |
| 存储 | SB, SH, SW | `0x23` |
| 立即算术 | ADDI, SLTI, SLTIU, XORI, ORI, ANDI, SLLI, SRLI, SRAI | `0x13` |
| 寄存器算术 | ADD, SUB, SLL, SLT, SLTU, XOR, SRL, SRA, OR, AND | `0x33` |
| 系统 | EBREAK, CSRRW, CSRRS, CSRRC, CSRRWI, CSRRSI, CSRRCI | `0x73` |

**支持的 CSR：**

| 地址 | 名称 | 访问 | 描述 |
|------|------|------|------|
| `0xF14` | `mhartid` | RO | Hart 标识 |
| `0x001` | `fflags` | R/W | 浮点异常标志 NX\|UF\|OF\|DZ\|NV |
| `0x002` | `frm` | R/W | 浮点舍入模式 |
| `0x003` | `fcsr` | R/W | 完整浮点控制状态 |

### 13.2 RV32F 单精度浮点指令

| 类型 | 指令 | funct7 |
|------|------|--------|
| 访存 | FLW, FSW | opcode `0x07`/`0x27` |
| 算术 | FADD.S, FSUB.S, FMUL.S, FDIV.S, FSQRT.S | `0x00`–`0x2C` |
| 融合乘加 | FMADD.S, FMSUB.S, FNMSUB.S, FNMADD.S | opcode `0x43`–`0x4F` |
| 符号注入 | FSGNJ.S, FSGNJN.S, FSGNJX.S | `0x10` |
| 极值 | FMIN.S, FMAX.S | `0x14` |
| 比较 | FEQ.S, FLT.S, FLE.S | `0x50` |
| 转换 | FCVT.W.S, FCVT.WU.S, FCVT.S.W, FCVT.S.WU | `0x60`/`0x68` |
| 移动/分类 | FMV.X.W, FMV.W.X, FCLASS.S | `0x70`/`0x78` |

**舍入模式（rm 字段）：**

| rm | 模式 | 描述 |
|----|------|------|
| 0 | RNE | 最近偶数舍入 |
| 1 | RTZ | 向零舍入 |
| 2 | RDN | 向负无穷舍入 |
| 3 | RUP | 向正无穷舍入 |
| 4 | RMM | 最近最大值舍入（近似为 RNE） |
| 7 | Dynamic | 使用 fcsr.frm |

### 13.3 低精度浮点扩展（自定义）

所有指令均在 OP_FP (opcode `0x53`) 编码空间内，通过 funct7 和 rs2 字段区分。

#### 13.3.1 BF16 (Bfloat16) — funct7 = `0x22`

格式：sign(1) + exp(8, bias=127) + mantissa(7) = 16 bit

| rs2 | 助记符 | 编码示例 | 描述 |
|-----|--------|----------|------|
| 0 | `FCVT.S.BF16 fd, fs1` | `0x4401_01D3` | BF16 → FP32 |
| 1 | `FCVT.BF16.S fd, fs1` | `0x4410_8153` | FP32 → BF16 |

#### 13.3.2 FP8 E4M3 / E5M2 — funct7 = `0x24`

- **E4M3**: sign(1) + exp(4, bias=7) + mantissa(3) = 8 bit, 最大值 ±448, 无 Inf
- **E5M2**: sign(1) + exp(5, bias=15) + mantissa(2) = 8 bit, 有 Inf/NaN

| rs2 | 助记符 | 描述 |
|-----|--------|------|
| 0 | `FCVT.S.E4M3 fd, fs1` | E4M3 → FP32（经 BF16 中转） |
| 1 | `FCVT.E4M3.S fd, fs1` | FP32 → E4M3（饱和模式） |
| 2 | `FCVT.S.E5M2 fd, fs1` | E5M2 → FP32（经 BF16 中转） |
| 3 | `FCVT.E5M2.S fd, fs1` | FP32 → E5M2（饱和模式） |

**Note**：E4M3 下转换时，超出 ±448 的有限值饱和到 ±448；+Inf/-Inf 也饱和到 ±448。E5M2 下转换时，超出范围的有限值饱和到格式最大值；Inf 保持为 Inf。

#### 13.3.3 FP4 E2M1 — funct7 = `0x26`

格式：sign(1) + exp(2, bias=1) + mantissa(1) = 4 bit

可表示的正数值：

| 编码 (3bit) | 值 |
|-------------|-----|
| `000` | 0 |
| `001` | 0.5 |
| `010` | 1.0 |
| `011` | 1.5 |
| `100` | 2.0 |
| `101` | 3.0 |
| `110` | 4.0 |
| `111` | 6.0 |

| rs2 | 助记符 | 描述 |
|-----|--------|------|
| 0 | `FCVT.S.E2M1 fd, fs1` | E2M1 → FP32（链：E2M1→E4M3→BF16→FP32） |
| 1 | `FCVT.E2M1.S fd, fs1` | FP32 → E2M1（手写阈值舍入，饱和到 ±6.0） |

**Note**：E2M1 无 Inf/NaN 表示。NaN 和 Inf 输入饱和到 ±6.0。超出 ±6.0 的有限值也饱和到 ±6.0。

---

## 14. 限制与已知约束

| 约束 | 说明 |
|------|------|
| Warp 大小固定 32 | 不可运行时修改 |
| 串行执行 | Warp/Block 顺序执行，非并行仿真 |
| num_cus / warps_per_cu | 仅影响 `DEV_CAPS` 寄存器，不影响调度 |
| Shared Memory | 寄存器已定义，功能未实现 |
| Kernel Args | 寄存器已定义，执行引擎未使用 |
| Barrier | 跨 Warp 屏障在串行模型下会死锁 |
| Doorbell BAR4 | 未实现（读写产生 LOG_UNIMP） |
| GPU 核心寻址 | 仅 32 位 |
| DMA 延迟 | 固定 1ms 虚拟时间 |
| RMM 舍入 | 近似为 RNE |
| Thread ID Y/Z | GPU 核心侧始终返回 0 |
