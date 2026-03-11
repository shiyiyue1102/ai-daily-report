# Matrix 安全审计与合规能力建设技术方案

## 一、方案概述

本方案设计 Matrix 平台 AI 资源（Prompt/Skill/MCP/AgentCard）的安全审核与合规能力，包括资源状态机、审计任务抽象接口、审核维度与实现方式。

---

## 二、资源状态机设计

### 2.1 状态定义

```
┌─────────────┐
│   DRAFT     │ ← 草稿状态
│   (草稿)     │
└──────┬──────┘
       │ 提交审核
       ▼
┌─────────────┐
│  REVIEWING  │ ← 审核中状态
│   (审核中)   │
└──────┬──────┘
       │
       ├──────────────┬──────────────┐
       │ 审核通过     │ 审核拒绝     │ 审核异常
       ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ PENDING_    │ │  REJECTED   │ │   ERROR     │
│  PUBLISH    │ │   (已拒绝)   │ │   (异常)    │
│  (待发布)    │ └─────────────┘ └─────────────┘
└──────┬──────┘       │               │
       │ 发布         │ 重新提交       │ 重试
       ▼              ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│   FORMAL    │ │   DRAFT     │ │  REVIEWING  │
│   (已发布)   │ │   (草稿)     │ │   (审核中)   │
└──────┬──────┘ └─────────────┘ └─────────────┘
       │
       │ 下架/归档
       ▼
┌─────────────┐
│  ARCHIVED   │
│   (已归档)   │
└─────────────┘
```

### 2.2 状态表设计

```sql
-- 资源状态表
CREATE TABLE resource_states (
    id UUID PRIMARY KEY,
    resource_id UUID REFERENCES resources(id),
    
    -- 当前状态
    current_state VARCHAR(50) NOT NULL, -- DRAFT, REVIEWING, PENDING_PUBLISH, FORMAL, REJECTED, ERROR, ARCHIVED
    
    -- 状态变更历史
    previous_state VARCHAR(50),
    state_changed_at TIMESTAMP DEFAULT NOW(),
    state_changed_by VARCHAR(100), -- 用户 ID 或系统
    
    -- 审核相关
    audit_task_id UUID, -- 关联的审核任务
    audit_result JSONB, -- 审核结果详情
    rejection_reason TEXT, -- 拒绝原因
    
    -- 版本信息
    version VARCHAR(50),
    is_current BOOLEAN DEFAULT true,
    
    created_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE(resource_id, version)
);

-- 状态变更日志表
CREATE TABLE resource_state_history (
    id UUID PRIMARY KEY,
    resource_id UUID NOT NULL,
    resource_version VARCHAR(50),
    
    -- 状态变更
    from_state VARCHAR(50),
    to_state VARCHAR(50) NOT NULL,
    
    -- 触发信息
    triggered_by VARCHAR(100), -- 用户/系统
    triggered_action VARCHAR(100), -- submit_for_review, approve, reject, publish
    
    -- 审核信息
    audit_task_id UUID,
    audit_result JSONB,
    rejection_reason TEXT,
    
    -- 元数据
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 2.3 状态机实现

```java
@Service
public class ResourceStateMachine {
    
    private final ResourceStateRepository stateRepository;
    private final AuditTaskService auditTaskService;
    private final ApplicationEventPublisher eventPublisher;
    
    /**
     * 提交审核：DRAFT → REVIEWING
     */
    @Transactional
    public ResourceState submitForReview(String resourceId, String userId) {
        ResourceState currentState = getCurrentState(resourceId);
        
        // 状态校验
        if (!currentState.getCurrentState().equals("DRAFT")) {
            throw new IllegalStateException("只有草稿状态的资源可以提交审核");
        }
        
        // 创建审核任务
        AuditTask auditTask = auditTaskService.createTask(
            resourceId,
            currentState.getVersion()
        );
        
        // 更新状态
        ResourceState newState = stateRepository.save(ResourceState.builder()
            .resourceId(resourceId)
            .currentState("REVIEWING")
            .previousState("DRAFT")
            .auditTaskId(auditTask.getId())
            .version(currentState.getVersion())
            .build());
        
        // 发布事件
        eventPublisher.publishEvent(new ResourceReviewingEvent(this, resourceId, auditTask.getId()));
        
        return newState;
    }
    
