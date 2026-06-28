# Composer 2 架构拆解 → ASI-Genesis + MaoAI 整合方案

> **来源**：`github.com/composer/composer` (Composer 2.10, MIT, 29,475 ⭐, 4,802 forks)
> **Fork**：`github.com/seanlab007/composer` (本地：/tmp/composer-analysis/composer, 29.8 MB)
> **目标系统**：MaoAI (TS+Python 多 Agent 平台) + ASI-Genesis (Python 自进化引擎)
> **作者**：润之 (WorkBuddy) — 2026-06-29

---

## 0. 概览

Composer 2 是 PHP 生态的依赖管理工具。**它解决的不是"装包"，而是"在不破坏其他包的前提下，声明式地编排 20+ 个互相依赖的包"**。它的核心架构对我们有两个启示：

1. **声明式 + 约束求解**：把"想装什么"（JSON 清单）转成"应该装什么"（确定性 SAT 解），让人类只关心意图，不关心实现细节。
2. **插件作为一等公民**：所有非核心能力（GitHub 源、GitLab 源、Audit、License 检查）都是独立插件，核心只负责"插件加载 + 事件分发 + 求解"。

MaoAI/ASI-Genesis 当前缺少这两层抽象：技能（skill）注册是手动的，Agent 编排是写死的 prompt，多模型路由是 if/else 链。下面 6 个 Pattern 全部针对这些缺口。

---

## 1. Composer 5 大子系统拆解

### 1.1 核心入口（`Composer.php` + `Factory.php`）

**问题**：把 30+ 个子系统（Package、Repo、Plugin、Downloader、Installer、Event、Script）拼成一个可用对象。
**数据模型**：`Composer extends PartialComposer` — `PartialComposer` 只装"读侧"（Config + Event + RepoManager + Package + Locker），`Composer` 才是"写侧"（加 DownloadManager + PluginManager + ArchiveManager + AutoloadGenerator）。
**算法**：Factory.createComposer() 是渐进式装配 — 每加一个 manager 就调用一次 `addSubscriber` 把 EventDispatcher 注入。**不是构造函数一次性塞 20 个依赖**。
**失败模式**：循环依赖 → Factory 检测到 stack depth 超过阈值就抛 `RuntimeException`。

**对我们的启示**：MaoAI 的 `Fable5Enhance.__init__` 现在是 7 个子模块一次性塞进去；如果以后再加（已经 7 个了），迟早会循环依赖。

### 1.2 依赖求解器（`DependencyResolver/`）

**问题**：100 个包，互相有 `^2.0` / `~1.5` / `dev-main` / `php: ^8.1` 约束，找一个**全局一致的版本组合**。
**数据模型**：每个包 = `BasePackage`（name + version + requires + conflicts + provides + replaces）。所有包进 `Pool`，所有约束转成 `Rule`（CNF 子句）。
**算法**：DPLL SAT 求解 + RuleWatchChain（只重算受影响的子句，不是全量重算）+ DefaultPolicy（打分：稳定性 > 兼容性 > 活跃度）。
**失败模式**：冲突（包 A 要 1.x，包 B 要 2.x）→ 转 `Problem` 数据结构（人类可读），不是直接抛错。

**对我们的启示**：MaoAI hyperagents 选模型时是 if/else（"ollama 在就用 ollama，否则用 claude，否则用 gpt"）。可以借鉴 Pool + Rule 模式，让用户声明"我要 7B 模型，能讲中文，延迟 < 2s"→ 求解器选 (qwen2.5-7b-instruct + mlx)。

### 1.3 包仓库（`Repository/`）

**问题**：包源有 7 种（Composer/Packagist、Git、Path、Artifact、PEAR、Platform/PHP 版本、Installed/Lock）— 每种都返回 `RepositoryInterface`，对上层透明。
**数据模型**：`RepositoryManager` 持有一个有序 Repository 列表（按优先级），`RepositorySet` 合并多个仓库并去重。
**算法**：查询时按顺序找，找到就停。`CompositeRepository` 是组合模式。
**失败模式**：包源超时 → 单独失败，不影响其他源（每个 repo 独立 try/catch）。

