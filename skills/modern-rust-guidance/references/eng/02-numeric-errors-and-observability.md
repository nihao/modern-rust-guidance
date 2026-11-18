# Eng 02 — 精确数值算术、强类型错误链、脱敏日志与非盲目重试

本分册定义涉精度领域零浮点与定点核算准则、三层 `thiserror` 因果链模型、边界单点脱敏日志以及后台异步任务非盲目重试规约。

## 1. 精确数值与定点算术安全（`num-*`）

### 1.1 涉精度领域绝对零浮点铁律
- **严禁使用 `f32`、`f64`**：凡要求精确相等与总额守恒的数值（金额、余额、调整额、汇率、费率、分摊权重、精确配额），在内存模型、中间求值、数据库编解码与 JSON 传输全链路中**绝对禁止使用二进制浮点数**（`0.1 + 0.2 != 0.3`、`NaN` 与结合律失效会直接破坏数值守恒与散列可复现性）。
- **明确尺度与有符号语义**：使用明确精度尺度的有符号整数最小单位（如 `i64` 配合宽位 `i128` 受检累加器）或固定小数位定点原语。不在类型外隐式假设全局统一小数位，也不盲目用无符号 `u64` 表示可能出现冲销、差额或反向变动的净值。

### 1.2 受检算术、宽位折算与尾差守恒
- **强制 `checked_*` 与 `TryFrom`**：所有存在溢出可能的加减乘除与位宽收窄必须使用 `checked_add`、`checked_sub`、`checked_mul`、`checked_div` 与 `TryFrom`（无损放宽使用 `From`）。**严禁使用 `as` 进行可能截断高位或翻转符号的数值强转**，严禁在精确核算中使用 `wrapping_*` 或 `saturating_*` 静默吞掉溢出；落库前先核验值域满足目标数据库列范围。
- **量纲隔离与显式尾差分配**：不同币种、不同量纲或不同定点尺度的数值严禁裸整数直接相加减；乘除比例折算必须先提升至更宽中间整数域（如 `i128`）完成明确舍入规则后再 `TryFrom` 收窄；多项按权重分摊必须显式定义最大余数或尾项吸收规则，并校验分摊后各项之和严格等于原总额。

```rust
// Good: 提升至 i128 执行百万分比半数远离零四舍五入 + TryFrom 安全收窄
pub fn mul_ppm_half_away_from_zero(amount_minor: i64, rate_ppm: i64) -> Result<i64, NumericError> {
    const SCALE: i128 = 1_000_000;
    let product = i128::from(amount_minor)
        .checked_mul(i128::from(rate_ppm))
        .ok_or(NumericError::Overflow)?;
    let half = if product >= 0 { SCALE / 2 } else { -(SCALE / 2) };
    let rounded = product.checked_add(half).ok_or(NumericError::Overflow)? / SCALE;
    i64::try_from(rounded).map_err(|_| NumericError::Overflow)
}
```

## 2. 错误分层模型与因果链留存（`err-*`）

### 2.1 三个正交判断维度与极简返回形态
对外错误响应（稳定机器码与脱敏提示）、内部可观测诊断（具名强类型与完整 `Error::source()` 因果链）与自动恢复决策（暂时故障强类型证据＋固定输入＋有效租约＋剩余预算）必须**相互独立判断**：绝不凭外部 HTTP 状态码反推是否重试，也不将底层 SQL／路径细节直接序列化给客户端。

| 场景条件 | 标准返回类型 | 禁止的反模式 |
| --- | --- | --- |
| 纯计算在全部合法输入下均成功 | 直接返回 `T` | 套一层永不返回 `Err` 的 `Result<T, XxxError>` |
| 按键单查未命中属正常业务分支 | `Option<T>`（含存储故障可能时用 `Result<Option<T>, E>`） | 把常规未命中抛成 `NotFound` 错误再让上层匹配忽略 |
| 调用方需按失败类别分支处理 | `Result<T, E>`（`E` 为具名 `thiserror` 枚举） | 返回 `String`、`()`、`Box<dyn Error>` 或全仓单一巨型错误枚举 |
| 实现标准转换 Trait 但逻辑必成功 | `type Error = std::convert::Infallible` | 造空错误类型或调用 `unwrap` |

- **禁用越权 `anyhow` 与生产 `unwrap`**：所有库 crate 及服务端请求协议处理层**严禁引入或返回 `anyhow::Error`**（`anyhow` 仅限进程 `main.rs` 启动引导与 CLI 顶层报告）。生产代码强制 `clippy::unwrap_used = "deny"`，严禁对任何外部输入、前置状态或 I/O 结果调用 `unwrap`、`expect`、`panic!` 或用 `catch_unwind` 伪造业务容错（仅局部代码已静态保证的不变量允许附清晰理由调用 `expect`）。

