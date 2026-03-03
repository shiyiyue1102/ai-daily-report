# Prompt 与 Skill 调优技术调研报告

## 摘要

本报告系统梳理当前主流的 Prompt 工程优化技术和 Skill（技能）调优方法，涵盖从基础技巧到高级策略的完整技术栈，为 AI 应用开发者提供可落地的调优指南。

---

## 一、Prompt 调优技术详解

### 1.1 基础调优方法

#### 1.1.1 清晰指令原则（Clear Instruction）

**核心思想**：模型只能理解你明确说出的内容，不能猜测你的意图。

**技术要点**：
```
❌ 差："写一篇文章"
✅ 好："写一篇关于 Python 异步编程的技术博客，目标读者是有 2 年经验的开发者，
      要求包含：1）async/await 基础概念 2）实际代码示例 3）常见陷阱和解决方案，
      字数控制在 1500-2000 字"
```

**优化维度**：
| 维度 | 优化策略 | 示例 |
|------|---------|------|
| **任务类型** | 明确指定输出格式 | "以 Markdown 格式输出" |
| **目标受众** | 定义读者背景 | "面向初学者" / "面向技术专家" |
| **内容结构** | 要求特定结构 | "包含引言、正文、结论三部分" |
| **风格语调** | 指定语言风格 | "使用专业但易懂的语言" |
| **约束条件** | 设置明确限制 | "避免使用专业术语" / "字数不超过 500" |

#### 1.1.2 少样本学习（Few-Shot Learning）

**核心思想**：通过提供示例，让模型理解任务模式和输出格式。

**技术实现**：
```
任务：将客户评论分类为正面、负面或中性

示例 1：
评论："这个产品太棒了，完全超出预期！"
分类：正面

示例 2：
评论："质量一般，对得起这个价格"
分类：中性

示例 3：
评论："完全不能用，退货！"
分类：负面

待分类评论："物流很快，但是包装有点破损"
分类：
```

**样本选择策略**：
- **数量**：通常 3-5 个示例效果最佳，过多会浪费上下文
- **多样性**：覆盖不同场景和边界情况
- **代表性**：选择典型、清晰的样本
- **一致性**：示例格式必须统一

#### 1.1.3 思维链提示（Chain-of-Thought, CoT）

**核心思想**：引导模型展示推理过程，而非直接给出答案。

**基础 CoT**：
```
问题：一个农场有鸡和兔，头共 35 个，脚共 94 只。鸡和兔各有多少只？

请按以下步骤思考：
1. 设鸡的数量为 x，兔的数量为 y
2. 根据题意列出方程组
3. 解方程组
4. 验证答案

请展示完整的推理过程。
```

**进阶 CoT 变体**：

| 变体 | 描述 | 适用场景 |
|------|------|---------|
| **Zero-Shot CoT** | 添加"让我们一步步思考" | 简单推理任务 |
| **Few-Shot CoT** | 提供带推理过程的示例 | 复杂推理任务 |
| **Self-Consistency** | 多次采样，选择最常见答案 | 需要高置信度的任务 |
| **Tree of Thoughts** | 探索多条推理路径 | 开放式问题求解 |

### 1.2 高级调优技术

#### 1.2.1 角色设定（Role Prompting）

**核心思想**：通过角色设定激活模型的特定知识领域和行为模式。

**技术框架**：
```
你是一位[专业角色]，拥有[相关经验/资质]。
你的任务是[具体任务]。
请从[专业角度]出发，[具体要求]。
```

**角色设计要素**：
- **身份**：资深 Python 工程师、产品总监、法律顾问等
- **背景**：10 年工作经验、服务过 Fortune 500 公司等
- **专长**：微服务架构、用户体验设计、合同审查等
- **风格**：严谨务实、创新大胆、保守稳健等

**示例**：
```
你是一位资深的数据库架构师，曾在阿里巴巴和字节跳动负责过大型分布式数据库设计。
你的专长是 MySQL 性能优化和分库分表策略。
请像给技术团队做分享一样，详细解释以下 SQL 查询的优化方案...
```

#### 1.2.2 结构化提示（Structured Prompting）

**核心思想**：使用标记语言（XML、Markdown、JSON）结构化 Prompt，提升模型理解能力。

