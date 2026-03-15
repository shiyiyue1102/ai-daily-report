# Skill 大文件存储技术实现方案（优化版）

## 一、核心设计优化

### 1.1 ContentRef 简化设计

**原方案**（冗余）：
```java
ContentRef {
    type: StorageType
    path: String
    size: Long           // ❌ 单独字段
    checksum: String     // ❌ 单独字段
    contentType: String
    provider: String     // ❌ 缺失
}
```

**优化方案**（JSON 化）：
```java
ContentRef {
    provider: String          // ✅ 存储提供者：s3, minio, oss
    bucket: String            // 存储桶
    key: String               // 对象键
    metadata: JsonNode        // ✅ 元数据（size, checksum 等）
}
```

**metadata JSON 结构**：
```json
{
  "size": 2048576,
  "checksum": "sha256:abc123...",
  "contentType": "text/plain",
  "createdAt": "2026-03-16T00:00:00Z",
  "expiresAt": null
}
```

---

### 1.2 数据库字段简化

**原方案**（多字段）：
```sql
content_inline TEXT,
content_ref VARCHAR(500),
content_size BIGINT,
content_type VARCHAR(100),
checksum VARCHAR(64)
```

**优化方案**（单字段）：
```sql
-- 小文件：直接存内容
-- 大文件：存 JSON 引用
content_data TEXT
```

**content_data 内容**：
```json
{
  "storage": "inline",
  "content": "..."
}
```

或

```json
{
  "storage": "external",
  "provider": "minio",
  "bucket": "matrix-skills",
  "key": "skills/default/123/v1.0.0",
  "metadata": {
    "size": 2048576,
    "checksum": "sha256:abc123...",
    "contentType": "text/plain",
    "createdAt": "2026-03-16T00:00:00Z"
  }
}
```

---

## 二、存储提供者抽象

### 2.1 Provider 接口

```java
public interface StorageProvider {
    
    /**
     * 提供者名称
     */
    String getName(); // "s3", "minio", "oss", "local"
    
    /**
     * 保存内容
     */
    StorageRef save(byte[] content, StorageContext context);
    
    /**
     * 读取内容
     */
    byte[] get(StorageRef ref);
    
    /**
     * 删除内容
     */
    void delete(StorageRef ref);
    
    /**
     * 获取下载 URL（用于 CDN/临时访问）
     */
    String getDownloadUrl(StorageRef ref, Duration expiry);
}
```

### 2.2 StorageRef 设计

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class StorageRef {
    
    /**
     * 存储提供者
     */
    private String provider; // "s3", "minio", "oss", "local"
    
    /**
     * 存储桶
     */
    private String bucket;
    
    /**
     * 对象键
     */
    private String key;
    
    /**
     * 元数据（JSON）
     */
    private JsonNode metadata;
    
    /**
     * 获取大小
     */
    public Long getSize() {
        return metadata.has("size") ? metadata.get("size").asLong() : null;
    }
    
    /**
     * 获取校验和
     */
    public String getChecksum() {
        return metadata.has("checksum") ? metadata.get("checksum").asText() : null;
    }
    
    /**
     * 获取内容类型
     */
    public String getContentType() {
        return metadata.has("contentType") ? metadata.get("contentType").asText() : "text/plain";
    }
    
    /**
     * 转为 JSON 存储到 DB
     */
    public String toJson() {
        return JsonUtils.toJson(this);
    }
    
    /**
     * 从 JSON 解析
     */
    public static StorageRef fromJson(String json) {
        return JsonUtils.fromJson(json, StorageRef.class);
    }
}
```

---

## 三、对外接口设计

### 3.1 关键问题：应用侧是否需要感知存储？

**答案：不需要！应用侧不应该感知存储细节。**

### 3.2 接口设计原则

| 原则 | 说明 |
|------|------|
| **存储透明** | 应用侧只关心内容，不关心存储位置 |
| **服务端中转** | 所有读写通过 Matrix Server，不直接暴露 OSS |
| **统一接口** | 无论 DB 还是 OSS，调用方式一致 |

### 3.3 为什么需要服务端中转？

**❌ 错误设计（应用侧直接读 OSS）**：
```
应用侧 → 获取 OSS URL → 直接访问 OSS
         ↓
      暴露 OSS 凭证
      无法控制权限
      无法审计访问
      无法缓存优化
