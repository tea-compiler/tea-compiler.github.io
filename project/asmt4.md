# 实验四：后端浮点支持

## 1. 作业概述

本次实验的目标是**扩展 teac 的 aarch64 后端**，打通浮点类型「源代码 → IR → 汇编 → 链接运行」的全流程。在实验 3 已经让 LLVM IR 携带浮点类型与浮点指令的基础上，本实验在后端实现对应的机器码生成路径，支持生成合法的 AArch64 汇编。

后端引入一条与既有整数路径兼容的浮点路径，涉及六个层面：

- 新增浮点寄存器类 `Fpr`（`s0`–`s31` / `d0`–`d31`），与既有的整数寄存器类 `Gpr` 并列；
- 新增浮点指令族 `fadd`、`fsub`、`fmul`、`fdiv`、`fcmp`、`scvtf`、`fcvtzs`、`fmov`，并在汇编打印阶段输出对应助记符；
- 按 AAPCS64 实现浮点参数传递（`s0`–`s7`）、浮点返回值（`s0`）的入口 / 出口 / 调用点 shim；
- 浮点可分配池取 caller-saved 的 `s18`–`s25`，把 `SaveCallerRegs` / `RestoreCallerRegs` 扩展到这一段，使跨 `bl` 活跃的浮点值由调用点保护（与整数 `x8`–`x15` 同构）；
- 在干扰图染色阶段维护两个独立的干扰图，分别为整数 vreg 与浮点 vreg 着色，二者着色到不相交的物理寄存器池；
- spill / reload 走对应字长的 `ldr s` / `str s`；
- phi 下降对 `f32` 操作数发出 `fmov`，对整数操作数发出 `mov`。

后端的数据类型（指令变体、寄存器类、寄存器宽度、AAPCS64 分类、栈帧的浮点 spill 尺寸）已在框架中实现完毕，本实验将实现浮点路径的具体逻辑：汇编打印器中浮点指令打印的五处 `todo!`、寄存器分配器重写阶段的五处 `todo!`、把浮点干扰图与整数干扰图分开染色的逻辑、AAPCS64 浮点参数 / 返回值的 shim，以及扩展到浮点池的 caller-save 包裹。

### 1.1 分数构成


| 部分        | 分值   | 说明                                                                                                      |
| --------- | ---- | ------------------------------------------------------------------------------------------------------- |
| **必做**    | 90 分 | 浮点类型 (`f32`) + 类型转换 (`as`) 的 aarch64 后端：FP 寄存器类、浮点指令族、AAPCS64 浮点 shim（参数 / 返回值 / 调用结果）、双干扰图染色、caller-saved FP 池 (`s18`–`s25`) 与扩展的 caller-save 包裹、FP spill/reload、`fmov` phi 下降 |
| **Bonus** | 10 分 | 开放任务：自行实现新语法特性的后端支持，或对 teac 做出有意义的改进（被合并进主分支）                                                           |


本实验后端新增的代码全部围绕浮点 `f32`，因为只有 `f32` 需要 FP 寄存器文件。`for` 循环、`impl` 方法、多维数组的 IR 位于整数 / 指针域（分支与 phi、修饰名函数与 `gep` / `call`、多级 `gep`），由既有整数后端直接处理，后端不需要为它们新增代码。

### 1.2 特性与测试用例


| 特性             | 对应测试用例                                                                | 涉及改动                                                                                                                 |
| -------------- | --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **浮点类型 (f32)** | `float_basic`, `float_arith`, `float_cmp`, `float_cast`, `float_func` | `types.rs`, `aapcs.rs`, `inst.rs`, `printer.rs`, `register_allocator.rs`, `function_generator.rs`, `phi_lowering.rs` |
| **类型转换 (as)**  | `float_cast`, `float_arith`, `float_func`                             | `printer.rs`（`scvtf`/`fcvtzs`）, `register_allocator.rs`, `function_generator.rs`                                     |
| **跨调用保护 / FP 溢出** | `float_func`                                                       | `printer.rs`（caller-save 包裹的 FP 半段）, `register_allocator.rs`（`s18`–`s25` 池 + FP 干扰图染色与 spill）                       |


共 5 个必做测试用例。每个对应 `tests/<name>/<name>.tea` 源程序与 `tests/<name>/<name>.out` 期望输出。需要让 `teac --emit asm <file>` 对每个浮点测试文件输出合法的 AArch64 汇编，链接运行后的输出与 `.out` 文件一致。

`float_`* 系列覆盖的浮点场景：


| 测试用例            | 内容                                                 |
| --------------- | -------------------------------------------------- |
| **float_basic** | 浮点变量声明、赋值、加减乘除                                     |
| **float_arith** | `&[f32]` 引用参数、`f32` 数组的下标读写、混合算术（3×3 矩阵乘法）         |
| **float_cmp**   | 浮点比较（`fcmp` + 条件分支），用于 if / while 条件               |
| **float_cast**  | `i32 as f32`（`scvtf`）与 `f32 as i32`（`fcvtzs`，向零截断） |
| **float_func**  | `f32` 参数（`s0`–`s7` 与栈传递）、`f32` 返回值（`s0`）、循环内浮点累加；一个调用结果跨另一次 `bl` 活跃（caller-save 包裹）；九个调用结果同时活跃超过 8 色 FP 池（强制 spill 到帧槽） |


### 1.3 交付物

修改以下文件（允许新增文件）：

