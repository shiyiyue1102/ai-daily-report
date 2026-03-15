# Skill 大文件存储技术实现方案

## 一、方案概述

### 1.1 存储策略

**混合存储方案**：
- **<1MB**：存 PostgreSQL（content_inline 字段）
- **>1MB**：存对象存储 OSS/S3/MinIO（content_ref 字段）

**核心优势**：
- ✅ 小文件快速访问（DB 直接读取）
- ✅ 大文件低成本存储（OSS 成本低）
- ✅ 透明切换（应用层无感知）

---

## 二、架构设计

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│              Skill 存储架构                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  应用层 (Matrix Server)                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  SkillService                                             │  │
│  │    └─ saveSkill() / getSkill()                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            │                                    │
│                            ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  ContentStorageService (存储抽象层)                        │  │
│  │    ├─ save(content) → 自动判断存储位置                    │  │
│  │    └─ get(ref) → 自动读取内容                            │  │
│  └──────────────────────────────────────────────────────────┘  │
│           │                              │                      │
│           ▼                              ▼                      │
│  ┌─────────────────┐          ┌─────────────────┐             │
│  │  PostgreSQL     │          │  Object Storage │             │
│  │  (元数据 + 小文件) │          │  (大文件)        │             │
│  │  - content_inline│          │  - S3/MinIO/OSS │             │
│  │  - content_ref   │          │  - CDN 加速      │             │
│  └─────────────────┘          └─────────────────┘             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 存储抽象层设计

**核心接口**：
```java
public interface ContentStorage {
    
    /**
     * 保存内容
     * @param content 内容字符串
     * @param context 上下文信息（用于生成存储 key）
     * @return 存储引用（DB 路径或 OSS 路径）
     */
    ContentRef save(String content, StorageContext context);
    
    /**
     * 读取内容
     * @param ref 存储引用
     * @return 内容字符串
     */
    String get(ContentRef ref);
    
    /**
     * 删除内容
     * @param ref 存储引用
     */
    void delete(ContentRef ref);
    
    /**
     * 检查内容是否存在
     * @param ref 存储引用
     * @return 是否存在
     */
    boolean exists(ContentRef ref);
}
```

**存储引用类**：
```java
@Data
@Builder
@AllArgsConstructor
public class ContentRef {
    
    /**
     * 存储类型
     */
    private StorageType type; // DATABASE 或 OBJECT_STORAGE
    
    /**
     * 存储路径
     * - DB: null
     * - OSS: "oss://bucket/path/to/file"
     */
    private String path;
    
    /**
     * 内容大小（字节）
     */
    private Long size;
    
    /**
     * SHA-256 校验和
     */
    private String checksum;
    
    /**
     * 内容类型
     */
    private String contentType; // "text/plain", "application/json" 等
    
    /**
     * 创建时间
     */
    private LocalDateTime createdAt;
    
    /**
     * 是否为内联存储（小文件）
     */
    public boolean isInline() {
        return type == StorageType.DATABASE;
    }
    
    /**
     * 是否为外部存储（大文件）
     */
    public boolean isExternal() {
        return type == StorageType.OBJECT_STORAGE;
    }
}

public enum StorageType {
    DATABASE,       // 数据库存储（小文件）
    OBJECT_STORAGE  // 对象存储（大文件）
}

@Data
@Builder
public class StorageContext {
    
    /**
     * 工作空间 ID
     */
    private String workspaceId;
    
    /**
     * 资源类型
     */
    private String resourceType; // "skill", "prompt", "mcp"
    
    /**
     * 资源 ID
     */
    private Long resourceId;
    
    /**
     * 版本号
     */
    private String version;
    
    /**
     * 生成存储 key
     */
    public String generateKey() {
        return String.format("%s/%s/%d/%s",
            resourceType,
            workspaceId,
            resourceId,
            version
        );
    }
}
```

---

## 三、核心实现

### 3.1 混合存储实现

