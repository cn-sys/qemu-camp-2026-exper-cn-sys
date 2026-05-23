# QEMU 训练营 2026 — Agent 指南

本仓库是 QEMU v10.2.50，从上游分支并针对 **QEMU 训练营 2026** 专业阶段定制。所有实验基于 **RISC-V** 架构。

## 快速开始（精确命令）

```bash
make -f Makefile.camp configure   # riscv64-softmmu + riscv64-linux-user + Rust
make -f Makefile.camp build        # meson/ninja 构建（JOBS=N 控制并行数）
```

每个实验有独立的测试目标：

```bash
make -f Makefile.camp test-cpu     # TCG 裸机测试题
make -f Makefile.camp test-soc     # QTest SoC 测试题
make -f Makefile.camp test-gpgpu   # QTest GPGPU（QOS 框架）
make -f Makefile.camp test-rust    # Rust 单元测试 + QTest
make -f Makefile.camp test         # 全部四个
```

测试目标内部的逐项命令：
- CPU: `make -C build check-gevico-tcg` — 编译裸机 RISC-V ELF，在 QEMU user/softmmu 下运行
- SoC: `meson test --no-rebuild --print-errorlogs "qtest-riscv64/<test-name>"`（在 `build/` 下执行）
- GPGPU: `build/tests/qtest/qos-test -p /riscv64/virt/generic-pcihost/pci-bus-generic/pci-bus/gpgpu/gpgpu-tests/<subtest> --tap -k`
- Rust 单元测试: `meson test --no-rebuild --test-args "<test_fn>" rust-i2c-unit`（在 `build/` 下执行）
- Rust QTest: 同 SoC 的 `meson test` 命令，测试名为 `qtest-riscv64/test-spi-rust-*`

## 评分体系

| 方向       | 测试及位置                                    | 评分方式               |
|------------|-----------------------------------------------|------------------------|
| **CPU**    | 10 道 TCG 裸机题 (`tests/gevico/tcg/`)       | 通过数 × 10            |
| **SoC**    | 10 道 QTest (`tests/gevico/qtest/`)           | 通过数 × 10            |
| **GPGPU**  | 17 个 QOS 子测试 (`tests/qtest/gpgpu-test.c`) | floor(通过数 × 100 / 17) |
| **Rust**   | 3 单元 + 7 QTest (`rust/hw/i2c/` + qtest)    | 通过数 × 10            |

成绩文件写入 `build/<方向>-result.log`（仅包含整数通过数）。

## 关键目录

| 路径 | 用途 |
|------|------|
| `hw/riscv/g233.c` | G233 SoC 板（训练营专用，约 2000 行） |
| `hw/gpgpu/` | GPGPU PCI 设备模型 |
| `include/hw/riscv/g233.h` | G233 寄存器映射、IRQ 分配 |
| `tests/gevico/tcg/riscv64/` | CPU 实验裸机测试（`.insn r` 自定义指令） |
| `tests/gevico/tcg/riscv64/crt/` | 裸机 CRT（crt.S、PL011 串口控制台、内存操作） |
| `tests/gevico/qtest/` | SoC/GPGPU/Rust 的 QTest 测试 |
| `tests/qtest/gpgpu-test.c` | GPGPU 子测试（约 1000 行，17 个子测试） |
| `rust/hw/i2c/src/` | Rust I2C 总线模型（含学生 TODO 标记） |
| `rust/tests/tests/vmstate_tests.rs` | Rust 集成测试 |
| `Makefile.camp` | 训练营构建封装（configure/build/test） |

## 实验详情

### CPU（TCG 测试题）

测试编译为裸机 RISC-V 二进制，通过 `riscv64-unknown-elf-gcc` 链接。使用 `.insn r` 伪指令编码自定义 CPU 指令。测试结构：

```c
// 模式：内联自定义指令，比较硬件结果与软件参考值
static inline void custom_op(int *dst, const int *a, const int *b) {
    asm volatile(".insn r 0x7b, 6, 30, %0, %1, %2" : : "r"(dst), "r"(a), "r"(b) : "memory");
}
```

CRT 位于 `crt/` 目录：`crt.S`（启动代码）、`console.c`（PL011 UART 输出）、`memory.c`（memcpy 等）、`crt.h`（assert/crt_assert 宏）。

**实现位置**: `target/riscv/` — 添加 custom-3（opcode 0x7b）指令的 decode 和 TCG 翻译实现。编码约束：funct6=6，funct3 取值 6(DMA)/14(GEMM)/22(SORT)/30(VADD)/38(CRUSH)/54(EXPAND)/70(VDOT)/86(VRELU)/102(VSCALE)/118(VMAX)。需在 `insn32.decode`（或新建 decode 文件）中定义指令格式，在 `insn_trans/` 下新增翻译处理函数。

### SoC（QTest，G233 板）

测试使用 QEMU 的 QTest 框架。初始化模式：

```c
QTestState *qts = qtest_init("-machine g233 -m 2G");
// 通过 qtest_readl/qtest_writel 进行 MMIO 读写
qtest_quit(qts);
```

G233 内存映射常量定义在 `include/hw/riscv/g233.h`（UART 位于 0x10000000，PLIC 位于 0x0C000000，CLINT 位于 0x02000000，DRAM 位于 0x80000000）。

