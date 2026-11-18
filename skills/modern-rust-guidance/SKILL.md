---
name: modern-rust-guidance
description: >-
  MANDATORY: 现代 Rust 2024 语言机制与高可靠服务端工程规范。在编写、修改、重构或审查任何 Rust 代码与数据库迁移前必须首先加载，不可跳过。分设语言语法层（Edition 2024 新特性、模式解构与组合子、所有权与内存布局、类型/Trait/泛型/宏、Future/并发原语/Unsafe）与工程质量层（DDD/DOD 范式分工与 CQS、精确整数/定点算术与错误因果链、PostgreSQL/SQLx 事务、Serde/Lint/高信噪比测试与审查）。
---

# Modern Rust 2024 语言机制与高可靠工程指南

本技能定义客观、自包含、跨业务中立的 **Rust 2024 语言特性惯用法** 与 **高可靠服务端工程规范**。具体业务规则、领域状态机与宿主仓库包拓扑由所在项目文档定义；本技能聚焦现代 Rust 语法机制、内存与并发健全性、数值精确性及工程实现质量。

### 规范用语

| 用语 | 含义 |
| --- | --- |
| **必须** | 适用范围内的硬性要求；缺失即违反规范 |
| **禁止／不得** | 适用范围内严禁出现的行为 |
| **应／优先／默认** | 推荐选择；偏离时须有真实需求、影响说明与验证依据 |
| **可以／按需** | 允许按真实场景选用，不构成新增抽象、依赖或测试的强制要求 |

## 1. 双层按需渐进加载路由（自包含分册）

触碰代码前，按任务涉及的**语言机制**与**工程边界**按需加载对应分册（无需一次性全量读取全部 8 份分册）：

### 1.1 语言与语法特性层（`references/lang/`）

| 分册 | 负责范围 | 何时必须读取 |
| --- | --- | --- |
| [lang/01-edition2024-and-syntax.md](references/lang/01-edition2024-and-syntax.md) | RPIT `+ use<...>`、`async \|...\|`、原生异步 Trait、`unsafe extern`/`#[unsafe(...)]`、`LazyLock`、穷尽解构与 `@`/切片模式、`Option`/`Result` 惰性组合子 | 编写返回 `impl Trait`、异步闭包／Trait、模式匹配或组合子链 |
| [lang/02-ownership-memory-and-collections.md](references/lang/02-ownership-memory-and-collections.md) | 参数借用契约、`Cow`、四步借用消解、`mem::take`、Drop 顺序、Niche 与大枚举 `Box` 瘦身、`Entry` API、`BTreeMap`、切片迭代与 `BufReader`/`BufWriter` | 处理借用冲突、设计大枚举／结构体布局、操作键值集合、切片或文件流 |
| [lang/03-types-traits-generics-and-macros.md](references/lang/03-types-traits-generics-and-macros.md) | 受检构造与 Newtype、关联类型 vs 泛型、`Borrow` vs `AsRef`、薄泛型壳去膨胀、标准 Trait 一致性、闭包捕获、`const {}`、卫生 `macro_rules!` | 定义新类型／枚举／Trait、编写泛型转换、回调闭包、常量断言或声明宏 |
| [lang/04-async-concurrency-and-unsafe.md](references/lang/04-async-concurrency-and-unsafe.md) | Future 跨 `.await` 体积、`spawn_blocking`、锁不跨 `.await`、`select!` 取消安全、有界 Channel 与 Atomic 内存序、Rust 2024 `unsafe` 与 FFI ABI | 编写异步并发任务、通道通信、原子同步、锁守卫或任何 `unsafe`/FFI 代码 |

### 1.2 高可靠服务端工程层（`references/eng/`）

| 分册 | 负责范围 | 何时必须读取 |
| --- | --- | --- |
| [eng/01-architecture-cqs-and-api.md](references/eng/01-architecture-cqs-and-api.md) | 轻量 DDD × 实用 DOD 范式分工（去 OOP 套壳）、用例级 CQS、纯核心零 I/O、薄 `mod.rs`、角色后缀、存储恢复边界与内部 API 克制 | 新建／拆分模块、设计领域状态机或批量计算 `engine`、设计 `commands`／`queries` |
| [eng/02-numeric-errors-and-observability.md](references/eng/02-numeric-errors-and-observability.md) | 涉精度零浮点与 `checked_*` 定点算术、三层 `thiserror` 因果链、禁越权 `anyhow`、边界单点脱敏日志、后台任务非盲目重试 | 编写精确数值／比率折算、定义或映射错误枚举、打日志或编写后台重试恢复 |
| [eng/03-postgres-sqlx-and-transactions.md](references/eng/03-postgres-sqlx-and-transactions.md) | 运行时参数化 SQLx、无状态连接矩阵、私有 `FromRow`、`ANY`/`UNNEST` 防 N+1、两类写事务流水线、审计／幂等失败关闭与追加迁移 | 编写 SQL 查询或 DML、编排写事务、控制大载荷配额或新增 `.sql` 迁移 |
| [eng/04-serde-lints-testing-and-review.md](references/eng/04-serde-lints-testing-and-review.md) | 受检 `#[serde(try_from)]`、规范固定键序、双 Feature 图、工作区 Lint 与 `#[expect]`、性能实测铁律、高信噪比测试分层与三级代码审查 | 修改序列化结构、`Cargo.toml` 特性／Lint、性能调优、编写测试、迁移或审查代码 |