```

**✅ 正确设计（服务端中转）**：
```
应用侧 → Matrix Server → 读取内容（DB 或 OSS）
         ↓
      统一权限控制
      访问审计日志
      CDN 缓存优化
      存储透明
```

### 3.4 API 设计

```java
@RestController
@RequestMapping("/api/resources")
public class ResourceController {
    
    @Autowired
    private ResourceService resourceService;
    
    /**
     * 获取资源详情（不包含内容）
     */
    @GetMapping("/{id}")
    public ResourceDTO getResource(@PathVariable Long id) {
        // 返回元数据，不返回内容
        return resourceService.getResource(id);
    }
    
    /**
     * 获取资源内容（按需加载）
     */
    @GetMapping("/{id}/content")
    public ResponseEntity<byte[]> getResourceContent(@PathVariable Long id) {
        byte[] content = resourceService.getContent(id);
        
        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("text/plain"))
            .body(content);
    }
    
    /**
     * 下载资源内容（带文件名）
     */
    @GetMapping("/{id}/download")
    public ResponseEntity<byte[]> downloadResource(@PathVariable Long id) {
        ResourceDTO resource = resourceService.getResource(id);
        byte[] content = resourceService.getContent(id);
        
        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("text/plain"))
            .header(HttpHeaders.CONTENT_DISPOSITION, 
                "attachment; filename=\"" + resource.getKey() + ".txt\"")
            .body(content);
    }
    
    /**
     * 获取临时访问 URL（用于 CDN/前端直传）
     */
    @GetMapping("/{id}/upload-url")
    public UploadUrlDTO getUploadUrl(@PathVariable Long id) {
        // 仅用于大文件上传场景
        return resourceService.getUploadUrl(id);
    }
}
```

### 3.5 应用侧调用示例

**Java SDK**：
```java
// 获取资源元数据
ResourceDTO resource = client.getResource(resourceId);
System.out.println("Key: " + resource.getKey());
System.out.println("Version: " + resource.getVersion());

// 获取资源内容（透明，不关心存储位置）
byte[] content = client.getContent(resourceId);
String code = new String(content, StandardCharsets.UTF_8);

// 或使用流式读取（大文件优化）
try (InputStream stream = client.getContentStream(resourceId)) {
    // 处理流式内容
}
```

**Python SDK**：
```python
# 获取资源元数据
resource = client.get_resource(resource_id)
print(f"Key: {resource.key}")

# 获取资源内容（透明）
content = client.get_content(resource_id)
code = content.decode('utf-8')

# 或流式读取
with client.get_content_stream(resource_id) as stream:
    for chunk in stream:
        process(chunk)
```

---

## 四、核心实现

### 4.1 ResourceService 实现

```java
@Service
public class ResourceService {
    
    @Autowired
    private ResourceVersionRepository repository;
    
    @Autowired
    private StorageProviderManager storageProviderManager;
    
    /**
     * 保存资源
     */
    @Transactional
    public ResourceVersion saveResource(SaveResourceRequest request) {
        byte[] contentBytes = request.getContent().getBytes(StandardCharsets.UTF_8);
        
        // 判断存储方式
        StorageRef ref;
        if (contentBytes.length < SIZE_THRESHOLD) {
            // 小文件：内联存储
            ref = StorageRef.builder()
                .provider("inline")
                .metadata(JsonUtils.createObjectNode()
                    .put("size", contentBytes.length)
                    .put("checksum", calculateChecksum(contentBytes))
                    .put("contentType", "text/plain"))
                .build();
        } else {
            // 大文件：外部存储
            StorageProvider provider = storageProviderManager.getDefaultProvider();
            StorageContext context = buildContext(request);
            ref = provider.save(contentBytes, context);
        }
        
        // 保存到 DB（content_data 字段存 JSON）
        ResourceVersion version = new ResourceVersion();
        version.setResourceId(request.getResourceId());
        version.setVersion(request.getVersion());
        version.setContentData(ref.toJson());
        
        return repository.save(version);
    }
    