- `src/asm/aarch64/types.rs` — 寄存器类 `RegisterClass`、寄存器宽度 `RegisterSize::S32`、`Dtype::F32 → RegisterSize::S32` 映射
- `src/asm/aarch64/aapcs.rs` — `Dtype::F32 → ArgumentClass::Float` 映射（NSRN 计数与 `s0`–`s7` 分配已就位）
- `src/asm/aarch64/printer.rs` — `fadd`/`fsub`/`fmul`/`fdiv`、`fcmp`、`scvtf`、`fcvtzs`、`fmov` 的汇编打印；`emit_save_caller_regs` / `emit_restore_caller_regs` 在 `current_fn_uses_fp` 为真时扩展到浮点池 `d18`–`d25`
- `src/asm/aarch64/register_allocator.rs` — 两个干扰图的拆分染色（整数着 `x8`–`x15`、浮点着 `s18`–`s25`），以及五条浮点指令的重写路径（spill / reload）
- `src/asm/aarch64/function_generator.rs` — 把 IR 的浮点语句下降为对应的 `Instruction`（与整数路径 `emit_biop` / `emit_cmp` 同构）；AAPCS64 浮点出口 / 调用点 shim：`emit_fpr_arg`、`return_inst` / `return_value_load` 的 `S32` 臂
- `src/asm/aarch64.rs` — `handle_arguments` 的 `ArgumentLocation::Fpr` 分支，把入参从 `s_n` 提取到 vreg
- `src/asm/aarch64/phi_lowering.rs` — phi 下降在 `f32` 操作数下选择 `fmov`

### 1.4 运行测试

```bash
# 运行全部 teac 主线端到端测试（不包括新增特性的测试）
cargo test

# 运行浮点测试（必做）：端到端汇编路径，teac 生成 .s，链接运行后比对 .out
cargo test --features float

# 运行某一个具体的浮点测试
cargo test --features float float_cast

# 只跑到 IR（clang 验证），用于先行确认 asmt-3 的 IR 正确
cargo test --features float,asmt-tests-ir

# 查看编译器对某个文件的汇编输出
cargo run -- tests/float_func/float_func.tea --emit asm -o float_func.s
```

不带 `asmt-tests-*` 的 `cargo test --features float` 走端到端汇编路径（`test_single`），由 teac 自身生成 `.s`，再用平台工具链链接运行；这条路径同时依赖 asmt-3 的 IR 浮点支持与本实验的后端浮点支持。`--features float,asmt-tests-ir` 收窄到 LLVM IR + `clang` 验证，可在动手写后端前确认 asmt-3 的 IR 输出正确。

## 2. AArch64 与 AAPCS64 简介

### 2.1 寄存器文件

AArch64 有两组架构上相互独立的寄存器文件，没有任何一条指令能同时从两组各取一个操作数：

**通用寄存器（GPR）**：31 个，记作 `x0`–`x30`（64 位）。每个 GPR 的低 32 位有独立名字 `w0`–`w30`，写 `w` 寄存器会把高 32 位清零。寄存器编号 31 在不同指令中或表示栈指针 `sp`，或表示零寄存器 `wzr` / `xzr`。AAPCS64 给若干 GPR 指定了固定角色：


| 寄存器           | 角色                                                 |
| ------------- | -------------------------------------------------- |
| `x0`–`x7`     | 整数 / 指针参数与整数返回值                                    |
| `x16` / `x17` | 过程内调用临时寄存器（IP0 / IP1），后端用作 `SCRATCH0` / `SCRATCH1` |
| `x29`         | 帧指针 `fp`                                           |
| `x30`         | 链接寄存器 `lr`（`bl` 写入的返回地址）                           |


**浮点 / SIMD 寄存器（FP）**：32 个 128 位寄存器 `v0`–`v31`。同一个 `v` 寄存器按访问宽度有不同视图：`b`（8 位）、`h`（16 位）、`s`（32 位单精度浮点）、`d`（64 位双精度浮点）、`q`（128 位）。`f32` 操作数使用 `s0`–`s31`。AAPCS64 给 FP 寄存器指定的角色：


| 寄存器                            | 角色                             |
| ------------------------------ | ------------------------------ |
| `s0`–`s7`（`v0`–`v7`）           | 浮点参数与浮点返回值                     |
| `v8`–`v15` 的低 64 位（`d8`–`d15`） | callee-saved：被调用方若使用须先保存、返回前恢复 |
| `v16`–`v31`                    | caller-saved：被调用方可自由覆写         |


GPR 编号与 FP 编号相互独立：`s16` 与 `x16` 是两个物理寄存器。后端正是利用这一点，让浮点临时寄存器 `s16` / `s17` 与整数临时寄存器 `x16` / `x17` 共用编号 16 / 17 而互不干扰。

### 2.2 指令集

本实验需要打印的浮点指令：

**浮点算术**（操作数与结果都在 FP 寄存器，宽度 `s`）

```asm
fadd s0, s1, s2     ; s0 = s1 + s2
fsub s0, s1, s2     ; s0 = s1 - s2
fmul s0, s1, s2     ; s0 = s1 * s2
fdiv s0, s1, s2     ; s0 = s1 / s2
```

**浮点比较**

```asm
fcmp s0, s1         ; 比较 s0 与 s1，结果写入 NZCV 标志位
```

`fcmp` 不写目标寄存器，结果隐含在 NZCV 标志中，由随后的条件分支 `b.<cond>` 读取。整数 `cmp` 也是同一套标志位机制，因此条件分支的打印与整数路径共用 `Cond` 与 `BCond`。

**整数 / 浮点转换**

```asm
scvtf s0, w1        ; s0 = (f32) (i32) w1     有符号整数 → 单精度浮点
fcvtzs w0, s1       ; w0 = (i32) (f32) s1     单精度浮点 → 有符号整数，向零截断
```

