# 实验二: 类型推断

## 1. 作业概述

本次实验的目标是**扩展 TeaLang 编译器的类型推断（Type Inference）阶段**，使其能够自动推断省略了返回类型标注的函数的返回类型。在已有的局部变量前向推断基础上，本实验引入**类型变量**和**约束求解**机制，实现跨函数的返回类型推断。

### 1.1 特性与测试用例


| 测试用例                 | 内容                                                                           |
| -------------------- | ---------------------------------------------------------------------------- |
| **type_infer_basic** | 已有局部变量推断的回归测试（所有函数都有显式返回类型）                                                  |
| **type_infer_1**     | 线性跨函数依赖链：`abs`→`max`→`min`→`clamp`→`clamp_positive`，每个函数省略返回类型，多分支 return 合并 |
| **type_infer_2**     | 自递归：`pow(base, exp)` 在函数体内调用自身，调用点的返回类型仍是未解析的类型变量                            |
| **type_infer_3**     | 引用参数 `&[i32]` 与推断返回类型共存：`bsearch` 有三个不同控制流路径上的 return 语句                     |
| **type_infer_4**     | 三级跨函数链 `is_prime`→`next_prime`→`nth_prime`，推断结果用在 while 循环条件和循环体内            |
| **type_infer_5**     | **负测试**：函数体内同时存在 `return;`（void）和 `return t;`（i32），编译器必须报错拒绝                 |


共 6 个测试用例（5 个正测试 + 1 个负测试），每个对应 `tests/<name>/<name>.tea` 文件。

### 1.2 交付物与框架结构

本次实验**需要修改** `src/ir` 下与类型推断有关的内容，以通过上述测试用例。

### 1.3 运行测试

```bash
# 运行全部测试（包括已有的端到端测试 + 新增的类型推断测试）
cargo test

# 只运行某个特定测试
cargo test type_infer_1

# 运行所有类型推断测试
cargo test type_infer_

# 查看编译器对某个文件的 IR 输出
cargo run -- tests/type_infer_1/type_infer_1.tea --emit ir

# 编译并运行（需要 aarch64 交叉编译环境 + qemu）
cargo run -- tests/type_infer_1/type_infer_1.tea -o /tmp/test.s
```

### 1.4 测试原理

**正测试**（`test_single`，对应 `type_infer_basic`、`type_infer_1` ~ `type_infer_4`）的检查逻辑：

1. **编译成功**：`teac --emit ir <file>` 退出码为 0；
2. **无错误输出**：stderr 为空；
3. **链接运行**：编译产出的汇编通过 `aarch64-linux-gnu-gcc` 链接为可执行文件；
4. **输出匹配**：在 QEMU 下运行，stdout 与 `<name>.out` 文件逐字节一致。

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

自递归问题揭示了 teac 现有类型推导机制的一个缺陷：朴素的"边分析边解析"无法在遇到 `let half = pow(base, exp / 2);` 这一行时立即给出 `half` 的具体类型，因为 `pow` 本身的返回类型还没被解析。因此，此时 teac 的类型推导系统会直接报错。

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

本节介绍实现全局约束求解所需的基础设施。从类型表示的扩展出发，逐步构建并查集数据结构和统一操作，并以一个完整的例子走完整个推断流程。

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

### 3.3 并查集（Union-Find）

**并查集**是解决等价类维护问题的经典数据结构。它支持两种核心操作：

- **Find(x)**：返回 x 所在等价类的**代表元**（一般是这棵树的根）
- **Union(x, y)**：把 x 和 y 所在的两个等价类合并为一个

为了维护 3.2 节的不变式，本实验在经典并查集之上增加一个字段：**每个等价类的"根"携带一个可选的具体类型**（`Option<Dtype>`）。

这样整个数据结构大致如下：

