# Gateway API（FastAPI REST）

---

## 1. 概述

Gateway API 是 DeerFlow 的 REST API 层，基于 FastAPI 构建，提供模型、MCP、Skills、Memory、文件上传、Artifact 等管理接口。前端和外部系统通过 Gateway 与 DeerFlow 交互。

```
Nginx (2026)
    │
    └─ /api/* → Gateway (8001)
```

Gateway 同时负责 IM 频道服务的生命周期管理和 Agent 运行时的初始化（Gateway 模式）。

---

## 2. 应用入口

```python
# app/gateway/app.py

@asynccontextmanager
async def lifespan(app: FastAPI):
    # 1. 加载配置
    get_app_config()
    # 2. 初始化 LangGraph Runtime（StreamBridge + RunManager）
    async with langgraph_runtime(app):
        # 3. 启动 IM Channel 服务
        channel_service = await start_channel_service()
        yield
        # 4. 关闭 IM Channel 服务
        await channel_service.stop()
```

---

## 3. 路由总览

| 路由模块 | 路径 | 方法 | 说明 |
|---------|------|------|------|
| `models` | `/api/models` | GET | 列出模型 |
| `mcp` | `/api/mcp/config` | GET/PUT | MCP 配置 |
| `skills` | `/api/skills` | GET | 列出技能 |
| `skills` | `/api/skills/{name}` | GET/PUT | 技能详情/更新 |
| `skills` | `/api/skills/install` | POST | 安装 .skill 归档 |
| `memory` | `/api/memory` | GET | 记忆数据 |
| `memory` | `/api/memory/reload` | POST | 强制重载 |
| `memory` | `/api/memory/config` | GET | 记忆配置 |
| `memory` | `/api/memory/status` | GET | 记忆状态（配置+数据） |
| `uploads` | `/api/threads/{id}/uploads` | POST | 上传文件 |
| `uploads` | `/api/threads/{id}/uploads/list` | GET | 列出上传文件 |
| `uploads` | `/api/threads/{id}/uploads/{filename}` | DELETE | 删除上传文件 |
| `threads` | `/api/threads/{id}` | DELETE | 删除本地线程数据 |
| `artifacts` | `/api/threads/{id}/artifacts/{path}` | GET | 获取 Artifact |
| `suggestions` | `/api/threads/{id}/suggestions` | POST | 生成后续建议 |
| `channels` | `/api/channels/status` | GET | 频道状态 |
| `runs` | `/api/runs/{thread_id}` | POST | 创建运行（Gateway 模式） |

---

## 4. Models 路由

```python
# app/gateway/routers/models.py

@router.get("/")
async def list_models():
    """列出所有配置的模型"""
    config = get_app_config()
    return {
        "models": [
            {
                "name": m.name,
                "display_name": m.display_name,
                "supports_thinking": m.supports_thinking,
                "supports_vision": m.supports_vision,
            }
            for m in config.models
        ]
    }
```

---

## 5. MCP 路由

```python
# app/gateway/routers/mcp.py

@router.put("/config")
async def update_mcp_config(servers: dict):
    """更新 MCP 服务器配置"""
    # 1. 写入 extensions_config.json
    # 2. mtime 变化 → MCP 工具缓存自动失效
```

---

## 6. Skills 路由

```python
# app/gateway/routers/skills.py

@router.get("/")
async def list_skills():
    """列出所有技能（含启用状态）"""
    from deerflow.skills import load_skills
    skills = load_skills()
    # 合并 extensions_config.json 中的启用状态
    return {"skills": [...]}

@router.post("/install")
async def install_skill(file: UploadFile):
    """从 .skill 归档安装技能"""
    # 验证 ZIP 格式
    # 解压到 skills/custom/
    # 返回安装结果
```

---

## 7. Memory 路由

```python
# app/gateway/routers/memory.py

@router.get("/")
async def get_memory():
    """获取记忆数据"""
    return get_memory_data()

@router.post("/reload")
async def reload_memory():
    """强制重载记忆（清除缓存）"""
    return reload_memory_data()

@router.get("/status")
async def memory_status():
    """获取记忆状态（配置+数据）"""
    return {
        "config": get_memory_config(),
        "data": get_memory_data(),
    }
```

---

## 8. Uploads 路由

```python
# app/gateway/routers/uploads.py

@router.post("/threads/{thread_id}/uploads")
async def upload_files(thread_id: str, files: list[UploadFile]):
    """上传文件到线程"""
    # 1. 验证（非目录路径）
    # 2. 存储到 .deer-flow/threads/{thread_id}/user-data/uploads/
    # 3. PDF/PPT/Excel/Word → markitdown 转为 Markdown
    # 4. 返回文件元数据（含 virtual_path）
```

### 8.1 支持格式

| 格式 | 处理方式 |
|------|---------|
| PDF | markitdown 转为 Markdown |
| PPT | markitdown 转为 Markdown |
| Excel | 转为 Markdown 表格 |
| Word | markitdown 转为 Markdown |
| 图片 | 保留原格式（供 view_image 使用） |
| 其他 | 保留原格式 |

---

## 9. Threads 路由

```python
# app/gateway/routers/threads.py

@router.delete("/threads/{thread_id}")
async def delete_thread(thread_id: str):
    """删除 DeerFlow 管理的本地线程数据"""
    # LangGraph 线程删除在前端通过 LangGraph SDK 完成
    # Gateway 仅负责清理 .deer-flow/threads/{thread_id}/
```

---

## 10. Artifacts 路由

```python
# app/gateway/routers/artifacts.py

@router.get("/threads/{thread_id}/artifacts/{path:path}")
async def get_artifact(thread_id: str, path: str, download: bool = False):
    """获取 Agent 生成的 Artifact 文件"""
    # 安全：验证 path 在允许范围内
    # XSS 防护：HTML/SVG 等内容类型强制下载
    return FileResponse(...)
```

---

## 11. Suggestions 路由

```python
# app/gateway/routers/suggestions.py

@router.post("/threads/{thread_id}/suggestions")
async def generate_suggestions(thread_id: str):
    """基于对话历史生成后续建议问题"""
    # 读取 thread 历史
    # 调用 LLM 生成 3-5 个建议问题
    # 支持富文本归一化（block/list 格式 → 纯文本）
```

---

## 12. 源码索引

| 文件 | 说明 |
|------|------|
| `app/gateway/app.py` | FastAPI 应用入口 + 生命周期 |
| `app/gateway/routers/models.py` | 模型列表 API |
| `app/gateway/routers/mcp.py` | MCP 配置 API |
| `app/gateway/routers/skills.py` | 技能管理 API |
| `app/gateway/routers/memory.py` | 记忆 API |
| `app/gateway/routers/uploads.py` | 文件上传 API |
| `app/gateway/routers/threads.py` | 线程清理 API |
| `app/gateway/routers/artifacts.py` | Artifact 服务 API |
| `app/gateway/routers/suggestions.py` | 后续建议 API |
| `app/gateway/routers/channels.py` | 频道状态 API |

---

*文档版本：v2.0 | 生成日期：2026/04/08*
