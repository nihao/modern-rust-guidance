# `modern-rust-guidance` 设计原则与上游来源说明

## 1. 定位与双层正交结构

本技能包是一套**自包含、业务中立的现代 Rust 2024 语言机制与服务端工程规范**，面向高可靠性、强一致性与精确计算的 Rust 工作区：

- **`references/lang/`（语言与语法特性层）**：聚焦 Rust 2024 Edition 语法与语义演进（RPIT `+ use<...>`、`async |...|`、原生异步 Trait、`unsafe extern`/`#[unsafe(...)]`、`LazyLock`）、穷尽模式解构与惰性组合子、所有权借用消解、Niche 与大枚举 `Box` 布局、析构顺序、Trait 与泛型单态化控制、卫生声明宏、Future 状态机体积、并发原语内存序及 `unsafe`/FFI 健全性。
- **`references/eng/`（高可靠服务端工程层）**：聚焦轻量 DDD（语义与一致性边界）与实用 DOD（紧凑连续布局与切片批处理）正交分工、垂直切片与用例级 CQS、纯核心零 I/O、涉精度零浮点与受检定点算术、三层 `thiserror` 因果链、单点脱敏日志与非盲目重试、PostgreSQL/SQLx 无状态连接与唯一事务提交者流水线、受检 Serde 与规范编码、工作区 Lint、性能实测纪律、高信噪比测试分层及三级代码审查标准。
- **零外部文档依赖**：所有分册仅引用本目录内的 `SKILL.md`、`LICENSE` 与 `references/{lang,eng}/*.md`，不依赖外部相对路径或特定业务域概念，可直接跨工作区复用。

## 2. 外部上游借鉴与工程收敛

本技能深度吸收了 `https://github.com/leonardomso/rust-skills`（MIT 许可证，见同目录 [LICENSE](LICENSE)）在 Rust 2024 语法特性、组合子、模式匹配、所有权、内存布局、Trait/泛型、异步并发与 `unsafe` 维度的规则精华，并针对严肃服务端工程做出严格收敛：

1. **优先编译器静态门禁**：将 `unsafe_op_in_unsafe_fn`、`undocumented_unsafe_blocks`、`missing_safety_doc`、`await_holding_lock`、`unwrap_used`、`allow_attributes`、`implicit_clone`、`inefficient_to_string` 等确定性检查直接固化至工作区 `[workspace.lints]`。
2. **剔除高风险与过度工程建议**：排除涉精度场景的浮点近似运算、库层泛用 `anyhow`、内部私有工作区泛用 Builder／密封 Trait／`#[non_exhaustive]`、`#[allow]` 压制、未经基准实测私自引入微优化第三方库（如 `smallvec`、`bumpalo`、`rayon`、`indexmap`）以及针对薄适配胶水的冗余 mock 单测。

## 3. 维护准则

修改本技能时，保持 `lang/` 与 `eng/` 职责正交、代码示例符合 Rust 2024 惯用法且目录内相对链接完整闭合；禁止向技能包内混入特定项目业务名词、环境凭据或跳出本技能目录的外部相对路径链接。
