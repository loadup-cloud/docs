# LoadUp Components :: Signature

## 概述

`loadup-components-signature` 是一个数字签名和摘要计算组件，提供统一的 API 支持多种签名算法（RSA、DSA、ECDSA）和摘要算法（MD5、SHA、HMAC）。

## 核心特性

- ✅ **多种签名算法**: RSA、DSA、ECDSA
- ✅ **多种摘要算法**: MD5、SHA-1、SHA-256、SHA-512
- ✅ **HMAC 支持**: HmacSHA256、HmacSHA512
- ✅ **双重 API**: Service（依赖注入）+ Utils（静态方法）
- ✅ **密钥管理**: 密钥对生成、格式转换
- ✅ **类型安全**: 使用枚举定义算法
- ✅ **完善测试**: 覆盖率 ≥ 80%

## 快速开始

### 1. 引入依赖

```xml
<dependency>
    <groupId>io.github.loadup-cloud</groupId>
    <artifactId>loadup-components-signature</artifactId>
</dependency>
```

### 2. 配置

```yaml
loadup:
  components:
    signature:
      enabled: true                           # 是否启用，默认 true
      default-signature-algorithm: SHA256_WITH_RSA  # 默认签名算法
      default-digest-algorithm: SHA256               # 默认摘要算法
      key-size:
        rsa: 2048                             # RSA 密钥长度
        dsa: 2048                             # DSA 密钥长度
        ec: 256                               # ECDSA 曲线长度
```

### 3. 使用示例

#### 方式一：使用 Service（依赖注入）

```java
@Service
@RequiredArgsConstructor
public class PaymentService {
    
    private final SignatureService signatureService;
    private final KeyPairService keyPairService;
    
    public void processPayment(PaymentRequest request) {
        // 1. 生成密钥对（通常在初始化时生成一次）
        KeyPairInfo keyPair = keyPairService.generateKeyPair(KeyAlgorithm.RSA, 2048);
        
        // 2. 签名
        String signature = signatureService.sign(
            request.getData(),
            keyPair.getPrivateKey(),
            SignatureAlgorithm.SHA256_WITH_RSA
        );
        
        // 3. 验签
        boolean valid = signatureService.verify(
            request.getData(),
            signature,
            keyPair.getPublicKey(),
            SignatureAlgorithm.SHA256_WITH_RSA
        );
    }
}
```

#### 方式二：使用 Utils（静态方法）

```java
public class QuickSignatureExample {
    
    public void example() {
        // RSA 签名
        String signature = SignatureUtils.signRSA(data, privateKey);
        boolean valid = SignatureUtils.verifyRSA(data, signature, publicKey);
        
        // MD5 摘要
        String md5 = DigestUtils.md5("hello world");
        
        // SHA-256 摘要
        String sha256 = DigestUtils.sha256("hello world");
        
        // HMAC-SHA256
        String hmac = DigestUtils.hmacSha256(data, secretKey);
    }
}
```

## 支持的算法

### 签名算法

| 算法                 | 枚举值                 | JCA 名称          | 密钥类型 | 推荐密钥长度 |
|--------------------|---------------------|-----------------|------|--------|
| RSA with SHA-256   | `SHA256_WITH_RSA`   | SHA256withRSA   | RSA  | 2048+  |
| RSA with SHA-512   | `SHA512_WITH_RSA`   | SHA512withRSA   | RSA  | 2048+  |
| DSA with SHA-256   | `SHA256_WITH_DSA`   | SHA256withDSA   | DSA  | 2048+  |
| ECDSA with SHA-256 | `SHA256_WITH_ECDSA` | SHA256withECDSA | EC   | 256+   |

### 摘要算法

| 算法          | 枚举值           | JCA 名称     | 输出长度              | 安全性    |
|-------------|---------------|------------|-------------------|--------|
| MD5         | `MD5`         | MD5        | 128 bit (32 Hex)  | ⚠️ 不推荐 |
| SHA-1       | `SHA1`        | SHA-1      | 160 bit (40 Hex)  | ⚠️ 不推荐 |
| SHA-256     | `SHA256`      | SHA-256    | 256 bit (64 Hex)  | ✅ 推荐   |
| SHA-512     | `SHA512`      | SHA-512    | 512 bit (128 Hex) | ✅ 推荐   |
| HMAC-SHA256 | `HMAC_SHA256` | HmacSHA256 | 256 bit (64 Hex)  | ✅ 推荐   |
| HMAC-SHA512 | `HMAC_SHA512` | HmacSHA512 | 512 bit (128 Hex) | ✅ 推荐   |

## API 文档

### SignatureService

