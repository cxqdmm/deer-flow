# 子 Agent 委托系统（Subagents）

---

## 1. 概述

DeerFlow 的 Lead Agent 可以将复杂任务委托给 **Sub-Agent**（子 Agent）执行。子 Agent 运行在独立上下文中，支持并行执行，从而实现复杂任务的高效分解与处理。

```
Lead Agent
    │
    └── task(description, prompt, subagent_type)
            │
            ├── general-purpose  ──→ 全功能 Agent
            └── bash              ──→ 命令执行专家
```

---

## 2. 内置子 Agent 类型

### 2.1 general-purpose

功能完整的通用子 Agent：

- 拥有 Lead Agent 的全部工具集（不含 `task` 工具，防止嵌套）
- 拥有 Lead Agent 的 Skills
- 适用于复杂多步骤任务

### 2.2 bash

命令执行专家：

- 仅暴露沙箱文件操作工具
- 适用于纯命令执行场景
- 仅在 `allow_host_bash=true` 或使用 `AioSandboxProvider` 时可用

---

## 3. 执行引擎

```python
# packages/harness/deerflow/subagents/executor.py

class SubagentExecutor:
    """子 Agent 执行器"""

    def __init__(self, config, tools, parent_model, sandbox_state,
                 thread_data, thread_id, trace_id):
        ...

    def execute_async(self, prompt: str, task_id: str | None = None) -> str:
        """异步执行子 Agent，返回 task_id"""
```

### 3.1 执行状态

```python
class SubagentStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"
    TIMED_OUT = "timed_out"
```

### 3.2 执行结果

```python
@dataclass
class SubagentResult:
    task_id: str
    trace_id: str
    status: SubagentStatus
    result: str | None        # 最终结果
    error: str | None          # 错误信息
    started_at: datetime | None
    completed_at: datetime | None
    ai_messages: list[dict]    # 执行过程中的 AI 消息
```

---

## 4. 线程池架构

```
┌─────────────────────────────────────────────┐
│         _scheduler_pool（3 workers）           │
│    负责任务调度 + 结果轮询                      │
└────────────────────┬────────────────────────┘
                     │  submit(execution_task)
                     ▼
┌─────────────────────────────────────────────┐
│         _execution_pool（3 workers）          │
│    实际运行 sub-agent（支持超时）              │
└─────────────────────────────────────────────┘

并发限制：
  MAX_CONCURRENT_SUBAGENTS = 3
  每个任务超时 = 15 分钟
  轮询间隔 = 5 秒
```

---

## 5. task 工具

```python
# packages/harness/deerflow/tools/builtins/task_tool.py

@tool("task")
async def task_tool(
    runtime: ToolRuntime,
    description: str,         # 任务简短描述（3-5 词）
    prompt: str,              # 详细任务描述
    subagent_type: str,       # "general-purpose" 或 "bash"
    tool_call_id: Annotated[str, InjectedToolCallId],
    max_turns: int | None = None,  # 最大 Agent 轮次
) -> str:
    """委托任务给子 Agent"""
```

### 5.1 执行流程

```
task_tool()
    │
    ▼
SubagentExecutor.execute_async(prompt, task_id=tool_call_id)
    │
    ▼
写入 _background_tasks[task_id] = SubagentResult(status=PENDING)
    │
    ▼
_scheduler_pool.submit(_run_subagent_async)
    │
    ▼
_execution_pool 实际执行
    │
    ▼
轮询 _background_tasks[task_id]
    │
    ├─ RUNNING → 发送 task_running 事件
    │
    ├─ COMPLETED → 发送 task_completed → 返回结果
    │
    ├─ FAILED → 发送 task_failed → 返回错误
    │
    └─ TIMED_OUT → 发送 task_timed_out → 返回超时信息
```

### 5.2 SSE 事件流

| 事件 | 内容 |
|------|------|
| `task_started` | `task_id`, `description` |
| `task_running` | `task_id`, `message`, `message_index`, `total_messages` |
| `task_completed` | `task_id`, `result` |
| `task_failed` | `task_id`, `error` |
| `task_timed_out` | `task_id`, `error` |
| `task_cancelled` | `task_id` |

---

## 6. 子 Agent 限流

`SubagentLimitMiddleware` 在 `after_model` 阶段截断超额调用：

```python
if subagent_enabled:
    max_concurrent_subagents = config.get("configurable", {}).get("max_concurrent_subagents", 3)
    middlewares.append(SubagentLimitMiddleware(max_concurrent=max_concurrent_subagents))
```

当 LLM 一次生成超过 `N` 个 `task` 调用时：
- 保留前 `N` 个
- 其余调用返回错误信息

---

## 7. 工具过滤

SubagentExecutor 创建时自动过滤工具：

```python
def _filter_tools(all_tools, allowed, disallowed):
    # 移除 task 工具（防止嵌套）
    # 按 allowed/disallowed 配置进一步过滤
    return filtered_tools
```

---

## 8. 子 Agent 注册表

```python
# packages/harness/deerflow/subagents/registry.py

def load_subagent_builtin(name: str) -> SubagentConfig:
    """加载内置子 Agent 配置"""

# 内置：
#   "general-purpose" → 全部工具（除 task）
#   "bash" → 仅文件/命令工具
```

---

## 9. 配置

### 9.1 config.yaml

```yaml
subagents:
  enabled: true          # 总开关
  general-purpose:
    timeout_seconds: 900  # 15 分钟超时
    max_turns: 50
  bash:
    timeout_seconds: 900
    max_turns: 50
```

### 9.2 运行时参数

```python
# 通过 config.configurable 传入
{
    "subagent_enabled": true,
    "max_concurrent_subagents": 3,
}
```

---

## 10. 源码索引

| 文件 | 说明 |
|------|------|
| `subagents/executor.py` | SubagentExecutor + 线程池 |
| `subagents/registry.py` | 子 Agent 注册表 |
| `subagents/builtins/general_purpose.py` | general-purpose 配置 |
| `subagents/builtins/bash.py` | bash 配置 |
| `subagents/config.py` | SubagentConfig 数据类 |
| `tools/builtins/task_tool.py` | task 工具实现 |
| `agents/middlewares/subagent_limit_middleware.py` | 并发限制中间件 |

---

*文档版本：v2.0 | 生成日期：2026/04/08*
