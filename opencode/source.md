# OpenCode 架构分析报告

> 分析日期: 2026-05-20
> 分析范围: packages/opencode/src/agent/, packages/opencode/src/session/prompt.ts
> 测试覆盖: packages/opencode/test/

---

## 1. Agent 架构

### 1.1 核心文件职责

| 文件 | 行数 | 实际职责 |
|------|------|----------|
| `src/agent/agent.ts` | 463 | Agent 注册表 + 配置解析 + 权限定义 |
| `src/session/prompt.ts` | 2157 | **真正的执行引擎**：ReAct 循环、工具调用、子 Agent 调度 |
| `src/acp/agent.ts` | 1507+ | ACP 协议实现（IDE 集成层） |

### 1.2 问题：`agent.ts` 只是注册表

用户期望 Agent 逻辑在 `agent/` 下，但实际的 ReAct 循环、工具执行、子任务调度全部在 `session/prompt.ts` 中。`agent.ts` 只做：
- 定义内置 Agent（build/plan/general/explore/scout/compaction/title/summary）
- 解析用户配置
- 合并权限规则

**影响**：新开发者找不到核心逻辑，调试困难。

---

## 2. ReAct 循环 (`runLoop`)

### 2.1 位置
`packages/opencode/src/session/prompt.ts:1643-1875`

### 2.2 循环流程

```
while (true) {
  1. 全量读取消息 (MessageV2.filterCompactedEffect)
  2. 退出判断 (finish + hasToolCalls)
  3. 特殊任务分发 (subtask / compaction)
  4. 准备下一轮 (Agent 查找 + insertReminders)
  5. 调用 LLM (resolveTools + handle.process)
  6. 判断下一步 (break / continue)
}
```

### 2.3 发现的缺陷

| 缺陷 | 位置 | 严重性 | 说明 |
|------|------|--------|------|
| **无重试机制** | L1821 `handle.process` | 🔴 高 | LLM 调用失败直接终止，无 exponential backoff |
| **无工具超时** | L578-608 `resolveTools` | 🔴 高 | 工具（如 bash）可无限期阻塞循环 |
| **无死循环检测** | L1770-1832 | 🔴 高 | LLM 反复调用同一工具不会被阻止 |
| **无并发控制** | L1821 `handle.process` | 🟡 中 | LLM 返回 10 个工具调用会全部并行执行 |
| **全量读消息** | L1655 | 🟡 中 | 每轮循环从数据库读全部消息，长会话性能差 |
| **step 计数不持久化** | L1648 `let step = 0` | 🟡 中 | 进程重启后 maxSteps 限制失效 |
| **同步阻塞压缩** | L1699-1707 | 🟡 中 | compaction.process 阻塞循环，用户无进度反馈 |
| **硬编码退出条件** | L1672-1678 | 🟢 低 | `["tool-calls"]` 应使用常量/枚举 |

### 2.4 退出条件分析

```typescript
// L1668-1679
if (
  lastAssistant?.finish &&
  !["tool-calls"].includes(lastAssistant.finish) &&
  !hasToolCalls &&
  lastUser.id < lastAssistant.id
) { break }
```

- ✅ 处理了 provider 返回 "stop" 但还有工具调用的边界情况
- ❌ 没有兜底退出（如果 finish 永远 undefined，只能靠 maxSteps）
- ❌ 没有检测死循环（反复调用同一工具）

---

## 3. 漂移问题 (Drift)

### 3.1 根因分析

| 原因 | 代码位置 | 机制 |
|------|----------|------|
| **无原始任务锚点** | L1657 `MessageV2.latest(msgs)` | 只看最后一条用户消息，不记得最初指令 |
| **Attention 衰减** | L1816 `toModelMessagesEffect(msgs)` | 长对话中原始指令被推到中间低权重区域 |
| **压缩丢失细节** | L1655 `filterCompactedEffect` | 压缩后工具调用结果被摘要替代，约束丢失 |
| **无计划跟踪** | 全局缺失 | 没有 TODO list 跟踪进度 |
| **无工作记忆** | 全局缺失 | 不维护已发现事实、已做决策 |

### 3.2 用户体验表现

- "卡手"：越用越慢，突然卡住（压缩阻塞）
- "漂移"：Agent 忘记最初要做什么，反复做同一件事

### 3.3 推荐改进方案

| 方案 | Token 开销 | 进度追踪 | 漂移检测 | 复杂度 |
|------|-----------|---------|---------|--------|
| 原始任务锚点 | 高 | ❌ | ❌ | 低 |
| **任务状态机** | 中 | ✅ | ⚠️ | 中 |
| 分层执行 | 低 | ✅ | ✅ | 高 |
| 工作记忆 | 极低 | ✅ | ✅ | 高 |
| 自反思循环 | 额外调用 | ✅ | ✅ | 中 |
| **约束持久化** | 低 | ❌ | ✅ | 低 |

**推荐组合**: 任务状态机 + 约束持久化 + 每 5 轮自反思