```java
public interface SignatureService {
    
    // 使用私钥对数据进行签名
    String sign(byte[] data, PrivateKey privateKey, SignatureAlgorithm algorithm);
    
    // 使用私钥对字符串进行签名
    String sign(String data, String privateKeyBase64, SignatureAlgorithm algorithm);
    
    // 使用公钥验证签名
    boolean verify(byte[] data, String signatureBase64, PublicKey publicKey, SignatureAlgorithm algorithm);
    
    // 使用公钥验证签名（字符串形式）
    boolean verify(String data, String signatureBase64, String publicKeyBase64, SignatureAlgorithm algorithm);
}
```

### DigestService

```java
public interface DigestService {
    
    // 计算数据的摘要
    String digest(byte[] data, DigestAlgorithm algorithm);
    
    // 计算字符串的摘要
    String digest(String data, DigestAlgorithm algorithm);
    
    // 计算 HMAC 摘要
    String hmac(byte[] data, byte[] key, DigestAlgorithm algorithm);
    
    // 计算字符串的 HMAC
    String hmac(String data, String key, DigestAlgorithm algorithm);
}
```

### KeyPairService

```java
public interface KeyPairService {
    
    // 生成密钥对
    KeyPairInfo generateKeyPair(KeyAlgorithm algorithm, int keySize);
    
    // 生成密钥对（使用默认密钥长度）
    KeyPairInfo generateKeyPair(KeyAlgorithm algorithm);
    
    // 从 Base64 字符串加载私钥
    PrivateKey loadPrivateKey(String base64PrivateKey, KeyAlgorithm algorithm);
    
    // 从 Base64 字符串加载公钥
    PublicKey loadPublicKey(String base64PublicKey, KeyAlgorithm algorithm);
}
```

## 使用场景

### 场景 1: API 签名验签（类似支付宝/微信支付）

```java
@Service
@RequiredArgsConstructor
public class ThirdPartyApiService {
    private final SignatureService signatureService;
    
    // 调用第三方 API 时签名
    public void callApi(ApiRequest request) {
        String signData = buildSignData(request);
        String signature = signatureService.sign(
            signData, 
            merchantPrivateKey, 
            SignatureAlgorithm.SHA256_WITH_RSA
        );
        request.setSignature(signature);
        httpClient.post(apiUrl, request);
    }
    
    // 验证第三方回调签名
    public boolean verifyCallback(CallbackRequest callback) {
        return signatureService.verify(
            callback.getData(),
            callback.getSignature(),
            platformPublicKey,
            SignatureAlgorithm.SHA256_WITH_RSA
        );
    }
}
```

### 场景 2: 文件完整性校验

```java
@Service
@RequiredArgsConstructor
public class FileService {
    private final DigestService digestService;
    
    public String uploadFile(MultipartFile file) {
        // 计算文件 SHA-256 摘要
        String checksum = digestService.digest(
            file.getBytes(), 
            DigestAlgorithm.SHA256
        );
        saveFileWithChecksum(file, checksum);
        return checksum;
    }
    
    public boolean verifyFile(String fileId, byte[] fileData) {
        String savedChecksum = getChecksumFromDB(fileId);
        String currentChecksum = digestService.digest(fileData, DigestAlgorithm.SHA256);
        return savedChecksum.equals(currentChecksum);
    }
}
```

### 场景 3: 快速工具类使用

```java
// 快速计算 MD5
String md5 = DigestUtils.md5("hello world");

// 快速 SHA-256
String sha256 = DigestUtils.sha256("sensitive data");

// 快速 RSA 签名
String signature = SignatureUtils.signRSA(data, privateKey);
boolean valid = SignatureUtils.verifyRSA(data, signature, publicKey);
```

## 最佳实践

### 1. 算法选择建议

**签名场景**：

- ✅ **推荐**: `SHA256_WITH_RSA` 或 `SHA512_WITH_RSA`（安全性高、兼容性好）
- ⚠️ **谨慎**: `SHA256_WITH_ECDSA`（性能好但兼容性略差）
- ❌ **不推荐**: DSA（已逐渐淘汰）

**摘要场景**：

- ✅ **推荐**: `SHA256` 或 `SHA512`（安全性高）
- ⚠️ **谨慎**: `MD5`、`SHA1`（仅用于非安全场景，如文件指纹）
- ✅ **HMAC**: 需要密钥的场景使用 `HMAC_SHA256`

### 2. 密钥管理

> ⚠️ **重要说明**：本组件只负责密钥的**生成和使用**，不负责密钥的**存储管理**。密钥的持久化和保护由业务方根据安全需求自行实现。

**生产环境**：

- ❌ 不要硬编码私钥在代码中
- ✅ 使用配置中心或密钥管理服务（如 AWS KMS、HashiCorp Vault）
- ✅ 私钥加密存储
- ✅ 定期轮换密钥

**密钥长度**：

- RSA: 最小 2048 位，推荐 4096 位
- DSA: 最小 2048 位
- ECDSA: 推荐 256 位（相当于 RSA 3072 位安全强度）

### 3. 性能优化

**避免频繁生成密钥对**：