`scvtf` 的源在 GPR、目标在 FP；`fcvtzs` 的源在 FP、目标在 GPR。这两条指令是仅有的跨寄存器组的标量数据通路，对应 IR 的 `sitofp` / `fptosi`。

**浮点搬运**

```asm
fmov s0, s1         ; FP → FP，寄存器拷贝
fmov s0, w1         ; GPR → FP，按位重解释（bit-for-bit）
```

`fmov s_d, s_n` 在 FP 寄存器之间拷贝，用于 phi 下降、AAPCS64 浮点参数 / 返回值搬运。`fmov s_d, w_n` 把一个 32 位整数的位模式原样搬入 FP 寄存器，用于从一个整数立即数物化浮点常量的位模式。`fmov` 不做数值转换，与做数值转换的 `scvtf` 区分使用。

**浮点访存**（spill / reload）

```asm
str s0, [x29, #-4]  ; 把 s0 存入栈槽
ldr s0, [x29, #-4]  ; 从栈槽读回 s0
```

`ldr` / `str` 的助记符在整数与浮点之间相同，寄存器组由操作数写法（`s` 还是 `w` / `x`）区分。32 位浮点 spill 走 4 字节槽，与 `w` 寄存器同宽。

### 2.3 AAPCS64 调用约定

AAPCS64 §6.4.2 用两个独立计数器为每个实参选定传递位置：

- **NGRN**（Next General Register Number）：从 `x0` 起递增，为整数类参数（`i1`、`i32`、指针、数组引用）分配 `x0`–`x7`；
- **NSRN**（Next SIMD/FP Register Number）：从 `s0` 起递增，为浮点参数分配 `s0`–`s7`。

某一类计数器饱和后，该类的后续参数按源顺序溢出到栈上连续的 8 字节槽。整数与浮点计数器独立推进：一个有 9 个 `f32` 参数、2 个 `i32` 参数的函数，前 8 个 `f32` 占 `s0`–`s7`、第 9 个 `f32` 溢出到栈，而 2 个 `i32` 仍占 `x0`、`x1`。

返回值：整数 / 指针在 `x0`，浮点在 `s0`。

teac 后端在 `aapcs::classify_args` 处实现了上述分类逻辑。

### 2.4 栈帧布局

`FrameLayout` 是单个函数栈帧划分与所有 fp / sp 相对寻址的唯一来源。各区域自上而下：


| 区域                    | 地址范围                          |
| --------------------- | ----------------------------- |
| 入参溢出槽                 | `[fp, STACK_ARG_FP_BASE + n]` |
| 保存的 fp/lr             | `[fp, 0]` 与 `[fp, 8]`         |
| 局部帧（alloca + spill 槽） | `[fp, slot.offset_from_fp]`   |
| 出参槽                   | `[sp, off]`（仅在一次 `bl` 期间存活）   |


prologue 把 fp/lr 压入 `[fp, 0]` / `[fp, 8]`（占 `SAVED_REGS_BYTES = 16` 字节），令 `fp = sp`，再为局部帧预留 `frame_size` 字节。局部帧把 alloca 栈对象与 spill 槽混合排布，整体向下增长并向上对齐到 16 字节。

浮点 spill 槽的尺寸策略由 `spill_slot_layout` 给出：`S32` 与 `W32` 占 4 字节、4 字节对齐；`X64` 占 8 字节、8 字节对齐。寄存器分配器决定哪些 vreg 溢出后，调用 `FrameLayout::alloc_spill(size)` 按 vreg 宽度预留槽位，因此浮点 vreg 的 spill 槽为 4 字节。

### 2.5 teac 的后端流水线

`AArch64AsmGenerator::generate` 对每个有函数体的函数依次跑四个阶段：


| 阶段      | 职责                                               | 关键代码                          |
| ------- | ------------------------------------------------ | ----------------------------- |
| 入口 shim | 按 AAPCS64 把入参从 `x_`/`s_`/栈提取到 vreg               | `handle_arguments`            |
| 指令选择    | 遍历 IR 基本块，下降为带 vreg 的 `Instruction` 流，并完成 phi 下降 | `FunctionGenerator::generate` |
| 寄存器分配   | 活跃性分析 + 干扰图染色 + spill，重写为物理寄存器                   | `RegisterAllocator::run`      |
| 汇编打印    | 把 `Instruction` 流打印为汇编文本                         | `AsmPrinter::emit_inst`       |


浮点指令在指令选择阶段产生：`FunctionGenerator` 把 IR 的浮点语句下降为对应的 `Instruction` 变体（`FBiOpStmt → Instruction::FBinOp`、`FCmpStmt → Instruction::FCmp`、`SIToFPStmt → Instruction::Scvtf`、`FPToSIStmt → Instruction::Fcvtzs`、`FloatConst → Instruction::Fmov`），与整数路径的 `emit_biop` / `emit_cmp` 同构。这条下降、随后的寄存器分配与汇编打印，三个阶段的浮点路径都由本实验补齐。

AAPCS64 浮点参数 / 返回值的 shim 由本实验实现，三处都在浮点位置发出 `fmov`：`handle_arguments` 对 `ArgumentLocation::Fpr(n)` 发出 `Fmov{ dst: vreg, src: s_n }`；`emit_fpr_arg` 在调用点发出 `Fmov{ dst: s_n, src: ... }`；`return_inst` / `return_value_load` 对 `S32` 在 `s0` 与 vreg 间发 `Fmov`。skeleton 已就位的是分类机制（`classify_args` 的 NSRN 计数与 `s0`–`s7` 分配）；这些浮点路径只在 `Dtype::F32` 经 `RegisterSize::try_from` 映射为 `S32`、经 `ArgumentClass::try_from` 映射为 `Float` 后被激活。

