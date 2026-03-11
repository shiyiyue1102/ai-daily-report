# Matrix 安全审计与合规能力建设技术方案（修订版）

## 一、方案概述

**修订说明**：基于 Matrix 现有数据模型（Resource + ResourceVersion），在现有表上扩展状态字段，不新增表。

---

## 二、现有数据模型分析

### 2.1 Resource 表（资源元数据）

**当前字段**：
```java
- id, workspace_id, type, key, name, description
- labels (jsonb), metadata (jsonb)
- current_formal_version, current_gray_version
- version_count, formal_count, gray_count, draft_count
- created_at, updated_at
```

### 2.2 ResourceVersion 表（资源版本）

**当前字段**：
```java
- id, resource_id, version
- status (DRAFT/GRAY/FORMAL/DEPRECATED/ARCHIVED)
- content, content_size, metadata (jsonb)
- checksum, gray_config, gray_priority
- published_at, expires_at, archived_at
- created_at, last_accessed_at
```

---

## 三、数据模型扩展方案

### 3.1 ResourceVersion 表扩展

**新增字段**：
```sql
ALTER TABLE resource_version ADD COLUMN audit_status VARCHAR(50);
-- PENDING_REVIEW(待审核), REVIEWING(审核中), APPROVED(已通过), REJECTED(已拒绝)

ALTER TABLE resource_version ADD COLUMN audit_task_id VARCHAR(100);
-- 关联的审核任务 ID

ALTER TABLE resource_version ADD COLUMN audit_result JSONB;
-- 审核结果详情

ALTER TABLE resource_version ADD COLUMN submitted_for_review_at TIMESTAMP;
-- 提交审核时间

ALTER TABLE resource_version ADD COLUMN submitted_for_review_by VARCHAR(100);
-- 提交审核人

ALTER TABLE resource_version ADD COLUMN audited_at TIMESTAMP;
-- 审核完成时间

ALTER TABLE resource_version ADD COLUMN audited_by VARCHAR(100);
-- 审核人

ALTER TABLE resource_version ADD COLUMN rejection_reason TEXT;
-- 拒绝原因
```

**状态流转**：
```
status 字段（生命周期状态）: DRAFT → GRAY → FORMAL → DEPRECATED → ARCHIVED
audit_status 字段（审核状态）: PENDING_REVIEW → REVIEWING → APPROVED/REJECTED
```

### 3.2 Resource 表扩展

**新增字段**：
```sql
ALTER TABLE resource ADD COLUMN audit_config JSONB;
-- 审核配置（是否启用审核、审核器配置等）

ALTER TABLE resource ADD COLUMN last_audit_status VARCHAR(50);
-- 最新版本的审核状态（冗余字段，便于查询）
```

---

## 四、状态机设计（基于现有模型）

### 4.1 完整状态流转

```
┌──────────────────────────────────────────────────────────────┐
│                   ResourceVersion 状态机                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  DRAFT ──提交审核──→ REVIEWING ──审核通过──→ PENDING_PUBLISH │
│    │                      │                                    │
│    │                      └──审核拒绝──→ REJECTED            │
│    │                      │                                    │
│    │                      └──审核异常──→ ERROR               │
│    │                                                         │
│    └──直接发布（免审）──→ PENDING_PUBLISH ──发布──→ FORMAL   │
│                                                              │
│  FORMAL ──下架──→ DEPRECATED ──归档──→ ARCHIVED             │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 状态定义

**status 字段（VersionStatus 枚举）**：
```java
public enum VersionStatus {
    DRAFT,           // 草稿
    REVIEWING,       // 审核中（新增）
    PENDING_PUBLISH, // 待发布（新增）
    GRAY,            // 灰度
    FORMAL,          // 正式
    REJECTED,        // 已拒绝（新增）
    ERROR,           // 异常（新增）
    DEPRECATED,      // 已弃用
    ARCHIVED         // 已归档
}
```

**audit_status 字段（AuditStatus 枚举）**：
```java
public enum AuditStatus {
    PENDING_REVIEW,  // 待审核
    REVIEWING,       // 审核中
    APPROVED,        // 已通过
    REJECTED,        // 已拒绝
    ERROR            // 异常
}
```

---

## 五、审计任务抽象接口（保持不变）

### 5.1 审计器接口

```java
public interface Auditor {
    Set<ResourceType> getSupportedResourceTypes();
    Set<AuditDimension> getSupportedDimensions();
    AuditResult audit(ResourceContent content, AuditContext context);
    int getPriority();
}
```

### 5.2 审计结果

```java
@Data
@Builder
public class AuditResult {
    private boolean passed;
    private Integer score;
    private List<AuditDimensionResult> dimensions;
    private String rejectionReason;
    private List<String> suggestions;
}
```

---

## 六、审核维度与实现方式（保持不变）

| 维度 | 实现方式 | 推荐组件 |
|------|---------|---------|
| **敏感信息检测** | 正则 + NER | Microsoft Presidio |
| **内容安全** | 第三方 API | 阿里云内容安全 |
| **Prompt 注入** | 分类模型 | PromptInject |
| **代码安全** | 静态分析 | Semgrep |
| **合规检查** | 规则引擎 | Drools |
| **质量评估** | 大模型 | Qwen/GPT-4 |

---

## 七、服务层实现

### 7.1 审核服务

```java
@Service
public class AuditService {
    
