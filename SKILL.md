---
name: golang-base
description: >-
  地道 Go 语言核心开发规范技能。以现代 Go 1.20+ 为演进基调，具备完备的 go.mod 版本自适应决策机制。
  涵盖 Channel 所有权原则、Goroutine 泄漏防范、errgroup 并发协同、错误包装 (fmt.Errorf %w)、
  Context 生命周期传递铁律、指针使用与内存语义黄金规范 (Uber Go 接收者一致性、禁止切片/Map/接口指针、
  typed nil 陷阱防护、不可变值对象)、切片容量预分配及旧版闭包与类型降级兼容。
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

---

# 2. 指针使用与内存语义黄金规范 (Pointers & Memory Semantics)

指针是 Go 语言中最强大也最容易被滥用或误用的机制。编写 Go 代码必须牢固树立**值语义 (Value Semantics) 与指针语义 (Pointer Semantics)** 的精准心智模型：

### 2.1 核心心智模型：值语义 vs 指针语义
- **值语义 (Value)**：关注数据的独立性与不可变性。在函数调用时发生浅拷贝，变量完全隔离、天然并发安全；**数据优先在栈 (Stack) 上分配，函数退出时直接销毁，对 GC 产生零负担**。
- **指针语义 (Pointer)**：关注数据的共享性与身份唯一性 (Identity)。多个调用方共享底层同一块内存；**极易引发编译器逃逸分析 (Escape Analysis) 将变量逃逸到堆 (Heap)，增加 GC 标记与扫描停顿压力**。
- 💡 **性能箴言**：**不要迷信“传指针一定比传值快”！** 对于 64 字节以内的小结构体，传值（寄存器或栈拷贝）的吞吐量与 CPU 缓存局部性往往远高于指针逃逸到堆上产生的综合开销。

---

### 2.2 必须/推荐使用指针的场景 (When to Use Pointers)

1. **需要修改内部状态 (State Mutation)**：
   - 如果函数或方法必须变更入参结构体的字段值，必须传递指针。
2. **结构体包含不可复制的同步原语或字段 (Non-Copyable Fields)**：
   - 结构体若包含 `sync.Mutex`、`sync.RWMutex`、`sync.WaitGroup`、`sync.Cond` 或 `atomic.Value`：
     - **严禁传值拷贝**（拷贝会导致锁副本失效，形同虚设）；
     - **其所有方法接收者必须强制使用指针接收者 `(m *MyStruct)`**。
3. **大型结构体传参 (Large Structs > 64 字节)**：
   - 包含多字段的实体模型（如数据库 ORM Model、包含大数组的结构体），传指针避免大内存拷贝开销。
4. **表达“可选 / 缺失 / Nullable”业务语义 (Optional Fields)**：
   - JSON 序列化/反序列化（需区分“字段传了零值 `0`/`""`”与“前端根本没传该字段 (`nil`)”时，定义为 `*int` / `*string`）；
   - SQL 中的 Nullable 列映射。
5. **工厂方法与构造函数 (Constructors)**：
   - 复杂的领域服务、配置中心、持有一组依赖的上下文结构体，构造函数统一返回指针：
     ```go
     func NewService(cfg *Config) (*Service, error)
     ```

---

### 2.3 绝对禁止使用指针的四大反常识禁区 (Forbidden Anti-Patterns)

### 🚨 禁区一：严禁对内置引用类型多重套指针
Go 的切片 (Slice)、字典 (Map)、通道 (Channel) 和接口 (Interface) 底层本身就是引用封装或指针：
- ❌ **严禁 `*[]T`（切片指针）**：切片自身仅占 24 字节 Header `(dataPtr, len, cap)`，传切片本身即可共享修改底层元素。除极罕见的底层 `append` 需要原地篡改调用方 Header 的算法外，日常业务代码传 `*[]T` 纯属画蛇添足；
- ❌ **严禁 `*map[K]V`（Map 指针）**：Map 底层是指向 `hmap` 的指针，直接传 `map[K]V` 即可传递引用；
- ❌ **严禁 `*chan T`（通道指针）**：Channel 底层是指向 `hchan` 的指针；
- ❌ **严禁 `*any` / `*interface{}`（接口指针）**：接口底层是由 `(type, value)` 组成的胖指针，传接口指针会导致类型断言失效并引发恶性 Bug。