```java
@Service
public class HybridContentStorage implements ContentStorage {
    
    /**
     * 大小阈值：1MB
     */
    private static final long SIZE_THRESHOLD = 1 * 1024 * 1024;
    
    @Autowired
    private DatabaseStorage databaseStorage;
    
    @Autowired
    private ObjectStorage objectStorage;
    
    @Override
    public ContentRef save(String content, StorageContext context) {
        byte[] contentBytes = content.getBytes(StandardCharsets.UTF_8);
        long size = contentBytes.length;
        
        if (size < SIZE_THRESHOLD) {
            // 小文件：存 DB
            log.debug("Saving small file (<1MB) to database: size={}", size);
            return databaseStorage.save(contentBytes, context);
        } else {
            // 大文件：存 OSS
            log.info("Saving large file (>1MB) to object storage: size={}, key={}", 
                size, context.generateKey());
            return objectStorage.save(contentBytes, context);
        }
    }
    
    @Override
    public String get(ContentRef ref) {
        if (ref.isInline()) {
            // 从 DB 读取
            return databaseStorage.get(ref);
        } else {
            // 从 OSS 读取
            return objectStorage.get(ref);
        }
    }
    
    @Override
    public void delete(ContentRef ref) {
        if (ref.isInline()) {
            databaseStorage.delete(ref);
        } else {
            objectStorage.delete(ref);
        }
    }
    
    @Override
    public boolean exists(ContentRef ref) {
        if (ref.isInline()) {
            return databaseStorage.exists(ref);
        } else {
            return objectStorage.exists(ref);
        }
    }
}
```

### 3.2 数据库存储实现

```java
@Service
public class DatabaseStorage implements ContentStorage {
    
    @Autowired
    private ResourceVersionRepository repository;
    
    @Override
    public ContentRef save(byte[] contentBytes, StorageContext context) {
        String content = new String(contentBytes, StandardCharsets.UTF_8);
        String checksum = calculateChecksum(contentBytes);
        
        return ContentRef.builder()
            .type(StorageType.DATABASE)
            .path(null) // DB 存储不需要 path
            .size((long) contentBytes.length)
            .checksum(checksum)
            .contentType("text/plain")
            .createdAt(LocalDateTime.now())
            .build();
    }
    
    public String get(ContentRef ref) {
        // 实际从 ResourceVersion 实体中读取 content_inline 字段
        // 这里只是示例
        return repository.findById(ref.getResourceId())
            .map(ResourceVersion::getContentInline)
            .orElseThrow(() -> new ContentNotFoundException(ref));
    }
    
    // 其他方法实现...
}
```

### 3.3 对象存储实现（MinIO 示例）

```java
@Service
public class MinioObjectStorage implements ObjectStorage {
    
    @Autowired
    private MinioClient minioClient;
    
    @Value("${storage.minio.bucket:matrix-skills}")
    private String bucket;
    
    @Override
    public ContentRef save(byte[] contentBytes, StorageContext context) {
        try {
            String key = context.generateKey();
            
            // 上传到 MinIO
            minioClient.putObject(
                PutObjectArgs.builder()
                    .bucket(bucket)
                    .object(key)
                    .stream(new ByteArrayInputStream(contentBytes), contentBytes.length, -1)
                    .contentType("text/plain")
                    .build()
            );
            
            log.info("Uploaded to MinIO: bucket={}, key={}, size={}", 
                bucket, key, contentBytes.length);
            
            String checksum = calculateChecksum(contentBytes);
            
            return ContentRef.builder()
                .type(StorageType.OBJECT_STORAGE)
                .path("oss://" + bucket + "/" + key)
                .size((long) contentBytes.length)
                .checksum(checksum)
                .contentType("text/plain")
                .createdAt(LocalDateTime.now())
                .build();
                
        } catch (Exception e) {
            log.error("Failed to upload to MinIO: key={}", context.generateKey(), e);
            throw new StorageException("Failed to upload to object storage", e);
        }
    }
    
    @Override
    public String get(ContentRef ref) {
        try {
            String key = extractKey(ref.getPath());
            
            try (InputStream stream = minioClient.getObject(
                    GetObjectArgs.builder()
                        .bucket(bucket)
                        .object(key)
                        .build()
                )) {
                return new String(stream.readAllBytes(), StandardCharsets.UTF_8);
            }
            
        } catch (Exception e) {
            log.error("Failed to download from MinIO: path={}", ref.getPath(), e);
            throw new ContentNotFoundException(ref);
        }
    }
    
    @Override
    public void delete(ContentRef ref) {
        try {
            String key = extractKey(ref.getPath());
            
            minioClient.removeObject(
                RemoveObjectArgs.builder()
                    .bucket(bucket)
                    .object(key)
                    .build()
            );
            
            log.info("Deleted from MinIO: key={}", key);
            
        } catch (Exception e) {
            log.error("Failed to delete from MinIO: path={}", ref.getPath(), e);
            throw new StorageException("Failed to delete from object storage", e);
        }
    }
    
    @Override
    public boolean exists(ContentRef ref) {
        try {
            String key = extractKey(ref.getPath());
            
            minioClient.statObject(
                StatObjectArgs.builder()
                    .bucket(bucket)
                    .object(key)
                    .build()
            );
            
            return true;
            
        } catch (Exception e) {
            return false;
        }
    }
    
    private String extractKey(String path) {
        // "oss://bucket/path/to/file" → "path/to/file"
        return path.replaceFirst("oss://[^/]+/", "");
    }
    
    private String calculateChecksum(byte[] contentBytes) {
        return DigestUtils.sha256Hex(contentBytes);
    }
}
```

