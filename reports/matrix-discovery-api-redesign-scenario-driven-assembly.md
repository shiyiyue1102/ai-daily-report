# Matrix Discovery 接口改造方案：场景驱动的 AgentSpec 智能组装

## 一、现状分析

### 当前 Discovery 接口（推测）

```http
GET /api/discover?query={keyword}&type={resource_type}
```

**返回结构**：
```json
{
  "prompts": [...],
  "skills": [...],
  "mcps": [...],
  "agentcards": [...]
}
```

**问题**：
- 返回零散资源列表，用户需要手动筛选组合
- 无法理解复杂场景需求
- 缺乏组装逻辑和效果评估

---

## 二、新接口设计：场景驱动组装

### 2.1 核心接口

```http
POST /api/discover/assemble
```

**请求体**：
```json
{
  "scenario": "我要做一个能分析用户评论情感，并自动生成回复客服机器人",
  "context": {
    "workspace": "default",
    "existing_resources": ["customer-db-mcp"],
    "constraints": {
      "max_latency": "500ms",
      "budget": "low",
      "language": "zh-CN"
    },
    "preferences": {
      "style": "professional",
      "complexity": "simple"
    }
  },
  "options": {
    "include_alternatives": true,
    "detail_level": "full"
  }
}
```

**响应体**：
```json
{
  "assembly_id": "asm_20260306_001",
  "status": "success",
  "scenario_analysis": {
    "intent": "build_customer_service_agent",
    "subtasks": [
      {
        "task": "sentiment_analysis",
        "description": "分析用户评论情感倾向",
        "required_capabilities": ["text_analysis", "sentiment_detection"]
      },
      {
        "task": "response_generation",
        "description": "生成客服回复内容",
        "required_capabilities": ["text_generation", "context_aware"]
      },
      {
        "task": "database_query",
        "description": "查询用户历史订单信息",
        "required_capabilities": ["sql_generation", "data_retrieval"]
      }
    ]
  },
  "recommended_spec": {
    "name": "智能客服助手",
    "key": "smart-cs-agent",
    "description": "基于情感分析的自动客服回复Agent",
    "components": {
      "prompt": {
        "resource": {
          "type": "prompt",
          "key": "customer-service-expert-v2",
          "version": "2.3.1",
          "name": "客服专家角色模板"
        },
        "config": {
          "temperature": 0.7,
          "max_tokens": 500
        }
      },
      "model": {
        "provider": "openai",
        "model": "gpt-4-turbo",
        "fallback": "gpt-3.5-turbo"
      },
      "skills": [
        {
          "resource": {
            "type": "skill",
            "key": "sentiment-analyzer",
            "version": "1.5.0",
            "name": "情感分析器"
          },
          "usage": "分析用户评论情感倾向，输出positive/negative/neutral"
        },
        {
          "resource": {
            "type": "skill",
            "key": "response-generator",
            "version": "2.1.0",
            "name": "客服回复生成器"
          },
          "config": {
            "tone": "professional",
            "language": "zh-CN"
          }
        }
      ],
      "mcps": [
        {
          "resource": {
            "type": "mcp",
            "key": "customer-db-mcp",
            "version": "1.2.0",
            "name": "客户数据库MCP"
          },
          "tools": ["query_order_history", "get_user_profile"],
          "required": true
        },
        {
          "resource": {
            "type": "mcp",
            "key": "knowledge-base-mcp",
            "version": "3.0.1",
            "name": "知识库MCP"
          },
          "tools": ["search_faq", "get_solution"],
          "required": false
        }
      ],
      "agentcards": [
        {
          "resource": {
            "type": "agentcard",
            "key": "escalation-agent",
            "version": "1.0.0",
            "name": "人工升级Agent"
          },
          "trigger": "sentiment.score < -0.8",
          "description": "情感极度负面时自动转人工"
        }
      ]
    },
    "workflow": {
      "steps": [
        {
          "step": 1,
          "action": "analyze_sentiment",
          "input": "user_message",
          "output": "sentiment_result",
          "component": "skill:sentiment-analyzer"
        },
        {
          "step": 2,
          "action": "query_context",
          "input": "user_id",
          "output": "user_context",
          "component": "mcp:customer-db-mcp",
          "condition": "sentiment_result.requires_context"
        },
        {
          "step": 3,
          "action": "generate_response",
          "input": ["user_message", "sentiment_result", "user_context"],
          "output": "response",
          "component": "skill:response-generator"
        },
        {
          "step": 4,
          "action": "check_escalation",
          "input": "sentiment_result",
          "output": "escalation_decision",
          "component": "agentcard:escalation-agent",
          "condition": "sentiment_result.score < -0.8"
        }
      ]
    }
  },
  "performance_prediction": {
    "success_rate": 0.92,
    "avg_latency": "380ms",
    "estimated_cost": "$0.02/query",
    "confidence": "high",
    "bottlenecks": [
      {
        "component": "mcp:customer-db-mcp",
        "risk": "数据库查询可能成为瓶颈",
        "suggestion": "建议添加缓存层"
      }
    ]
  },
  "alternatives": [
    {
      "name": "低成本方案",
      "description": "使用轻量级模型降低成本",
      "changes": {
        "model": "gpt-3.5-turbo",
        "skills": ["light-sentiment-analyzer"]
      },
      "trade_offs": {
        "cost": "-60%",
        "accuracy": "-8%"
      }
    },
    {
      "name": "高性能方案",
      "description": "使用更强大的模型和更多技能",
      "changes": {
        "model": "gpt-4",
        "skills": ["sentiment-analyzer", "response-generator", "intent-classifier"]
      },
      "trade_offs": {
        "cost": "+150%",
        "accuracy": "+12%"
      }
    }
  ],
  "validation": {
    "warnings": [],
    "errors": [],
    "recommendations": [
      "建议在实际部署前进行小流量A/B测试",
      "情感分析阈值可根据业务需求调整"
    ]
  },
  "deployment": {
    "can_deploy": true,
    "estimated_time": "5 minutes",
    "preview_url": "/api/agentspec/preview/asm_20260306_001",
    "deploy_url": "/api/agentspec/deploy/asm_20260306_001"
  }
}
```