**XML 标记法**：
```xml
<context>
  你正在开发一个电商平台的订单系统。
  当前使用的是微服务架构，订单服务需要与库存服务、支付服务通信。
</context>

<requirements>
  <functional>
    - 支持订单创建、查询、取消
    - 支持库存预占和释放
    - 支持多种支付方式
  </functional>
  <non-functional>
    - QPS > 1000
    - 延迟 < 200ms
    - 可用性 99.9%
  </non-functional>
</requirements>

<task>
  设计订单服务的核心接口和数据模型。
</task>
```

**优势**：
- 边界清晰，减少歧义
- 便于程序化处理和模板化
- 支持嵌套和层级关系

#### 1.2.3 上下文压缩（Context Compression）

**核心思想**：在有限的上下文窗口内，最大化信息密度。

**技术策略**：

| 策略 | 方法 | 适用场景 |
|------|------|---------|
| **摘要替代** | 用摘要代替全文 | 长文档处理 |
| **关键词提取** | 只保留关键信息 | 信息检索 |
| **向量化检索** | 动态加载相关片段 | RAG 系统 |
| **分层结构** | 先概览后详情 | 复杂文档分析 |

**实现示例**：
```
[文档摘要]
这是一份关于新劳动法修正案的政策文件，核心变化包括：
1. 试用期最长缩短至 3 个月（原 6 个月）
2. 远程办公权益正式写入法律
3. 加班补偿标准提高 20%

[原文片段]
（仅保留与问题相关的具体条款）

[用户问题]
我们公司当前的试用期政策是否需要调整？
```

#### 1.2.4 动态提示（Dynamic Prompting）

**核心思想**：根据上下文和用户输入，动态组装 Prompt。

**技术架构**：
```python
class DynamicPromptBuilder:
    def __init__(self):
        self.base_template = "..."
        self.context_fragments = []
        
    def add_context(self, context_type, content):
        """动态添加上下文"""
        self.context_fragments.append({
            "type": context_type,
            "content": content
        })
    
    def build(self, user_query):
        """组装最终 Prompt"""
        prompt_parts = [self.base_template]
        
        # 根据查询意图选择相关上下文
        relevant_contexts = self.select_contexts(user_query)
        for ctx in relevant_contexts:
            prompt_parts.append(f"[{ctx['type']}]\n{ctx['content']}")
        
        prompt_parts.append(f"[用户问题]\n{user_query}")
        return "\n\n".join(prompt_parts)
```

### 1.3 Prompt 评估与迭代

#### 1.3.1 评估指标体系

| 维度 | 指标 | 测量方法 |
|------|------|---------|
| **准确性** | 输出与预期的一致性 | 人工标注、自动评分 |
| **完整性** | 是否覆盖所有要求 | 检查清单验证 |
| **一致性** | 多次执行结果稳定性 | 多次采样对比 |
| **合规性** | 是否遵循安全规范 | 敏感词检测、人工审核 |
| **效率** | Token 使用效率 | 计算输入输出 Token 比 |

#### 1.3.2 A/B 测试框架

```python
class PromptABTest:
    def __init__(self, prompt_a, prompt_b):
        self.variants = {
            'A': prompt_a,
            'B': prompt_b
        }
        self.results = {'A': [], 'B': []}
    
    def run_test(self, test_cases, sample_size=100):
        """执行 A/B 测试"""
        for case in test_cases:
            # 随机分配变体
            variant = random.choice(['A', 'B'])
            prompt = self.variants[variant]
            
            # 执行并记录结果
            result = self.execute_prompt(prompt, case)
            self.results[variant].append({
                'case': case,
                'output': result,
                'metrics': self.evaluate(result, case.expected)
            })
    
    def analyze(self):
        """统计分析结果"""
        stats = {}
        for variant in ['A', 'B']:
            metrics = [r['metrics'] for r in self.results[variant]]
            stats[variant] = {
                'accuracy': np.mean([m['accuracy'] for m in metrics]),
                'avg_tokens': np.mean([m['token_count'] for m in metrics]),
                'latency': np.mean([m['latency'] for m in metrics])
            }
        return stats
```

---

## 二、Skill 调优技术详解

### 2.1 Skill 设计原则

#### 2.1.1 单一职责原则（Single Responsibility）

**核心思想**：一个 Skill 只做一件事，并做好一件事。

**反例**：
```
❌ 差设计：
Skill: "数据处理"
功能：清洗数据 + 特征工程 + 模型训练 + 结果可视化
问题：功能臃肿，难以复用和维护
```

