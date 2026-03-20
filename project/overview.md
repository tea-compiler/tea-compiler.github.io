# 实验总览

本学期课程实验中，我们将基于 teac，拓展 TeaLang 的语法，涵盖 浮点类型 (`f32`)、多维数组、类型转换、`for` 循环、`impl` 块与方法，帮助大家在实践中理解编译器的语法解析、IR 生成、优化和后端代码生成。此外，学有余力的同学可以在 bonus 中自行实现感兴趣的语法特性，或对 teac 当前实现做出有意义的改进（即被合并进入 teac 主分支）。

实验分数构成为：

1. 多维数组、`for` 循环、`impl` 块与方法**三选一**，60 分；
2. 浮点类型 (`f32`) + 类型转换 (`as`) 必做，30 分；
3. Bonus 开放任务，10 分。

## 多维数组

### 数组类型

新增 `arrayType` 规则，通过递归支持任意维度：

```
arrayType := < [ > typeSpec < ; > num < ] >
```

`typeSpec` 修改后包含 `arrayType`。这使得嵌套类型如 `[[i32; 4]; 3]` 能被正确解析。其外层由 `typed_array_decl` 匹配 `identifier : [ typeSpec ; num ]`，内层 `[i32; 4]` 由 `typeSpec → arrayType` 匹配。

### 索引

多维数组索引通过链式 `exprSuffix` 实现，无需额外改动。`mat[i][j]` 中，第一个 `[i]` 是一个 `exprSuffix`，第二个 `[j]` 是另一个 `exprSuffix`。

### 示例

```rust
let mat: [[i32; 4]; 3];           // 2D: 3 行 × 4 列
let cube: [[[i32; 4]; 3]; 2];     // 3D
let q: [[[[i32; 16]; 8]; 4]; 2];  // 4D
mat[i][j] = val;
let v:i32 = cube[i][j][k];
let row: [f32; 3];                // f32 数组
```

## `for` 循环

### `for` 语句

新增 `forStmt` 和 `rangeBound` 规则：

```
forStmt := < for > identifier < in > rangeBound < .. > rangeBound
           < { > codeBlockStmt* < } >

rangeBound := < ( > arithExpr < ) >     // 括号表达式
            | fnCall                     // 函数调用
            | num                        // 数字字面量
            | identifier                 // 变量
```

> **实现提示**：`..` 须定义为原子 token（如 `dot_dot = @{ ".." }`），不可拆为两个 `dot`。`rangeBound` 不支持负字面量（如 `-5`），需使用括号形式 `(-5)`。

`codeBlockStmt` 修改为包含 `forStmt`：

```
codeBlockStmt := varDeclStmt | assignmentStmt | callStmt | ifStmt
               | whileStmt | forStmt | returnStmt | continueStmt
               | breakStmt | nullStmt
```

### 语义

- 循环变量类型为 `i32`，作用域限定在循环体内
- `..` 产生半开区间 `[start, end)`
- 循环变量从 `start` 开始，每次迭代加 1，`>= end` 时退出
- 若 `start >= end`，循环体执行零次
- 两个边界在循环开始前各求值一次
- `break` 和 `continue` 与 while 循环行为一致

### 示例

```rust
for i in 0..10 { sum = sum + i; }         // 字面量边界
for i in 0..n { sum = sum + i; }           // 变量边界
for i in 0..get_limit() { fsum = fsum + i; }  // 函数调用边界
for i in 1..(n + 1) { result = result * i; }  // 括号表达式边界
for i in 5..5 { unreachable = 0; }         // 空范围，不执行

// 嵌套 for 循环
for i in 0..rows {
    for j in 0..cols {
        mat[i][j] = i * cols + j;
    }
}
```

## `impl` 块与方法

### `impl` 块定义

新增 `implDef` 规则，作为新的顶层元素加入 `program`：

```
implDef := < impl > identifier < { > fnDef* < } >
```

### `self` 参数

方法的第一个参数可以是 `self` 参数，可以是不可变引用或可变引用：

```
selfParam := < & > < mut > < self >      // 可变引用
           | < & > < self >               // 不可变引用
```

`paramDecl` 修改为支持 self 参数：