    /**
     * 审核通过：REVIEWING → PENDING_PUBLISH
     */
    @Transactional
    public ResourceState approveAudit(String resourceId, UUID auditTaskId, AuditResult result) {
        ResourceState currentState = getCurrentState(resourceId);
        
        if (!currentState.getCurrentState().equals("REVIEWING")) {
            throw new IllegalStateException("只有审核中的资源可以审批通过");
        }
        
        ResourceState newState = stateRepository.save(ResourceState.builder()
            .resourceId(resourceId)
            .currentState("PENDING_PUBLISH")
            .previousState("REVIEWING")
            .auditResult(toJson(result))
            .version(currentState.getVersion())
            .build());
        
        eventPublisher.publishEvent(new ResourceApprovedEvent(this, resourceId, result));
        
        return newState;
    }
    
    /**
     * 审核拒绝：REVIEWING → REJECTED
     */
    @Transactional
    public ResourceState rejectAudit(String resourceId, UUID auditTaskId, String reason) {
        ResourceState currentState = getCurrentState(resourceId);
        
        if (!currentState.getCurrentState().equals("REVIEWING")) {
            throw new IllegalStateException("只有审核中的资源可以拒绝");
        }
        
        ResourceState newState = stateRepository.save(ResourceState.builder()
            .resourceId(resourceId)
            .currentState("REJECTED")
            .previousState("REVIEWING")
            .rejectionReason(reason)
            .version(currentState.getVersion())
            .build());
        
        eventPublisher.publishEvent(new ResourceRejectedEvent(this, resourceId, reason));
        
        return newState;
    }
    
    /**
     * 发布：PENDING_PUBLISH → FORMAL
     */
    @Transactional
    public ResourceState publish(String resourceId, String userId) {
        ResourceState currentState = getCurrentState(resourceId);
        
        if (!currentState.getCurrentState().equals("PENDING_PUBLISH")) {
            throw new IllegalStateException("只有待发布状态的资源可以发布");
        }
        
        ResourceState newState = stateRepository.save(ResourceState.builder()
            .resourceId(resourceId)
            .currentState("FORMAL")
            .previousState("PENDING_PUBLISH")
            .version(currentState.getVersion())
            .build());
        
        eventPublisher.publishEvent(new ResourcePublishedEvent(this, resourceId));
        
        return newState;
    }
}
```

---

## 三、审计任务抽象接口设计

### 3.1 审计任务接口

```java
/**
 * 审计任务抽象接口
 */
public interface AuditTask {
    
    /**
     * 任务 ID
     */
    String getId();
    
    /**
     * 资源 ID
     */
    String getResourceId();
    
    /**
     * 资源类型
     */
    ResourceType getResourceType();
    
    /**
     * 资源内容
     */
    ResourceContent getContent();
    
    /**
     * 执行审计
     */
    AuditResult execute();
    
    /**
     * 审计状态
     */
    AuditStatus getStatus();
    
    /**
     * 创建时间
     */
    Instant getCreatedAt();
}

/**
 * 审计结果
 */
@Data
@Builder
public class AuditResult {
    
    /**
     * 是否通过
     */
    private boolean passed;
    
    /**
     * 审计得分（0-100）
     */
    private Integer score;
    
    /**
     * 审计维度详情
     */
    private List<AuditDimensionResult> dimensions;
    
    /**
     * 拒绝原因
     */
    private String rejectionReason;
    
    /**
     * 建议修复项
     */
    private List<String> suggestions;
    
    /**
     * 审计元数据
     */
    private AuditMetadata metadata;
}

/**
 * 审计维度结果
 */
@Data
@Builder
public class AuditDimensionResult {
    
    /**
     * 维度名称
     */
    private String dimensionName;
    
    /**
     * 是否通过
     */
    private boolean passed;
    
    /**
     * 得分
     */
    private Integer score;
    
    /**
     * 详情
     */
    private String details;
    
    /**
     * 风险等级
     */
    private RiskLevel riskLevel;
}
```

### 3.2 审计器接口

```java
/**
 * 审计器抽象接口
 */
public interface Auditor {
    
    /**
     * 支持的资源类型
     */
    Set<ResourceType> getSupportedResourceTypes();
    
