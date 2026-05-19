# OpenCode 架构分析报告

> 分析日期: 2026-05-20
> 分析范围: packages/opencode/src/agent/, packages/opencode/src/session/prompt.ts
> 测试覆盖: packages/opencode/test/
> 分析方法: 静态代码阅读（高置信度）+ 逻辑推导（中置信度）
> 用户反馈: 未验证，标注为"推测"

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
| **全量读消息** | L1655 | 🟡 中 | 每轮循环从数据库读全部消息，长会话性能差 |
| **step 计数不持久化** | L1648 `let step = 0` | 🟡 中 | 进程重启后 maxSteps 限制失效 |
| **同步阻塞压缩** | L1699-1707 | 🟡 中 | compaction.process 阻塞循环，用户无进度反馈。前提：假设 compaction 只有同步模式（需确认）。若支持异步，此缺陷不成立 |
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

### 3.1 用户感知（推测，未验证）

- "卡手"：越用越慢，突然卡住 — 可能原因：compaction 阻塞、全量读消息
- "漂移"：Agent 忘记最初要做什么，反复做同一件事 — 可能原因：Attention 衰减

以上描述来自代码分析推导，未收集实际用户反馈验证。

### 3.2 根因分析

| 原因 | 代码位置 | 机制 |
|------|----------|------|
| **无原始任务锚点** | L1657 `MessageV2.latest(msgs)` | 只看最后一条用户消息，不记得最初指令 |
| **Attention 衰减** | L1816 `toModelMessagesEffect(msgs)` | 长对话中原始指令被推到中间低权重区域 |
| **压缩丢失细节** | L1655 `filterCompactedEffect` | 压缩后工具调用结果被摘要替代，约束丢失 |
| **无计划跟踪** | 全局缺失 | 没有 TODO list 跟踪进度 |
| **无工作记忆** | 全局缺失 | 不维护已发现事实、已做决策 |

### 3.3 推荐改进方案

| 方案 | Token 开销 | 进度追踪 | 漂移检测 | 复杂度 | 建议排期 |
|------|-----------|---------|---------|--------|---------|
| 原始任务锚点 | 高 | ❌ | ❌ | 低 | P0（~10 行，不改逻辑，零风险） |
| **任务状态机** | 中 | ✅ | ⚠️ | 中 | P1（需新增 interface + 序列化 + 存储） |
| 分层执行 | 低 | ✅ | ✅ | 高 | P2 |
| 工作记忆 | 极低 | ✅ | ✅ | 高 | P2 |
| 自反思循环 | 额外调用 | ✅ | ✅ | 中 | P1 |
| **约束持久化** | 低 | ❌ | ✅ | 低 | P1 |

**推荐组合**: 任务状态机 + 约束持久化 + 每 5 轮自反思

**优先级排序理由**：

```
P0（原始任务锚点）优先：
- 改动量约 10 行，system prompt 加一条消息，不影响任何逻辑
- 无风险，可立即上线观察效果

P1（任务状态机 + 自反思 + 约束持久化）延后：
- 三项均需新增数据结构和序列化逻辑
- 任务状态机依赖先确认 task/original 的存储位置
- 约束持久化需要先定义"约束"的提取规则
- 自反思需要先有漂移的判定标准
```

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
| 并发控制 | ✅（并行） | ✅（并行） | ✅（并行） | ✅（并行） |
| 任务状态跟踪 | ❌ | ✅ | ❌ | ✅ |

---

## 7. 改进建议

### 7.1 高优先级（P0）

**加原始任务锚点**
system prompt 始终包含第一条用户消息，约 10 行改动，不改变现有逻辑。

**加工具超时**
```typescript
Effect.timeout(item.execute(...), "30 seconds")
```
约 3 行改动，防止 shell 命令无限阻塞。

**加死循环检测**
记录最近 5 次工具调用，3 次相同时强制 break。约 15 行改动。

### 7.2 中优先级（P1）

**任务状态机**
新增 `TaskState` 接口，存储 original/current/completed/constraints，每轮以结构化数据注入。

**约束持久化**
从原始任务中提取不可变约束，每轮强制注入 system prompt。

**自反思循环**
每 5 轮执行一次元认知检查，判断是否偏离目标。

### 7.3 长期（P2）

- 分层执行：原始任务 → 子任务 → 工具调用
- 工作记忆：维护 facts/decisions/openQuestions
- 增量消息读取：只读新消息，缓存历史
- 拆分 prompt.ts

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
