# Hermes Agent 项目技术分析报告

## 一、项目概述

**项目名称**：Hermes Agent  
**GitHub**：https://github.com/NousResearch/hermes-agent  
**开发团队**：Nous Research  
**License**：MIT  
**当前版本**：v0.8.0  
**语言**：Python  

**核心定位**：
> "The self-improving AI agent — 自改进的 AI Agent"

**核心特性**：
- 🧠 **自学习循环**：从经验中创建技能，使用过程中自我改进
- 💾 **持久化记忆**：跨会话记忆、用户建模、对话搜索
- 🛠️ **40+ 工具**：丰富的工具集，支持 MCP 协议
- 📱 **多平台**：Telegram、Discord、Slack、WhatsApp、Signal、CLI
- ☁️ **多后端**：本地、Docker、SSH、Daytona、Singularity、Modal
- ⏰ **定时任务**：内置 cron 调度器

---

## 二、核心架构

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                   Hermes Agent                          │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  接入层                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  CLI (TUI)   │  │  Gateway     │  │  API         │  │
│  │  终端界面     │  │  多平台网关  │  │  REST API    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                          ↓                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Agent Core                         │   │
│  │  - Agent Loop (agent/agent.py)                 │   │
│  │  - State Management (hermes_state.py)          │   │
│  │  - Tool Execution (tools/)                     │   │
│  │  - Skill System (skills/)                      │   │
│  └─────────────────────────────────────────────────┘   │
│                          ↓                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Memory & Learning                  │   │
│  │  - Persistent Memory (memory/)                 │   │
│  │  - Skill Creation (acp_skills/)                │   │
│  │  - User Modeling (honcho integration)          │   │
│  │  - Session Search (FTS5 + LLM summary)         │   │
│  └─────────────────────────────────────────────────┘   │
│                          ↓                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Execution Backends                 │   │
│  │  - Local / Docker / SSH / Daytona / Modal      │   │
│  │  - Tool Execution (model_tools.py)             │   │
│  │  - Subagent Delegation                         │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 2.2 核心组件

| 组件 | 位置 | 职责 |
|------|------|------|
| **Agent Core** | `agent/` | Agent 循环、决策、工具调用 |
| **State Management** | `hermes_state.py` | 状态管理、会话持久化 |
| **Tools** | `tools/` | 40+ 工具实现 |
| **Skills** | `skills/` | 技能系统、自学习 |
| **Memory** | `memory/` | 持久化记忆、用户建模 |
| **Gateway** | `gateway/` | 多平台消息网关 |
| **CLI** | `hermes_cli/` | 终端界面 |
| **Cron** | `cron/` | 定时任务调度 |

---

## 三、核心特性详解

### 3.1 自学习循环

**学习流程**：
```
1. 执行复杂任务
   ↓
2. 自动创建技能（Skill Creation）
   ↓
3. 技能存入 Skills Hub
   ↓
4. 下次遇到类似任务自动调用
   ↓
5. 使用过程中持续改进技能
```

**技能创建**：
```python
# 复杂任务完成后
agent.create_skill(
    name="deploy_to_vps",
    description="Deploy application to VPS via SSH",
    trajectory=conversation_history,
    tags=["deployment", "ssh", "vps"]
)
```

### 3.2 持久化记忆

**记忆类型**：
| 类型 | 描述 | 存储位置 |
|------|------|---------|
| **会话记忆** | 当前对话历史 | SQLite |
| **跨会话记忆** | 所有历史对话 | SQLite + FTS5 全文搜索 |
| **用户画像** | 用户偏好、习惯 | Honcho 集成 |
| **技能记忆** | 已创建的技能 | Skills Hub |

**记忆搜索**：
```python
# FTS5 全文搜索 + LLM 总结
results = agent.search_memory(
    query="deployment instructions",
    limit=10,
    summarize=True  # LLM 总结搜索结果
)
```

### 3.3 多平台支持

**支持平台**：
| 平台 | 状态 | 功能 |
|------|------|------|
| **CLI (TUI)** | ✅ | 完整功能 |
| **Telegram** | ✅ | 完整功能 |
| **Discord** | ✅ | 完整功能 |
| **Slack** | ✅ | 完整功能 |
| **WhatsApp** | ✅ | 完整功能 |
| **Signal** | ✅ | 完整功能 |
| **Email** | ✅ | 基础功能 |

