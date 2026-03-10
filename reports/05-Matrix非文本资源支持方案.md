# Matrix 非文本资源支持方案

## 一、方案概述

**核心思路**：抽象存储接口 + 元数据与文件分离 + 客户端直传

## 二、架构设计

```
┌─────────────────────────────────────────────────────────┐
│              Matrix Resource Service                     │
├─────────────────────────────────────────────────────────┤
│  API Layer: 上传/下载/删除/查询                          │
├─────────────────────────────────────────────────────────┤
│  Storage Abstraction Layer ⭐                            │
│  • StorageProvider 接口                                  │
│  • S3Adapter / MinIOAdapter / OSSAdapter                │
├─────────────────────────────────────────────────────────┤
│  Third Party Storage                                     │
│  • AWS S3 / 阿里云 OSS / MinIO / 本地存储               │
└─────────────────────────────────────────────────────────┘
```

## 三、表结构改动（最小化）

```sql
-- resources 表新增字段
ALTER TABLE resources ADD COLUMN content_type VARCHAR(50); -- text, image, executable, model
ALTER TABLE resources ADD COLUMN storage_ref JSONB; -- 存储地址引用

-- storage_ref 结构示例：
-- {
--   "provider": "s3",
--   "bucket": "matrix-resources",
--   "key": "skills/abc123/v1.0/model.onnx",
--   "cdn_url": "https://cdn.matrix.ai/...",
--   "size": 104857600,
--   "checksum": "sha256:abc123..."
-- }
```

## 四、核心接口

```java
public interface StorageProvider {
    // 生成预签名上传URL（客户端直传）
    PresignedUrl generateUploadUrl(String bucket, String key, long expiry);
    
    // 生成预签名下载URL
    PresignedUrl generateDownloadUrl(String bucket, String key, long expiry);
    
    // 上传（服务器端，小文件）
    UploadResult upload(InputStream data, String bucket, String key);
    
    // 删除
    void delete(String bucket, String key);
    
    // 获取元数据
    FileMetadata getMetadata(String bucket, String key);
}
```

## 五、实施步骤

1. **创建 StorageProvider 接口**（1天）
2. **实现 S3/MinIO适配器**（2天）
3. **修改 Resource 表结构**（0.5天）
4. **实现上传/下载API**（2天）
5. **安全检查（病毒扫描、沙箱）**（2天）
6. **客户端SDK支持**（2天）

**总计**：约10个工作日

## 六、配置示例

```yaml
storage:
  provider: minio  # s3, minio, oss, local
  minio:
    endpoint: http://localhost:9000
    access-key: minioadmin
    secret-key: minioadmin
    bucket: matrix-resources
  cdn:
    base-url: https://cdn.matrix.ai
```

## 七、安全策略

| 资源类型 | 安全措施 |
|---------|---------|
| 图片 | NSFW检测、EXIF清理 |
| 可执行文件 | 病毒扫描、沙箱测试、签名验证 |
| 模型文件 | 模型扫描、推理验证 |
| 配置文件 | 密钥扫描、schema验证 |

---

*生成时间：2026年3月10日*