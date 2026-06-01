# Hermes Agent 代码库深度分析报告

> 生成时间: 2026-06-01
> 探索范围: 8个并行子agent同时探索不同子系统

---

## 1. 项目概述

Hermes Agent 是 Nous Research 构建的**多平台 AI Agent 框架**，通过工具调用循环（tool-calling loop）与 LLM API 交互。核心能力包括：

- **17+ 消息平台集成**（Telegram、Discord、WhatsApp 等）通过适配器架构
- **6 种终端执行后端**（本地、Docker、SSH、Modal、Daytona、Singularity）
- **40+ 工具**按 toolsets 组织，支持并行执行
- **渐进式披露技能系统**（Skills），通过 SKILL.md 文档提供领域知识
- **SQLite FTS5 持久化会话**，支持全文搜索和会话恢复
- **MCP/ACP 适配器**，支持 VS Code、Zed、JetBrains、Claude Code 等 IDE 集成

---

## 2. 核心架构

```
┌─────────────────────────────────────────────────────┐
│              CLI / TUI (prompt_toolkit)             │
│              hermes_cli/main.py                     │
└─────────────────────┬───────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│              GatewayRunner                          │
│         多平台消息路由 & 会话管理                      │
│    (Telegram, Discord, WhatsApp, BlueBubbles)       │
└─────────────────────┬───────────────────────────────┘
                      │
    ┌─────────────────┼─────────────────┐
    │                 │                 │
    ▼                 ▼                 ▼
┌─────────┐    ┌──────────┐    ┌──────────────┐
│   MCP   │◄──►│  AIAgent │◄──►│ ToolRegistry │
│ Clients │    │ (核心循环) │    │  (40+工具)   │
└─────────┘    └────┬─────┘    └──────────────┘
                    │
    ┌───────────────┼───────────────┐
    │               │               │
    ▼               ▼               ▼
┌─────────┐    ┌──────────┐    ┌──────────┐
│ Skills  │    │Environments│   │ SessionDB │
│(渐进式)  │    │(多后端)    │    │ (FTS5)   │
└─────────┘    └──────────┘    └──────────┘
```

---

## 3. 核心子系统详解

### 3.1 Agent Core (`run_agent.py`, `toolsets.py`, `model_tools.py`)

**核心类: AIAgent** (run_agent.py:438-7000+)

| 关键方法 | 职责 |
|----------|------|
| `__init__` | 初始化 agent（模型、toolsets、回调、迭代预算） |
| `_build_system_prompt` | 7层系统提示词组装 |
| `run_conversation` | 主循环入口，执行完整工具调用直到完成 |
| `_execute_tool_calls` | 分析工具独立性，决定并行/串行执行 |
| `_invoke_tool` | 通过 registry 分发或处理 agent 级工具 |
| `_compress_context` | 达到 token 上限时触发上下文压缩 |

**关键机制:**

1. **工具调度流程**: `tool_calls → 分析独立性 → 并行/串行执行 → registry dispatch → 结果追加到消息`
2. **系统提示词7层**: SOUL.md → 用户/网关提示 → 持久记忆 → 技能指导 → 上下文文件 → 时间戳+平台提示
3. **上下文压缩算法**: 修剪旧工具结果 → 保护头尾 → 预算内尾部保护 → 结构化摘要 → 迭代更新
4. **API模式抽象**: `chat_completions` / `codex_responses` / `anthropic_messages` 三种模式

**设计模式:**
- 工具自注册（`registry.register()` at import time）
- 路径冲突检测的并行工具执行
- 线程本地持久化事件循环避免"Event loop is closed"错误
- 前缀缓存（系统提示词仅在压缩时重建）
- 迭代预算在父子代理间共享，支持 refund

---

### 3.2 CLI (`cli.py`, `hermes_cli/`)

**核心类: HermesCLI** (hermes_cli/main.py)

