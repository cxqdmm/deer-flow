# Agent 系统（Lead Agent + 中间件链）

---

## 1. 概述

DeerFlow 的 Agent 系统基于 LangGraph 构建，核心是 **Lead Agent**——一个可配置的智能体，通过**中间件链**在每次请求中执行一系列横切关注点（cross-cutting concerns）。

```
请求 → Middleware 1 → Middleware 2 → ... → Middleware N → LLM + Tools
```

中间件在 LLM 调用前后执行，负责线程隔离、记忆管理、上下文压缩、图像处理等职责。

---

## 2. ThreadState（线程状态）

`ThreadState` 扩展了 LangGraph 的 `AgentState`，增加了 DeerFlow 特有的字段：

```python
class ThreadState(AgentState):
    # 继承自 AgentState
    messages: list[BaseMessage]      # 对话历史

    # DeerFlow 扩展
    sandbox: SandboxState             # {sandbox_id: str}
    thread_data: ThreadDataState     # {workspace_path, uploads_path, outputs_path}
    title: str | None               # 自动生成的对话标题
    artifacts: list[str]             # 呈现给用户的文件路径列表
    todos: list[dict] | None        # Plan Mode 任务列表
    uploaded_files: list[dict] | None # 上传文件元数据
    viewed_images: dict[str, ViewedImageData]  # {path: {base64, mime_type}}
```

### 2.1 自定义 Reducer

| 字段 | Reducer | 行为 |
|------|---------|------|
| `artifacts` | `merge_artifacts` | 追加去重，保持顺序 |
| `viewed_images` | `merge_viewed_images` | 合并字典；空 dict `{}` 清空所有 |

---

## 3. Lead Agent 工厂

```python
# packages/harness/deerflow/agents/lead_agent/agent.py

def make_lead_agent(config: RunnableConfig):
    """创建 Lead Agent"""
    return create_agent(
        model=create_chat_model(...),
        tools=get_available_tools(...),
        middleware=_build_middlewares(...),
        system_prompt=apply_prompt_template(...),
        state_schema=ThreadState,
    )
```

### 3.1 运行时配置参数

通过 `config.configurable` 传入：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `thinking_enabled` | bool | `True` | 启用模型的深度思考模式 |
| `model_name` | str | 第一个模型 | 指定使用的模型 |
| `reasoning_effort` | str | `None` | 模型推理努力程度（特定模型） |
| `is_plan_mode` | bool | `False` | 启用 Plan Mode（TodoList） |
| `subagent_enabled` | bool | `False` | 允许子 Agent 委托 |
| `max_concurrent_subagents` | int | `3` | 最大并发子 Agent 数 |
| `agent_name` | str | `None` | 自定义 Agent 名（对应 SOUL.md） |
| `is_bootstrap` | bool | `False` | 引导模式（创建自定义 Agent 时） |

---

## 4. 中间件链（Middleware Chain）

中间件按固定顺序执行（**不可调整顺序**，因为存在依赖关系）：

```
1. ThreadDataMiddleware      → 创建线程目录
2. UploadsMiddleware         → 注入上传文件列表
3. SandboxMiddleware         → 获取沙箱环境
4. DanglingToolCallMiddleware → 补全悬空工具调用
5. ToolErrorHandlingMiddleware → 工具异常转 ToolMessage
6. SummarizationMiddleware   → 上下文压缩（可选）
7. TodoListMiddleware        → Plan Mode 任务管理（可选）
8. TokenUsageMiddleware      → Token 使用统计（可选）
9. TitleMiddleware           → 自动生成标题
10. MemoryMiddleware          → 记忆更新入队
11. ViewImageMiddleware       → 图像 base64 注入（vision 模型）
12. DeferredToolFilterMiddleware → 过滤延迟工具 schema（tool_search）
13. SubagentLimitMiddleware  → 子 Agent 并发限制
14. LoopDetectionMiddleware   → 重复调用循环检测
15. [Custom Middlewares]      → 用户注入的自定义中间件
16. ClarificationMiddleware  → 截获澄清请求（最后）
```

### 4.1 各中间件详解

#### ThreadDataMiddleware

在 `before_agent` 阶段执行：

1. 从 `runtime.context.thread_id` 获取线程 ID
2. 调用 `Paths.create_thread_dirs(thread_id)` 创建目录结构：
   ```
   {base_dir}/threads/{thread_id}/
   ├── user-data/workspace/
   ├── user-data/uploads/
   └── user-data/outputs/
   ```
3. 将路径存入 `state.thread_data`

#### UploadsMiddleware