## 2. 全局十大硬红线速查（Critical Invariants）

1. **纯核心同步零 I/O**：领域纯计算模块与算子引擎保持同步确定性计算，禁止引入 `async`、数据库驱动、异步运行时、网络、文件系统、环境变量、系统时钟或随机源。
2. **组合根与 CQS 单向正交**：服务宿主只做装配、协议解包与派发，禁止内嵌业务 SQL 或直接提交业务事务；写命令改变状态并唯一提交，查询只读投影已提交事实，**禁止写命令反向依赖 `queries` 模块**。
3. **涉精度数值绝对零浮点**：凡要求精确守恒的金额、数量、比率、权重与阈值，严禁使用 `f32`／`f64`、`as` 截断强转或 `wrapping_*`／`saturating_*`；必须使用明确尺度的有符号整数或定点类型及 `checked_*`／`TryFrom`。
4. **强类型因果链与禁用越权 `anyhow`**：生产代码严禁 `.unwrap()`；库 crate 及请求协议处理层严禁引入 `anyhow`（`anyhow` 仅限进程启动引导与 CLI 顶层）；使用 `thiserror` 保留原始 `#[source]`，禁止在分类前 `.to_string()` 擦除类型。
5. **唯一事务提交者与失败关闭**：对外写命令持有连接池开启事务并唯一 `commit`，同事务协作函数接收 `&mut Transaction`；状态写入、必要审计证据与幂等收据必须同事务原子提交，任一失败整笔回滚。
6. **运行时参数化 SQL 与防 N+1**：统一使用参数化运行时查询配合模块私有 `FromRow`（禁持池 `Repository` 套壳与 `Arc<PgPool>`）；严禁循环内 `.await` 逐条查库，必须使用 `ANY($1)`／`UNNEST` 批量读写。
7. **Edition 2024 现代语法与受检构造**：遵循 RPIT `+ use<...>` 精确捕获、原生 `async |...|` 与惰性组合子 `*_else`；携带不变量的类型保持私有字段＋受检构造（反序列化与存储恢复严禁绕过结构校验），内部非发布 crate 拒绝推测性 Builder／`Service` 套壳／默认 `#[non_exhaustive]`。
8. **确定性顺序与加密摘要**：可观察输出、分页列表和规范序列化严禁依赖 `HashMap`／`HashSet` 随机遍历顺序；持久化内容摘要必须基于确定性规范编码与加密散列，严禁用 `DefaultHasher`。
9. **并发守卫与 Rust 2024 `unsafe`**：同步锁严禁跨越 `.await`；RAII 守卫必须具名绑定（禁 `let _ = guard`）；阻塞 CPU 重计算卸载至 `spawn_blocking`；`unsafe fn` 内仍须包裹最小 `unsafe {}` 块并配齐 `# Safety` 与 `// SAFETY:`。
10. **Lint 纪律与高信噪比测试**：严禁源码 `#![deny(warnings)]` 与 `#[allow]`，单点误报仅用最窄作用域 `#[expect(clippy::..., reason = "...")]`；**仅对纯逻辑算子与协议闭环守卫写单元测试**，持久化验证按每包唯一 `tests/postgres.rs` 聚合编译入口收敛。

## 3. 标准开发与验证纪律

1. **局部快速反馈（能 `check` 不 `build`）**：单包迭代优先运行 `cargo check -p <crate> --all-targets --locked` 与纯库单测 `cargo test -p <crate> --lib --locked`。
2. **双构建图核验**：修改基础纯 crate 依赖或 Cargo feature 时，分别校验孤立纯包构建图与上层服务合并 feature 构建图，防止隐式特性传染。
3. **零警告门禁**：跨模块收口运行 `cargo fmt --all -- --check` 与 `cargo clippy --workspace --all-targets --locked -- -D warnings`。
4. **诚实交付报告**：明确区分静态检查、纯单测、数据库集成测试、真实进程与端到端验证；未实际执行的检查项一律如实标为 `NOT RUN`。