---

## 四、数据库表设计

### 4.1 resource_version 表扩展

```sql
-- 添加存储相关字段
ALTER TABLE resource_version 
ADD COLUMN content_inline TEXT,              -- 小内容内联存储
ADD COLUMN content_ref VARCHAR(500),         -- 大内容引用（OSS 路径）
ADD COLUMN content_size BIGINT DEFAULT 0,    -- 内容大小（字节）
ADD COLUMN content_type VARCHAR(100) DEFAULT 'text/plain', -- 内容类型
ADD COLUMN checksum VARCHAR(64);             -- SHA-256 校验和

-- 添加索引（优化查询）
CREATE INDEX idx_resource_version_content_size ON resource_version(content_size);
CREATE INDEX idx_resource_version_content_ref ON resource_version(content_ref);

-- 添加注释
COMMENT ON COLUMN resource_version.content_inline IS '小内容内联存储（<1MB）';
COMMENT ON COLUMN resource_version.content_ref IS '大内容引用（>1MB，OSS 路径）';
COMMENT ON COLUMN resource_version.content_size IS '内容大小（字节）';
COMMENT ON COLUMN resource_version.content_type IS '内容类型（text/plain, application/json 等）';
COMMENT ON COLUMN resource_version.checksum IS 'SHA-256 校验和（用于完整性验证）';
```

### 4.2 存储统计表（可选）

```sql
-- 存储统计信息表
CREATE TABLE storage_statistics (
    id BIGSERIAL PRIMARY KEY,
    
    -- 统计维度
    stat_date DATE NOT NULL,
    workspace_id VARCHAR(100),
    resource_type VARCHAR(50),
    
    -- 存储统计
    total_count BIGINT DEFAULT 0,           -- 总数量
    total_size BIGINT DEFAULT 0,            -- 总大小（字节）
    db_count BIGINT DEFAULT 0,              -- DB 存储数量
    db_size BIGINT DEFAULT 0,               -- DB 存储大小
    oss_count BIGINT DEFAULT 0,             -- OSS 存储数量
    oss_size BIGINT DEFAULT 0,              -- OSS 存储大小
    
    -- 大小分布
    size_0_100kb BIGINT DEFAULT 0,          -- 0-100KB
    size_100kb_1mb BIGINT DEFAULT 0,        -- 100KB-1MB
    size_1mb_10mb BIGINT DEFAULT 0,         -- 1MB-10MB
    size_10mb_plus BIGINT DEFAULT 0,        -- 10MB+
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    UNIQUE(stat_date, workspace_id, resource_type)
);

-- 每天凌晨 2 点统计
CREATE INDEX idx_storage_statistics_date ON storage_statistics(stat_date);
```

---

## 五、配置管理

### 5.1 application.yml 配置

```yaml
storage:
  # 大小阈值配置
  threshold:
    database: 1048576        # 1MB（DB 存储上限）
    warning: 5242880         # 5MB（超过告警）
    max: 104857600           # 100MB（绝对上限）
  
  # MinIO 配置
  minio:
    enabled: true
    endpoint: http://localhost:9000
    access-key: minioadmin
    secret-key: minioadmin
    bucket: matrix-skills
    auto-create-bucket: true
  
  # 阿里云 OSS 配置（备选）
  oss:
    enabled: false
    endpoint: oss-cn-hangzhou.aliyuncs.com
    access-key-id: ${OSS_ACCESS_KEY_ID}
    access-key-secret: ${OSS_ACCESS_KEY_SECRET}
    bucket: matrix-skills
  
  # AWS S3 配置（备选）
  s3:
    enabled: false
    region: us-east-1
    bucket: matrix-skills
  
  # CDN 配置（加速 OSS 访问）
  cdn:
    enabled: false
    domain: https://cdn.matrix.ai
    cache-ttl: 86400  # 24 小时
  
  # 监控配置
  monitoring:
    enabled: true
    log-large-files: true  # 记录大文件上传
    large-file-threshold: 5242880  # 5MB
```

