---
name: golang-base
description: >-
  地道 Go 语言核心开发规范技能。以现代 Go 1.20+ 为演进基调，具备完备的 go.mod 版本自适应决策机制。
  涵盖 Channel 所有权原则、Goroutine 泄漏防范、errgroup 并发协同、错误包装 (fmt.Errorf %w)、
  Context 生命周期传递铁律、指针 vs 值接收者选型、切片容量预分配及旧版闭包与类型降级兼容。
---

# 地道 Go 语言核心开发规范技能 (Idiomatic Go Mastery Skill)

## 概述 (Overview)

本技能定义了在编写**通用 Go 语言基础库、并发组件、底层工具包或核心业务代码**时的现代工程规范与地道惯用法（Idiomatic Go）。深刻践行 **“现代基调先行，环境自适应决策”** 的务实工程哲学，吸收 **Uber Go Style Guide** 与并发安全实践。

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

# 2. 并发原语与协程安全（核心红线）

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

# 3. 地道错误处理闭环 (Error Handling)

- 传递错误时必须附带当前层级的上下文信息，使用 `%w` 动词包装原始错误：
  ```go
  if err := db.Query(ctx); err != nil {
      return fmt.Errorf("query user record failed [id=%d]: %w", userID, err)
  }
  ```
- 判定错误类型使用 `errors.Is` 代替 `==`；提取自定义错误使用 `errors.As` 代替类型断言；
- **绝对禁止使用 `_` 裸忽略非空 error**。

---

# 4. Context 上下文传递铁律

1. **第一参数原则**：`ctx context.Context` 必须显式作为所有 I/O、RPC、并发函数的**第一个形参**；
2. **禁止结构体存储**：**严禁将 `Context` 保存在结构体内部作为字段**；
3. **及时 Cancel**：派生子上下文时，必须紧跟 `defer cancel()` 防止内存泄漏。

---

# 5. 内存与性能优化

- 接收者选型：修改内部状态或包含 Mutex 时强制指针接收者；不可变纯数值小对象使用值接收者；
- 切片容量预分配：已知数量时强制 `make([]T, 0, cap)` 消除动态扩容开销；
- 字符串批量拼接强制使用 `strings.Builder`。

---

# 6. Go 基础开发 Checklist

- [ ] 是否已检查 `go.mod` 并根据实际版本（如 < 1.22）安全处置循环变量拷贝？
- [ ] 函数入参中 `ctx context.Context` 是否位于第一参数位置并跟紧 `defer cancel()`？
- [ ] 带有 Mutex 的结构体方法是否全部采用了指针接收者？
- [ ] 错误返回是否全部使用 `fmt.Errorf("%w", err)` 正确保留了因果链？
