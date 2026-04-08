# HiClaw 项目技术分析报告（基于实际源码）

## 一、项目概述

**项目名称**：HiClaw  
**GitHub**：https://github.com/higress-group/hiclaw  
**开发团队**：Higress Group（阿里云团队）  
**License**：Apache 2.0  
**当前版本**：v1.0.6 (2026-03-14)  
**语言**：Go + Python

**核心定位**：
> "HiClaw 是一个开源的协作式多智能体操作系统。让多个 Agent 在 Matrix 房间中协作，人类全程可见、随时可介入。"

**关键特性**：
- 🧑‍💻 **Manager-Workers 架构**：Agent 管理 Agents
- 🦞 **自定义 Agent**：支持 OpenClaw、Copaw、NanoClaw、ZeroClaw 等
- 📚 **MinIO 共享文件系统**：降低多 Agent 协作 Token 消耗
- ⛑️ **Higress AI Gateway**：凭证零暴露，流量统一管控
- 🎨 **Matrix IM 驱动**：透明可审计，支持分布式部署

---

## 二、技术架构

### 2.1 整体架构

```
┌─────────────────────────────────────────────┐
│         hiclaw-manager-agent                │
│  Higress │ Tuwunel │ MinIO │ Element Web    │
│  Manager Agent (OpenClaw)                   │
└──────────────────┬──────────────────────────┘
                   │ Matrix + HTTP Files
┌──────────────────┴──────┐  ┌────────────────┐
│  hiclaw-worker-agent    │  │  hiclaw-worker │
│  Worker Alice (OpenClaw)│  │  Worker Bob    │
└─────────────────────────┘  └────────────────┘
```

### 2.2 核心组件

| 组件 | 语言 | 职责 |
|------|------|------|
| **hiclaw-controller** | Go | 控制器，管理 Worker 生命周期 |
| **copaw** | Python | Manager/Worker Agent 实现 |
| **hiclaw-worker** | Python | Worker Agent 运行时 |
| **docker-proxy** | Go | Docker 安全代理 |
| **manager** | Shell/Python | 安装和管理脚本 |

### 2.3 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| **协议层** | Matrix IM | 实时通信协议 |
| **网关层** | Higress AI Gateway | 流量管理、凭证管理 |
| **存储层** | MinIO | 文件共享 |
| **IM 客户端** | Element Web | Web 客户端 |
| **IM 服务器** | Tuwunel | Matrix 服务器 |
| **Agent 运行时** | OpenClaw/Copaw | Agent 执行引擎 |

---

## 三、核心功能

### 3.1 Manager-Workers 架构

**Manager Agent 职责**：
- 创建 Worker（对话式创建）
- 分配任务给 Worker
- 心跳检查 Worker 状态
- 协调多个 Worker 协作

**Worker Agent 职责**：
- 执行具体任务
- 通过 Higress AI Gateway 访问 API
- 只持有消费者令牌，不接触真实 API Key

**使用示例**：
```
你：帮我创建一个名为 alice 的前端 Worker

Manager：好的，Worker alice 已创建。
         房间：Worker: Alice
         可以直接在房间里给 alice 分配任务了。

你：@alice 帮我用 React 实现一个登录页面

Alice：收到，正在处理……[几分钟后]
       完成了！PR 已提交：https://github.com/xxx/pull/1
```

### 3.2 安全模型

```
Worker（只持有消费者令牌）
    → Higress AI 网关（持有真实 API Key、GitHub PAT）
        → LLM API / GitHub API / MCP Server
```

**安全特性**：
- Worker 永远不持有真实 API Key
- 即使 Worker 被攻击，攻击者也拿不到任何真实凭证
- Higress AI 网关统一管理所有真实凭证
- Manager 知道 Worker 在做什么，但同样接触不到真实的 Key

### 3.3 人工全程监督

**每个 Matrix 房间包含**：
- 人类用户
- Manager Agent
- 相关 Worker Agents

**人类可以**：
- 随时观察 Agent 对话
- 实时干预或修正 Agent 行为
- 直接给 Worker 分配任务
- 没有黑盒，没有隐藏的 Agent 间调用

---

## 四、与 Nacos AI Registry 的关系

### 4.1 实际集成点

从源码分析（`hiclaw-controller/internal/executor/nacos_agentspec.go`）：

