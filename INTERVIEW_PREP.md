# OpenAI Codex CLI 面试深度准备指南

> 本文档为 LLM 算法实习面试准备，聚焦于 LLM & Agent、应用 & 后训练、Code Agent 方向的深度考察点。

---

## 目录

1. [项目概述与定位](#1-项目概述与定位)
2. [系统架构全景](#2-系统架构全景)
3. [LLM & Agent 核心考察点](#3-llm--agent-核心考察点)
4. [工具系统深度剖析](#4-工具系统深度剖析)
5. [上下文管理与 Prompt 工程](#5-上下文管理与-prompt-工程)
6. [多 Agent 系统设计](#6-多-agent-系统设计)
7. [沙箱与安全机制](#7-沙箱与安全机制)
8. [MCP 协议与扩展性](#8-mcp-协议与扩展性)
9. [流式传输与性能优化](#9-流式传输与性能优化)
10. [面试深挖问题集](#10-面试深挖问题集)
11. [如何介绍这个项目](#11-如何介绍这个项目)
12. [进阶思考题](#12-进阶思考题)

---

## 1. 项目概述与定位

### 是什么

OpenAI Codex CLI 是 OpenAI 开源的**本地运行编程代理**，是 ChatGPT Codex Web 产品的本地版本。它允许 LLM 直接在用户电脑上：

- 读写本地文件系统
- 执行 Shell 命令（沙箱化）
- 搜索代码（通过 `rg`）
- 编辑文件（通过 `apply_patch`）
- 连接 MCP 服务器扩展能力
- 管理多轮对话，支持会话持久化与恢复

### 技术栈

| 维度 | 技术选型 |
|------|----------|
| 主语言 | Rust (2024 edition)，约 112 个 crate 的 workspace |
| 异步运行时 | Tokio (多线程) |
| TUI 框架 | ratatui + crossterm |
| CLI 解析 | clap |
| 数据库 | SQLite (通过 SQLx) |
| 流式传输 | SSE + WebSocket |
| 沙箱 | macOS Seatbelt / Linux Landlock + bwrap / Windows 自定义 |
| 构建 | Cargo + Bazel 双构建系统 |

### 面试可讲的点

> "这个项目用 Rust 从零构建了一个生产级的 Code Agent 系统，核心挑战在于：如何在保证安全性的前提下，让 LLM 高效地与本地环境交互，同时管理越来越长的上下文和复杂的多 Agent 协作。"

---

## 2. 系统架构全景

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户界面层                                │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│   │   CLI    │   │   TUI    │   │ VS Code  │   │  SDKs    │    │
│   │ (clap)   │   │(ratatui) │   │ Extension│   │(TS/Py)   │    │
│   └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘    │
│        │              │              │              │           │
│        └──────────────┴──────────────┴──────────────┘           │
│                              │                                   │
│                    ┌─────────▼──────────┐                       │
│                    │   App Server       │                       │
│                    │  (JSON-RPC 2.0)    │                       │
│                    │  stdio/ws/unix     │                       │
│                    └─────────┬──────────┘                       │
├─────────────────────────────┼───────────────────────────────────┤
│                        核心引擎层                                │
│                    ┌─────────▼──────────┐                       │
│                    │     Session        │                       │
│                    │  ┌───────────────┐ │                       │
│                    │  │  Turn Loop    │ │                       │
│                    │  │  ┌─────────┐  │ │                       │
│                    │  │  │ Context │  │ │                       │
│                    │  │  │ Builder │  │ │                       │
│                    │  │  └────┬────┘  │ │                       │
│                    │  │       ▼       │ │                       │
│                    │  │  ┌─────────┐  │ │                       │
│                    │  │  │  Model  │  │ │                       │
│                    │  │  │ Client  │  │ │                       │
│                    │  │  └────┬────┘  │ │                       │
│                    │  │       ▼       │ │                       │
│                    │  │  ┌─────────┐  │ │                       │
│                    │  │  │  Tool   │  │ │                       │
│                    │  │  │ Runtime │  │ │                       │
│                    │  │  └─────────┘  │ │                       │
│                    │  └───────────────┘ │                       │
│                    └─────────┬──────────┘                       │
│                              │                                   │
├─────────────────────────────┼───────────────────────────────────┤
│                        工具与沙箱层                               │
│        ┌────────────────────┼────────────────────┐              │
│        ▼                    ▼                    ▼              │
│  ┌──────────┐        ┌──────────┐        ┌──────────┐          │
│  │  Shell   │        │apply_patch│       │MCP Tools │          │
│  │ Handler  │        │ Handler   │       │ Handler  │          │
│  └────┬─────┘        └────┬─────┘       └────┬─────┘          │
│       ▼                   ▼                  ▼                  │
│  ┌─────────────────────────────────────────────────┐           │
│  │              Sandbox Orchestrator               │           │
│  │   macOS: Seatbelt | Linux: Landlock+bwrap       │           │
│  │   Windows: Restricted Token | External Sandbox  │           │
│  └─────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### 核心模块职责

| 模块 | 位置 | 职责 |
|------|------|------|
| Session | `core/src/session/` | 主会话对象，编排整个 Agent 循环 |
| ModelClient | `core/src/client.rs` | 管理与 OpenAI API 的通信 |
| ToolRouter | `core/src/tools/router.rs` | 工具分发与路由 |
| ToolOrchestrator | `core/src/tools/orchestrator.rs` | 审批→沙箱→执行→重试流水线 |
| ContextManager | `core/src/context/` | 上下文构建与管理 |
| AgentControl | `core/src/agent/` | 多 Agent 编排与控制 |
| Sandbox | `core/src/sandboxing/` | 跨平台沙箱抽象 |

---

## 3. LLM & Agent 核心考察点

### 3.1 Agent Loop（核心循环）

**这是整个系统最核心的设计，面试必问。**

```rust
// 简化的 Agent Loop 伪代码
loop {
    // 1. 构建上下文（历史 + 工具定义 + 系统提示 + 技能注入）
    let prompt = build_prompt(history, tools, instructions, context_fragments);

    // 2. 调用 LLM（流式响应）
    let response = call_model(prompt).await;

    // 3. 处理响应
    match response {
        // 模型请求工具调用
        ToolCalls(calls) => {
            // 3a. 并行执行工具调用
            let results = execute_tools_in_parallel(calls).await;
            // 3b. 将结果追加到历史
            history.extend(results);
            // 3c. 继续循环，让模型处理工具结果
            continue;
        }
        // 模型给出最终回答
        AssistantMessage(msg) => {
            // 检查 stop hooks（可能强制继续）
            if run_stop_hooks().should_stop {
                break;
            }
        }
    }

    // 4. 如果上下文接近 token 限制，自动压缩
    if context_near_limit() {
        run_auto_compact().await;
    }
}
```

**面试深挖点：**

- **Q: 为什么用循环而不是一次性调用？**
  - A: LLM 需要多轮工具调用才能完成复杂任务。每轮工具结果都需要反馈给模型，让它决定下一步。这是 ReAct 模式的经典实现。

- **Q: 循环什么时候终止？**
  - A: 三种情况：(1) 模型返回纯文本消息（无工具调用）；(2) 用户中断；(3) 达到 token 限制且无法压缩。

- **Q: 如何处理工具执行失败？**
  - A: 工具执行错误会作为 `FunctionCallError` 返回给模型，让模型自己决定如何处理（重试、换方案、报告错误）。这是**让 LLM 自主处理错误**的设计哲学。

### 3.2 Tokenizer-Free 设计

这个项目**没有使用 tokenizer**。上下文管理的策略是：

1. **分片限制**：每个上下文片段（Context Fragment）单独限制在 10K tokens 以内
2. **增量构建**：上下文只增不减，避免 cache miss
3. **自动压缩**：当接近上下文窗口限制时，运行 `auto_compact` 压缩历史

**面试深挖点：**

- **Q: 不用 tokenizer 怎么判断 token 数量？**
  - A: 依赖模型 API 返回的 `usage` 字段（input_tokens / output_tokens）。在发送请求前，通过历史记录累计的 token 使用量来估算。

- **Q: 为什么不自己做 tokenization？**
  - A: (1) 不同模型的 tokenizer 不同；(2) 模型 API 是 ground truth；(3) 避免维护 tokenizer 的额外复杂度。

### 3.3 Streaming 与增量优化

系统支持两种传输方式：

```
WebSocket (首选)          HTTP SSE (降级)
     │                        │
     ▼                        ▼
┌─────────────┐         ┌─────────────┐
│ OpenAI WS   │         │ OpenAI HTTP │
│ responses   │         │ responses   │
│ websocket   │         │ streaming   │
└──────┬──────┘         └──────┬──────┘
       │                       │
       ▼                       ▼
  增量发送优化            全量发送
  (只发 delta)          (每次都完整)
```

**WebSocket 增量优化的核心逻辑：**

```rust
fn get_incremental_items(&self, request, last_response) -> Option<Vec<ResponseItem>> {
    // 检查是否是同一个会话的连续请求
    if !properties_match(previous, current) { return None; }

    // 检查当前输入是否是前一次的严格扩展
    let delta = current_input.strip_prefix(previous_input)?;

    // 只发送新增的部分
    Some(delta.to_vec())
}
```

**面试深挖点：**

- **Q: 为什么要做增量优化？**
  - A: 减少网络传输量，降低延迟。在多轮工具调用场景下，每次只需要发送新增的工具结果，而不是整个历史。

- **Q: WebSocket 和 SSE 的选择策略？**
  - A: 优先 WebSocket（支持增量、更低延迟），失败时降级到 SSE。一旦降级，后续所有请求都用 SSE（session-scoped 决策）。

---

## 4. 工具系统深度剖析

### 4.1 工具架构

```
ToolRouter
    │
    ├── CoreToolRuntime (trait)
    │   ├── ShellToolRuntime      # 执行 shell 命令
    │   ├── ApplyPatchRuntime     # 编辑文件
    │   ├── McpToolRuntime        # MCP 外部工具
    │   ├── SpawnAgentRuntime     # 生成子 Agent
    │   └── ...
    │
    └── ToolOrchestrator
        ├── Phase 1: Approval     # 用户/策略审批
        ├── Phase 2: Execute      # 沙箱化执行
        └── Phase 3: Retry        # 沙箱拒绝后升级重试
```

### 4.2 工具执行流水线（Orchestrator Pattern）

这是整个工具系统最精妙的设计：

```rust
pub async fn run(&mut self, tool, request, ...) -> Result<ToolOutput> {
    // Phase 1: 审批检查
    let requirement = tool.exec_approval_requirement(&request);
    match requirement {
        Skip => { /* 自动批准 */ }
        NeedsApproval => {
            let decision = request_approval(...).await?;
            if decision != Approved { return Err(Rejected); }
        }
        Forbidden => { return Err(Rejected(reason)); }
    }

    // Phase 2: 沙箱化执行
    let sandbox = select_sandbox(policy);
    let result = tool.execute(request, &sandbox).await;

    match result {
        Ok(output) => return Ok(output),
        Err(SandboxDenied) if tool.supports_escalation() => {
            // Phase 3: 升级重试（移除沙箱）
            let approval = request_unsandboxed_approval().await?;
            if approval == Approved {
                return tool.execute(request, &Sandbox::None).await;
            }
        }
        Err(e) => return Err(e),
    }
}
```

**面试深挖点：**

- **Q: 为什么需要三个阶段？**
  - A: 安全性与灵活性的平衡。默认沙箱化执行保证安全；当沙箱限制了合法操作时，通过审批机制升级权限。这是**最小权限原则**的体现。

- **Q: 审批机制有哪些模式？**
  - A: (1) 自动批准（信任的操作）；(2) 用户审批（TUI 中弹出确认）；(3) Guardian 审批（子 Agent 自动审查）；(4) 策略引擎审批（基于规则）。

### 4.3 Shell 命令执行

```rust
// 简化的 Shell 执行流程
async fn execute_shell_command(command: &str, cwd: &Path) -> Result<ExecOutput> {
    // 1. 检测 shell 类型（bash/zsh/powershell/cmd）
    let shell = detect_shell();

    // 2. 构建执行请求
    let request = ExecRequest {
        command: shell.wrap_command(command),
        cwd,
        env: collect_env(),
        timeout: Duration::from_secs(60),
        sandbox: select_sandbox(),
    };

    // 3. 通过 PTY 执行
    let child = spawn_with_pty(request).await?;

    // 4. 流式收集输出（有大小限制）
    let output = consume_output(child, MAX_OUTPUT_BYTES).await;

    // 5. 处理超时和取消
    tokio::select! {
        result = child.wait() => { /* 正常退出 */ }
        _ = timeout => { /* 超时，kill 进程 */ }
        _ = ctrl_c => { /* 用户中断 */ }
    }
}
```

**面试深挖点：**

- **Q: 为什么用 PTY 而不是普通的 subprocess？**
  - A: PTY 支持颜色输出、交互式程序、正确的信号传播。很多工具（如 npm、git）会检测是否在 TTY 中运行，PTY 能获得更完整的输出。

- **Q: 输出大小限制是怎么做的？**
  - A: 有 `EXEC_OUTPUT_MAX_BYTES` 常量限制。超出时截断并告知模型输出被截断。避免超大输出撑爆上下文。

### 4.4 apply_patch 工具

这是一个**自由格式工具**（Freeform Tool），使用 Lark 语法定义，而非 JSON：

```rust
// 模型输出的格式（非 JSON，是结构化文本）
/*
*** Begin Patch
*** Update File: src/main.rs
@@ -10, +3 @@
- old line 1
- old line 2
+ new line 1
+ new line 2
+ new line 3
*** End Patch
*/
```

**面试深挖点：**

- **Q: 为什么 apply_patch 用自由格式而不是 JSON？**
  - A: (1) 对于文件编辑这种结构化 diff，Lark 语法比 JSON 更紧凑；(2) 减少 token 消耗；(3) 更容易让模型学习和输出。

- **Q: 安全性如何保证？**
  - A: `assess_patch_safety()` 检查：(1) 修改的文件是否在允许的目录；(2) 是否会覆盖敏感文件；(3) 根据沙箱策略决定自动批准还是询问用户。

---

## 5. 上下文管理与 Prompt 工程

### 5.1 Context Fragment 系统

上下文被拆分为**独立的、有界的片段**，每个片段实现 `ContextualUserFragment` trait：

```rust
trait ContextualUserFragment {
    // 返回该片段的内容（作为 user/developer 消息）
    fn content(&self) -> String;

    // 该片段的 token 预算（硬限制 10K）
    fn token_budget(&self) -> usize;

    // 是否应该注入（可能根据条件动态决定）
    fn should_include(&self) -> bool;
}
```

**核心片段类型：**

| 片段 | 作用 | 内容示例 |
|------|------|----------|
| `EnvironmentContext` | 环境信息 | CWD、git 分支、OS、shell |
| `PermissionsInstructions` | 权限说明 | 沙箱策略、审批规则 |
| `UserInstructions` | 用户自定义 | AGENTS.md 内容 |
| `AvailableSkillsInstructions` | 可用技能 | 技能列表 |
| `PersonalitySpecInstructions` | 个性设定 | 响应风格偏好 |
| `CurrentTimeReminder` | 时间提醒 | 当前时间 |
| `TokenBudgetContext` | Token 预算 | 剩余 token 数 |

**面试深挖点：**

- **Q: 为什么要拆分成片段而不是一个大 prompt？**
  - A: (1) **可组合性**：不同场景可以灵活组合不同片段；(2) **可测试性**：每个片段可以独立测试；(3) **缓存友好**：片段变化频率不同，拆分后可以最大化 prompt cache 命中。

- **Q: 如何保证不超过上下文窗口？**
  - A: (1) 每个片段有硬限制（10K tokens）；(2) 累计接近限制时触发 auto-compact；(3) auto-compact 会压缩历史对话。

### 5.2 System Prompt 设计

系统提示的核心指令（简化版）：

```markdown
You are Codex, based on GPT-5. You are running as a coding agent in the
Codex CLI on a user's computer.

## Tool Preferences
- Prefer `rg` over `grep` for searching
- Use `apply_patch` for single-file edits
- Use `shell` for multi-file operations or commands

## Git Safety
- NEVER revert existing changes unless explicitly asked
- NEVER run `git reset --hard` without user request
- Always create new branches for experiments

## Response Format
- Plain text with GitHub-flavored Markdown
- Use backtick paths: `src/main.rs`
- Flat lists, no nested hierarchies

## Plan Tool
- Skip for trivial tasks (single step)
- No single-step plans
- Update plans as you progress
```

**面试深挖点：**

- **Q: 为什么要在 prompt 中强调 "Prefer rg over grep"？**
  - A: `rg` (ripgrep) 比 `grep` 快很多，且支持 `.gitignore` 过滤。在代码搜索场景下，性能差异巨大。这种**工具偏好指令**能显著影响模型的行为。

- **Q: Git Safety 指令为什么重要？**
  - A: Agent 有文件系统写权限，如果模型随意 `git reset --hard`，可能丢失用户工作。这是**安全护栏**的体现。

### 5.3 自动压缩（Auto-Compact）

当上下文接近 token 限制时，系统会自动压缩：

```rust
async fn run_auto_compact(session: &Session, ...) -> Result<()> {
    // 1. 获取当前历史
    let history = session.clone_history().await;

    // 2. 计算需要压缩的量
    let tokens_to_remove = calculate_compact_target(usage, limit);

    // 3. 执行压缩（通过 LLM 总结旧对话）
    let summary = compact_history(history, tokens_to_remove).await?;

    // 4. 替换历史
    session.replace_history(summary).await;

    Ok(())
}
```

**面试深挖点：**

- **Q: 压缩策略是什么？**
  - A: 有两种策略：(1) **本地压缩**：简单的截断和摘要；(2) **远程压缩**：调用 LLM 生成历史摘要（更智能但更贵）。

- **Q: 压缩会丢失信息吗？**
  - A: 会。但系统设计为**关键信息应该在当前上下文中**，而不是依赖很久以前的历史。AGENTS.md 等持久信息会重新注入。

---

## 6. 多 Agent 系统设计

### 6.1 Agent 树结构

```
Root Agent (用户交互)
    │
    ├── Sub-Agent: Code Reviewer
    │   └── (审查代码变更)
    │
    ├── Sub-Agent: Guardian
    │   └── (自动审批危险操作)
    │
    └── Sub-Agent: Task Worker
        └── (执行特定任务)
```

### 6.2 Agent 生命周期

```rust
// 生成子 Agent
async fn spawn_agent(config: AgentConfig, task: String) -> Result<AgentId> {
    // 1. 检查深度限制
    if depth >= MAX_AGENT_DEPTH {
        return Err("Agent depth limit reached");
    }

    // 2. 继承父 Agent 配置
    let mut config = parent_config.clone();
    apply_role_overrides(&mut config, role);

    // 3. 创建新 Session
    let child_session = Session::new(config);

    // 4. 注册到 AgentRegistry
    agent_registry.register(child_session.id(), task);

    // 5. 启动执行
    tokio::spawn(async move {
        child_session.run(task).await;
        // 完成后通知父 Agent
        notify_parent_on_completion();
    });

    Ok(child_session.id())
}
```

### 6.3 Agent 间通信

```rust
// 通信方式
enum InterAgentCommunication {
    // 父 → 子
    SendInput { agent_id, message },
    Interrupt { agent_id },

    // 子 → 父
    Completion { agent_id, result },
    StatusUpdate { agent_id, status },
    RequestApproval { agent_id, operation },
}
```

**面试深挖点：**

- **Q: 为什么需要多 Agent？**
  - A: (1) **关注点分离**：不同 Agent 专注不同任务；(2) **并行执行**：多个任务可以同时进行；(3) **安全审查**：Guardian Agent 可以自动审查危险操作。

- **Q: 如何避免 Agent 无限递归？**
  - A: 通过 `agent_max_depth` 限制嵌套深度。超出限制时返回错误，让当前 Agent 自己解决。

- **Q: 子 Agent 的上下文从哪来？**
  - A: 继承父 Agent 的配置，但有独立的对话历史。可以通过 `fork_context` 参数继承部分上下文。

---

## 7. 沙箱与安全机制

### 7.1 多平台沙箱

| 平台 | 沙箱技术 | 特点 |
|------|----------|------|
| macOS | Seatbelt (`sandbox-exec`) | 基于 `.sbpl` 策略文件 |
| Linux | Landlock + bwrap | 内核级访问控制 |
| Windows | Restricted Token | 进程级权限限制 |

### 7.2 沙箱策略

```rust
enum SandboxPolicy {
    ReadOnly,           // 只读访问
    WorkspaceWrite,     // 工作区可写
    FullAccess,         // 完全访问（需审批）
    External(String),   // 外部沙箱（如 Docker）
}
```

### 7.3 网络控制

```rust
// 网络访问策略
struct NetworkSandboxPolicy {
    allow_outbound: bool,      // 允许出站
    allow_inbound: bool,       // 允许入站
    allowed_hosts: Vec<String>, // 允许的主机
}
```

**面试深挖点：**

- **Q: 为什么需要多层沙箱？**
  - A: **纵深防御**。即使一层被突破，还有其他层。例如：(1) 操作系统级沙箱限制文件访问；(2) 应用级策略控制命令执行；(3) 审批机制控制高危操作。

- **Q: 沙箱拒绝后怎么办？**
  - A: Orchestrator 的 Phase 3 处理：请求用户批准后，可以降级执行（移除沙箱限制）。这是**安全与可用性的权衡**。

---

## 8. MCP 协议与扩展性

### 8.1 MCP (Model Context Protocol) 概述

MCP 是 Anthropic 提出的标准协议，允许 LLM 调用外部工具。Codex 同时支持：
- **作为 MCP Client**：连接外部 MCP 服务器，使用其提供的工具
- **作为 MCP Server**：将自身能力暴露给其他 MCP 客户端

### 8.2 MCP 工具集成

```rust
// MCP 工具发现与注册
async fn discover_mcp_tools(server: &McpServer) -> Vec<ToolSpec> {
    // 1. 连接 MCP 服务器（stdio 或 HTTP）
    let connection = connect_mcp(server).await?;

    // 2. 列出可用工具
    let tools = connection.list_tools().await?;

    // 3. 转换为本地 ToolSpec
    tools.into_iter().map(|t| ToolSpec {
        name: format!("mcp/{}/{}", server.name, t.name),
        description: t.description,
        parameters: t.input_schema,
    }).collect()
}

// MCP 工具调用
async fn call_mcp_tool(tool: &McpTool, args: Value) -> Result<ToolOutput> {
    let connection = get_mcp_connection(&tool.server).await?;
    connection.call_tool(&tool.name, args).await
}
```

**面试深挖点：**

- **Q: MCP 与 Function Calling 有什么区别？**
  - A: Function Calling 是模型 API 的一部分，定义在请求中；MCP 是独立的协议，工具可以在运行时动态发现和注册。MCP 更像"工具即服务"。

- **Q: 如何处理 MCP 服务器不可用？**
  - A: (1) 连接失败时记录错误，不注册该服务器的工具；(2) 工具调用失败时返回错误给模型，让它决定是否重试或换方案。

---

## 9. 流式传输与性能优化

### 9.1 双传输策略

```
首选: WebSocket
├── 支持增量发送
├── 更低延迟
├── 双向通信
└── 失败时降级

降级: HTTP SSE
├── 单向流式
├── 更好的兼容性
└── Session-scoped 决策
```

### 9.2 并行工具执行

```rust
// 使用 RwLock 实现并行控制
pub struct ToolCallRuntime {
    parallel_execution: Arc<RwLock<()>>,
}

// 并行安全的工具用 read lock（多个可同时执行）
// 非并行安全的工具用 write lock（独占）
let _guard = if supports_parallel {
    lock.read().await    // 共享：多个工具可并行
} else {
    lock.write().await   // 独占：阻塞其他工具
};
```

**面试深挖点：**

- **Q: 哪些工具可以并行？哪些不能？**
  - A: 只读操作（搜索、读文件）可以并行；写操作（编辑文件、执行命令）需要串行，避免竞态条件。

- **Q: 如何处理并行工具的取消？**
  - A: 使用 `tokio::select!` 在工具执行和取消令牌之间选择。取消时发送 SIGTERM 给子进程。

### 9.3 Prompt Cache 优化

```rust
// Prompt Cache Key 的生成
fn prompt_cache_key(&self) -> String {
    // 基于：模型 + 用户 + 项目
    format!("{}:{}:{}", model, user_id, project_path)
}
```

**面试深挖点：**

- **Q: 什么是 Prompt Cache？**
  - A: OpenAI API 的优化功能。如果连续请求的前缀相同，可以复用计算结果，显著降低延迟和成本。

- **Q: 如何最大化 cache 命中？**
  - A: (1) 上下文增量构建，不修改历史；(2) 系统提示保持稳定；(3) 使用固定的 cache key。

---

## 10. 面试深挖问题集

### 10.1 Agent 设计类

**Q1: 设计一个 Code Agent 的核心循环，需要考虑哪些因素？**

参考答案要点：
1. **循环 vs 单次调用**：复杂任务需要多轮工具调用，必须是循环
2. **终止条件**：模型返回最终回答 / 用户中断 / token 限制
3. **错误处理**：工具错误应该反馈给模型，让 LLM 自主处理
4. **上下文管理**：长对话需要压缩机制
5. **并行执行**：多个工具可以并行，提高效率
6. **安全控制**：危险操作需要审批

**Q2: 如何让 LLM 更好地使用工具？**

参考答案要点：
1. **清晰的工具描述**：模型需要理解每个工具的用途和参数
2. **工具偏好指令**：在 prompt 中说明首选工具（如 rg > grep）
3. **错误反馈**：工具执行失败时，返回详细错误信息
4. **示例演示**：在 prompt 中提供工具使用的例子
5. **工具组合**：支持工具之间的数据传递

**Q3: 如何设计一个安全的 Agent 系统？**

参考答案要点：
1. **最小权限原则**：默认沙箱化，限制访问
2. **多层防御**：OS 沙箱 + 应用策略 + 审批机制
3. **危险操作识别**：自动识别高危命令（rm -rf、git reset --hard）
4. **用户确认**：关键操作需要用户明确批准
5. **审计日志**：记录所有操作，便于回溯

### 10.2 工程实现类

**Q4: Rust 的所有权系统在这个项目中有什么优势？**

参考答案要点：
1. **内存安全**：无需 GC，无数据竞争
2. **并发安全**：编译期保证线程安全
3. **零成本抽象**：trait object 实现多态，无运行时开销
4. **生命周期管理**：Session、Client 等资源的自动管理

**Q5: 如何处理 LLM API 的流式响应？**

参考答案要点：
1. **SSE 解析**：逐行解析 Server-Sent Events
2. **增量处理**：边接收边处理，不等完整响应
3. **取消支持**：用户中断时立即停止
4. **错误重试**：网络错误时自动重试
5. **传输降级**：WebSocket 失败时降级到 SSE

**Q6: 如何设计一个可扩展的工具系统？**

参考答案要点：
1. **Trait 抽象**：`CoreToolRuntime` trait 定义统一接口
2. **注册机制**：工具在启动时注册到 `ToolRegistry`
3. **动态发现**：MCP 工具可以在运行时发现和注册
4. **生命周期钩子**：Pre/Post hooks 支持审计、日志等
5. **并行控制**：RwLock 控制工具的并行/串行执行

### 10.3 Prompt 工程类

**Q7: 如何设计一个好的 System Prompt？**

参考答案要点：
1. **角色定义**：明确 AI 的身份和能力边界
2. **工具说明**：详细说明每个工具的用途和使用场景
3. **行为准则**：安全规则、输出格式、错误处理
4. **示例演示**：提供典型的交互示例
5. **约束条件**：明确禁止的行为

**Q8: 如何处理长上下文？**

参考答案要点：
1. **分片管理**：将上下文拆分为独立片段
2. **大小限制**：每个片段有硬限制（10K tokens）
3. **自动压缩**：接近限制时压缩历史
4. **增量构建**：只增不减，最大化 cache 命中
5. **持久化**：关键信息存储在文件中，需要时重新注入

---

## 11. 如何介绍这个项目

### 11.1 一分钟版本

> "我研究了 OpenAI 开源的 Codex CLI 项目，这是一个用 Rust 实现的本地 Code Agent。核心是一个 ReAct 循环：LLM 接收用户请求，决定调用什么工具（Shell、文件编辑、MCP），执行后将结果反馈给模型，循环直到完成任务。
>
> 技术亮点包括：(1) 多层沙箱安全机制，支持 macOS/Linux/Windows；(2) 可扩展的工具系统，支持 MCP 协议动态注册工具；(3) 多 Agent 协作，可以生成子 Agent 进行代码审查或任务执行；(4) 上下文自动压缩，支持长对话。
>
> 这个项目让我深入理解了生产级 Agent 系统的设计，特别是安全性和可扩展性的平衡。"

### 11.2 三分钟版本（技术深挖）

> "这个项目的核心是 `core/src/session/turn.rs` 中的 Agent Loop。每次循环包含：上下文构建、模型调用、工具执行三个阶段。
>
> **上下文管理**采用了 Context Fragment 设计，将系统提示拆分为环境信息、权限说明、用户指令等独立片段，每个片段限制在 10K tokens 以内。这样做的好处是可组合、可测试、且最大化 Prompt Cache 命中。
>
> **工具系统**采用了三层架构：ToolRouter 负责分发，CoreToolRuntime trait 定义接口，ToolOrchestrator 管理审批-沙箱-执行-重试的完整流水线。特别的是 apply_patch 工具使用了 Lark 语法的自由格式，比 JSON 更紧凑。
>
> **安全机制**是多层的：OS 级沙箱（macOS Seatbelt、Linux Landlock）、应用级策略引擎、用户/Guardian 审批。当沙箱限制了合法操作时，可以通过审批升级权限。
>
> **多 Agent 系统**支持生成子 Agent，通过 AgentControl 编排，AgentRegistry 跟踪状态。子 Agent 可以用于代码审查（Reviewer）、安全审查（Guardian）、任务执行（Worker）等场景。
>
> **流式传输**支持 WebSocket 和 HTTP SSE 双模式，WebSocket 支持增量发送优化，失败时自动降级。"

### 11.3 关键数字记忆

| 指标 | 数值 |
|------|------|
| Rust crate 数量 | ~112 |
| 单个上下文片段限制 | 10K tokens |
| 最大 Agent 嵌套深度 | 可配置（默认 3-5） |
| Shell 输出最大大小 | 有常量限制 |
| 工具并行控制 | RwLock |
| 传输方式 | WebSocket + HTTP SSE |

---

## 12. 进阶思考题

### 12.1 如果让你改进这个系统

1. **更好的压缩策略**：当前压缩可能丢失信息，能否用知识图谱保留关键信息？
2. **工具学习**：能否让 Agent 从历史中学习工具使用的最佳实践？
3. **多模态支持**：如何扩展支持图像、音频等模态？
4. **分布式执行**：如何将工具执行分布到多台机器？
5. **自适应沙箱**：能否根据任务类型自动调整沙箱策略？

### 12.2 如果让你设计一个新的 Code Agent

1. **架构选择**：单 Agent vs 多 Agent？为什么？
2. **工具设计**：需要哪些核心工具？如何设计工具接口？
3. **安全模型**：如何平衡安全性和可用性？
4. **上下文管理**：如何处理超长代码库？
5. **评估指标**：如何衡量 Code Agent 的效果？

### 12.3 LLM 应用的工程挑战

1. **幻觉问题**：LLM 生成的代码可能有 bug，如何处理？
2. **延迟优化**：如何降低 Agent 的响应延迟？
3. **成本控制**：如何优化 token 使用，降低 API 成本？
4. **用户体验**：如何让用户信任 Agent 的操作？
5. **错误恢复**：Agent 执行失败时如何优雅恢复？

---

## 附录：关键代码位置索引

| 功能 | 文件路径 | 行数参考 |
|------|----------|----------|
| Agent Loop | `codex-rs/core/src/session/turn.rs` | `run_turn()` 函数 |
| 工具注册 | `codex-rs/core/src/tools/registry.rs` | `CoreToolRuntime` trait |
| 工具执行 | `codex-rs/core/src/tools/orchestrator.rs` | `run()` 方法 |
| Shell 执行 | `codex-rs/core/src/tools/handlers/shell.rs` | Shell 工具处理 |
| apply_patch | `codex-rs/core/src/tools/apply_patch_spec.rs` | 自由格式工具定义 |
| 上下文构建 | `codex-rs/core/src/context/` | 各个 Fragment 实现 |
| 多 Agent | `codex-rs/core/src/agent/` | AgentControl |
| 沙箱实现 | `codex-rs/core/src/sandboxing/` | 平台特定实现 |
| MCP 集成 | `codex-rs/codex-mcp/` | MCP 连接管理 |
| 流式传输 | `codex-rs/core/src/client.rs` | ModelClient |
| 系统提示 | `codex-rs/core/*.md` | Prompt 模板 |

---

*最后更新：2026-06-27*

*祝面试顺利！*