---

## 4. 测试覆盖分析

### 4.1 测试架构

```
TestLLMServer (本地 HTTP mock)
  ↓
SessionPrompt.layer (真实业务逻辑)
  ↓
Mock 服务 (MCP, LSP, Summary 等)
```

- 使用 `testEffect` + `it.instance` 框架
- 临时目录自动清理
- Effect Layer 注入

### 4.2 覆盖场景

| 类别 | 测试数 | 示例 |
|------|--------|------|
| 循环退出 | 4 | stop 退出、tool-calls 继续 |
| 取消语义 | 7 | 中断 loop、子任务取消 |
| 并发 | 4 | 并发 loop 同结果 |
| Shell | 7 | 捕获输出、busy 拒绝 |
| 子任务 | 4 | 失败保留 metadata |
| Agent 配置 | 25+ | 权限、模式、steps |

### 4.3 测试盲区

| 缺失场景 | 风险 |
|----------|------|
| **10+ 轮工具调用** | 无法检测漂移 |
| **上下文压缩后行为** | 无法验证约束保留 |
| **重复工具调用检测** | 死循环无测试 |
| **长会话性能** | 全量读消息无回归测试 |
| **LLM 失败重试** | 无重试机制可测 |
| **工具超时** | 无超时机制可测 |

### 4.4 为什么漂移没被测试到

```typescript
// 最长的多轮测试只有 2 次 LLM 调用
yield* llm.tool("first", { value: "first" })
yield* llm.text("second")
const result = yield* prompt.loop({ sessionID: session.id })
expect(yield* llm.calls).toBe(2)
```

- 测试 focus 在"机制能跑"，不是"Agent 跑得好"
- 每轮都要 mock LLM 回复，10 轮要写 10 个 `yield* llm.*`
- 没有定义"漂移"的可观测指标

---

## 5. 代码质量评估

### 5.1 `prompt.ts` (2157 行)

| 维度 | 评分 | 说明 |
|------|------|------|
| Effect 规范 | ⭐⭐⭐⭐ | 正确使用 Service/Layer/Effect.fn |
| 架构设计 | ⭐⭐⭐ | 服务分层清晰，但单文件职责过多 |
| 可读性 | ⭐⭐ | 函数过长，嵌套深 |
| 可维护性 | ⭐⭐ | 2000+ 行难以导航 |
| 鲁棒性 | ⭐⭐⭐ | 缺少重试、超时、死循环检测 |
| 可扩展性 | ⭐⭐⭐⭐ | 工具/Agent 通过注册表添加 |

### 5.2 违反 Style Guide

| 规则 | 现状 |
|------|------|
| Keep things in one function | `layer` 内定义 20+ 内部函数 |
| Avoid unnecessary destructuring | L710: `const { task, model, ... } = input` |
| Complex Logic → happy path + helpers | `runLoop` 未拆分退出检查、任务分发 |

### 5.3 建议拆分

```
session/prompt/
  layer.ts          # 依赖注入 + Service 注册
  loop.ts           # runLoop 核心循环
  tools.ts          # resolveTools
  subtask.ts        # handleSubtask
  user-message.ts   # createUserMessage
  shell.ts          # shellImpl
  command.ts        # command
```

---

## 6. 与业界对比

| 特性 | OpenCode | Claude Code | Cursor | LangGraph |
|------|----------|-------------|--------|-----------|
| 流式工具执行 | ✅ | ✅ | ✅ | ✅ |
| 中断恢复 | ✅ | ✅ | ❌ | ✅ |
| 自动压缩 | ✅ | ✅ | ❌ | 手动 |
| 工具超时 | ❌ | ✅ | ✅ | ✅ |
| 死循环检测 | ❌ | ✅ | ⚠️ | ✅ |
| 重试机制 | ❌ | ✅ | ✅ | ✅ |
| 并发控制 | ❌ | ✅ | ✅ | ✅ |
| 任务状态跟踪 | ❌ | ✅ | ❌ | ✅ |

---

## 7. 优先改进建议

### 7.1 最小改动（3 个）

1. **加原始任务锚点**: system prompt 始终包含第一条用户消息
2. **加工具超时**: `Effect.timeout(item.execute(...), "30 seconds")`
3. **加死循环检测**: 记录最近 5 次工具调用，3 次相同强制 break

### 7.2 中期改进

4. **任务状态机**: 维护 Goal/Progress/Completed/Constraints
5. **约束持久化**: 不可变约束单独提取，每轮强制注入
6. **增量消息读取**: 只读新消息，缓存历史

### 7.3 长期改进

7. **分层执行**: 原始任务 → 子任务 → 工具调用
8. **工作记忆**: 维护 facts/decisions/openQuestions
9. **自反思循环**: 每 5 轮检查是否偏离目标
10. **拆分 prompt.ts**: 按职责拆分为多个文件

---

