# agent-skill-golang-base

> ⚡ 地道 Go 语言核心开发规范、Uber Go 风格实践、并发原语、错误包装与内存安全 Agent Skill。

## 🌟 核心特性 (Features)

- **地道并发原语管理**：Channel 所有权闭环、`errgroup` 并发子任务协同、防范 Goroutine 泄漏、禁止锁拷贝。
- **地道错误闭环**：强制 `%w` 包装错误链、`errors.Is`/`errors.As` 解包断言、严禁裸 `_` 忽略错误。
- **Context 传递铁律**：第一参数原则、严禁结构体持久化持有、`defer cancel()` 防泄露。
- **极致内存与性能**：切片容量预分配、接收者选型黄金法则、`strings.Builder` 零多余堆分配。

## 📦 安装与多 Agent 使用指南 (Installation & Multi-Agent Usage)

### 1. Google Antigravity / Gemini Code Assist
- **全局安装（推荐）**：
  ```bash
  git clone git@github.com:Garfield247/agent-skill-golang-base.git ~/.gemini/config/skills/golang-base
  ```
- **项目工作区局部引入**：
  ```bash
  mkdir -p .agents/skills
  git clone git@github.com:Garfield247/agent-skill-golang-base.git .agents/skills/golang-base
  ```

### 2. Anthropic Claude (Claude Code / Claude Projects)
- **Claude Code (CLI 终端智能体)**：
  ```bash
  mkdir -p ~/.claude/skills
  git clone git@github.com:Garfield247/agent-skill-golang-base.git ~/.claude/skills/golang-base
  ```
- **Claude Projects (Web 客户端)**：
  直接将仓库中的 `SKILL.md` 内容复制并粘贴至 Project 的 **Project Knowledge**。

### 3. Cursor / GitHub Copilot / OpenAI Codex
- **Cursor (现代 MDC 规则体系)**：
  ```bash
  mkdir -p .cursor/rules
  git clone git@github.com:Garfield247/agent-skill-golang-base.git .cursor/rules/golang-base
  ```
- **GitHub Copilot / Codex**：
  将本技能规范注入项目的 `.github/copilot-instructions.md`。

## 📄 开源协议 (License)
本项目采用 [MIT License](LICENSE) 授权。
