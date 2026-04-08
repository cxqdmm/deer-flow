# 工具系统（Tools）

本文档详细描述 DeerFlow 的工具系统架构、工具分类、源码结构和配置方式。

---

## 1. 概述

DeerFlow 的工具系统是 Agent 与外部世界交互的核心接口。工具按职责分为四类：

| 分类 | 说明 | 源码路径 |
|------|------|---------|
| **沙箱工具** | 文件系统操作、Shell 执行 | `deerflow/sandbox/tools.py` |
| **内置工具** | Agent 内部能力（提问、呈现、子 Agent 等） | `deerflow/tools/builtins/` |
| **社区工具** | Web 搜索/抓取、图像搜索 | `deerflow/community/*/tools.py` |
| **MCP 工具** | 第三方 Model Context Protocol 服务器 | `deerflow/mcp/tools.py` |

所有工具均实现 LangChain 的 `BaseTool` 接口，通过 `get_available_tools()` 统一组装后绑定到 Lead Agent。

---

## 2. 工具加载机制

### 2.1 统一入口

```
get_available_tools()
├── 1. config.yaml 定义的工具（通过 resolve_variable 动态加载）
│       ├── group=bash → bash_tool（可按 sandbox 配置禁用）
│       ├── group=search → web_search_tool
│       ├── group=web → web_fetch_tool
│       └── group=file → ls/read/write/glob/grep/str_replace
│
├── 2. 内置工具（始终加载，无配置依赖）
│       ├── present_file_tool（始终）
│       ├── ask_clarification_tool（始终）
│       ├── view_image_tool（仅当模型支持 vision）
│       ├── task_tool（仅当 subagent_enabled=True）
│       ├── setup_agent（仅当 skill_evolution.enabled=True）
│       └── skill_manage_tool（仅当 skill_evolution.enabled=True）
│
├── 3. MCP 工具（从 extensions_config.json 读取配置，延迟加载）
│
└── 4. ACP Agent 工具（从 config.yaml acp_agents 读取）
```

### 2.2 源码入口

```python
# packages/harness/deerflow/tools/__init__.py
from .tools import get_available_tools

# packages/harness/deerflow/tools/tools.py
def get_available_tools(
    groups: list[str] | None = None,
    include_mcp: bool = True,
    model_name: str | None = None,
    subagent_enabled: bool = False,
) -> list[BaseTool]:
    """组装所有可用工具"""
```

### 2.3 沙箱 Bash 安全策略

当使用 `LocalSandboxProvider` 时，主机 Bash 默认**禁用**。判断逻辑：

```python
# packages/harness/deerflow/sandbox/security.py
def is_host_bash_allowed() -> bool:
    # sandbox.allow_host_bash = true → 允许
    # AioSandboxProvider（Docker 模式）→ 允许
    # LocalSandboxProvider + allow_host_bash 未设置 → 禁用
```

---

## 3. 沙箱工具（Sandbox Tools）

### 3.1 工具列表

| 工具名 | 函数 | 用途 |
|--------|------|------|
| `bash` | `bash_tool` | 执行 Shell 命令 |
| `ls` | `ls_tool` | 列出目录内容（树形，最多 2 层） |
| `read_file` | `read_file_tool` | 读取文件内容（支持行范围） |
| `write_file` | `write_file_tool` | 写入文件（支持追加模式） |
| `str_replace` | `str_replace_tool` | 替换文件中的字符串 |
| `glob` | `glob_tool` | 按模式匹配文件路径 |
| `grep` | `grep_tool` | 在文件中搜索文本/正则 |

### 3.2 虚拟路径系统

沙箱工具通过虚拟路径与物理路径的双向映射实现线程隔离：

| 虚拟路径（Agent 可见） | 物理路径 |
|------------------------|---------|
| `/mnt/user-data/workspace` | `{base_dir}/threads/{thread_id}/user-data/workspace` |
| `/mnt/user-data/uploads` | `{base_dir}/threads/{thread_id}/user-data/uploads` |
| `/mnt/user-data/outputs` | `{base_dir}/threads/{thread_id}/user-data/outputs` |
| `/mnt/skills` | `deer-flow/skills/`（只读） |
| `/mnt/acp-workspace` | `{base_dir}/threads/{thread_id}/acp-workspace/`（只读） |

**路径替换函数**：

```python
# packages/harness/deerflow/sandbox/tools.py

# Agent 视角 → 物理路径
replace_virtual_path(path, thread_data)

# 物理路径 → Agent 视角（输出掩码）
mask_local_paths_in_output(output, thread_data)

# 命令字符串中的路径替换
replace_virtual_paths_in_command(command, thread_data)
```

### 3.3 输出截断机制