    private final ResourceVersionRepository versionRepository;
    private final AuditChainExecutor auditChainExecutor;
    private final ApplicationEventPublisher eventPublisher;
    
    /**
     * 提交审核：DRAFT → REVIEWING
     */
    @Transactional
    public ResourceVersion submitForReview(Long versionId, String userId) {
        ResourceVersion version = versionRepository.findById(versionId)
            .orElseThrow(() -> new ResourceNotFoundException(versionId));
        
        // 状态校验
        if (version.getStatus() != VersionStatus.DRAFT) {
            throw new IllegalStateException("只有草稿版本可以提交审核");
        }
        
        // 更新状态
        version.setStatus(VersionStatus.REVIEWING);
        version.setAuditStatus(AuditStatus.REVIEWING);
        version.setSubmittedForReviewAt(LocalDateTime.now());
        version.setSubmittedForReviewBy(userId);
        
        ResourceVersion saved = versionRepository.save(version);
        
        // 发布事件（异步触发审核任务）
        eventPublisher.publishEvent(new AuditSubmittedEvent(this, saved));
        
        return saved;
    }
    
    /**
     * 执行审核
     */
    @Async
    @Transactional
    public void executeAudit(ResourceVersion version) {
        try {
            // 构建审计上下文
            AuditContext context = AuditContext.builder()
                .resourceId(version.getResource().getId())
                .resourceType(version.getResource().getType())
                .workspaceId(version.getResource().getWorkspaceId())
                .submitterId(version.getSubmittedForReviewBy())
                .build();
            
            // 构建资源内容
            ResourceContent content = ResourceContent.builder()
                .text(version.getContent())
                .metadata(version.getMetadata())
                .build();
            
            // 执行审计链
            AuditResult result = auditChainExecutor.executeAuditChain(content, context);
            
            // 更新审核结果
            if (result.isPassed()) {
                approveAudit(version.getId(), result, "system");
            } else {
                rejectAudit(version.getId(), result.getRejectionReason(), "system");
            }
            
        } catch (Exception e) {
            handleAuditError(version.getId(), e);
        }
    }
    
    /**
     * 审核通过：REVIEWING → APPROVED → PENDING_PUBLISH
     */
    @Transactional
    public ResourceVersion approveAudit(Long versionId, AuditResult result, String auditorId) {
        ResourceVersion version = versionRepository.findById(versionId)
            .orElseThrow(() -> new ResourceNotFoundException(versionId));
        
        version.setStatus(VersionStatus.PENDING_PUBLISH);
        version.setAuditStatus(AuditStatus.APPROVED);
        version.setAuditResult(toJson(result));
        version.setAuditedAt(LocalDateTime.now());
        version.setAuditedBy(auditorId);
        
        ResourceVersion saved = versionRepository.save(version);
        
        eventPublisher.publishEvent(new AuditApprovedEvent(this, saved, result));
        
        return saved;
    }
    
    /**
     * 审核拒绝：REVIEWING → REJECTED
     */
    @Transactional
    public ResourceVersion rejectAudit(Long versionId, String reason, String auditorId) {
        ResourceVersion version = versionRepository.findById(versionId)
            .orElseThrow(() -> new ResourceNotFoundException(versionId));
        
        version.setStatus(VersionStatus.REJECTED);
        version.setAuditStatus(AuditStatus.REJECTED);
        version.setRejectionReason(reason);
        version.setAuditedAt(LocalDateTime.now());
        version.setAuditedBy(auditorId);
        
        ResourceVersion saved = versionRepository.save(version);
        
        eventPublisher.publishEvent(new AuditRejectedEvent(this, saved, reason));
        
        return saved;
    }
    
    /**
     * 发布：PENDING_PUBLISH → FORMAL
     */
    @Transactional
    public ResourceVersion publish(Long versionId, String userId) {
        ResourceVersion version = versionRepository.findById(versionId)
            .orElseThrow(() -> new ResourceNotFoundException(versionId));
        
        if (version.getStatus() != VersionStatus.PENDING_PUBLISH) {
            throw new IllegalStateException("只有待发布状态可以发布");
        }
        
        // 检查审核状态
        if (version.getAuditStatus() != AuditStatus.APPROVED) {
            throw new IllegalStateException("只有通过审核的版本可以发布");
        }
        
        version.setStatus(VersionStatus.FORMAL);
        version.setPublishedAt(LocalDateTime.now());
        
        ResourceVersion saved = versionRepository.save(version);
        
        // 更新 Resource 表的版本统计
        Resource resource = version.getResource();
        resource.setCurrentFormalVersion(version.getVersion());
        resource.setFormalCount(resource.getFormalCount() + 1);
        resource.setLastAuditStatus(AuditStatus.APPROVED.name());
        
        eventPublisher.publishEvent(new ResourcePublishedEvent(this, saved));
        
        return saved;
    }
}
```

---

## 八、API 设计

### 8.1 提交审核

```http
POST /api/admin/resources/{resourceId}/versions/{versionId}/audit/submit

