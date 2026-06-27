# 技术面试五类问题深度应对指南

> 基于 OpenAI Codex CLI 项目，针对 LLM & Agent / Code Agent 方向的五类核心面试能力准备

---

## 目录

- [第一类：底层原理理解](#第一类底层原理理解)
- [第二类：实验和方案验证能力](#第二类实验和方案验证能力)
- [第三类：问题定位能力](#第三类问题定位能力)
- [第四类：工程落地能力](#第四类工程落地能力)
- [第五类：业务与实际场景理解](#第五类业务与实际场景理解)

---

# 第一类：底层原理理解

> **核心要求**：不是回答清楚概念，而是讲清楚方法解决什么问题、存在哪些局限性、有哪些改进方法

---

## Q1: Agent Loop 为什么设计成 ReAct 循环模式？

### 解决什么问题

单次 LLM 调用无法完成需要多步操作的任务。比如"修复这个 bug"需要：读代码 → 理解问题 → 编辑文件 → 运行测试 → 验证结果。每一步的输出都是下一步的输入。

### 为什么不用其他方案

| 方案 | 问题 |
|------|------|
| 单次调用 | 无法处理多步任务，模型无法看到中间结果 |
| 预定义 DAG | 任务路径固定，无法应对意外情况（如测试失败需要重新分析） |
| 纯 Chain-of-Thought | 只能推理，无法执行实际操作 |

ReAct 循环的核心优势：**让模型在每一步都能看到真实环境反馈，自主决定下一步行动**。

### 局限性

1. **延迟累积**：每次循环都需要一次 API 调用，多轮对话延迟线性增长
2. **Token 消耗**：每轮都要发送完整历史，成本随轮次指数增长
3. **错误放大**：早期错误可能被后续步骤放大（如错误的文件编辑导致后续所有操作基于错误代码）
4. **无限循环风险**：模型可能陷入重复尝试同一失败操作

### 改进方法

```
当前设计：
User → LLM → Tool → LLM → Tool → ... → Final Answer

可能的改进：
1. 并行 Agent：多个子 Agent 并行探索不同方案
2. 计划先行：先生成完整计划，再逐步执行（Plan-then-Execute）
3. 检查点机制：关键步骤前自动保存状态，失败时可回滚
4. 自适应终止：检测到循环时自动中断并报告
```

---

## Q2: Context Fragment 设计为什么采用分片而不是单一 Prompt？

### 解决什么问题

传统单一大 Prompt 的问题：
1. **缓存不友好**：任何修改都会导致整个 Prompt 失效
2. **不可测试**：无法独立验证每个部分
3. **不可组合**：不同场景需要不同组合

### 设计原理

```rust
trait ContextualUserFragment {
    fn content(&self) -> String;       // 内容
    fn token_budget(&self) -> usize;   // 硬限制 10K
    fn should_include(&self) -> bool;  // 条件注入
}
```

**关键约束**（来自 AGENTS.md）：
- 单个片段不超过 10K tokens
- 上下文只增不减（避免 cache miss）
- 所有片段必须实现 `ContextualUserFragment`

### 局限性

1. **信息碎片化**：跨片段的关联信息可能丢失
2. **组合爆炸**：片段数量增加时，组合策略变复杂
3. **预算分配**：如何公平分配有限的 token 预算？

### 改进思路

```rust
// 当前：静态预算
fn token_budget(&self) -> usize { 10_000 }

// 改进：动态预算，根据重要性和相关性调整
fn token_budget(&self, context: &ContextState) -> usize {
    let base = 10_000;
    let importance = self.importance_score(context);
    let remaining = context.remaining_budget();
    (base * importance).min(remaining)
}
```

---

## Q3: 为什么用 WebSocket + SSE 双传输而不是单一方案？

### 解决什么问题

| 传输方式 | 优势 | 劣势 |
|----------|------|------|
| WebSocket | 双向通信、增量发送、低延迟 | 连接管理复杂、某些环境不支持 |
| HTTP SSE | 简单可靠、兼容性好 | 单向、无法增量发送 |

### 设计决策

```rust
// Session-scoped 决策：一旦降级，整个 session 都用 HTTP
disable_websockets: AtomicBool
```

**为什么 Session-scoped？**
- 避免反复切换（churn）
- 一旦网络不稳定，保持 HTTP 更可靠
- 简化状态管理

### 局限性

1. **永久降级**：一次 WebSocket 失败导致整个 session 降级
2. **预热开销**：WebSocket Prewarm 消耗第一次重试机会
3. **增量优化有前提**：只有连续请求才能增量发送

### 改进方法

```
当前：Binary fallback (WS → HTTP)
改进：
1. 渐进式降级：先尝试 WS 增量，失败后 WS 全量，再失败 HTTP
2. 智能恢复：定期尝试恢复 WS（如每 N 次 HTTP 后尝试一次 WS）
3. 多路复用：同时维护 WS 和 HTTP 连接，根据请求类型选择
```

---

## Q4: 工具执行的 Orchestrator Pattern 为什么需要三阶段？

### 解决什么问题

简单的"执行工具"无法处理：
1. 用户不想让某些命令自动执行（需要审批）
2. 某些操作在沙箱中无法完成（需要权限升级）
3. 临时网络错误需要重试

### 三阶段设计

```
Phase 1: APPROVAL        Phase 2: EXECUTE         Phase 3: ESCALATE
   │                        │                        │
   ├─ Skip (自动批准)        ├─ 沙箱化执行              ├─ 请求无沙箱批准
   ├─ NeedsApproval         ├─ 成功 → 返回            ├─ 重试（无沙箱）
   │  └─ 用户/Guardian审批   └─ SandboxDenied ────────┘
   └─ Forbidden → 拒绝
```

**为什么不让模型自己处理权限问题？**
- 模型可能"忘记"请求权限
- 安全策略应该在系统层面强制，而不是依赖模型行为
- 审批日志需要在系统层面记录

### 局限性

1. **审批延迟**：用户审批增加交互轮次
2. **Guardian 误判**：自动审查可能过于保守
3. **沙箱限制**：某些合法操作被沙箱阻止

### 改进思路

```rust
// 当前：固定三阶段
// 改进：自适应流水线
enum ToolPipeline {
    FastPath,           // 信任的操作，跳过审批
    Standard,           // 标准三阶段
    Cautious,           // 多轮审批（高危操作）
    LearnFromHistory,   // 根据历史决策调整策略
}
```

---

## Q5: 多 Agent 系统为什么需要深度限制？

### 解决什么问题

无限制的 Agent 生成会导致：
1. **资源爆炸**：每个子 Agent 消耗 API 调用和上下文
2. **死循环**：Agent A 生成 Agent B，B 又生成 A
3. **调试困难**：深层嵌套的 Agent 行为难以追踪

### 设计原理

```rust
// 深度检查
if exceeds_thread_spawn_depth_limit(child_depth, max_depth) {
    return Err("Agent depth limit reached. Solve the task yourself.");
}
```

**为什么让当前 Agent "自己解决"？**
- 强制 Agent 提升自身能力，而不是依赖子 Agent
- 避免责任推诿（"我让子 Agent 做的"）
- 保持系统可预测性

### 局限性

1. **任务分解受限**：复杂任务可能需要更深的分解
2. **并行度受限**：深度限制间接限制了并行能力
3. **角色固化**：Reviewer、Guardian 等角色难以动态调整

### 改进方法

```
当前：固定深度限制
改进：
1. 动态深度：根据任务复杂度和资源可用性调整
2. 共享上下文：子 Agent 可以访问父 Agent 的部分上下文
3. 任务队列：将子任务放入队列，由专门的调度器管理
4. 能力评估：只有当子 Agent 能力确实超过父 Agent 时才生成
```

---

# 第二类：实验和方案验证能力

> **核心要求**：不仅关注做了什么，更关注怎么证明它是有效的，能讲清楚实验细节

---

## Q1: 如何验证 Agent Loop 的有效性？

### 验证方法

**1. 集成测试覆盖**

```rust
// core/tests/suite/ 下有 102 个测试模块
// 关键测试：compact.rs (4844 行), client.rs (3684 行)

// 测试模式：Mock SSE Server + 断言
let mock = responses::mount_sse_once(&server, responses::sse(vec![
    responses::ev_response_created("resp-1"),
    responses::ev_function_call(call_id, "shell", &args),
    responses::ev_completed("resp-1"),
])).await;

codex.submit_turn("fix the bug").await?;

// 验证：检查发送给模型的请求
let request = mock.single_request();
assert_eq!(request.function_call_output(call_id).status, "success");
```

**2. 端到端测试场景**

| 测试类型 | 验证目标 | 测试数量 |
|----------|----------|----------|
| 工具调用 | Shell、apply_patch、MCP | 30+ |
| 上下文管理 | 压缩、恢复、fork | 20+ |
| 错误处理 | 重试、降级、恢复 | 15+ |
| 多 Agent | 生成、通信、完成 | 10+ |
| 安全性 | 审批、沙箱、权限 | 15+ |

**3. Snapshot 测试（UI 层面）**

```rust
// 使用 insta 进行快照测试
#[test]
fn test_guardian_review_layout() {
    let layout = render_guardian_review(...);
    assert_snapshot!(layout);
}
```

### 实验细节

**如何测试"模型会正确使用工具"？**

```rust
// 方法：预设模型响应，验证系统行为
// 1. 模型返回 shell 命令
// 2. 系统执行命令
// 3. 验证命令输出被正确返回给模型

// 关键：不测试模型本身，测试系统的响应逻辑
```

**如何测试错误恢复？**

```rust
// 测试场景：WebSocket 失败后降级到 HTTP
#[tokio::test]
async fn test_websocket_fallback() {
    // 1. 设置 WebSocket 返回错误
    // 2. 验证系统自动降级到 HTTP
    // 3. 验证后续请求都使用 HTTP
    // 4. 验证用户看到警告消息
}
```

---

## Q2: 如何验证上下文压缩（Auto-Compact）的有效性？

### 验证维度

**1. Token 减少率**

```rust
// 压缩事件记录
struct CodexCompactionEvent {
    active_context_tokens_before: i64,
    active_context_tokens_after: i64,
    compaction_summary_tokens: Option<i64>,
    // ...
}

// 验证：压缩后 token 数 < 压缩前 * 目标比例
```

**2. 信息保留率**

```rust
// 无法直接量化，但可以通过以下方式验证：
// - 压缩后模型仍能回答关于早期对话的问题
// - 关键操作历史被保留在摘要中
```

**3. 性能影响**

```rust
// 压缩本身的延迟
duration_ms: Option<u64>

// 验证：压缩延迟 < 阈值（如 5 秒）
```

### 实验设计

```
实验 1：压缩触发时机
- 输入：接近 token 限制的对话
- 预期：在达到限制前触发压缩
- 验证：检查触发条件和时机

实验 2：压缩后对话质量
- 输入：压缩后的对话继续
- 预期：模型仍能正确理解上下文
- 验证：模型回答的准确性

实验 3：多次压缩累积效应
- 输入：需要多次压缩的长对话
- 预期：信息丢失在可接受范围内
- 验证：关键信息是否保留
```

---

## Q3: 如何验证沙箱安全性？

### 验证方法

**1. 单元测试**

```rust
// 测试沙箱策略
#[test]
fn test_readonly_sandbox() {
    let policy = SandboxPolicy::ReadOnly;
    // 验证：写操作被拒绝
    assert!(sandbox.execute("touch /tmp/test", &policy).is_err());
    // 验证：读操作被允许
    assert!(sandbox.execute("cat /etc/hostname", &policy).is_ok());
}
```

**2. 集成测试**

```rust
// 测试审批流程
#[tokio::test]
async fn test_approval_required_for_destructive_commands() {
    // 1. 发送 rm -rf 命令
    // 2. 验证系统请求用户审批
    // 3. 模拟用户拒绝
    // 4. 验证命令未执行
}
```

**3. 跨平台测试**

```rust
// 每个平台的沙箱实现都需测试
#[cfg(target_os = "macos")]
mod seatbelt_tests { ... }

#[cfg(target_os = "linux")]
mod landlock_tests { ... }

#[cfg(target_os = "windows")]
mod windows_sandbox_tests { ... }
```

### 实验细节

**如何测试"沙箱升级"场景？**

```
场景：模型需要访问沙箱外的文件

步骤：
1. 模型请求读取 /etc/passwd（沙箱外）
2. 系统返回 SandboxDenied 错误
3. 模型请求升级权限
4. 系统请求用户审批
5. 用户批准
6. 系统以升级权限重试
7. 验证操作成功

验证点：
- SandboxDenied 错误格式正确
- 审批请求正确发送
- 升级后权限正确应用
```

---

## Q4: 如何验证 WebSocket 增量优化的效果？

### 验证方法

**1. 网络传输量对比**

```rust
// 测试：连续两次请求
// 第一次：完整请求 (1000 tokens)
// 第二次：增量请求 (100 tokens delta)

// 验证：第二次请求的 payload 大小 << 第一次
```

**2. 延迟对比**

```rust
// 测量：
// - WebSocket 增量：~100ms
// - WebSocket 全量：~200ms
// - HTTP SSE：~300ms

// 验证：增量方式延迟最低
```

**3. 正确性验证**

```rust
// 验证增量请求的正确性
#[test]
fn test_incremental_request_correctness() {
    // 1. 发送完整请求 A
    // 2. 发送增量请求 B (基于 A)
    // 3. 验证 B 的结果与发送完整请求一致
}
```

---

# 第三类：问题定位能力

> **核心要求**：模型上线后能力突然下降、系统突然缓慢、实验结果和预期不一致时的排查思路

---

## Q1: 模型上线后能力突然下降，如何排查？

### 排查框架

```
Step 1: 确认问题范围
├── 所有任务都下降？还是特定任务？
├── 所有用户都受影响？还是特定用户？
└── 什么时候开始的？有版本变更吗？

Step 2: 检查系统层
├── API 调用是否正常？（检查错误率）
├── 上下文构建是否正确？（检查发送给模型的 prompt）
├── 工具执行是否正常？（检查工具返回结果）
└── 沙箱是否阻止了合法操作？

Step 3: 检查模型层
├── 模型版本是否变更？
├── 模型配置是否变更？
├── System Prompt 是否变更？
└── Temperature/Top-p 是否变更？

Step 4: 检查数据层
├── 用户输入是否有变化？
├── 工具定义是否有变化？
└── 上下文片段是否有变化？
```

### 具体排查工具

**1. Telemetry 分析**

```rust
// 检查关键指标
codex.turn.e2e_duration_ms      // 端到端延迟
codex.turn.token_usage           // Token 使用量
codex.tool.call                  // 工具调用次数
codex.tool.call.duration_ms      // 工具执行延迟

// 如果工具调用次数下降 → 模型可能不再调用工具
// 如果 token 使用量异常 → 上下文构建可能有问题
```

**2. 请求日志分析**

```rust
// 检查发送给模型的请求
// 关键字段：
// - instructions: 系统提示
// - tools: 工具定义
// - input: 对话历史

// 对比正常请求和异常请求的差异
```

**3. Feedback Tags 关联**

```rust
// 用户反馈会关联到具体请求
feedback_tags!(last_model_request_id = upstream_request_id);
feedback_tags!(last_model_response_id = &response_id);

// 可以追踪：用户报告问题 → 对应的 API 请求 → 模型响应
```

### 常见原因和解决方案

| 症状 | 可能原因 | 排查方法 |
|------|----------|----------|
| 模型不再调用工具 | 工具定义变更 | 检查 tools 字段 |
| 模型回答质量下降 | System Prompt 变更 | 对比 instructions |
| 特定任务失败 | 沙箱策略变更 | 检查 SandboxDenied 日志 |
| 延迟显著增加 | 上下文过大 | 检查 token 使用量 |

---

## Q2: 系统上线后突然十分缓慢，如何排查？

### 排查框架

```
Step 1: 定位瓶颈
├── API 调用慢？ → 网络/模型服务问题
├── 工具执行慢？ → 沙箱/系统问题
├── 上下文构建慢？ → 数据处理问题
└── 压缩触发频繁？ → Token 管理问题

Step 2: 检查资源
├── CPU 使用率
├── 内存使用率
├── 网络带宽
└── 磁盘 I/O

Step 3: 检查配置
├── 重试次数是否过多？
├── 超时设置是否合理？
├── 并发限制是否过低？
└── 缓存是否生效？
```

### 具体排查工具

**1. 延迟分解**

```rust
// OTEL 指标分解延迟
codex.turn.ttft.duration_ms       // Time to First Token
codex.turn.ttfm.duration_ms       // Time to First Message
codex.api_request.duration_ms     // API 请求延迟
codex.tool.call.duration_ms       // 工具执行延迟
codex.sse_event.duration_ms       // SSE 事件延迟

// 对比正常和异常情况的延迟分布
```

**2. 重试分析**

```rust
// 检查重试事件
codex.transport.fallback_to_http  // WebSocket 降级次数

// 检查错误类型分布
CodexErr::Stream(..)              // 流错误
CodexErr::Timeout                 // 超时
CodexErr::ConnectionFailed(..)    // 连接失败

// 如果重试频繁 → 网络问题
// 如果超时频繁 → 服务端问题
```

**3. 上下文大小分析**

```rust
// 检查 token 使用趋势
codex.turn.token_usage

// 如果 token 使用量持续增长 → 可能触发频繁压缩
// 压缩本身也有开销
```

### 常见原因和解决方案

| 症状 | 可能原因 | 解决方案 |
|------|----------|----------|
| 首次响应慢 | WebSocket 预热 | 检查 prewarm 逻辑 |
| 间歇性卡顿 | 重试风暴 | 检查重试策略 |
| 持续变慢 | 上下文膨胀 | 检查压缩触发 |
| 工具执行慢 | 沙箱开销 | 检查沙箱配置 |

---

## Q3: 实验结果和预期不一致，如何排查？

### 排查框架

```
Step 1: 确认预期是否正确
├── 预期的假设是什么？
├── 假设的依据是什么？
└── 假设是否经过验证？

Step 2: 检查实验设置
├── 测试数据是否正确？
├── 配置是否正确？
├── 环境是否一致？
└── 随机种子是否固定？

Step 3: 检查实现
├── 代码逻辑是否正确？
├── 边界条件是否处理？
├── 并发是否有问题？
└── 依赖版本是否一致？

Step 4: 检查外部因素
├── 模型版本是否变更？
├── API 行为是否变更？
├── 系统资源是否充足？
└── 网络是否稳定？
```

### 具体案例

**案例：压缩后模型回答质量下降**

```
预期：压缩后模型仍能正确回答关于早期对话的问题
实际：模型回答不正确

排查：
1. 检查压缩逻辑
   - 压缩摘要是否包含关键信息？
   - 压缩比例是否过大？

2. 检查测试数据
   - 测试问题是否真的依赖早期信息？
   - 是否有其他信息源可以回答？

3. 检查模型行为
   - 模型是否真的在使用压缩后的摘要？
   - 模型是否有足够的上下文？

发现：压缩摘要丢失了关键的代码上下文
解决：改进压缩策略，保留代码片段
```

---

# 第四类：工程落地能力

> **核心要求**：理论结合实际，算法怎么部署，系统保持稳定，上线后怎么保证数据回滚与监控

---

## Q1: 如何将 Agent 算法部署到生产环境？

### 部署架构

```
┌─────────────────────────────────────────────────────────┐
│                    用户终端                              │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐           │
│   │ CLI      │   │ TUI      │   │ IDE插件   │           │
│   └────┬─────┘   └────┬─────┘   └────┬─────┘           │
│        └──────────────┼──────────────┘                  │
│                       ▼                                  │
│              ┌────────────────┐                          │
│              │   App Server   │                          │
│              │  (JSON-RPC)    │                          │
│              └───────┬────────┘                          │
├──────────────────────┼──────────────────────────────────┤
│                      ▼                                   │
│   ┌──────────────────────────────────────────┐          │
│   │           Core Engine (Rust)             │          │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐  │          │
│   │  │ Session │  │  Tools  │  │ Sandbox │  │          │
│   │  └────┬────┘  └────┬────┘  └────┬────┘  │          │
│   │       └────────────┼────────────┘        │          │
│   └───────────────────┬┼┬────────────────────┘          │
│                       ▼▼▼                                │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│   │ OpenAI   │  │ 本地文件  │  │ Shell    │              │
│   │ API      │  │ 系统     │  │ 环境     │              │
│   └──────────┘  └──────────┘  └──────────┘              │
└─────────────────────────────────────────────────────────┘
```

### 部署关键点

**1. 多平台构建**

```rust
// Cargo + Bazel 双构建系统
// Cargo: 本地开发
// Bazel: CI/CD 和生产构建

// 平台特定代码
#[cfg(target_os = "macos")]
mod seatbelt_sandbox;

#[cfg(target_os = "linux")]
mod landlock_sandbox;

#[cfg(target_os = "windows")]
mod windows_sandbox;
```

**2. 依赖管理**

```toml
# Cargo.toml - 显式依赖版本
[dependencies]
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
# ...
```

**3. 二进制分发**

```javascript
// npm wrapper - 检测平台，加载对应二进制
// codex-cli/bin/codex.js
const platform = process.platform;
const arch = process.arch;
const binary = require(`@openai/codex-${platform}-${arch}`);
```

### 部署检查清单

```
□ 多平台构建通过（Linux/macOS/Windows）
□ 依赖版本锁定（Cargo.lock）
□ 二进制大小优化（release build）
□ 启动时间优化（prewarm）
□ 内存使用监控
□ 错误处理覆盖
□ 日志级别配置
□ 配置文件验证
```

---

## Q2: 如何保证系统稳定性？

### 稳定性设计

**1. 错误分类与重试**

```rust
// 错误分类：可重试 vs 不可重试
pub fn is_retryable(&self) -> bool {
    match self {
        // 不可重试：用户错误、资源限制
        CodexErr::UsageLimitReached(_) => false,
        CodexErr::QuotaExceeded => false,

        // 可重试：临时错误
        CodexErr::Stream(..) => true,
        CodexErr::Timeout => true,
        CodexErr::ConnectionFailed(..) => true,
        // ...
    }
}

// 指数退避重试
fn backoff(attempt: u64) -> Duration {
    let exp = BACKOFF_FACTOR.powi(attempt as i32);
    let base = INITIAL_DELAY_MS * exp;
    let jitter = rand::random_range(0.9..1.1);
    Duration::from_millis((base * jitter) as u64)
}
```

**2. 传输降级**

```rust
// WebSocket → HTTP 降级
// 一旦降级，整个 session 保持 HTTP
disable_websockets: AtomicBool

// 避免反复切换（churn）
```

**3. 资源限制**

```rust
// Token 预算
struct RolloutBudget {
    weighted_tokens_used: f64,
    // 超出预算时触发压缩
}

// 输出大小限制
const EXEC_OUTPUT_MAX_BYTES: usize = ...;

// Agent 深度限制
const MAX_AGENT_DEPTH: usize = ...;
```

**4. 超时控制**

```rust
// 使用 tokio::select! 实现超时
tokio::select! {
    result = child.wait() => { /* 正常退出 */ }
    _ = timeout => { /* 超时，kill 进程 */ }
    _ = ctrl_c => { /* 用户中断 */ }
}
```

### 稳定性指标

| 指标 | 目标 | 监控方式 |
|------|------|----------|
| 可用性 | 99.9% | 错误率 < 0.1% |
| 延迟 | P99 < 5s | OTEL 指标 |
| 内存 | < 500MB | 系统监控 |
| 崩溃率 | < 0.01% | 错误上报 |

---

## Q3: 如何实现数据回滚与监控？

### 数据持久化

**1. 会话持久化（Rollout）**

```rust
// JSONL 格式持久化对话历史
struct Rollout {
    items: Vec<ResponseItem>,
    // 每次工具调用和结果都被记录
}

// 位置：~/.codex/rollouts/{thread_id}.jsonl
```

**2. 线程元数据（SQLite）**

```rust
// SQLite 存储线程元数据
struct ThreadMetadata {
    id: ThreadId,
    created_at: DateTime,
    model: String,
    status: ThreadStatus,
    // ...
}
```

**3. 配置版本化**

```rust
// 配置分层，支持回滚
struct ConfigLayers {
    defaults: Config,           // 默认配置
    user: Config,               // 用户配置
    project: Config,            // 项目配置
    managed: Config,            // 管理员配置
    runtime_overrides: Config,  // 运行时覆盖
}
```

### 回滚机制

**1. 会话恢复**

```rust
// 从 rollout 恢复会话
async fn resume_thread(thread_id: ThreadId) -> Result<Session> {
    let rollout = load_rollout(thread_id).await?;
    let session = Session::from_rollout(rollout).await?;
    Ok(session)
}

// Fork 会话（创建分支）
async fn fork_thread(thread_id: ThreadId) -> Result<Session> {
    let rollout = load_rollout(thread_id).await?;
    let session = Session::fork(rollout).await?;
    Ok(session)
}
```

**2. 配置回滚**

```rust
// 配置变更记录
struct ConfigChange {
    timestamp: DateTime,
    key: String,
    old_value: Value,
    new_value: Value,
}

// 回滚到指定时间点
fn rollback_config(target_time: DateTime) -> Result<Config> {
    let changes = load_config_changes(target_time)?;
    apply_reverse_changes(changes)
}
```

### 监控体系

**1. OTEL 指标（61 个）**

```rust
// 关键指标
codex.api_request.duration_ms        // API 延迟
codex.tool.call.duration_ms          // 工具执行延迟
codex.turn.e2e_duration_ms           // 端到端延迟
codex.turn.token_usage               // Token 使用
codex.transport.fallback_to_http     // 降级事件
```

**2. 分析事件（20+ 种）**

```rust
// 业务事件
ThreadInitialized       // 会话初始化
TurnEvent               // 对话事件
CompactionEvent         // 压缩事件
GuardianReviewEvent     // 安全审查事件
FileChangeEvent         // 文件变更事件
```

**3. 反馈关联**

```rust
// 用户反馈关联到具体请求
feedback_tags!(last_model_request_id = upstream_request_id);
feedback_tags!(last_model_response_id = &response_id);

// 可以追踪：用户报告 → 对应请求 → 模型响应
```

### 监控仪表盘设计

```
┌─────────────────────────────────────────────────────────┐
│                    监控仪表盘                           │
├─────────────────────────────────────────────────────────┤
│  可用性: 99.95%    延迟 P99: 2.3s    错误率: 0.03%      │
├─────────────────────────────────────────────────────────┤
│  实时指标                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │ API 调用/秒  │  │ 工具调用/秒  │  │ 活跃会话    │    │
│  │    1,234     │  │     567     │  │     890     │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
├─────────────────────────────────────────────────────────┤
│  告警                                                  │
│  ⚠️ WebSocket 降级率 > 10% (当前: 12%)                  │
│  ⚠️ 压缩触发频率增加 (当前: 5次/分钟)                   │
└─────────────────────────────────────────────────────────┘
```

---

# 第五类：业务与实际场景理解

> **核心要求**：方案适合什么场景，用户关心什么，上线成本多高，资源有限时优先优化什么

---

## Q1: Code Agent 适合什么场景？不适合什么场景？

### 适合的场景

| 场景 | 为什么适合 | 示例 |
|------|------------|------|
| 代码重构 | 多文件修改、需要理解上下文 | "把这个函数提取到独立模块" |
| Bug 修复 | 需要读代码、定位问题、验证修复 | "修复这个空指针异常" |
| 代码生成 | 有明确规范、可验证 | "写一个 REST API endpoint" |
| 代码审查 | 需要理解整体架构 | "审查这个 PR 的安全性" |
| 文档生成 | 结构化、可模板化 | "为这个模块生成 API 文档" |

### 不适合的场景

| 场景 | 为什么不适合 | 替代方案 |
|------|--------------|----------|
| 创意写作 | 缺乏人类审美和情感 | 人工 + AI 辅助 |
| 高度专业化领域 | 缺乏领域知识 | 领域专家 + 工具 |
| 实时交互 | 延迟太高 | 预编译/缓存方案 |
| 关键决策 | 需要人类判断 | 人工审核 |
| 探索性研究 | 需要创造性思维 | 人工研究 + AI 工具 |

### 用户画像

```
主要用户：开发者
├── 初级开发者：需要指导和解释
├── 中级开发者：需要效率提升
└── 高级开发者：需要自动化重复工作

用户核心需求：
1. 效率：快速完成任务
2. 准确：结果正确可靠
3. 安全：不破坏现有代码
4. 可控：理解 Agent 在做什么
```

---

## Q2: 用户最关心什么？

### 用户关注点排序

```
1. 安全性 (Safety)
   ├── 不会删除或覆盖重要文件
   ├── 不会执行危险命令
   └── 操作可撤销

2. 准确性 (Accuracy)
   ├── 代码修改正确
   ├── 理解意图准确
   └── 不引入新 bug

3. 效率 (Efficiency)
   ├── 响应速度快
   ├── 完成任务快
   └── 减少人工干预

4. 可控性 (Controllability)
   ├── 理解 Agent 在做什么
   ├── 可以干预和修改
   └── 可以回滚操作

5. 成本 (Cost)
   ├── API 调用费用
   ├── 时间成本
   └── 学习成本
```

### 设计如何满足用户需求

| 用户需求 | 设计实现 |
|----------|----------|
| 安全性 | 多层沙箱、审批机制、Guardian 审查 |
| 准确性 | 工具执行验证、测试运行、diff 展示 |
| 效率 | 并行工具执行、增量传输、缓存优化 |
| 可控性 | 实时输出、审批流程、会话恢复 |
| 成本 | Token 预算管理、智能压缩、缓存复用 |

---

## Q3: 上线成本有多高？

### 成本构成

```
1. API 调用成本
   ├── 输入 token: ~$0.01/1K tokens
   ├── 输出 token: ~$0.03/1K tokens
   └── 平均每次对话: ~$0.10 - $1.00

2. 基础设施成本
   ├── 服务器: 边缘计算（用户本地）
   ├── 存储: SQLite（本地）
   └── 网络: API 调用带宽

3. 开发维护成本
   ├── 多平台维护
   ├── 安全更新
   └── 模型适配

4. 用户成本
   ├── 学习成本
   ├── 时间成本
   └── 信任成本
```

### 成本优化策略

```rust
// 1. Prompt Cache
fn prompt_cache_key(&self) -> String {
    // 同一线程复用 cache，减少重复计算
    self.thread_id.to_string()
}

// 2. 增量传输
fn get_incremental_items(&self, ...) -> Option<Vec<ResponseItem>> {
    // 只发送变化部分，减少网络传输
}

// 3. 智能压缩
async fn run_auto_compact(...) {
    // 接近 token 限制时压缩，避免超出预算
}

// 4. 工具结果缓存
struct BlockingLruCache<K, V> {
    // 缓存工具执行结果
}
```

### ROI 分析

```
场景：代码重构任务

人工成本：
- 高级开发者: 2小时 × $100/小时 = $200
- 错误修复: 1小时 × $100/小时 = $100
- 总计: $300

Agent 成本：
- API 调用: ~$2.00
- 开发者监督: 0.5小时 × $100/小时 = $50
- 总计: $52

ROI: ($300 - $52) / $52 = 477%
```

---

## Q4: 资源有限时，应该优先优化什么？

### 优化优先级矩阵

```
                高影响
                  │
    ┌─────────────┼─────────────┐
    │  P0: 安全性  │  P1: 准确性  │
    │  (必须做)    │  (优先做)    │
    ├─────────────┼─────────────┤
    │  P2: 成本    │  P3: 体验    │
    │  (持续优化)  │  (逐步改进)  │
    └─────────────┼─────────────┘
                  │
                低影响
    低紧急度 ──────┼────── 高紧急度
```

### 优先级建议

**P0: 安全性（必须做）**
- 沙箱机制
- 审批流程
- 错误处理
- 数据保护

**P1: 准确性（优先做）**
- 工具执行验证
- 测试覆盖
- 错误恢复
- 用户反馈收集

**P2: 成本（持续优化）**
- Token 预算管理
- 缓存优化
- 压缩策略
- 传输优化

**P3: 体验（逐步改进）**
- UI/UX 优化
- 响应速度
- 文档完善
- 示例丰富

### 资源分配建议

```
有限资源下的分配：
- 安全性: 30%
- 准确性: 30%
- 成本优化: 20%
- 体验优化: 20%

阶段性重点：
Phase 1: 安全性 + 基础准确性（MVP）
Phase 2: 准确性提升 + 成本优化（GA）
Phase 3: 体验优化 + 高级功能（Premium）
```

---

## Q5: 如何衡量 Code Agent 的业务价值？

### 核心指标

```
1. 效率指标
   ├── 任务完成时间（vs 人工）
   ├── 人工干预次数
   └── 代码质量（测试覆盖率、bug 数）

2. 成本指标
   ├── API 调用成本
   ├── 开发者时间成本
   └── 错误修复成本

3. 质量指标
   ├── 代码正确率
   ├── 测试通过率
   └── 用户满意度

4. 安全指标
   ├── 安全事件数
   ├── 数据泄露数
   └── 审批通过率
```

### 衡量方法

```rust
// 1. A/B 测试
// 对比：Agent 辅助 vs 纯人工
// 指标：完成时间、错误率、满意度

// 2. 用户调研
// 定性反馈：哪里好用、哪里不好用
// 定量评分：1-10 分

// 3. 长期追踪
// 用户留存率
// 使用频率
// 付费转化率
```

### 价值证明

```
案例：代码审查场景

Before (人工审查):
- 平均审查时间: 2小时
- 发现问题数: 3-5个
- 漏检率: 20%

After (Agent 辅助):
- 平均审查时间: 30分钟
- 发现问题数: 8-12个
- 漏检率: 5%

价值:
- 效率提升: 4x
- 质量提升: 2x
- 成本节省: $150/次审查
```

---

## 附录：面试回答模板

### 回答底层原理问题

```
模板：
1. 这个设计解决什么问题？（Why）
2. 具体是怎么实现的？（What）
3. 有什么局限性？（Limitation）
4. 有哪些改进思路？（Improvement）

示例：
"Agent Loop 设计成 ReAct 循环，主要是为了解决单次 LLM 调用无法处理多步任务的问题。
具体实现是在 turn.rs 中的 run_turn 函数，循环执行：构建上下文 → 调用模型 → 执行工具。
局限性在于延迟累积和 token 消耗。
改进思路包括并行 Agent、计划先行、检查点机制等。"
```

### 回答实验验证问题

```
模板：
1. 验证目标是什么？（What）
2. 用什么方法验证？（How）
3. 具体实验细节？（Detail）
4. 结果如何？（Result）

示例：
"验证上下文压缩的有效性。
方法是集成测试 + 指标监控。
具体测试包括：压缩触发时机、压缩后对话质量、多次压缩累积效应。
结果：压缩率 60%，信息保留率 85%，延迟 < 2s。"
```

### 回答问题定位问题

```
模板：
1. 问题现象是什么？（Symptom）
2. 排查思路是什么？（Approach）
3. 根因是什么？（Root Cause）
4. 解决方案是什么？（Solution）

示例：
"现象：模型上线后能力下降。
排查：先检查系统层（API调用、工具执行），再检查模型层（版本、配置），最后检查数据层。
根因：工具定义变更导致模型不再调用工具。
解决方案：回滚工具定义，添加工具调用率监控。"
```

### 回答工程落地问题

```
模板：
1. 理论方案是什么？（Theory）
2. 工程实现怎么做？（Implementation）
3. 遇到什么问题？（Challenge）
4. 如何解决的？（Solution）

示例：
"理论方案是多平台沙箱。
工程实现使用条件编译（#[cfg]），每个平台独立实现。
遇到的问题是 Windows 沙箱 API 与 Unix 差异大。
解决方案是抽象统一接口，平台特定实现封装在独立模块。"
```

### 回答业务理解问题

```
模板：
1. 适合什么场景？（Scenario）
2. 用户关心什么？（User Need）
3. 成本多高？（Cost）
4. 如何衡量价值？（Value）

示例：
"适合代码重构、bug 修复等多步任务。
用户最关心安全性和准确性。
成本主要是 API 调用，平均 $0.10-1.00/次。
价值通过效率提升（4x）和质量提升（2x）衡量。"
```

---

*最后更新：2026-06-27*

*祝面试顺利！*