| 工具 | 截断策略 | 默认上限 |
|------|---------|---------|
| `bash` | 中间截断（保留首尾） | 20,000 字符 |
| `read_file` | 头部截断 | 50,000 字符 |
| `ls` | 头部截断 | 20,000 字符 |
| `glob` | 受 `max_results` 限制 | 200 条 |
| `grep` | 受 `max_results` 限制 | 100 条 |

### 3.4 路径安全验证

```python
# packages/harness/deerflow/sandbox/tools.py

validate_local_tool_path(path, thread_data, read_only=False)
# 允许：
#   /mnt/user-data/* — 读写
#   /mnt/skills/* — 只读
#   /mnt/acp-workspace/* — 只读
#   自定义挂载路径 — 按配置决定读写权限

validate_local_bash_command_paths(command, thread_data)
# 仅允许：
#   /mnt/user-data/* 等虚拟路径
#   白名单系统路径（/bin/, /usr/bin/, /dev/ 等）
#   MCP filesystem server 允许的路径
# 阻止 file:// URL 和路径遍历（..）
```

---

## 4. 内置工具（Built-in Tools）

### 4.1 present_files

```python
# packages/harness/deerflow/tools/builtins/present_file_tool.py

@tool("present_files")
def present_file_tool(filepaths: list[str], tool_call_id) -> Command:
    """使文件对用户可见"""
    # 将文件路径规范化为 /mnt/user-data/outputs/* 后更新 artifacts 状态
```

### 4.2 ask_clarification

```python
# packages/harness/deerflow/tools/builtins/clarification_tool.py

@tool("ask_clarification", return_direct=True)
def ask_clarification_tool(question, clarification_type, context, options) -> str:
    """向用户请求澄清"""
    # ClarificationMiddleware 拦截此工具调用并中断执行
    return "Clarification request processed by middleware"
```

### 4.3 view_image

```python
# packages/harness/deerflow/tools/builtins/view_image_tool.py

@tool("view_image")
def view_image_tool(image_path, tool_call_id) -> Command:
    """读取图像文件并注入 base64 数据到 viewed_images"""
    # 仅当模型 supports_vision=True 时加载
    # 支持格式：jpg, jpeg, png, webp
```

### 4.4 task（子 Agent 委托）

```python
# packages/harness/deerflow/tools/builtins/task_tool.py

@tool("task")
async def task_tool(description, prompt, subagent_type, tool_call_id, max_turns) -> str:
    """委托任务给子 Agent"""
    # 启动后台执行 → 轮询状态 → 返回结果
    # 事件流：task_started → task_running → task_completed/failed/timed_out
```

### 4.5 setup_agent

```python
# packages/harness/deerflow/tools/builtins/setup_agent_tool.py

@tool
def setup_agent(soul, description, runtime) -> Command:
    """创建自定义 DeerFlow Agent（SOUL.md + config.yaml）"""
    # 写入 {agent_dir}/SOUL.md 和 config.yaml
```

### 4.6 skill_manage（实验性）

```python
# packages/harness/deerflow/tools/skill_manage_tool.py

@tool("skill_manage")
async def skill_manage_tool(action, name, content, path, find, replace) -> str:
    """管理 custom 技能：create/edit/patch/delete/write_file/remove_file"""
    # 仅当 skill_evolution.enabled=True 时可用
```

---

## 5. 社区工具（Community Tools）

社区工具通过 `config.yaml` 中的 `tools[].use` 字段配置。

### 5.1 Web 搜索工具

| 提供者 | 工具 | 源码 | 需要 API Key |
|--------|------|------|-------------|
| **DuckDuckGo**（默认） | `web_search` | `deerflow/community/ddg_search/` | 否 |
| **Tavily** | `web_search` | `deerflow/community/tavily/` | `TAVILY_API_KEY` |
| **Firecrawl** | `web_search` | `deerflow/community/firecrawl/` | `FIRECRAWL_API_KEY` |
| **InfoQuest**（字节） | `web_search` | `deerflow/community/infoquest/` | `INFOQUEST_API_KEY` |

### 5.2 Web 抓取工具

| 提供者 | 工具 | 源码 | 特点 |
|--------|------|------|------|
| **Jina AI**（默认） | `web_fetch` | `deerflow/community/jina_ai/` | 可读性提取，转 Markdown |
| **Tavily** | `web_fetch` | `deerflow/community/tavily/` | raw_content 提取 |
| **Firecrawl** | `web_fetch` | `deerflow/community/firecrawl/` | Markdown 格式 |
| **InfoQuest** | `web_fetch` | `deerflow/community/infoquest/` | 可读性提取 |

### 5.3 图像搜索工具

| 提供者 | 工具 | 源码 | 需要 API Key |
|--------|------|------|-------------|
| **DuckDuckGo**（默认） | `image_search` | `deerflow/community/image_search/` | 否 |
| **InfoQuest** | `image_search` | `deerflow/community/infoquest/` | `INFOQUEST_API_KEY` |