Request:
{
  "submitterId": "user_123"
}

Response:
{
  "code": 200,
  "data": {
    "versionId": 456,
    "status": "REVIEWING",
    "auditStatus": "REVIEWING",
    "submittedAt": "2026-03-11T13:00:00Z"
  }
}
```

### 8.2 审核通过

```http
POST /api/admin/resources/{resourceId}/versions/{versionId}/audit/approve

Request:
{
  "auditorId": "admin_456",
  "result": {
    "passed": true,
    "score": 95,
    "dimensions": [...]
  }
}

Response:
{
  "code": 200,
  "data": {
    "versionId": 456,
    "status": "PENDING_PUBLISH",
    "auditStatus": "APPROVED"
  }
}
```

### 8.3 审核拒绝

```http
POST /api/admin/resources/{resourceId}/versions/{versionId}/audit/reject

Request:
{
  "auditorId": "admin_456",
  "reason": "发现敏感信息：API Key"
}

Response:
{
  "code": 200,
  "data": {
    "versionId": 456,
    "status": "REJECTED",
    "auditStatus": "REJECTED",
    "rejectionReason": "发现敏感信息：API Key"
  }
}
```

### 8.4 发布

```http
POST /api/admin/resources/{resourceId}/versions/{versionId}/publish

Request:
{
  "publisherId": "admin_456"
}

Response:
{
  "code": 200,
  "data": {
    "versionId": 456,
    "status": "FORMAL",
    "publishedAt": "2026-03-11T14:00:00Z"
  }
}
```

---

## 九、数据库迁移脚本

```sql
-- 1. ResourceVersion 表扩展
ALTER TABLE resource_version 
ADD COLUMN audit_status VARCHAR(50) DEFAULT 'PENDING_REVIEW',
ADD COLUMN audit_task_id VARCHAR(100),
ADD COLUMN audit_result JSONB,
ADD COLUMN submitted_for_review_at TIMESTAMP,
ADD COLUMN submitted_for_review_by VARCHAR(100),
ADD COLUMN audited_at TIMESTAMP,
ADD COLUMN audited_by VARCHAR(100),
ADD COLUMN rejection_reason TEXT;

-- 2. Resource 表扩展
ALTER TABLE resource 
ADD COLUMN audit_config JSONB DEFAULT '{}',
ADD COLUMN last_audit_status VARCHAR(50);

-- 3. 更新 VersionStatus 枚举（PostgreSQL）
ALTER TYPE version_status ADD VALUE IF NOT EXISTS 'REVIEWING';
ALTER TYPE version_status ADD VALUE IF NOT EXISTS 'PENDING_PUBLISH';
ALTER TYPE version_status ADD VALUE IF NOT EXISTS 'REJECTED';
ALTER TYPE version_status ADD VALUE IF NOT EXISTS 'ERROR';

-- 4. 创建索引（加速查询）
CREATE INDEX idx_resource_version_audit_status ON resource_version(audit_status);
CREATE INDEX idx_resource_version_submitted_at ON resource_version(submitted_for_review_at);
CREATE INDEX idx_resource_last_audit_status ON resource(last_audit_status);
```

---

## 十、实施路线图（保持不变）

### Phase 1（0-2 周）：基础框架
- [ ] 数据库迁移（扩展字段）
- [ ] 状态机实现
- [ ] 敏感信息检测（Presidio）

### Phase 2（2-4 周）：核心能力
- [ ] 内容安全检测（阿里云 API）
- [ ] Prompt 注入检测（PromptInject）
- [ ] 审核 API 开发

### Phase 3（4-6 周）：增强能力
- [ ] 代码安全检测（Semgrep）
- [ ] 合规检查（Drools）
- [ ] 审核管理后台

### Phase 4（6-8 周）：优化完善
- [ ] 审核任务异步化
- [ ] 审核规则可配置
- [ ] 审核数据分析与报表

---

## 十一、总结

### 修订要点

1. **不新增表** - 在现有 ResourceVersion 表上扩展字段
2. **双状态设计** - status（生命周期）+ audit_status（审核状态）
3. **向后兼容** - 不影响现有功能，审核为可选能力
4. **灵活配置** - 可通过 audit_config 配置是否启用审核

### 核心优势

| 设计决策 | 原方案 | 修订方案 |
|---------|--------|---------|
| 表结构 | 新增 resource_states 表 | 扩展现有表 |
| 复杂度 | 高（多表关联） | 低（单表扩展） |
| 迁移成本 | 高 | 低 |
| 查询性能 | 中（JOIN） | 高（单表） |

---

*方案生成时间：2026 年 3 月 11 日*  
*版本：v2.0（修订版）*  
*修订原因：基于 Matrix 现有数据模型优化设计*