```rust
struct UnionFind {
    parent: Vec<TypeId>,            // parent[i] 是 i 的父节点；parent[i] == i 时 i 是根
    rank: Vec<usize>,               // 按秩合并用的秩
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
fn unify(&mut self, a: &Ty, b: &Ty, symbol: &str) -> Result<(), Error> {
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

** unify 是整个推断系统的核心。** 原来代码中所有进行类型兼容性检查的地方（比如 `check_compatible`、赋值时的类型匹配、分支合并、参数类型检查等），在引入类型变量后都需要替换为 unify 操作。

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

**Pass 2（签名注册）**：为 `abs` 分配返回类型变量 α₀，把占位签名写入 `Registry`。

```
pending_function_returns: { "abs" → α₀ }
Registry["abs"] = (args=[x: i32], return=Void)  ← Void 只是占位
并查集: fresh(α₀) →  等价类: {α₀}, concrete: [None]
```

**Pass 3a（约束收集 — abs）**：

- 进入 `abs`，`return_ty` = `Ty::Var(α₀)`（因为 abs 在 pending 集合中）
- 处理 `return 0 - x;`：`typeOf(0 - x) = Ty::Concrete(i32)`
  - 调用 `unify(Ty::Var(α₀), Ty::Concrete(i32))`
  - 匹配情况 ②：`bind(α₀, i32)` → α₀ 所在类被锁定为 i32
- 处理 `return x;`：`typeOf(x) = Ty::Concrete(i32)`（x 是参数）
  - 调用 `unify(Ty::Var(α₀), Ty::Concrete(i32))`
  - 再次 bind，但目标类型一致 → 无操作

并查集状态：`等价类: {α₀}, concrete: [Some(i32)]`。

**Pass 3a（约束收集 — main）**：

- `return_ty` = `Ty::Concrete(i32)`（main 有显式返回类型，不在 pending）
- 处理 `return abs(10);`：
  - `typeOf(abs(10))`：abs 在 pending 集合 → 返回 `Ty::Var(α₀)`
  - 调用 `unify(Ty::Concrete(i32), Ty::Var(α₀))`
  - 匹配情况 ②：`bind(α₀, i32)` → 目标类型一致，无操作

**Pass 3b（求解）**：

- `resolve(α₀)` → `Some(i32)` → abs 的返回类型是 i32
- 把 `Registry["abs"].return_dtype` 从 Void 改写为 i32

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

本节描述框架在模块层面的整体结构，以及数据如何在各个阶段之间流动。

### 4.1 从三个 Pass 到四个 Pass

原来的 `module_gen.rs` 中 `IrGenerator::generate()` 有三遍扫描。引入全局约束求解后，原来的 Pass 3 被拆分为**约束收集 → 求解 → IR 生成**三个子阶段。四个 Pass 及其数据流：

```
         ┌──────────┐
源程序 ──►│  Pass 1  │ use 语句 → 注册外部符号
         └────┬─────┘
              ▼
         ┌──────────┐
         │  Pass 2  │ 注册全局变量、函数签名、结构体
         └────┬─────┘
              │   Registry（部分函数的 return 是占位值）
              │   pending_function_returns: { fn_name → α_id }
              │   shared UnionFind（已分配对应的类型变量槽位）
              ▼
         ┌──────────┐
         │ Pass 3a  │ 逐函数收集约束，所有约束写入全局求解器
         │ 约束收集  │ 每个函数保存 PartialInference（含 env + return_ty）
         └────┬─────┘
              │   shared UnionFind（约束全部到位）
              │   partials: [(fn_def, PartialInference)]
              ▼
         ┌──────────┐
         │ Pass 3b  │ 统一求解每个 PartialInference → InferenceResult
         │ 求解回写  │ 把推断出的返回类型写回 Registry
         └────┬─────┘
              │   Registry（所有函数签名完整）
              │   finalised: [(fn_def, InferenceResult)]
              ▼
         ┌──────────┐
         │ Pass 3c  │ 逐函数运行 FunctionGenerator 生成 IR
         │  IR 生成 │
         └──────────┘