在 `after_model` 阶段执行：

1. 读取 `state.uploaded_files`
2. 格式化为文件列表字符串
3. 追加到 `messages` 末尾供 LLM 看到

#### SandboxMiddleware

在 `before_agent` 阶段执行：

1. 调用 `ensure_sandbox_initialized(runtime)` 获取沙箱
2. 将 `sandbox_id` 存入 `state.sandbox`

#### DanglingToolCallMiddleware

在 `after_model` 阶段执行：

- 检测 `AIMessage.tool_calls` 中缺少 `ToolMessage` 响应的情况
- 注入占位 `ToolMessage`（通常由用户中断触发）
- 防止 LLM 看到不完整的调用历史

#### ToolErrorHandlingMiddleware

在 `after_tool_call` 阶段执行：

- 将工具抛出的异常转换为格式化的 `ToolMessage`
- 确保工具错误不中断 Agent 执行流

#### SummarizationMiddleware（可选）

当 `config.yaml` 中 `summarization.enabled=true` 时启用：

- 在 `before_agent` 阶段检查 token 数量
- 超过阈值时调用 LLM 压缩旧消息
- 保留最近消息，压缩早期对话

#### TodoListMiddleware（可选）

当 `is_plan_mode=True` 时启用：

- 注册 `write_todos` 工具
- 工具用于管理复杂多步骤任务
- 规则：仅 1 个 `in_progress` 任务（非并行时）、完成后立即标记

#### TokenUsageMiddleware（可选）

当 `config.yaml` 中 `token_usage.enabled=true` 时启用：

- 记录每次 LLM 调用的输入/输出 token 数
- 日志输出

#### TitleMiddleware

在 `after_agent` 阶段执行：

- 仅首轮交换后执行一次（`state.title` 为空时）
- 调用 LLM 生成对话标题
- 支持结构化消息归一化

#### MemoryMiddleware

在 `after_agent` 阶段执行：

- 过滤消息（仅保留用户输入 + 最终 AI 响应）
- 提取 `correction_detected` / `reinforcement_detected` 信号
- 调用 `MemoryUpdateQueue.add()` 加入更新队列

#### ViewImageMiddleware（vision 模型时）

在 `before_agent` 阶段执行：

1. 读取 `state.viewed_images`
2. 查找未处理的图像
3. 调用 `view_image_tool` 获取 base64 数据
4. 将图像内容追加到 `messages`

#### DeferredToolFilterMiddleware（tool_search 时）

在 `before_model` 阶段执行：

- 读取 `<available-deferred-tools>` 中的工具名
- 从 `bind_tools()` 调用中过滤掉这些工具
- 节省上下文 token

#### SubagentLimitMiddleware

在 `after_model` 阶段执行：

- 检测 `task` 工具调用数量
- 超出 `max_concurrent_subagents` 时截断多余调用

#### LoopDetectionMiddleware

检测连续相同工具调用序列：

- 连续 3 次相同调用 → 警告
- 连续 5 次 → 中断并返回错误

#### ClarificationMiddleware（最后）

在 `after_tool_call` 阶段执行：

- 检测 `ask_clarification` 工具调用
- 发送 `Command(goto=END)` 中断 Agent 执行
- 前端展示澄清问题给用户

---

## 5. 系统提示词生成

`apply_prompt_template()` 生成完整的系统提示词，包含：

```
<system>
[DeerFlow 引导]
[Memory 上下文]
[Enabled Skills 列表]
[Tools 使用指南]
[Subagent 委托指南]
[Plan Mode 指南（is_plan_mode 时）]
[Vision 模型指南（supports_vision 时）]
</system>
```

### 5.1 Skills 注入

```python
apply_prompt_template(
    available_skills={"deep-research", "data-analysis"},
    ...
)
# 输出 <skills> 标签，列出启用的 Skill 及其描述和允许工具
```

### 5.2 Memory 上下文注入

```python
# 在 <memory> 标签中注入：
# - Top 15 事实（按置信度排序）
# - 用户上下文（workContext / personalContext / topOfMind）
# - 历史背景（recentMonths / earlierContext）
```

---

## 6. 源码索引

| 文件 | 说明 |
|------|------|
| `agents/lead_agent/agent.py` | Lead Agent 工厂 + 中间件链构建 |
| `agents/thread_state.py` | ThreadState 定义 + Reducer |
| `agents/lead_agent/prompt.py` | 系统提示词模板生成 |
| `agents/middlewares/` | 各中间件实现 |

---

*文档版本：v2.0 | 生成日期：2026/04/08*