---

## 三、核心能力实现

### 3.1 场景解析引擎

```python
class ScenarioAnalyzer:
    """场景解析：自然语言 → 结构化任务"""
    
    def analyze(self, scenario: str, context: dict) -> ScenarioAnalysis:
        # 1. 意图识别
        intent = self.intent_classifier.classify(scenario)
        # e.g., "build_customer_service_agent"
        
        # 2. 任务分解（使用LLM）
        subtasks = self.task_decomposer.decompose(scenario)
        # e.g., [sentiment_analysis, response_generation, db_query]
        
        # 3. 能力需求提取
        required_capabilities = []
        for task in subtasks:
            caps = self.capability_mapper.map(task)
            required_capabilities.extend(caps)
        
        # 4. 约束条件解析
        constraints = self.constraint_parser.parse(context)
        
        return ScenarioAnalysis(
            intent=intent,
            subtasks=subtasks,
            required_capabilities=required_capabilities,
            constraints=constraints
        )
```

**关键技术**：
- **意图分类器**：Fine-tuned BERT/Qwen，支持50+常见意图
- **任务分解**：使用LLM（如Claude/GPT-4）进行 Chain-of-Thought 分解
- **能力映射**：预定义能力-任务映射表 + 语义匹配

---

### 3.2 资源匹配引擎

```python
class ResourceMatcher:
    """基于能力需求匹配最佳资源"""
    
    def match(self, capabilities: List[str], constraints: dict) -> ResourceSet:
        candidates = []
        
        for cap in capabilities:
            # 1. 向量检索语义相似资源
            semantic_results = self.vector_search.search(
                query=cap,
                filters={
                    "status": "formal",
                    "language": constraints.get("language"),
                    "complexity": constraints.get("complexity")
                }
            )
            
            # 2. 知识图谱扩展（相关资源）
            graph_results = self.knowledge_graph.find_related(
                capability=cap,
                relation="implements"
            )
            
            # 3. 效果预测排序
            scored_results = self.effect_predictor.predict(
                resources=semantic_results + graph_results,
                context=constraints
            )
            
            # 4. 选择Top-K
            best_match = scored_results[0]
            candidates.append(best_match)
        
        # 5. 冲突检测与解决
        resolved_set = self.conflict_resolver.resolve(candidates)
        
        return resolved_set
```

**关键技术**：
- **多路召回**：向量检索 + 图谱扩展 + 标签过滤
- **效果预测**：基于历史数据的成功率预测模型
- **冲突检测**：检查资源间的依赖、版本冲突

---

### 3.3 智能组装引擎