### 2.6 指令表示

后端的指令定义在 `src/asm/aarch64/inst.rs` 的 `Instruction` 枚举。浮点相关变体：

```rust
enum Instruction {
    // ... 整数变体 ...
    FBinOp { op: FBinOp, dst: Register, lhs: Register, rhs: Register },  // fadd/fsub/fmul/fdiv
    FCmp   { lhs: Register, rhs: Register },                              // fcmp，结果在 NZCV
    Scvtf  { dst: Register, src: Register },                             // scvtf s_d, w_n
    Fcvtzs { dst: Register, src: Register },                             // fcvtzs w_d, s_n
    Fmov   { dst: Register, src: Operand },                              // fmov s_d, {s|w}_n / 立即数
}
```

`Register`、`RegisterSize`、`RegisterClass` 定义在 `src/asm/aarch64/types.rs`：

```rust
enum Register {
    Virtual(usize),     // 寄存器分配前的虚拟寄存器
    Physical(u8),       // 架构寄存器编号
    StackPointer,       // sp
}

enum RegisterSize {
    W32,                // w0–w30，也用于 i1
    X64,                // x0–x30，也用于指针
    S32,                // s0–s31，单精度浮点
}

enum RegisterClass {
    Gpr,                // 通用整数：x0–x30, sp
    Fpr,                // 浮点 / SIMD：s0–s31
}
```

`Register::Physical(n)` 自身不携带寄存器组信息：`Physical(16)` 配 `RegisterSize::X64` 是 `x16`，配 `RegisterSize::S32` 是 `s16`。寄存器组由伴随操作数的 `RegisterSize` 决定。`RegisterSize::class()` 把宽度映射回寄存器类（`W32`/`X64 → Gpr`，`S32 → Fpr`），寄存器分配器用它把 vreg 分桶。

`Instruction::defined_vreg_with_size` 给出每条指令定义的 vreg 及其宽度，是 spill 槽尺寸与寄存器组归属的来源：`FBinOp` / `Scvtf` / `Fmov` 的目标是 `S32`，`Fcvtzs` 的目标是 `W32`（整数），`Mov` / `BinOp` / `Ldr` 携带显式的 `size` 字段。

## 3. 浮点支持的实现 [必做]

### 3.1 浮点寄存器类 Fpr

AArch64 的整数寄存器组与浮点寄存器组架构独立，后端用 `RegisterClass` 区分一个 vreg 属于哪一组。`RegisterClass` 与 `RegisterSize::S32` 已在 `types.rs` 中定义；浮点路径被激活的前提是 `RegisterSize` 能从 `Dtype::F32` 得到 `S32`。

`RegisterSize::try_from(&ir::Dtype)` 当前对 `Dtype` 的变体做穷尽匹配：

```rust
impl TryFrom<&ir::Dtype> for RegisterSize {
    type Error = Error;
    fn try_from(dtype: &ir::Dtype) -> Result<Self, Self::Error> {
        match dtype {
            ir::Dtype::I1 | ir::Dtype::I32 => Ok(RegisterSize::W32),
            ir::Dtype::Pointer { .. } => Ok(RegisterSize::X64),
            ir::Dtype::Void | ir::Dtype::Struct { .. } | ir::Dtype::Array { .. } => {
                Err(Error::UnsupportedDtype { dtype: dtype.clone() })
            }
        }
    }
}
```

asmt-3 给 `ir::Dtype` 添加 `F32` 变体后，这个 `match` 不再穷尽，编译失败。需要补上一条分支，把 `Dtype::F32` 映射为 `RegisterSize::S32`：

```rust
ir::Dtype::F32 => Ok(RegisterSize::S32),
```

这条映射一旦补上，整条浮点 shim（入参 / 返回值 / 调用结果）随之激活：`handle_arguments`、`emit_fpr_arg`、`return_inst`、`return_value_load`、`emit_call_result` 都通过 `RegisterSize::try_from` 决定走整数还是浮点路径。

### 3.2 浮点指令族与汇编打印

汇编打印器 `src/asm/aarch64/printer.rs` 已经为每条浮点指令预留了打印方法与调度入口，方法体是 `todo!`。`AsmPrint::emit_inst` 的浮点分支已接好线：

```rust
Instruction::FBinOp { op, dst, lhs, rhs } => self.emit_fbinop(*op, *dst, *lhs, *rhs)?,
Instruction::FCmp { lhs, rhs }            => self.emit_fcmp(*lhs, *rhs)?,
Instruction::Scvtf { dst, src }           => self.emit_scvtf(*dst, *src)?,
Instruction::Fcvtzs { dst, src }          => self.emit_fcvtzs(*dst, *src)?,
Instruction::Fmov { dst, src }            => self.emit_fmov(*dst, *src)?,
```

`reg_name(r, RegisterSize::S32)` 已能把 `Physical(n)` 打印为 `s{n}`，因此五处打印只需按助记符拼装字符串：

- `**emit_fbinop**`：把 `FBinOp` 的四个变体映射到 `fadd` / `fsub` / `fmul` / `fdiv`，三个操作数都用 `S32`（`s` 寄存器），形如 `fadd s_d, s_n, s_m`。
- `**emit_fcmp**`：打印 `fcmp s_n, s_m`，两个操作数用 `S32`，无目标寄存器。
- `**emit_scvtf**`：打印 `scvtf s_d, w_n`，目标用 `S32`、源用 `W32`（源是整数）。
- `**emit_fcvtzs**`：打印 `fcvtzs w_d, s_n`，目标用 `W32`（目标是整数）、源用 `S32`。
- `**emit_fmov**`：按源操作数分派。源是 `Operand::Register`（FP 寄存器）时打印 `fmov s_d, s_n`；源是 `Operand::Immediate(bits)`（浮点常量的 IEEE-754 单精度位模式）时，先把位模式用 `emit_mov_imm` 物化到一个整数临时寄存器，再打印 `fmov s_d, w_scratch`（GPR → FP 按位重解释）。

