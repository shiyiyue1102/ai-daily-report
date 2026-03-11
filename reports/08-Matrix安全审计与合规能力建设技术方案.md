# Matrix 安全审计与合规能力建设技术方案（v3.0）

## 一、数据模型设计

### 1.1 整体设计

| 表名 | 用途 | 设计方式 |
|------|------|---------|
| **resource** | 资源元数据 | 扩展字段（audit_config, last_audit_status） |
| **resource_version** | 资源版本 | 扩展字段（审核状态、关联任务 ID） |
| **audit_task** | 审核任务 | **新建表**（独立管理审核任务） |

---

### 1.2 AuditTask 表（新建）

```java
/**
 * 审核任务实体
 */
@Entity
@Table(name = "audit_task",
       indexes = {
           @Index(columnList = "resource_id, version_id"),
           @Index(columnList = "status"),
           @Index(columnList = "created_at")
       })
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class AuditTask {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    /**
     * 任务编号（业务可见）
     */
    @Column(name = "task_no", unique = true, length = 50)
    private String taskNo;

    /**
     * 关联资源
     */
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "resource_id", nullable = false)
    private Resource resource;

    /**
     * 关联版本
     */
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "version_id", nullable = false)
    private ResourceVersion version;

    /**
     * 任务类型
     */
    @Column(nullable = false, length = 50)
    @Enumerated(EnumType.STRING)
    private TaskType taskType;

    /**
     * 任务状态
     */
    @Column(nullable = false, length = 50)
    @Enumerated(EnumType.STRING)
    private TaskStatus status;

    /**
     * 审核维度（JSON 数组）
     * ["SENSITIVE_INFO", "CONTENT_SAFETY", "PROMPT_INJECTION"]
     */
    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    private String auditDimensions;

    /**
     * 审核结果
     */
    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    private String result;

    /**
     * 审核得分（0-100）
     */
    @Column(name = "score")
    private Integer score;

    /**
     * 是否通过
     */
    @Column(name = "passed")
    private Boolean passed;

    /**
     * 拒绝原因
     */
    @Column(name = "rejection_reason", columnDefinition = "TEXT")
    private String rejectionReason;

    /**
     * 建议修复项（JSON 数组）
     */
    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    private String suggestions;

    /**
     * 提交人
     */
    @Column(name = "submitted_by", length = 100)
    private String submittedBy;

    /**
     * 提交时间
     */
    @Column(name = "submitted_at")
    private LocalDateTime submittedAt;

    /**
     * 审核人
     */
    @Column(name = "audited_by", length = 100)
    private String auditedBy;

    /**
     * 审核完成时间
     */
    @Column(name = "audited_at")
    private LocalDateTime auditedAt;

    /**
     * 审核耗时（毫秒）
     */
    @Column(name = "audit_duration_ms")
    private Long auditDurationMs;

    /**
     * 审计日志（JSON 数组，记录每个审计器的执行结果）
     */
    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    private String auditLogs;

    /**
     * 元数据
     */
    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    private String metadata;

    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;

    /**
     * 任务类型枚举
     */
    public enum TaskType {
        SUBMIT_REVIEW,      // 提交审核
        MANUAL_REVIEW,      // 人工复审
        RANDOM_CHECK,       // 随机抽检
        RE_AUDIT            // 重新审核
    }

    /**
     * 任务状态枚举
     */
    public enum TaskStatus {
        PENDING,            // 待处理
        IN_PROGRESS,        // 审核中
        COMPLETED,          // 已完成
        CANCELLED           // 已取消
    }
}
```

---

### 1.3 ResourceVersion 表扩展

```java
// 新增字段
@Column(name = "audit_status", length = 50)
@Enumerated(EnumType.STRING)
private AuditStatus auditStatus;  // PENDING_REVIEW, REVIEWING, APPROVED, REJECTED

@Column(name = "audit_task_id")
private Long auditTaskId;  // 关联审核任务 ID

@Column(name = "submitted_for_review_at")
private LocalDateTime submittedForReviewAt;

@Column(name = "submitted_for_review_by", length = 100)
private String submittedForReviewBy;

@Column(name = "rejection_reason", columnDefinition = "TEXT")
private String rejectionReason;
```

**审核状态枚举**：
```java
public enum AuditStatus {
    PENDING_REVIEW,   // 待审核
    REVIEWING,        // 审核中
    APPROVED,         // 已通过
    REJECTED          // 已拒绝
}
```

---

### 1.4 Resource 表扩展

```java
// 新增字段
@JdbcTypeCode(SqlTypes.JSON)
@Column(name = "audit_config", columnDefinition = "jsonb")
private String auditConfig;  // 审核配置

@Column(name = "last_audit_status", length = 50)
private String lastAuditStatus;  // 最新版本的审核状态（冗余字段）
```

