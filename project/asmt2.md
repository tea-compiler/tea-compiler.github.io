# 实验二：函数返回类型推断

## 1. 作业概述

本次实验的目标是**扩展 TeaLang 编译器的类型推断（Type Inference）阶段**，使 teac 能自动推断省略了返回类型标注的函数的返回类型。在现有的**局部变量前向推断**之上，本实验引入**类型变量**和**并查集（Union‑Find）约束求解**机制，实现**跨函数**的返回类型推断，包括线性调用链与自递归。

函数返回类型推断是一个实现性特性，没有纳入 TeaLang 主线语法，因此其通过一个独立模块（`src/experimental/return_infer.rs`，实现在 `experimental/` 目录里。该特性通过 Cargo feature `return-type-inference` 开关，**默认关闭**。关闭时编译器回退到"省略返回类型 = 返回 void"的主线语法；`cargo test` 在默认配置下只运行 **teac 主线** 回归测试，预期全部 PASS。本实验测试使用命令 `cargo test --features return-type-inference`，此时返回值类型推断会被加入 IR 生成管线，`type_infer_1..5` 也会被纳入测试。

### 1.1 测试用例

| 测试用例               | 内容                                                                           |
| -------------------- | ---------------------------------------------------------------------------- |
| **type_infer_basic** | 已有局部变量推断的回归测试（所有函数都有显式返回类型）                                                  |
| **type_infer_1**     | 线性跨函数依赖链：`abs`→`max`→`min`→`clamp`→`clamp_positive`，每个函数省略返回类型，多分支 return 合并 |
| **type_infer_2**     | 自递归：`pow(base, exp)` 在函数体内调用自身，调用点的返回类型仍是未解析的类型变量                            |
| **type_infer_3**     | 引用参数 `&[i32]` 与推断返回类型共存：`bsearch` 有三个不同控制流路径上的 return 语句                     |
| **type_infer_4**     | 三级跨函数链 `is_prime`→`next_prime`→`nth_prime`，推断结果用在 while 循环条件和循环体内            |
| **type_infer_5**     | **负测试**：函数体内同时存在 `return;`（void）和 `return t;`（i32），编译器必须报错拒绝                 |

共 6 个测试用例（5 个正测试 + 1 个负测试），每个对应 `tests/<name>/<name>.tea` 文件。

### 1.2 交付物

本次实验**仅需修改** `src/experimental/return_infer.rs`。

### 1.3 运行测试

返回类型推断默认关闭，所以下面的命令分成两类。

```bash
# 运行所有 teac 主线端到端测试（不包括 type_infer_1..5）
cargo test

# 只运行 teac 主线的推断测试
cargo test type_infer_basic

# 打开返回类型推断，运行完整测试
cargo test --features return-type-inference

# 只运行本实验的某一个测试
cargo test --features return-type-inference type_infer_1
```

**单文件调试**：

```bash
# 查看编译器对某个文件的 IR 输出（teac 主线）
cargo run -- tests/type_infer_1/type_infer_1.tea --emit ir

# 打开返回类型推断后再运行一次
cargo run --features return-type-inference -- tests/type_infer_1/type_infer_1.tea --emit ir
```

> **⚠️ 在实验尚未完成时运行 `cargo test --features return-type-inference` 会看到大量 FAILs**：不仅 `type_infer_1..5` 会 panic（符合预期），**其余 9 个**涉及返回 void 的测试（`bfs`、`brainfk`、`conv`、`dfs`、`dijkstra`、`expr_eval`、`hanoi`、`int_io`、`sort`）也会 FAIL。这是因为本实验调整了 TeaLang**函数签名省略返回类型**的语义，即 teac 主线省略返回类型代表返回 void，本实验的实验性特性要求省略返回类型时做类型推断，既可能是 void，也可能是 i32 等其他类型。

### 1.4 测试原理

**正测试**（`test_single`，对应 `type_infer_basic`、`type_infer_1` ~ `type_infer_4`）的检查逻辑：

1. **编译成功**：`teac --emit asm <file>` 退出码为 0；
2. **无错误输出**：stderr 为空；
3. **链接运行**：编译产出的汇编通过 `aarch64-linux-gnu-gcc` 链接为可执行文件；
4. **输出匹配**：在 QEMU 下运行（或 Apple Silicon 上原生运行），stdout 与 `<name>.out` 文件逐字节一致。

**负测试**（`test_compile_error`，对应 `type_infer_5`）的检查逻辑：

1. **编译失败**：`teac --emit ir <file>` 退出码非 0；
2. **有错误信息**：stderr 不为空（不要求特定的错误格式，只要有诊断输出）。