浮点比较的条件分支沿用整数路径：IR 的浮点比较谓词在指令选择阶段已映射为 `Cond`（`Eq`/`Ne`/`Lt`/`Le`/`Gt`/`Ge`），并记入 `cond_map`；`emit_fcmp` 只负责打印 `fcmp`，随后的 `b.<cond>` 由既有 `BCond` 分支打印，读取 `fcmp` 写下的 NZCV。在不含 NaN 的测试输入下，`fcmp` + `b.lt` / `b.gt` / `b.eq` 等给出与有序比较一致的结果。

### 3.3 AAPCS64 浮点调用约定

`aapcs::classify_args` 的 NSRN 计数、`s0`–`s7` 分配、栈溢出逻辑均已就位。分类的入口是 `ArgumentClass::try_from`，它当前对 `Dtype` 做穷尽匹配，且 `Float` 变体带 `#[allow(dead_code)]`：

```rust
enum ArgumentClass { Int, #[allow(dead_code)] Float }

impl TryFrom<&ir::Dtype> for ArgumentClass {
    type Error = Error;
    fn try_from(dtype: &ir::Dtype) -> Result<Self, Self::Error> {
        match dtype {
            ir::Dtype::I1 | ir::Dtype::I32
            | ir::Dtype::Pointer { .. } | ir::Dtype::Array { .. } => Ok(Self::Int),
            ir::Dtype::Void | ir::Dtype::Struct { .. } => Err(Error::UnsupportedDtype {
                dtype: dtype.clone(),
            }),
        }
    }
}
```

补上 `Dtype::F32 → Float` 分支后，浮点参数走 NSRN、占用 `s0`–`s7`；`Float` 变体被构造后 `#[allow(dead_code)]` 可移除：

```rust
ir::Dtype::F32 => Ok(Self::Float),
```

入口、出口与调用点 shim 由本实验实现，三处都在浮点位置发出 `fmov`，与整数路径（`Mov` 走 `x_`）同构：

- **入口**（`handle_arguments`，`src/asm/aarch64.rs`）：`ArgumentLocation::Fpr(n)` 发出 `Fmov{ dst: vreg, src: Physical(n) }`，把入参从 `s_n` 提取到 vreg。
- **出口**（`return_inst` / `return_value_load`，`function_generator.rs`）：返回宽度为 `S32` 时，`return_inst` 发出 `Fmov{ dst: s0, src: ... }` 把返回值安置到 `s0`；`return_value_load` 对调用结果发出 `Fmov{ dst: vreg, src: s0 }`（经 `emit_call_result`）。
- **调用点**（`emit_fpr_arg`，`function_generator.rs`）：发出 `Fmov{ dst: s_n, src: ... }` 把出参安置到 `s_n`。

整数臂已实现，浮点臂复制其结构、把 `Mov` 换成 `Fmov`、目标 / 源用 `s_`。`RegisterSize::try_from(&Dtype::F32) = S32` 是浮点臂被选中的前提。

**caller-saved 浮点池与调用点包裹。** 整数可分配池 `x8`–`x15` 是 caller-saved，调用点用 `SaveCallerRegs` / `RestoreCallerRegs` 在 `bl` 两侧把它们压栈、弹栈：

```rust
// printer.rs：emit_save_caller_regs 在 bl 前压入 x8–x15
str x15, [sp, #-16]!
stp x13, x14, [sp, #-16]!
stp x11, x12, [sp, #-16]!
stp x9,  x10, [sp, #-16]!
str x8,  [sp, #-16]!
```

浮点可分配池同样取 caller-saved 段 `s18`–`s25`（位于 `v16`–`v31`，与 FP scratch `s16` / `s17`、参数 / 返回值 `s0`–`s7` 都不相交）。被调用方可自由覆写这一段，因此跨 `bl` 活跃的浮点值由调用点保存——把 caller-save 包裹扩展到浮点池：

```rust
// printer.rs：emit_save_caller_regs 压完 x8–x15 后，current_fn_uses_fp 为真时压入 d18–d25
stp d24, d25, [sp, #-16]!
stp d22, d23, [sp, #-16]!
stp d20, d21, [sp, #-16]!
stp d18, d19, [sp, #-16]!
```

以 `d` 寄存器成对压栈，使 `sp` 保持 16 字节对齐（`s` 对只移动 8 字节会破坏对齐），低 32 位即承载 `f32`；`emit_restore_caller_regs` 以 `ldp d` 逆序恢复。FP 半段按 `current_fn_uses_fp` 门控：不使用 FP 的函数其调用序列里没有任何 FP 存取。这样后端永不使用 callee-saved 的 `d8`–`d15`，无需在序言 / 尾声保存恢复它们。`register_allocator.rs` 中的 `ALLOCATABLE_FPRS` 与 `F_SCRATCH0` / `F_SCRATCH1` 据此选定：

```rust
const ALLOCATABLE_FPRS: [u8; NUM_COLORS] = [18, 19, 20, 21, 22, 23, 24, 25];
const F_SCRATCH0: u8 = SCRATCH0;   // s16，与整数 x16 同编号、不同寄存器组
const F_SCRATCH1: u8 = SCRATCH1;   // s17
```