| 关键方法 | 职责 |
|----------|------|
| `run()` | TUI 主循环 |
| `process_command()` | slash 命令解析和分发 |
| `_init_agent()` | Agent 初始化 |
| `_handle_model_switch()` | 运行时模型切换 |

**关键机制:**

1. **Slash 命令注册表**: `COMMAND_REGISTRY` 是唯一真实来源，派生 `COMMANDS` / `COMMANDS_BY_CATEGORY` 等查找表
2. **Profile 覆盖**: `--profile/-p` 设置 `HERMES_HOME` 环境变量实现配置隔离
3. **配置迁移**: `_config_version` 版本化迁移系统
4. **皮肤引擎**: YAML 驱动的数据配置，定义颜色/品牌/动画

**复杂热点:**
- `process_command()` 超过 300 行的巨型 if-elif 分发链
- `select_provider_and_model()` 超过 20 种 provider 的选择流程
- `cmd_update()` 超过 470 行的巨型函数

---

### 3.3 Gateway (`gateway/`)

**核心类: GatewayRunner** (gateway/run.py)

| 关键方法 | 职责 |
|----------|------|
| `start()` / `stop()` | 生命周期管理 |
| `_handle_message()` | 消息路由和认证 |
| `_create_adapter()` | 平台适配器工厂 |
| `_session_expiry_watcher()` | 会话自动重置 |

**关键机制:**

1. **平台适配器模式**: Telegram/Discord/WhatsApp 等继承 `BasePlatformAdapter`
2. **消息优先级处理**: interrupt / queue / ignore 新消息
3. **会话自动重置**: 5分钟检查过期会话，自动 flush 记忆后重置
4. **双存储**: SQLite 主存储 + JSONL 降级存储
5. **Telegram MarkdownV2**: 12步格式转换

**复杂热点:**
- `_handle_message()` 超过 250 行，认证/命令路由/优先级处理/agent执行交织
- 配置加载逻辑极其复杂（config.yaml/env/gateway.json 多源合并）

---

### 3.4 Tools System (`tools/`)

**核心类: ToolRegistry** (tools/registry.py)

| 关键方法 | 职责 |
|----------|------|
| `register` | 注册工具（名称、toolset、schema、handler） |
| `deregister` | 注销工具（MCP 动态发现时使用） |
| `dispatch` | 执行工具 handler |
| `get_definitions` | 返回 check_fn 通过的工具 schema |

**工具分类:**

| 工具 | 文件 | 职责 |
|------|------|------|
| terminal_tool | tools/terminal_tool.py | 命令执行到多环境 |
| ShellFileOperations | tools/file_operations.py | 基于 shell 的文件操作 |
| delegate_task | tools/delegate_tool.py | 子代理生成和并行执行 |
| MCPServerTask | tools/mcp_tool.py | MCP 服务器连接管理 |
| SkillTools | tools/skills_tool.py | 技能列表/查看/安装 |

**关键机制:**

1. **多后端环境抽象**: Local/Docker/Singularity/SSH/Modal/Daytona 实现统一 `execute()` 接口
2. **环境生命周期**: `_active_environments` 按 task_id 跟踪，300秒后清理
3. **写保护系统**: `WRITE_DENIED_PATHS` 保护 `~/.ssh/`、`/etc/sudoers` 等
4. **子代理隔离**: 独立对话历史、受限工具集（无 delegate/memory/execute_code）
5. **MCP 后台事件循环**: 专用线程运行 asyncio event loop

---

### 3.5 Skills System (`skills/`, `tools/skills_hub.py`)

**核心概念: 渐进式披露 (Progressive Disclosure)**

| Tier | 方法 | 返回内容 |
|------|------|----------|
| Tier 1 | `skills_list()` | 仅名称和描述 |
| Tier 2-3 | `skill_view()` | 完整 SKILL.md + 标签 + 环境变量 |
| Tier 3+ | `skill_view(name, file_path)` | 加载关联文件 |

**关键机制:**