### 2.2 三层 `thiserror` 与 `#[source]` 留存规范
1. **纯领域层错误**：仅表达值域违规、守恒失败、非法状态跃迁等纯规则拒绝，派生 `thiserror::Error`，**保持零数据库／运行时依赖**。
2. **存储与适配层错误**：携带静态表名／动作位置，并通过 `#[source]`（或无歧义 `#[from]`）保留原始驱动、编解码或领域错误实例；**严禁在构造错误或分类前调用 `.to_string()` 擦除强类型根因**。严格区分客户端输入违规、并发／唯一约束冲突（结合精确 SQLSTATE 如 `23505`、`40001`、`40P01` 与约束名判定，不靠模糊匹配报错文本）与已持久化数据损坏。
3. **公开用例出口边界错误**：在公开用例边界按引用匹配分类（`400`/`422` 参数与规则违规、`401`/`403` 认证权限、`404` 未命中或安全隐匿、`409` 版本／幂等／状态冲突、`503` 暂时过载、`500` 存储损坏或内部故障），同时将完整原强类型错误挂入边界错误的 `source` 链。

```rust
// Good: 按引用分类映射对外错误码，同时保留完整强类型原错误作为 source
pub fn map_publish_error(err: PublishError) -> BoundaryError {
    match &err {
        PublishError::VersionConflict { .. } => {
            BoundaryError::conflict("VERSION_CONFLICT", "目标记录已被并发更新，请刷新后重试")
                .with_source(err)
        }
        PublishError::UnbalancedEntries { .. } => {
            BoundaryError::unprocessable("UNBALANCED_ENTRIES", "明细借贷总额不守恒")
                .with_source(err)
        }
        PublishError::Storage(_) | PublishError::CorruptRecord { .. } => {
            BoundaryError::internal().with_source(err)
        }
    }
}
```

## 3. 边界单点日志、脱敏与后台非盲目重试（`obs-*`）

### 3.1 边界单点结构化日志与强制脱敏
- **拒绝沿调用栈重复打日志**：内部函数只向上补充类型化错误上下文，禁止每层 `map_err` 都打一遍 `tracing::error!` 造成单次故障刷屏；统一在协议响应出口或后台作业单次尝试结束处记录一次（预期业务拒绝／并发冲突用 `DEBUG`／`WARN`，内部故障与数据损坏用 `ERROR`）。
- **结构化键值与脱敏红线**：统一使用 `tracing` 结构化字段（如 `tracing::warn!(%request_id, %job_id, attempt, "...")`）。展开 `source()` 链前必须脱敏——**严禁将口令、私钥、令牌、连接串、原始大报文或敏感明细写入日志流**（数据库报错可能回显绑定参数，无法确认安全时只记录错误类别、SQLSTATE、静态操作点与追踪 ID）。
- **异步追踪上下文与 `skip_all` 护栏**：多线程异步任务在 `.await` 挂起点可能跨工作线程迁移，**严禁使用 `std::thread_local!` 跨 `.await` 传递请求或作业上下文**（统一显式传参，或在请求／作业入口挂载 `tracing` Span／`tokio::task_local!`）。使用 `#[tracing::instrument]` 时**必须显式声明 `skip_all`** 并仅按白名单记录安全标量标识（如 `#[tracing::instrument(skip_all, fields(job_id = %id))]`），严禁默认自动 `Debug` 格式化全部函数实参。

### 3.2 后台异步任务非盲目重试规约
自动退避重试必须**同时满足四项硬前提**：① 从 `source` 链确认强类型暂时故障证据、② 锁定原任务不可变输入快照（严禁重跑时改读当前新数据）、③ 当前执行者租约仍然有效、④ 剩余尝试预算大于零，并在带随机抖动的有界退避后**完整重放业务用例**（严禁只重跑半截失败 SQL）：

| 故障根因类别 | 标准恢复动作 |
| --- | --- |
| 已证实暂时故障（死锁 `40P01`、可串行化失败 `40001`、短暂连接闪断、瞬时配额满） | 核验租约与尝试预算后，按原固定输入完整重放用例 |
| 前置状态头、版本或依赖资格已在外部演进 | 重新加载当前状态并重走纯规则判定，严禁强行落库过期计算结果 |
| 同一幂等键被绑定至不同请求摘要 | 立即以幂等冲突终止，严禁重试 |
| 提交结果未知（提交阶段超时或连接断开） | 先按确切作业 ID 与幂等台账查询已落库事实；严禁更换幂等键盲目重发 |
| 存储数据损坏、必填证据缺失或确定性规则拒绝 | 立即终止自动重试并标记终态诊断，交由受控更正流程处理 |
| 取消信号触发、租约失守或预算耗尽 | 立即中止执行，终局写事务严禁接受失守任务的产出 |