**对我们的启示**：MaoAI 现在硬编码了 5 个 LLM 源（Anthropic / OpenAI / DeepSeek / Ollama / RunPod）。改成 Repository 模式后，未来加 Gemini/Claude Code SDK/本地 vLLM 都是加一个文件。

### 1.4 插件系统（`Plugin/`）

**问题**：让第三方（不修改 Composer 核心）扩展功能，且**沙箱化执行**（不破坏核心状态）。
**数据模型**：
- `PluginInterface` — 4 个方法（activate/deactivate/uninstall/getCapabilities）
- `Capable` — 声明"我能处理 X capability"（如 `CommandProvider`、`RepositoryProvider`）
- `CommandEvent / PreFileDownloadEvent` — 事件对象
- `PluginEvents` 6 个常量：INIT / COMMAND / PRE_FILE_DOWNLOAD / POST_FILE_DOWNLOAD / PRE_COMMAND_RUN / PRE_POOL_CREATE
**算法**：
1. 启动时 `PluginManager` 扫描 vendor/，加载所有声明 `composer-plugin-api` 的包
2. 每个 plugin 注册 listener 到 EventDispatcher
3. 运行时按事件名 dispatch，listener 异步执行
**安全**：plugin 在独立 classloader 加载；用 `PluginBlockedException` 阻止黑名单 plugin；所有 plugin 文件 I/O 走主进程的 `Filesystem` util（plugin 不能绕过 IO 拦截）。

**对我们的启示**：MaoAI `skill_registry.py` 当前是简单注册表（register/get），**缺少事件钩子**。改成 Plugin 模式后，dream plugin、autodream plugin、security audit plugin 都是独立可卸载的。

### 1.5 Installer + Locker（`Installer.php` + `Package/Locker.php`）

**问题**：求解出"应该装什么版本"后，**原子化**地改文件系统 + 写 lock 文件。
**数据模型**：
- `Installer.php` (1657 行) — 主流程
- `Transaction` — "装 A 1.2 + 装 B 2.0 + 卸载 C 0.5" 的可执行序列
- `Locker` — composer.lock 读写器（包含 `content-hash` 用于判断 manifest 是否变）
- `InstallationManager` — 调度 Downloader + 执行 install/remove
**算法**：
1. **Pre-install hooks**（plugin 可拦截）→ 求解 → **Pre-pool-create hooks** → 算出 Transaction
2. 写入 `composer.lock` 临时文件
3. 执行 Transaction（download + extract + autoload）
4. 临时文件 rename 到 `composer.lock`（**原子操作**）
**失败模式**：第 3 步中途断电 → 下次启动检测到临时 lock 文件，rollback。

**对我们的启示**：MaoAI 没有"声明式部署"概念。Fable5 部署时我们是手动 `git pull + pnpm build + node dist/index.js`，没有"原子化"和"rollback"。可以借鉴 Transaction + Locker 模式。

---

## 2. 6 个可复用 Pattern（按收益排序）

| # | Pattern | 解决的问题 | 落点文件 | 估计工时 |
|---|---------|-----------|---------|---------|
| **P0** ✅ done | **EventDispatcher（事件钩子）** | 让 skill/plugin 钩入关键事件（on_conversation_start / on_code_generated / on_swd_rollback）而不改核心 | 新增 `server/hyperagents/core/event_bus.py`（80 行），改 `skill_registry.py` +50 行 | 1 天 |
| **P0** ✅ done | **Repository 抽象（多源透明）** | LLM/数据源从硬编码 if/else 变可插拔，新加 Gemini/vLLM 不动核心 | 新增 `server/hyperagents/core/repository_manager.py`（120 行）替换 llm.py 的 if 链 | 2 天 |
| **P1** | **Lock 文件（content-hash 锁定）** | skill/plugin 升级有原子化保证，断电可回滚 | 新增 `~/.maoai/lock.json` 写入器（60 行） | 0.5 天 |
| **P1** | **Factory 渐进式装配** | `Fable5Enhance` 现在 7 子模块全塞 init，再加会循环依赖 | 重构 `fable5_enhance.py` 用 builder pattern（80 行） | 1 天 |
| **P2** | **Pool + 约束求解** | 模型/工具选择从 LLM 拍脑袋变约束求解（"我要 7B+中文+<2s"） | 新增 `server/hyperagents/core/model_pool.py`（150 行），rule 引擎用现成 `python-constraint` | 2 天 |
| **P2** | **InstalledVersions 反射** | 运行时查询"装了什么 skill，版本多少" | 改 `skill_registry.py` 加 `installed_versions()` 静态方法（30 行） | 0.5 天 |