### 3.4 双干扰图寄存器分配

#### 3.4.1 图着色寄存器分配回顾

teac 通过基于图着色的寄存器分配算法把无限多的 vreg 映射到有限的物理寄存器。`register_allocator.rs` 的流程：

1. **活跃性分析**：以指令流为节点构 CFG，用 `BackwardLiveness` 在 `Bitset` 格上做后向数据流，得到每条指令的 `live_out`（出口处活跃的 vreg 集合）。
2. **构造干扰图**：若 vreg `a` 与 `b` 在某点同时活跃，则二者不能着同一物理寄存器，连一条干扰边。`InterferenceGraph::build` 对每条指令，让其定义的 vreg 与该点 `live_out` 中的其它 vreg 互相连边。
3. **化简（simplify）**：反复移除度数 `< NUM_COLORS` 的节点压入栈；无低度节点时，按度数最大挑一个标记为潜在 spill 并移除（乐观着色）。
4. **选择（select）**：逆序弹栈，给每个节点分配一个不与已着色邻居冲突的颜色；无可用颜色则真正 spill。
5. **spill**：为 spill 的 vreg 在栈帧预留槽位，重写阶段在每次使用前 reload、定义后 store。

物理寄存器池 `ALLOCATABLE_REGS = [8, 9, ..., 15]`（`x8`–`x15`），`NUM_COLORS = 8`。

#### 3.4.2 为什么要两个图

整数 vreg 只能着色到 GPR，浮点 vreg 只能着色到 FP 寄存器。把两类 vreg 放进同一个图、用同一个池染色会出错：一个浮点 vreg 若被着色成 `x9`，打印时无法表达，因为没有指令能把 FP 值放进 GPR。

同时，跨组的两个 vreg 即便在同一点活跃，也不构成干扰：一个活跃的整数 vreg 占 `x_`、一个活跃的浮点 vreg 占 `s_`，二者用不同物理寄存器，可以「同色」（同编号）而不冲突。因此整数与浮点应当在各自的图里、各自的池上独立染色。

#### 3.4.3 按寄存器类拆分

`build_gen_kill` 已经在收集 `gen` / `kill` 集合的同时记录每个 vreg 的 `RegisterSize`（`vreg_sizes`），`RegisterSize::class()` 据此把 vreg 判定为 `Gpr` 或 `Fpr`。染色阶段据此拆分：

- 整数 vreg（`Gpr`）在只含整数 vreg 的干扰子图上染色，池为 `ALLOCATABLE_REGS`（`x8`–`x15`）；
- 浮点 vreg（`Fpr`）在只含浮点 vreg 的干扰子图上染色，池为 `ALLOCATABLE_FPRS`（`s18`–`s25`）。

两张子图各自跑 simplify / select / spill。一种实现方式是按寄存器类过滤 `present` 位集，对每一类分别构造干扰图并染色，再合并两份 `Location` 映射；由于跨组 vreg 不连边，也可在同一邻接表上染色而对两类分别选池。最终每个 vreg 得到一个 `Location`：着色到物理寄存器，或 spill 到栈槽。spill 的浮点 vreg 经 `vreg_sizes` 得到 `S32`，`alloc_spill(S32)` 预留 4 字节槽，因此浮点 spill 槽为 4 字节。

#### 3.4.4 重写浮点指令

`InstRewriter::rewrite_inst` 把带 vreg 的指令重写为物理寄存器形式，五条浮点指令的分支当前是 `todo!`：

```rust
Instruction::FBinOp { .. } => todo!("asmt-4: rewrite Inst::FBinOp ..."),
Instruction::FCmp   { .. } => todo!("asmt-4: rewrite Inst::FCmp ..."),
Instruction::Scvtf  { .. } => todo!("asmt-4: rewrite Inst::Scvtf — dst is Fpr, src is Gpr"),
Instruction::Fcvtzs { .. } => todo!("asmt-4: rewrite Inst::Fcvtzs — dst is Gpr, src is Fpr"),
Instruction::Fmov   { .. } => todo!("asmt-4: rewrite Inst::Fmov ..."),
```

整数指令的重写已经给出可复用的模板：`load_src_reg` 把一个源 vreg 解析为物理寄存器（若 spill 则先 reload 到 scratch），`write_to_dst` 把目标 vreg 解析为物理寄存器（若 spill 则写到 scratch、再 store 回槽）。浮点重写沿用同一套，但 scratch 与 spill 宽度按寄存器组选取：

- `**FBinOp**`：三个操作数都在 FP 组。lhs / rhs 用 `load_src_reg(..., S32, F_SCRATCH0/F_SCRATCH1)` reload 到 `s16` / `s17`；dst 用 `write_to_dst(..., S32, F_SCRATCH0)`，spill 时 `str s` 到 4 字节槽。
- `**FCmp**`：两个 FP 源 reload 到 `s16` / `s17`，无目标。
- `**Scvtf**`：dst 在 FP 组（reload/spill 用 `S32` + `F_SCRATCH`），src 在 GPR 组（reload 用 `W32` + 整数 `SCRATCH`）。
- `**Fcvtzs**`：dst 在 GPR 组（`W32` + 整数 `SCRATCH`），src 在 FP 组（`S32` + `F_SCRATCH`）。
- `**Fmov**`：dst 在 FP 组。源是寄存器时按 FP 处理（`S32` + `F_SCRATCH`）；源是立即数时透传给打印器（由 `emit_fmov` 物化）。

