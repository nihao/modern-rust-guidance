# Eng 04 — 契约保稳、工作区 Lint、性能实测、高信噪比测试与审查

本分册定义 Serde 序列化安全与契约保稳、双 Cargo Feature 图核验、工作区 Lint 门禁、性能实测与外部依赖克制、高信噪比测试分层以及代码变更审查标准。

## 1. Serde 契约保稳与双 Feature 图验证（`serde-*` / `proj-*`）

### 1.1 受检反序列化与传输层隔离
- **严禁绕过受检构造器**：携带格式、非空或范围不变量的领域类型在实现 `Deserialize` 时，必须通过 `#[serde(try_from = "RawRepr")]` 或手写 `Deserialize` 委托给受检构造器；**严禁直接对含不变量的私有字段类型派生 `#[derive(Deserialize)]` 让外部报文或数据库 JSON 列绕过校验**。
- **协议表示修饰留存在边界 DTO**：外部接口特有的大小写规则（如 `#[serde(rename_all = "camelCase")]`）、空字段省略或特殊编码（`#[serde(with = "...")]`）统一声明在协议边界 `Request`／`Response` DTO 上，不向纯领域模型泄漏传输协议细节。

```rust
// Good: 反序列化强制经由 TryFrom 执行非空裁剪校验
#[derive(Debug, Clone, PartialEq, Eq, serde::Serialize, serde::Deserialize)]
#[serde(try_from = "String", into = "String")]
pub struct NonEmptyIdentifier(String);

impl TryFrom<String> for NonEmptyIdentifier {
    type Error = IdentifierError;
    fn try_from(raw: String) -> Result<Self, Self::Error> {
        let trimmed = raw.trim();
        if trimmed.is_empty() {
            return Err(IdentifierError::Empty);
        }
        Ok(Self(trimmed.to_owned()))
    }
}

impl From<NonEmptyIdentifier> for String {
    fn from(value: NonEmptyIdentifier) -> Self {
        value.0
    }
}
```

### 1.2 规范编码稳定性与双 Feature 构建图核验
- **规范序列化固定键序**：凡进入持久化摘要、数字签名或跨版本等价核验的 JSON／字节流，必须显式固定对象键字典序并保持数组顺序，严禁依赖随机映射，严禁修改历史预期摘要掩盖编码漂移。
- **内部重构零外部破坏**：仅重命名内部 Rust 结构体、变体或模块路径时，必须通过 `#[serde(rename = "...")]` 及公开 Schema 名称绑定保持线上报文格式零变化；确需修改公开契约时必须显式对齐全部消费者并同步更新契约定义。
- **孤立与合并双 Feature 图校验**：Cargo 工作区会自动对下游可选特性取并集（Feature Unification）。修改基础纯 crate 依赖或 `#[cfg(feature = "...")]` 时，必须分别核验**孤立纯 crate 构建图**（确保未隐式引入异步／数据库依赖）与**上层服务合并构建图**（确保特性并集未改变数值精度、键序或散列后端行为）。

## 2. 工作区 Lint 纪律与性能实测铁律（`lint-*` / `opt-*`）

### 2.1 工作区级 Lint 继承与窄作用域 `#[expect]`
- 成员 crate 统一通过 `[lints] workspace = true` 继承根 `Cargo.toml` 的编译器与 Clippy 规则（因 `unsafe_code = "forbid"` 等原因独立声明 `[lints]` 表的 crate 必须完整镜像同步其余全部工作区规则）。
- **严禁源码 `#![deny(warnings)]` 与 `#[allow(...)]`**：源码级 `#![deny(warnings)]` 易随工具链升级无差别破坏历史构建；`#[allow]` 在误报消失后会永久静默残留。遇到确实无法重构消除的单点误报时，仅在最窄语法项上使用 **`#[expect(clippy::lint_name, reason = "客观技术原因")]`**。测试目标的 `unwrap`/`expect`/`panic`/`dbg` 放行统一收口至根 `clippy.toml`。

