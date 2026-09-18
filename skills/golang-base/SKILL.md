---
name: golang-base
description: >-
  地道 Go 语言核心开发规范技能。以现代 Go 1.20+ 为演进基调，具备完备的 go.mod 版本自适应决策机制。
  涵盖时间格式化标准与常量定义规范、常量与 iota 枚举设计铁律 (防零值歧义、杜绝魔法值)、
  代码封装原则、pkg 一级分类收紧与二级嵌套标准、核心工具包规范 (slicex/mapx/stringx/timex)、
  Channel 所有权原则、Goroutine 泄漏防范、errgroup 并发协同、错误包装 (fmt.Errorf %w)、
  Context 生命周期传递铁律、指针使用与内存语义黄金规范 (Uber Go 接收者一致性、禁止切片/Map/接口指针、
  typed nil 陷阱防护、不可变值对象) 及切片容量预分配。
---

# 地道 Go 语言核心开发规范技能 (Idiomatic Go Mastery Skill)

## 概述 (Overview)

本技能定义了在编写**通用 Go 语言基础库、并发组件、底层工具包、公共 pkg 层或核心业务代码**时的现代工程规范与地道惯用法（Idiomatic Go）。深刻践行 **“现代基调先行，环境自适应决策”** 的务实工程哲学，吸收 **Uber Go Style Guide**、Go 官方 **CodeReviewComments** 与高性能并发内存语义实践。

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

# 2. 代码封装原则 (Encapsulation Principles)

封装的本质是**隐藏内部复杂度与控制变化边界**，使调用方关注“做什么”而非“怎么做”。

### 2.1 最小知识原则 (迪米特法则 Law of Demeter)
- 严禁连续链式穿透调用内部字段（如 `order.User.Account.Wallet.Balance -= 100`）；
- 必须通过封装的意图方法操作（如 `err := order.DeductPayment(100)`）。

### 2.2 选项模式 (Functional Options Pattern)
- 当函数/构造函数入参超过 3 个或包含可选配置时，强制使用选项模式，保证零破坏向后兼容：
  ```go
  type Option func(*Server)
  func WithPort(port int) Option { return func(s *Server) { s.port = port } }
  func NewServer(addr string, opts ...Option) *Server
  ```

### 2.3 规避“原始类型偏执” (Avoid Primitive Obsession)
- 具有明确业务边界的概念（如金额、状态、电话），禁止使用裸 `string`/`int` 传递，提炼为强类型自定义类型或值对象。

### 2.4 内部自闭环并发安全 (Self-Contained Concurrency)
- 结构体内部若需持锁保持并发安全，必须在方法内部消化加锁与释放（`mu.Lock(); defer mu.Unlock()`），绝对禁止要求外部调用方手动加锁。

### 2.5 拒绝透传空壳封装 (No Pass-Through Wrappers)
- 业务链路从 Controller 到 Model 严格控制在 3~4 层内。若某层封装未带来隔离、转换或解耦价值，严禁充当无意义的透传转发空壳。

---

# 3. `pkg/` 基础层分层架构标准：一级分类收紧，二级按需嵌套

公共基础层 `pkg/` 是系统的地基。为防目录散乱失控与依赖污染，遵循“**一级分类极度收紧，二级视情况按需嵌套**”的核心法则。工具类命名统一采用**轻量扩展 `*x` 风格**（规避无间隔挤压拼接，杜绝与标准库冲突）：

```text
pkg/                                           # 全局公共基础设施与契约底座 (一级分类收敛在 6~7 个)
│
├── enums/                                     # 🚨【一级收紧：领域枚举】(二级不嵌套，按实体单层分文件，package enums)
│   ├── account_status.go
│   └── session_status.go
│
├── types/                                     # 🚨【一级收紧：通用实体】(二级不嵌套，单层多文件，package types)
│   ├── paging.go                              #     通用泛型分页请求与响应
│   └── token.go                               #     JWT 鉴权 Payload 实体
│
├── errorx/                                    # 🚨【一级收紧：统一错误】(二级不嵌套，错误码与因果包装)
│   ├── codes.go                               #     全局分段业务错误码
│   └── error.go                               #     CodeError 统一结构体
│
├── ctxdata/                                   # 🚨【一级收紧：上下文层】(二级不嵌套，私有类型 Context Key)
│   └── ctxdata.go                             #     安全 Getter/Setter
│
├── util/                                      # 🛠️【一级收紧：通用工具箱】(👉 二级视情况“必须嵌套独立包”，统一 *x 风格)
│   ├── slicex/                                #     🔪 切片扩展包：slicex.Chunk, Unique, Diff, SafeSubSlice
│   ├── mapx/                                  #     🗺️ 字典扩展包：mapx.KeyBy, GroupBy, Keys, GetOrDefault
│   ├── stringx/                               #     🔤 字符串扩展：stringx.SafeTruncate(UTF-8), MaskPhone
│   ├── timex/                                 #     ⏰ 时间扩展包：timex.StartOfDay, SafeParseTime, 格式常量
│   ├── convx/                                 #     🔄 类型转换包：convx.SafeParseInt, StringToSlice
│   ├── crypto/                                #     🔐 加解密包：crypto.AesEncrypt, HmacSha256
│   └── excel/                                 #     📊 报表处理包：excel.Export (隔离 excelize 重依赖)
│
├── storage/ (或 xdb/xredis)                   # 🗄️【一级收紧：存储基础设施】(二级按驱动嵌套 xdb/xredis)
│   ├── xdb/                                   #     数据库连接池与慢查拦截
│   └── xredis/                                #     Redis 客户端与 Lua 分布式锁
│
└── middleware/                                # 🛡️【一级收紧：通用拦截层】(二级不嵌套，按拦截能力独立文件)
    ├── jwt_auth.go
    ├── trace.go
    └── recovery.go
```

