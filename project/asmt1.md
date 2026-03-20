# 实验一: 解析抽象语法树

## 1. 作业概述

本次实验的目标是**扩展 TeaLang 编译器的语法分析（Parser）阶段**，使其能够解析新增的语法特性。我们将在已有编译器 teac 的基础上，新增语法规则（Pest Grammar）、抽象语法树节点（AST Types）和解析代码（Parser），实现"源代码 → AST"这一编译器前端核心流程。

### 1.1 分数构成


| 部分        | 分值   | 说明                                       |
| --------- | ---- | ---------------------------------------- |
| **三选一**   | 60 分 | 多维数组、`for` 循环、`impl` 块与方法任选其一完整实现        |
| **必做**    | 30 分 | 浮点类型 (`f32`) + 类型转换 (`as`)               |
| **Bonus** | 10 分 | 开放任务：自行实现新语法特性，或对 teac 做出有意义的改进（被合并进主分支） |


### 1.2 特性与测试用例


| 特性             | 对应测试用例                                                                                                                | 涉及改动                                   |
| -------------- | --------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| **浮点类型 (f32)** | `float_basic`, `float_arith`, `float_cmp`, `float_cast`, `float_func`                                                 | Grammar, AST Types, Parser             |
| **类型转换 (as)**  | `float_cast`, `float_arith`, `float_func`                                                                             | Grammar, AST Expr, Parser              |
| **多维数组**       | `array_2d_basic`, `array_2d_init`, `array_2d_matmul`, `array_3d`, `attention`                                         | Grammar, AST Types, Parser             |
| **For-In 循环**  | `for_basic`, `for_continue`, `for_mixed`, `for_nested`, `for_range`                                                   | Grammar, AST Stmt, Parser              |
| **Impl 块与方法**  | `struct_method_basic`, `struct_method_calls`, `struct_method_loop`, `struct_method_namespace`, `struct_method_nested` | Grammar, AST Program/Expr/Stmt, Parser |


共 20 个新测试用例，每个对应 `tests/<name>/<name>.tea` 文件。你需要让 `teac --emit ast <file>` 对你选择的特性涉及的所有测试文件都能成功输出 AST。

> **注意**：浮点（f32）和类型转换（as）是**必做**的，因此所有同学的测试用例都包含 `float_`* 系列。三选一的特性再额外增加 5 个测试。

### 1.3 交付物

修改以下文件（允许新增文件）：

- `src/tealang.pest` — Pest 语法定义
- `src/ast/*.rs` — AST 节点定义
- `src/parser/*.rs` — Parser 实现（模块化结构，详见第 4 节）
- `src/ast.rs` — 模块导出（如新增了 AST 文件）

### 1.4 运行测试

```bash
# 运行全部测试（包括已有通过的 30 个 + 你需要实现的新测试）
cargo test

# 只运行某个特定测试
cargo test for_basic

# 运行某一类测试
cargo test float_
cargo test for_
cargo test struct_method_
cargo test array_

# 查看编译器对某个文件的 AST 输出
cargo run -- tests/for_basic/for_basic.tea --emit ast
```

### 1.5 测试原理

AST 测试（`test_ast_parse`）的检查逻辑：

1. **解析成功**：`teac --emit ast <file>` 退出码为 0
2. **无错误输出**：stderr 为空
3. **AST 非空**：stdout 包含非空的 AST 输出
4. **关键标识符存在**：AST 中包含源程序中的关键函数名、结构体名、变量名

**重要**：测试不要求特定的 AST 输出格式。你可以自由设计 AST 节点的名称和结构，只要能正确解析源代码即可。例如，`for` 循环可以用 `ForStmt` 节点表示，也可以在 AST 层面降解（desugar）为 `WhileStmt` + 循环变量。

## 2. PEG 简介

### 2.1 什么是 PEG

**PEG**（Parsing Expression Grammar，解析表达式文法）是由 Bryan Ford 于 2004 年提出的一种形式化文法，用于描述编程语言的语法。与大家之前学过的 **CFG**（上下文无关文法）不同，PEG 的核心特点是**确定性解析**，即对于任何输入，PEG 有且仅有一种解析方式，不存在歧义。

PEG 的形式定义：一个 PEG 是一个四元组 $ G = (V_N, V_T, R, e_S) $，其中 $ V_N $ 是非终结符集合，$ V_T $ 是终结符集合，$ R $ 是规则集合（每个非终结符恰好有一条规则），$ e_S $ 是起始表达式。

### 2.2 PEG 的核心操作

PEG 的"解析表达式"由以下基本操作构成：


