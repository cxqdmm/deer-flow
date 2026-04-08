# 技能系统（Skills）

---

## 1. 概述

Skills（技能）是 DeerFlow 的核心扩展机制。每个 Skill 是一个结构化的 Markdown 文件，定义了特定领域的**工作流、最佳实践和参考资源**，Agent 在处理相关任务时加载对应的 Skill 以获得领域指导。

```
skills/
├── public/               # 内置技能（随仓库发布）
│   ├── deep-research/    # 深度研究
│   ├── data-analysis/   # 数据分析
│   ├── ppt-generation/   # PPT 生成
│   ├── image-generation/ # 图像生成
│   └── ...
└── custom/              # 自定义技能（gitignored）
```

---

## 2. SKILL.md 格式

每个 Skill 是一个 `SKILL.md` 文件，支持 YAML frontmatter：

```markdown
---
name: Deep Research
description: Conduct comprehensive research on any topic
license: MIT
allowed-tools:
  - web_search
  - web_fetch
  - bash
  - read_file
  - write_file
---

# Deep Research Skill

## Overview
This skill provides a systematic approach to conducting deep research...

## Workflow
1. Search for relevant sources
2. Fetch and analyze content
3. Synthesize findings
4. Generate report

## Best Practices
- Always verify information from multiple sources
- ...
```

### 2.1 Frontmatter 字段

| 字段 | 必填 | 说明 |
|------|------|------|
| `name` | 是 | 技能名称 |
| `description` | 是 | 一句话描述 |
| `license` | 否 | 许可证 |
| `allowed-tools` | 否 | Agent 使用此 Skill 时允许的工具列表 |
| `version` | 否 | 版本号（外部 Skill 支持） |
| `author` | 否 | 作者（外部 Skill 支持） |
| `compatibility` | 否 | 兼容性说明（外部 Skill 支持） |

### 2.2 支持文件

Skill 目录可以包含子目录和参考资源：

```
skills/public/deep-research/
├── SKILL.md
├── references/
│   ├── search-strategies.md
│   └── citation-format.md
└── templates/
    └── report-template.md
```

---

## 3. 技能加载

### 3.1 加载流程

```python
# packages/harness/deerflow/skills/loader.py

def load_skills(skills_path=None, use_config=True, enabled_only=False) -> list[Skill]:
    """递归扫描 skills/public 和 skills/custom"""
    skills = []
    for category in ["public", "custom"]:
        for root, dirs, files in os.walk(category_path):
            if "SKILL.md" not in files:
                continue
            skill = parse_skill_file(skill_file, category, relative_path)
            skills.append(skill)
    return sorted(skills, key=lambda s: s.name)
```

### 3.2 解析 SKILL.md

```python
# packages/harness/deerflow/skills/parser.py

def parse_skill_file(path, category, relative_path) -> Skill:
    # 1. 解析 YAML frontmatter
    # 2. 读取 Markdown 正文
    # 3. 构建 Skill 对象
```

### 3.3 启用状态

技能的启用状态存储在 `extensions_config.json`：

```json
{
  "skills": {
    "deep-research": { "enabled": true },
    "data-analysis": { "enabled": false }
  }
}
```

---

## 4. 技能注入系统提示词

```python
# packages/harness/deerflow/agents/lead_agent/prompt.py

apply_prompt_template(available_skills={"deep-research", "data-analysis"})
```

生成的提示词片段：

```xml
<skills>
## Available Skills

### deep-research
Description: Conduct comprehensive research on any topic
Allowed Tools: web_search, web_fetch, bash, read_file, write_file

Path: /mnt/skills/public/deep-research/SKILL.md

### data-analysis
Description: Analyze datasets and generate insights
Allowed Tools: bash, read_file, write_file, glob

Path: /mnt/skills/public/data-analysis/SKILL.md
</skills>
```

Agent 可以通过 `/mnt/skills` 路径访问技能目录中的参考文件和模板。

---

## 5. 技能安装

### 5.1 从 .skill 归档安装

`.skill` 归档是一个 ZIP 包，包含 Skill 的完整目录结构：