```java
// ❌ 错误：每次请求都生成
public void badExample() {
    KeyPairInfo keyPair = keyPairService.generateKeyPair(KeyAlgorithm.RSA); // 很慢！
    String signature = signatureService.sign(data, keyPair.getPrivateKey(), ...);
}

// ✅ 正确：初始化时生成一次，缓存复用
@PostConstruct
public void init() {
    this.keyPair = keyPairService.generateKeyPair(KeyAlgorithm.RSA);
}
```

## 异常处理

所有异常都封装为 `SignatureException`，包含以下错误码：

| 错误码                     | 说明     |
|-------------------------|--------|
| `INVALID_KEY`           | 无效的密钥  |
| `INVALID_ALGORITHM`     | 不支持的算法 |
| `SIGN_FAILED`           | 签名失败   |
| `VERIFY_FAILED`         | 验签失败   |
| `DIGEST_FAILED`         | 摘要计算失败 |
| `KEY_GENERATION_FAILED` | 密钥生成失败 |
| `ENCODING_FAILED`       | 编码转换失败 |

```java
try {
    String signature = signatureService.sign(...);
} catch (SignatureException e) {
    log.error("签名失败: code={}, message={}", 
        e.getErrorCode(), e.getMessage());
    // 业务处理...
}
```

## 安全建议

1. **算法选择**：生产环境禁用 MD5 和 SHA-1
2. **密钥保护**：私钥必须加密存储，禁止明文存储
3. **签名字段顺序**：对签名字段按字典序排序，避免顺序不一致导致验签失败
4. **时间戳防重放**：签名数据中包含时间戳，验签时检查时效性
5. **Nonce 防重放**：高安全场景使用随机 Nonce

## 工具类快速参考

### DigestUtils

```java
// MD5
String md5 = DigestUtils.md5("data");

// SHA-256
String sha256 = DigestUtils.sha256("data");

// SHA-512
String sha512 = DigestUtils.sha512("data");

// HMAC-SHA256
String hmac = DigestUtils.hmacSha256("data", "secret-key");

// HMAC-SHA512
String hmac = DigestUtils.hmacSha512("data", "secret-key");
```

### SignatureUtils

```java
// RSA 签名
String signature = SignatureUtils.signRSA(data, privateKeyBase64);

// RSA 验签
boolean valid = SignatureUtils.verifyRSA(data, signature, publicKeyBase64);

// 自定义算法签名
String signature = SignatureUtils.sign(data, privateKey, SignatureAlgorithm.SHA512_WITH_RSA);

// 自定义算法验签
boolean valid = SignatureUtils.verify(data, signature, publicKey, SignatureAlgorithm.SHA512_WITH_RSA);
```

## 常见问题

### Q1: KeyPairService 生成的密钥如何保存？

**A**: 本组件**只负责密钥的生成和使用**，不负责密钥的存储管理。业务方需要根据实际场景选择存储方案：

- **数据库存储**：适合多租户场景，私钥必须加密存储
- **配置中心**：适合单租户，建议使用 Nacos、Apollo 等配置中心
- **密钥管理服务**：生产环境推荐使用 HashiCorp Vault、AWS KMS 等专业 KMS
- **内存缓存**：仅适用于开发/测试环境，重启后密钥丢失

### Q2: RSA 和 ECDSA 如何选择？

**A**:

- **RSA**: 兼容性好、应用广泛、密钥较长
- **ECDSA**: 性能更好、密钥更短、安全性相当

推荐：新项目使用 ECDSA，需兼容旧系统使用 RSA。

### Q3: MD5 可以用吗？

**A**:

- ❌ 不要用于安全场景（已被破解）
- ✅ 可用于非安全场景（如文件指纹、缓存 Key）

### Q4: 签名后数据被篡改会怎样？

**A**: 验签会返回 `false`，业务层应拒绝请求。

### Q5: 如何防止重放攻击？

**A**: 签名数据中包含：

1. **时间戳**: 验签时检查时效性（如 5 分钟内）
2. **Nonce**: 一次性随机数，记录到数据库防重放

## 性能指标

基于 JDK 21 + MacBook Pro (M1) 的性能测试：

| 操作           | 吞吐量 (ops/sec) | 平均耗时     |
|--------------|---------------|----------|
| RSA-2048 签名  | ~5000         | 0.2 ms   |
| RSA-2048 验签  | ~15000        | 0.07 ms  |
| ECDSA-256 签名 | ~20000        | 0.05 ms  |
| SHA-256 摘要   | ~500000       | 0.002 ms |
| HMAC-SHA256  | ~300000       | 0.003 ms |

## 后续优化方向

1. **国密算法**: 支持 SM2/SM3/SM4（使用 Bouncy Castle）
2. **证书支持**: 支持 X.509 证书的签名验签
3. **流式签名**: 支持大文件的流式签名
4. **性能监控**: 暴露签名/验签的 Micrometer 指标
5. **密钥缓存**: 缓存已加载的密钥对象

## 许可证

本组件基于 Apache 2.0 许可证开源。