1. **多源适配器**: GitHub / skills.sh / ClawHub / LobeHub 实现统一 `SkillSource` ABC
2. **隔离安装模式**: 下载 → 隔离到 quarantine → 安全扫描 → 用户确认 → 移动到安装目录
3. **信任感知策略**: `INSTALL_POLICY` 根据 trust_level 和 verdict 决定安装行为
4. **安全扫描**: `skills_guard.py` 使用 THREAT_PATTERNS 检测 exfiltration/injection/destructive 等威胁

**SKILL.md Frontmatter 格式:**
```yaml
---
name: skill-name
description: 简短描述
version: 1.0.0
platforms: [macos, linux]        # 可选：平台限制
required_environment_variables:   # 可选：安全的环境变量元数据
  - name: API_KEY
    prompt: API key
metadata:
  hermes:
    tags: [Category, Subcategory]
    fallback_for_toolsets: [web]  # 工具不可用时显示
    requires_toolsets: [terminal] # 工具可用时显示
---
```

---

### 3.6 Memory & Context (`hermes_state.py`, `agent/`)

**核心类: SessionDB** (hermes_state.py)

| 关键方法 | 职责 |
|----------|------|
| `create_session` | 创建新会话 |
| `append_message` | 追加消息 |
| `get_messages_as_conversation` | 获取对话历史 |
| `search_messages` | FTS5 全文搜索 |
| `_execute_write` | 带抖动重试的 WAL 写入 |

**关键机制:**

1. **SQLite FTS5 会话存储**: WAL 模式、FTS5 触发器自动维护索引、抖动重试处理写入冲突
2. **上下文压缩算法**:
   - Phase 1: 修剪旧工具结果（>200字符替换为占位符）
   - Phase 2: 保护头部（系统提示 + 第一轮对话）
   - Phase 3: 按 token 预算保护尾部（~20%上下文）
   - Phase 4: 结构化摘要中间轮次
   - Phase 5: 迭代更新保留历史摘要
3. **两层缓存**: 进程内 LRU + 磁盘快照，mtime 验证
4. **辅助客户端提供商链**: OpenRouter > Nous Portal > Custom endpoint > Codex OAuth > Anthropic

---

### 3.7 Environments (`tools/environments/`)

**核心类: BaseEnvironment** (tools/environments/base.py)

| 关键方法 | 职责 |
|----------|------|
| `execute()` | 统一的命令执行流程 |
| `init_session()` | 初始化会话快照 |
| `_run_bash()` | 后端特定的 bash 执行 |
| `cleanup()` | 资源清理 |

**执行后端:**

| 后端 | 文件 | 特点 |
|------|------|------|
| Local | base.py | 直接 subprocess.Popen |
| Docker | docker.py | 安全加固（cap-drop ALL、PID限制、tmpfs） |
| SSH | ssh.py | ControlMaster 连接复用、rsync 文件同步 |
| Modal | modal.py | 异步 worker + 自有事件循环 |
| Daytona | daytona.py | 持久化沙箱（stop/resume） |

**关键机制:**

1. **会话快照**: `export -p` / `declare -f` / `alias -p` 捕获 shell 环境，命令执行前重新加载
2. **CWD 标记**: 远程后端通过 stdout 的 `__HERMES_CWD_{session}__` 标记通信工作目录
3. **中断处理**: `_wait_for_process()` 轮询 `is_interrupted()`，杀进程、抽干线程、返回 130

---

### 3.8 MCP & Integrations (`mcp_serve.py`, `acp_adapter/`)

**两个集成接口:**

1. **MCP Server** (`mcp_serve.py`): stdio 服务器，暴露 9 个工具
   - conversations_list, conversation_get, messages_read
   - attachments_fetch, events_poll, events_wait
   - messages_send, permissions_list_open, permissions_respond

2. **ACP Adapter** (`acp_adapter/`): 异步 JSON-RPC stdio 服务器
   - VS Code、Zed、JetBrains 集成
   - SessionManager 管理内存中的 AIAgent 实例 + SessionDB 持久化

