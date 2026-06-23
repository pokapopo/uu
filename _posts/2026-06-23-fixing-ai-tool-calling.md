---
layout: post
title: "修复 AI 工具调用：归一化存储、原子写入与防御性序列化"
date: 2026-06-23
permalink: /2026/06/23/fixing-ai-tool-calling
lang: zh-CN
---

在给 uuHalo 网关做工具调用（Tool Calling）重构时，我踩了三个坑，最终沉淀出一套**存储/传输分离**的架构模式。这篇文章把问题和解法讲清楚。

## 问题：你的 AI 网关有这三个漏洞

假设你的架构是：用户 → 网关 → LLM API → 网关收到 tool_calls → 执行工具 → 结果回传 LLM。

多数人第一次实现时会这样存数据库：

```sql
-- assistant 说"我要调 write_diary 和 search_web"
INSERT INTO messages (role, content, tool_calls)
VALUES ('assistant', null,
  '[{"id":"c1","name":"write_diary","arguments":"{...}"},
    {"id":"c2","name":"search_web","arguments":"{...}"}]');

-- 每个工具结果单独存一行
INSERT INTO messages (role, tool_call_id, tool_name, content)
VALUES ('tool', 'c1', 'write_diary', '{"ok":true}');
INSERT INTO messages (role, tool_call_id, tool_name, content)
VALUES ('tool', 'c2', 'search_web', '{"ok":true}');
```

这看起来很自然——每个 tool 结果是独立的一轮对话嘛。但它引爆三个问题：

### 漏洞 1：崩溃丢数据

用户说"帮我写日记"，网关收到 LLM 返回的 `write_diary` 调用。Assistant 行已经 INSERT 了，然后网关开始执行 `POST /api/diary`。这时候用户刷新了页面——`executeTool` 的 Promise 还在跑，但 HTTP 连接断了。**Assistant 行永远留在了 DB 里，tool 结果永远消失了。**

下次用户发消息，`buildContext` 从 DB 加载历史 → 发现一条 assistant 消息有 `tool_calls` 但没有对应的 `role: 'tool'` 结果 → 发给 LLM → **LLM 报 400 协议错误**。

### 漏洞 2：碎片化查询

每条 tool 结果独立 INSERT。加载一轮对话 = 1 条 assistant + N 条 tool = **N+1 条 SELECT**。前端渲染历史消息还要拼装关联，麻烦且慢。

### 漏洞 3：并发写入竞态

如果你用并行执行工具（`Promise.all`），N 条 `INSERT INTO messages (role: 'tool')` 之间互相独立，没问题。但如果你把全部 tool 结果塞进同一条记录的 JSONB 数组里，并行 UPDATE 就会互相覆盖，丢数据。

---

## 解法：三个核心设计

### 1. 归一化存储（内部 Domain Model）

不再用 `role: 'tool'` 的独立行。一行搞定一条 assistant 消息的全部工具调用 + 全部执行结果：

```sql
-- 归一化后，一行包含全部调用和结果
CREATE TABLE messages (
  id UUID PRIMARY KEY,
  role VARCHAR(20) NOT NULL,  -- 'user' | 'assistant' | 'system'
  content TEXT,
  tool_calls JSONB,  -- Array<{ id, name, arguments, output? }>
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 实际存储的数据
-- role: 'assistant'
-- tool_calls: [
--   { id: "c1", name: "write_diary", arguments: "{...}", output: "{\"ok\":true}" },
--   { id: "c2", name: "search_web",  arguments: "{...}", output: "{\"ok\":true}" }
-- ]
```

核心思想：**工具调用和结果不是两个独立实体，而是同一个交互的两个阶段。** 存成一行是符合领域语义的。

`role` 收拢为 `user | assistant | system`，`tool` 角色不再出现在存储层——它只是传输协议的产物。

### 2. 两段式原子写入（堵崩溃漏洞）

```
阶段 1：LLM 返回 tool_calls
  → INSERT assistant 行，output 全部为空
     { id, name, arguments }
     ↑ 第一时间存盘，拿到 recordId

阶段 2：逐个执行工具
  → for each tool_call (串行):
      result = await executeTool(tc)
      → UPDATE messages SET tool_calls[i].output = result
        WHERE id = recordId
      ↑ 每完成一个立刻补 output
```

**为什么串行？** 因为所有 tool_calls 都在同一条记录的同一个 JSONB 字段里。并行 UPDATE 同一行 = 竞态覆盖。串行 `for...of` 保证每次 UPDATE 看到的是最新状态——零竞态，零分布式锁。

**为什么不怕崩溃了？** 假设 executeTool 执行到第 2 个工具时进程被杀。DB 里残留的这条消息：tool_calls[0] 有 output，tool_calls[1] 没有 output。下次 buildContext 加载时——