**注意：正测试是端到端**的，即从 `.tea` 源码经过 Parser、类型推断、IR 生成、后端代码生成，最终编译运行并检查输出。这意味着不能跳过类型检查，因为跳过检查会导致生成错误的 IR，从而让运行结果不匹配。

## 2. teac 类型推断概述

当前 teac 已经具备了局部变量类型推断能力。当程序员写 `let x;` 时，编译器可以根据后续的赋值语句自动确定 `x` 的类型。但函数签名中的返回类型仍然是必须显式标注的：

```rust
fn abs(x: i32) -> i32 {    // 必须写 -> i32
    if x < 0 {
        return 0 - x;
    }
    return x;
}
```

本次实验要求编译器能够接受省略返回类型的函数定义，并从函数体的 `return` 语句中推断出正确的返回类型：

```rust
fn abs(x: i32) {           // 省略 -> i32，teac 自动推断
    if x < 0 {
        return 0 - x;
    }
    return x;
}
```

### 2.1 已有的局部变量推断

当前 `type_infer.rs` 实现的是一个**前向流（forward-flow）类型推断**，逐条遍历函数体中的语句，维护一个类型环境 `TypeEnv`，将每个变量映射到 `Resolved(Dtype)` 或 `Pending` 状态。推断规则如下：

| #   | 形式                        | 效果                                        |
| --- | ------------------------- | ----------------------------------------- |
| R1  | `let x: T;`               | x → Resolved(T)                           |
| R2  | `let x: T = e;`           | check typeOf(e) = T; x → Resolved(T)      |
| R3  | `let x = e;`              | t = typeOf(e), t 必须是具体类型; x → Resolved(t) |
| R4  | `let x;`                  | x → Pending                               |
| R5  | `x = e;`（x 为 Pending）     | x → Resolved(typeOf(e))                   |
| R6  | `x = e;`（x 为 Resolved(T)） | check typeOf(e) = T                       |
| R7  | `if … else …`             | 各分支独立处理，然后合并                              |
| R8  | `while …`                 | 处理循环体一次，然后合并回去                            |

这套规则有一个根本假设：**typeOf(e) 总是能得到一个具体的 `Dtype`**。当 `e` 是一个函数调用时，推断器会从 `Registry` 中查找函数签名的 `return_dtype`，这就要求所有函数的返回类型在调用前就已经确定。

### 2.2 函数返回类型推断的挑战

当函数省略了返回类型标注时，编译器面临三个核心挑战：

**挑战一：类型在注册时未知。** 在 IR 生成的 Pass 2（签名注册阶段），遇到 `fn abs(x: i32) { ... }` 时，编译器不能立刻知道返回类型是 `i32` 还是 `void`。这里 `abs` 的返回值类型取决于函数体中的 `return` 语句，而函数体要到 Pass 3 才会被分析。

**挑战二：跨函数依赖。** 考虑这段代码：

```rust
fn abs(x: i32) {
    if x < 0 { return 0 - x; }
    return x;
}

fn clamp(x: i32, lo: i32, hi: i32) {
    return max(lo, min(hi, abs(x)));
}
```

`clamp` 的返回类型依赖于 `abs` 的返回类型。如果推断器在分析 `clamp` 时还不知道 `abs` 返回 `i32`，就无法确定 `clamp` 的返回类型。

**挑战三：自递归。** 考虑 `pow` 函数：

```rust
fn pow(base: i32, exp: i32) {
    if exp == 0 { return 1; }
    let half = pow(base, exp / 2);  // 调用自己
    return square(half);
}
```

这里存在一个循环依赖：`pow` 递归调用自己，所以其返回类型取决于其递归终点的返回值的类型。

### 2.3 解决方案：类型变量 + 约束求解

解决上述挑战的经典方案是引入**类型变量（type variable）**：

1. 为每个省略返回类型的函数分配一个**类型变量** $\alpha_f$。其本质上是一个占位符，代表"尚未确定的类型"
2. 在分析函数体时，从 `return` 语句中收集**约束**：
  - `return e;` 产生约束 $\alpha_f = \mathrm{typeOf}(e)$
  - `return;`（无值返回）产生约束 $\alpha_f = \mathrm{void}$
3. 用某种**约束求解**机制维护这些等价关系，当两个约束冲突时（如 $\alpha_f = \mathrm{void}$ 且 $\alpha_f = \mathrm{i32}$）立即报错
4. 分析结束后，如果 $\alpha_f$ 被解析为某个具体类型 T，则该函数的返回类型就是 T

类型变量解决了挑战一和挑战三：

