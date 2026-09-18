---
name: golang-base
description: >-
  地道 Go 语言核心开发规范技能。以现代 Go 1.20+ 为演进基调，具备完备的 go.mod 版本自适应决策机制。
  涵盖时间格式化标准与常量定义规范、常量与 iota 枚举设计铁律 (防零值歧义、杜绝魔法值)、
  Channel 所有权原则、Goroutine 泄漏防范、errgroup 并发协同、错误包装 (fmt.Errorf %w)、
  Context 生命周期传递铁律、指针使用与内存语义黄金规范 (Uber Go 接收者一致性、禁止切片/Map/接口指针、
  typed nil 陷阱防护、不可变值对象) 及切片容量预分配。
---

# 地道 Go 语言核心开发规范技能 (Idiomatic Go Mastery Skill)

## 概述 (Overview)

本技能定义了在编写**通用 Go 语言基础库、并发组件、底层工具包或核心业务代码**时的现代工程规范与地道惯用法（Idiomatic Go）。深刻践行 **“现代基调先行，环境自适应决策”** 的务实工程哲学，吸收 **Uber Go Style Guide**、Go 官方 **CodeReviewComments** 与高性能并发内存语义实践。

---

# 1. 架构基调与版本自适应决策 (Version Baseline & Adaptive Matrix)

### 1.1 演进基调 (Modern Baseline)
在新立项或现代环境中，**统一以 Go 1.20+ / 1.22+ 作为演进基调**，充分利用原生泛型、`any` 别名、性能逃逸优化与新循环变量作用域。

### 1.2 go.mod 版本自适应决策表
进入项目后，优先检查 `go.mod` 文件中的 `go 1.xx` 声明：

| 语法 / 特性维度 | 推荐现代基调 (Go >= 1.22) | 历史版本自适应兼容 (Go 1.18 ~ 1.21) | 早期版本兼容 (Go < 1.18) |
| :--- | :--- | :--- | :--- |
| **空接口类型** | 统一采用 `any` | 统一采用 `any` | 降级为 `interface{}` |
| **泛型与泛型约束** | 完整支持泛型与 `comparable` | 完整支持泛型 | 严禁使用泛型，改用接口或具体类型 |
| **循环变量闭包捕获** | 原生每个迭代产生独立实例，可直接在 Goroutine 内部引用 | **强制显式局部变量拷贝**：`item := item` 避免竞态与数据覆盖 | **强制显式局部变量拷贝**：`item := item` |
| **整型 Range 语法** | 支持 `for i := range 10` | 必须使用传统 `for i := 0; i < 10; i++` | 必须使用传统循环形式 |
| **内置时间格式常量** | 原生 `time.DateTime`, `time.DateOnly`, `time.TimeOnly` (1.20+) | 1.18/1.19 无内置 `time.DateTime`，必须从项目自定义常量引入 | 必须从项目自定义常量引入 |

---

# 2. 时间格式化与时间常量规范 (Time Formatting & Constants)

Go 语言的时间格式化机制与 C/Java/Python 完全不同。Go 不使用 `%Y-%m-%d` 或 `yyyy-MM-dd`，而是使用**特定的基准参考时间 (Mon Jan 2 15:04:05 MST 2006，助记：`1 2 3 4 5 6 -0700`)**。

### 2.1 优先使用 `time` 标准库内置常量
在现代 Go (1.20+) 中，标准库已原生提供最常用格式，**严禁手写字面量**：
- **完整日期时间**：优先使用 `time.DateTime`（即 `"2006-01-02 15:04:05"`）；
- **纯日期**：优先使用 `time.DateOnly`（即 `"2006-01-02"`）；
- **纯时间**：优先使用 `time.TimeOnly`（即 `"15:04:05"`）；
- **网络与标准协议传输**：强制使用 `time.RFC3339`（`"2006-01-02T15:04:05Z07:00"`）或带纳秒的 `time.RFC3339Nano`。