```python
class SpecAssembler:
    """将匹配的资源组装成完整的 AgentSpec"""
    
    def assemble(self, resources: ResourceSet, scenario: ScenarioAnalysis) -> AgentSpec:
        spec = AgentSpec()
        
        # 1. 选择基础Prompt
        spec.prompt = self.select_prompt(
            intent=scenario.intent,
            style=scenario.constraints.get("style")
        )
        
        # 2. 选择模型
        spec.model = self.select_model(
            complexity=scenario.complexity,
            latency_constraint=scenario.constraints.get("max_latency"),
            budget=scenario.constraints.get("budget")
        )
        
        # 3. 组装Skills
        spec.skills = self.order_skills(resources.skills, scenario.subtasks)
        
        # 4. 组装MCPs
        spec.mcps = self.configure_mcps(
            resources.mcps,
            existing=scenario.context.get("existing_resources")
        )
        
        # 5. 生成工作流
        spec.workflow = self.generate_workflow(
            subtasks=scenario.subtasks,
            components=spec.components
        )
        
        # 6. 优化配置
        spec = self.optimizer.optimize(spec, scenario.constraints)
        
        return spec
```

**关键技术**：
- **Prompt选择**：基于意图和风格的智能匹配
- **模型选型**：考虑延迟、成本、准确性的多目标优化
- **工作流生成**：基于子任务依赖关系自动生成执行流程
- **配置优化**：自动调整temperature、max_tokens等参数

---

### 3.4 效果预测引擎

```python
class PerformancePredictor:
    """预测组装后Agent的性能表现"""
    
    def predict(self, spec: AgentSpec, context: dict) -> PerformancePrediction:
        features = self.extract_features(spec, context)
        
        # 1. 成功率预测
        success_rate = self.success_rate_model.predict(features)
        
        # 2. 延迟预测
        latency = self.latency_model.predict(features)
        
        # 3. 成本预测
        cost = self.cost_model.predict(features)
        
        # 4. 瓶颈识别
        bottlenecks = self.identify_bottlenecks(spec)
        
        # 5. 置信度评估
        confidence = self.assess_confidence(features)
        
        return PerformancePrediction(
            success_rate=success_rate,
            latency=latency,
            cost=cost,
            bottlenecks=bottlenecks,
            confidence=confidence
        )
```

**关键技术**：
- **特征工程**：资源特征 + 历史表现 + 用户特征
- **预测模型**：GBDT/LightGBM，轻量高效
- **在线学习**：根据实际生产数据持续优化

---

## 四、数据模型设计

### 4.1 场景模板库

```json
{
  "scenario_templates": [
    {
      "template_id": "customer_service_bot",
      "name": "智能客服机器人",
      "description": "能自动回复用户咨询的客服Agent",
      "keywords": ["客服", "回复", "咨询", "support"],
      "required_capabilities": [
        "intent_recognition",
        "knowledge_retrieval", 
        "response_generation"
      ],
      "recommended_components": {
        "prompt_type": "customer_service_expert",
        "skills": ["intent_classifier", "faq_retriever", "response_generator"],
        "mcps": ["knowledge_base", "ticket_system"]
      },
      "example_queries": [
        "我要做个客服机器人",
        "自动回复用户消息",
        "智能客服助手"
      ]
    }
  ]
}
```

### 4.2 能力-资源映射表

```json
{
  "capability_map": {
    "sentiment_analysis": {
      "description": "分析文本情感倾向",
      "required_resources": {
        "skills": ["sentiment-analyzer", "emotion-detector"],
        "prompts": ["sentiment-analysis-instruction"]
      },
      "metrics": {
        "accuracy": ">=0.85",
        "latency": "<100ms"
      }
    }
  }
}
```

### 4.3 组装规则引擎

```yaml
assembly_rules:
  - rule_id: "model_latency_budget"
    condition: "constraints.max_latency < 300ms"
    action: "select_model: [gpt-3.5-turbo, claude-instant]"
    priority: 10
    
  - rule_id: "high_accuracy_required"  
    condition: "requirements.accuracy == 'high'"
    action: "select_model: [gpt-4, claude-3-opus]"
    priority: 9
    
  - rule_id: "skill_conflict_resolution"
    condition: "skills.have_conflict('v1', 'v2')"
    action: "select_latest_version"
    priority: 8
```

---

## 五、API 扩展设计

### 5.1 异步组装（复杂场景）

```http
POST /api/discover/assemble/async

# 返回
task_id: "asm_task_001"
poll_url: "/api/discover/assemble/status/asm_task_001"

# 查询状态
GET /api/discover/assemble/status/asm_task_001

# 返回
{
  "status": "processing",  // processing | completed | failed
  "progress": 75,
  "stage": "优化配置"
}
```

