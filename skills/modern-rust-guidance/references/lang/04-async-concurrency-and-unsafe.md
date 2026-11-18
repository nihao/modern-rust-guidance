# Lang 04 — Future 状态机、并发原语与 Unsafe/FFI 健全性

本分册定义异步 Future 状态机机制、阻塞隔离、取消安全性、通道与原子内存序选型，以及 Rust 2024 `unsafe` 与跨语言 FFI 安全门禁。

## 1. 异步状态机机制与取消安全（`async-*`）

### 1.1 控制跨 `.await` 的 Future 状态机体积
编译器会将 `async fn` 转换为保存跨挂起点局部变量的枚举状态机：
- **禁止大栈缓冲跨越 `.await`**：若在 `.await` 之前声明大栈数组（如 `[0_u8; 65536]`）并在挂起点后继续引用，整个外层 Future 的体积将暴增数十 KB，拖累每次任务轮询与栈拷贝。大临时变量应在 `.await` 前通过独立 `{ ... }` 块提前析构，或改用堆缓冲 `Vec<u8>`。
- **严禁阻塞异步执行器线程**：密码学重度散列、压缩归档、大型文档编解码或单次耗时超过 `10–100 µs` 的密集 CPU 计算，必须卸载至 `tokio::task::spawn_blocking` 或独立计算池，严禁在异步工作线程内裸跑同步重计算或阻塞系统调用。

### 1.2 锁守卫生命周期、具名绑定与无失败 `Drop`
- **同步锁严禁跨越 `.await`**：强制启用 `clippy::await_holding_lock = "deny"`。`std::sync::MutexGuard`／`RwLockReadGuard`／`RwLockWriteGuard` 必须在进入 `.await` 前于局部块显式释放（否则直接死锁工作线程）：

```rust
// Good: 在独立作用域内完成同步锁读写并立即释放守卫，随后再执行 .await
let snapshot = {
    let mut state = shared.lock().expect("state mutex poisoned");
    state.advance_cursor()
};
persist_cursor(snapshot).await?;
```

- **仅跨异步 I/O 串行化时使用异步锁**：纯内存短临界区优先用同步锁；确需跨 `.await` 独占资源时才使用 `tokio::sync::Mutex`，且严禁持锁调用外部未知回调。
- **RAII 守卫强制具名绑定**：锁守卫、信号量许可与作用域清理器必须绑定具名变量（如 `let _permit = sem.acquire().await?;`），**严禁写 `let _ = lock.lock();`（右值在分号处立即析构，临界区完全裸奔）**。`Drop` 实现只做确定性内存与计数释放，严禁在 `Drop` 内执行可失败的事务提交、阻塞网络 I/O 或主动 `panic!`。

### 1.3 `tokio::select!` 取消安全性（Cancel Safety）
- `select!` 中任一分支就绪会立即 `drop` 其余未完成分支的 Future。对含多步写入、半程流读取或状态变更的非取消安全异步调用，**严禁在循环内每轮重新构造 Future 挂入 `select!`**；必须使用 `tokio::pin!` 固定单次 Future 句柄或轮询 `&mut JoinHandle` 跨轮次续接。
- 彼此独立且不共享独占句柄的异步读取可用 `tokio::try_join!` 并发收敛，任一分支出错立即短路取消其余分支。

## 2. 并发原语与原子内存序选型（`conc-*`）

### 2.1 有界异步通道选型矩阵（拒绝无界堆积）
默认一律使用**有界通道**建立显式背压，**严禁在生产路径默认使用 `unbounded_channel`** 任由上游突发流量耗尽内存：

| 协作拓扑 | 标准原语 | 边界约束 |
| --- | --- | --- |
| 多生产者 → 单消费者工作队列 | `tokio::sync::mpsc::channel(cap)` | 显式规划容量上限与满载拒绝／退避策略 |
| 单次异步计算结果回传／停机通知 | `tokio::sync::oneshot` | 发送端被提前丢弃时接收端必须显式处理取消错误 |
| 配置热快照／最新状态单值广播 | `tokio::sync::watch` | 仅保证消费者看到最新状态，不作逐条事件日志使用 |
| 多消费者实时事件扇出 | `tokio::sync::broadcast::channel(cap)` | 消费者必须显式处理慢消费滞后 `RecvError::Lagged(n)` |

### 2.2 原子类型内存序（Atomic Ordering）与锁边界
- **按真实同步语义选内存序**：
  - 纯统计计数或独立标志（不充当其他内存读写的同步屏障） → `Ordering::Relaxed`。
  - 跨线程发布—订阅配对（线程 A 写妥数据后置就绪位，线程 B 读到就绪位后安全读取数据） → 写端 `Release`、读端 `Acquire`、读改写 `AcqRel`。
  - 禁止不加分析全篇盲目写 `Ordering::SeqCst` 掩盖并发模型缺陷。
- **多字段耦合不变量直接用锁**：一旦不变量横跨两个及以上字段（如“余额 ＋ 版本号”或“容量 ＋ 队列头尾”），必须用单一 `Mutex`／`RwLock` 原子保护，**严禁拼凑多个独立 `Atomic*` 制造中间不一致观测窗口**。

## 3. Rust 2024 Unsafe 与 FFI 安全门禁（`unsafe-*`）

工作区应强制启用 `unsafe_op_in_unsafe_fn = "deny"`、`clippy::undocumented_unsafe_blocks = "deny"` 与 `clippy::missing_safety_doc = "deny"`（纯业务模块优先配置 `unsafe_code = "forbid"`）：

1. **最小模块安全边界**：仅在系统 FFI 绑定或经严格实测证明安全 Rust 无法表达的底层原语中使用 `unsafe`。将其隔离在最小私有子模块内并对外提供 100% 安全的封装接口——注意：**只要同模块内的安全函数可以随意读写影响内存安全的前置私有字段，整个模块都属于安全审计边界**。
2. **Rust 2024 显式块与双向证明**：`unsafe fn` 必须在文档注释 `# Safety` 小节逐条声明调用方必须满足的前置不变量，且函数体内部的不安全调用仍须用最小范围 `unsafe { ... }` 显式包裹；每一个 `unsafe` 块与 `unsafe impl` 正上方必须紧邻 `// SAFETY:` 注释逐条证明前置条件在此处成立。
3. **未初始化内存、别名与 ABI 铁律**：
   - 操作未初始化内存必须使用 `std::mem::MaybeUninit<T>` 与原始指针方法（`ptr::write`、`addr_of_mut!`），**严禁对未初始化、悬垂或未对齐地址构造 `&T`／`&mut T` 引用（即使不解引用亦立即触发未定义行为）**。
   - 严禁违背独占别名规则让可变引用与任何其他活跃引用指向重叠内存；严禁用 `std::mem::transmute` 延长生命周期或强转无明确 `#[repr(C)]`／`#[repr(transparent)]` 保证的类型。
   - 跨语言 FFI 入口必须显式校验空指针、对齐、缓冲区上限与内存释放配对归属，并**严禁让 Rust panic 跨越非 `-unwind` 外部 ABI 栈帧展开**（边界处用 `std::panic::catch_unwind` 拦截并转化为错误码）。