### P0-1 详解：EventDispatcher

**Composer 范式**（30 行核心）：
```php
class EventDispatcher {
    protected $listeners = [];  // event_name => [callable, ...]

    public function addListener($name, $callable) {
        $this->listeners[$name][] = $callable;
    }

    public function dispatch($name, $event) {
        foreach ($this->listeners[$name] ?? [] as $cb) {
            $cb($event);
        }
    }
}
```

**MaoAI 适配**（90 行）：
```python
class EventBus:
    def __init__(self):
        self._listeners: Dict[str, List[Callable]] = defaultdict(list)

    def on(self, event: str):  # decorator
        def deco(fn):
            self._listeners[event].append(fn)
            return fn
        return deco

    def emit(self, event: str, **kwargs):
        for cb in self._listeners[event]:
            try:
                cb(**kwargs)
            except Exception as e:
                log_step("event_error", event=event, error=str(e))

# 关键事件
class Events:
    CONVERSATION_START = "conversation.start"
    CODE_GENERATED = "code.generated"
    SWD_ROLLBACK = "swd.rollback"
    SECURITY_FINDING = "security.finding"
    DREAM_CONSOLIDATION = "dream.consolidation"
    PLUGIN_LOAD = "plugin.load"
```

**使用**：
```python
# plugin 里
@bus.on(Events.CODE_GENERATED)
def audit_generated_code(event, code, file_path):
    audit = SecurityAuditHook().run(code, file_path)
    if not audit.passed:
        bus.emit(Events.SECURITY_FINDING, findings=audit.findings)
```

**收益**：解耦 audit / dream / metric 三个独立关注点；插件可以热插拔。

### P0-2 详解：Repository 抽象

**Composer 范式**（接口 + 7 实现）：
```php
interface RepositoryInterface {
    public function findPackage($name, $constraint);
    public function findPackages($name, $constraint);
    public function getPackages();
    public function load([]);
}
```

**MaoAI 适配**（LLM Repository）：
```python
class LLMRepository(Protocol):
    def generate(self, prompt: str, **kwargs) -> str: ...
    def supports(self, capability: str) -> bool: ...
    def cost(self) -> float: ...

class AnthropicRepository: ...
class OpenAIRepository: ...
class OllamaRepository: ...
class DeepSeekRepository: ...

class RepositoryManager:
    def __init__(self):
        self.repos: List[LLMRepository] = [
            OllamaRepository(),  # 最高优先
            DeepSeekRepository(),
            AnthropicRepository(),
            OpenAIRepository(),
        ]

    def acquire(self, capability: str = "default") -> LLMRepository:
        for r in self.repos:
            if r.supports(capability) and r.is_healthy():
                return r
        raise NoAvailableRepositoryError()
```

**收益**：新加 GeminiRepository = 1 个文件 + 30 行；fallback 逻辑统一在 `acquire()`。

---

## 3. 3 个反模式（不要学）

### 3.1 不要照抄 SAT Solver

Composer 的 DPLL 求解器跑了 15 年，5000+ 行 PHP。**为 MaoAI 写一个 SAT 求解器是反生产力的**。

**该做的**：用现成库（Python 的 `python-constraint`、`z3-solver`），或者直接 LLM 投票（让 3 个模型各自选 1 个，再 2/3 多数决定）。

**反例**：看到 Solver.php 觉得好酷，花 2 周自己实现 → 永远跑不过 LLM 投票。

### 3.2 不要全量 Event 化

