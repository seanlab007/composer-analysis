# Composer 2 架构分析 → MaoAI / ASI-Genesis 整合方案

对 Composer 2 源码（PHP 依赖管理工具，29,475 ⭐）的深度架构拆解，提取 6 个可复用的架构 Pattern，反向指导 MaoAI 多 Agent 平台与 ASI-Genesis 自进化引擎的重构。

> **Fork 源仓库**：[seanlab007/composer](https://github.com/seanlab007/composer)（forked from [composer/composer](https://github.com/composer/composer)，29,475 ⭐，4,802 forks）
> **作者**：润之（WorkBuddy） · 2026-06-29

---

## 3 Stages Analysis

### Stage 1 · 拆解（5 大子系统）
1. **核心入口** `Composer.php` + `Factory.php` — PartialComposer（读侧）vs Composer（写侧）的渐进式装配
2. **依赖求解器** `DependencyResolver/` — DPLL SAT + RuleWatchChain 增量重算 + Policy 打分
3. **包仓库** `Repository/` — 7 种异构源统一为 `RepositoryInterface`，组合模式 + 有序 fallback
4. **插件系统** `Plugin/` — Plugin 作为 EventDispatcher 的 subscriber，capability 声明
5. **安装器** `Installer/` + **事件总线** `EventDispatcher` — InstallOperation 队列 + 6 个核心事件

### Stage 2 · Pattern（6 个可复用架构模式）
| # | Pattern | 移植到 MaoAI/ASI-Genesis |
|---|---------|--------------------------|
| P1 | **Partial + Factory 渐进式装配** | `Fable5Enhance.__init__` 改 lazy property |
| P2 | **Pool + Rule + SAT 求解** | 模型选择从 if/else 改为约束求解 |
| P3 | **Repository 接口 + 组合模式** | LLM 源从硬编码 5 个改为可插拔 |
| P4 | **Plugin = Event subscriber** | Skill 注册表改为 capability + listener |
| P5 | **InstallOperation 队列 + 原子化** | skill 安装加 `.lock.tmp` + 断电 rollback |
| P6 | **content-hash 锁文件** | lock 文件记录 `skill_registry.py` 的 SHA256 |

### Stage 3 · 整合方案（Top 3 优先整合）

| Priority | Pattern | 文件 | 工作量 | 收益 |
|----------|---------|------|--------|------|
| **P1** | **EventBus** | `server/hyperagents/core/event_bus.py`（90 行）| 1 天 | 解锁 3 个独立 plugin（dream / audit / metrics）解耦集成 |
| **P2** | **LLM Repository Manager** | `server/hyperagents/core/repository_manager.py`（180 行）| 2 天 | 未来加 Gemini / vLLM / Claude Code SDK 都是 1 文件 + 30 行 |
| **P3** | **Factory 渐进式装配** | 重构 `fable5_enhance.py` | 1 天 | 再加第 8、第 9 个子模块不会循环依赖；启动更快 |

---

## 核心文件

- [`analysis.md`](./analysis.md) — 完整 350 行分析（5 子系统 + 6 Pattern + 3 反模式 + 2 隐藏宝石 + Top 3 整合路径 + 验证清单）

## 不做什么（明确边界）

- 不照搬 SAT Solver — 用 `python-constraint` 或 LLM 投票
- 不全量 Event 化 — 先做 3 个核心事件就够
- 不一个 Repo 包打天下 — 4 个 LLM 源都是 OpenAI 协议，1 个 adapter 足矣
- 不做 composer.json manifest — 已有 `package.json` + `SKILL.md`
- 不做 Audit 集成 — MaoAI `SecurityAuditor` 已经更好

## 长期愿景

- **短期（1-2 周）**：EventBus + RepositoryManager 上线 → 插件生态起飞
- **中期（1-2 月）**：Plugin 完整化 → 第三方开发者可写 MaoAI 插件
- **长期（6 月+）**：MaoAI Plugin Marketplace，类似 Packagist，但插件 = Agent 能力而非包

---

_License: MIT（分析文档）/ Composer 2 源码: MIT（composer/composer）_