`emit_spill_load` / `emit_spill_store` 已经接受 `RegisterSize` 参数并打印 `Ldr` / `Str`，对 `S32` 会让 `reg_name` 打印 `s` 寄存器，因此浮点 spill / reload 不需要新增指令，只需在重写时传 `S32` 与 FP scratch 编号。

### 3.5 浮点 spill / reload

浮点 vreg 溢出时，重写阶段在每次使用前用 `ldr s` 从栈槽读回 FP scratch，在定义后用 `str s` 写回。栈槽尺寸由 `spill_slot_layout(S32) = (4, 4)` 给出，与 `w` 寄存器同宽。

FP scratch 取 `s16` / `s17`（`F_SCRATCH0` / `F_SCRATCH1`），它们位于 caller-saved 的 `v16`–`v31` 区间，调用点不需要保护。它们与整数 scratch `x16` / `x17` 共用编号 16 / 17，但因寄存器组独立而互不影响：一条同时涉及整数源与浮点源的指令（`scvtf` / `fcvtzs`）可以让整数源用 `w16`、浮点目标用 `s16` 而不冲突。

### 3.6 phi 下降与 fmov

teac 的 IR 是 SSA 形式，控制流合流点的 phi 节点在汇编生成前被销毁：`phi_lowering::plan` 把每个 phi 拆成沿控制流边的并行拷贝，临界边上插入新的拆分块，再由 `FunctionGenerator::emit_parallel_copies` 把并行拷贝序列化为单条搬运。

整数 phi 拷贝下降为 `mov w_d, w_n`；浮点 phi 拷贝须下降为 `fmov s_d, s_n`，因为 `mov` 在 FP 寄存器之间不可用。拷贝指令的选择依据操作数的 `Dtype`：`Dtype::F32` 选 `Instruction::Fmov`，整数选 `Instruction::Mov`。`emit_copy` 现按 `RegisterSize::try_from(dst.dtype())` 取宽度并发出 `Mov`，需要在 `dtype` 为 `F32`（宽度 `S32`）时改发 `Fmov`。并行拷贝里打破环用的临时 vreg 也继承源操作数的 `Dtype`，因此环里的浮点拷贝同样走 `fmov`。

`float_func` 的 `compute` 在 while 循环里累加 `result = result + y`，mem2reg 把 `result` 提升为循环头的 phi 节点，循环回边上的拷贝即一条 `fmov`。

## 4. 端到端示例

以 `float_func` 中的 `fadd` 为例：

```rust
fn fadd(a:f32, b:f32) -> f32 {
    return a + b;
}
```

asmt-3 生成的 IR（mem2reg 后，参数直接被使用）：

```llvm
define float @fadd(float %r0, float %r1) {
fadd:
    %r2 = fadd float %r0, %r1
    ret float %r2
}
```

后端下降并染色后的汇编（Linux 符号；macOS 下符号带 `_` 前缀，浮点 vreg 着色到 `s18`–`s25`，具体编号取决于分配）：

```asm
.text
.globl fadd
.p2align 2
fadd:
	stp x29, x30, [sp, #-16]!
	mov x29, sp
	fmov s18, s0         ; 入口 shim：a (s0) → vreg
	fmov s19, s1         ; 入口 shim：b (s1) → vreg
	fadd s18, s18, s19   ; a + b
	fmov s0, s18         ; 返回值 → s0
	mov sp, x29
	ldp x29, x30, [sp], #16
	ret
```

关键点：

- 浮点参数 `a` / `b` 经入口 shim 从 `s0` / `s1` 搬入浮点 vreg；
- 加法用 `fadd`，三个操作数都是 `s` 寄存器；
- 返回值经 `fmov s0, s_n` 安置到 `s0`；
- 浮点 vreg 着色到 caller-saved 的 `s18`–`s25`；`fadd` 是叶函数，不调用其它函数，因此调用点的 caller-save 包裹不出现（一旦函数体内有 `bl` 且使用 FP，包裹的 FP 半段才发射）。

类型转换 `as` 的例子（`float_func` 中 `let si:i32 = s as i32;`）：

```rust
let s:f32 = fadd(a, b);
let si:i32 = s as i32;
```

```asm
	bl fadd
	fmov s18, s0         ; 调用结果 (s0) → vreg
	fcvtzs w9, s18       ; f32 → i32，向零截断
```

反向转换 `i32 as f32` 下降为 `scvtf s_d, w_n`。

## 5. 实现方案简述

### 5.1 改动总览

按数据流自前向后排列：


| #   | 位置                                   | 算法依据 | 职责                                                 |
| --- | ------------------------------------ | ---- | -------------------------------------------------- |
| 1   | `types.rs`：`RegisterSize::try_from`  | §3.1 | 补 `Dtype::F32 → S32`，浮点路径取 `S32` 宽度               |
| 2   | `aapcs.rs`：`ArgumentClass::try_from` | §3.3 | 补 `Dtype::F32 → Float`，浮点参数走 NSRN                  |
| 3   | `function_generator.rs`：浮点语句下降       | §2.5 | IR 浮点语句 → `Instruction` 浮点变体，记 `cond_map`          |
| 4   | `printer.rs`：`emit_fbinop`           | §3.2 | 打印 `fadd`/`fsub`/`fmul`/`fdiv s_d, s_n, s_m`       |
| 5   | `printer.rs`：`emit_fcmp`             | §3.2 | 打印 `fcmp s_n, s_m`                                 |
| 6   | `printer.rs`：`emit_scvtf`            | §3.2 | 打印 `scvtf s_d, w_n`                                |
| 7   | `printer.rs`：`emit_fcvtzs`           | §3.2 | 打印 `fcvtzs w_d, s_n`                               |
| 8   | `printer.rs`：`emit_fmov`             | §3.2 | 寄存器源打印 `fmov s_d, s_n`，立即数源经整数 scratch 物化后打印 `fmov s_d, w_n` |
| 9   | `register_allocator.rs`：双图染色         | §3.4 | 按 `RegisterClass` 拆分，整数着 `x8`–`x15`、浮点着 `s18`–`s25` |
| 10  | `register_allocator.rs`：五条浮点重写       | §3.4 | reload/spill 走 FP scratch 与 `S32` 宽度               |
| 11  | `phi_lowering.rs` / `emit_copy`      | §3.6 | `f32` 拷贝选 `fmov`                                   |
| 12  | `aarch64.rs` / `function_generator.rs`：浮点 shim | §3.3 | `handle_arguments` 的 `Fpr` 臂、`emit_fpr_arg`、`return_inst` / `return_value_load` 的 `S32` 臂发 `fmov` |
| 13  | `printer.rs`：caller-save 包裹的 FP 半段   | §3.3 | `current_fn_uses_fp` 为真时，在 `bl` 两侧 `stp/ldp d18`–`d25` |