**网关架构**：
```
┌─────────────────────────────────────────┐
│           Hermes Gateway                │
├─────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐    │
│  │  Telegram    │  │   Discord    │    │
│  │  Adapter     │  │   Adapter    │    │
│  └──────────────┘  └──────────────┘    │
│  ┌──────────────┐  ┌──────────────┐    │
│  │   Slack      │  │  WhatsApp    │    │
│  │  Adapter     │  │   Adapter    │    │
│  └──────────────┘  └──────────────┘    │
│           ↓                              │
│  ┌──────────────────────────────────┐   │
│  │       Unified Message Queue      │   │
│  └──────────────────────────────────┘   │
│           ↓                              │
│  ┌──────────────────────────────────┐   │
│  │         Agent Core               │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

### 3.4 多后端执行

**支持后端**：
| 后端 | 描述 | 适用场景 |
|------|------|---------|
| **Local** | 本地执行 | 开发、测试 |
| **Docker** | Docker 容器 | 隔离环境 |
| **SSH** | 远程 SSH | 远程服务器 |
| **Daytona** | 无服务器 | 按需执行、低成本 |
| **Modal** | 无服务器 GPU | AI 任务、GPU 加速 |
| **Singularity** | HPC 容器 | 高性能计算 |

**后端选择**：
```bash
# 设置执行后端
hermes backend set local      # 本地
hermes backend set docker     # Docker
hermes backend set ssh        # SSH
hermes backend set daytona    # 无服务器
hermes backend set modal      # 无服务器 GPU
```

### 3.5 工具系统

**内置工具**（40+）：
| 类别 | 工具数 | 示例 |
|------|--------|------|
| **文件系统** | 8+ | read_file, write_file, search_files |
| **终端命令** | 6+ | run_command, run_script |
| **网络** | 5+ | http_request, dns_lookup |
| **数据库** | 4+ | sql_query, redis_command |
| **Git** | 4+ | git_commit, git_push |
| **AI** | 6+ | generate_image, transcribe_audio |
| **其他** | 7+ | send_email, schedule_task |

**MCP 集成**：
```bash
# 连接 MCP Server
hermes mcp connect filesystem
hermes mcp connect github
hermes mcp connect postgres
```

### 3.6 定时任务

**Cron 调度**：
```bash
# 创建定时任务
hermes cron add "0 9 * * *" "Send daily report"
hermes cron add "0 2 * * 0" "Weekly backup"

# 查看任务
hermes cron list

# 删除任务
hermes cron remove <task_id>
```

**任务交付**：
- 支持所有平台（Telegram、Discord、Email 等）
- 自然语言描述任务
- 自动执行并发送结果

---

## 四、与 OpenClaw/HiClaw 对比

### 4.1 定位对比

| 维度 | OpenClaw | HiClaw | Hermes Agent |
|------|---------|--------|-------------|
| **定位** | 单 Agent CLI | 多 Agent 操作系统 | 自改进单 Agent |
| **架构** | 单体 | Manager-Workers | Agent + Skills |
| **协议** | 自定义 | Matrix IM | 多平台网关 |
| **学习** | 无 | 无 | ✅ 自学习循环 |
| **记忆** | 基础 | 基础 | ✅ 持久化 + 搜索 |
| **多平台** | 有限 | Matrix 客户端 | ✅ 7+ 平台 |

### 4.2 特性对比

| 特性 | OpenClaw | HiClaw | Hermes |
|------|---------|--------|--------|
| **CLI 界面** | ✅ | ❌ | ✅ (TUI) |
| **多平台** | ⚠️ | ✅ (Matrix) | ✅ (7+ 平台) |
| **技能系统** | ✅ | ✅ | ✅ (自学习) |
| **记忆系统** | 基础 | 基础 | ✅ (FTS5+LLM) |
| **定时任务** | ❌ | ❌ | ✅ (Cron) |
| **多后端** | ❌ | ❌ | ✅ (6 后端) |
| **子 Agent** | ❌ | ✅ | ✅ (Subagents) |
| **MCP 支持** | ✅ | ✅ | ✅ |
| **自学习** | ❌ | ❌ | ✅ |

### 4.3 学习曲线

| 项目 | 上手难度 | 配置复杂度 | 适用场景 |
|------|---------|-----------|---------|
| **OpenClaw** | 低 | 低 | 个人 CLI 使用 |
| **HiClaw** | 中 | 中 | 企业多 Agent 协作 |
| **Hermes** | 中 | 中 | 个人/团队多平台使用 |

---

## 五、技术亮点

### 5.1 自学习技能系统

**技能创建流程**：
```
1. 用户执行复杂任务
   ↓
2. Agent 记录完整轨迹（trajectory）
   ↓