```
my-custom-skill.skill.zip
├── SKILL.md
├── references/
│   └── ...
└── templates/
    └── ...
```

通过 Gateway API 安装：

```bash
POST /api/skills/install
Content-Type: multipart/form-data
file: my-custom-skill.skill.zip
```

安装后自动解压到 `skills/custom/` 目录。

### 5.2 前端安装

```python
# app/gateway/routers/skills.py

@router.post("/install")
async def install_skill(file: UploadFile):
    # 验证 .skill 归档格式
    # 解压到 skills/custom/
    # 返回安装结果
```

---

## 6. 技能管理

### 6.1 skill_manage 工具（实验性）

当 `skill_evolution.enabled=true` 时，Agent 可以通过 `skill_manage` 工具直接管理自定义技能：

```python
# packages/harness/deerflow/tools/skill_manage_tool.py

@tool("skill_manage")
async def skill_manage_tool(
    action: str,    # create/edit/patch/delete/write_file/remove_file
    name: str,       # 技能名称（kebab-case）
    content: str | None = None,
    path: str | None = None,
    find: str | None = None,
    replace: str | None = None,
    expected_count: int | None = None,
) -> str:
```

### 6.2 操作类型

| 操作 | 说明 |
|------|------|
| `create` | 创建新 Skill |
| `edit` | 替换整个 SKILL.md |
| `patch` | 局部替换（find/replace） |
| `delete` | 删除 Skill |
| `write_file` | 写入支持文件 |
| `remove_file` | 删除支持文件 |

### 6.3 安全扫描

每次写入都会经过安全扫描：

```python
# packages/harness/deerflow/skills/security_scanner.py

async def scan_skill_content(content, executable, location):
    # 检测可执行内容（脚本路径、shell 命令等）
    # 返回：allow / block / warn
```

---

## 7. 技能目录结构

### 7.1 内置技能（public）

```
skills/public/
├── academic-paper-review/     # 学术论文评审
├── bootstrap/                  # 引导/模板生成
├── chart-visualization/        # 图表可视化（20+ 图表类型）
├── claude-to-deerflow/        # Claude Code 集成
├── code-documentation/         # 代码文档生成
├── consulting-analysis/         # 咨询分析
├── data-analysis/             # 数据分析
├── deep-research/              # 深度研究
├── find-skills/               # 技能发现
├── frontend-design/            # 前端设计
├── github-deep-research/       # GitHub 深度研究
├── image-generation/           # 图像生成
├── newsletter-generation/      # 新闻简报生成
├── podcast-generation/         # 播客生成
├── ppt-generation/            # PPT 生成
├── skill-creator/             # 技能创建助手
├── video-generation/          # 视频生成
└── web-design-guidelines/     # 网页设计指南
```

### 7.2 自定义技能（custom）

```
skills/custom/
└── your-custom-skill/         # 用户安装的技能
    ├── SKILL.md
    └── ...
```

`custom/` 目录被 `.gitignore` 忽略，不会随仓库提交。

---

## 8. 配置

### 8.1 config.yaml

```yaml
skills:
  path: ./skills                    # 技能目录路径（相对于项目根目录）
  container_path: /mnt/skills       # 容器内路径（只读）
```

### 8.2 extensions_config.json

```json
{
  "skills": {
    "skill-name": {
      "enabled": true
    }
  }
}
```

### 8.3 Skill Evolution（实验性）

```yaml
# config.yaml
skill_evolution:
  enabled: false  # 实验性：允许 Agent 通过 skill_manage 工具管理技能
```

---

## 9. 源码索引

| 文件 | 说明 |
|------|------|
| `skills/loader.py` | 技能递归扫描与加载 |
| `skills/parser.py` | SKILL.md 解析 |
| `skills/manager.py` | 技能安装、删除、验证 |
| `skills/types.py` | Skill 数据类型定义 |
| `skills/security_scanner.py` | 安全内容扫描 |
| `tools/skill_manage_tool.py` | skill_manage 工具实现 |

---

*文档版本：v2.0 | 生成日期：2026/04/08*