```


| Pass        | 职责          | 变更                                    |
| ----------- | ----------- | ------------------------------------- |
| **Pass 1**  | 处理 `use` 语句 | 无                                     |
| **Pass 2**  | 注册签名        | 省略返回类型的函数：参数正常注册，返回类型写占位值，分配类型变量      |
| **Pass 3a** | 约束收集        | 新增。遍历所有函数体，把约束写入全局求解器，保存每个函数的"部分推断结果" |
| **Pass 3b** | 求解 + 回写     | 新增。统一求解所有类型变量，把推断出的返回类型写回 Registry    |
| **Pass 3c** | IR 生成        | 原 Pass 3 的下半部分，但现在所有函数签名都已完整          |


**关键不变式**：**约束求解器的生命周期覆盖整个 Pass 3**（从 3a 开始到 3b 结束）。它不随单个函数的处理结束而销毁，即所有函数的约束都往同一个求解器中累积，直到 3b 才统一求解。

### 4.2 Pass 2：签名注册

原来的 `handle_fn_decl` 和 `handle_fn_def` 用 `FunctionType::try_from(decl)` 一步到位构造签名。`try_from` 要求返回类型是已知的，对省略返回类型的函数不适用。框架把流程拆开：**参数总是显式的，照常处理；返回类型可能缺失，分情况处理**。

处理规则：

- **显式返回类型**（`fn abs(x: i32) -> i32 { ... }`）：行为不变。正常构造 `FunctionType { return_dtype: i32, arguments: ... }`，写入 Registry。
- **省略返回类型**（`fn abs(x: i32) { ... }`）：
  1. 在 Registry 中注册一个**占位签名**：`FunctionType { return_dtype: Void, arguments: ... }`（Void 只是占位，后续会被回写）
  2. 分配一个类型变量 id（用 `next_type_var` 计数器）
  3. 在 `pending_function_returns` 中记录 `函数名 → 类型变量 id`

**声明与定义的一致性**：如果同一个函数既有 `fn foo() -> i32;`（声明）又有 `fn foo() { ... }`（定义），框架保证两者参数列表完全一致，返回类型按"两者都给时必须一致；一方给另一方没给就继承"的规则处理。

**Pass 2 结束时的状态快照**。假设源程序是：

```rust
fn abs(x: i32) { ... }          // 省略返回类型
fn pow(base: i32, exp: i32) { ... }  // 省略返回类型
fn main() -> i32 { ... }        // 显式 i32
```

Pass 2 结束后，各数据结构的状态：

```
Registry.function_types:
    "abs"  → FunctionType { return_dtype: Void (占位), arguments: [x: i32] }
    "pow"  → FunctionType { return_dtype: Void (占位), arguments: [base: i32, exp: i32] }
    "main" → FunctionType { return_dtype: I32, arguments: [] }

pending_function_returns:
    "abs" → 0
    "pow" → 1

UnionFind:
    等价类: {α₀}, {α₁}       （两个孤立的变量，对应 abs 和 pow 的返回类型）
    concrete: [None, None]
```

### 4.3 Pass 3a：约束收集

框架遍历所有 `fn_def`，对每个函数调用 `collect_constraints`：

1. 推理引擎把所有约束写入**全局**约束求解器
2. 引擎返回一个 `PartialInference`，包含：
  - 函数名
  - 局部类型环境（变量 → `Ty`，类型可能含类型变量）
  - 函数的返回类型表示（也是 `Ty`）
3. 框架把 `PartialInference` 存到一个 `Vec` 中，等 3b 使用

**这一阶段不解析任何类型变量。** 例如，分析 main 时遇到 `return abs(10);`，此时 α₀ 还没有任何绑定。但没关系，我们只需生成约束 `unify(main 的 return_ty, α₀)` 并继续。

### 4.4 Pass 3b：统一求解与回写

框架遍历所有 `PartialInference`，对每个调用 `finalize_partial`：

1. 解析返回类型：把 `return_ty` 中的类型变量解析为具体 `Dtype`
2. 解析局部环境：把 `env` 中每个变量的 `Ty` 解析为 `Dtype`
3. 验证约束：
  - 任何无法解析的类型变量 → `TypeNotDetermined` 错误
  - 推断出的返回类型必须是 `Void` 或 `I32`（后端限制）→ 否则报 `UnsupportedReturnType`

框架拿到返回的 `InferenceResult` 后，把 `return_dtype` 写回 Registry，并从 `pending_function_returns` 中移除对应条目。

### 4.5 Pass 3c：IR 生成

此时所有函数签名都已完整。框架对每个函数：

1. 用 Pass 3b 产生的 `InferenceResult.resolved_locals` 初始化 `FunctionGenerator`
2. 运行 `FunctionGenerator::generate(fn_def)` 生成 IR 语句
3. 收集为基本块，挂到 `module.function_list` 中对应的 `Function` 上

**这一步的逻辑与原来的 Pass 3 IR 生成部分几乎完全一致**，只是类型解析已经在 3b 做完了，这里拿到的都是具体 `Dtype`。

### 4.6 推断引擎的 API 接口

需要实现的两个入口函数的签名如下（已在 `type_infer.rs` 中给出）：

```rust
// Pass 3a 调用：收集约束，返回部分推断结果
pub(crate) fn collect_constraints(
    registry: &Registry,
    pending_returns: &HashMap<String, TypeId>,
    globals: &IndexMap<Rc<str>, GlobalDef>,
    fn_def: &ast::FnDef,
    uf: &mut UnionFind,                 // 关键：可变引用，约束写入这里
) -> Result<PartialInference, Error>;