- **类型在注册时未知**：注册时放入一个类型变量，分析函数体时逐步约束它。
- **自递归**：`pow` 调用自己时，返回的是类型变量 $\alpha_\text{pow}$。当同一函数的 `return 1;` 被处理时，约束 $\alpha_\text{pow} = \mathrm{i32}$ 被加入，从而 `let half = pow(...)` 自然得到类型 `i32`。

### 2.4 全局约束求解

自递归问题揭示了 teac 现有类型推导机制的一个缺陷：朴素的"边分析边解析"无法在遇到 `let half = pow(base, exp / 2);` 这一行时立即给出 `half` 的具体类型，因为 `pow` 本身的返回类型还没被解析。

本次实验要求实现的方案是：把"约束收集"与"求解"拆成两个阶段，即：

1. **约束收集阶段**：遍历所有函数体，为每个 `return` 语句、每个函数调用收集约束，放入一个**全局共享的约束求解器**中。此时允许任意类型变量保持未解析状态。
2. **求解阶段**：所有函数体分析完毕后统一求解。此时所有约束都已就绪，求解器拥有足够的信息把每一个类型变量归到某个具体类型上。

以自递归的 `pow` 为例：

```rust
fn pow(base: i32, exp: i32) {               // α_pow 待推断
    if exp == 0 { return 1; }               // 约束 A: α_pow = i32
    let half = pow(base, exp / 2);          // 约束 B: typeOf(half) = α_pow
    return square(half);                    // 约束 C: α_pow = typeOf(square(half))
}
```

分析过程中 $\alpha_\text{pow}$ 一直保持"未绑定"状态。同时，类型推导器不会立刻解析 B 的类型。类型推导器只会把 `half` 的类型登记为 $\alpha_\text{pow}$ 的等价类。当约束 A 贡献 $\alpha_\text{pow}$ = i32 时，整个等价类就确定为 `i32`，`half`、`square(half)` 的类型随之确定。

**这样的「延迟求解」的策略的优势还体现在 `type_infer_5`**：当一个函数体内既有 `return;` 又有 `return t;` 时，两条约束会先后进入同一个等价类，求解器的一次 bind 调用就能发现 void 与 i32 的冲突并报错，并且不需要分析顺序上的任何假设。

## 3. 约束求解与并查集

本节介绍实现全局约束求解所需的基础设施。从类型表示的扩展出发，逐步构建并查集数据结构和合一操作，并以一个完整的例子走完整个推断流程。

### 3.1 类型表示的扩展

当前推断器中，所有类型都是具体的 `Dtype`（如 `I32`、`Void`、`Array<I32, 5>` 等）。引入类型变量后，类型系统需要能够表示两种东西：

- **具体类型**：编译器已经知道的类型，如 `I32`
- **类型变量**：尚待确定的占位符，每个变量有一个唯一的整数 id，如 $\alpha_3$

一个自然的设计是把它们统一到一个带标签的联合类型中：

```rust
enum Ty {
    Concrete(Dtype),    // 已知类型
    Var(TypeId),        // 类型变量（TypeId 就是一个 usize）
}
```

从"可能已知"的角度来看，`Ty` 是 `Dtype` 的**超集**：任何 `Dtype` 都能包装成 `Ty::Concrete`，反向则需要求解。推断过程中所有需要表示类型的地方（局部变量的状态、表达式的求值结果、函数的返回类型），都需要改成 `Ty`。只有在"结果收集"阶段，推断器才会尝试把每个 `Ty` 解析回 `Dtype`。

**注意**：本实验中类型变量**只用于函数返回类型**，其他位置（如参数、结构体字段、数组元素）的类型总是显式标注的，永远不会是类型变量。这个约束简化了类型变量的生命周期管理。

**更重要的注意**：`Ty` 是 `return_infer.rs` 的**内部类型**。它不应当（也不会）泄漏到 `type_infer.rs`、`function_gen.rs` 或 `Registry` 里去，这是本作业架构的核心（见第 4 节）。

### 3.2 约束、等价类与不变式

推断过程中会不断产生形如"类型 A 等于类型 B"的**约束**。A 和 B 各自可能是具体类型或类型变量，组合起来有三种情况：

| 情况  | 左侧      | 右侧      | 处理方式                |
| --- | ------- | ------- | ------------------- |
| ①   | 具体类型 T₁ | 具体类型 T₂ | T₁ 必须等于 T₂，否则报类型错误  |
| ②   | 类型变量 α  | 具体类型 T  | 把 T 绑定到 α 所在的等价类上   |
| ③   | 类型变量 α  | 类型变量 β  | 把 α 和 β 所在的等价类合并为一个 |