### 2.2 自定义时间格式强制常量化 (No Inline Magic Layout Strings)
当标准库没有满足特定业务协议要求的时间格式时（如紧凑流水号、带毫秒/微秒日志、斜杠日期），**必须以显式命名的常量形式统一收敛**，严禁在业务逻辑中散落硬编码字符串：

```go
// ✅ 统一收敛在公共包 (如 pkg/timeutil/ 或领域常量包) 中
const (
    // LayoutCompactDateTime 紧凑时间戳 (常用于订单号或导出流水号)
    LayoutCompactDateTime = "20060102150405"
    // LayoutCompactDate 紧凑日期
    LayoutCompactDate = "20060102"
    // LayoutSlashDateTime 斜杠分隔时间
    LayoutSlashDateTime = "2006/01/02 15:04:05"
    // LayoutMilliDateTime 毫秒精度时间 (常用于高精度日志与审计)
    LayoutMilliDateTime = "2006-01-02 15:04:05.000"
    // LayoutYearMonth 年月分表分区标识
    LayoutYearMonth = "200601"
)
```

### 2.3 严禁其他语言时间占位符混入 (Anti-Pattern)
- ❌ **致命反例**：写成 `t.Format("YYYY-MM-DD HH:mm:ss")` 或 `t.Format("%Y-%m-%d")`；
  - Go 不会报错，但会原样输出混乱字母或字面量，导致生成离奇的日期！
- ✅ **标准记忆口诀**：`1 2 3 4 5 6 7`
  - 月份：`01` 或 `1`
  - 日期：`02` 或 `2`
  - 小时：`15` (24小时制) 或 `03` (12小时制)
  - 分钟：`04`
  - 秒数：`05`
  - 年份：`2006` 或 `06`
  - 时区：`-07:00` 或 `MST`

---

# 3. 常量定义与 iota 枚举设计工程铁律 (Constants & iota Enum)

常量是系统稳定性的基石。必须彻底消除一切未命名的**魔法数字 (Magic Numbers)** 与**魔法字符串 (Magic Strings)**。

### 3.1 常量分组与语义化命名
- 相同业务维度的常量，必须使用 `const (...)` 块分组，并配以清晰注释；
- **命名规范**：遵循驼峰命名法（公开大驼峰 `PublicConst`，私有小驼峰 `privateConst`），严禁使用下划线风格。

```go
// ✅ 统一分组声明
const (
    DefaultPageSize = 20
    MaxPageSize     = 100
    MaxBatchLimit   = 500
)
```

### 3.2 iota 枚举设计两大铁律 (Enum Best Practices)

### 🚨 铁律一：必须预留首行 `0` 值为 `Unknown` / `None` (防零值歧义)
- **踩坑根因**：Go 语言所有未显式赋值的变量默认具有零值（数值即为 `0`）。如果将业务含义上的有效状态（如 `StatusActive`）赋值为 `0`，那么一个**因反序列化遗漏或未初始化的结构体实例，会被系统错误地识别为合法状态**！
- ❌ **危险写法**：
  ```go
  const (
      StatusActive   = iota // 0 (致命隐患：未初始化的变量默认等于有效状态！)
      StatusDisabled        // 1
  )
  ```
- ✅ **标准工业级写法**：
  ```go
  type AccountStatus uint8

  const (
      AccountStatusUnknown  AccountStatus = iota // 0 (预留未初始化状态)
      AccountStatusActive                        // 1
      AccountStatusDisabled                      // 2
      AccountStatusFrozen                        // 3
  )
  ```

### 🚨 铁律二：位掩码 (Bitmask) 使用位移操作
- 当设计多重权限或复合开关标识时，利用 `1 << iota` 构建幂次位掩码：
  ```go
  type Permission uint32

  const (
      PermRead   Permission = 1 << iota // 1 (1 << 0)
      PermWrite                         // 2 (1 << 1)
      PermExec                          // 4 (1 << 2)
      PermAdmin                         // 8 (1 << 3)
  )
  ```

---

# 4. 指针使用与内存语义黄金规范 (Pointers & Memory Semantics)