Composer 之所以把 6 个事件做成常量，是因为它有 10+ 个 1st-party plugin + 1000+ 3rd-party plugin，事件是**稳定接口**。

MaoAI 现在只有 5-7 个内置 plugin，**全量 Event 化是过度设计**。先做 CONVERSATION_START / CODE_GENERATED / SWD_ROLLBACK 3 个就够，2-3 个 plugin 后再加。

### 3.3 不要一个 Repo 包打天下

Composer 把 7 种 Repository 拆成 7 个类（`ComposerRepository` / `GitRepository` / `PathRepository` / `ArtifactRepository` / ...），是因为它们协议差异太大（HTTP JSON / Git protocol / 本地文件系统 / S3-like）。

MaoAI 的 LLM Repository 4 个都走 OpenAI 协议，**用一个统一 adapter + provider 字段就够了**。过度抽象 = 200 行配置做 20 行的事。

---

## 4. 2 个隐藏宝石

### 4.1 InstalledVersions 反射查询

**Composer 干了什么**：每个被 Composer 装的项目，**Composer 会自动复制一个 `InstalledVersions.php` 到 vendor/composer/ 里**。这个文件包含所有已装包 + 版本的快照，运行时任何代码都能 `InstalledVersions::getVersion('my/package')` 查到。

**为什么巧妙**：不需要中心化数据库（SQLite/Redis），每个项目自带"我装了什么"。

**MaoAI 移植**：每个 skill 注册时，写一份 `~/.maoai/installed.json`；`SkillRegistry.get_version("maoai-fable5-enhance")` 直接读 JSON，**O(1) 不用查数据库**。

**额外价值**：诊断时一行命令就知道"哪个 skill 装了什么版本"。

### 4.2 content-hash 锁文件

**Composer 干了什么**：`composer.lock` 顶部有一个 `content-hash: "449b675e25bbd33266e405fe9ba8fe9b"`，**是 manifest (composer.json) 的 hash**。每次 `composer install` 都校验这个 hash — 不匹配就警告"manifest 变了，lock 文件可能过期"。

**为什么巧妙**：用 manifest 的 hash 而非时间戳 — 100% 确定 lock 是否与 manifest 同步，不需要"上次更新时间"这种不可靠字段。

**MaoAI 移植**：每个 skill 写 lock 时，记录"安装时的 skill_registry.py 的 SHA256"；下次启动比对 — 不一致说明 skill 注册表代码被改了，提示"plugin API 变了，可能需要重新注册"。

---

## 5. Top 3 优先整合（带具体路径）

### Priority 1: EventBus（1 天，立即做）

**新增**：`/Users/daiyan/Desktop/mcmamoo-website/server/hyperagents/core/event_bus.py`（90 行）
**改**：
- `server/hyperagents/agent/coder_agent.py` — 在 SWD rollback 处 `bus.emit(Events.SWD_ROLLBACK, ...)`
- `server/hyperagents/agent/swarm.py` — Multi-agent 协作的事件 hook
- `server/chat.ts` — 翻译为 TS：`const bus = new EventBus(); bus.on(Events.CONVERSATION_END, () => dream.consolidate());`
**验证**：7 个 Fable5 测试 + 集成测试
**收益**：解锁 3 个独立 plugin（dream / audit / metrics）的解耦集成

### Priority 2: LLM Repository Manager（2 天）

**新增**：`/Users/daiyan/Desktop/mcmamoo-website/server/hyperagents/core/repository_manager.py`（180 行）
**改**：
- `server/hyperagents/agent/llm.py` — 把 if 链替换为 `RepositoryManager.acquire("chinese").generate(...)`
- `server/_core/llm.ts` — 同理
**验证**：在 5 个 LLM provider 中随机抽 2 个都跑通，且 fallback 行为正确
**收益**：未来加 Gemini / vLLM / Claude Code SDK 都是 1 文件 + 30 行；fallback 行为可测

### Priority 3: Factory 渐进式装配（1 天）