| 操作    | PEG 记法    | Pest 记法    | 含义                       |
| ----- | --------- | ---------- | ------------------------ |
| 字面量   | `"abc"`   | `"abc"`    | 匹配精确字符串 `abc`            |
| 字符类   | `[a-z]`   | `'a'..'z'` | 匹配范围内任意字符                |
| 任意字符  | `.`       | `ANY`      | 匹配任何单个字符                 |
| 序列    | `e1 e2`   | `e1 ~ e2`  | 先匹配 `e1`，成功后继续匹配 `e2`    |
| 有序选择  | `e1 / e2` | `e1 | e2`  | 先尝试 `e1`，失败则尝试 `e2`      |
| 零次或多次 | `e`*      | `e*`       | 贪心地重复匹配 `e`，匹配零次也成功      |
| 一次或多次 | `e+`      | `e+`       | 贪心地重复匹配 `e`，至少匹配一次       |
| 可选    | `e?`      | `e?`       | 尝试匹配 `e`，失败也算成功（匹配零次）    |
| 正向前瞻  | `&e`      | `&e`       | 检查 `e` 能否匹配，但**不消耗**任何输入 |
| 负向前瞻  | `!e`      | `!e`       | 检查 `e` **不能**匹配，不消耗输入    |


### 2.3 PEG 与 CFG 的关键区别

PEG 和 CFG 最重要的区别在于**有序选择 vs. 无序选择**。

**CFG 的无序选择**：`A → α | β` 表示 `A` 可以推导出 `α`，也可以推导出 `β`，两者是平等的。如果输入同时匹配 `α` 和 `β`，就产生了**歧义**（ambiguity），而 CFG 本身不告诉你该选哪个。在 yacc/bison 等 LALR 解析器中，歧义会导致 shift-reduce 冲突，需要手动指定优先级来消解。

**PEG 的有序选择**：`A ← α / β` 表示先尝试 `α`，只有当 `α` 失败时才尝试 `β`。对于同一个输入，PEG 最多只有一种解析结果，即要么 `α` 成功，要么 `α` 失败后 `β` 成功，要么都失败。**PEG 天然不存在歧义。**

这一特性带来两个重要后果：

1. **不需要单独的 lexer**：PEG 是"无扫描器"（scannerless）的，即关键字、运算符、标识符的识别和语法解析在同一层完成。不需要像 flex + bison 那样分成词法分析和语法分析两个阶段。
2. **备选项的顺序至关重要**：在 PEG 中，调换 `α` 和 `β` 的顺序可能改变解析结果。编写 PEG 规则时，必须仔细考虑各备选项的排列顺序。

### 2.4 贪心匹配与无回溯

PEG 的重复操作符 `*`、`+` 是**贪心**的，即它们会尽可能多地匹配，并且一旦匹配成功就不会回退。这与正则表达式的贪心匹配类似，但与 CFG 不同（CFG 的 `*` 不承诺贪心）。

例如，假设有规则：

```
S ← A* A
A ← "a"
```

给定输入 `"aaa"`，`A*` 会贪心地消耗所有三个 `a`，然后最后一个 `A` 匹配时发现输入已耗尽，整个匹配失败。在 CFG 中，解析器可以回溯让 `A*` 只消耗两个 `a`，留一个给最后的 `A`。**PEG 不会这样回溯。**

这意味着编写 PEG 规则时，要格外注意贪心匹配可能"吞掉"后续需要的输入。

### 2.5 前瞻操作

前瞻是 PEG 相比传统 CFG 的独有能力（CFG 本身没有前瞻的概念；LL/LR 解析器中的 lookahead 是算法层面的，不是文法层面的）。

**正向前瞻** `&e`：检查当前位置是否能匹配 `e`，但不消耗任何字符。如果 `e` 匹配，`&e` 成功；否则失败。典型用途是关键字的尾随断言：

```
kw_break = @{ "break" ~ &(WHITESPACE | ";") }
```

这里 `&(WHITESPACE | ";")` 确保 `"break"` 后面跟着空白或分号，防止 `"breaking"` 被误识别为关键字 `break`。由于是前瞻，空白或分号本身不会被消耗，留给后续规则匹配。

**负向前瞻** `!e`：检查当前位置**不能**匹配 `e`。如果 `e` 匹配，`!e` 失败；如果 `e` 不匹配，`!e` 成功。典型用途是阻止贪心匹配：

```
suffix_non_method = { "." ~ identifier ~ !(lparen) }
```

这里 `!(lparen)` 确保成员访问 `.field` 后面不跟 `(`。如果后面跟了 `(`，说明这是方法调用而非成员访问，应该留给 method_call 规则处理。

### 2.6 PEG 的限制

PEG 有一个重要限制：**不允许左递归**。例如以下规则是非法的：

```
expr ← expr "+" term    // 左递归会导致无限递归
```

解决方法是改写为迭代形式：

```
expr ← term ("+" term)*
```

这正是 teac 中算术表达式的写法，即用 `arith_term ~ (arith_add_op ~ arith_term)*` 代替左递归。

## 3. Pest Grammar

### 3.1 Pest 简介