    /**
     * 获取资源内容
     */
    public byte[] getContent(Long versionId) {
        ResourceVersion version = repository.findById(versionId)
            .orElseThrow(() -> new ResourceNotFoundException(versionId));
        
        StorageRef ref = StorageRef.fromJson(version.getContentData());
        
        if ("inline".equals(ref.getProvider())) {
            // 内联存储：从 JSON 中提取内容
            return extractInlineContent(ref);
        } else {
            // 外部存储：从 OSS 读取
            StorageProvider provider = storageProviderManager.getProvider(ref.getProvider());
            return provider.get(ref);
        }
    }
    
    /**
     * 获取临时上传 URL（用于大文件直传）
     */
    public UploadUrlDTO getUploadUrl(Long versionId) {
        // 仅用于超大文件（>10MB）
        // 应用侧直传 OSS，完成后回调更新 DB
        StorageProvider provider = storageProviderManager.getDefaultProvider();
        String uploadUrl = provider.getUploadUrl(buildContext(versionId));
        
        return UploadUrlDTO.builder()
            .uploadUrl(uploadUrl)
            .method("PUT")
            .expiresIn(3600)
            .build();
    }
}
```

### 4.2 StorageProviderManager

```java
@Component
public class StorageProviderManager {
    
    private final Map<String, StorageProvider> providers = new ConcurrentHashMap<>();
    
    @Autowired
    public StorageProviderManager(List<StorageProvider> providerList) {
        for (StorageProvider provider : providerList) {
            providers.put(provider.getName(), provider);
        }
    }
    
    /**
     * 获取默认提供者
     */
    public StorageProvider getDefaultProvider() {
        return providers.get("minio"); // 默认 MinIO
    }
    
    /**
     * 根据名称获取提供者
     */
    public StorageProvider getProvider(String name) {
        StorageProvider provider = providers.get(name);
        if (provider == null) {
            throw new IllegalArgumentException("Unknown storage provider: " + name);
        }
        return provider;
    }
}
```

### 4.3 MinIO Provider 实现

```java
@Component
@ConditionalOnProperty(name = "storage.minio.enabled", havingValue = "true")
public class MinioStorageProvider implements StorageProvider {
    
    @Autowired
    private MinioClient minioClient;
    
    @Value("${storage.minio.bucket:matrix-skills}")
    private String bucket;
    
    @Override
    public String getName() {
        return "minio";
    }
    
    @Override
    public StorageRef save(byte[] content, StorageContext context) {
        try {
            String key = context.generateKey();
            
            // 上传到 MinIO
            minioClient.putObject(
                PutObjectArgs.builder()
                    .bucket(bucket)
                    .object(key)
                    .stream(new ByteArrayInputStream(content), content.length, -1)
                    .contentType("text/plain")
                    .build()
            );
            
            // 构建引用
            ObjectNode metadata = JsonUtils.createObjectNode()
                .put("size", content.length)
                .put("checksum", calculateChecksum(content))
                .put("contentType", "text/plain")
                .put("createdAt", LocalDateTime.now().toString());
            
            return StorageRef.builder()
                .provider("minio")
                .bucket(bucket)
                .key(key)
                .metadata(metadata)
                .build();
                
        } catch (Exception e) {
            throw new StorageException("Failed to upload to MinIO", e);
        }
    }
    
    @Override
    public byte[] get(StorageRef ref) {
        try {
            try (InputStream stream = minioClient.getObject(
                    GetObjectArgs.builder()
                        .bucket(ref.getBucket())
                        .object(ref.getKey())
                        .build()
                )) {
                return stream.readAllBytes();
            }
        } catch (Exception e) {
            throw new ContentNotFoundException(ref);
        }
    }
    