### 5.4 InfoQuest 高级配置

```yaml
tools:
  - name: web_search
    use: deerflow.community.infoquest.tools:web_search_tool
    tool_extra:
      search_time_range: 30d    # 搜索时间范围（天）
      # 无配置则不限制

  - name: web_fetch
    use: deerflow.community.infoquest.tools:web_fetch_tool
    tool_extra:
      fetch_time: 30d           # 抓取页面的时间限制
      timeout: 30               # 请求超时（秒）
      navigation_timeout: 60    # 页面导航超时（秒）

  - name: image_search
    use: deerflow.community.infoquest.tools:image_search_tool
    tool_extra:
      image_search_time_range: 30d
      image_size: "i"           # 图像尺寸（i/m/l/w/h）
```

---

## 6. MCP 工具（MCP Tools）

### 6.1 架构

```
extensions_config.json
    ↓
ExtensionsConfig.from_file()
    ↓
build_servers_config() → servers_config dict
    ↓
MultiServerMCPClient(servers_config)
    ↓
client.get_tools() → list[BaseTool]
    ↓
缓存 + mtime 失效
```

### 6.2 支持的传输类型

| 传输类型 | 配置字段 | OAuth 支持 |
|---------|---------|-----------|
| `stdio` | `command`, `args`, `env` | 否 |
| `http` | `url`, `headers` | `client_credentials`, `refresh_token` |
| `sse` | `url`, `headers` | `client_credentials`, `refresh_token` |

### 6.3 延迟初始化与缓存

```python
# packages/harness/deerflow/mcp/cache.py
def get_cached_mcp_tools() -> list[BaseTool]:
    """基于 extensions_config.json 的 mtime 自动刷新缓存"""

# packages/harness/deerflow/mcp/tools.py
async def get_mcp_tools() -> list[BaseTool]:
    """加载所有启用 MCP 服务器的工具"""
    # OAuth 拦截器自动注入 Authorization header
    # 异步工具同步包装器（_make_sync_tool_wrapper）
```

### 6.4 配置示例

```json
// extensions_config.json
{
  "mcpServers": {
    "github": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "$GITHUB_TOKEN" }
    },
    "filesystem": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/allowed/path"]
    },
    "secure-http": {
      "enabled": true,
      "type": "http",
      "url": "https://api.example.com/mcp",
      "oauth": {
        "enabled": true,
        "token_url": "https://auth.example.com/oauth/token",
        "grant_type": "client_credentials",
        "client_id": "$MCP_OAUTH_CLIENT_ID",
        "client_secret": "$MCP_OAUTH_CLIENT_SECRET"
      }
    }
  }
}
```

### 6.5 OAuth Token 自动刷新

```python
# packages/harness/deerflow/mcp/oauth.py
class OAuthInterceptor:
    """在每个工具调用前检查 token 有效性，过期则自动刷新"""
```

---

## 7. Tool Search（工具搜索）

### 7.1 问题背景

当 MCP 服务器暴露大量工具时，全部加载到 Agent 上下文会：
- 消耗大量 Token
- 降低工具选择准确性

### 7.2 解决方案：延迟工具加载

```
Agent 系统提示词
    ↓
<available-deferred-tools>
    github_search, github_create_issue, github_add_comment, ...
    ↓（仅工具名，无 schema）
Agent 调用 tool_search("github")
    ↓
DeferredToolRegistry.search("github")
    ↓
返回匹配的完整 schema（OpenAI function format）
    ↓
DeferredToolFilterMiddleware 停止过滤该工具
Agent 可以调用该工具
```

### 7.3 搜索查询格式

| 格式 | 含义 | 示例 |
|------|------|------|
| `select:name1,name2` | 精确名称选择 | `select:Read,Edit` |
| `+keyword rest` | 名称必须含 keyword | `+slack send message` |
| `keyword query` | 正则匹配名称+描述 | `github issue create` |

### 7.4 启用方式

```yaml
# config.yaml
tool_search:
  enabled: true
```

---

## 8. ACP Agent 工具

### 8.1 架构

```
config.yaml: acp_agents
    ↓
build_invoke_acp_agent_tool(agents)
    ↓
ACP Client（agent-client-protocol 包）
    ↓
spawn_agent_process() → stdio 通信
    ↓
per-thread ACP workspace: {base_dir}/threads/{thread_id}/acp-workspace/
```

### 8.2 配置

```yaml
# config.yaml
acp_agents:
  codex:
    description: "Codex CLI agent for code tasks"
    command: "npx"
    args: ["-y", "@zed-industries/codex-acp"]
    # auto_approve_permissions: false
    # env: {}
    # model: "gpt-4o"
```

### 8.3 工具说明