### 5.2 交互式组装（多轮澄清）

```http
POST /api/discover/assemble/interactive

# 第一轮：模糊场景
{
  "scenario": "我想做个AI助手"
}

# 返回：需要澄清
{
  "status": "needs_clarification",
  "questions": [
    {
      "field": "use_case",
      "question": "这个助手主要用来做什么？",
      "options": ["客服", "写作", "编程", "数据分析", "其他"]
    },
    {
      "field": "target_users", 
      "question": "主要服务对象是谁？",
      "options": ["内部员工", "外部客户", "开发者"]
    }
  ]
}

# 第二轮：补充信息
{
  "scenario": "我想做个AI助手",
  "clarifications": {
    "use_case": "客服",
    "target_users": "外部客户"
  }
}

# 返回：组装结果
{
  "status": "completed",
  "recommended_spec": {...}
}
```

### 5.3 相似场景推荐

```http
GET /api/discover/similar?scenario_hash={hash}&limit=5

# 返回
{
  "similar_assemblies": [
    {
      "assembly_id": "asm_20260301_123",
      "scenario": "电商客服机器人，处理退换货咨询",
      "adoption_rate": 0.95,
      "avg_rating": 4.8
    }
  ]
}
```

---

## 六、实施路线图

### Phase 1：MVP（4-6周）

**目标**：基础场景识别 + 简单组装

**功能范围**：
- [ ] 10个常见场景模板（客服、写作、编程等）
- [ ] 基于模板的规则匹配（非AI生成）
- [ ] 简单的资源组合（Prompt + 1-2个Skill）
- [ ] 基础效果预估（基于历史平均值）

**接口**：
```
POST /api/discover/assemble (同步，响应时间<2s)
```

---

### Phase 2：智能增强（8-12周）

**目标**：AI驱动的任务分解 + 多资源组装

**新增能力**：
- [ ] LLM驱动的场景解析（意图识别 + 任务分解）
- [ ] 知识图谱支持（资源关系推理）
- [ ] 多Skill组合（3-5个Skill协同）
- [ ] MCP自动配置（工具选择和参数填充）
- [ ] 工作流自动生成（子任务依赖关系）

**接口**：
```
POST /api/discover/assemble (异步，支持复杂场景)
POST /api/discover/assemble/interactive (交互式)
```

---

### Phase 3：效果优化（12-16周）

**目标**：效果预测 + 在线学习

**新增能力**：
- [ ] 效果预测模型（成功率、延迟、成本）
- [ ] 瓶颈识别与优化建议
- [ ] 替代方案推荐
- [ ] 基于生产数据的在线学习
- [ ] A/B测试框架集成

**接口**：
```
GET /api/discover/similar (相似场景推荐)
POST /api/discover/feedback (效果反馈)
```

---

## 七、核心竞争力构建

### 7.1 数据飞轮

```
用户描述场景 → 组装AgentSpec → 部署使用 → 收集效果数据 → 优化模型 → 提升组装质量
        ↑                                                                            ↓
        └──────────────── 更多用户信赖，更多数据积累 ───────────────────────────────┘
```

**关键数据资产**：
- 场景-资源映射关系
- 资源组合效果数据
- 用户偏好模式
- 领域特定知识

### 7.2 竞争壁垒

| 能力 | 壁垒深度 | 建设周期 |
|------|---------|---------|
| 场景模板库 | 🟡 中 | 3-6个月 |
| 知识图谱 | 🟢 高 | 6-12个月 |
| 效果预测模型 | 🟢🟢 极高 | 12-18个月 |
| 领域专业知识 | 🟢🟢 极高 | 持续积累 |

---

## 八、总结

### 8.1 核心设计思想

**从"资源检索"到"解决方案生成"**：
- 传统：用户搜关键词 → 返回资源列表 → 用户自己组装
- 新方案：用户描述场景 → 系统解析需求 → 自动生成完整AgentSpec

### 8.2 关键成功因素

1. **场景理解准确**：意图识别和任务分解的准确性决定组装质量
2. **资源匹配精准**：基于能力和效果的匹配，而非简单关键词
3. **效果可预测**：让用户在部署前就知道预期效果
4. **持续优化**：根据生产数据不断改进组装策略

### 8.3 一句话描述

> **新的 Discovery 接口是"AI Solution Architect"——用户只需描述业务场景，系统自动设计、组装、优化出完整的Agent解决方案。**

---

*设计时间：2026年3月6日*  
*版本：v1.0*  
*建议用途：技术方案评审和开发规划*