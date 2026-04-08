# 沙箱执行系统（Sandbox）

---

## 1. 概述

DeerFlow 的沙箱系统为每个对话线程提供隔离的执行环境。Agent 在沙箱内操作文件系统、执行命令，与宿主机完全隔离。

沙箱支持三种模式：

| 模式 | Provider | 隔离级别 | 适用场景 |
|------|----------|---------|---------|
| **Local** | `LocalSandboxProvider` | 无隔离（直接文件系统） | 本地开发 |
| **Docker** | `AioSandboxProvider` | 容器级隔离 | Docker 部署 |
| **Kubernetes** | `AioSandboxProvider` + Provisioner | Pod 级隔离 | 大规模生产 |

---

## 2. 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    SandboxProvider（工厂模式）                  │
│           acquire(thread_id) → get(sandbox_id) → release()     │
└─────────────────────┬───────────────────────────────────────┘
                      │
     ┌────────────────┼────────────────┐
     ▼                ▼                ▼
┌──────────┐  ┌─────────────────┐  ┌──────────────┐
│  Local   │  │   AioSandbox   │  │  Provisioner │
│ Sandbox  │  │   (Docker)     │  │ (K8s Pod)   │
└──────────┘  └─────────────────┘  └──────────────┘
```

---

## 3. 核心接口

```python
# packages/harness/deerflow/sandbox/sandbox.py

class Sandbox(ABC):
    @abstractmethod
    def execute_command(self, command: str) -> str:
        """在沙箱中执行 Shell 命令"""

    @abstractmethod
    def read_file(self, path: str) -> str:
        """读取文件内容"""

    @abstractmethod
    def write_file(self, path: str, content: str, append: bool = False) -> None:
        """写入文件"""

    @abstractmethod
    def list_dir(self, path: str) -> list[str]:
        """列出目录内容"""

    @abstractmethod
    def glob(self, root: str, pattern: str, include_dirs: bool = False, max_results: int = 200) -> tuple[list[str], bool]:
        """按模式匹配文件"""

    @abstractmethod
    def grep(self, root: str, pattern: str, glob: str | None = None, literal: bool = False, case_sensitive: bool = False, max_results: int = 100) -> tuple[list[GrepMatch], bool]:
        """在文件中搜索"""
```

---

## 4. SandboxProvider 模式

```python
# packages/harness/deerflow/sandbox/sandbox_provider.py

class SandboxProvider(ABC):
    @abstractmethod
    def acquire(self, thread_id: str) -> str:
        """获取沙箱实例，返回 sandbox_id"""

    @abstractmethod
    def get(self, sandbox_id: str) -> Sandbox | None:
        """通过 sandbox_id 获取沙箱实例"""

    def release(self, sandbox_id: str) -> None:
        """释放沙箱实例"""
```

### 4.1 LocalSandboxProvider

本地文件系统沙箱，单例模式：

```python
class LocalSandboxProvider(SandboxProvider):
    _instance: LocalSandboxProvider | None = None

    def acquire(self, thread_id: str) -> str:
        # sandbox_id = "local"
        # 创建线程目录
        Paths.create_thread_dirs(thread_id)
        return "local"

    def get(self, sandbox_id: str) -> LocalSandbox:
        return LocalSandbox()
```

**特点**：
- 单例，`sandbox_id` 固定为 `"local"`
- 命令直接在宿主机执行
- 文件操作通过虚拟路径映射

### 4.2 AioSandboxProvider

Docker 容器沙箱：

```python
# packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py

class AioSandboxProvider(SandboxProvider):
    def acquire(self, thread_id: str) -> str:
        # 从容器池获取可用容器
        # 或通过 Provisioner 在 K8s 中启动 Pod
        container = self._pool.acquire()
        return container.id
```

**特点**：
- 容器池管理（复用已有容器）
- 通过 Provisioner 支持 Kubernetes
- `/mnt/user-data` 已挂载到容器内

---

## 5. 虚拟路径系统

### 5.1 路径映射表

| 虚拟路径（Agent 可见） | 物理路径 |
|------------------------|---------|
| `/mnt/user-data/workspace` | `{base_dir}/threads/{thread_id}/user-data/workspace` |
| `/mnt/user-data/uploads` | `{base_dir}/threads/{thread_id}/user-data/uploads` |
| `/mnt/user-data/outputs` | `{base_dir}/threads/{thread_id}/user-data/outputs` |
| `/mnt/skills` | `deer-flow/skills/`（只读） |
| `/mnt/acp-workspace` | `{base_dir}/threads/{thread_id}/acp-workspace/`（只读） |

### 5.2 路径替换函数

```python
# packages/harness/deerflow/sandbox/tools.py

# Agent → 物理路径（读/写时）
replace_virtual_path(path, thread_data)