### 3.1 二级目录嵌套与 `*x` 命名决策原则
1. **单层不嵌套**：`enums`、`types`、`errorx`、`ctxdata`、`middleware`，业务内聚度高，同一个包名语义纯粹，二级不嵌套；
2. **强制嵌套独立包**：`util/` 工具箱。
   - **采用 `*x` 风格核心理由**：
     - **短促且语义明确**：`slicex.Chunk(ids, 100)`、`stringx.MaskPhone(phone)` 紧凑有力，一眼识别操作对象；
     - **无挤压拼接**：避免了 `maputil` 这种违背项目规范的无间隔单词挤压；
     - **杜绝标准库命名冲突**：加 `x` 完美规避了与 Go 1.21+ 官方标准库 `slices`、`maps`、`strings`、`time` 的同名冲突（Shadowing）；
     - **隔离重型依赖**：`excel/` 隔离大型第三方库，避免全局依赖污染。

---

# 4. 核心工具包规范与高频标杆实现 (`pkg/util/*x`)

### 4.1 切片工具 (`pkg/util/slicex`)
```go
package slicex

// Chunk 批量分块 (防超大 SQL 与批量 RPC 必备)
func Chunk[T any](items []T, size int) [][]T {
    if size <= 0 || len(items) == 0 {
        return nil
    }
    chunks := make([][]T, 0, (len(items)+size-1)/size)
    for i := 0; i < len(items); i += size {
        end := i + size
        if end > len(items) {
            end = len(items)
        }
        chunks = append(chunks, items[i:end])
    }
    return chunks
}

// Unique 原序去重
func Unique[T comparable](items []T) []T {
    seen := make(map[T]struct{}, len(items))
    res := make([]T, 0, len(items))
    for _, item := range items {
        if _, exists := seen[item]; !exists {
            seen[item] = struct{}{}
            res = append(res, item)
        }
    }
    return res
}

// SafeSubSlice 安全截取，杜绝下标越界 Panic
func SafeSubSlice[T any](items []T, start, limit int) []T {
    if start < 0 {
        start = 0
    }
    if start >= len(items) {
        return []T{}
    }
    end := start + limit
    if end > len(items) {
        end = len(items)
    }
    return items[start:end]
}
```

### 4.2 字典工具 (`pkg/util/mapx`)
```go
package mapx

// KeyBy 切片转字典索引 (如 []*Order -> map[ID]*Order)
func KeyBy[K comparable, T any](items []T, keyFn func(T) K) map[K]T {
    res := make(map[K]T, len(items))
    for _, item := range items {
        res[keyFn(item)] = item
    }
    return res
}

// GroupBy 切片按指定属性聚类
func GroupBy[K comparable, T any](items []T, keyFn func(T) K) map[K][]T {
    res := make(map[K][]T)
    for _, item := range items {
        k := keyFn(item)
        res[k] = append(res[k], item)
    }
    return res
}
```

### 4.3 字符串工具 (`pkg/util/stringx`)
```go
package stringx

import "unicode/utf8"

// SafeTruncate UTF-8 安全截断 (绝对禁止用 string[:10] 强截中文，会截半导致乱码)
func SafeTruncate(s string, maxRunes int, tail string) string {
    if utf8.RuneCountInString(s) <= maxRunes {
        return s
    }
    runes := []rune(s)
    if maxRunes < len(runes) {
        return string(runes[:maxRunes]) + tail
    }
    return s
}

// MaskPhone 手机号前 3 后 4 掩码脱敏
func MaskPhone(phone string) string {
    if len(phone) < 7 {
        return phone
    }
    return phone[:3] + "****" + phone[len(phone)-4:]
}
```

### 4.4 时间工具 (`pkg/util/timex`)
```go
package timex

import "time"

// StartOfDay 获取当天 00:00:00 自然起始时间
func StartOfDay(t time.Time) time.Time {
    y, m, d := t.Date()
    return time.Date(y, m, d, 0, 0, 0, 0, t.Location())
}

// EndOfDay 获取当天 23:59:59 自然结束时间
func EndOfDay(t time.Time) time.Time {
    y, m, d := t.Date()
    return time.Date(y, m, d, 23, 59, 59, int(time.Second-time.Nanosecond), t.Location())
}
```

---