    /**
     * 支持的审计维度
     */
    Set<AuditDimension> getSupportedDimensions();
    
    /**
     * 执行审计
     */
    AuditResult audit(ResourceContent content, AuditContext context);
    
    /**
     * 审计器优先级（多个审计器链式调用）
     */
    default int getPriority() {
        return 0;
    }
}

/**
 * 审计上下文
 */
@Data
@Builder
public class AuditContext {
    
    /**
     * 资源 ID
     */
    private String resourceId;
    
    /**
     * 资源类型
     */
    private ResourceType resourceType;
    
    /**
     * 工作空间 ID
     */
    private String workspaceId;
    
    /**
     * 提交者 ID
     */
    private String submitterId;
    
    /**
     * 审计配置
     */
    private AuditConfig config;
}
```

### 3.3 审计器链

```java
@Service
public class AuditChainExecutor {
    
    private final List<Auditor> auditors;
    
    public AuditChainExecutor(List<Auditor> auditors) {
        // 按优先级排序
        this.auditors = auditors.stream()
            .sorted(Comparator.comparingInt(Auditor::getPriority))
            .collect(Collectors.toList());
    }
    
    /**
     * 执行审计链
     */
    public AuditResult executeAuditChain(ResourceContent content, AuditContext context) {
        List<AuditDimensionResult> allResults = new ArrayList<>();
        boolean allPassed = true;
        int totalScore = 0;
        
        for (Auditor auditor : auditors) {
            // 检查是否支持该资源类型
            if (!auditor.getSupportedResourceTypes().contains(context.getResourceType())) {
                continue;
            }
            
            // 执行审计
            AuditResult result = auditor.audit(content, context);
            
            // 合并结果
            allResults.addAll(result.getDimensions());
            totalScore += result.getScore();
            
            // 如果有维度不通过，标记为失败
            if (!result.isPassed()) {
                allPassed = false;
            }
        }
        
        // 综合结果
        return AuditResult.builder()
            .passed(allPassed)
            .score(totalScore / allResults.size())
            .dimensions(allResults)
            .build();
    }
}
```

---

## 四、审核维度与实现方式

### 4.1 审核维度总览

| 维度 | 审核内容 | 实现方式 | 优先级 |
|------|---------|---------|--------|
| **敏感信息检测** | PII、密钥、密码等 | 正则 + NER 模型 | P0 |
| **内容安全** | 违法、暴力、色情等 | 内容安全 API | P0 |
| **Prompt 注入检测** | 注入攻击、越狱等 | 分类模型 | P0 |
| **代码安全** | 恶意代码、漏洞等 | 静态分析工具 | P1 |
| **合规检查** | 版权、许可证等 | 规则引擎 | P1 |
| **质量评估** | 完整性、可用性等 | 大模型评分 | P2 |

### 4.2 各维度详细实现

#### 4.2.1 敏感信息检测（P0）

**实现方式：正则 + NER 模型**

```java
@Component
public class SensitiveInfoAuditor implements Auditor {
    
    private final Pattern apiKeyPattern = Pattern.compile("(?i)(api[_-]?key|apikey)\\s*[:=]\\s*['\"]?[a-zA-Z0-9]{20,}['\"]?");
    private final Pattern passwordPattern = Pattern.compile("(?i)(password|passwd|pwd)\\s*[:=]\\s*['\"]?.+['\"]?");
    private final Pattern privateKeyPattern = Pattern.compile("-----BEGIN (RSA |EC )?PRIVATE KEY-----");
    
    private final NERModel nerModel; // 命名实体识别模型
    
    @Override
    public Set<ResourceType> getSupportedResourceTypes() {
        return Set.of(ResourceType.PROMPT, ResourceType.SKILL);
    }
    
    @Override
    public Set<AuditDimension> getSupportedDimensions() {
        return Set.of(AuditDimension.SENSITIVE_INFO);
    }
    
