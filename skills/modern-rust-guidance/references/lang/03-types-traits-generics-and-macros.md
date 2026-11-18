# Lang 03 — 类型系统、Trait 与泛型机制、闭包与卫生宏

本分册定义类型驱动不变量建模、Trait 与泛型分发机制、标准转换契约、闭包捕获、编译期求值及声明宏卫生规则。

## 1. 类型驱动不变量建模（`type-*`）

### 1.1 Parse, Don't Validate（受检构造与私有字段）
- **构造即收敛不变量**：承载格式、非空或值域约束的类型必须保持**字段私有**，仅通过返回 `Result<Self, Error>` 的构造器或 `TryFrom` 暴露受检入口；解析函数直接返回强类型新值（如 `CurrencyCode::new(raw) -> Result<CurrencyCode, ...>`），避免返回 `Result<(), E>` 后继续在调用链裸传原始字符串或整数。
- **元组 Newtype 隔离同底层原语**：同一底层类型承载不同语义身份或物理量纲时（如 `SourceNodeId(Uuid)` vs `TargetNodeId(Uuid)`、`BasisPoints(i32)` vs `PartsPerMillion(i64)`），必须定义独立元组 Newtype，不可仅用 `type` 类型别名（别名无法阻止实参顺序颠倒）。

```rust
// Good: 栈上定长 ASCII 码 + 私有字段构造一次终生有效
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct CurrencyCode([u8; 3]);

impl CurrencyCode {
    pub fn new(raw: &str) -> Result<Self, CurrencyCodeError> {
        let Ok(bytes) = <[u8; 3]>::try_from(raw.trim().as_bytes()) else {
            return Err(CurrencyCodeError::InvalidLength);
        };
        if !bytes.iter().all(u8::is_ascii_uppercase) {
            return Err(CurrencyCodeError::NotAsciiUppercase);
        }
        Ok(Self(bytes))
    }

    pub fn as_str(&self) -> &str {
        std::str::from_utf8(&self.0).expect("CurrencyCode guarantees valid ASCII")
    }
}
```

### 1.2 互斥状态枚举、具名二元类型与克制 Typestate
- **带载荷枚举消除非法组合**：互斥生命周期阶段及各阶段专属字段必须建模为 `enum` 变体载荷，根除“当 `status == Failed` 时 `error_code` 必为 `Some`”的脆弱运行期约定。
- **具名双变体枚举替代模糊 `bool`**：调用点字面量 `true`／`false` 无法自解释含义时（如 `sync(data, true, false)`），改用具名双变体枚举（如 `FlushPolicy::Immediate`）。
- **局部短周期 Typestate**：利用消费 `self` 或 `PhantomData<Stage>` 的编译期状态转换，仅用于高价值短生命周期协议（如“未封口缓冲 → 封口校验缓冲”），禁止将持久化记录状态机械拆成十余个泛型结构体。

## 2. Trait 设计、转换契约与泛型单态化控制（`trait-*` / `conv-*`）

### 2.1 关联类型 vs 泛型参数与标准转换契约
- **关联类型 vs 泛型参数**：实现类型到目标类型为 **1 对 1** 关系时使用关联类型 `type Output;`（消除调用侧多余泛型标注）；同一类型需对多种输入／输出分别实现（**1 对多**）时使用泛型参数 `trait From<T>`。
- **标准转换 Trait 红线**：无损值转换实现 `From<T>`，可失败值转换实现 `TryFrom<T>`（**严禁直接手写 `Into` 或 `TryInto`**，统一由标准库自动推导），文本解析实现 `std::str::FromStr`（配对 `Display` 时保证往返语义一致）。
- **严格区分 `AsRef<T>` 与 `Borrow<Q>`**：
  - `AsRef<T>`：仅表示零分配廉价借用视图提取，无等价与排序约束。
  - `Borrow<Q>`：**必须保证借用视图的 `Eq`、`Ord`、`Hash` 与拥有值完全一致**（如 `String: Borrow<str>` 供映射表按键查找）；若两者哈希或比较逻辑有丝毫差异，严禁实现 `Borrow`。