```
paramDecl := selfParam (< , > varDeclList)?
           | varDeclList
```

> **实现提示**：`self` 和 `Self` 在语法层面由 `identifier` 规则匹配，特殊语义在语义分析阶段处理。

在方法体内，`self` 指代接收者实例：`self.field` 访问字段，`self.method(args)` 调用同类型方法。`Self` 是当前 impl 类型的别名，`Self::method(self, args)` 等价于 `TypeName::method(self, args)`，接收者须显式传递。

### 方法调用

方法调用有两种语法：

**点语法**（dot syntax）：`receiver.method(args)` ，其中接收者隐式传递为 `self`。

```
methodCall := identifier exprSuffixNonMethod* < . > identifier
              < ( > rightValList? < ) >

exprSuffixNonMethod := < [ > indexExpr < ] >
                     | < . > identifier    （仅当后面不跟 < ( > 时）
```

**静态语法**（type-prefixed call）：`TypeName::method(receiver, args)` — 接收者显式作为首参数传递。`Self` 在 impl 块内是当前类型的别名，因此 `Self::method(self, args)` 等价于 `TypeName::method(self, args)`。

静态语法与现有的 `modulePrefixedCall` 规则匹配，无需额外语法改动。

`fnCall` 修改为包含 `methodCall`：

```
fnCall := modulePrefixedCall | methodCall | localCall
```

### 示例

```rust
// Impl 块定义
impl Counter {
    fn get(&self) -> i32 {
        return self.value;
    }

    fn add(&mut self, delta:i32) -> i32 {
        self.value = self.value + delta;
        return self.value;
    }

    fn add_twice(&mut self, delta:i32) -> i32 {
        self.add(delta);            // 点语法
        Self::add(self, delta);     // 静态语法，Self 为类型别名
        return Self::get(self);     // 等价于 Counter::get(self)
    }
}

// 外部方法调用
c[0].get()                  // 点语法
arr[1].scale_add(3)         // 带参数的点语法
b[0].pos.len1()             // 嵌套成员上的方法调用
Counter::get(c[0])          // 静态语法（接收者显式传递）

// impl 块内部方法调用
self.add(delta)             // 点语法
Self::add(self, delta)      // 静态语法，Self 为类型别名
Self::get(self)             // 等价于 Counter::get(self)
```

## 浮点类型 (`f32`) 与浮点字面量

### 浮点字面量

新增 `floatLiteral` 规则——可选的整数部分 + 小数点 + 必须的小数部分：

```
floatLiteral := [0-9]* < . > [0-9]+
```

> **实现提示**：`floatLiteral` 在 Pest 中须标记为原子规则（`@`），内部不允许空白。

示例：`3.14`, `2.0`, `.5`, `0.001`, `100.0`

### 类型系统变更

`f32` 作为新的内建类型，与 `i32` 并列。`typeSpec` 规则修改为：

```
typeSpec := refType | arrayType | < i32 > | < f32 > | identifier
```

`f32` 可用于变量声明、函数参数、函数返回值、数组元素类型等所有 `typeSpec` 出现的位置。

### 表达式单元变更

`exprUnit` 新增 `floatLiteral` 相关项。`floatLiteral` 必须在 `num` 之前尝试匹配，`< - > floatLiteral` 必须在 `< - > num` 之前：

```
exprUnit := < ( > arithExpr < ) >
          | fnCall
          | < & > identifier
          | < - > floatLiteral          // 负浮点数（新增）
          | < - > num
          | floatLiteral                // 浮点字面量（新增）
          | num
          | identifier exprSuffix*
```

### 示例

```rust
let a:f32 = 3.14;
let b:f32 = 2.0;
let c:f32 = .5;            // 无整数部分
fn fadd(a:f32, b:f32) -> f32 { return a + b; }
let row: [f32; 3];         // f32 数组
fn dot(v: &[f32]) -> f32;  // f32 引用参数
```

## 类型转换 (`as`)

### Cast 表达式

新增 `castExpr` 规则，插入到算术表达式层次中，位于 `exprUnit` 之上、`arithTerm` 之内：

```
castExpr := exprUnit (< as > typeSpec)?
```