指针是 Go 语言中最强大也最容易被滥用或误用的机制。编写 Go 代码必须牢固树立**值语义 (Value Semantics) 与指针语义 (Pointer Semantics)** 的精准心智模型：

### 4.1 核心心智模型：值语义 vs 指针语义
- **值语义 (Value)**：关注数据的独立性与不可变性。在函数调用时发生浅拷贝，变量完全隔离、天然并发安全；**数据优先在栈 (Stack) 上分配，函数退出时直接销毁，对 GC 产生零负担**。
- **指针语义 (Pointer)**：关注数据的共享性与身份唯一性 (Identity)。多个调用方共享底层同一块内存；**极易引发编译器逃逸分析 (Escape Analysis) 将变量逃逸到堆 (Heap)，增加 GC 标记与扫描停顿压力**。
- 💡 **性能箴言**：**不要迷信“传指针一定比传值快”！** 对于 64 字节以内的小结构体，传值（寄存器或栈拷贝）的吞吐量与 CPU 缓存局部性往往远高于指针逃逸到堆上产生的综合开销。

### 4.2 必须/推荐使用指针的场景 (When to Use Pointers)
1. **需要修改内部状态 (State Mutation)**：如果函数或方法必须变更入参结构体的字段值，必须传递指针。
2. **结构体包含不可复制的同步原语或字段 (Non-Copyable Fields)**：
   - 结构体若包含 `sync.Mutex`、`sync.RWMutex`、`sync.WaitGroup`、`sync.Cond` 或 `atomic.Value`：
     - **严禁传值拷贝**（拷贝会导致锁副本失效，形同虚设）；
     - **其所有方法接收者必须强制使用指针接收者 `(m *MyStruct)`**。
3. **大型结构体传参 (Large Structs > 64 字节)**：包含多字段的实体模型（如数据库 ORM Model、包含大数组的结构体），传指针避免大内存拷贝开销。
4. **表达“可选 / 缺失 / Nullable”业务语义 (Optional Fields)**：
   - JSON 序列化/反序列化（需区分“字段传了零值 `0`/`""`”与“前端根本没传该字段 (`nil`)”时，定义为 `*int` / `*string`）；
   - SQL 中的 Nullable 列映射。
5. **工厂方法与构造函数 (Constructors)**：
   - 复杂的领域服务、配置中心、持有一组依赖的上下文结构体，构造函数统一返回指针：`func NewService(cfg *Config) (*Service, error)`。

### 4.3 绝对禁止使用指针的四大反常识禁区 (Forbidden Anti-Patterns)
1. ❌ **严禁对内置引用类型多重套指针**：
   - 严禁 `*[]T`（切片指针，直接传 `[]T`）；
   - 严禁 `*map[K]V`（直接传 `map`）；
   - 严禁 `*chan T`（直接传 `chan`）；
   - 严禁 `*any` / `*interface{}`（直接传接口）。
2. ❌ **严禁对轻量不可变值对象使用指针**：
   - 官方明确指出：**`time.Time` 是值对象，严禁将其作为指针传递**（除结构体字段的可选反序列化外）；
   - 简单坐标 `Point{X, Y float64}` 等数值小对象传值。
3. ❌ **臭名昭著的“带类型的 nil 接口”陷阱 (Typed Nil Trap)**：
   - 返回 `error` 或任何接口时，**如果无错误，必须显式书写 `return nil`，绝对不能返回一个值为 nil 的具体类型指针变量**（会导致 `err != nil` 永远为 `true`！）。
4. ❌ **并发协程中的指针共享竞态 (Pointer Race in Goroutines)**：
   - 当把一个指针传入异步 Goroutine 时，若外部循环仍在迭代改变该指针，将引发数据脏写；启动前必须进行值拷贝。

### 4.4 方法接收者一致性原则 (Uber Go Receiver Consistency)
- 一个类型的方法集，**尽量保持接收者类型统一**（要么全值，要么全指针）；
- 只要有任意一个方法修改状态或持有 Mutex，该类型**所有方法统一使用指针接收者**。

