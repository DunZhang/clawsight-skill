# OpenClaw 对话数据速查

## 文件位置

```
~/.openclaw/agents/{agentId}/sessions/
├── sessions.json              # 元数据索引，记录所有 session 的基本信息
├── {sessionId}.jsonl           # 每个 session 的完整对话记录（一个文件 = 一段对话）
└── {sessionId}-topic-{threadId}.jsonl   # Telegram topic 专用
```

`{agentId}` 默认是 `main`。不确定的话跑一下：

```bash
find ~/.openclaw -name "sessions.json" -type f
```

---

## sessions.json 格式

JSON 文件。结构是 `{ sessionKey: SessionEntry, ... }` 的键值对。

```json
{
  "agent:main:main": {
    "sessionId": "95558fdc-f4dc-401e-91e0-202dffccb9cd",
    "updatedAt": 1710000000,
    "lastChannel": "telegram",
    "chatType": "direct",
    "model": "anthropic/claude-sonnet-4-5",
    "provider": "anthropic",
    "lastFrom": "user123",
    "lastTo": "bot1",
    "compactionCount": 2,
    "totalTokens": 26000,
    "inputTokens": 1523,
    "outputTokens": 87,
    "origin": {
      "label": "Alice",
      "provider": "telegram",
      "from": "user123",
      "to": "bot1"
    }
  },
  "agent:main:telegram:dm:456789": {
    "sessionId": "3c00d854-cb35-4571-9363-91b93c380f06",
    "..."
  }
}
```

用 `sessionId` 拼出 JSONL 文件名：`{sessionId}.jsonl`。如果条目里有 `sessionFile` 字段则优先用它。

---

## JSONL 对话记录格式

每行一个 JSON 对象，append-only。

### 第一行：session header

```json
{"type":"session","id":"95558fdc-...","cwd":"/home/user/project","timestamp":"2026-03-15T10:00:00.000Z"}
```

### 后续行：对话条目

每行都有 `id` 和 `parentId`（树形结构，支持分支）。核心是 `type: "message"` 的条目，通过 `message.role` 区分角色。

**用户消息** (`message.role: "user"`)：

```json
{
  "type": "message",
  "id": "entry-1",
  "parentId": "session-header-id",
  "timestamp": "2026-03-15T10:00:01.000Z",
  "message": {
    "role": "user",
    "content": [
      {"type": "text", "text": "帮我看一下目录下有什么文件"}
    ]
  }
}
```

**助手回复** (`message.role: "assistant"`)，可能包含工具调用：

```json
{
  "type": "message",
  "id": "entry-2",
  "parentId": "entry-1",
  "timestamp": "2026-03-15T10:00:03.000Z",
  "message": {
    "role": "assistant",
    "content": [
      {"type": "text", "text": "好的，让我查看一下。"},
      {"type": "tool_use", "id": "toolu_01ABC", "name": "run_command", "input": {"command": "ls -la"}}
    ],
    "usage": {
      "inputTokens": 1523,
      "outputTokens": 87,
      "cost": {"total": 0.0042}
    }
  }
}
```

**工具返回** (`message.role: "toolResult"`)：

```json
{
  "type": "message",
  "id": "entry-3",
  "parentId": "entry-2",
  "timestamp": "2026-03-15T10:00:04.000Z",
  "message": {
    "role": "toolResult",
    "toolCallId": "toolu_01ABC",
    "toolName": "run_command",
    "content": [
      {"type": "text", "text": "README.md\nsrc/\npackage.json"}
    ],
    "isError": false
  }
}
```

**Compaction 摘要** (`type: "compaction"`)：

```json
{
  "type": "compaction",
  "id": "compact-1",
  "parentId": "entry-40",
  "summary": "用户讨论了部署方案，决定使用 Docker...",
  "firstKeptEntryId": "entry-42",
  "tokensBefore": 95000
}
```

### content 数组中的 block 类型

| type 值 | 含义 |
|---------|------|
| `text` | 文本（`.text` 字段） |
| `tool_use` | 工具调用（`.name`, `.input`, `.id`） |
| `thinking` | 模型的 thinking 输出（`.text` 字段） |

---

## 快速验证

```bash
# 看有哪些 session
jq -r 'keys[]' ~/.openclaw/agents/main/sessions/sessions.json

# 看某个 session 的前 5 行
head -5 ~/.openclaw/agents/main/sessions/<sessionId>.jsonl | jq .

# 提取用户说了啥
jq -r 'select(.message.role == "user") | .message.content[]? | select(.type == "text") | .text' <sessionId>.jsonl

# 提取工具调用了啥
jq -r 'select(.message.role == "assistant") | .message.content[]? | select(.type == "tool_use") | .name' <sessionId>.jsonl | sort | uniq -c | sort -rn
```
