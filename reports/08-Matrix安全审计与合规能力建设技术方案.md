# Matrix 安全审计与合规能力建设技术方案（v3.1）

## 一、数据模型设计

### 1.1 整体设计

| 表名 | 设计方式 | 用途 |
|------|---------|------|
| **resource** | 不扩展 | 资源元数据 |
| **resource_version** | 扩展字段 | 资源版本 + 审核状态 |
| **audit_task** | **新建表** | 独立管理审核任务 |

---

### 1.2 AuditTask 表（新建）

```java
@Entity
@Table(name = "audit_task",
       indexes = {
           @Index(columnList = "resource_id, version_id"),
           @Index(columnList = "status"),
           @Index(columnList = "created_at")
       })
public class AuditTask {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "task_no", unique = true, length = 50)
    private String taskNo;  // AUDIT-PRO-20260311-00001

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "resource_id", nullable = false)
    private Resource resource;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "version_id", nullable = false)
    private ResourceVersion version;

    @Enumerated(EnumType.STRING)
    private TaskType taskType;  // SUBMIT_REVIEW, MANUAL_REVIEW, RANDOM_CHECK

    @Enumerated(EnumType.STRING)
    private TaskStatus status;  // PENDING, IN_PROGRESS, COMPLETED, CANCELLED

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    private String auditDimensions;  // ["SENSITIVE_INFO", "CONTENT_SAFETY"]

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    private String result;  // 审核结果

    @Column(name = "score")
    private Integer score;  // 0-100

    @Column(name = "passed")
    private Boolean passed;

    @Column(name = "rejection_reason", columnDefinition = "TEXT")
    private String rejectionReason;

    @Column(name = "submitted_by", length = 100)
    private String submittedBy;

    @Column(name = "submitted_at")
    private LocalDateTime submittedAt;

    @Column(name = "audited_by", length = 100)
    private String auditedBy;

    @Column(name = "audited_at")
    private LocalDateTime auditedAt;

    @Column(name = "audit_duration_ms")
    private Long auditDurationMs;

    @CreationTimestamp
    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;
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

---

### 1.4 Resource 表

**不需要扩展字段** - 审核配置放在系统配置或配置文件，审核状态通过关联 ResourceVersion 或 AuditTask 查询。

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
    submitted_by VARCHAR(100),
    submitted_at TIMESTAMP,
    audited_by VARCHAR(100),
    audited_at TIMESTAMP,
    audit_duration_ms BIGINT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_task_resource_version ON audit_task(resource_id, version_id);
CREATE INDEX idx_audit_task_status ON audit_task(status);
CREATE INDEX idx_audit_task_submitted_at ON audit_task(submitted_at);

-- 2. ResourceVersion 表扩展
ALTER TABLE resource_version 
ADD COLUMN audit_status VARCHAR(50) DEFAULT 'PENDING_REVIEW',
ADD COLUMN audit_task_id BIGINT REFERENCES audit_task(id),
ADD COLUMN submitted_for_review_at TIMESTAMP,
ADD COLUMN submitted_for_review_by VARCHAR(100),
ADD COLUMN rejection_reason TEXT;

CREATE INDEX idx_resource_version_audit_status ON resource_version(audit_status);
CREATE INDEX idx_resource_version_audit_task_id ON resource_version(audit_task_id);

-- 3. Resource 表不需要扩展
```

---

## 三、系统配置

### 3.1 审核配置（application.yml）

```yaml
matrix:
  audit:
    enabled: true
    auto-audit: true  # 是否自动审核
    required-dimensions:  # 必选审核维度
      - SENSITIVE_INFO
      - CONTENT_SAFETY
      - PROMPT_INJECTION
    optional-dimensions:  # 可选审核维度
      - CODE_SECURITY
      - COMPLIANCE_CHECK
      - QUALITY_EVALUATION
    passing-score: 60  # 通过分数
    manual-review:
      enabled: true
      required-for-high-risk: true  # 高风险资源需要人工复审
```

### 3.2 资源级别配置（Resource metadata）

```json
{
  "audit": {
    "enabled": true,
    "auto_audit": false,  # 该资源需要人工审核
    "required_dimensions": ["SENSITIVE_INFO", "CONTENT_SAFETY"]
  }
}
```

---

## 四、审核维度与实现

| 维度 | 实现方式 | 推荐组件 |
|------|---------|---------|
| **敏感信息检测** | 正则 + NER | Microsoft Presidio |
| **内容安全** | 第三方 API | 阿里云内容安全 |
| **Prompt 注入** | 分类模型 | PromptInject |
| **代码安全** | 静态分析 | Semgrep |
| **合规检查** | 规则引擎 | Drools |
| **质量评估** | 大模型 | Qwen/GPT-4 |

---

## 五、核心服务实现

### 5.1 审核任务服务

```java
@Service
public class AuditTaskService {
    
    /**
     * 创建审核任务
     */
    @Transactional
    public AuditTask createTask(ResourceVersion version, String submitterId) {
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
    public void executeAudit(AuditTask task) {
        long startTime = System.currentTimeMillis();
        
        try {
            task.setStatus(AuditTask.TaskStatus.IN_PROGRESS);
            taskRepository.save(task);
            
            // 执行审计链
            AuditResult result = auditChainExecutor.executeAuditChain(...);
            
            // 更新任务结果
            task.setStatus(AuditTask.TaskStatus.COMPLETED);
            task.setScore(result.getScore());
            task.setPassed(result.isPassed());
            task.setAuditDurationMs(System.currentTimeMillis() - startTime);
            taskRepository.save(task);
            
            // 更新 ResourceVersion 状态
            updateVersionStatus(task, result);
            
        } catch (Exception e) {
            handleAuditError(task, e, startTime);
        }
    }
}
```

---

## 六、API 设计

### 6.1 提交审核

```http
POST /api/admin/resources/{resourceId}/versions/{versionId}/audit/submit

Response:
{
  "taskId": 789,
  "taskNo": "AUDIT-PRO-20260311-00001",
  "status": "IN_PROGRESS"
}
```

### 6.2 查询审核任务列表

```http
GET /api/admin/audit-tasks?status=PENDING&page=0&size=20

Response:
{
  "total": 15,
  "items": [
    {
      "taskId": 789,
      "taskNo": "AUDIT-PRO-20260311-00001",
      "resourceType": "prompt",
      "status": "PENDING",
      "submittedAt": "2026-03-11T13:00:00Z"
    }
  ]
}
```

### 6.3 人工审核

```http
POST /api/admin/audit-tasks/{taskId}/review

Request:
{
  "approved": true,
  "auditorId": "admin_456"
}
```

---

## 七、总结

### 数据模型

| 表名 | 设计方式 | 核心字段 |
|------|---------|---------|
| **resource** | 不扩展 | - |
| **resource_version** | 扩展 | audit_status, audit_task_id |
| **audit_task** | 新建 | 完整审核任务生命周期 |

### 核心优势

1. ✅ **审核任务独立管理** - 便于追踪、统计、审计
2. ✅ **Resource 表保持简洁** - 不增加无关字段
3. ✅ **完整的审核历史** - 每次审核都有独立任务记录
4. ✅ **支持人工审核** - 自动审核 + 人工复审双模式
5. ✅ **灵活配置** - 系统级配置 + 资源级配置

---

*方案生成时间：2026 年 3 月 11 日*  
*版本：v3.1*  
*修订：移除 Resource 表扩展字段*