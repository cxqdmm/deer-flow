# 持久记忆系统（Memory）

---

## 1. 概述

DeerFlow 的记忆系统让 Agent 能够跨会话记住用户的信息、偏好和工作习惯。与传统 Key-Value 记忆不同，DeerFlow 使用 LLM 驱动的**自动提取**机制，从对话中分析并持久化结构化的用户知识。

```
用户对话
    │
    ▼
MemoryMiddleware（截取对话）
    │
    ▼
MemoryQueue（防抖 + 去重）
    │
    ▼
MemoryUpdater（LLM 提取）
    │
    ▼
memory.json（持久化）
    │
    ▼（下次对话）
Agent 系统提示词（Top 15 facts + context）
```

---

## 2. 记忆数据结构

```json
// backend/.deer-flow/memory.json
{
  "userContext": {
    "workContext": "用户的工作背景...",
    "personalContext": "用户的个人情况...",
    "topOfMind": "用户当前最关心的事情..."
  },
  "history": {
    "recentMonths": "最近几个月的重要事件...",
    "earlierContext": "早期背景...",
    "longTermBackground": "长期积累的背景..."
  },
  "facts": [
    {
      "id": "uuid",
      "content": "用户偏好使用 React 框架",
      "category": "preference",        // preference | knowledge | context | behavior | goal
      "confidence": 0.85,              // 0.0 - 1.0
      "createdAt": "2026-04-01T10:00:00Z",
      "source": "thread_abc123"
    }
  ],
  "meta": {
    "version": 1,
    "lastUpdated": "2026-04-08T10:00:00Z"
  }
}
```

### 2.1 事实类别

| 类别 | 说明 | 示例 |
|------|------|------|
| `preference` | 用户偏好 | "喜欢简洁的设计风格" |
| `knowledge` | 用户知识 | "熟悉 Python 和 React" |
| `context` | 上下文信息 | "正在开发一个电商项目" |
| `behavior` | 行为模式 | "通常在周末处理文档" |
| `goal` | 目标/计划 | "计划下个月上线产品" |

---

## 3. MemoryMiddleware

```python
# packages/harness/deerflow/agents/memory/middleware.py

class MemoryMiddleware:
    def after_agent(self, messages, state, config):
        # 过滤消息：仅保留用户输入 + 最终 AI 响应
        # 提取 correction_detected / reinforcement_detected 信号
        # 调用 MemoryUpdateQueue.add()
```

### 3.1 消息过滤

```python
def _filter_messages_for_memory(messages):
    """仅保留用于记忆提取的消息"""
    filtered = []
    for msg in messages:
        if isinstance(msg, HumanMessage):
            filtered.append(msg)       # 保留用户输入
        elif isinstance(msg, AIMessage):
            # 检查是否是最终响应（非工具调用结果）
            if not has_tool_calls(msg):
                filtered.append(msg)   # 保留最终 AI 响应
    return filtered
```

### 3.2 强化与修正信号

- `correction_detected`：用户在对话中纠正了 Agent 的错误
- `reinforcement_detected`：用户对 Agent 的回答表示满意/确认

这些信号传递给 MemoryUpdater，用于调整事实置信度。

---

## 4. MemoryQueue（防抖队列）

```python
# packages/harness/deerflow/agents/memory/queue.py

class MemoryUpdateQueue:
    def add(self, thread_id, messages, agent_name,
            correction_detected, reinforcement_detected):
        # 防抖：30s（可配置）后批量处理
        # per-thread 去重：同一 thread 只保留最新消息
```

### 4.1 防抖机制

```python
# 收到对话后，启动/重置 30s 定时器
# 定时器到期后，批量处理队列中所有对话
_timer = threading.Timer(debounce_seconds, _process_batch)
```

---

## 5. MemoryUpdater（LLM 驱动提取）

```python
# packages/harness/deerflow/agents/memory/updater.py

class MemoryUpdater:
    def update(self, thread_id, messages, agent_name,
               correction_detected, reinforcement_detected):
        # 调用 LLM 提取上下文更新 + 新事实
        # 原子写入 memory.json（临时文件 + rename）
```

### 5.1 提示词模板

```python
# packages/harness/deerflow/agents/memory/prompt.py

MEMORY_UPDATE_PROMPT = """
Given the following conversation between a user and an AI assistant,
extract updates to the user's memory.

Current memory:
{existing_memory}

Conversation:
{formatted_conversation}

Extract:
1. Updates to userContext (workContext, personalContext, topOfMind)
2. New facts about the user (preferences, knowledge, context, behavior, goals)
3. Updates to existing facts (if user corrected or reinforced previous knowledge)
"""
```

### 5.2 事实去重

```python
def apply_facts(memory_data, new_facts):
    # whitespace-normalized 去重
    existing = {f["content"].strip(): f for f in memory_data["facts"]}
    for fact in new_facts:
        key = fact["content"].strip()
        if key not in existing:
            # 新事实，添加
            memory_data["facts"].append(fact)
        elif fact["confidence"] > existing[key]["confidence"]:
            # 更高置信度，更新
            memory_data["facts"][index] = fact
```

### 5.3 原子写入

```python
def save_memory(memory_data):
    # 1. 写入临时文件
    tmp_path = path.with_suffix(".tmp")
    tmp_path.write_text(json.dumps(memory_data))
    # 2. rename（原子操作）
    tmp_path.rename(path)
```

---

## 6. 注入系统提示词

```python
# packages/harness/deerflow/agents/lead_agent/prompt.py

def get_memory_context() -> str:
    """生成 <memory> 标签内容"""
    facts = get_top_facts(limit=15, threshold=0.7)
    return f"""
<memory>
User Context:
{userContext}

Recent Facts:
{facts_list}

History:
{history}
</memory>
"""
```

---

## 7. 配置

### 7.1 config.yaml

```yaml
memory:
  enabled: true                    # 记忆总开关
  injection_enabled: true         # 注入到系统提示词
  storage_path: .deer-flow/memory.json
  debounce_seconds: 30           # 防抖等待时间
  model_name: null                # 使用默认模型（null）或指定模型
  max_facts: 100                  # 最多保留的事实数
  fact_confidence_threshold: 0.7  # 注入提示词的最低置信度
  max_injection_tokens: 2000      # 提示词中记忆的最大 token 数
```

---

## 8. Per-Agent 记忆

当使用自定义 Agent 时（`agent_name` 参数），每个 Agent 有独立的记忆文件：

```
.deer-flow/
├── memory.json              # 全局记忆
└── agents/
    └── {agent_name}/
        └── memory.json       # Agent 专属记忆
```

MemoryMiddleware 根据运行时 `agent_name` 参数选择正确的记忆存储。

---

## 9. 源码索引

| 文件 | 说明 |
|------|------|
| `agents/memory/middleware.py` | MemoryMiddleware |
| `agents/memory/updater.py` | MemoryUpdater（LLM 提取） |
| `agents/memory/queue.py` | MemoryQueue（防抖队列） |
| `agents/memory/prompt.py` | 记忆提取提示词模板 |
| `agents/memory/storage.py` | 记忆文件读写（支持 per-agent） |
| `agents/memory/types.py` | 记忆数据类型定义 |

---

*文档版本：v2.0 | 生成日期：2026/04/08*