**实现位置**: `hw/riscv/g233.c` — 添加 GPIO（0x10012000）、PWM、SPI（0x10018000）、WDT、Flash 控制器等设备模型到 G233 板初始化函数 `virt_machine_init()` 中，创建 sysbus 设备并映射 MMIO、连接 IRQ。设备模型可选上游已有的 `sifive_gpio`、`sifive_spi`、`sifive_pwm`，或自行实现。

### GPGPU（QTest 通过 QOS）

单个 PCI 设备：vendor 0x1234，device 0x1337。通过 QOS 图形框架测试，提供 17 个可单独选择的子测试。寄存器宏定义在 `tests/qtest/gpgpu-test.c`（BAR0 偏移量：DEV_ID、VRAM、GLOBAL_CTRL、IRQ、DMA、SIMT 等）。

**实现位置**:
- `hw/gpgpu/gpgpu.c` — TODO 标记位置：MMIO 控制寄存器读写（第 25-42 行）、VRAM 读写（第 54-71 行）、DMA 完成处理（第 110 行）、Kernel 完成处理（第 116 行）
- `hw/gpgpu/gpgpu_core.c` — TODO 标记位置：warp 初始化（第 15-29 行）、warp 执行/RV32I+RV32F 解释器（第 31-38 行）、kernel 分发执行（第 40-44 行）

### Rust（I2C 总线）

纯 Rust 实现的 I2C 总线模型，位于 `rust/hw/i2c/src/`：
- `lib.rs`：`I2CEvent` 枚举、`I2CSlave` trait、`I2CBus` 结构体（含 TODO 方法）
- 单元测试通过 `meson test --test-args "<test_fn>" rust-i2c-unit` 运行
- QTest 测试使用 G233 SoC 上的 SPI+I2C 设备

**实现位置**: `rust/hw/i2c/src/lib.rs` — 完成 `I2CBus` 结构体中 6 个 TODO 标记方法：`attach()`（第 91 行）、`device_count()`（第 96 行）、`start_transfer()`（第 114 行）、`end_transfer()`（第 124 行）、`send()`（第 132 行）、`recv()`（第 140 行）

## 开发环境准备

### 所需工具
- Ubuntu 24.04
- RISC-V 裸机工具链：`riscv64-unknown-elf-gcc`（从 riscv-collab 发布版下载）
- Rust 工具链：`stable` 版本，还需 `cargo install bindgen-cli`
- QEMU 构建依赖：`apt-get build-dep qemu`

### RISC-V 工具链
```bash
# 解压到 /opt/riscv，将 /opt/riscv/bin 加入 PATH
export PATH="/opt/riscv/bin:$PATH"
```

## 代码与工作流约定

### C 代码风格
- **Stroustrup** 大括号风格（`.dir-locals.el`），**4 空格缩进，不使用 Tab**
- EditorConfig 强制 LF 换行符、UTF-8 编码、文件末尾换行
- Vim 配置：`set expandtab shiftwidth=4 smarttab`（`.exrc`）
- 遵循 QEMU 的 `CODING_STYLE` 指南（该文件位于 docs/ 目录下）
- 提交前运行 `scripts/checkpatch.pl` 检查

### Rust 代码风格
- Edition 2021
- `cargo fmt` 通过 `rustfmt.toml`：import 粒度为 "Crate"，StdExternalCrate 分组
- Clippy 配置在 `clippy.toml`（MSRV 1.83.0）

### 构建约定
- **所有构建和测试在 `build/` 目录内执行** — 仅支持 out-of-tree 构建
- `Makefile.camp` 封装了 QEMU 的 meson 构建系统，始终使用它（切勿直接运行 `configure` 或 `meson`）
- QEMU 自带的 meson 位于 `build/pyvenv/bin/meson`
- Rust 依赖通过 meson subprojects 方式存放在 `subprojects/` 下

### Git 约定
- 提交信息使用 conventional commits 格式（feat/fix/chore 前缀），可参考提交历史
- `scripts/checkpatch.pl` 强制要求 Signed-off-by 和代码风格
- CI 在推送到 `main` 分支时运行（GitHub Classroom）
- CI 仅当 `README.md` 和 `docs/` 之外的文件发生变更时触发

### meson 测试套件
- `qtest-riscv64/` — riscv64-softmmu 的 QTest 测试
- `rust-i2c-unit` — I2C 总线的 Rust 单元测试
- `rust-integration` — Rust vmstate 集成测试
- `unit` / `rust` — Rust 测试的 suite 标签
- `quick` / `slow` / `thorough` — 测试速度等级（默认: quick）

### 常见陷阱
- **G233 QTest 初始化必须使用 `-machine g233 -m 2G`** — 其他机器没有训练营专用设备
- **必须使用 QEMU 自带的 meson**（位于 `build/pyvenv/bin/meson`），不要用系统 meson（版本不匹配）
- **Rust bindgen** 需要通过 cargo 额外安装 `bindgen-cli`
- **TCG 测试题**在 configure 时就需要 RISC-V 交叉编译器在 PATH 中（会被编译进配置）
- **`make -f Makefile.camp rebuild` = `make -f Makefile.camp clean build`**（不是 distclean）
- **`make -f Makefile.camp distclean`** 会删除整个 `build/` 目录
- **不要直接编辑 `build/` 中的构建产物**；编辑源码后重新构建即可
- **GPGPU 测试直接运行 `qos-test` 二进制**（不通过 meson），且必须设置 `QTEST_QEMU_BINARY`
- **Rust 测试使用 `--test-threads 1`** 以避免竞态条件