**正例**：
```
✅ 好设计：
Skill 1: "数据清洗" - 处理缺失值、异常值、格式统一
Skill 2: "特征工程" - 特征选择、特征变换、特征组合
Skill 3: "模型训练" - 训练指定模型并保存
Skill 4: "结果可视化" - 生成图表和报告
```

#### 2.1.2 接口契约设计

**核心思想**：明确定义 Skill 的输入输出契约，便于组合和调试。

**契约模板**：
```yaml
skill_name: text_summarizer
version: "1.0.0"
description: "将长文本摘要为指定长度的短文"

input_schema:
  type: object
  required: [text, max_length]
  properties:
    text:
      type: string
      description: "待摘要的原始文本"
      maxLength: 10000
    max_length:
      type: integer
      description: "摘要最大字数"
      minimum: 50
      maximum: 500
    style:
      type: string
      enum: [concise, detailed, bullet_points]
      default: concise

output_schema:
  type: object
  required: [summary, compression_ratio]
  properties:
    summary:
      type: string
      description: "生成的摘要文本"
    compression_ratio:
      type: number
      description: "压缩率（摘要长度/原文长度）"
    key_points:
      type: array
      items:
        type: string
      description: "关键要点列表"

error_codes:
  TEXT_TOO_LONG: "输入文本超过最大长度限制"
  INVALID_LENGTH: "指定的摘要长度超出范围"
  SUMMARIZATION_FAILED: "摘要生成失败"
```

### 2.2 Skill 实现优化

#### 2.2.1 提示词工程优化

**分层提示词架构**：
```
[系统层] System Prompt
├── 角色定义
├── 能力边界
└── 安全约束

[技能层] Skill Prompt
├── 任务描述
├── 输入格式
├── 处理逻辑
└── 输出格式

[上下文层] Context
├── 历史对话
├── 用户偏好
└── 环境信息

[输入层] User Input
└── 具体指令和数据
```

**优化技巧**：
1. **模板化**：使用 Jinja2 等模板引擎动态渲染
2. **参数化**：关键配置提取为可调参数
3. **版本化**：Prompt 变更纳入版本控制

#### 2.2.2 性能优化策略

| 优化方向 | 策略 | 效果 |
|---------|------|------|
| **延迟优化** | 流式输出、异步处理、预加载 | 减少用户等待时间 |
| **成本优化** | 上下文压缩、模型选择、缓存机制 | 降低 Token 消耗 |
| **质量优化** | 后置校验、重试机制、多模型集成 | 提升输出质量 |
| **稳定性** | 限流、熔断、降级 | 保障服务可用性 |

**缓存策略实现**：
```python
class SkillCache:
    def __init__(self, ttl=3600):
        self.cache = {}
        self.ttl = ttl
    
    def get_cache_key(self, skill_name, inputs):
        """生成缓存键（基于输入哈希）"""
        content = f"{skill_name}:{json.dumps(inputs, sort_keys=True)}"
        return hashlib.md5(content.encode()).hexdigest()
    
    def get(self, skill_name, inputs):
        key = self.get_cache_key(skill_name, inputs)
        if key in self.cache:
            entry = self.cache[key]
            if time.time() - entry['timestamp'] < self.ttl:
                return entry['result']
            else:
                del self.cache[key]
        return None
    
    def set(self, skill_name, inputs, result):
        key = self.get_cache_key(skill_name, inputs)
        self.cache[key] = {
            'result': result,
            'timestamp': time.time()
        }
```

#### 2.2.3 错误处理与重试

**分级重试策略**：
```python
class SkillExecutor:
    def __init__(self):
        self.retry_policy = {
            'max_attempts': 3,
            'backoff_factor': 2,
            'retryable_errors': [
                'RateLimitError',
                'TimeoutError',
                'ServiceUnavailableError'
            ]
        }
    
    async def execute_with_retry(self, skill, inputs):
        for attempt in range(self.retry_policy['max_attempts']):
            try:
                result = await skill.execute(inputs)
                
                # 后置校验
                if self.validate_result(result, skill.output_schema):
                    return result
                else:
                    raise ValidationError("Output validation failed")
                    
            except Exception as e:
                if attempt < self.retry_policy['max_attempts'] - 1 \
                   and type(e).__name__ in self.retry_policy['retryable_errors']:
                    wait_time = self.retry_policy['backoff_factor'] ** attempt
                    await asyncio.sleep(wait_time)
                    continue
                else:
                    raise
```

