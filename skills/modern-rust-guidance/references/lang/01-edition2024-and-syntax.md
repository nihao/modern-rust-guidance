# Lang 01 — Rust 2024 语法演进、模式匹配与组合子

本分册定义 Rust 2024 Edition 核心语法与语义规则、穷尽模式匹配解构以及 `Option`／`Result` 零浪费组合子惯用法，消除滞后于旧版 Edition 的过时写法。

## 1. Rust 2024 Edition 核心语法与语义规则

### 1.1 RPIT 默认全捕获与 `+ use<...>` 精确捕获
Rust 2024 中，返回位置 `-> impl Trait`（RPIT）默认捕获函数签名作用域内的**全部泛型类型参数与生命周期参数**（改变了 Rust 2021 仅按提及捕获的旧规则）：
- 返回值确实借用输入引用时，直接写 `fn tokens(src: &str) -> impl Iterator<Item = &str>`，无需旧式 `+ '_` 样板。
- 返回值**不借用**输入短期引用（仅消费其派生出的拥有值）时，**必须显式追加 `+ use<...>` 精确捕获列表**释放输入引用的生命周期，否则调用方将被迫继续持有原输入借用。

```rust
// Bad (Rust 2024): 默认捕获了 'a，导致 prefix 释放后返回的迭代器无法继续存活
fn prefixed_ids_bad<'a>(prefix: &'a str, count: usize) -> impl Iterator<Item = String> {
    let owned = prefix.to_owned();
    (0..count).map(move |i| format!("{owned}-{i}"))
}

// Good (Rust 2024): 用 + use<> 明确声明返回类型不捕获 'a
pub fn prefixed_ids<'a>(prefix: &'a str, count: usize) -> impl Iterator<Item = String> + use<> {
    let owned = prefix.to_owned();
    (0..count).map(move |i| format!("{owned}-{i}"))
}
```

### 1.2 原生异步闭包与 Trait 异步方法
- **优先使用异步闭包 `async |...|`**：高阶异步回调统一使用 `AsyncFn`／`AsyncFnMut`／`AsyncFnOnce` 约束及 `async |x| { ... }` 语法，使闭包返回的 Future 可自然借用闭包捕获的局部变量，禁止再退回 `Fn(T) -> Pin<Box<dyn Future<Output = R> + Send + '_>>` 堆分配样板。
- **Trait 原生 `async fn` 与返回类型标记（RTN）**：静态分发 Trait 直接声明原生 `async fn`，**禁止在无需动态分发的内部 Trait 上套用 `#[async_trait]` 宏制造逐次调用堆分配**。多线程调度器需要 `Send` 约束时，在泛型调用点使用返回类型标记 `where H: Handler, H::handle(..): Send` 或显式关联 Future；仅确需 `dyn Trait` 异构集合时才单独封装对象安全适配层。

### 1.3 显式不安全声明、全局初始化与作用域收窄
- **`unsafe extern` 与 `#[unsafe(...)]` 属性**：Edition 2024 强制外部块声明为 `unsafe extern "C" { ... }`（块内仅经核实无任何未定义行为前置条件的函数方可标注 `safe fn`）；修改链接符号或段的编译器属性必须写为 `#[unsafe(no_mangle)]`、`#[unsafe(export_name = "...")]`、`#[unsafe(link_section = "...")]`。
- **禁用 `static mut`，统一用 `LazyLock`／`OnceLock`**：Rust 2024 默认禁止对 `static mut` 取引用（`static_mut_refs` 硬错误）。延迟单次初始化统一使用标准库 `std::sync::LazyLock` 或 `OnceLock`（禁止新引入 `lazy_static!` 或 `once_cell`）；无锁全局状态使用 `std::sync::atomic::Atomic*`。
- **临时值析构收窄与保留字**：Edition 2024 收窄了 `if let` 条件表达式与块尾表达式中临时值的生命周期（进入 `else` 分支前即完成析构，避免临时锁守卫意外跨分支驻留）；`gen` 为保留关键字，标识符应命名为 `generate`／`build`，避免使用 `r#gen`。