**核心类:**

| 类 | 文件 | 职责 |
|----|------|------|
| EventBridge | mcp_serve.py | 后台轮询 SessionDB，维护事件队列 |
| HermesACPAgent | acp_adapter/server.py | ACP 协议实现，包装 AIAgent |
| SessionManager | acp_adapter/session.py | 线程安全的会话管理 |
| Callback Bridge | acp_adapter/events.py | `asyncio.run_coroutine_threadsafe()` 桥接同步/异步 |

---

## 4. 关键设计模式

| 模式 | 位置 | 用途 |
|------|------|------|
| **单例注册表** | ToolRegistry, HookRegistry | 集中注册，延迟填充 |
| **模板方法** | BaseEnvironment.execute() | 算法骨架在基类，步骤在子类 |
| **适配器** | _ThreadedProcessHandle | 接口转换 |
| **观察者/事件** | HookRegistry, EventBridge | 解耦事件通知 |
| **工厂** | _create_adapter, SkillSource ABC | 集中对象创建 |
| **策略** | SessionResetPolicy (daily/idle/both/none) | 可互换算法 |
| **代理** | delegate_tool 子代理隔离 | 访问控制 |
| **断路器** | MCP 重连（最多5次）、provider fallback | 容错与优雅降级 |
| **渐进式披露** | Skills (分层加载)、System prompt (7层) | Token 优化 |
| **双存储** | SQLite + JSONL、进程内 LRU + 磁盘快照 | 优雅降级和持久化 |
| **隔离安装** | skills_guard 安装流程 | 安全执行不受信内容 |

---

## 5. 数据流

### 典型对话流程

```
1. 用户输入
   CLI: prompt_toolkit → HermesCLI.process_command()
   Gateway: PlatformAdapter.handle_message() → GatewayRunner._handle_message()

2. 会话解析
   SessionStore.get_or_create_session() → SessionDB
   → 从 SQLite/JSONL 加载历史

3. Agent 执行
   AIAgent.run_conversation():
   a) 构建系统提示词（7层，缓存）
   b) 准备消息（历史 + 用户输入）
   c) API 调用 LLM
   d) 如果有 tool_calls:
      - 分析独立性决定并行/串行
      - 执行每个工具 → registry.dispatch()
      - 结果追加到消息 → 回到步骤 c
   e) 如果是文本响应：返回

4. 工具执行（示例：terminal_tool）
   terminal_tool() → _create_environment() → BaseEnvironment.execute()
   → _run_bash() 后端特定实现
   → 快照重载、CWD 跟踪、中断处理

5. 上下文压缩（token 预算超限时）
   ContextCompressor.should_compress() → compress():
   - 修剪旧工具结果
   - 保护头尾
   - 通过辅助 LLM 摘要中间部分

6. 会话持久化
   AIAgent.on_session_end() → SessionDB.append_message()
   → WAL 提交（抖动重试）

7. Gateway 响应
   GatewayRunner._send_response() → PlatformAdapter.send()
   → 平台特定格式化（如 Telegram MarkdownV2）
```

---

## 6. 安全模型

| 层级 | 实现 |
|------|------|
| **危险命令检测** | `tools/approval.py` 的 `check_dangerous_command` |
| **审批回调** | CLI 交互式审批、ACP `make_approval_callback` |
| **写保护** | `WRITE_DENIED_PATHS` 保护 `~/.ssh/`、`/etc/sudoers` |
| **Docker 加固** | `cap-drop ALL`、`no-new-privileges`、`pids-limit 256` |
| **子代理隔离** | 受限工具集（无 delegate_task、memory、execute_code） |
| **MCP OAuth 2.1 PKCE** | HTTP MCP 服务器的身份验证 |
| **凭证清理** | 错误消息中过滤 `ghp_*/sk-*/Bearer/token=` |
| **技能安全扫描** | THREAT_PATTERNS 检测 exfiltration/injection/destructive |
| **PII 处理** | WhatsApp/Signal/Telegram 使用 ID 哈希 |