### 3. 防御性序列化（堵 400 错误）

发送给 LLM 之前，必须把内部存储格式"翻译"成 OpenAI 线缆格式。这个翻译层做最后一公里安检：

```typescript
function serializeForProvider(rows: MessageRow[]): ChatMessage[] {
  for (const row of rows) {
    const tcs = row.toolCalls;

    // 🔒 半死不活检查：有 input 没 output → 整条丢弃
    if (tcs?.some(tc => tc.arguments && !('output' in tc))) {
      continue;  // 不发给 LLM
    }

    if (tcs?.length) {
      // 1. 发 assistant 消息（带 tool_calls）
      result.push({
        role: 'assistant',
        tool_calls: tcs.map(tc => ({
          id: tc.id,
          type: 'function',
          function: { name: tc.name, arguments: tc.arguments }
        }))
      });
      // 2. 逐条发 tool 结果
      for (const tc of tcs) {
        result.push({
          role: 'tool',
          tool_call_id: tc.id,
          content: tc.output
        });
      }
    } else {
      // 无工具调用，直通
      result.push({ role: row.role, content: row.content });
    }
  }
}
```

翻译过程：

```
DB 存储（一行归一化）          线缆格式（扁平化展开）
┌─────────────────────┐       ┌──────────────────────────┐
│ role: "assistant"   │       │ role: "assistant"        │
│ toolCalls: [        │  →→→  │ tool_calls: [            │
│   {id, name,        │       │   {id, type:"function",  │
│    arguments,       │       │    function:{name,args}} │
│    output: "..."},  │       │ ]                        │
│   {id, name,        │       │ role: "tool"             │
│    arguments,       │       │ tool_call_id: "c1"       │
│    output: "..."}   │       │ content: "{\"ok\":true}" │
│ ]                   │       │ role: "tool"             │
└─────────────────────┘       │ tool_call_id: "c2"       │
                               │ content: "{\"ok\":true}" │
                               └──────────────────────────┘
```

**关键点：** 内部存储格式（Domain Model）和传输协议（Wire Protocol）是两种不同的东西。数据库只管存，不管发。翻译逻辑集中在一个纯函数里，不散落在路由、适配器各处。

---

## 架构图

```
用户消息
    ↓
┌───────────────────────────────────────────┐
│  chat.ts (路由)                           │
│    1. INSERT user 消息                    │
│    2. buildContext() → 加载历史           │
│    3. 调 LLM API                          │
│    4. 收到 tool_calls?                    │
│       → INSERT assistant (output 全空)    │
│       → for each (串行):                   │
│           executeTool() → UPDATE output   │
│       → 继续循环                          │
│    5. INSERT 最终文本回复                  │
└───────────────────────────────────────────┘
    ↓ 读历史                           ↑ 写消息
┌───────────────────────────────────────────┐
│  serialize.ts (翻译层)                     │
│    DB JSONB ←→ OpenAI Wire Format         │
│    + 半死不活过滤（缺 output → 丢弃）       │
└───────────────────────────────────────────┘
    ↓ 序列化后
┌───────────────────────────────────────────┐
│  adapter.ts (AI 适配器)                    │
│    POST /chat/completions                 │
└───────────────────────────────────────────┘
```

---

## 为什么这个设计好

| 维度 | 旧方案 | 新方案 |
|------|--------|--------|
| **崩溃恢复** | 脏数据残留，LLM 报 400 | 残缺调用自动过滤，自愈 |
| **查询效率** | N+1 条 SELECT | 1 条 |
| **前端渲染** | 需要拼装 tool_call_id | 直接拿完整 JSONB |
| **加新厂商** | 改 adapter + context 两处 | 只改 serialize.ts |
| **并发安全** | 独立 INSERT 互不干扰 | 串行 UPDATE 同一行，零竞态 |
| **数据语义** | 调用和结果分家 | 一个交互一个实体 |

---

## 参考

设计理念受 [LobeChat](https://github.com/lobehub/lobehub) 的 `GroupMessageFlattenProcessor` 和 `ToolCallProcessor` 启发——它们用 `messages.tools` JSONB 归一化存储 + Processor Pipeline 做格式转换。uuHalo 的实现是 LobeChat 的简化直连版：去掉了多 Agent UI 需要的 `assistantGroup` 中间层，只保留最核心的存储/传输分离。

代码实现：[uuHalo-gateway/src/ai/serialize.ts](https://github.com/pokapopo/uuHalo/blob/master/uuHalo-gateway/src/ai/serialize.ts)

---

*标签: `#architecture` `#ai` `#tool-calling` `#gateway` `#database-design`*