```go
// HiClaw 使用 Nacos 管理 AgentSpec
package executor

import (
    "github.com/nacos-group/nacos-sdk-go/v2/client"
)

// 从 Nacos 加载 AgentSpec
func LoadAgentSpec(nacosClient client.INacosClient, agentSpecRef string) (*AgentSpec, error) {
    // 从 Nacos 配置中心获取 AgentSpec 配置
    config, err := nacosClient.GetConfig(agentSpecRef)
    if err != nil {
        return nil, err
    }
    
    // 解析 AgentSpec
    spec := &AgentSpec{}
    json.Unmarshal([]byte(config), spec)
    
    return spec, nil
}
```

### 4.2 关系定位

| 维度 | HiClaw | Nacos AI Registry |
|------|--------|------------------|
| **定位** | 多 Agent 操作系统 | AI 资源治理平台 |
| **功能** | Agent 执行、协作、监督 | 资源注册、版本管理、审核 |
| **关系** | **使用 Nacos 管理 AgentSpec** | **被 HiClaw 使用** |
| **依赖** | 依赖 Nacos 存储 AgentSpec | 不依赖 HiClaw |

**结论**：
- HiClaw **不是** Nacos AI Registry 的运行时
- HiClaw **使用** Nacos 来管理 AgentSpec 配置
- 两者是**互补关系**，不是包含关系

---

## 五、核心代码分析

### 5.1 Controller（Go）

**位置**：`hiclaw-controller/cmd/controller/main.go`

**职责**：
- 监听 Worker 状态
- 管理 Worker 生命周期
- 与 Higress AI Gateway 集成

**核心逻辑**：
```go
func main() {
    // 初始化 Nacos 客户端
    nacosClient := initNacosClient()
    
    // 初始化 HTTP 服务器
    httpServer := internal.NewServer()
    
    // 注册 Worker 控制器
    workerController := controller.NewWorkerController(nacosClient)
    httpServer.Register("/api/workers", workerController)
    
    // 启动文件监听器
    watcher := watcher.NewFileWatcher()
    watcher.Watch("/var/hiclaw/workers", workerController.OnWorkerChange)
    
    // 启动服务
    httpServer.Start()
}
```

### 5.2 Copaw Manager（Python）

**位置**：`copaw/src/copaw_manager/`

**职责**：
- Manager Agent 实现
- 创建 Worker
- 分配任务
- 心跳检查

**核心逻辑**：
```python
class CopawManager:
    def __init__(self, config: Config):
        self.config = config
        self.matrix_channel = MatrixChannel(config.matrix)
        self.nacos_client = NacosClient(config.nacos)
    
    async def create_worker(self, name: str, role: str):
        # 从 Nacos 加载 Worker 配置
        worker_spec = await self.nacos_client.get_agent_spec(f"worker-{role}")
        
        # 创建 Worker
        worker = WorkerAgent(
            name=name,
            spec=worker_spec,
            matrix_channel=self.matrix_channel
        )
        
        # 启动 Worker
        await worker.start()
        
        # 创建 Matrix 房间
        room = await self.matrix_channel.create_room(f"Worker: {name}")
        await room.invite(worker)
        
        return worker
    
    async def assign_task(self, worker: WorkerAgent, task: str):
        # 分配任务给 Worker
        await worker.execute(task)
```

### 5.3 Worker Agent（Python）

**位置**：`copaw/src/copaw_worker/worker.py`

**职责**：
- 执行具体任务
- 通过 Higress AI Gateway 访问 API
- 上报状态

**核心逻辑**：
```python
class WorkerAgent:
    def __init__(self, name: str, spec: AgentSpec, matrix_channel: MatrixChannel):
        self.name = name
        self.spec = spec
        self.matrix_channel = matrix_channel
        self.higress_gateway = HigressGateway(spec.gateway_url)
    
    async def execute(self, task: str):
        # 通过 Higress Gateway 调用 LLM
        response = await self.higress_gateway.call_llm(task)
        
        # 执行工具调用
        if response.tools:
            for tool in response.tools:
                result = await self.higress_gateway.call_tool(tool)
                response.context.append(result)
        
        # 上报结果
        await self.matrix_channel.send_message(response.text)
```

### 5.4 Higress AI Gateway

**位置**：`docker-proxy/security.go`