[Pest](https://pest.rs/) 是 Rust 生态中的一个 PEG 解析器生成器。它从 `.pest` 文件读取语法定义，在编译期通过 `pest_derive` 过程宏自动生成解析器代码。在 teac 中，语法定义在 `src/tealang.pest`，解析器入口在 `src/parser/mod.rs`。

Pest 的使用方式：

```rust
use pest::Parser as PestParser;
use pest_derive::Parser as DeriveParser;

#[derive(DeriveParser)]
#[grammar = "tealang.pest"]       // 指向 .pest 文件
struct TeaLangParser;
```

编译时，`pest_derive` 会读取 `tealang.pest`，为每条规则生成一个 `Rule` 枚举变体（如 `Rule::program`、`Rule::identifier`、`Rule::for_stmt` 等），并生成解析方法。调用方式：

```rust
let pairs = <TeaLangParser as PestParser<Rule>>::parse(Rule::program, input)?;
```

返回值 `pairs` 是一棵**解析树**，由层层嵌套的 `Pair` 节点构成。每个 `Pair` 代表某条规则的一次成功匹配，记录了：

- `as_rule()` → 匹配的规则名（`Rule::xxx`）
- `as_str()` → 匹配的原始文本
- `as_span()` → 在源码中的起止位置
- `into_inner()` → 该节点的子节点（对应规则体中引用的其他规则）

### 3.2 规则类型

Pest 提供四种规则类型，行为各不相同：


| 语法                | 类型     | 自动插入空白 | 生成 Pair 节点 | 用途                       |
| ----------------- | ------ | ------ | ---------- | ------------------------ |
| `rule = { ... }`  | 普通规则   | 是      | 是          | 大部分语法结构                  |
| `rule = @{ ... }` | 原子规则   | **否**  | 是          | 关键字、标识符、数字字面量            |
| `rule = ${ ... }` | 复合原子规则 | **否**  | 是          | 需要精确控制内部匹配但仍生成子 Pair     |
| `rule = _{ ... }` | 静默规则   | 是      | **否**      | WHITESPACE、COMMENT 等辅助规则 |


**空白自动插入**是 Pest 最方便的特性之一。对于普通规则 `{ ... }`，Pest 会在规则体中每两个相邻元素之间自动尝试匹配 `WHITESPACE`*。例如：

```pest
if_stmt = { kw_if ~ bool_expr ~ lbrace ~ code_block_stmt* ~ rbrace }
```

实际效果等价于：

```pest
if_stmt = {
    WHITESPACE* ~ kw_if ~ WHITESPACE* ~ bool_expr ~ WHITESPACE*
    ~ lbrace ~ WHITESPACE* ~ code_block_stmt* ~ WHITESPACE* ~ rbrace
}
```

这让我们可以忽略空白问题，专注于语法结构本身。

但在**原子规则** `@{ ... }` 中，空白**不会**自动插入。这对于关键字、标识符、数字这类 token 级别的规则至关重要，我们不希望 `"for"` 和后面的空白之间再插入额外的空白匹配逻辑。

### 3.3 空白与注释处理

teac 中的空白和注释通过两条特殊的静默规则定义：

```pest
WHITESPACE = _{ " " | "\t" | "\n" | "\r" }

COMMENT = _{
    line_comment
    | block_comment
}

line_comment = { "//" ~ (!("\n" | "\r") ~ ANY)* }
block_comment = { "/*" ~ (!"*/" ~ ANY)* ~ "*/" }
```

`WHITESPACE` 和 `COMMENT` 是 Pest 的**保留规则名**。它们被定义为静默规则（`_{ ... }`），意味着不会生成 Pair 节点。Pest 会在普通规则的元素之间自动尝试匹配 `(WHITESPACE | COMMENT)`*，从而实现空白和注释的自动跳过。

### 3.4 关键字定义模式

TeaLang 的关键字使用一种固定的模式来防止与标识符混淆：

**模式一：后跟必须的空白** 用于 `for`、`in`、`impl`、`fn`、`let` 等后面一定跟空格的关键字：

```pest
kw_for = @{ "for" ~ WHITESPACE }
kw_in = @{ "in" ~ WHITESPACE }
```

`@{ ... }` 保证内部不自动插入空白，所以 `"for" ~ WHITESPACE` 表示 `for` 后面必须紧跟至少一个空白字符。

**模式二：后跟前瞻断言** 用于 `i32`、`break`、`continue` 等后面可能跟分号、括号等非空白字符的关键字：

```pest
kw_i32 = @{ "i32" ~ &(WHITESPACE | semicolon | comma | rparen | lbrace | rbrace | rbracket | op_assign) }
kw_break = @{ "break" ~ &(WHITESPACE | semicolon) }
```

`&(...)` 是正向前瞻，即检查后面的字符但不消耗。这确保 `i32` 不会匹配 `i32x` 中的 `i32`，因为 `x` 不在允许的后续字符列表中。

**为什么要用这个模式？** 因为 `identifier` 规则匹配 `[a-zA-Z_][a-zA-Z0-9_]`*，如果没有尾随断言，`"for"` 可能匹配 `format` 的前三个字符，然后 `mat` 作为剩余输入会导致解析错误。原子规则 + 尾随断言保证关键字只在正确的边界上匹配。

### 3.5 teac 现有语法概览

当前 `tealang.pest` 的整体结构如下（伪代码概括）：

```
program           = SOI ~ (use_stmt | program_element)* ~ EOI
program_element   = var_decl_stmt | struct_def | fn_decl_stmt | fn_def
fn_def            = fn_decl ~ { ~ code_block_stmt* ~ }
code_block_stmt   = var_decl_stmt | assignment_stmt | call_stmt
                  | if_stmt | while_stmt | return_stmt
                  | continue_stmt | break_stmt | null_stmt
right_val         = bool_expr | arith_expr
arith_expr        = arith_term ~ (arith_add_op ~ arith_term)*
arith_term        = expr_unit ~ (arith_mul_op ~ expr_unit)*
expr_unit         = (arith_expr) | fn_call | &identifier
                  | -num | num | identifier ~ expr_suffix*
```

这是一个典型的**递归下降**结构，即 `program` 包含 `program_element`，`program_element` 包含 `fn_def`，`fn_def` 包含 `code_block_stmt`，`code_block_stmt` 包含表达式和子语句。

### 3.6 新增语法规则概览

根据 `[assignment1-syntax.md](assignment1-syntax.md)`，本次实验需要新增以下规则：


| 规则                    | 定义                                                                        |
| --------------------- | ------------------------------------------------------------------------- |
| `floatLiteral`        | `[0-9]* "." [0-9]+`                                                       |
| `arrayType`           | `"[" typeSpec ";" num "]"`                                                |
| `castExpr`            | `exprUnit ("as" typeSpec)?`                                               |
| `forStmt`             | `"for" identifier "in" rangeBound ".." rangeBound "{" codeBlockStmt* "}"` |
| `rangeBound`          | `"(" arithExpr ")" | fnCall | num | identifier`                           |
| `implDef`             | `"impl" identifier "{" fnDef* "}"`                                        |
| `selfParam`           | `"&" "mut" "self" | "&" "self"`                                           |
| `methodCall`          | `identifier exprSuffixNonMethod* "." identifier "(" rightValList? ")"`    |
| `exprSuffixNonMethod` | `"[" indexExpr "]" | "." identifier （不跟 "("）`                             |


以及对以下现有规则的修改：


| 规则              | 变更                                     |
| --------------- | -------------------------------------- |
| `program`       | 新增 `implDef` 选项                        |
| `typeSpec`      | 新增 `arrayType` 和 `f32` 选项              |
| `paramDecl`     | 新增 `selfParam` 前缀选项                    |
| `arithTerm`     | `exprUnit` → `castExpr`                |
| `exprUnit`      | 新增 `floatLiteral` 和 `-floatLiteral` 选项 |
| `fnCall`        | 新增 `methodCall` 选项                     |
| `codeBlockStmt` | 新增 `forStmt` 选项                        |


### 3.7 有序选择的常见陷阱

在编写 Pest 规则时，备选项的顺序是最容易犯错的地方。以下是几个典型场景：

**陷阱一：`identifier` 吞掉函数调用**

```pest
// 错误：identifier 会消耗 "get_limit"，留下 "()" 无法匹配
range_bound = { identifier | fn_call | num }

// 正确：fn_call 先尝试，失败（后面没有 "("）才回退到 identifier
range_bound = { fn_call | num | identifier }
```

为什么 `fn_call` 放在前面不会误匹配变量？因为 `fn_call` 内部的 `local_call` 规则是 `identifier ~ lparen ~ ...`，即如果标识符后面没有 `(`，`fn_call` 就会失败，Pest 回退到下一个备选项 `identifier`。

**陷阱二：`num` 吞掉浮点数的整数部分**

```pest
// 错误：输入 "3.14" 时，num 匹配 "3"，留下 ".14" 无法解析
expr_unit = { ... | num | float_literal | ... }

// 正确：float_literal 先尝试，"3.14" 整体被匹配为浮点数
expr_unit = { ... | float_literal | num | ... }
```

**陷阱三：贪心的 `suffix`* 吞掉方法调用的目标**

```pest
// 错误：对于 "c[0].get()"，suffix* 贪心消耗 ".get"，留下 "()" 无法解析
method_call = { identifier ~ expr_suffix* ~ dot ~ identifier ~ lparen ~ ... }

// 正确：使用负前瞻版的 suffix
expr_suffix_no_method = {
    lbracket ~ index_expr ~ rbracket
    | dot ~ identifier ~ !(lparen)       // 后面不跟 "(" 才消耗
}
method_call = { identifier ~ expr_suffix_no_method* ~ dot ~ identifier ~ lparen ~ ... }
```

## 4. Parser

### 4.1 模块结构

Parser 代码采用模块化结构，按职责拆分为多个文件，位于 `src/parser/` 目录下：

| 文件 | 职责 |
|------|------|
| `mod.rs` | `Parser` 结构体、`Generator` trait 实现、`ParseContext` 定义、解析入口 |
| `common.rs` | `Error` 枚举、`TeaLangParser`（Pest 派生）、类型别名、辅助函数 |
| `decl.rs` | 声明与定义的解析（use 语句、变量声明/定义、函数声明/定义、结构体定义） |
| `expr.rs` | 表达式的解析（算术表达式、布尔表达式、函数调用、左值、右值） |
| `stmt.rs` | 语句的解析（赋值、调用、if、while、return、break、continue） |

所有 `parse_xxx` 函数都是 `ParseContext` 结构体的方法，而非独立的自由函数。这提供了统一的调用风格（`self.parse_xxx(pair)`）和可扩展性（可在 `ParseContext` 中增加诊断收集、源码映射等上下文状态）。

### 4.2 `ParseContext` 与解析流程

teac 的解析分为两个阶段：

1. **Pest 解析**：`TeaLangParser::parse(Rule::program, input)` 将源代码转换为 Pest 的解析树（一棵由 `Pair` 节点构成的树）。
2. **AST 构建**：`ParseContext` 上的方法遍历 Pest 解析树，将其转换为 teac 自定义的 AST 节点。

`ParseContext` 是解析过程的上下文对象，所有解析方法挂载在它上面：

```rust
pub(crate) struct ParseContext<'a> {
    input: &'a str,
}
```

`Parser` 结构体持有输入和解析结果，通过 `Generator` trait 暴露给外部。在 `generate()` 时创建 `ParseContext` 并调用其 `parse()` 入口方法：

```rust
impl<'a> Generator for Parser<'a> {
    type Error = Error;

    fn generate(&mut self) -> Result<(), Error> {
        let ctx = ParseContext::new(self.input);
        self.program = Some(ctx.parse()?);
        Ok(())
    }
    // ...
}
```

`ParseContext::parse()` 是解析的入口：

```rust
impl<'a> ParseContext<'a> {
    fn parse(&self) -> ParseResult<Box<ast::Program>> {
        let pairs = <TeaLangParser as PestParser<Rule>>::parse(Rule::program, self.input)
            .map_err(|e| Error::Syntax(e.to_string()))?;

        let mut use_stmts = Vec::new();
        let mut elements = Vec::new();

        for pair in pairs {
            if pair.as_rule() == Rule::program {
                for inner in pair.into_inner() {
                    match inner.as_rule() {
                        Rule::use_stmt => use_stmts.push(self.parse_use_stmt(inner)?),
                        Rule::program_element => {
                            if let Some(elem) = self.parse_program_element(inner)? {
                                elements.push(*elem);
                            }
                        }
                        Rule::EOI => {}
                        _ => {}
                    }
                }
            }
        }

        Ok(Box::new(ast::Program { use_stmts, elements }))
    }
}
```

### 4.3 类型别名与错误处理

`common.rs` 定义了两个重要的类型别名：

```rust
type ParseResult<T> = Result<T, Error>;
type Pair<'a> = pest::iterators::Pair<'a, Rule>;
```

`Pair<'a>` 是 Pest 解析树的核心类型。它代表某条规则的一次匹配。`ParseResult<T>` 统一了成功和失败的返回类型。

错误类型定义：

```rust
pub enum Error {
    Syntax(String),         // Pest 解析失败（语法错误）
    InvalidNumber { ... },  // 数字字面量解析失败
    Io(std::io::Error),     // I/O 错误
    Grammar(String),        // 解析树结构不符合预期
}
```

其中 `Grammar` 错误通过辅助函数 `grammar_error` 生成。当遍历解析树时遇到意外的节点结构，调用它来报告内部错误：

```rust
fn grammar_error(context: &'static str, pair: &Pair<'_>) -> Error {
    let span = pair.as_span();
    let (line, column) = span.start_pos().line_col();
    let near = compact_snippet(span.as_str());
    Error::Grammar(format!(
        "{context} at line {line}, column {column}, near `{near}`"
    ))
}
```

### 4.4 `parse_xxx` 方法模式

`ParseContext` 上的方法为每条 Pest 规则编写一个对应的 `parse_xxx` 方法。这些方法的签名和结构遵循固定模式：

**基本模式**：遍历子节点，按规则类型分发：

```rust
fn parse_program_element(&self, pair: Pair) -> ParseResult<Option<Box<ast::ProgramElement>>> {
    for inner in pair.into_inner() {
        match inner.as_rule() {
            Rule::var_decl_stmt => {
                return Ok(Some(Box::new(ast::ProgramElement {
                    inner: ast::ProgramElementInner::VarDeclStmt(
                        self.parse_var_decl_stmt(inner)?
                    ),
                })));
            }
            Rule::struct_def => {
                return Ok(Some(Box::new(ast::ProgramElement {
                    inner: ast::ProgramElementInner::StructDef(
                        self.parse_struct_def(inner)?
                    ),
                })));
            }
            Rule::fn_def => {
                return Ok(Some(Box::new(ast::ProgramElement {
                    inner: ast::ProgramElementInner::FnDef(
                        self.parse_fn_def(inner)?
                    ),
                })));
            }
            // ...
            _ => {}
        }
    }
    Ok(None)
}
```

**序列模式**：按顺序收集多个子节点：

```rust
fn parse_struct_def(&self, pair: Pair) -> ParseResult<Box<ast::StructDef>> {
    let mut identifier = String::new();
    let mut decls = Vec::new();

    for inner in pair.into_inner() {
        match inner.as_rule() {
            Rule::identifier => identifier = inner.as_str().to_string(),
            Rule::var_decl_list => decls = self.parse_var_decl_list(inner)?,
            _ => {}   // 跳过 lbrace、rbrace 等分隔符
        }
    }

    Ok(Box::new(ast::StructDef { identifier, decls }))
}
```

**二元运算模式**：处理左结合的运算符链：

```rust
fn parse_arith_expr(&self, pair: Pair) -> ParseResult<Box<ast::ArithExpr>> {
    let inner_pairs: Vec<_> = pair.into_inner().collect();

    let mut expr = self.parse_arith_term(inner_pairs[0].clone())?;

    let mut i = 1;
    while i < inner_pairs.len() {
        if inner_pairs[i].as_rule() == Rule::arith_add_op {
            let op = self.parse_arith_add_op(inner_pairs[i].clone())?;
            let right = self.parse_arith_term(inner_pairs[i + 1].clone())?;
            expr = Box::new(ast::ArithExpr {
                pos: expr.pos,
                inner: ast::ArithExprInner::ArithBiOpExpr(Box::new(ast::ArithBiOpExpr {
                    op, left: expr, right,
                })),
            });
            i += 2;
        } else {
            i += 1;
        }
    }

    Ok(expr)
}
```

### 4.5 新增方法的放置规则

根据方法解析的语法元素类型，将其放入对应的文件：

| 解析内容 | 放入文件 | 示例 |
|---------|---------|------|
| 声明与定义（program 顶层元素） | `decl.rs` | `parse_fn_def`、`parse_struct_def`、`parse_var_decl`、`parse_type_spec` |
| 表达式（算术、布尔、值） | `expr.rs` | `parse_arith_expr`、`parse_bool_expr`、`parse_fn_call`、`parse_right_val` |
| 语句（函数体内） | `stmt.rs` | `parse_if_stmt`、`parse_while_stmt`、`parse_assignment_stmt` |
| 通用工具 | `common.rs` | `grammar_error`、`parse_num`、`get_pos` |

例如，新增 `for` 循环时，`parse_for_stmt` 应放入 `stmt.rs`；新增 `cast_expr` 时应放入 `expr.rs`；新增 `impl_def` 时应放入 `decl.rs`。

### 4.6 关键 API 一览

与 `Pair` 交互的常用方法：


| 方法                            | 返回类型             | 用途                      |
| ----------------------------- | ---------------- | ----------------------- |
| `pair.as_rule()`              | `Rule`           | 获取此节点匹配的规则名             |
| `pair.as_str()`               | `&str`           | 获取匹配的原始文本               |
| `pair.as_span()`              | `Span`           | 获取在源码中的位置信息             |
| `pair.into_inner()`           | `Pairs`          | 获取子节点的迭代器               |
| `pair.clone()`                | `Pair`           | 克隆（Pair 内部使用 `Rc`，克隆廉价） |
| `span.start_pos().line_col()` | `(usize, usize)` | 获取行号和列号                 |


**重要**：`into_inner()` 会消耗 `Pair`。如果后续还需要用到这个 `Pair`（例如生成错误信息），应该先 `clone()` 一份保存：

```rust
fn parse_code_block_stmt(&self, pair: Pair) -> ParseResult<Box<ast::CodeBlockStmt>> {
    let pair_for_error = pair.clone();  // 为错误信息保留一份
    for inner in pair.into_inner() {    // 消耗原始 pair
        match inner.as_rule() {
            // ...
            _ => {}
        }
    }
    Err(grammar_error("code_block_stmt", &pair_for_error))
}
```

### 4.7 现有 `expr_unit` 的解析逻辑

`parse_expr_unit` 是 Parser 中最复杂的方法之一，位于 `expr.rs`，值得详细分析。Pest 规则：

```pest
expr_unit = {
    lparen ~ arith_expr ~ rparen     // 括号表达式
    | fn_call                         // 函数调用
    | ampersand ~ identifier          // 取引用
    | op_sub ~ num                    // 负整数
    | num                             // 正整数
    | identifier ~ expr_suffix*       // 标识符 + 后缀链
}
```

由于 Pest 的有序选择，`fn_call` 会在 `identifier ~ expr_suffix*` 之前尝试。如果输入是 `std::getint()`，`fn_call` 成功匹配；如果输入是 `arr[i]`，`fn_call` 失败（因为 `[` 不是 `(`），回退到 `identifier ~ expr_suffix*`。

在 Parser 代码中，`parse_expr_unit` 通过检查子节点的规则类型来判断匹配了哪个备选项：

```rust
fn parse_expr_unit(&self, pair: Pair) -> ParseResult<Box<ast::ExprUnit>> {
    let inner_pairs: Vec<_> = pair.into_inner().collect();
    // 过滤掉 lparen/rparen
    let filtered: Vec<_> = inner_pairs.iter()
        .filter(|p| !matches!(p.as_rule(), Rule::lparen | Rule::rparen))
        .cloned().collect();

    // "-" num → 负整数
    if filtered.len() == 2
        && filtered[0].as_rule() == Rule::op_sub
        && filtered[1].as_rule() == Rule::num { ... }

    // arith_expr → 括号表达式
    if filtered.len() == 1 && filtered[0].as_rule() == Rule::arith_expr { ... }

    // fn_call → 函数调用
    if !filtered.is_empty() && filtered[0].as_rule() == Rule::fn_call { ... }

    // num → 正整数
    if filtered.len() == 1 && filtered[0].as_rule() == Rule::num { ... }

    // identifier ~ expr_suffix* → 标识符或成员/数组访问
    if !inner_pairs.is_empty() && inner_pairs[0].as_rule() == Rule::identifier { ... }

    Err(grammar_error("expr_unit", &pair_for_error))
}
```

你在新增 `float_literal` 时，需要在 `expr.rs` 的这个方法中添加对应的处理分支。

## 5. AST Display 支持

当你运行 `cargo run -- <file> --emit ast` 时，编译器将解析出的 AST 以树状格式打印到标准输出。这个功能通过 `Display` 和 `DisplayAsTree` 两个 trait 实现。

### 5.1 AST 模块结构

AST 的代码分布在 `src/ast/` 目录下：


| 文件           | 内容                                                                |
| ------------ | ----------------------------------------------------------------- |
| `types.rs`   | `BuiltIn`、`TypeSpecifier`、`TypeSpecifierInner`                    |
| `ops.rs`     | `ArithBiOp`、`ComOp`、`BoolBiOp`、`BoolUOp`                          |
| `expr.rs`    | `ArithExpr`、`ExprUnit`、`BoolExpr`、`FnCall`、`LeftVal`、`RightVal` 等 |
| `stmt.rs`    | `IfStmt`、`WhileStmt`、`AssignmentStmt`、`CodeBlockStmt` 等           |
| `decl.rs`    | `VarDecl`、`VarDef`、`FnDecl`、`FnDef`、`ParamDecl`、`StructDef`       |
| `program.rs` | `Program`、`ProgramElement`、`UseStmt`                              |
| `display.rs` | `Display` trait 的实现（单行文本表示）                                       |
| `tree.rs`    | `DisplayAsTree` trait 的实现（树状缩进表示）                                 |


`src/ast.rs` 作为模块入口，通过 `pub use` 将所有公开类型导出到 `crate::ast` 命名空间。

### 5.2 `DisplayAsTree` trait

`DisplayAsTree` 是 teac 自定义的 trait，用于将 AST 节点以带缩进和连接线的树状格式输出：

```rust
pub trait DisplayAsTree {
    fn fmt_tree(
        &self,
        f: &mut Formatter<'_>,
        indent_levels: &[bool],   // 祖先节点的"是否为最后一个"状态
        is_last: bool,            // 当前节点是否为其父节点的最后一个子节点
    ) -> Result<(), Error>;

    fn fmt_tree_root(&self, f: &mut Formatter<'_>) -> Result<(), Error> {
        self.fmt_tree(f, &[], true)
    }
}
```

`indent_levels` 是一个布尔数组，记录从根到当前节点路径上每一层的"是否为最后子节点"状态。这决定了缩进时使用 `│`  （表示还有同级节点）还是    ``（表示是最后一个）。`is_last` 决定当前节点的前缀是 `└─`（最后一个）还是 `├─`（非最后一个）。

辅助函数 `tree_indent` 根据这些信息生成缩进字符串：

```rust
fn tree_indent(indent_levels: &[bool], is_last: bool) -> String {
    let mut s = String::new();
    for &last in indent_levels.iter() {
        if last {
            s.push_str("   ");     // 祖先是最后子节点 → 无竖线
        } else {
            s.push_str("│  ");    // 祖先不是最后 → 画竖线
        }
    }
    if is_last {
        s.push_str("└─");        // 当前节点是最后子节点
    } else {
        s.push_str("├─");        // 当前节点不是最后
    }
    s
}
```

输出效果示例：

```
└─Program
   ├─FnDef main
   │  └─ReturnStmt 0
   └─FnDef add
      ├─Params:
      │  └─VarDeclList
      │     ├─x: int@...
      │     └─y: int@...
      └─ReturnStmt (x add y)
```

### 5.3 实现 `DisplayAsTree` 的模式

以 `WhileStmt` 为例：

```rust
impl DisplayAsTree for WhileStmt {
    fn fmt_tree(
        &self,
        f: &mut Formatter<'_>,
        indent_levels: &[bool],
        is_last: bool,
    ) -> Result<(), Error> {
        // 1. 打印当前节点
        writeln!(f, "{}WhileStmt Cond: {}",
            tree_indent(indent_levels, is_last),
            self.bool_unit
        )?;
        // 2. 为子节点准备新的缩进层级
        let mut new_indent = indent_levels.to_vec();
        new_indent.push(is_last);     // 把当前的 is_last 推入
        // 3. 打印子节点
        writeln!(f, "{}Body:", tree_indent(&new_indent, false))?;
        self.stmts.fmt_tree(f, &new_indent, true)
    }
}
```

**关键步骤**：

1. **打印自身**：调用 `tree_indent` 获取缩进前缀，写入节点名称和关键信息。
2. **构造子缩进**：将当前的 `is_last` 推入 `indent_levels`，形成新的缩进上下文。
3. **递归打印子节点**：最后一个子节点传 `is_last = true`，其他传 `false`。

### 5.4 `Display` trait

`Display` trait（标准库的 `std::fmt::Display`）为 AST 节点提供单行文本表示，主要用在 `DisplayAsTree` 的输出中。例如 `WhileStmt` 在打印条件时使用 `self.bool_unit`（调用 `BoolUnit` 的 `Display`）。

以几个关键类型为例：

```rust
impl Display for ExprUnitInner {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result<(), Error> {
        match self {
            ExprUnitInner::Num(n) => write!(f, "{}", n),
            ExprUnitInner::Id(id) => write!(f, "{}", id),
            ExprUnitInner::FnCall(fc) => write!(f, "{}", fc),
            ExprUnitInner::ArrayExpr(ae) => write!(f, "{}", ae),
            ExprUnitInner::MemberExpr(me) => write!(f, "{}", me),
            ExprUnitInner::Reference(id) => write!(f, "&{}", id),
            ExprUnitInner::ArithExpr(a) => write!(f, "{}", a),
        }
    }
}
```

**注意**：测试只检查 AST 中是否包含关键标识符（如函数名、变量名），因此 `Display` 的具体格式不重要，只要标识符字符串出现在输出中即可。但良好的输出格式对调试非常有帮助。

### 5.5 为新 AST 节点添加 Display 支持

新增 AST 节点后，需要：

1. **在 `display.rs` 中实现 `Display`**：提供单行文本表示。
2. **在 `tree.rs` 中实现 `DisplayAsTree`**：提供树状缩进表示。

例如，如果你新增了 `ForStmt` 节点：

```rust
// display.rs
impl Display for ForStmt {
    fn fmt(&self, f: &mut Formatter<'_>) -> Result<(), Error> {
        write!(f, "for {} in {}..{}", self.iterator, self.range_start, self.range_end)
    }
}

// tree.rs
impl DisplayAsTree for ForStmt {
    fn fmt_tree(
        &self,
        f: &mut Formatter<'_>,
        indent_levels: &[bool],
        is_last: bool,
    ) -> Result<(), Error> {
        writeln!(f, "{}ForStmt {} in ...",
            tree_indent(indent_levels, is_last),
            self.iterator
        )?;
        let mut new_indent = indent_levels.to_vec();
        new_indent.push(is_last);
        self.stmts.fmt_tree(f, &new_indent, true)
    }
}
```

同时，在 `CodeBlockStmtInner` 的 `DisplayAsTree` 实现中添加新的 match 分支：

```rust
impl DisplayAsTree for CodeBlockStmtInner {
    fn fmt_tree(&self, f: &mut Formatter<'_>, indent_levels: &[bool], is_last: bool)
        -> Result<(), Error>
    {
        match self {
            // ... 现有分支 ...
            CodeBlockStmtInner::For(stmt) => stmt.fmt_tree(f, indent_levels, is_last),
        }
    }
}
```

## 6. 提交检查

- 必做：`float_*` 系列全部通过（`cargo test float_`）
- 三选一对应的测试全部通过
- 原有 30 个端到端测试仍然通过（`cargo test`）
- 代码能编译（`cargo build` 无错误）
- 使用 `cargo run -- tests/<name>/<name>.tea --emit ast` 能产生可读的 AST 输出