### 5.2 各处实现要点

- **改动 1（`RegisterSize::try_from`）**：单行，`ir::Dtype::F32 => Ok(RegisterSize::S32)`。asmt-3 添加 `Dtype::F32` 后，未补此分支会让 `match` 非穷尽而编译失败，穷尽性约束确保新标量类型在处理前无法通过编译。`S32` 是浮点 shim 与重写选中浮点臂的前提。
- **改动 2（`ArgumentClass::try_from`）**：`ir::Dtype::F32 => Ok(Self::Float)`，并移除 `Float` 变体的 `#[allow(dead_code)]`。
- **改动 3（浮点语句下降）**：仿照 `emit_biop` / `emit_cmp` 写浮点版本。浮点二元运算两操作数都需在 FP 寄存器里，常量操作数先经 `Fmov` 物化；浮点比较把谓词记入 `cond_map` 后发 `Instruction::FCmp`，与整数比较一致由后续 `CJump` 消费。
- **改动 4–8（打印）**：`reg_name(r, S32)` 已能打印 `s` 寄存器，五处只是字符串拼装。`emit_fmov` 是唯一需要分派的：寄存器源打印 `fmov s_d, s_n`，立即数源经整数 scratch 物化位模式后打印 `fmov s_d, w_scratch`。
- **改动 9（双图染色）**：用 `vreg_sizes[v].class()` 把 vreg 分桶；整数桶在 `ALLOCATABLE_REGS` 上染色，浮点桶在 `ALLOCATABLE_FPRS` 上染色。跨组 vreg 不连边，可分别构图或在同一邻接表上分别选池。
- **改动 10（浮点重写）**：复用 `load_src_reg` / `write_to_dst`，对 FP 操作数传 `RegisterSize::S32` 与 `F_SCRATCH0` / `F_SCRATCH1`；`Scvtf` / `Fcvtzs` 跨组，整数侧传 `W32` 与整数 `SCRATCH`、浮点侧传 `S32` 与 FP scratch。
- **改动 11（phi 的 fmov）**：`emit_copy` 在 `dst.dtype()` 为 `Dtype::F32` 时发 `Instruction::Fmov`，整数操作数发 `Instruction::Mov`；打破环的临时 vreg 继承源 `Dtype`，自动走对应搬运。
- **改动 12（浮点 shim）**：三处都把整数臂（`Mov` 走 `x_`）复制成浮点臂（`Fmov` 走 `s_`）。入口 `handle_arguments` 对 `ArgumentLocation::Fpr(n)` 发 `Fmov{ dst: vreg, src: Physical(n) }`；调用点 `emit_fpr_arg` 发 `Fmov{ dst: Physical(n), src: ... }`；返回 `return_inst` / `return_value_load` 的 `S32` 臂在 `s0` 与 vreg 间发 `Fmov`。
- **改动 13（FP caller-save 包裹）**：`emit_save_caller_regs` 在压完 `x8`–`x15` 后，于 `current_fn_uses_fp` 为真时以 `stp d` 成对压入 `d18`–`d25`；`emit_restore_caller_regs` 以 `ldp d` 逆序恢复。`d` 对保持 `sp` 16 字节对齐，门控使纯整数函数的调用序列不变。`AArch64AsmGenerator` 计算每个函数的 `uses_fp` 并在发射函数体前 `set_uses_fp`。

### 5.3 实现完成后的状态

- `cargo test`（不带 feature）：主线 30 个端到端测试仍全部通过；纯整数函数不发射 FP caller-save 半段（按 `uses_fp` 门控）。
- `cargo test --features float`：`float_basic`、`float_arith`、`float_cmp`、`float_cast`、`float_func` 五个端到端测试全部通过。
- `cargo build` 无 `todo!` 触发的 panic，`RegisterSize::try_from` / `ArgumentClass::try_from` 穷尽。

## 6. 提交检查

- 必做：`float_`* 系列五个测试全部通过（`cargo test --features float`）
- 原有 30 个主线端到端测试仍然通过（`cargo test`）
- 代码能编译（`cargo build` 无错误、无警告）
- 使用 `cargo run -- tests/float_func/float_func.tea --emit asm` 能产生可读且可链接运行的汇编
- `float_func` 覆盖跨调用保护与 FP 溢出：缺少扩展的 FP caller-save 包裹时其输出会偏离 `.out`；其 `sum9` 调用使九个 `f32` 结果同时活跃，迫使 FP vreg 溢出到帧槽（`[x29, #-N]` 上出现 `s` 寄存器的 `stur` / `ldur`）