`arithTerm` 修改为使用 `castExpr`：

```
arithTerm := castExpr (arithMulOp castExpr)*
```

`as` 的绑定力高于算术运算符：`a + b as i32` 等价于 `a + (b as i32)`。

### 示例

```rust
let f:f32 = x as f32;         // i32 → f32
let result:i32 = half as i32;  // f32 → i32（向零截断，如 -3.7 → -3）
let v:i32 = r[i] as i32;      // 对数组元素做转换
```

## 语法规则变更汇总

### 新增规则

| 规则 | 定义 |
|------|------|
| `floatLiteral` | `[0-9]* < . > [0-9]+` |
| `arrayType` | `< [ > typeSpec < ; > num < ] >` |
| `castExpr` | `exprUnit (< as > typeSpec)?` |
| `forStmt` | `< for > identifier < in > rangeBound < .. > rangeBound < { > codeBlockStmt* < } >` |
| `rangeBound` | `< ( > arithExpr < ) > \| fnCall \| num \| identifier` |
| `implDef` | `< impl > identifier < { > fnDef* < } >` |
| `selfParam` | `< & > < mut > < self > \| < & > < self >` |
| `methodCall` | `identifier exprSuffixNonMethod* < . > identifier < ( > rightValList? < ) >` |
| `exprSuffixNonMethod` | `< [ > indexExpr < ] > \| < . > identifier (不跟 < ( >)` |

### 修改的现有规则

| 规则 | 变更 |
|------|------|
| `program` | 新增 `implDef` 选项 |
| `typeSpec` | 新增 `arrayType` 和 `< f32 >` 选项 |
| `paramDecl` | 新增 `selfParam` 前缀选项（原为 `varDeclList`） |
| `arithTerm` | `exprUnit` → `castExpr` |
| `exprUnit` | 新增 `floatLiteral` 和 `< - > floatLiteral` 选项 |
| `fnCall` | 新增 `methodCall` 选项 |
| `codeBlockStmt` | 新增 `forStmt` 选项 |

### 新增关键字

`impl`, `for`, `in`, `f32`, `as`, `self`, `Self`, `mut`

## 运算符优先级

实现全部特性后的优先级（从高到低）：

1. **Primary**：字面量、标识符、括号表达式、函数/方法调用
2. **Suffix**：数组索引 `[]`、成员访问 `.`
3. **Cast**：`as`
4. **Multiplicative**：`*`, `/`
5. **Additive**：`+`, `-`
6. **Comparison**：`<`, `>`, `<=`, `>=`, `==`, `!=`
7. **Logical AND**：`&&`
8. **Logical OR**：`||`

> **注意**：由于 `boolComparison` 操作数为 `exprUnit`，比较运算符两侧的算术或 `as` 表达式须用括号包裹，如 `(a + b) > c`、`(x as f32) > 0.0`。

## 完整示例

> 以下示例综合使用了全部特性（多维数组、`for` 循环、`impl` 块、`f32`、`as`）。三选一的同学请按需裁剪。

```rust
use std;

struct Counter {
    value:i32
}

impl Counter {
    fn get(&self) -> i32 {
        return self.value;
    }

    fn add(&mut self, delta:i32) -> i32 {
        self.value = self.value + delta;
        return self.value;
    }
}

fn dot_product(a: &[f32], b: &[f32], n:i32) -> f32 {
    let sum:f32 = 0.0;
    for i in 0..n {
        let prod:f32 = a[i] * b[i];
        sum = sum + prod;
    }
    return sum;
}

fn main() -> i32 {
    // 多维数组
    let mat: [[i32; 3]; 2];
    for i in 0..2 {
        for j in 0..3 {
            mat[i][j] = i * 3 + j;
        }
    }

    // 浮点与类型转换
    let x:i32 = 42;
    let f:f32 = x as f32;
    let half:f32 = f / 2.0;
    let result:i32 = half as i32;
    std::putint(result);
    std::putch(10);

    // 方法调用
    let counters: [Counter; 1];
    counters[0].value = 0;
    counters[0].add(10);
    std::putint(counters[0].get());
    std::putch(10);

    return 0;
}
```