情况 ③ 最微妙。两个类型变量被约束为相等，但都还不知道具体是什么。这在自递归中出现：分析 `pow` 时，`pow(base, exp / 2)` 的返回类型和 `pow` 自身的返回类型都是同一个变量 $\alpha_\text{pow}$，此时都没绑定到具体类型。

**核心不变式：每个等价类关联至多一个具体类型。** 换言之：

- 一个等价类初始为"空"（未绑定具体类型）
- 当某个约束要求该类必须是某个具体类型 T 时，T 被绑定到这个等价类上
- 之后任何新约束如果要求这个类是另一个具体类型 T' ≠ T，就是一个**类型冲突**

我们通过并查集维护这个不变式。

### 3.3 并查集（Union‑Find）

**并查集**是解决等价类维护问题的经典数据结构。它支持两种核心操作：

- **Find(x)**：返回 x 所在等价类的**代表元**（一般是这棵树的根）
- **Union(x, y)**：把 x 和 y 所在的两个等价类合并为一个

为了维护 3.2 节的不变式，本实验在经典并查集之上增加一个字段：**每个等价类的"根"携带一个可选的具体类型**（`Option<Dtype>`）。

这样整个数据结构大致如下：

```rust
struct UnionFind {
    parent: Vec<TypeId>,            // parent[i] 是 i 的父节点；parent[i] == i 时 i 是根
    rank: Vec<u32>,                 // 按秩合并用的秩
    concrete: Vec<Option<Dtype>>,   // concrete[root] 是 root 所代表等价类的具体类型
}
```

并查集上需要提供的操作包括：

| 操作            | 语义                                          |
| ------------- | ------------------------------------------- |
| `fresh()`     | 分配一个新的类型变量，返回它的 `TypeId`。初始时它自成一个等价类、无具体类型。 |
| `find(x)`     | 找到 x 所在等价类的根。                               |
| `union(x, y)` | 合并 x 和 y 所在的两个等价类。若两边都有具体类型且不一致，报错。         |
| `bind(x, T)`  | 把具体类型 T 绑定到 x 所在的等价类上。若该类已绑定到不同类型，报错。       |
| `resolve(x)`  | 查询 x 所在等价类的具体类型。若尚未绑定，返回 None。              |

此外，路径压缩（Find 时将沿途节点直接指向根，摊销查找时间接近 O(1)）和按秩合并（Union 时把矮树挂到高树下，保持树的平衡性）是并查集的常见优化手段，两者配合使得单次操作的摊销时间接近常数。

#### 工作示例

以下面的状态变化展示并查集如何维护等价类（左侧是输入约束，右侧是并查集的状态）：

初始状态 — 三个类型变量 α₀、α₁、α₂，各自独立：

```
等价类: {α₀}, {α₁}, {α₂}
concrete: [None, None, None]
```

约束 1：`union(α₀, α₁)` — 合并 α₀ 和 α₁：

```
等价类: {α₀, α₁}, {α₂}
concrete: α₀ 所在类 = None
```

约束 2：`bind(α₁, i32)` — 绑定 α₁ 到 i32。由于 α₁ 和 α₀ 在同一等价类，整个 {α₀, α₁} 都被锁定为 i32：

```
等价类: {α₀, α₁}, {α₂}
concrete: α₀ 所在类 = i32
```

约束 3：`union(α₂, α₀)` — 合并 α₂ 和 α₀。由于 α₀ 所在类已经是 i32，合并后 α₂ 也"继承"为 i32：

```
等价类: {α₀, α₁, α₂}
concrete: 唯一类 = i32
```

约束 4：`bind(α₂, void)` — 尝试把 void 绑定到 α₂。但该类已经是 i32 ≠ void：

```
→ 报 TypeMismatch 错误
```

### 3.4 Unification

在并查集之上，我们还需要实现一个 unify 操作。它接受两个 `Ty`，确保它们在当前约束下相等。

unify 操作只是对 3.2 节的三种情况做模式匹配：

```rust
fn unify(uf: &mut UnionFind, a: &Ty, b: &Ty, symbol: &str) -> Result<(), Error> {
    match (a, b) {
        // 情况 ①：两个具体类型 -> 必须相等
        (Ty::Concrete(x), Ty::Concrete(y)) => { /* 比较 x == y */ }
        // 情况 ②：类型变量 + 具体类型 -> 绑定
        (Ty::Var(v), Ty::Concrete(c)) | (Ty::Concrete(c), Ty::Var(v)) => {
            // 调用 uf.bind(*v, c.clone(), symbol)
        }
        // 情况 ③：两个类型变量 -> 合并等价类
        (Ty::Var(x), Ty::Var(y)) => {
            // 调用 uf.union(*x, *y, symbol)
        }
    }
}
```

