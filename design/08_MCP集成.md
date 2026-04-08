# MCP 集成（Model Context Protocol）

---

## 1. 概述

MCP（Model Context Protocol）是 Anthropic 提出的标准化协议，用于连接 AI 模型与外部数据源和工具。DeerFlow 通过 `langchain-mcp-adapters` 实现 MCP 集成，可以动态加载任意 MCP 服务器提供的工具。

```
extensions_config.json
    │
    ▼
MultiServerMCPClient
    │
    ├── MCP Server 1（stdio/HTTP/SSE）
    ├── MCP Server 2（stdio）
    └── MCP Server 3（HTTP + OAuth）
            │
            ▼
    MCP Tools → Agent 工具集
```

---

## 2. 传输类型

### 2.1 stdio

通过子进程的标准输入/输出通信：

```json
{
  "type": "stdio",
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-github"],
  "env": {
    "GITHUB_TOKEN": "$GITHUB_TOKEN"
  }
}
```

### 2.2 HTTP / SSE

通过 HTTP 请求或 Server-Sent Events 通信：

```json
{
  "type": "http",
  "url": "https://api.example.com/mcp",
  "headers": {
    "Authorization": "Bearer ..."
  }
}
```

### 2.3 OAuth 支持

HTTP/SSE 传输支持 OAuth 2.0 自动刷新：

```json
{
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
```

支持两种授权模式：
- `client_credentials`：自动获取并刷新 token
- `refresh_token`：使用 refresh_token 获取 access_token

---

## 3. 客户端架构

```python
# packages/harness/deerflow/mcp/client.py

def build_servers_config(extensions_config: ExtensionsConfig) -> dict[str, dict]:
    """将 ExtensionsConfig 转换为 langchain-mcp-adapters 格式"""
    for server_name, config in enabled_servers:
        if config.type == "stdio":
            params = {
                "transport": "stdio",
                "command": config.command,
                "args": config.args,
                "env": config.env,
            }
        elif config.type in ("http", "sse"):
            params = {
                "transport": config.type,
                "url": config.url,
                "headers": config.headers,
            }
        servers_config[server_name] = params
```

### 3.1 OAuth Token 自动注入

```python
# packages/harness/deerflow/mcp/oauth.py

class OAuthInterceptor:
    """工具调用拦截器，自动刷新过期 token"""
    async def __call__(self, tool, *args, **kwargs):
        if token_expired(server_name):
            token = await refresh_token(server_name)
            inject_header(server_name, token)
        return await tool(*args, **kwargs)
```

---

## 4. 工具加载

```python
# packages/harness/deerflow/mcp/tools.py

async def get_mcp_tools() -> list[BaseTool]:
    """异步加载所有 MCP 工具"""
    extensions_config = ExtensionsConfig.from_file()
    servers_config = build_servers_config(extensions_config)

    client = MultiServerMCPClient(
        servers_config,
        tool_interceptors=[OAuthInterceptor(...)],
        tool_name_prefix=True,
    )

    tools = await client.get_tools()
    return tools
```

### 4.1 同步包装

MCP 工具默认是异步的，但 DeerFlow 工具系统需要同步调用：

```python
def _make_sync_tool_wrapper(coro, tool_name):
    def sync_wrapper(*args, **kwargs):
        if running_loop_exists():
            # 使用全局线程池执行
            future = _SYNC_TOOL_EXECUTOR.submit(asyncio.run, coro(*args, **kwargs))
            return future.result()
        else:
            return asyncio.run(coro(*args, **kwargs))
    return sync_wrapper
```

---

## 5. 缓存与失效

```python
# packages/harness/deerflow/mcp/cache.py

def get_cached_mcp_tools():
    """基于 extensions_config.json 的 mtime 自动刷新缓存"""
    # 读取文件的修改时间
    # 如果 mtime 变化 → 重新加载工具
    # 否则返回缓存
```

这样做的好处：
- Gateway 和 LangGraph 分别运行在不同进程
- 通过文件 mtime 检测配置变更，无需进程间通信

---

## 6. 延迟加载（Tool Search）

当 MCP 服务器暴露大量工具时，全部加载会：
- 消耗大量上下文 token
- 降低工具选择准确性

DeerFlow 支持**延迟加载**：

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
DeferredToolFilterMiddleware 停止过滤
Agent 调用该工具
```

启用方式：

```yaml
# config.yaml
tool_search:
  enabled: true
```

---

## 7. 配置

### 7.1 extensions_config.json

```json
{
  "mcpServers": {
    "github": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "$GITHUB_TOKEN"
      }
    },
    "filesystem": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/allowed/path"]
    },
    "secure-api": {
      "enabled": true,
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "X-API-Key": "$MY_API_KEY"
      },
      "oauth": {
        "enabled": true,
        "token_url": "https://auth.example.com/oauth/token",
        "grant_type": "client_credentials",
        "client_id": "$OAUTH_CLIENT_ID",
        "client_secret": "$OAUTH_CLIENT_SECRET"
      }
    }
  }
}
```

### 7.2 文件系统 MCP 允许路径

```python
# _get_mcp_allowed_paths()
# 解析 filesystem server 的 args，提取允许的绝对路径
for arg in args:
    if arg.startswith("/"):
        allowed_paths.append(arg.rstrip("/") + "/")
```

这些路径在 `validate_local_bash_command_paths` 中被允许作为 Bash 命令参数。

---

## 8. 源码索引

| 文件 | 说明 |
|------|------|
| `mcp/client.py` | 服务器配置构建 |
| `mcp/tools.py` | MCP 工具异步加载 + 同步包装 |
| `mcp/cache.py` | MCP 工具缓存（mtime 失效） |
| `mcp/oauth.py` | OAuth Token 自动刷新拦截器 |
| `mcp/manager.py` | MCP 管理器入口 |

---

*文档版本：v2.0 | 生成日期：2026/04/08*