**审核配置示例**：
```json
{
  "enabled": true,
  "auto_audit": false,
  "required_dimensions": ["SENSITIVE_INFO", "CONTENT_SAFETY"],
  "optional_dimensions": ["QUALITY_EVALUATION"],
  "passing_score": 60,
  "manual_review_required": true
}
```

---

## 二、数据库迁移脚本

```sql
-- 1. 创建 audit_task 表
CREATE TABLE audit_task (
    id BIGSERIAL PRIMARY KEY,
    task_no VARCHAR(50) UNIQUE NOT NULL,
    resource_id BIGINT NOT NULL REFERENCES resource(id),
    version_id BIGINT NOT NULL REFERENCES resource_version(id),
    task_type VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL,
    audit_dimensions JSONB,
    result JSONB,
    score INTEGER,
    passed BOOLEAN,
    rejection_reason TEXT,
    suggestions JSONB,
    submitted_by VARCHAR(100),
    submitted_at TIMESTAMP,
    audited_by VARCHAR(100),
    audited_at TIMESTAMP,
    audit_duration_ms BIGINT,
    audit_logs JSONB,
    metadata JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- 创建索引
CREATE INDEX idx_audit_task_resource_version ON audit_task(resource_id, version_id);
CREATE INDEX idx_audit_task_status ON audit_task(status);
CREATE INDEX idx_audit_task_created_at ON audit_task(created_at);
CREATE INDEX idx_audit_task_submitted_at ON audit_task(submitted_at);

-- 2. ResourceVersion 表扩展
ALTER TABLE resource_version 
ADD COLUMN audit_status VARCHAR(50) DEFAULT 'PENDING_REVIEW',
ADD COLUMN audit_task_id BIGINT REFERENCES audit_task(id),
ADD COLUMN submitted_for_review_at TIMESTAMP,
ADD COLUMN submitted_for_review_by VARCHAR(100),
ADD COLUMN rejection_reason TEXT;

-- 创建索引
CREATE INDEX idx_resource_version_audit_status ON resource_version(audit_status);
CREATE INDEX idx_resource_version_audit_task_id ON resource_version(audit_task_id);

-- 3. Resource 表扩展
ALTER TABLE resource 
ADD COLUMN audit_config JSONB DEFAULT '{}',
ADD COLUMN last_audit_status VARCHAR(50);

-- 创建索引
CREATE INDEX idx_resource_last_audit_status ON resource(last_audit_status);

-- 4. 创建审计日志表（可选，用于详细追踪）
CREATE TABLE audit_task_log (
    id BIGSERIAL PRIMARY KEY,
    task_id BIGINT NOT NULL REFERENCES audit_task(id),
    auditor_name VARCHAR(100) NOT NULL,
    auditor_type VARCHAR(50) NOT NULL,  -- AUTO, MANUAL
    dimension VARCHAR(50) NOT NULL,
    passed BOOLEAN NOT NULL,
    score INTEGER,
    details TEXT,
    risk_level VARCHAR(20),
    duration_ms BIGINT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_task_log_task_id ON audit_task_log(task_id);
```

---

## 三、服务层实现

### 3.1 审核任务服务