**unify 是整个推断系统的核心**。在 `return_infer.rs` 内部，所有涉及类型相等的地方（比如变量首次定义时 RHS 与声明类型的对齐、赋值时的类型匹配、分支合并、return 语句的处理等）都通过 unify 来实现。

### 3.5 端到端案例：`abs` 的推断

以下面这段代码为例，完整追踪推断过程：

```rust
fn abs(x: i32) {
    if x < 0 {
        return 0 - x;
    }
    return x;
}

fn main() -> i32 {
    return abs(10);
}
```

**Pass 2（签名注册）**：把两个函数写入 Registry。`abs` 因为没写 `-> T`，按旧语义被**暂时**注册为 `-> void`。这个 Void 只是 Pass 2 的默认值，稍后会被覆盖。

**Pass 2.5（种子阶段）**：扫一遍所有 `FnDef`，为每个 `return_dtype = None` 的函数分配一个类型变量，记入 `pending_returns`。

```
pending_returns: { "abs" → α₀ }
UnionFind: 等价类: {α₀},  concrete: [None]
```

**Pass 2.5 — 收集阶段（abs）**：进入 `abs`，函数级状态 `return_var = Some(α₀)`。

- 处理 `return 0 - x;`：算术表达式类型是 `Ty::Concrete(i32)`
  - 调用 `unify(Ty::Var(α₀), Ty::Concrete(i32))`
  - 匹配情况 ②：`bind(α₀, i32)` → α₀ 所在类被锁定为 i32
- 处理 `return x;`：`x` 是参数，类型 `Ty::Concrete(i32)`
  - 再次 bind，但目标类型一致 → 无操作

并查集状态：`等价类: {α₀},  concrete: [Some(i32)]`。

**Pass 2.5 — 收集阶段（main）**：`return_var = None`（main 有显式返回类型）。

- 处理 `return abs(10);`：`abs(10)` 的类型 = `Ty::Var(α₀)`（abs 在 pending 集合）
  - main 的返回类型是显式的，这个 Pass 不再在此处强制对齐（该对齐由 Pass 3 forward‑flow 处理）。

**Pass 2.5 — 求解阶段**：

- `resolve(α₀)` → `Some(i32)` → abs 的返回类型是 i32
- 把 `Registry["abs"].return_dtype` 从占位值 `void` 覆写为 `i32`

自此 Pass 3 看到的 `Registry` 和"程序员一开始就写了 `fn abs(x: i32) -> i32`"的版本是**完全一致**的，forward‑flow `type_infer.rs` 照常运行，`FunctionGenerator` 照常生成 IR。

### 3.6 冲突检测示例：`modular_inverse`

再看 `type_infer_5` 中的错误代码：

```rust
fn modular_inverse(a: i32, m: i32) {
    let g = gcd(a, m);
    if g != 1 {
        return;         // 约束 A: α_ret = void
    }
    // ... 算法主体 ...
    return t;           // 约束 B: α_ret = i32
}
```

- 约束 A：`bind(α_ret, void)` → α_ret 所在类被锁定为 void
- 约束 B：`bind(α_ret, i32)` → 检查已有绑定 void ≠ i32 → **报 `TypeMismatch` 错误**

编译器应该拒绝这段代码并输出有意义的诊断。这正是 `type_infer_5` 测试验证的场景。

## 4. 整体架构

本节描述新 Pass 如何挂接到编译管线，以及数据如何在各个阶段之间流动。

### 4.1 从三个 Pass 到 3 + 1 个 Pass

原来的 `module_gen.rs::generate()` 有三遍扫描（use → 签名注册 → 函数体 IR）。本次实验**在 Pass 2 与 Pass 3 之间**插入一个独立的 **Pass 2.5**：

```rust
fn generate(program) -> Result<()> {
    // Pass 1: use 语句 → 注册外部符号
    for use_stmt in program.use_stmts {
        handle_use_stmt(use_stmt)?;
    }

    // Pass 2: 注册全局变量、函数签名、结构体
    //   省略返回类型的函数先按占位值 Void 注册，留待 Pass 2.5 覆写
    for elem in program.elements {
        match elem {
            VarDeclStmt(s) => handle_global_var_decl(s)?,
            FnDeclStmt(d)  => handle_fn_decl(d)?,
            FnDef(d)       => handle_fn_def(d)?,      // 仅登记签名
            StructDef(s)   => handle_struct_def(s)?,
        }
    }

    // Pass 2.5: 可插拔的 module-level passes
    //   feature `return-type-inference` 开启 → 列表包含 ReturnTypeInferPass:
    //     ① 给每个 pending FnDef 分配类型变量 α_f
    //     ② 遍历所有函数体收集约束
    //     ③ 求解 & 把具体 Dtype 回写 Registry
    //   feature 关闭              → 列表为空,本轮空转,行为与 teac 主线一致
    self.module_passes.run(self)?;

    // Pass 3: 对每个函数做类型检查并生成 IR(本实验未改动)
    //   此时 Registry 里已不存在占位值 / 类型变量
    for fn_def in program.fn_defs {
        let resolved_types = type_infer::infer_function(&registry, &globals, fn_def)?;
        let body = FunctionGenerator::new(&registry, &globals, resolved_types)
            .generate(fn_def)?;
        module.add_function(body);
    }
    Ok(())
}
```