### 2.2 外部微优化依赖克制与实测基准铁律
- **未经基准实测严禁引入微优化第三方库**：在缺乏端到端火焰图或基准数据证明存在真实瓶颈前，禁止私自引入 `smallvec`、`tinyvec`、`bumpalo`、`rayon`、`parking_lot`、`ahash`、`indexmap` 或自定义全局分配器，禁止盲目铺设 `#[inline(always)]`，也禁止在子 crate 私自覆盖全局编译优化 profile。
- **量化证据铁律**：任何性能优化声明必须附带**同输入规模、同编译器与 feature 图、同预热条件**下的前后对比数据（P50/P99 延迟、吞吐、堆分配次数与字节数、峰值 RSS 或数据库 `EXPLAIN (ANALYZE, BUFFERS)`）；代码行数减少不代表运行期更快。

## 3. 高信噪比测试分层、重构迁移与代码审查（`test-*`）

### 3.1 高信噪比测试边界（拒绝薄胶水防御单测）
以最小编译链接开销守住最高价值不变量，主动清理过时、重复与墓碑测试：
- **单元测试（Unit Tests）**：**仅针对纯领域核心、精确数值／公式算子、状态机转移、规范编解码不变量、声明式策略求值器以及全局路由／契约闭环守卫编写单元测试**；纯算子具明确代数守恒或幂等律时可配轻量属性检验。**严禁为薄适配胶水（单端点参数解包、纯 DTO／Row 字段搬运、`thiserror` 文本展示、单句 SQL 调用包装、CLI 参数转发）编写防御性单测**，严禁引入重型数据库 mock 框架，严禁在生产 `src/` 内混入真实数据库集成测试。
- **数据库集成测试（Integration Tests）**：每个需要持久化验证的库 crate 统一收敛至**唯一 `tests/postgres.rs` 顶层测试二进制入口**（按子域在 `tests/postgres/` 目录下拆分子模块挂载，杜绝多个 `tests/*.rs` 二进制引发十倍重复编译链接开销），通过按迁移摘要缓存的模板库执行 `CREATE DATABASE ... TEMPLATE` 快速克隆隔离测试库；聚焦主线业务闭环、事务原子性、并发锁重验与数据库完整性约束，不编写脆弱的网络流量截断代理或锁表故障注入线束。

### 3.2 行为保持迁移与三级审查标准
- **重构与跨语言移植前置盘点**：动手前先锁定可观察契约（合法／非法边界、空集与重复项语义、确定性排序、数值符号／尺度／舍入／尾差分配、规范散列测试向量、错误分类与重试前提、事务锁序与幂等不变量）。用地道 Rust 类型与所有权重新建模，严禁将源语言十进制／高精度数值降级为 `f64`；新路径闭环验证后立即清除旧实现与兼容死代码。
- **代码审查分级与诚实报告**：按 **“正确性、内存与并发安全、数值守恒、权限与事务失败关闭 → 错误因果链留存、锁序与防误用类型 → 所有权清晰度、批量查询性能与可读性”** 展开，每条结论定位到精确 `文件:行号`、触发路径、违反准则与最小修复代码：

| 级别 | 判定标准 | 交付处理 |
| --- | --- | --- |
| **必须修复（Blocker）** | 计算／守恒错误、可达路径非契约 panic、内存／并发不安全、公开契约意外破坏、鉴权／审计／事务原子性漏洞 | 交付前必须修复或如实上报阻断 |
| **应修复（Major）** | 擦除 `#[source]` 根因链、职责越界或双份状态真相、循环 `.await` N+1／无界内存分配、易误用的内部签名 | 说明具体风险并给出最小修复方案 |
| **建议（Minor）** | 局部命名、导入整理或等价惯用法微调，无正确性与安全影响 | 不阻断交付，不借机扩大为无关重构 |

- **最小充分验证与诚实交付**：提交前检查 `git diff` 无意外改动或调试残留；按改动面执行匹配的检查组合并分项报告结果，**凡未实际执行的检查项一律如实标为 `NOT RUN`**。