```java
@Service
public class AuditTaskService {
    
    private final AuditTaskRepository taskRepository;
    private final ResourceVersionRepository versionRepository;
    private final AuditChainExecutor auditChainExecutor;
    private final ApplicationEventPublisher eventPublisher;
    
    /**
     * 创建审核任务
     */
    @Transactional
    public AuditTask createTask(ResourceVersion version, String submitterId) {
        // 生成任务编号
        String taskNo = generateTaskNo(version.getResource().getType());
        
        AuditTask task = AuditTask.builder()
            .taskNo(taskNo)
            .resource(version.getResource())
            .version(version)
            .taskType(AuditTask.TaskType.SUBMIT_REVIEW)
            .status(AuditTask.TaskStatus.PENDING)
            .submittedBy(submitterId)
            .submittedAt(LocalDateTime.now())
            .build();
        
        AuditTask saved = taskRepository.save(task);
        
        // 更新 ResourceVersion 状态
        version.setAuditStatus(AuditStatus.REVIEWING);
        version.setAuditTaskId(saved.getId());
        version.setSubmittedForReviewAt(LocalDateTime.now());
        version.setSubmittedForReviewBy(submitterId);
        versionRepository.save(version);
        
        // 发布事件（异步执行审核）
        eventPublisher.publishEvent(new AuditTaskCreatedEvent(this, saved));
        
        return saved;
    }
    
    /**
     * 执行审核（异步）
     */
    @Async
    @Transactional
    public void executeAudit(AuditTask task) {
        long startTime = System.currentTimeMillis();
        
        try {
            // 更新任务状态
            task.setStatus(AuditTask.TaskStatus.IN_PROGRESS);
            taskRepository.save(task);
            
            // 构建审计上下文
            AuditContext context = AuditContext.builder()
                .resourceId(task.getResource().getId())
                .resourceType(task.getResource().getType())
                .workspaceId(task.getResource().getWorkspaceId())
                .submitterId(task.getSubmittedBy())
                .build();
            
            // 构建资源内容
            ResourceContent content = ResourceContent.builder()
                .text(task.getVersion().getContent())
                .metadata(task.getVersion().getMetadata())
                .build();
            
            // 执行审计链
            AuditResult result = auditChainExecutor.executeAuditChain(content, context);
            
            // 更新任务结果
            task.setStatus(AuditTask.TaskStatus.COMPLETED);
            task.setResult(toJson(result));
            task.setScore(result.getScore());
            task.setPassed(result.isPassed());
            task.setRejectionReason(result.isPassed() ? null : result.getRejectionReason());
            task.setSuggestions(toJson(result.getSuggestions()));
            task.setAuditedAt(LocalDateTime.now());
            task.setAuditDurationMs(System.currentTimeMillis() - startTime);
            
            taskRepository.save(task);
            
            // 更新 ResourceVersion 状态
            updateVersionStatus(task, result);
            
            // 发布事件
            if (result.isPassed()) {
                eventPublisher.publishEvent(new AuditApprovedEvent(this, task, result));
            } else {
                eventPublisher.publishEvent(new AuditRejectedEvent(this, task, result));
            }
            
        } catch (Exception e) {
            handleAuditError(task, e, startTime);
        }
    }
    
    /**
     * 人工审核
     */
    @Transactional
    public AuditTask manualReview(Long taskId, boolean approved, String reason, String auditorId) {
        AuditTask task = taskRepository.findById(taskId)
            .orElseThrow(() -> new AuditTaskNotFoundException(taskId));
        
        if (task.getStatus() != AuditTask.TaskStatus.IN_PROGRESS) {
            throw new IllegalStateException("任务状态不正确");
        }
        
        task.setStatus(AuditTask.TaskStatus.COMPLETED);
        task.setPassed(approved);
        task.setScore(approved ? 100 : 0);
        task.setRejectionReason(approved ? null : reason);
        task.setAuditedBy(auditorId);
        task.setAuditedAt(LocalDateTime.now());
        task.setAuditDurationMs(
            ChronoUnit.MILLIS.between(task.getSubmittedAt(), LocalDateTime.now())
        );
        
        taskRepository.save(task);
        
        // 更新 ResourceVersion 状态
        ResourceVersion version = task.getVersion();
        if (approved) {
            version.setAuditStatus(AuditStatus.APPROVED);
        } else {
            version.setAuditStatus(AuditStatus.REJECTED);
            version.setRejectionReason(reason);
        }
        versionRepository.save(version);
        
        return task;
    }
    
    /**
     * 更新版本状态
     */
    private void updateVersionStatus(AuditTask task, AuditResult result) {
        ResourceVersion version = task.getVersion();
        
        if (result.isPassed()) {
            version.setAuditStatus(AuditStatus.APPROVED);
        } else {
            version.setAuditStatus(AuditStatus.REJECTED);
            version.setRejectionReason(result.getRejectionReason());
        }
        
        versionRepository.save(version);
    }
    
    /**
     * 生成任务编号
     */
    private String generateTaskNo(ResourceType resourceType) {
        String prefix = resourceType.name().substring(0, 3).toUpperCase();
        String date = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));
        String seq = String.format("%05d", getNextSeq());
        return String.format("AUDIT-%s-%s-%s", prefix, date, seq);
    }
    
    private Long getNextSeq() {
        // 使用 Redis 或数据库序列生成
        return redisTemplate.opsForValue().increment("audit_task_seq");
    }
}
```

---

### 3.2 审核查询服务

```java
@Service
public class AuditQueryService {
    
    private final AuditTaskRepository taskRepository;
    
    /**
     * 分页查询审核任务
     */
    public Page<AuditTaskVO> queryTasks(AuditTaskQuery query, Pageable pageable) {
        return taskRepository.findAll(buildSpecification(query), pageable)
            .map(this::toVO);
    }
    
    /**
     * 获取审核任务详情
     */
    public AuditTaskDetailVO getTaskDetail(Long taskId) {
        AuditTask task = taskRepository.findById(taskId)
            .orElseThrow(() -> new AuditTaskNotFoundException(taskId));
        
        return toDetailVO(task);
    }
    
    /**
     * 统计审核数据
     */
    public AuditStatisticsVO getStatistics(LocalDateTime from, LocalDateTime to) {
        return taskRepository.findStatistics(from, to);
    }
    
    /**
     * 查询待审核任务列表
     */
    public List<AuditTaskVO> getPendingTasks(String workspaceId) {
        return taskRepository.findByStatusAndWorkspace(
                AuditTask.TaskStatus.PENDING, workspaceId)
            .stream()
            .map(this::toVO)
            .collect(Collectors.toList());
    }
}
```