    @Override
    public String getDownloadUrl(StorageRef ref, Duration expiry) {
        try {
            return minioClient.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                    .method(Method.GET)
                    .bucket(ref.getBucket())
                    .object(ref.getKey())
                    .expiry((int) expiry.getSeconds())
                    .build()
            );
        } catch (Exception e) {
            throw new StorageException("Failed to generate download URL", e);
        }
    }
    
    // 其他方法实现...
}
```

---

## 五、数据库表设计（最终版）

### 5.1 resource_version 表

```sql
CREATE TABLE resource_version (
    id BIGSERIAL PRIMARY KEY,
    resource_id BIGINT NOT NULL REFERENCES resource(id),
    version VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL,
    
    -- 内容存储（JSON 格式）
    -- 小文件：{"storage":"inline","content":"..."}
    -- 大文件：{"storage":"external","provider":"minio","bucket":"...","key":"...","metadata":{...}}
    content_data TEXT,
    
    -- 元数据
    metadata JSONB DEFAULT '{}',
    gray_config JSONB,
    gray_priority INTEGER DEFAULT 0,
    
    -- 时间戳
    published_at TIMESTAMP,
    expires_at TIMESTAMP,
    archived_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    last_accessed_at TIMESTAMP,
    
    UNIQUE(resource_id, version)
);

-- 索引
CREATE INDEX idx_resource_version_resource_id ON resource_version(resource_id);
CREATE INDEX idx_resource_version_status ON resource_version(status);

-- 注释
COMMENT ON COLUMN resource_version.content_data IS '内容存储（JSON 格式，内联或外部引用）';
```

### 5.2 示例数据

**小文件（内联存储）**：
```json
{
  "storage": "inline",
  "content": "def hello():\n    print('Hello, World!')"
}
```

**大文件（外部存储）**：
```json
{
  "storage": "external",
  "provider": "minio",
  "bucket": "matrix-skills",
  "key": "skills/default/123/v1.0.0",
  "metadata": {
    "size": 2048576,
    "checksum": "sha256:abc123...",
    "contentType": "text/plain",
    "createdAt": "2026-03-16T00:00:00Z"
  }
}
```

---

## 六、配置管理

### 6.1 application.yml

```yaml
storage:
  # 大小阈值
  threshold:
    inline: 1048576        # 1MB（内联存储上限）
    warning: 5242880       # 5MB（超过告警）
    max: 104857600         # 100MB（绝对上限）
  
  # 默认提供者
  default-provider: minio
  
  # MinIO 配置
  minio:
    enabled: true
    endpoint: http://localhost:9000
    access-key: minioadmin
    secret-key: minioadmin
    bucket: matrix-skills
    auto-create-bucket: true
  
  # 阿里云 OSS 配置
  oss:
    enabled: false
    endpoint: oss-cn-hangzhou.aliyuncs.com
    access-key-id: ${OSS_ACCESS_KEY_ID}
    access-key-secret: ${OSS_ACCESS_KEY_SECRET}
    bucket: matrix-skills
  
  # CDN 配置
  cdn:
    enabled: false
    domain: https://cdn.matrix.ai
    cache-ttl: 86400
```

---

## 七、总结

### 7.1 核心优化

| 优化点 | 原方案 | 优化方案 |
|--------|--------|---------|
| **ContentRef** | 多字段 | JSON 化（provider + metadata） |
| **数据库字段** | 5 个字段 | 1 个字段（content_data） |
| **存储提供者** | 硬编码 | Provider 抽象（可扩展） |
| **对外接口** | 可能暴露 OSS | 服务端中转（透明） |

### 7.2 设计原则

1. **应用侧透明**：不感知存储细节
2. **服务端中转**：统一权限控制和审计
3. **JSON 化**：content_data 存 JSON，灵活扩展
4. **Provider 抽象**：支持多存储提供者

### 7.3 接口调用流程

```
应用侧
  │
  ├─ getResource(id) ────▶ 返回元数据（不含内容）
  │
  └─ getContent(id) ──────▶ Matrix Server ──┬─ DB（小文件）
                                              │
                                              └─ OSS（大文件）
```

---

*方案生成时间：2026 年 3 月 16 日*  
*版本：v2.0（优化版）*  
*核心洞察：ContentRef JSON 化、应用侧透明、服务端中转*