## 2. 模式匹配与穷尽解构（`pat-*`）

### 2.1 扁平化控制流：`let-else`、`if let` 链与 `matches!`
- **提前剥离异常分支**：使用 `let Some(header) = raw.header() else { return Err(...); };` 保持主干零嵌套，替代深层金字塔式 `match` 或 `if let`。
- **`if let` 链式组合**：多个模式解构与布尔守卫并列时，直接使用原生 `if let Some(cfg) = opt && let Ok(port) = cfg.port.parse::<u16>() && port > 1024`。
- **布尔模式断言**：仅判定值是否匹配特定变体或范围时，使用 `matches!(status, Status::Active | Status::Draining { .. })`。

### 2.2 枚举与结构体穷尽解构（防演进漏改）
- **内部枚举禁滥用 `_ =>`**：匹配本工作区定义的状态、命令或错误枚举时，显式列出全部变体；禁止用 `_ =>` 通配分支掩盖未来新增变体时漏改的处理逻辑。
- **关键函数结构体全字段解构**：在编写规范编码、内容散列、手工等价比较或跨层无损转换函数时，使用**不含 `..` 的全字段解构**，使日后新增字段时编译器直接报错定位：

```rust
// Good: 新增任何字段都会在此处触发编译错误，彻底杜绝散列或编码漏字段
pub fn encode_canonical(record: &PostingItem, out: &mut Vec<u8>) {
    let PostingItem { account_id, amount_minor, side } = record;
    out.extend_from_slice(account_id.as_bytes());
    out.extend_from_slice(&amount_minor.to_be_bytes());
    out.push(side.tag_byte());
}
```

### 2.3 `@` 子模式绑定与切片模式
- **`@` 整体与局部同时捕获**：既要检验子结构／数值区间又要保留完整值或变体载荷时，使用 `@` 绑定（如 `code @ 400..=499 => handle_client_err(code)` 或 `Some(node @ Node { depth: 0, .. })`）。
- **切片模式替代长度＋下标**：解析定长协议头、单元素特化或头尾拆分时，优先使用切片模式 `match parts { [] => ..., [single] => ..., [head, middle @ .., tail] => ... }`，在编译期消除越界访问风险。

## 3. `Option` 与 `Result` 组合子惯用法（`opt-*` / `err-*`）

### 3.1 惰性求值 vs 急切求值红线
组合子实参在进入方法前就会被求值。凡涉及函数调用、堆分配（`String::from`、`format!`、`Vec::new`）或复杂错误构造的回退分支，**必须使用接收闭包的 `*_else` 惰性变体**：

| 急切求值（仅限字面量／`Copy` 常量／零载荷变体） | 惰性求值（含函数调用、堆分配或构造开销时必须使用） |
| --- | --- |
| `opt.unwrap_or(0)` | `opt.unwrap_or_else(|| default_name())` |
| `opt.ok_or(ParseError::Empty)` | `opt.ok_or_else(|| ParseError::InvalidKey(raw.to_owned()))` |
| `opt.map_or(0, |s| s.len())` | `opt.map_or_else(Vec::new, |s| s.as_bytes().to_vec())` |

### 3.2 结构转换、原地槽位操作与短路收集
- **嵌套互转与透传观察**：`Option<Result<T, E>>` 与 `Result<Option<T>, E>` 之间直接调用 `.transpose()?`；链中只读观测使用 `.inspect(|v| ...)`／`.inspect_err(|e| ...)`；两个 `Option` 同时存在才执行计算时使用 `lhs.zip(rhs)`。
- **可变槽位零克隆抽取**：更新或消费 `&mut Option<T>` 内部值时，使用 `slot.take()` 或 `slot.replace(next)`，禁止先 `.as_ref().cloned()` 再赋 `None`。
- **可失败迭代短路收集**：`Iterator<Item = Result<T, E>>` 直接调用 `.collect::<Result<Vec<T>, E>>()` 或 `.try_fold(...)`，在遇到首个错误时立即停止后续迭代与多余分配。