上面的 `ReturnTypeInferPass` 实现在 `src/experimental/return_infer.rs`。`experimental/` 是未来所有 opt-in、feature-gated 扩展的统一家，和 `ir/` / `opt/` / `asm/` 并列在 `src/` 顶层；feature 关掉后整个目录对编译器不可见。

| Pass        | 职责                       | 改动 |
| ----------- | ------------------------- | ---- |
| **Pass 1**  | 处理 `use` 语句            | 无 |
| **Pass 2**  | 注册签名                   | 无，省略返回类型的函数依旧按 `Void` 占位值注册 |
| **Pass 2.5**| 返回类型推断                | **本次实验新增**；独立文件、feature‑gated |
| **Pass 3**  | 类型检查 + IR 生成          | 无，看到的 Registry 已经没有任何 α |

Pass 2.5 退出后，`Registry.function_types[*].return_dtype` 里不再有占位值（或者说待定值）。

### 4.2 Pass 2：签名注册

原本的 `handle_fn_decl` 和 `handle_fn_def` 用 `FunctionType::try_from(decl)` 一步到位构造签名。该函数里 `decl.return_dtype.as_ref().map_or(Dtype::Void, Dtype::from)` 保证了：**省略返回类型的函数签名会先被暂时注册为 `-> void`**。本实验对 Pass 2 无需任何修改；这个"占位 Void"就是后续 Pass 2.5 要覆写的对象。

Pass 2 结束时，假设源程序是：

```rust
fn abs(x: i32) { ... }                    // 省略返回类型
fn pow(base: i32, exp: i32) { ... }       // 省略返回类型
fn main() -> i32 { ... }                  // 显式 i32
```

Registry 的状态：

```
Registry.function_types:
    "abs"  → FunctionType { return_dtype: Void (占位), arguments: [x: i32] }
    "pow"  → FunctionType { return_dtype: Void (占位), arguments: [base: i32, exp: i32] }
    "main" → FunctionType { return_dtype: I32, arguments: [] }
```

### 4.3 Pass 2.5：返回类型推断

Pass 2.5 的核心逻辑在 `return_infer` 模块里，内部的入口函数是：

```rust
fn resolve_return_types(
    registry: &mut Registry,
    elements: &[ast::ProgramElement],
    globals: &IndexMap<Rc<str>, GlobalDef>,
) -> Result<(), Error>;
```

它被一个**零大小的插件包装器**暴露出来，即实现 [`src/common/pass.rs`](../src/common/pass.rs) 里定义的 `ModulePass` trait：

```rust
// src/experimental/return_infer.rs
pub(crate) struct ReturnInferPass;

impl ModulePass for ReturnInferPass {
    fn run(&self, gen: &mut IrGenerator<'_>) -> Result<(), Error> {
        resolve_return_types(&mut gen.registry, &gen.input.elements, &gen.module.global_list)
    }
}
```

调用端（`module_gen.rs::generate`）**不知道 `ReturnInferPass` 的存在**，它只是遍历注册进来的模块级 pass。

```rust
let passes = std::mem::take(&mut self.module_passes);
let result = passes.run(self);
self.module_passes = passes;
result?;
```

真正把 `ReturnInferPass` 注册进来的是 `src/ir.rs` 里的函数 `install_default_passes`：

```rust
// src/ir.rs
#[allow(unused_variables)]
pub fn install_default_passes(gen: &mut IrGenerator<'_>) {
    #[cfg(feature = "return-type-inference")]
    gen.add_module_pass(Box::new(ReturnInferPass));
}
```

`ReturnInferPass` 内部分三步。三步均在 `return_infer.rs` 内实现。

1. **种子（Seed）**：扫一遍 `elements`，对每个 `FnDef` 且 `fn_decl.return_dtype.is_none()` 的函数，用 `UnionFind::fresh()` 分配一个 α，放进 `pending_returns: HashMap<String, TypeId>`。
   - 早退路径：如果 `pending_returns` 为空，直接返回，零开销。