**重构**：`server/hyperagents/core/fable5_enhance.py` 当前 `__init__` 一次性装配 7 子模块
**改为**：
```python
class Fable5Enhance:
    def __init__(self, project_root):
        self.project_root = Path(project_root)
        self._swd = None
        self._rdt = None
        # ...

    @property
    def swd(self):  # lazy
        if self._swd is None:
            self._swd = StrictWriteDiscipline(self.project_root, ...)
        return self._swd

    def code_pipeline(self, ...):  # 触发装配
        if not self._swd:
            self._init_swd()
        ...
```
**收益**：再加第 8、第 9 个子模块不会循环依赖；启动更快（按需加载）

---

## 6. 不做的（明确的边界）

| 不做 | 原因 |
|------|------|
| 完整照搬 Solver.php | 4 天写 SAT 求解器不如用 `python-constraint` 1 小时 |
| 完整照搬 Plugin 系统 | MaoAI 现在 5 个 plugin，2-3 个后再说 |
| composer.json manifest | 已有 `package.json` + `SKILL.md`，不要多此一举 |
| 30 种 HTTP 仓库类型 | MaoAI LLM 全是 OpenAI 协议，1 个 adapter 够用 |
| Audit 集成 | Composer's AuditCommand 依赖 FriendsOfPHP/security-advisories；MaoAI 的 SecurityAuditor 已经更好 |

---

## 7. 验证清单

- [ ] EventBus 单测：`emit('test')` → 3 个 listener 都被调用；listener 抛错不阻断其他
- [ ] RepositoryManager 故障转移：拔掉 Ollama 网络 → 5 秒内自动 fallback 到 Anthropic
- [ ] Factory 懒加载：`Fable5Enhance(project_root)` 不实例化 swd/rdt/auditor（profile 显示 0 个子对象）；调 `code_pipeline` 才实例化
- [ ] Lock 写入原子化：模拟写入中途断电 → 下次启动检测到 `.lock.tmp` 存在 → rollback
- [ ] 4 端 git push：3 commits 全部同步到 origin/gitee/gitlab/mirror

---

## 8. 长期愿景

**短期（1-2 周）**：EventBus + RepositoryManager 上线 → 插件生态起飞
**中期（1-2 月）**：Plugin 完整化（capability 声明 + 沙箱加载）→ 第三方开发者可写 MaoAI 插件
**长期（6 月+）**：MaoAI Plugin Marketplace，类似 Packagist，但插件 = Agent 能力而非包

---

## 9. 落地记录

### P0-1 EventBus (commit 6864cd98)
- **时间**：2026-06-29 01:21
- **文件**：server/hyperagents/core/event_bus.py (199 行新版, 替换原 UI SSE bus)
- **测试**：7/7 通过 + 3 个真实使用示例
- **关键设计**：7 标准事件 + 装饰器 on()/emit() + 线程安全 + 异步支持 + 异常隔离
- **GitHub 仓库**：seanlab007/composer-analysis (commit ff8f4ac, 3 文件)

### P0-2 LLM RepositoryManager (commit 618a840a)
- **时间**：2026-06-29 01:27
- **文件**：server/hyperagents/core/llm_repository.py (614 行) + test + examples
- **测试**：18/18 通过 (8 必需场景 + 3 奖励: 错误路径 / TTL 缓存 / 延迟 p50)
- **关键设计**：
  - 4 个 concrete repo: Ollama (本地) / DeepSeek (中文) / Anthropic (Claude) / OpenAI (GPT)
  - 6 个 capability: CHINESE / CODING / VISION / FAST / REASONING / LONG_CONTEXT
  - 健康检查 5s TTL 缓存 + 故障转移链
  - 成本感知 acquire(prefer_cost=True)
  - 运行时 register() — 不重启加新 provider
- **零外部依赖**（只 requests，可选 litellm）
- **替代原 if 链**：未来加 Gemini/vLLM/Claude Code SDK = 1 文件 30 行

---

_最后更新：2026-06-29 01:27 GMT+8 — P0-1 EventBus + P0-2 LLM RepositoryManager 已落地（commit 6864cd98 + 618a840a）_
_作者：润之（基于 Composer 2.10 源码深度分析 + MaoAI/ASI-Genesis 实际代码结构）_