3. 自动提取可复用模式
   ↓
4. 创建新技能（Skill）
   ↓
5. 存入 Skills Hub
   ↓
6. 下次自动调用
```

**技能改进**：
```python
# 技能使用过程中持续改进
class Skill:
    def execute(self, input):
        result = self._execute(input)
        
        # 记录执行结果
        self.log_execution(input, result)
        
        # 分析改进点
        improvements = self.analyze_improvements()
        
        # 更新技能
        if improvements:
            self.update(improvements)
```

### 5.2 记忆搜索系统

**FTS5 全文搜索**：
```sql
-- SQLite FTS5 全文搜索
SELECT * FROM conversations
WHERE conversations MATCH 'deployment instructions'
ORDER BY rank
LIMIT 10;
```

**LLM 总结**：
```python
# 搜索结果 LLM 总结
def summarize_search_results(results):
    prompt = f"""
    Summarize these search results about '{query}':
    
    {results}
    
    Provide a concise summary with key points.
    """
    return llm.generate(prompt)
```

### 5.3 用户画像系统

**Honcho 集成**：
```python
# 用户画像
class UserProfile:
    def __init__(self):
        self.preferences = {}  # 偏好
        self.habits = {}       # 习惯
        self.knowledge = {}    # 知识领域
        self.personality = {}  # 性格特征
    
    def update_from_conversation(self, conversation):
        # 从对话中提取用户信息
        insights = extract_user_insights(conversation)
        self.merge_insights(insights)
```

### 5.4 多后端执行

**后端抽象**：
```python
class ExecutionBackend(ABC):
    @abstractmethod
    def execute(self, command, env):
        pass
    
    @abstractmethod
    def upload_file(self, path, content):
        pass
    
    @abstractmethod
    def download_file(self, path):
        pass

# 实现
class LocalBackend(ExecutionBackend):
    def execute(self, command, env):
        return subprocess.run(command, shell=True, env=env)

class DockerBackend(ExecutionBackend):
    def execute(self, command, env):
        return docker_client.exec(command, env=env)

class SSHBackend(ExecutionBackend):
    def execute(self, command, env):
        return ssh_client.exec(command, env=env)
```

---

## 六、安装与使用

### 6.1 快速安装

```bash
# 一键安装
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# 启动
hermes
```

### 6.2 基础使用

```bash
# 开始对话
hermes

# 选择模型
hermes model

# 配置工具
hermes tools

# 启动网关（多平台）
hermes gateway start

# 查看技能
hermes skills

# 创建定时任务
hermes cron add "0 9 * * *" "Send daily report"
```

### 6.3 从 OpenClaw 迁移

```bash
# 自动迁移
hermes claw migrate

# 预览迁移内容
hermes claw migrate --dry-run

# 迁移特定内容
hermes claw migrate --preset user-data  # 不含密钥
hermes claw migrate --overwrite         # 覆盖冲突
```

**迁移内容**：
- SOUL.md（人格文件）
- Memories（MEMORY.md、USER.md）
- Skills（用户创建的技能）
- API Keys（允许的密钥）
- 消息平台设置
- TTS 资源

---

## 七、总结

### 7.1 核心价值

**Hermes Agent** = **自改进 + 多平台 + 持久化记忆**

| 价值 | 描述 |
|------|------|
| **自学习** | 从经验中创建技能，持续改进 |
| **多平台** | 7+ 平台支持，统一体验 |
| **持久化** | 跨会话记忆、用户画像 |
| **灵活性** | 6 种执行后端，按需选择 |
| **自动化** | 内置 Cron 调度器 |

### 7.2 适用场景

**适合**：
- 个人 AI 助手（多平台访问）
- 需要持久化记忆的场景
- 需要自学习能力的场景
- 多后端执行需求
- 定时任务自动化

**不适合**：
- 企业级多 Agent 协作（HiClaw 更适合）
- 需要 Matrix 协议的场景
- 简单 CLI 使用（OpenClaw 更轻量）

### 7.3 与 Nacos AI Registry 关系

**无直接关系**：
- Hermes 是独立的 Agent 框架
- 不使用 Nacos 管理资源
- 有自己的技能系统（Skills Hub）

**潜在集成点**：
- MCP Server 可以通过 Nacos 注册
- Skills 可以发布到 Nacos Skill Registry
- 使用 Nacos 管理 MCP 配置

---

*报告生成时间：2026-04-09*  
*版本：v1.0*  
*数据来源：https://github.com/NousResearch/hermes-agent*