    @Override
    public AuditResult audit(ResourceContent content, AuditContext context) {
        List<AuditDimensionResult> results = new ArrayList<>();
        
        // 1. 正则检测
        List<String> regexMatches = new ArrayList<>();
        if (apiKeyPattern.matcher(content.getText()).find()) {
            regexMatches.add("API Key");
        }
        if (passwordPattern.matcher(content.getText()).find()) {
            regexMatches.add("Password");
        }
        if (privateKeyPattern.matcher(content.getText()).find()) {
            regexMatches.add("Private Key");
        }
        
        // 2. NER 检测（PII）
        List<Entity> entities = nerModel.predict(content.getText());
        List<String> piiTypes = entities.stream()
            .filter(e -> e.getType().startsWith("PII_"))
            .map(Entity::getType)
            .collect(Collectors.toList());
        
        boolean passed = regexMatches.isEmpty() && piiTypes.isEmpty();
        
        results.add(AuditDimensionResult.builder()
            .dimensionName("敏感信息检测")
            .passed(passed)
            .score(passed ? 100 : 0)
            .details(String.format("发现敏感信息：API Key[%d], Password[%d], PII[%d]",
                regexMatches.size(), regexMatches.size(), piiTypes.size()))
            .riskLevel(passed ? RiskLevel.LOW : RiskLevel.HIGH)
            .build());
        
        return AuditResult.builder()
            .passed(passed)
            .score(passed ? 100 : 0)
            .dimensions(results)
            .suggestions(passed ? List.of() : List.of("请移除所有敏感信息后重新提交"))
            .build();
    }
}
```

**推荐开源组件**：
- **Microsoft Presidio** - PII 检测与脱敏
- **spaCy** - NER 模型
- **Hugging Face Transformers** - 预训练 NER 模型

---

#### 4.2.2 内容安全检测（P0）

**实现方式：第三方内容安全 API**

```java
@Component
public class ContentSafetyAuditor implements Auditor {
    
    private final ContentSafetyClient safetyClient; // 内容安全 API 客户端
    
    @Override
    public AuditResult audit(ResourceContent content, AuditContext context) {
        // 调用内容安全 API
        ContentSafetyResponse response = safetyClient.check(content.getText());
        
        boolean passed = response.getViolations().isEmpty();
        
        List<String> violations = response.getViolations().stream()
            .map(v -> v.getCategory() + ": " + v.getDescription())
            .collect(Collectors.toList());
        
        return AuditResult.builder()
            .passed(passed)
            .score(passed ? 100 : response.getScore())
            .dimensions(List.of(AuditDimensionResult.builder()
                .dimensionName("内容安全检测")
                .passed(passed)
                .score(response.getScore())
                .details(passed ? "内容安全" : "发现违规内容：" + String.join(", ", violations))
                .riskLevel(passed ? RiskLevel.LOW : RiskLevel.HIGH)
                .build()))
            .suggestions(passed ? List.of() : List.of("请修改违规内容后重新提交"))
            .build();
    }
}
```

**推荐服务**：
- **阿里云内容安全** - 支持中文，价格低
- **Azure Content Safety** - 多语言支持
- **Google Perspective API** - 毒性检测
- **开源方案**：Detoxify（Hugging Face）

---

#### 4.2.3 Prompt 注入检测（P0）

**实现方式：分类模型**

```java
@Component
public class PromptInjectionAuditor implements Auditor {
    
    private final ClassificationModel injectionModel;
    
    @Override
    public AuditResult audit(ResourceContent content, AuditContext context) {
        // 检测 Prompt 注入
        ClassificationResult result = injectionModel.predict(content.getText());
        
        boolean isInjection = result.getLabel().equals("INJECTION");
        double confidence = result.getConfidence();
        
        boolean passed = !isInjection || confidence < 0.7;
        
        return AuditResult.builder()
            .passed(passed)
            .score(passed ? 100 : (int)((1 - confidence) * 100))
            .dimensions(List.of(AuditDimensionResult.builder()
                .dimensionName("Prompt 注入检测")
                .passed(passed)
                .score(passed ? 100 : (int)((1 - confidence) * 100))
                .details(passed ? "未检测到注入攻击" : 
                    String.format("检测到 Prompt 注入攻击（置信度：%.2f）", confidence))
                .riskLevel(confidence > 0.9 ? RiskLevel.CRITICAL : RiskLevel.MEDIUM)
                .build()))
            .build();
    }
}
```

**推荐模型**：
- **PromptInject** - 专门检测 Prompt 注入
- **LLM Guard** - Hugging Face 上的 Prompt 安全工具包
- **自训练模型** - 基于 BERT/RoBERTa 微调

---

#### 4.2.4 代码安全检测（P1）

**实现方式：静态分析工具**

```java
@Component
public class CodeSecurityAuditor implements Auditor {
    