### 2.3 Skill 组合与编排

#### 2.3.1 串行编排（Sequential Pipeline）

```
[输入] → [Skill A] → [Skill B] → [Skill C] → [输出]
```

**适用场景**：
- 数据流处理（清洗 → 转换 → 分析）
- 内容生成（大纲 → 正文 → 润色）
- 审批流程（初审 → 复审 → 终审）

**代码示例**：
```python
class SequentialPipeline:
    def __init__(self, skills: List[Skill]):
        self.skills = skills
    
    async def execute(self, initial_input):
        context = initial_input
        for skill in self.skills:
            context = await skill.execute(context)
        return context
```

#### 2.3.2 并行编排（Parallel Execution）

```
         ┌→ [Skill A] ─┐
[输入] ──┼→ [Skill B] ─┼→ [聚合 Skill] → [输出]
         └→ [Skill C] ─┘
```

**适用场景**：
- 多维度分析（情感分析 + 关键词提取 + 主题分类）
- 多源信息整合（网页搜索 + 知识库检索 + 数据库查询）
- 多模型集成（多个模型同时预测，投票决定）

#### 2.3.3 条件编排（Conditional Routing）

```
              ┌─ 条件A ─→ [Skill A] ─┐
[输入] → [路由]─┼─ 条件B ─→ [Skill B] ─┼→ [输出]
              └─ 条件C ─→ [Skill C] ─┘
```

**路由决策实现**：
```python
class ConditionalRouter:
    def __init__(self):
        self.routes = []
    
    def add_route(self, condition: Callable, skill: Skill):
        self.routes.append((condition, skill))
    
    async def route(self, inputs):
        for condition, skill in self.routes:
            if condition(inputs):
                return await skill.execute(inputs)
        
        # 默认路由
        return await self.default_skill.execute(inputs)
```

#### 2.3.4 循环编排（Loop/Iteration）

```
[输入] → [Skill A] → [条件判断] ─┬─ 满足 ─→ [输出]
            ↑_______________└─ 不满足 ─┘
```

**适用场景**：
- 迭代优化（代码 review → 修改 → 再 review）
- 多轮对话（直到获取完整信息）
- 自适应处理（直到结果达标）

---

## 三、Prompt 与 Skill 协同优化

### 3.1 元提示技术（Meta-Prompting）

**核心思想**：使用模型来优化 Prompt 和 Skill。

**自动 Prompt 优化**：
```
[元提示]
你是一位 Prompt 工程专家。
请分析以下 Prompt，并提出优化建议：

原始 Prompt：
"""
{original_prompt}
"""

请从以下维度分析：
1. 清晰度：指令是否明确？
2. 完整性：是否覆盖所有必要信息？
3. 效率：是否存在冗余？
4. 安全性：是否存在注入风险？

输出格式：
- 问题列表
- 优化后的 Prompt
- 优化说明
```

### 3.2 自适应 Skill 选择

```python
class AdaptiveSkillSelector:
    def __init__(self, skill_registry, embedding_model):
        self.skills = skill_registry
        self.embedder = embedding_model
        
        # 预计算每个 Skill 的描述向量
        self.skill_vectors = {}
        for name, skill in self.skills.items():
            desc = skill.get_description()
            self.skill_vectors[name] = self.embedder.encode(desc)
    
    def select_skill(self, user_query, top_k=3):
        """基于语义相似度选择最合适的 Skill"""
        query_vector = self.embedder.encode(user_query)
        
        similarities = {}
        for name, skill_vector in self.skill_vectors.items():
            sim = cosine_similarity(query_vector, skill_vector)
            similarities[name] = sim
        
        # 返回 Top-K 最相关的 Skill
        return sorted(similarities.items(), key=lambda x: x[1], reverse=True)[:top_k]
```

### 3.3 反馈驱动的持续优化

**优化闭环**：
```
[执行] → [收集反馈] → [分析问题] → [优化 Prompt/Skill] → [验证效果] → [部署上线]
   ↑_______________________________________________________________|
```

**反馈收集维度**：
- **显式反馈**：用户评分、点赞/点踩、修改建议
- **隐式反馈**：重试次数、完成时间、后续操作
- **业务指标**：转化率、留存率、任务完成率