**职责**：
- 统一管理 API Key
- Worker 只持有消费者令牌
- 真实凭证对 Worker 不可见

**核心逻辑**：
```go
func (g *Gateway) HandleRequest(w http.ResponseWriter, r *http.Request) {
    // 验证消费者令牌
    consumerToken := r.Header.Get("X-Consumer-Token")
    if !g.validateConsumerToken(consumerToken) {
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return
    }
    
    // 获取真实 API Key
    apiKey := g.getRealAPIKey(consumerToken)
    
    // 转发请求到 LLM API
    req := r.Clone(r.Context())
    req.Header.Set("Authorization", "Bearer "+apiKey)
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // 返回响应
    io.Copy(w, resp.Body)
}
```

---

## 六、部署与使用

### 6.1 快速开始

**前置条件**：
- Docker Desktop（Windows/macOS）或 Docker Engine（Linux）
- 最低 2C4GB 内存，推荐 4C8GB

**安装命令**：
```bash
# macOS / Linux
bash <(curl -sSL https://higress.ai/hiclaw/install.sh)

# Windows (PowerShell 7+)
Set-ExecutionPolicy Bypass -Scope Process -Force
$wc=New-Object Net.WebClient
$wc.Encoding=[Text.Encoding]::UTF8
iex $wc.DownloadString('https://higress.ai/hiclaw/install.ps1')
```

**访问方式**：
- Web：http://127.0.0.1:18088
- 移动端：Element / FluffyChat（Matrix 客户端）

### 6.2 配置 Nacos

**配置 AgentSpec**：
```yaml
# Nacos Config
dataId: worker-frontend
group: DEFAULT_GROUP
content: |
  name: frontend-worker
  role: frontend-development
  tools:
    - github-copilot
    - code-review
  gateway:
    url: http://hiclaw-gateway:8080
```

**HiClaw 从 Nacos 加载**：
```python
# HiClaw Manager
worker_spec = await nacos_client.get_agent_spec("worker-frontend")
worker = WorkerAgent(name="alice", spec=worker_spec)
```

---

## 七、与 OpenClaw 对比

| 维度 | OpenClaw 原生 | HiClaw |
|------|--------------|--------|
| **部署方式** | 单进程 | 分布式容器 |
| **Agent 创建** | 手动配置 + 重启 | 对话式 |
| **凭证管理** | 每个 Agent 持有真实 Key | Worker 只持有消费者令牌 |
| **人工可见性** | 可选 | 内置（Matrix 房间） |
| **移动端访问** | 取决于渠道配置 | 任意 Matrix 客户端，零配置 |
| **监控** | 无 | Manager 心跳，房间内可见 |
| **Nacos 集成** | 无 | 使用 Nacos 管理 AgentSpec |

---

## 八、总结

### 8.1 核心价值

**HiClaw** = **多 Agent 操作系统**

| 价值 | 描述 |
|------|------|
| **Manager-Workers 架构** | Agent 管理 Agents，无需人工监督每个 Worker |
| **企业级安全** | Worker 不持有真实 API Key，凭证零暴露 |
| **人工全程监督** | Matrix 房间透明可见，随时可介入 |
| **Nacos 集成** | 使用 Nacos 管理 AgentSpec 配置 |
| **开箱即用** | 一条命令部署所有组件 |

### 8.2 与 Nacos AI Registry 关系

**互补关系**：
- HiClaw **使用** Nacos 管理 AgentSpec
- Nacos AI Registry **提供** AgentSpec 存储和管理
- 两者不是包含关系，是独立产品

**集成方式**：
```
HiClaw Manager
    ↓ (从 Nacos 加载)
Nacos Config (AgentSpec)
    ↓
HiClaw Worker (执行)
    ↓ (通过 Higress Gateway)
LLM API / MCP Server
```

### 8.3 适用场景

**适合**：
- 需要多 Agent 协作的复杂任务
- 需要人工监督和介入的场景
- 企业级安全要求（凭证零暴露）
- 需要透明可审计的 Agent 通信

**不适合**：
- 简单的单 Agent 任务
- 不需要人工监督的自动化场景
- 对性能要求极高的场景（容器化有开销）

---

*报告生成时间：2026-04-07*  
*版本：v1.0*  
*数据来源：https://github.com/higress-group/hiclaw（实际源码分析）*