# 5. 时间格式化与时间常量规范 (Time Formatting & Constants)

Go 语言的时间格式化基于基准时间 `2006-01-02 15:04:05 MST`（助记：`1 2 3 4 5 6 -0700`）。

### 5.1 优先使用 `time` 标准库内置常量
- 完整时间：优先 `time.DateTime`；纯日期：`time.DateOnly`；纯时间：`time.TimeOnly`；网络传输：`time.RFC3339`。

### 5.2 自定义时间格式强制常量化
- 严禁散落硬编码字符串，收敛在 `timex/constants.go` 中：
  ```go
  const (
      LayoutCompactDateTime = "20060102150405"
      LayoutCompactDate     = "20060102"
      LayoutSlashDateTime   = "2006/01/02 15:04:05"
      LayoutMilliDateTime   = "2006-01-02 15:04:05.000"
      LayoutYearMonth       = "200601"
  )
  ```
- **严禁使用 `%Y-%m-%d` 或 `yyyy-MM-dd`** 等其他语言占位符。

---

# 6. 常量定义与 iota 枚举设计工程铁律 (Constants & iota Enum)

- **常量分组与驼峰命名**：使用 `const (...)` 分组，严禁下划线；
- **🚨 铁律一：必须预留首行 `0` 值为 `Unknown` (防零值歧义)**：
  - 未初始化的变量默认为 `0`。若有效状态设为 `0`，未赋值结构体会被误识别为有效状态！
  ```go
  type AccountStatus uint8
  const (
      AccountStatusUnknown AccountStatus = iota // 0 (预留未初始化状态)
      AccountStatusActive                       // 1
      AccountStatusDisabled                     // 2
  )
  ```
- **🚨 铁律二：复合开关采用位掩码**：使用 `1 << iota` 构建多重权限位。

---

# 7. 指针使用与内存语义黄金规范 (Pointers & Memory Semantics)

### 7.1 核心心智模型：值语义 vs 指针语义
- **值语义**：数据独立隔离、天然并发安全；**栈分配，随函数退出直接销毁，零 GC 负担**；
- **指针语义**：共享底层内存；**易引发逃逸分析逃逸到堆，增加 GC 扫描停顿**；
- 💡 **性能箴言**：对于 64 字节以内的小结构体，传值比指针逃逸到堆更快。

### 7.2 必须/推荐使用指针的场景
1. **修改内部状态**；
2. **结构体包含不可复制的同步原语**（`sync.Mutex` 等必须指针接收者，严禁传值拷贝）；
3. **大型结构体传参 (> 64 字节)**；
4. **表达“可选 / Nullable”业务语义**；
5. **工厂方法与构造函数**（如 `NewServiceContext() *ServiceContext`）。

### 7.3 绝对禁止使用指针的四大反常识禁区
1. ❌ **严禁对内置引用类型套指针**：`*[]T`、`*map`、`*chan`、`*any`；
2. ❌ **严禁对轻量不可变值对象使用指针**：官方明确 `time.Time` 应当作为值传递；
3. ❌ **臭名昭著的“带类型的 nil 接口”陷阱**：返回 `error` 接口时无错必须显式 `return nil`，绝不能返回值为 nil 的具体指针变量；
4. ❌ **并发协程中的指针共享竞态**：启动异步 Goroutine 前必须完成值拷贝，避免外部循环修改指针。

### 7.4 Uber Go 方法接收者一致性原则
- 一个类型的方法集尽量统一为全值或全指针；只要有任意方法修改状态或持有 Mutex，该类型**所有方法统一使用指针接收者**。

---

# 8. 并发原语与协程安全

- **Channel 所有权**：只有生产者有权关闭 Channel，消费端双返回值判空 `v, ok := <-ch`；
- **防 Goroutine 泄漏**：子任务调度统一使用 `errgroup.WithContext` 或 `select` 监听退出；
- **锁拷贝禁令**：带有 Mutex 的结构体严禁传值。

---

# 9. 地道错误处理与 Context 传递

- 错误包装使用 `%w` 动词保留因果堆栈；`errors.Is` 替代 `==`；
- `ctx context.Context` 稳居第一参数，严禁存入结构体，派生上下文紧跟 `defer cancel()`。

---

# 10. Go 基础开发审查 Checklist

- [ ] **代码封装**：是否杜绝了跨层穿透调用？入参是否使用了选项模式？
- [ ] **pkg 分层治理**：一级目录是否收紧？工具包是否归拢于 `pkg/util/` 且二级采用 `*x` 扩展风格（`slicex/mapx/stringx/timex`）？
- [ ] **工具纯函数**：切片/字典工具是否为纯函数？字符串截断是否做了 UTF-8 rune 防护？
- [ ] **时间与常量规范**：时间格式是否优先标准库？`iota` 是否预留了 `0` 为 `Unknown`？
- [ ] **指针规范**：是否杜绝了 `*[]T`、`*map`？返回 `error` 是否显式 `return nil`？Mutex 结构体方法是否全为指针接收者？
- [ ] **并发安全**：异步协程传入指针是否已做值隔离拷贝？
