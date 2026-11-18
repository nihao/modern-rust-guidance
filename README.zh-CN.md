<p align="center">
  <img src="assets/logo.svg" alt="Modern Rust Guidance Logo" width="112" />
</p>

# Modern Rust Guidance

<p align="center"><a href="README.md">English</a> · <b>简体中文</b></p>

面向 AI 编码代理的 **Rust 2024 语言惯用法与高可靠服务端工程规范** 插件。本技能包保持客观、自包含、业务中立，覆盖 Rust 2024 语法演进、内存与并发健全性、精确整数与定点核算、三层 `thiserror` 错误因果链、PostgreSQL/SQLx 无状态事务流水线及高信噪比测试分层。

---

## 安装方式

<details open>
<summary><b>Google Antigravity</b></summary>

```shell
agy plugin install https://github.com/nihao/modern-rust-guidance
```

</details>

<details>
<summary><b>Vercel Skills CLI (<code>npx skills</code>)</b></summary>

```shell
npx skills add nihao/modern-rust-guidance
```

</details>

<details>
<summary><b>GitHub CLI</b></summary>

```shell
gh skill install nihao/modern-rust-guidance
```

</details>

<details>
<summary><b>Claude Code</b></summary>

```shell
/plugin marketplace add nihao/modern-rust-guidance
/plugin install modern-rust-guidance@nihao
```

</details>

<details>
<summary><b>OpenAI Codex</b></summary>

```shell
codex plugin marketplace add nihao/modern-rust-guidance
codex plugin add modern-rust-guidance@nihao
```

</details>

<details>
<summary><b>手动复制到工作区</b></summary>

将 [`skills/modern-rust-guidance`](skills/modern-rust-guidance/SKILL.md) 复制到目标项目的 `.agents/skills/modern-rust-guidance` 目录。

</details>

---

## 双层按需渐进加载分册

入口规范文件为 [`skills/modern-rust-guidance/SKILL.md`](skills/modern-rust-guidance/SKILL.md)。编码代理按任务触及的**语言机制**与**工程边界**按需读取对应分册，无需一次性加载全部文档：

### 1. 语言与语法特性层（`references/lang/`）

| 分册 | 覆盖范围 |
| --- | --- |
| [01-edition2024-and-syntax.md](skills/modern-rust-guidance/references/lang/01-edition2024-and-syntax.md) | RPIT `+ use<...>`、`async \|...\|`、原生异步 Trait、`unsafe extern`/`#[unsafe(...)]`、`LazyLock`、穷尽解构与 `@`/切片模式、惰性组合子 |
| [02-ownership-memory-and-collections.md](skills/modern-rust-guidance/references/lang/02-ownership-memory-and-collections.md) | 参数借用契约、`Cow`、四步借用消解、`mem::take`、Drop 顺序、Niche 与大枚举 `Box` 瘦身、`Entry` API、`BTreeMap`、缓冲流 I/O |
| [03-types-traits-generics-and-macros.md](skills/modern-rust-guidance/references/lang/03-types-traits-generics-and-macros.md) | 受检构造与 Newtype、关联类型 vs 泛型、`Borrow` vs `AsRef`、薄泛型壳去膨胀、标准 Trait 一致性、闭包捕获、`const {}`、卫生 `macro_rules!` |
| [04-async-concurrency-and-unsafe.md](skills/modern-rust-guidance/references/lang/04-async-concurrency-and-unsafe.md) | Future 跨 `.await` 体积、`spawn_blocking`、锁不跨 `.await`、`select!` 取消安全、有界 Channel 与 Atomic 内存序、Rust 2024 `unsafe`/FFI |

### 2. 高可靠服务端工程层（`references/eng/`）

| 分册 | 覆盖范围 |
| --- | --- |
| [01-architecture-cqs-and-api.md](skills/modern-rust-guidance/references/eng/01-architecture-cqs-and-api.md) | 轻量 DDD × 实用 DOD 范式分工、用例级 CQS、纯核心零 I/O、薄 `mod.rs`、工程角色后缀、存储恢复边界与内部 API 克制 |
| [02-numeric-errors-and-observability.md](skills/modern-rust-guidance/references/eng/02-numeric-errors-and-observability.md) | 涉精度零浮点与 `checked_*` 定点算术、三层 `thiserror` 因果链、禁用越权 `anyhow`、边界单点脱敏日志、后台非盲目重试 |
| [03-postgres-sqlx-and-transactions.md](skills/modern-rust-guidance/references/eng/03-postgres-sqlx-and-transactions.md) | 运行时参数化 SQLx、无状态连接矩阵、私有 `FromRow`、`ANY`/`UNNEST` 防 N+1、两类写事务流水线、失败关闭与追加迁移 |
| [04-serde-lints-testing-and-review.md](skills/modern-rust-guidance/references/eng/04-serde-lints-testing-and-review.md) | 受检 `#[serde(try_from)]`、规范固定键序、双 Feature 图、工作区 Lint 与 `#[expect]`、性能实测铁律、高信噪比测试与三级审查 |

---

## 全局十大硬红线

1. **纯核心同步零 I/O**：领域纯计算模块保持同步确定性计算，不引入 `async`、数据库驱动、网络、文件、环境变量、时钟或随机源。
2. **组合根与 CQS 单向正交**：服务宿主仅负责装配与协议解包；写命令唯一提交状态，查询只读投影已提交事实，写命令不反向依赖 `queries`。
3. **涉精度数值绝对零浮点**：要求精确守恒的金额、数量、比率、权重与阈值禁用 `f32`/`f64` 与截断 `as`，使用明确尺度的有符号整数及 `checked_*`/`TryFrom`。
4. **强类型因果链与禁用越权 `anyhow`**：生产代码禁用 `.unwrap()`；库与请求处理层禁用 `anyhow`，用 `thiserror` 保留 `#[source]`。
5. **唯一事务提交者与失败关闭**：对外写命令开启事务并唯一 `commit`，协作函数借用 `&mut Transaction`；状态写入、审计与幂等收据同事务原子提交。
6. **运行时参数化 SQL 与防 N+1**：使用参数化运行时 SQLx 配私有 `FromRow`（禁持池 `Repository` 套壳）；批量查询用 `ANY($1)`/`UNNEST`，禁循环内 `.await`。
7. **Edition 2024 现代语法与受检构造**：遵循 RPIT `+ use<...>`、原生 `async |...|` 与惰性组合子；带不变量类型保持私有字段与受检构造。
8. **确定性顺序与加密摘要**：可观察输出与规范序列化不依赖 `HashMap`/`HashSet` 随机顺序；持久化摘要使用规范编码与加密散列，禁用 `DefaultHasher`。
9. **并发守卫与 Rust 2024 `unsafe`**：同步锁不跨 `.await`；RAII 守卫必须具名绑定；CPU 重计算卸载至 `spawn_blocking`；`unsafe fn` 内包裹最小 `unsafe {}` 并配 `# Safety` 与 `// SAFETY:`。
10. **Lint 纪律与高信噪比测试**：禁用源码 `#![deny(warnings)]` 与 `#[allow]`，单点误报用 `#[expect]`；单元测试聚焦纯逻辑算子与闭环守卫，持久化验证收敛至唯一 `tests/postgres.rs`。

---

## 上游致谢与许可证

- 语言层规则精华深度借鉴自 [`leonardomso/rust-skills`](https://github.com/leonardomso/rust-skills)（MIT License），并针对高可靠后端工程与编译器静态门禁做了系统收敛，详见 [`UPSTREAM.md`](skills/modern-rust-guidance/UPSTREAM.md)。
- 本项目基于 [MIT License](LICENSE) 发布。