// Pass 3b 调用：解析部分推断结果为最终结果
pub(crate) fn finalize_partial(
    partial: PartialInference,
    uf: &mut UnionFind,
) -> Result<InferenceResult, Error>;
```

两个辅助结构也已提供：

```rust
pub(crate) struct PartialInference {
    // 外壳已给出，字段需自行填充，取决于推断器内部的组织方式
}

pub(crate) struct InferenceResult {
    pub(crate) resolved_locals: HashMap<String, Dtype>,   // 已解析为具体类型
    pub(crate) return_dtype: Dtype,
}
```

**为什么把 UnionFind 作为参数而非让 `collect_constraints` 自行创建？** 因为它是**全局共享**的，即所有函数的约束都进同一个求解器。如果在 `collect_constraints` 内部 `let mut uf = UnionFind::default()`，每个函数就有独立的求解器，跨函数的等价关系就会丢失。框架在 Pass 3 开始前创建 UnionFind，通过参数传入；这样在 3b 求解阶段，某个函数的返回类型变量既可能被该函数自身的 return 语句约束（自递归时），也可能被调用它的函数约束。

**UnionFind 的存放位置**：可以放在 `IrGenerator` 结构体上，也可以在 Pass 3 开始时创建、通过参数传递。两种设计都可行，放在 `IrGenerator` 上更直接，放在 Pass 3 局部更能体现"Pass 3 专用"的语义。

## 5. 测试用例详解

### 5.1 type_infer_basic（基线回归）

从原 `type_infer` 重命名而来。所有函数都有显式返回类型（`-> i32`），测试局部变量推断和基本的端到端流程。你的改动不应破坏这个测试。

### 5.2 type_infer_1（线性跨函数链）

五个函数形成线性依赖链，每个函数都省略返回类型：

```
abs → max → min → clamp → clamp_positive
```

每个函数有 ≥2 个 return 语句（分布在 if/else 中），推断器需要在多分支间统一返回类型。`clamp` 的函数体是 `return max(lo, min(hi, x));`——嵌套了三层待推断函数的调用。

### 5.3 type_infer_2（自递归）

`pow(base, exp)` 使用快速幂算法，内部调用 `pow(base, exp / 2)`。调用自身时，`pow` 的返回类型仍然是一个类型变量。`return 1;` 和 `return square(half) * base;` 会将该类型变量约束为 `i32`，从而使自递归调用的返回类型也被确定。

### 5.4 type_infer_3（引用参数 + 推断返回类型）

`bsearch(arr: &[i32], n: i32, target: i32)` 有引用参数。签名注册时，参数的类型是确定的，但返回类型是类型变量。三个 return 语句（循环中找到目标、循环结束未找到）都返回 `i32` 值。

### 5.5 type_infer_4（三级跨函数链）

三个函数形成三级链：`is_prime` → `next_prime` → `nth_prime`。`is_prime` 有三个 return 点。推断结果被用在 while 循环的条件中（`while prime == 0`），这要求调用 `is_prime` 时推断就已经完成。

### 5.6 type_infer_5（负测试：冲突返回类型）

`modular_inverse` 函数的程序员犯了一个错误：一个分支写了 `return;`（void），另一个分支写了 `return t;`（i32）。这两个约束通过约束求解器传播后必然冲突，编译器必须拒绝这段代码并给出错误信息。

## 7. 提交检查

- `type_infer_basic` 通过（`cargo test type_infer_basic`）
- `type_infer_1` ~ `type_infer_4` 通过
- `type_infer_5` 通过（编译失败且有错误信息）
- 全部类型推断测试通过（`cargo test type_infer`_）
- 原有端到端测试仍然通过（`cargo test`）
- 代码能编译（`cargo build` 无错误）
- `cargo run -- tests/type_infer_1/type_infer_1.tea --emit ir` 能产生正确的 IR 输出

