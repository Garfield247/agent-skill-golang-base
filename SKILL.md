---
name: golang-base
description: >-
  地道 Go 语言核心开发规范、Uber Go 风格实践、并发原语、错误包装与内存安全技能。
  涵盖 Channel 所有权原则、Goroutine 泄漏防范、errgroup 并发协同、错误包装 (fmt.Errorf %w)、
  Context 生命周期传递铁律、指针 vs 值接收者选型及切片内存预分配。
---

# 地道 Go 语言核心开发规范技能 (Idiomatic Go Mastery Skill)

## 概述 (Overview)

本技能定义了在编写**通用 Go 语言基础库、并发组件、底层工具包或非框架特定业务代码**时的现代工程规范与地道惯用法（Idiomatic Go）。深刻吸收 **Uber Go Style Guide**、**Effective Go** 与顶级开源项目的并发安全实践。

---

# 1. 并发原语与协程安全（核心红线）

### 🚨 铁律一：Channel 所有权与生命周期管理
- **关闭权铁律**：**只有 Channel 的创建者/生产者才有权关闭通道**；接收者绝对禁止调用 `close(ch)`；
- **防向已关闭通道发送**：向已关闭的 channel 发送数据会触发不可恢复的 `panic: send on closed channel`；
- **读取状态判空**：从通道消费时，必须使用双返回值判断通道是否已关闭：
  ```go
  v, ok := <-ch
  if !ok {
      // 通道已关闭，优雅退出
      return
  }
  ```

### 🚨 铁律二：终结 Goroutine 泄漏 (Prevent Goroutine Leaks)
- 严禁启动一个没有明确退出机制的“野生”Goroutine；
- 并发子任务调度必须使用 `golang.org/x/sync/errgroup` 或带有退出的 `select`：
  ```go
  // ✅ 推荐：使用 errgroup 协同管理并发子任务与生命周期
  g, ctx := errgroup.WithContext(parentCtx)
  for _, item := range items {
      item := item // Go 1.22 之前需防变量捕获陷阱
      g.Go(func() error {
          return processItem(ctx, item)
      })
  }
  if err := g.Wait(); err != nil {
      return fmt.Errorf("batch processing failed: %w", err)
  }
  ```

### 🚨 铁律三：锁拷贝红线 (No Copying Mutex)
- `sync.Mutex` 和 `sync.RWMutex` 本质是指针包装或含有状态的结构体，**绝对禁止传值拷贝**；
- 包含 Mutex 的结构体方法，**接收者必须强制使用指针接收者 `(m *MyStruct)`**；
- 函数传参时，严禁以值形式传递带有锁的结构体。

---

# 2. 地道错误处理闭环 (Idiomatic Error Handling)

### 2.1 错误包装与上下文追溯
- 传递错误时必须附带当前层级的上下文信息，使用 `%w` 动词包装原始错误：
  ```go
  // ✅ 正确：包装底层错误，保留因果调用链
  if err := db.Query(ctx); err != nil {
      return fmt.Errorf("query user record failed [id=%d]: %w", userID, err)
  }
  ```
- 严禁直接丢弃底层错误或直接返回裸字符串。

### 2.2 错误判断与断言标准
- 判定错误类型使用 `errors.Is` 代替 `==`（支持逐层解包匹配）：
  ```go
  if errors.Is(err, sql.ErrNoRows) {
      // 资源不存在分支
  }
  ```
- 提取自定义结构化错误使用 `errors.As` 代替类型断言：
  ```go
  var pathErr *os.PathError
  if errors.As(err, &pathErr) {
      log.Printf("Path error on: %s", pathErr.Path)
  }
  ```
- **绝对禁止使用 `_` 裸忽略非空 error**。

---

# 3. Context 上下文传递铁律

1. **第一参数原则**：`ctx context.Context` 必须显式作为所有 I/O、RPC、并发函数的**第一个形参**；
2. **禁止结构体存储**：**严禁将 `Context` 保存在结构体内部作为字段**，Context 是调用栈维度的瞬态变量，不可作为对象状态长期持有；
3. **及时 Cancel**：使用 `context.WithTimeout` 或 `context.WithCancel` 派生子上下文时，必须紧跟 `defer cancel()` 防止 Goroutine 和 Timer 内存泄漏：
   ```go
   ctx, cancel := context.WithTimeout(parentCtx, 3*time.Second)
   defer cancel() // 强制立即 defer，保证退出时释放定时器资源
   ```

---

# 4. 内存、性能与接收者选型 (Performance & Memory)

### 4.1 结构体接收者选型黄金法则
| 场景 | 推荐选型 | 核心考量 |
| :--- | :---: | :--- |
| **方法需要修改结构体内部状态** | **指针接收者 `(s *Service)`** | 值接收者修改的是拷贝副本，无法影响原对象。 |
| **结构体包含 Mutex 等同步原语** | **指针接收者 `(s *Service)`** | 禁止锁拷贝。 |
| **结构体字段较多或包含大数组** | **指针接收者 `(s *Service)`** | 避免每次方法调用产生堆栈内存全量拷贝。 |
| **纯不可变只读小结构体（如 Point, Time）** | 值接收者 `(p Point)` | 可安全传值，利于逃逸分析分配至栈上。 |

### 4.2 切片容量预分配 (Slice Pre-allocation)
在已知元素数量时，必须预先指定容量（Cap），避免多次倍数扩容导致频繁内存重新申请与数据搬迁：
```go
// ❌ 负例：触发多次扩容与内存重新分配
var list []User
for _, id := range ids {
    list = append(list, fetch(id))
}

// ✅ 正例：预分配底层数组容量
list := make([]User, 0, len(ids))
for _, id := range ids {
    list = append(list, fetch(id))
}
```

### 4.3 高效字符串拼接
在循环或大批量拼接字符串时，严禁使用 `+` 产生大量中间垃圾对象，统一采用 `strings.Builder`：
```go
var builder strings.Builder
builder.Grow(256) // 可选预分配
for _, s := range stringList {
    builder.WriteString(s)
}
result := builder.String()
```

---

# 5. 专属排障武器库 (Troubleshooting & Debugging)

- **竞态检测器 (Data Race Detector)**：
  ```bash
  go test -race ./...
  go run -race main.go
  ```
- **逃逸分析 (Escape Analysis)**：
  检查变量究竟分配在栈（Stack）还是逃逸到了堆（Heap）：
  ```bash
  go build -gcflags="-m -m" main.go
  ```
- **死锁检测**：
  Go 运行时具备主 Goroutine 死锁检测，若出现 `fatal error: all goroutines are asleep - deadlock!`，检查无缓冲 channel 是否在单协程内阻塞自读自写。

---

# 6. Go 基础开发 Checklist

- [ ] 函数入参中 `ctx context.Context` 是否位于第一参数位置？
- [ ] 派生子 context 是否紧跟了 `defer cancel()`？
- [ ] 是否存在 Channel 接收方误调用 `close` 的隐患？
- [ ] 带有 Mutex 的结构体方法是否全部采用了指针接收者？
- [ ] 错误返回是否全部使用 `fmt.Errorf("%w", err)` 正确保留了因果链？
- [ ] 切片初始化时是否已根据预期长度进行了容量预分配？