### 4.5 防御性空指针检查与安全解引用 (Safe Dereferencing)
1. 公开 API 的指针入参，首行必须显式防御判空 `if ptr == nil`；
2. 严禁危险的盲目链式解引用，提供安全 Getter 或逐层判空。

---

# 5. 并发原语与协程安全（核心红线）

### 🚨 铁律一：Channel 所有权与生命周期管理
- **关闭权铁律**：**只有 Channel 的创建者/生产者才有权关闭通道**；接收者绝对禁止调用 `close(ch)`；
- **防向已关闭通道发送**：向已关闭的 channel 发送数据会触发不可恢复的 `panic: send on closed channel`；
- **读取状态判空**：从通道消费时，必须使用双返回值判断通道是否已关闭：`v, ok := <-ch`。

### 🚨 铁律二：终结 Goroutine 泄漏 (Prevent Goroutine Leaks)
- 严禁启动一个没有明确退出机制的“野生”Goroutine；
- 并发子任务调度必须使用 `golang.org/x/sync/errgroup` 或带有退出的 `select`。

### 🚨 铁律三：锁拷贝红线 (No Copying Mutex)
- `sync.Mutex` 和 `sync.RWMutex` **绝对禁止传值拷贝**；
- 包含 Mutex 的结构体方法，**接收者必须强制使用指针接收者 `(m *MyStruct)`**。

---

# 6. 地道错误处理闭环 (Error Handling)

- 传递错误时必须附带当前层级的上下文信息，使用 `%w` 动词包装原始错误：`fmt.Errorf("...: %w", err)`；
- 判定错误类型使用 `errors.Is` 代替 `==`；提取自定义错误使用 `errors.As` 代替类型断言；
- **绝对禁止使用 `_` 裸忽略非空 error**。

---

# 7. Context 上下文传递铁律

1. **第一参数原则**：`ctx context.Context` 必须显式作为所有 I/O、RPC、并发函数的**第一个形参**；
2. **禁止结构体存储**：**严禁将 `Context` 保存在结构体内部作为字段**；
3. **及时 Cancel**：派生子上下文时，必须紧跟 `defer cancel()` 防止内存泄漏。

---

# 8. 内存与切片性能优化

1. **切片容量预分配**：已知元素数量时，必须显式 `make([]T, 0, cap)`，彻底避免多次扩容复制；
2. **字符串批量拼接**：循环大批量拼接中，强制使用 `strings.Builder`，严禁使用 `+`；
3. **Map 容量初始化**：预估规模时强制 `make(map[K]V, hint)`。

---

# 9. Go 基础开发审查 Checklist

在编写与提交 Go 代码前，严格执行以下自审：
- [ ] **时间格式化规范**：是否优先使用 `time.DateTime` 等标准库常量？自定义格式是否已统一抽取为常量？是否杜绝了 `%Y-%m-%d` 等外语言占位符？
- [ ] **常量与枚举设计**：是否消除了所有魔法数字与字符串？`iota` 枚举是否将 `0` 预留为 `Unknown` 避免了零值歧义？
- [ ] **指针选型规范**：
  - [ ] 是否坚决杜绝了 `*[]T`、`*map`、`*chan`、`*any` 等多重引用指针？
  - [ ] `time.Time` 是否以值形式传递，没有误写成 `*time.Time`？
  - [ ] 是否在返回 `error` 接口时显式书写 `return nil`，彻底规避了 **typed nil trap**？
  - [ ] 包含 `sync.Mutex` 的结构体方法是否全部采用了指针接收者？
- [ ] **安全判空**：公开函数中暴露的指针入参，是否在首行进行了 `if ptr == nil` 防御？
- [ ] **并发安全**：异步 Goroutine 传入指针参数时，是否已规避外部循环并发脏写？
- [ ] **Context 规范**：`ctx context.Context` 是否稳居第一参数，且未被私自存入结构体？
- [ ] **错误因果链**：是否全部使用 `fmt.Errorf("%w", err)` 保留底层堆栈与因果？