### 🚨 禁区二：严禁对轻量不可变值对象使用指针
- ❌ **严禁传递 `*time.Time`**：
  - Go 官方设计原则明确指出：**`time.Time` 是值对象，严禁将其作为指针传递**（除结构体字段的可选 JSON 反序列化外）；
  - `time.Time` 内部包含单调时钟与挂钟时间，传值能保证时间的不可变性与线程安全。
- ❌ **轻量简单数值结构体**：如坐标 `Point{X, Y float64}`、简单的两三个整型/字符串包装对象。

### 🚨 禁区三：臭名昭著的“带类型的 nil 接口”陷阱 (Typed Nil Trap)
这是 Go 语言最经典的线上事故诱因！Go 接口判定 `if i == nil` 必须要求动态类型与动态值**同时为 nil**。
- ❌ **致命反例**：
  ```go
  type MyError struct{}
  func (e *MyError) Error() string { return "fail" }

  func RunTask() error {
      var err *MyError = nil // 具体类型的 nil 指针
      if condition {
          err = &MyError{}
      }
      return err // 💥 致命陷阱：返回后 error 接口内持有了类型 *MyError，值虽然是 nil，但 err == nil 永远为 FALSE！
  }

  // 调用方：
  if err := RunTask(); err != nil {
      // 永远会进这里！触发误报与误判！
  }
  ```
- ✅ **铁律防御**：**返回 `error` 或任何接口时，如果无错误，必须显式书写 `return nil`，绝对不能返回一个值为 nil 的具体类型指针变量！**
  ```go
  func RunTask() error {
      if condition {
          return &MyError{}
      }
      return nil // 必须显式 return nil
  }
  ```

### 🚨 禁区四：并发协程中的指针共享竞态 (Pointer Race in Goroutines)
- 传递指针即传递共享可变性。
- 当把一个指针传入异步 Goroutine 时，若主协程或后续代码依然在持有或修改该指针指向的内存（尤其在 `for` 循环中），将引发数据脏写或 Panic：
  ```go
  // ❌ 危险竞态：item 在下一次迭代被修改
  for _, item := range list {
      go worker(&item) 
  }
  // ✅ 安全做法：传值或在进入协程前执行浅/深拷贝
  for _, item := range list {
      itemCopy := item
      go worker(&itemCopy)
  }
  ```

---

### 2.4 方法接收者一致性原则 (Uber Go Receiver Consistency)

- **要么全值，要么全指针**：一个类型的方法集，**尽量保持接收者类型统一**。
- **强制指针接收者判定**：
  1. 只要有任意一个方法需要修改接收者的字段，**该类型的所有方法统一使用指针接收者**；
  2. 只要结构体内部包含 `sync.Mutex` 或类似同步原语，**该类型的所有方法统一使用指针接收者**；
- **接口断言安全**：指针接收者的方法集不属于值类型。如果混用，将导致 `T` 无法完整满足接口约束，只有 `*T` 能满足，造成隐性类型断言崩溃。

---

### 2.5 防御性空指针检查与安全解引用 (Safe Dereferencing)

1. **公开 API 入参判空**：所有接收指针参数的公开函数与方法，必须在第一行显式防御判空：
   ```go
   func ProcessOrder(order *Order) error {
       if order == nil {
           return errors.New("order cannot be nil")
       }
       // 业务逻辑
   }
   ```
2. **禁止危险的盲目链式解引用**：
   - 严禁 `req.User.Profile.Address.City` 这种“走钢丝”代码；
   - 必须逐层判空，或提供安全提取函数 `user.GetCity()` 防止产生 `panic: runtime error: invalid memory address or nil pointer dereference`。

---

# 3. 并发原语与协程安全（核心红线）