---

## 7. 学习路径

对于代码库新手，推荐阅读顺序：

1. **`run_agent.py`** (AIAgent) — 理解核心工具调用循环
2. **`tools/registry.py`** (ToolRegistry) — 理解工具注册和分发
3. **`hermes_state.py`** (SessionDB) — 理解会话持久化
4. **`agent/prompt_builder.py`** — 理解系统提示词组装
5. **`agent/context_compressor.py`** — 理解上下文压缩
6. **`hermes_cli/main.py`** (HermesCLI) — 理解 CLI 命令处理
7. **`gateway/run.py`** (GatewayRunner) — 理解多平台消息路由
8. **`tools/terminal_tool.py`** — 理解环境抽象
9. **`tools/delegate_tool.py`** — 理解子代理生成
10. **`tools/mcp_tool.py`** — 理解 MCP 服务器集成
11. **`tools/skills_hub.py`** — 理解技能发现和安装
12. **`tools/environments/`** — 理解多后端执行抽象
13. **`mcp_serve.py` / `acp_adapter/`** — 理解外部客户端集成

---

## 8. 复杂区域

### 需要特别注意的区域

1. **上下文压缩边界对齐** (`context_compressor.py`)
   - `_align_boundary_forward/backward` 确保压缩边界不拆分 tool_call/result 对
   - 1.5x 软上限是经验性的 workaround

2. **并行工具路径冲突检测** (`run_agent.py`)
   - `path.parts` 比较在符号链接、大小写敏感混合时有边缘情况

3. **MCP 异步/同步桥接** (`mcp_tool.py`)
   - `_AsyncWorker` 在后台线程运行自己的 asyncio 事件循环

4. **SSH ControlMaster 生命周期** (`environments/ssh.py`)
   - 错误路径可能泄漏 socket 文件

5. **SessionDB 写入竞争** (`hermes_state.py`)
   - WAL 模式的抖动重试需要仔细分析正确性

6. **MCP Sampling 消息转换** (`mcp_tool.py`)
   - `SamplingHandler` 的 `toolUseId` 配对和 `structuredContent` 处理有多个边缘情况

7. **环境清理竞争条件** (`terminal_tool.py`)
   - Phase 1 在锁内标记清理，Phase 2 在锁外执行

8. **模型元数据解析链** (`model_metadata.py`)
   - 10 步解析涉及 provider 推断、URL 解析、模糊匹配

---

## 9. 跨领域关注点

### 各模块的相似模式

**缓存策略:**
- Skills: 进程内 LRU + 磁盘快照（mtime 验证）
- 模型元数据: 内存 TTL + 持久 YAML 文件
- SessionDB: WAL 模式 + 应用级重试
- EventBridge: mtime 优化 + DB 轮询

**后台任务模式:**
- Gateway: `_session_expiry_watcher`、`_platform_reconnect_watcher`
- Terminal: `_cleanup_inactive_envs` 守护线程
- MCP: `_mcp_loop` 后台 asyncio 事件循环
- Modal: `_AsyncWorker` 后台线程 + 事件循环

**并发原语:**
- 线程本地循环: `_tool_loop`、`_worker_thread_local`
- 线程安全计数器: `IterationBudget`
- 锁: `_env_lock`、`_creation_locks`

**配置加载:**
- 分层合并: 用户配置 + 默认配置
- 环境变量展开: `${VAR}` 语法
- 版本迁移: 增量迁移步骤

**错误处理:**
- Fast Fail: 依赖不可用时提前返回错误字典
- 优雅降级: 双存储、provider fallback 链
- 冷却机制: `_SUMMARY_FAILURE_COOLDOWN_SECONDS=600`、重连指数退避

---

*报告由 8 个并行探索子 agent 生成，涵盖 8 个功能域。详细实现细节请参考各域提供的文件路径和行号。*