```python
@tool("invoke_acp_agent")
def invoke_acp_agent_tool(agent: str, prompt: str) -> str:
    """调用外部 ACP 兼容 Agent"""
    # 每个线程独立 workspace
    # MCP 服务器自动传递给 ACP session
    # 权限请求：auto_approve 或 cancel
```

---

## 9. 工具配置

### 9.1 config.yaml 结构

```yaml
# 工具分组
tool_groups:
  - name: search
    tools: ["web_search"]
  - name: web
    tools: ["web_fetch"]
  - name: file
    tools: ["ls", "read_file", "write_file", "glob", "grep", "str_replace"]
  - name: bash
    tools: ["bash"]

# 工具定义
tools:
  - name: web_search
    use: deerflow.community.ddg_search.tools:web_search_tool
    group: search
    # tool_extra: {}

  - name: web_fetch
    use: deerflow.community.jina_ai.tools:web_fetch_tool
    group: web

  - name: bash
    use: deerflow.sandbox.tools:bash_tool
    group: bash

# 工具搜索（延迟加载）
tool_search:
  enabled: false

# ACP Agents
acp_agents: {}
```

### 9.2 每个工具的额外配置

```yaml
tools:
  - name: web_search
    use: deerflow.community.tavily.tools:web_search_tool
    tool_extra:
      api_key: $TAVILY_API_KEY  # 覆盖全局
      max_results: 5            # 搜索结果数上限
```

### 9.3 沙箱 Bash 配置

```yaml
# config.yaml
sandbox:
  use: deerflow.sandbox.local:LocalSandboxProvider
  allow_host_bash: false  # 默认禁用，Docker 模式自动允许
```

---

## 10. 工具执行流程

### 10.1 工具调用生命周期

```
Agent LLM 生成 tool_calls
    ↓
PreToolUse Hook（可选）
    ↓
GuardrailMiddleware.evaluate()（可选）
    ↓
BaseTool.execute() / tool.coroutine()
    ↓
沙箱路径验证（validate_local_tool_path）
    ↓
沙箱执行（LocalSandbox / AioSandbox）
    ↓
输出掩码（mask_local_paths_in_output）
    ↓
PostToolUse Hook（可选）
    ↓
ToolMessage 返回给 LLM
```

### 10.2 本地沙箱执行路径

```
bash_tool(runtime, command)
    ↓
ensure_sandbox_initialized(runtime) → LocalSandbox
    ↓
is_host_bash_allowed() → False? return 错误
    ↓
validate_local_bash_command_paths(command, thread_data)
    ↓
replace_virtual_paths_in_command(command, thread_data)
    ↓
_apply_cwd_prefix(command, thread_data)
    ↓
sandbox.execute_command(command)
    ↓
_truncate_bash_output(mask_local_paths_in_output(output), max_chars)
```

---

## 11. 源码文件索引

| 功能 | 文件路径 |
|------|---------|
| 工具加载入口 | `packages/harness/deerflow/tools/__init__.py` |
| 工具组装函数 | `packages/harness/deerflow/tools/tools.py` |
| 沙箱工具（bash/ls/read/write/glob/grep） | `packages/harness/deerflow/sandbox/tools.py` |
| 内置工具入口 | `packages/harness/deerflow/tools/builtins/__init__.py` |
| present_files | `packages/harness/deerflow/tools/builtins/present_file_tool.py` |
| ask_clarification | `packages/harness/deerflow/tools/builtins/clarification_tool.py` |
| view_image | `packages/harness/deerflow/tools/builtins/view_image_tool.py` |
| task（子 Agent） | `packages/harness/deerflow/tools/builtins/task_tool.py` |
| setup_agent | `packages/harness/deerflow/tools/builtins/setup_agent_tool.py` |
| invoke_acp_agent | `packages/harness/deerflow/tools/builtins/invoke_acp_agent_tool.py` |
| tool_search | `packages/harness/deerflow/tools/builtins/tool_search.py` |
| skill_manage | `packages/harness/deerflow/tools/skill_manage_tool.py` |
| MCP 工具加载 | `packages/harness/deerflow/mcp/tools.py` |
| MCP OAuth | `packages/harness/deerflow/mcp/oauth.py` |
| DuckDuckGo 搜索 | `packages/harness/deerflow/community/ddg_search/tools.py` |
| DuckDuckGo 图像搜索 | `packages/harness/deerflow/community/image_search/tools.py` |
| Tavily 搜索/抓取 | `packages/harness/deerflow/community/tavily/tools.py` |
| Firecrawl 搜索/抓取 | `packages/harness/deerflow/community/firecrawl/tools.py` |
| Jina AI 抓取 | `packages/harness/deerflow/community/jina_ai/tools.py` |
| InfoQuest 搜索/抓取 | `packages/harness/deerflow/community/infoquest/tools.py` |

---

*文档版本：v2.0 | 生成日期：2026/04/08*
