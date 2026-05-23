# GPGPU 虚拟加速器实验手册

> 来源：https://qemu.gevico.online/exercise/2026/stage1/gpu/gpu-exper-manual/

---

本文档是专业阶段 GPGPU 方向的实验手册。

专业阶段 GPGPU 方向的实验围绕虚拟 PCIe 3D 加速器的设备建模展开，你需要按照 [GPGPU 虚拟加速器硬件手册](./gpu-datasheet.md) 给出的硬件参数，完成 PCI 设备注册、MMIO 寄存器读写、VRAM 访问与 DMA、SIMT 上下文管理、RV32I/RV32F 指令解释器、以及低精度浮点扩展等实验任务。

---

## 目录

- [环境搭建](#环境搭建)
- [提交代码](#提交代码)
- [测评验收](#测评验收)
- [实验介绍](#实验介绍)
- [进阶实验](#进阶实验)

---

## 环境搭建

第一步，安装 QEMU 开发依赖。

```bash
# Ubuntu 24.04
sudo sed -i 's/^Types: deb$/Types: deb deb-src/' /etc/apt/sources.list.d/ubuntu.sources
sudo apt-get update
sudo apt-get build-dep -y qemu

# 安装 Rust 工具链
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
. "$HOME/.cargo/env"
cargo install bindgen-cli
```

> **提示**：GPGPU 方向使用 QTest 测试框架，测题在宿主机侧编译运行，**不需要** RISC-V 交叉编译工具链。

第二步，点击 GitHub Classroom 邀请链接 加入实验，系统会自动在组织下为你创建专属仓库并赋予 maintainer 权限。

> **注意**：请通过上方链接获取仓库，**不支持手动 fork**。

第三步，clone 仓库到本地：

```bash
git clone git@github.com:gevico/qemu-camp-2026-exper-<你的 github 用户名>.git
```

第四步，添加上游远程仓库，用于同步上游代码变更：

```bash
git remote add upstream git@github.com:gevico/gevico-classroom-qemu-camp-2026-exper-qemu-camp-2026-exper.git
git pull upstream main --rebase
```

> **提示**：使用 SSH 地址需要在 GitHub 上配置 SSH Key，请参考 [GitHub SSH Key 配置指南](https://docs.github.com/zh/authentication/connecting-to-github-with-ssh)。

第五步，配置并编译：

```bash
make -f Makefile.camp configure
make -f Makefile.camp build
```

---

## 提交代码

所有实验的测题源码，均放在仓库路径：`tests/qtest/gpgpu-test.c`。

GPGPU 方向使用 QEMU 的 **QTest 测试框架**。QTest 测题在宿主机侧编译和运行，通过 QOS 图框架与 QEMU 进程通信，直接读写 MMIO 寄存器（`qpci_io_readl` / `qpci_io_writel`）来验证设备模型的行为，无需编写客户机程序。

你需要熟读每个测题源码，理解每个测题的测试意图，并实现对应的 QEMU 建模功能（需要修改 `hw/gpgpu/` 下的设备源码，非测题源码），文末会给出具体实验的介绍，辅助你阅读测题源码。

每次实验完成后，需要将你的代码提交到你的仓库中。

```bash
git add .
git commit -m "feat: subject..."
git push origin main
```

> **Note**：请确保你的代码符合仓库的代码规范，包括代码格式、注释等。

---

## 测评验收

本地运行全部测题的方式：

```bash
# 使用 Makefile.camp（推荐，自动计算得分）
make -f Makefile.camp test-gpgpu

# 使用 qos-test 直接运行全部 GPGPU 测题
cd build
QTEST_QEMU_BINARY=./qemu-system-riscv64 ./tests/qtest/qos-test -p /riscv64/virt/generic-pcihost/pci-bus-generic/pci-bus/gpgpu

# 运行单个子测试（通过 QOS 路径过滤）
QTEST_QEMU_BINARY=./qemu-system-riscv64 ./build/tests/qtest/qos-test -p /riscv64/virt/generic-pcihost/pci-bus-generic/pci-bus/gpgpu/gpgpu-tests/device-id
```

GPGPU 方向全部测题通过的情况下，你会看到如下输出：

```
ok 1 /riscv64/.../gpgpu/gpgpu-tests/device-id
ok 2 /riscv64/.../gpgpu/gpgpu-tests/vram-size
ok 3 /riscv64/.../gpgpu/gpgpu-tests/global-ctrl
ok 4 /riscv64/.../gpgpu/gpgpu-tests/dispatch-regs
ok 5 /riscv64/.../gpgpu/gpgpu-tests/vram-access
ok 6 /riscv64/.../gpgpu/gpgpu-tests/dma-regs
ok 7 /riscv64/.../gpgpu/gpgpu-tests/irq-regs
ok 8 /riscv64/.../gpgpu/gpgpu-tests/simt-thread-id
ok 9 /riscv64/.../gpgpu/gpgpu-tests/simt-block-id
ok 10 /riscv64/.../gpgpu/gpgpu-tests/simt-warp-lane
ok 11 /riscv64/.../gpgpu/gpgpu-tests/simt-thread-mask
ok 12 /riscv64/.../gpgpu/gpgpu-tests/simt-reset
ok 13 /riscv64/.../gpgpu/gpgpu-tests/kernel-exec
ok 14 /riscv64/.../gpgpu/gpgpu-tests/fp-kernel-exec
ok 15 /riscv64/.../gpgpu/gpgpu-tests/lp-convert
ok 16 /riscv64/.../gpgpu/gpgpu-tests/lp-convert-e5m2-e2m1
ok 17 /riscv64/.../gpgpu/gpgpu-tests/lp-convert-saturate
```

如果你想运行某个测例，比如 `kernel-exec`，可以使用如下命令：

```bash
cd build
QTEST_QEMU_BINARY=./qemu-system-riscv64 ./tests/qtest/qos-test -p /riscv64/virt/generic-pcihost/pci-bus-generic/pci-bus/gpgpu/gpgpu-tests/kernel-exec
```

如果你想调试某个测例的设备模型，可以结合 GDB 在 QEMU 源码中设断点。由于 QTest 框架在宿主机侧运行，你可以直接在另一个终端用 GDB 附加到运行中的 QEMU 进程。

> **Note**：你需要熟读 [GPGPU 虚拟加速器硬件手册](./gpu-datasheet.md) 和测题的源码，来理解每个实验的测试意图，这会极大地方便你调试，提高开发效率。

---

## 实验介绍

GPGPU 方向的全部实验围绕虚拟 PCIe 3D 加速器的设备建模与 SIMT 执行引擎展开，涵盖 PCI 设备注册、MMIO 寄存器、VRAM 访问与 DMA、SIMT 上下文、RV32I 整数指令解释器、Warp 执行循环、RV32F 浮点指令、以及低精度浮点转换（BF16/FP8/FP4）等功能。

所有外设的地址映射、寄存器定义和编程模型，详见 [GPGPU 虚拟加速器硬件手册](./gpu-datasheet.md)。

所有实验的测题，均在 `tests/qtest/gpgpu-test.c` 文件中，通过 `qos_add_test` 注册。

### 实验一 test-device-id

| 项目 | 内容 |
|------|------|
| 测试函数 | `gpgpu_test_device_id` |
| 功能描述 | 验证 GPGPU PCI 设备基本工作，包括设备使能、BAR0 映射、设备标识和版本号寄存器 |
| 基础代码 | `hw/gpgpu/gpgpu.c`（PCI realize、BAR 注册） |
| 详细规格 | GPGPU 硬件手册 §3, §5 |

**测试步骤：**

| 测试内容 | 预期结果 |
|----------|----------|
| 读取 `DEV_ID` (0x0000) | 返回 `0x47505055` ("GPPU") |
| 读取 `DEV_VERSION` (0x0004) | 返回 `0x00010000` (v1.0.0) |

### 实验二 test-vram-size

| 项目 | 内容 |
|------|------|
| 测试函数 | `gpgpu_test_vram_size` |
| 功能描述 | 验证 VRAM 大小寄存器返回正确的配置值 |
| 基础代码 | `hw/gpgpu/gpgpu.c`（`gpgpu_ctrl_read` 中 `VRAM_SIZE_LO/HI` 分支） |
| 详细规格 | GPGPU 硬件手册 §5.1 |

**测试步骤：**

| 测试内容 | 预期结果 |
|----------|----------|
| 读取 `VRAM_SIZE_LO` (0x000C) + `VRAM_SIZE_HI` (0x0010) | 组合 64 位值，等于 64 MiB (`0x04000000`) |

### 实验三 test-global-ctrl

| 项目 | 内容 |
|------|------|
| 测试函数 | `gpgpu_test_global_ctrl` |
| 功能描述 | 验证全局控制和状态寄存器工作正常 |
| 基础代码 | `hw/gpgpu/gpgpu.c`（`gpgpu_ctrl_read/write` 中 `GLOBAL_CTRL/STATUS` 分支） |
| 详细规格 | GPGPU 硬件手册 §6 |

**测试步骤：**

| 测试内容 | 预期结果 |
|----------|----------|
| 读取 `GLOBAL_STATUS` | 初始状态应包含 `READY` (bit 0) |
| 写入 `GLOBAL_CTRL = ENABLE` | 使能设备 |
| 读回 `GLOBAL_CTRL` | 验证 `ENABLE` 位已设置 |

### 实验四 test-dispatch-regs

| 项目 | 内容 |
|------|------|
| 测试函数 | `gpgpu_test_dispatch_regs` |
| 功能描述 | 验证内核分发配置寄存器（Grid/Block 维度）可正确读写 |
| 基础代码 | `hw/gpgpu/gpgpu.c`（`gpgpu_ctrl_read/write` 中 Grid/Block 维度偏移） |
| 详细规格 | GPGPU 硬件手册 §8 |

**测试步骤：**

| 测试内容 | 预期结果 |
|----------|----------|
| 写入 `GRID_DIM_X=64, Y=32, Z=1` | 读回验证 |
| 写入 `BLOCK_DIM_X=256, Y=1, Z=1` | 读回验证 |

### 实验五 test-vram-access

| 项目 | 内容 |
|------|------|
| 测试函数 | `gpgpu_test_vram_access` |
| 功能描述 | 验证 VRAM 区域 (BAR2) 可正确读写 |
| 基础代码 | `hw/gpgpu/gpgpu.c`（`gpgpu_vram_read/write`、BAR2 注册） |
| 详细规格 | GPGPU 硬件手册 §4.1 |

**测试步骤：**

| 测试内容 | 预期结果 |
|----------|----------|
| 写入 `0xDEADBEEF` 到偏移 `0x0` | 读回验证 |
| 写入 `0x12345678` 到偏移 `0x100` | 读回验证 |
| 写入 `0xCAFEBABE` 到偏移 `0x1000` | 读回验证 |

### 实验六 test-dma-regs + test-irq-regs

| 项目 | 内容 |
|------|------|
| 测试函数 | `gpgpu_test_dma_regs`, `gpgpu_test_irq_regs` |
| 功能描述 | 验证 DMA 控制寄存器可正确配置，中断使能/状态寄存器工作正常 |
| 基础代码 | `hw/gpgpu/gpgpu.c`（`gpgpu_ctrl_read/write` 中 DMA/IRQ 偏移） |
| 详细规格 | GPGPU 硬件手册 §7, §9 |

**测试步骤：**

| 测试内容 | 预期结果 |
|----------|----------|
| 写入 DMA `SRC=0x1000, DST=0x2000, SIZE=4096` | 读回验证所有字段 |
| 写入 `IRQ_ENABLE = 0x7` | 读回验证，读 `IRQ_STATUS` 应为 0 |

### 实验七 test-simt-thread-id ~ test-simt-reset（共 5 个测例）

| 项目 | 内容 |
|------|------|
| 测试函数 | `gpgpu_test_thread_id_regs`, `gpgpu_test_block_id_regs`, `gpgpu_test_warp_lane_regs`, `gpgpu_test_thread_mask_reg`, `gpgpu_test_simt_reset` |
| 功能描述 | 验证 SIMT 上下文寄存器（Thread/Block/Warp/Lane ID、Thread Mask）可读写，以及软复位清除所有 SIMT 状态 |
| 基础代码 | `hw/gpgpu/gpgpu.c`（SIMT 寄存器读写、`gpgpu_reset` 中的上下文清零） |
| 详细规格 | GPGPU 硬件手册 §10, §11 |

**测试用例：**

| 测试名称 | 测试内容 |
|----------|----------|
| `test_thread_id` | 写/读 `THREAD_ID_X=15, Y=7, Z=3`，验证初始值为 0 |
| `test_block_id` | 写/读 `BLOCK_ID_X=63, Y=31, Z=1` |
| `test_warp_lane` | 写/读 `WARP_ID=3, LANE_ID=17` |
| `test_thread_mask` | 读初始值 0，写入 `0xFFFFFFFF` 和 `0x0000FFFF` 读回验证 |
| `test_simt_reset` | 设置上下文后触发软复位（`GLOBAL_CTRL.RESET=1`），验证所有 SIMT 寄存器清零 |

### 实验八 test-kernel-exec

| 项目 | 内容 |
|------|------|
| 测试函数 | `gpgpu_test_kernel_exec` |
| 功能描述 | 验证 RV32I 指令解释器和 SIMT 执行模型。上传一个整数 kernel 到 VRAM，每个线程将自己的 lane ID 写入输出数组 |
| 基础代码 | `hw/gpgpu/gpgpu_core.c`（`exec_one_inst` RV32I 分支、`gpgpu_core_exec_warp`、`gpgpu_core_exec_kernel`）、`hw/gpgpu/gpgpu.c`（`gpgpu_dispatch_kernel`） |
| 详细规格 | GPGPU 硬件手册 §12, §13.1 |

**测试步骤：**

| 步骤 | 操作 |
|------|------|
| 1 | 使能设备，上传 7 条指令的 kernel 到 VRAM |
| 2 | 配置 Grid(1×1×1)、Block(8×1×1) |
| 3 | 触发 DISPATCH |
| 4 | 等待完成（STATUS 恢复 READY） |
| 5 | 验证输出数组 `C[i] == i` 对 i=0..7 |

**内核功能**：`C[lane_id] = lane_id`，使用 `csrrs mhartid`、`andi`、`slli`、`lui`、`add`、`sw`、`ebreak` 指令。

### 实验九 test-fp-kernel-exec

| 项目 | 内容 |
|------|------|
| 测试函数 | `gpgpu_test_fp_kernel_exec` |
| 功能描述 | 验证 RV32F 浮点指令。上传一个浮点 kernel，每个线程计算 `(int)(tid * 2.0 + 1.0)` |
| 基础代码 | `hw/gpgpu/gpgpu_core.c`（`exec_one_inst` 中 `OPCODE_OP_FP` 分支、浮点寄存器 fpr） |
| 详细规格 | GPGPU 硬件手册 §13.2 |

**测试步骤：**

| 步骤 | 操作 |
|------|------|
| 1 | 使能设备，上传 13 条指令的 kernel 到 VRAM |
| 2 | 配置 Grid(1×1×1)、Block(8×1×1) |
| 3 | 触发 DISPATCH |
| 4 | 等待完成 |
| 5 | 验证输出数组 `output[i] == 2*i + 1` 对 i=0..7 |

**内核功能**：`output[tid] = (int)(tid * 2.0 + 1.0)`，使用 `fcvt.s.w`、`fmul.s`、`fadd.s`、`fcvt.w.s` 等 RV32F 指令。

### 实验十 test-lp-convert / test-lp-convert-e5m2-e2m1 / test-lp-convert-saturate（共 3 个测例）

| 项目 | 内容 |
|------|------|
| 测试函数 | `gpgpu_test_lp_convert`, `gpgpu_test_lp_convert_e5m2_e2m1`, `gpgpu_test_lp_convert_saturate` |
| 功能描述 | 验证低精度浮点转换扩展：BF16、FP8 (E4M3/E5M2)、FP4 (E2M1) 的往返转换精度、负数处理、以及溢出饱和行为 |
| 基础代码 | `hw/gpgpu/gpgpu_core.c`（`exec_one_inst` 中 `FUNCT7_FCVT_BF16/FP8/FP4` 分支、`float32_to_float4_e2m1` 手写转换） |
| 详细规格 | GPGPU 硬件手册 §13.3 |

**测试用例：**

| 测试名称 | 测试内容 |
|----------|----------|
| `lp-convert` | BF16 往返 `42 → bf16 → f32 → 42`；E4M3 往返 `2 → e4m3 → f32 → 2`。4 线程验证一致性 |
| `lp-convert-e5m2-e2m1` | E5M2 往返 `4→4`；E2M1 往返 `2→2`；BF16 负数 `-3→-3`；E4M3 负数 `-2→-2`。单线程 4 输出 |
| `lp-convert-saturate` | E4M3 零值 `0→0`；E2M1 零值 `0→0`；E2M1 溢出饱和 `100→6`；E4M3 溢出饱和 `1000→448`；E4M3 Inf 饱和 `+Inf→448`。单线程 5 输出 |

> **Note**：E2M1 的 `f32 → e2m1` 转换是手写实现（非 softfloat），你需要理解 E2M1 的 4-bit 格式并用 FP32 位模式阈值实现正确的舍入与饱和逻辑。

---

## 进阶实验

> **说明**：进阶实验为开放题目，不计入 100 分基础测评，但会作为训练营评优与推荐的重要参考。这两道题目在基础实验的 GPGPU 设备模型之上，向「软件栈」与「真实 cmodel 集成」两个方向延伸，难度显著提升。

### 进阶实验一 设计类 CUDA 的 GPGPU 软件栈

基础实验只验证了设备端（QEMU 里的 GPGPU 模型）的正确性，测题通过 QTest 直接写 MMIO 寄存器触发 kernel，并没有驱动与编程模型的概念。这道题要你补齐软件侧。

**挑战目标**：为本方向实现的虚拟 GPGPU 设计一套**类 CUDA 风格**的最小软件栈，包含：

- **内核驱动**：在 Linux 或 ArceOS/rCore 中注册 PCI 设备，管理 BAR、MSI-X、DMA 描述符，暴露字符设备或系统调用接口。
- **用户态运行时库**（libgpgpu）：封装 `gpgpuMalloc` / `gpgpuMemcpy` / `gpgpuLaunchKernel` 等 API，内部通过 ioctl + mmap 与驱动通信。
- **编程模型**：规定 kernel 函数签名（`__global__` 风格）、grid/block 维度、内置变量（threadIdx/blockIdx）；提供一个简易前端（可以是 clang 插件、或仅约定手写 RV32 汇编 + 头文件）。
- **Demo**：至少跑通 vector add、矩阵乘、ReLU 三个 kernel，端到端验证 Host → Device → Host 数据链路。

**评估维度**：软件栈的层次清晰度、API 设计的简洁性、以及对基础 17 道 QTest 所建模能力的覆盖程度。

### 进阶实验二 把 Vortex simx 集成进 QEMU 并适配 AI 软件栈

基础实验中 `hw/gpgpu/gpgpu_core.c` 的 SIMT 执行引擎是手写的简化 RV32I/RV32F 解释器，缺失真实 GPGPU 的 cache、scheduler、divergence handling。Vortex（[vortex.cc](https://vortex.cc/)）是一个学术界广泛使用的开源 RISC-V GPGPU 项目，其 `simx` 是一个用 C++ 写的 cycle-level 模拟器。

**挑战目标**：把 Vortex 的 `simx` 作为 GPGPU 的后端直接嵌入到 QEMU 中，替换当前的简化 cmodel。

**建议路径**：

- 保留 `hw/gpgpu/gpgpu.c`（PCIe 前端、BAR、寄存器、DMA），只把 `gpgpu_core.c` 的执行后端替换为 `simx`。
- 解决 C 与 C++ 的 ABI 桥接：用 extern "C" 封装 `simx` 的 `CoreSim`，在 kernel dispatch 时喂入程序计数器、参数、VRAM 视图。
- 正确对接中断：kernel 执行完毕后通过 simx 回调触发 MSI-X。
- 适配 AI 软件栈：把进阶实验一设计的 libgpgpu/驱动与 Vortex 的 POCL / OpenCL 运行时对齐，至少跑通一个 Vortex 原生示例（如 `sgemm`）。
- 目标客户端 OS：ArceOS 或 rCore，验证内核驱动可以在两套真实 RISC-V OS 上工作。

**评估维度**：集成的完整度、simx 相对手写 cmodel 的指令覆盖增量（cache/barrier/divergence 是否生效）、以及在真实 AI workload 上的正确性。