    private final SemgrepScanner semgrepScanner;
    private final DependencyChecker dependencyChecker;
    
    @Override
    public Set<ResourceType> getSupportedResourceTypes() {
        return Set.of(ResourceType.SKILL);
    }
    
    @Override
    public AuditResult audit(ResourceContent content, AuditContext context) {
        List<String> issues = new ArrayList<>();
        
        // 1. 代码扫描（Semgrep）
        List<SemgrepFinding> semgrepFindings = semgrepScanner.scan(content.getCode());
        issues.addAll(semgrepFindings.stream()
            .map(f -> f.getRuleId() + ": " + f.getMessage())
            .collect(Collectors.toList()));
        
        // 2. 依赖漏洞扫描
        List<Vulnerability> vulnerabilities = dependencyChecker.check(content.getDependencies());
        issues.addAll(vulnerabilities.stream()
            .map(v -> v.getPackage() + "@" + v.getVersion() + ": " + v.getDescription())
            .collect(Collectors.toList()));
        
        boolean passed = issues.isEmpty();
        
        return AuditResult.builder()
            .passed(passed)
            .score(passed ? 100 : Math.max(0, 100 - issues.size() * 10))
            .dimensions(List.of(AuditDimensionResult.builder()
                .dimensionName("代码安全检测")
                .passed(passed)
                .score(passed ? 100 : Math.max(0, 100 - issues.size() * 10))
                .details(passed ? "代码安全" : "发现安全问题：" + String.join(", ", issues))
                .riskLevel(issues.size() > 5 ? RiskLevel.HIGH : RiskLevel.MEDIUM)
                .build()))
            .build();
    }
}
```

**推荐工具**：
- **Semgrep** - 轻量级代码扫描
- **OWASP Dependency-Check** - 依赖漏洞扫描
- **Snyk** - 商业方案，功能全面
- **TruffleHog** - 密钥泄露检测

---

#### 4.2.5 合规检查（P1）

**实现方式：规则引擎**

```java
@Component
public class ComplianceAuditor implements Auditor {
    
    private final RuleEngine ruleEngine;
    
    @Override
    public AuditResult audit(ResourceContent content, AuditContext context) {
        List<RuleViolation> violations = ruleEngine.evaluate(content, context);
        
        boolean passed = violations.isEmpty();
        
        return AuditResult.builder()
            .passed(passed)
            .score(passed ? 100 : Math.max(0, 100 - violations.size() * 20))
            .dimensions(List.of(AuditDimensionResult.builder()
                .dimensionName("合规检查")
                .passed(passed)
                .score(passed ? 100 : Math.max(0, 100 - violations.size() * 20))
                .details(passed ? "符合合规要求" : 
                    "违反合规规则：" + violations.stream()
                        .map(v -> v.getRuleId() + ": " + v.getDescription())
                        .collect(Collectors.joining(", ")))
                .riskLevel(violations.size() > 3 ? RiskLevel.HIGH : RiskLevel.MEDIUM)
                .build()))
            .build();
    }
}
```

**推荐引擎**：
- **Drools** - Java 规则引擎
- **Easy Rules** - 轻量级规则引擎
- **自研规则引擎** - 基于 JSON 配置

---

#### 4.2.6 质量评估（P2）

**实现方式：大模型评分**

```java
@Component
public class QualityAuditor implements Auditor {
    
    private final LLMClient llmClient;
    