---

## 四、工具与框架推荐

### 4.1 Prompt 管理工具

| 工具 | 特点 | 适用场景 |
|------|------|---------|
| **LangSmith** | 全流程追踪、A/B 测试 | 生产环境 Prompt 管理 |
| **PromptLayer** | 版本控制、性能监控 | 团队协作 Prompt 开发 |
| **Weights & Biases** | 实验追踪、可视化 | 研究性 Prompt 调优 |
| **OpenAI Playground** | 快速迭代、参数调节 | 原型设计和测试 |

### 4.2 Skill 开发框架

| 框架 | 特点 | 适用场景 |
|------|------|---------|
| **LangChain** | 生态丰富、组件化 | 复杂 Skill 编排 |
| **LlamaIndex** | RAG 专长、数据集成 | 知识库类 Skill |
| **Semantic Kernel** | 微软生态、企业级 | 企业 Skill 开发 |
| **AutoGen** | 多 Agent 协作 | Multi-Agent 系统 |

### 4.3 评估与测试工具

| 工具 | 功能 |
|------|------|
**Promptfoo** | 系统化 Prompt 测试、回归测试
| **DeepEval** | LLM 输出评估指标库 |
| **Ragas** | RAG 系统评估框架 |
| **TruLens** | LLM 应用可解释性分析 |

---

## 五、最佳实践总结

### 5.1 Prompt 调优 Checklist

- [ ] 明确任务类型和输出格式
- [ ] 提供充分的上下文信息
- [ ] 使用少样本示例引导
- [ ] 要求模型展示推理过程（CoT）
- [ ] 设置明确的约束条件
- [ ] 采用角色设定激活专业知识
- [ ] 使用结构化标记提升可读性
- [ ] 建立评估指标和测试集
- [ ] 实施 A/B 测试验证效果
- [ ] 版本控制所有 Prompt 变更

### 5.2 Skill 开发 Checklist

- [ ] 遵循单一职责原则
- [ ] 定义清晰的输入输出契约
- [ ] 实现完善的错误处理
- [ ] 添加后置结果校验
- [ ] 实施缓存策略优化性能
- [ ] 支持流式输出降低延迟
- [ ] 记录执行日志便于调试
- [ ] 设计可组合和可扩展的架构
- [ ] 建立监控和告警机制
- [ ] 编写使用文档和示例代码

### 5.3 避免的常见错误

**Prompt 层面**：
1. ❌ 指令过于模糊或过于复杂
2. ❌ 忽视 Token 限制导致截断
3. ❌ 未处理敏感信息泄露风险
4. ❌ 缺乏对输出格式的约束
5. ❌ 未考虑多语言和文化差异

**Skill 层面**：
1. ❌ 功能过于臃肿，违反单一职责
2. ❌ 缺乏输入校验导致错误传播
3. ❌ 未实现幂等性，重复执行产生副作用
4. ❌ 忽视并发安全和资源竞争
5. ❌ 缺乏版本管理，升级导致兼容性问题

---

## 六、未来趋势

### 6.1 Prompt 自动优化
- **AutoPrompt**：自动搜索最优 Prompt
- **Prompt Tuning**：可学习的 Soft Prompt
- **Prompt 合成**：基于示例自动生成 Prompt

### 6.2 Skill 标准化
- **统一接口标准**：类似 OpenAPI 的 Skill 规范
- **Skill 市场**：可交易和复用的 Skill 生态
- **Skill 组合**：可视化编排降低开发门槛

### 6.3 多模态 Skill
- **跨模态理解**：文本 + 图像 + 音频的统一处理
- **模态转换**：自动生成配图、配音、视频
- **富媒体输出**：交互式图表、3D 模型等

---

## 参考资源

1. OpenAI Prompt Engineering Guide: https://platform.openai.com/docs/guides/prompt-engineering
2. Anthropic Prompt Design: https://docs.anthropic.com/claude/docs/prompt-design
3. Google Prompt Engineering Whitepaper: https://www.kaggle.com/whitepaper-prompt-engineering
4. LangChain Documentation: https://python.langchain.com/docs/get_started/introduction
5. DSPy: Compiling Declarative Language Model Calls: https://github.com/stanfordnlp/dspy

---

*报告生成时间：2026年3月3日*  
*版本：v1.0*