# 物理路径 → Agent（输出时掩码）
mask_local_paths_in_output(output, thread_data)

# 命令字符串中的路径批量替换
replace_virtual_paths_in_command(command, thread_data)
```

### 5.3 自定义挂载

在 `config.yaml` 中配置自定义卷：

```yaml
sandbox:
  use: deerflow.sandbox.local:LocalSandboxProvider
  mounts:
    - host_path: /Users/mac/data
      container_path: /mnt/shared
      read_only: true
```

---

## 6. 沙箱工具

沙箱工具在 `packages/harness/deerflow/sandbox/tools.py` 中定义：

| 工具 | 函数 | 说明 |
|------|------|------|
| `bash` | `bash_tool` | 执行 Shell 命令 |
| `ls` | `ls_tool` | 树形列出目录（最多 2 层） |
| `read_file` | `read_file_tool` | 读取文件（支持行范围） |
| `write_file` | `write_file_tool` | 写入/追加文件 |
| `str_replace` | `str_replace_tool` | 字符串替换 |
| `glob` | `glob_tool` | 模式匹配文件 |
| `grep` | `grep_tool` | 文本/正则搜索 |

### 6.1 输出截断

| 工具 | 策略 | 默认上限 |
|------|------|---------|
| `bash` | 中间截断（保留首尾 50/50） | 20,000 字符 |
| `read_file` | 头部截断 | 50,000 字符 |
| `ls` | 头部截断 | 20,000 字符 |
| `glob` | 结果数限制 | 200 条 |
| `grep` | 结果数限制 | 100 条 |

---

## 7. 安全机制

### 7.1 路径验证

```python
validate_local_tool_path(path, thread_data, read_only=False)
# 允许：
#   /mnt/user-data/* — 读写
#   /mnt/skills/* — 只读
#   /mnt/acp-workspace/* — 只读
#   自定义挂载 — 按配置
# 阻止：路径遍历（..）
```

### 7.2 Bash 命令验证

```python
validate_local_bash_command_paths(command, thread_data)
# 仅允许：
#   /mnt/user-data/* 等虚拟路径
#   白名单路径（/bin/, /usr/bin/, /dev/ 等）
#   MCP filesystem server 允许的路径
# 阻止：
#   file:// URL
#   非白名单绝对路径
```

### 7.3 Host Bash 控制

```python
# config.yaml
sandbox:
  allow_host_bash: false  # 默认禁用
```

当 `allow_host_bash: false` 且使用 `LocalSandboxProvider` 时：
- `bash` 工具返回错误信息
- `task` 工具的 `subagent_type: "bash"` 被拒绝

Docker 模式（AioSandboxProvider）始终允许 bash。

---

## 8. 沙箱生命周期

```
SandboxMiddleware（before_agent）
    │
    ▼
ensure_sandbox_initialized(runtime)
    │
    ├─ sandbox_state 已有 sandbox_id？
    │   └─ 是 → get(sandbox_id) → 返回现有沙箱
    │   └─ 否 → acquire(thread_id) → 创建/获取新沙箱
    │
    ▼
工具调用（bash/ls/read/write/glob/grep）
    │
    ▼
ensure_thread_directories_exist(runtime)
    │
    ▼
沙箱执行 + 路径替换 + 输出掩码
```

---

## 9. 配置

### 9.1 config.yaml

```yaml
sandbox:
  use: deerflow.sandbox.local:LocalSandboxProvider
  # 或 Docker：
  # use: deerflow.community.aio_sandbox:AioSandboxProvider
  # 或 Kubernetes：
  # use: deerflow.community.aio_sandbox:AioSandboxProvider
  # provisioner_url: http://localhost:8002
  allow_host_bash: false
  mounts: []

  # 输出截断配置
  bash_output_max_chars: 20000
  read_file_output_max_chars: 50000
  ls_output_max_chars: 20000
```

---

## 10. 源码索引

| 文件 | 说明 |
|------|------|
| `sandbox/sandbox.py` | 抽象 Sandbox 接口 |
| `sandbox/sandbox_provider.py` | SandboxProvider 工厂 |
| `sandbox/local/__init__.py` | LocalSandboxProvider 实现 |
| `sandbox/tools.py` | 沙箱工具（bash/ls/read/write 等） |
| `sandbox/middleware.py` | SandboxMiddleware 生命周期 |
| `sandbox/security.py` | 安全配置与检查 |
| `sandbox/file_operation_lock.py` | 文件操作锁（并发安全） |
| `sandbox/search.py` | glob / grep 结果类型 |
| `community/aio_sandbox/aio_sandbox_provider.py` | AioSandboxProvider |
| `community/aio_sandbox/backend.py` | Docker/Provisioner 后端抽象 |

---

*文档版本：v2.0 | 生成日期：2026/04/08*