## 8. 关键代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| Agent 注册表 | `src/agent/agent.ts` | 126-278 |
| ReAct 循环 | `src/session/prompt.ts` | 1643-1875 |
| 工具解析 | `src/session/prompt.ts` | 522-700 |
| 子任务调度 | `src/session/prompt.ts` | 702-893 |
| 用户消息构建 | `src/session/prompt.ts` | 1092-1612 |
| 上下文压缩 | `src/session/compaction.ts` | - |
| 退出判断 | `src/session/prompt.ts` | 1668-1679 |
| LLM 调用 | `src/session/prompt.ts` | 1821-1832 |
| Agent 测试 | `test/agent/agent.test.ts` | 732 行 |
| Prompt 测试 | `test/session/prompt.test.ts` | 1528+ 行 |

---

# 漂移问题专项分析

## 症状

- Agent 执行过程中"忘记"最初要做什么
- 反复做同一件事（原地打转）
- 压缩上下文后开始偏离约束

## 根因

### 1. Attention 机制的位置效应

```
[系统提示] ← 高权重
[早期对话] ← 权重随距离衰减
[中间对话] ← 最低权重（"中间迷失"现象）
[最近对话] ← 高权重（recency bias）
```

随着对话变长，原始指令被推到中间区域，注意力权重最低。

### 2. 代码层面的缺失

| 机制 | 现状 | 代码位置 |
|------|------|----------|
| 原始任务锚点 | ❌ 不存在 | `runLoop` 只看 `lastUser` |
| 任务状态跟踪 | ❌ 不存在 | 无 Goal/Progress 结构 |
| 死循环检测 | ❌ 不存在 | 无 tool call 去重 |
| 约束持久化 | ❌ 不存在 | 压缩后约束丢失 |
| 计划跟踪 | ❌ 不存在 | 无 TODO list |

### 3. 上下文压缩的副作用

```typescript
// L1655
let msgs = yield* MessageV2.filterCompactedEffect(sessionID)
// 压缩后的消息变成摘要，工具调用的具体结果被丢弃
```

压缩把 10 轮对话压缩成一段摘要，但摘要里可能丢失了关键约束（如"只改 src/ 目录"）。

## 推荐方案

### 方案 A: 任务状态机（推荐优先实现）

```typescript
interface TaskState {
  original: string      // 原始意图
  current: string       // 当前子目标
  completed: string[]   // 已完成的步骤
  constraints: string[] // 不可变的约束
  modified: boolean     // 用户是否修改过目标
}
```

每轮注入结构化状态而非原始文本：

```
System Prompt:
## Task State
Goal: 找 API 端点并写文档
Progress: 3/5 steps completed
Completed: [grep api, read users.ts, read posts.ts]
Next: write API_DOCS.md
Constraints: 只改 src/ 目录
```

**优势**:
- Token 少（只传结构化数据）
- 有进度跟踪
- 约束与目标分离
- 用户修改目标时只更新 `current`

### 方案 B: 约束持久化

把不可变约束单独提取，每轮强制注入：

```typescript
const constraints = extractConstraints(originalTask)
// "只改 src/" → { type: "path", pattern: "src/**", action: "allow" }

// 每轮注入：
system.push(`<constraints>${constraints.map(c => c.toXml()).join('\n')}</constraints>`)
```

### 方案 C: 自反思循环

每 N 轮执行一次元认知检查：

```typescript
const reflection = yield* llm.stream({
  system: `Check if the agent is still on track.`,
  messages: [
    { role: "user", content: `Original task: "${originalTask}"` },
    { role: "assistant", content: `Recent actions: ${recentActions}` },
  ]
})

if (reflection.drift) {
  msgs.push({
    role: "system",
    content: `You have drifted. Refocus on: ${originalTask}`
  })
}
```

## 测试建议

```typescript
it.instance("agent stays on track after 10 tool calls", () =>
  Effect.gen(function* () {
    const { llm } = yield* useServerConfig(providerCfg)
    const prompt = yield* SessionPrompt.Service
    const chat = yield* sessions.create({
      permission: [{ permission: "*", action: "allow", pattern: "*" }]
    })

    // 预设 10 轮工具调用
    for (let i = 0; i < 10; i++) {
      yield* llm.tool("read", { file: `file${i}.ts` })
    }
    yield* llm.text("All files read. Original task: write docs for API endpoints")

    const result = yield* prompt.loop({ sessionID: chat.id })

    // 断言：最终回复包含原始任务关键词
    const text = result.parts.find(p => p.type === "text")?.text ?? ""
    expect(text).toContain("API")  // 漂移检测
  }),
)
```

## 实施优先级

1. **P0**: 加原始任务锚点（最小改动，立即见效）
2. **P0**: 加工具超时（解决"卡手"问题）
3. **P1**: 加死循环检测（防止原地打转）
4. **P1**: 任务状态机（结构化进度跟踪）
5. **P2**: 约束持久化（压缩后保留约束）
6. **P2**: 自反思循环（主动检测漂移）