### 2.2 静态单态化、对象安全与薄泛型壳去膨胀
- **静态分发与封闭枚举优先**：默认用具体类型或窄泛型约束触发编译期内联；变体集合封闭的内部运行期多态（如固定几类存储或连接器后端）优先用具名 `enum` 分发，仅在开放异构集合、错误链擦除或第三方动态接口边界使用 `dyn Trait`（需保持 dyn 兼容的方法不得含方法级泛型参数、不得按值返回 `Self`）。`where` 约束只附加在真正需要该能力的 `impl` 块或方法上，**不在 `struct` 声明处冗余捆绑 Trait 约束**。
- **薄泛型入口转私有单态内核（Polymorphization）**：当函数为调用便利接收 `impl AsRef<Path>` 或 `impl Into<String>` 且函数体较大、又不在纳秒级热循环内时，用两行泛型外壳立即转为具体切片引用调用私有单态函数，避免按每种实参类型重复编译大段机器码：

```rust
// Good: 调用侧可传 &str / String / &Path / PathBuf，而大段实现仅单态化编译一次
pub fn read_schema(path: impl AsRef<std::path::Path>) -> Result<Schema, SchemaError> {
    read_schema_inner(path.as_ref())
}

fn read_schema_inner(path: &std::path::Path) -> Result<Schema, SchemaError> {
    let raw = std::fs::read_to_string(path)?;
    Schema::parse(&raw)
}
```

### 2.3 标准库核心 Trait 实现红线
1. **`Debug` 敏感数据强制脱敏**：密钥、口令、签名种子、访问令牌、连接串类型必须手写 `fmt::Debug` 输出 `"[REDACTED]"`，严禁 `#[derive(Debug)]`。
2. **`Eq`／`Ord`／`PartialOrd`／`Hash` 严格联动**：全序必须满足 `partial_cmp(a, b) == Some(cmp(a, b))` 且相等实例的 `Hash` 必相等；手写其中任一忽略附属字段时，必须同步手写全部四个 Trait，严禁与 `derive` 混搭。
3. **持久化摘要禁用 `DefaultHasher`**：标准库哈希器带随机种子且跨版本不固定；落盘或跨进程摘要必须使用确定性规范编码配加密散列。
4. **克制 `Default`、`Deref` 与标注 `#[must_use]`**：无天然零值／空集的类型（如实体 ID、版本号、币种）严禁虚构 `Default`；`Deref` 仅用于透明智能指针，严禁模拟类继承；忽略返回值即代表逻辑遗漏的纯算子与守卫类型必须加 `#[must_use]`（`Result` 返回值已自带，无需重标）。

## 3. 闭包、编译期 `const` 与卫生声明宏（`closure-*` / `const-*` / `macro-*`）

### 3.1 闭包约束与最小字段捕获
- **最宽调用能力约束**：接收同步回调参数时按最小必需能力声明：`FnOnce`（调用一次） > `FnMut`（多次可变调用） > `Fn`（并发或可重入只读调用）；返回闭包优先写 `-> impl Fn(...)` 消除堆分配。
- **精确字段捕获**：虽然 Rust 支持闭包自动按表达式路径拆分借用，但 `move` 闭包或对 `self` 调方法仍会捕获整个宿主对象。构造跨任务 `move` 闭包前，先在闭包外提取所需的标量字段或 `Arc::clone` 局部变量。

### 3.2 编译期求值（`const fn` 与内联 `const {}`）
- 无运行期依赖的纯常量换算、掩码推导与静态表构造优先声明为 `const fn`；纯值常量一律用 `const` 而非占据全局地址空间的 `static`。
- 对固定分块上限、参数倍数关系与结构尺寸预算，使用原生内联 **`const { assert!(CHUNK_SIZE > 0 && CHUNK_SIZE <= MAX_LIMIT); }`** 在编译阶段阻断配置错误。

### 3.3 宏克制、可检索性与声明宏卫生性
普通函数与泛型可表达的逻辑严禁写宏；工作区内部**禁止为消除普通样板自建过程宏（`proc-macro` crate）或拼接生成隐式标识符**，确保所有符号可被词法搜索、LSP 与静态 AST 守卫直接定位。确需局部 `macro_rules!` 消除重复语法样板时：
- 优先使用精确片段说明符（`:ident`、`:ty`、`:expr`、`:pat`、`:vis`），避免无必要地退化为难以诊断错误的 `:tt`；规则末尾统一支持可选尾随逗号 `$(,)?`。
- 宏体内跨模块引用一律使用 `$crate::` 绝对卫生路径；展开所需的内部辅助函数／模块必须标记 `#[doc(hidden)]`；**严禁在宏内部隐藏权限核验、加锁顺序、精确数值舍入或事务提交动作**。