---

## 四、API 设计

### 4.1 提交审核

```http
POST /api/admin/resources/{resourceId}/versions/{versionId}/audit/submit

Request:
{
  "submitterId": "user_123",
  "auditDimensions": ["SENSITIVE_INFO", "CONTENT_SAFETY"]
}

Response:
{
  "code": 200,
  "data": {
    "taskId": 789,
    "taskNo": "AUDIT-PRO-20260311-00001",
    "status": "IN_PROGRESS",
    "submittedAt": "2026-03-11T13:00:00Z"
  }
}
```

### 4.2 查询审核任务列表

```http
GET /api/admin/audit-tasks?status=PENDING&resourceType=prompt&page=0&size=20

Response:
{
  "code": 200,
  "data": {
    "total": 15,
    "items": [
      {
        "taskId": 789,
        "taskNo": "AUDIT-PRO-20260311-00001",
        "resourceType": "prompt",
        "resourceKey": "customer-service-prompt",
        "version": "1.0.0",
        "status": "PENDING",
        "submittedAt": "2026-03-11T13:00:00Z",
        "submittedBy": "user_123"
      }
    ]
  }
}
```

### 4.3 获取审核任务详情

```http
GET /api/admin/audit-tasks/{taskId}

Response:
{
  "code": 200,
  "data": {
    "taskId": 789,
    "taskNo": "AUDIT-PRO-20260311-00001",
    "resource": {...},
    "version": {...},
    "status": "COMPLETED",
    "score": 85,
    "passed": true,
    "dimensions": [
      {
        "name": "敏感信息检测",
        "passed": true,
        "score": 100
      },
      {
        "name": "内容安全",
        "passed": true,
        "score": 90
      }
    ],
    "auditLogs": [...],
    "auditDurationMs": 1523
  }
}
```

### 4.4 人工审核

```http
POST /api/admin/audit-tasks/{taskId}/review

Request:
{
  "approved": true,
  "reason": null,
  "auditorId": "admin_456"
}

Response:
{
  "code": 200,
  "data": {
    "taskId": 789,
    "status": "COMPLETED",
    "passed": true,
    "auditedAt": "2026-03-11T14:00:00Z"
  }
}
```

### 4.5 审核统计

```http
GET /api/admin/audit-tasks/statistics?from=2026-03-01&to=2026-03-11

Response:
{
  "code": 200,
  "data": {
    "totalTasks": 150,
    "completedTasks": 140,
    "pendingTasks": 10,
    "passedTasks": 130,
    "rejectedTasks": 10,
    "avgScore": 82.5,
    "avgDurationMs": 1523
  }
}
```

---

## 五、审核维度与实现（保持不变）

| 维度 | 实现方式 | 推荐组件 |
|------|---------|---------|
| **敏感信息检测** | 正则 + NER | Microsoft Presidio |
| **内容安全** | 第三方 API | 阿里云内容安全 |
| **Prompt 注入** | 分类模型 | PromptInject |
| **代码安全** | 静态分析 | Semgrep |
| **合规检查** | 规则引擎 | Drools |
| **质量评估** | 大模型 | Qwen/GPT-4 |

---

## 六、总结

### 数据模型设计

| 表名 | 设计方式 | 核心字段 |
|------|---------|---------|
| **resource** | 扩展字段 | audit_config, last_audit_status |
| **resource_version** | 扩展字段 | audit_status, audit_task_id, rejection_reason |
| **audit_task** | **新建表** | 完整记录审核任务全生命周期 |
| **audit_task_log** | 可选 | 详细审计日志（每个审计器执行结果） |

### 核心优势

1. **审核任务独立管理** - 便于追踪、统计、审计
2. **不破坏现有模型** - Resource 和 ResourceVersion 仅扩展字段
3. **完整的审核历史** - 每次审核都有独立任务记录
4. **支持人工审核** - 自动审核 + 人工复审双模式
5. **灵活配置** - 可配置审核维度、通过分数、是否免审

### 实施优先级

| 阶段 | 工作内容 | 工期 |
|------|---------|------|
| **Phase 1** | 数据库迁移 + 基础框架 | 2 周 |
| **Phase 2** | 敏感信息 + 内容安全检测 | 2 周 |
| **Phase 3** | Prompt 注入 + 代码安全 | 2 周 |
| **Phase 4** | 审核管理后台 + 统计报表 | 2 周 |

---

*方案生成时间：2026 年 3 月 11 日*  
*版本：v3.0*  
*核心改进：新建 AuditTask 表独立管理审核任务*