    @Override
    public AuditResult audit(ResourceContent content, AuditContext context) {
        // 使用大模型评估质量
        String prompt = String.format(
            "请评估以下 AI 资源的质量，从 0-100 打分，并给出评估理由：\n\n" +
            "资源类型：%s\n" +
            "资源内容：\n%s\n\n" +
            "评估维度：\n" +
            "1. 完整性：是否包含所有必要信息\n" +
            "2. 清晰性：描述是否清晰易懂\n" +
            "3. 可用性：是否可以直接使用\n" +
            "4. 安全性：是否存在安全隐患\n\n" +
            "请以 JSON 格式返回：{\"score\": 85, \"reason\": \"...\", \"suggestions\": [...]}",
            context.getResourceType(),
            content.getText()
        );
        
        QualityEvaluation evaluation = llmClient.evaluate(prompt, QualityEvaluation.class);
        
        boolean passed = evaluation.getScore() >= 60;
        
        return AuditResult.builder()
            .passed(passed)
            .score(evaluation.getScore())
            .dimensions(List.of(AuditDimensionResult.builder()
                .dimensionName("质量评估")
                .passed(passed)
                .score(evaluation.getScore())
                .details(evaluation.getReason())
                .riskLevel(passed ? RiskLevel.LOW : RiskLevel.MEDIUM)
                .build()))
            .suggestions(evaluation.getSuggestions())
            .build();
    }
}
```

**推荐模型**：
- **Qwen/GPT-4** - 通用评估
- **自训练评估模型** - 基于历史审核数据微调

---

## 五、技术架构

### 5.1 整体架构

```
┌─────────────────────────────────────────────────────────┐
│              Matrix Resource Service                     │
├─────────────────────────────────────────────────────────┤
│  API Layer                                               │
│  • 提交审核 • 审批 • 发布 • 查询                         │
├─────────────────────────────────────────────────────────┤
│  State Machine Layer                                     │
│  • 状态管理 • 状态变更日志 • 事件发布                    │
├─────────────────────────────────────────────────────────┤
│  Audit Chain Layer                                       │
│  • 审计任务管理 • 审计器链执行 • 结果聚合                │
├─────────────────────────────────────────────────────────┤
│  Auditor Implementations                                 │
│  • 敏感信息检测 • 内容安全 • Prompt 注入 • 代码安全      │
│  • 合规检查 • 质量评估                                   │
├─────────────────────────────────────────────────────────┤
│  External Services                                       │
│  • 内容安全 API • LLM • 代码扫描工具 • 规则引擎          │
└─────────────────────────────────────────────────────────┘
```

### 5.2 推荐技术栈

| 组件 | 推荐方案 | 备选方案 |
|------|---------|---------|
| **敏感信息检测** | Microsoft Presidio | spaCy + 自训练 NER |
| **内容安全** | 阿里云内容安全 | Azure Content Safety |
| **Prompt 注入** | PromptInject | 自训练 BERT 模型 |
| **代码扫描** | Semgrep | SonarQube |
| **规则引擎** | Drools | Easy Rules |
| **质量评估** | Qwen/GPT-4 | 自训练评估模型 |
| **工作流引擎** | Temporal | Camunda |

---

## 六、实施路线图

### Phase 1（0-2 周）：基础框架
- [ ] 状态机设计与实现
- [ ] 审计任务抽象接口
- [ ] 敏感信息检测（Presidio）

### Phase 2（2-4 周）：核心能力
- [ ] 内容安全检测（阿里云 API）
- [ ] Prompt 注入检测（PromptInject）
- [ ] 审计结果聚合与展示

### Phase 3（4-6 周）：增强能力
- [ ] 代码安全检测（Semgrep）
- [ ] 合规检查（Drools）
- [ ] 质量评估（LLM）

### Phase 4（6-8 周）：优化完善
- [ ] 审计任务异步化
- [ ] 审核规则可配置
- [ ] 审核数据分析与报表

---

## 七、总结

### 审核维度优先级

| 优先级 | 维度 | 实现方式 | 工作量 |
|--------|------|---------|--------|
| **P0** | 敏感信息检测 | Presidio + 正则 | 3 天 |
| **P0** | 内容安全 | 阿里云 API | 2 天 |
| **P0** | Prompt 注入 | PromptInject | 3 天 |
| **P1** | 代码安全 | Semgrep | 5 天 |
| **P1** | 合规检查 | Drools | 5 天 |
| **P2** | 质量评估 | LLM | 4 天 |

### 核心设计原则

1. **抽象接口** - 审计器可插拔，易于扩展
2. **链式执行** - 多个审计器顺序执行，结果聚合
3. **状态驱动** - 状态机管理资源生命周期
4. **事件驱动** - 状态变更发布事件，解耦业务逻辑

---

*方案生成时间：2026 年 3 月 11 日*  
*版本：v1.0*