2. **收集（Collect）**：遍历所有 `FnDef`（pending 的、非 pending 的都要遍历，因为非 pending 函数体里可能也有对 pending 函数的调用，形成约束）。对每个函数体：
   - 维护一个**局部 `Ty` 环境**，参数先作为 `Ty::Concrete(...)` 种进去；
   - 逐条语句处理：`let`、赋值、`if/else`、`while`、`return`、函数调用；
   - 对函数调用：如果被调方在 `pending_returns` 中，返回 `Ty::Var(α_callee)`；否则从 Registry 取已知的 `Dtype` 包成 `Ty::Concrete`；
   - 对本函数（pending）的 `return e;`：emit `unify(α_self, typeOf(e))`；`return;` emit `unify(α_self, void)`；
   - 所有约束集成到一个 `UnionFind` 上，跨函数共享。

3. **求解（Resolve）**：对每个 `pending_returns[name] = α_f`：
   - `uf.resolve(α_f)` 得到 `Option<Dtype>`；
   - 若为 `None`（没有任何 `return` 约束过它），**回退为 `Void`**。这是为了向后兼容，没有 `return` 的函数在旧语义下就是 void 函数，如 `fn move(...) { putch(...); putch(...); }`。
   - 若结果不是 `Void`/`I32`，报错 `UnsupportedReturnType`；
   - 把结果写回 `registry.function_types[name].return_dtype`。

**内部数据结构** 包括：

```rust
type TypeId = usize;

enum Ty { Concrete(Dtype), Var(TypeId) }

struct UnionFind { /* parent / rank / concrete */ }

struct Collector<'a> {
    registry: &'a Registry,
    globals: &'a IndexMap<Rc<str>, GlobalDef>,
    pending: &'a HashMap<String, TypeId>,
    uf: &'a mut UnionFind,
    fn_name: &'a str,
    return_var: Option<TypeId>,       // Some(α_self) for pending；None 其它
    env: HashMap<String, Ty>,          // 局部变量的 Ty
}
```

### 4.4 Pass 3：不变

Pass 3 完全沿用原有实现。对每个 `FnDef`：

1. 调用 `type_infer::infer_function` 做 forward‑flow 局部变量推断；
2. 用得到的 `HashMap<String, Dtype>` 构造 `FunctionGenerator`；
3. 调用 `FunctionGenerator::generate(fn_def)` 生成 IR 语句；
4. 经过 `harvest_function_irs` 切成基本块，挂到 `module.function_list` 对应的 `Function` 上。

## 5. 测试用例详解

### 5.1 type_infer_basic（基线回归）

所有函数都有显式返回类型（`-> i32`），测试局部变量推断和基本的端到端流程。你的改动不应破坏这个测试。无论 feature 开或关，它都应该通过。

### 5.2 type_infer_1（线性跨函数链）

五个函数形成线性依赖链，每个函数都省略返回类型：

```
abs → max → min → clamp → clamp_positive
```

每个函数有 ≥2 个 return 语句（分布在 if/else 中），推断器需要在多分支间统一返回类型。`clamp` 的函数体是 `return max(lo, min(hi, x));`，嵌套了三层待推断函数的调用。

### 5.3 type_infer_2（自递归）

`pow(base, exp)` 使用快速幂算法，内部调用 `pow(base, exp / 2)`。调用自身时，`pow` 的返回类型仍然是一个类型变量。`return 1;` 和 `return square(half) * base;` 会将该类型变量约束为 `i32`，从而使自递归调用的返回类型也被确定。

### 5.4 type_infer_3（引用参数 + 推断返回类型）

`bsearch(arr: &[i32], n: i32, target: i32)` 有引用参数。签名注册时，参数的类型是确定的，但返回类型是类型变量。三个 return 语句（循环中找到目标、循环结束未找到）都返回 `i32` 值。

### 5.5 type_infer_4（三级跨函数链）

三个函数形成三级链：`is_prime` → `next_prime` → `nth_prime`。`is_prime` 有三个 return 点。推断结果被用在 while 循环的条件中（`while prime == 0`），这要求调用 `is_prime` 时推断就已经完成。

### 5.6 type_infer_5（负测试：冲突返回类型）

`modular_inverse` 函数的程序员犯了一个错误：一个分支写了 `return;`（void），另一个分支写了 `return t;`（i32）。这两个约束通过约束求解器传播后必然冲突，编译器必须拒绝这段代码并给出错误信息。

## 6. 实现方案简述

本节把第 3、4 节的算法对应到 `src/experimental/return_infer.rs` 中**具体的 10 处 `todo!`** 上，给出每一处的实现要点。

### 6.1 改动总览

按算法层次自底向上排列：