### 🚨 铁律一：Channel 所有权与生命周期管理
- **关闭权铁律**：**只有 Channel 的创建者/生产者才有权关闭通道**；接收者绝对禁止调用 `close(ch)`；
- **防向已关闭通道发送**：向已关闭的 channel 发送数据会触发不可恢复的 `panic: send on closed channel`；
- **读取状态判空**：从通道消费时，必须使用双返回值判断通道是否已关闭：
  ```go
  v, ok := <-ch
  if !ok {
      return // 通道已关闭，优雅退出
  }
  ```

### 🚨 铁律二：终结 Goroutine 泄漏 (Prevent Goroutine Leaks)
- 严禁启动一个没有明确退出机制的“野生”Goroutine；
- 并发子任务调度必须使用 `golang.org/x/sync/errgroup` 或带有退出的 `select`：
  ```go
  g, ctx := errgroup.WithContext(parentCtx)
  for _, item := range items {
      item := item // 兼容低版本 Go 循环变量捕获
      g.Go(func() error {
          return processItem(ctx, item)
      })
  }
  if err := g.Wait(); err != nil {
      return fmt.Errorf("batch processing failed: %w", err)
  }
  ```

### 🚨 铁律三：锁拷贝红线 (No Copying Mutex)
- `sync.Mutex` 和 `sync.RWMutex` **绝对禁止传值拷贝**；
- 包含 Mutex 的结构体方法，**接收者必须强制使用指针接收者 `(m *MyStruct)`**；
- 函数传参时，严禁以值形式传递带有锁的结构体。

---

# 4. 地道错误处理闭环 (Error Handling)

- 传递错误时必须附带当前层级的上下文信息，使用 `%w` 动词包装原始错误：
  ```go
  if err := db.Query(ctx); err != nil {
      return fmt.Errorf("query user record failed [id=%d]: %w", userID, err)
  }
  ```
- 判定错误类型使用 `errors.Is` 代替 `==`；提取自定义错误使用 `errors.As` 代替类型断言；
- **绝对禁止使用 `_` 裸忽略非空 error**。

---

# 5. Context 上下文传递铁律

1. **第一参数原则**：`ctx context.Context` 必须显式作为所有 I/O、RPC、并发函数的**第一个形参**；
2. **禁止结构体存储**：**严禁将 `Context` 保存在结构体内部作为字段**；
3. **及时 Cancel**：派生子上下文时，必须紧跟 `defer cancel()` 防止内存泄漏。

---

# 6. 内存与切片性能优化

1. **切片容量预分配 (Pre-allocate Slice Capacity)**：
   - 已知元素数量时，必须显式 `make([]T, 0, cap)`，彻底避免多次翻倍扩容引起的内存拷贝与 GC 碎片。
2. **字符串批量拼接**：
   - 在循环或大批量拼接中，强制使用 `strings.Builder`，严禁使用 `+` 产生大量中间临时字符串。
3. **Map 容量初始化**：
   - 能够预估规模的 Map，强制 `make(map[K]V, hint)` 规避哈希桶多次 Rehash 迁移。

---

# 7. Go 基础开发与指针审查 Checklist

在编写与提交 Go 代码前，严格执行以下代码自审：
- [ ] **版本自适应**：是否已检查 `go.mod` 并针对特定版本正确处置循环变量？
- [ ] **指针选型规范**：
  - [ ] 是否坚决杜绝了 `*[]T`、`*map`、`*chan`、`*any` 等多重引用指针？
  - [ ] `time.Time` 是否以值形式传递，没有误写成 `*time.Time`？
  - [ ] 是否在返回 `error` 接口时显式书写 `return nil`，彻底规避了 **typed nil trap**？
  - [ ] 包含 `sync.Mutex` 的结构体方法是否全部采用了指针接收者？
- [ ] **安全判空**：公开函数中暴露的指针入参，是否在首行进行了 `if ptr == nil` 防御？
- [ ] **并发安全**：异步 Goroutine 传入指针参数时，是否已规避外部循环并发脏写？
- [ ] **Context 规范**：`ctx context.Context` 是否稳居第一参数，且未被私自存入结构体？
- [ ] **错误因果链**：是否全部使用 `fmt.Errorf("%w", err)` 保留底层堆栈与因果？