### 5.2 动态配置（支持运行时调整）

```java
@Configuration
@ConfigurationProperties(prefix = "storage")
@Data
public class StorageProperties {
    
    /**
     * DB 存储阈值（字节）
     */
    private long databaseThreshold = 1048576; // 1MB
    
    /**
     * 告警阈值（字节）
     */
    private long warningThreshold = 5242880; // 5MB
    
    /**
     * 最大阈值（字节）
     */
    private long maxThreshold = 104857600; // 100MB
    
    /**
     * MinIO 配置
     */
    private MinioProperties minio = new MinioProperties();
    
    @Data
    public static class MinioProperties {
        private boolean enabled = true;
        private String endpoint = "http://localhost:9000";
        private String accessKey = "minioadmin";
        private String secretKey = "minioadmin";
        private String bucket = "matrix-skills";
        private boolean autoCreateBucket = true;
    }
}
```

---

## 六、数据迁移方案

### 6.1 从纯 DB 迁移到混合存储

**迁移脚本**：
```java
@Component
public class StorageMigrationService {
    
    @Autowired
    private ResourceVersionRepository repository;
    
    @Autowired
    private ContentStorage contentStorage;
    
    @Autowired
    private StorageProperties properties;
    
    /**
     * 迁移大文件到 OSS
     * 分批处理，避免内存溢出
     */
    @Scheduled(cron = "0 0 2 * * ?") // 每天凌晨 2 点
    @Transactional
    public void migrateLargeFiles() {
        long threshold = properties.getDatabaseThreshold();
        
        log.info("Starting migration of large files (>{}MB)", threshold / 1024 / 1024);
        
        int batchSize = 100;
        int pageNum = 0;
        int totalMigrated = 0;
        
        while (true) {
            // 分页查询大文件
            Page<ResourceVersion> page = repository.findByContentSizeGreaterThanEqual(
                threshold,
                PageRequest.of(pageNum, batchSize)
            );
            
            if (page.isEmpty()) {
                break;
            }
            
            for (ResourceVersion version : page.getContent()) {
                try {
                    migrateVersion(version);
                    totalMigrated++;
                } catch (Exception e) {
                    log.error("Failed to migrate version: id={}", version.getId(), e);
                }
            }
            
            pageNum++;
        }
        
        log.info("Migration completed: totalMigrated={}", totalMigrated);
    }
    
    private void migrateVersion(ResourceVersion version) {
        String content = version.getContentInline();
        if (content == null || version.getContentRef() != null) {
            return; // 已迁移或无内容
        }
        
        // 保存到 OSS
        StorageContext context = StorageContext.builder()
            .workspaceId(version.getWorkspaceId())
            .resourceType(version.getResource().getType().name())
            .resourceId(version.getResource().getId())
            .version(version.getVersion())
            .build();
        
        ContentRef ref = contentStorage.save(content, context);
        
        // 更新数据库
        version.setContentRef(ref.getPath());
        version.setContentInline(null);
        version.setContentType(ref.getContentType());
        
        repository.save(version);
        
        log.info("Migrated version to OSS: id={}, size={}, path={}", 
            version.getId(), ref.getSize(), ref.getPath());
    }
}
```

### 6.2 回滚方案

```java
@Service
public class StorageRollbackService {
    
    /**
     * 回滚指定版本的存储位置
     * 用于 OSS 故障时临时切回 DB
     */
    @Transactional
    public void rollbackToDatabase(Long versionId) {
        ResourceVersion version = repository.findById(versionId)
            .orElseThrow(() -> new ResourceNotFoundException(versionId));
        
        if (version.getContentRef() == null) {
            return; // 已经在 DB 中
        }
        
        // 从 OSS 读取内容
        String content = contentStorage.get(ContentRef.builder()
            .path(version.getContentRef())
            .build());
        
        // 存回 DB
        version.setContentInline(content);
        version.setContentRef(null);
        
        repository.save(version);
        
        log.info("Rolled back to database: versionId={}", versionId);
    }
}
```

---

## 七、监控与告警

### 7.1 指标采集