| #   | 函数 / 位置                              | 算法依据  | 一句话职责                                  |
| --- | ------------------------------------ | ----- | -------------------------------------- |
| 1   | `UnionFind::bind`                    | §3.3  | 把具体 `Dtype` 锁定到一个等价类                   |
| 2   | `UnionFind::union`                   | §3.3  | 合并两个等价类，按秩合并，处理冲突                      |
| 3   | `UnionFind::resolve`                 | §3.3  | 查询某变量所在等价类的具体类型                        |
| 4   | `unify`                              | §3.4  | 对 `(Ty, Ty)` 三种组合分派到 bind / union / 直接比较 |
| 5   | `Collector::process_var_def`         | §3.4  | `let x [: T] = e;` → 记 env，必要时 unify    |
| 6   | `Collector::merge_branches`          | §3.4  | if/else 两分支 env unify 回主 env           |
| 7   | `Collector::merge_with_body`         | §3.4  | while 体 env unify 回主 env               |
| 8   | `Collector::process_return`          | §4.3  | `return [e];` → unify(α_self, typeOf(e)) |
| 9   | `Collector::type_of_fn_call`         | §4.3  | 调用 pending 函数 → `Ty::Var(α_callee)`    |
| 10  | `resolve_return_types`（collect+resolve 阶段） | §4.3 | 串起 Phase 2/3，把结果回写 Registry           |

### 6.2 各 TODO 的实现要点

骨架在每个 `todo!` 上方的 doc-comment 里已经把步骤写得很细，这里只列每个 TODO 的一句话职责和最容易踩坑的地方，详情看代码注释。

- **TODO 1 `UnionFind::bind`**：先 `find(x)` 拿到根再读 `concrete`，`None` 写入、相同则忽略、不同则 `TypeMismatch`。
- **TODO 2 `UnionFind::union`**：合并两根的 `concrete` 槽（5 种组合，最后一种冲突报错），按秩挂载，把合并结果写到**新根**，旧根的 `concrete` 从此作废。
- **TODO 3 `UnionFind::resolve`**：`self.concrete[self.find(x)].clone()`。一定走 `find` 以触发路径压缩并取到正确的根。
- **TODO 4 `unify`**：三 arm 模式匹配，分别转调 `bind` / `union` / 直接 `Dtype` 比较。
- **TODO 5 `process_var_def`**：Scalar 有声明类型时 `unify(lhs, rhs)` 后写 env，无声明则直接把 `rhs` 写 env（可能是 `Ty::Var`）；Array 调 `check_array_initializer` 后写入具体 `Ty`。关键差异：声明类型与 RHS 不再"直接相等"而是 `unify`，让 RHS 是 α 的情况能反向锁定 α。
- **TODO 6 `merge_branches`**：仿 `type_infer.rs::merge_envs`，对 `self.env` 中每个 key 取两侧 `Ty`（缺则用 base）然后 `unify`，再回写 `self.env`。
- **TODO 7 `merge_with_body`**：仿 `type_infer.rs::merge_env_single`，对每个 base key `unify(base_ty, body_ty)` 即可，无需回写。
- **TODO 8 `process_return`**：`return e;` → `Ty::Concrete(...)`，`return;` → `Ty::Concrete(Void)`。`return_var` 为 `Some(α)` 时 `unify(Ty::Var(α), actual)`；为 `None`（已显式声明返回类型）时**什么都不做**，交给 Pass 3 的 forward-flow，避免重复诊断。
- **TODO 9 `type_of_fn_call`**：先递归 `type_of_right_val` 走每个实参（参数表达式里可能藏着另一个 pending 调用），再判断 callee 是否在 `pending`：是则返回 `Ty::Var(α_callee)`，否则查 Registry 包成 `Ty::Concrete`。
- **TODO 10 `resolve_return_types` 的 Phase 2/3**：Phase 2 遍历**所有** `FnDef`（含非 pending，因为它们的体内也可能调 pending）调 `collect_constraints`；Phase 3 对每个 `(name, α_f)` 做 `uf.resolve(α_f).unwrap_or(Dtype::Void)`，校验是 `Void`/`I32`，回写 `registry.function_types[name].return_dtype`。`None → Void` 这条 fallback 是 9 个 teac 主线测试在 feature 打开后仍然全绿的关键。

### 6.3 实现完成后的状态

把 10 处 `todo!` 全部替换为实现后：

- `cargo test`（feature off）：`type_infer_basic` 通过；其余主线测试与改动前完全一致。
- `cargo test --features return-type-inference`：所有 35 个测试全部通过。


## 7. 提交检查

- `cargo test` 通过
- `cargo test --features return-type-inference` 通过