```java
@Component
public class StorageMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public StorageMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        // 注册指标
        Gauge.builder("storage.db.size", this, StorageMetrics::getDbSize)
            .description("Database storage size (bytes)")
            .baseUnit("bytes")
            .register(meterRegistry);
        
        Gauge.builder("storage.oss.size", this, StorageMetrics::getOssSize)
            .description("Object storage size (bytes)")
            .baseUnit("bytes")
            .register(meterRegistry);
        
        Counter.builder("storage.large.file.count")
            .description("Count of large files (>1MB)")
            .register(meterRegistry);
    }
    
    private double getDbSize() {
        // 从数据库统计
        return repository.sumContentSizeByStorageType("DATABASE");
    }
    
    private double getOssSize() {
        // 从数据库统计
        return repository.sumContentSizeByStorageType("OBJECT_STORAGE");
    }
}
```

### 7.2 告警规则

```yaml
# Prometheus 告警规则
groups:
  - name: storage
    rules:
      # DB 存储过大告警
      - alert: DatabaseStorageTooLarge
        expr: storage_db_size_bytes > 10737418240  # 10GB
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Database storage size is too large"
          description: "DB storage size is {{ $value }} bytes"
      
      # 大文件上传告警
      - alert: LargeFileUploaded
        expr: increase(storage_large_file_count[1h]) > 100
        for: 5m
        labels:
          severity: info
        annotations:
          summary: "Many large files uploaded"
          description: "{{ $value }} large files uploaded in last hour"
      
      # OSS 存储失败告警
      - alert: ObjectStorageUploadFailed
        expr: increase(storage_oss_upload_failed_total[5m]) > 5
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Object storage upload failures"
          description: "{{ $value }} upload failures in last 5 minutes"
```

---

## 八、性能优化

### 8.1 CDN 加速

```java
@Service
public class CdnService {
    
    @Autowired
    private ObjectStorage objectStorage;
    
    @Value("${storage.cdn.domain:https://cdn.matrix.ai}")
    private String cdnDomain;
    
    /**
     * 获取 CDN 加速 URL
     */
    public String getCdnUrl(ContentRef ref) {
        if (!ref.isExternal()) {
            return null; // DB 存储不支持 CDN
        }
        
        String key = extractKey(ref.getPath());
        return cdnDomain + "/" + key;
    }
    
    /**
     * 预加热 CDN 缓存
     */
    public void preloadCdn(ContentRef ref) {
        String cdnUrl = getCdnUrl(ref);
        if (cdnUrl != null) {
            // 发起请求预热 CDN
            restTemplate.getForObject(cdnUrl, String.class);
            log.info("Preloaded CDN: url={}", cdnUrl);
        }
    }
}
```

### 8.2 懒加载优化

```java
@Service
public class SkillService {
    
    /**
     * 获取 Skill 详情（内容懒加载）
     */
    public SkillDTO getSkillDetail(Long id) {
        ResourceVersion version = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException(id));
        
        SkillDTO dto = toDTO(version);
        
        // 不立即加载内容，只返回引用
        // 前端按需请求内容
        dto.setLazyLoad(true);
        dto.setContentRef(version.getContentRef());
        
        return dto;
    }
    
    /**
     * 单独获取内容（按需加载）
     */
    public String getSkillContent(Long id) {
        ResourceVersion version = repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException(id));
        
        return contentStorage.get(ContentRef.builder()
            .path(version.getContentRef())
            .type(version.getContentRef() != null ? StorageType.OBJECT_STORAGE : StorageType.DATABASE)
            .build());
    }
}
```

---

## 九、总结

### 9.1 实施步骤

**阶段 1：基础建设（1-2 周）**
- [ ] 添加数据库字段（content_ref, content_size 等）
- [ ] 实现 ContentStorage 接口
- [ ] 集成 MinIO/OSS

**阶段 2：迁移工具（1 周）**
- [ ] 开发数据迁移脚本
- [ ] 测试迁移流程
- [ ] 准备回滚方案

**阶段 3：灰度上线（1 周）**
- [ ] 新 Skill 使用混合存储
- [ ] 监控指标采集
- [ ] 配置告警规则

**阶段 4：全量迁移（按需）**
- [ ] 分批迁移历史大文件
- [ ] 验证数据完整性
- [ ] 清理 DB 冗余数据

### 9.2 关键要点

1. **阈值设定**：1MB 是经验值，可根据实际情况调整
2. **校验和**：必须计算 checksum，确保数据完整性
3. **监控告警**：实时监控存储大小和失败率
4. **懒加载**：内容按需加载，提升列表查询性能
5. **CDN 加速**：大文件访问通过 CDN 加速

---

*方案生成时间：2026 年 3 月 16 日*  
*版本：v1.0*  
*核心洞察：DB+OSS 混合存储，1MB 阈值自动分流，透明切换无感知*