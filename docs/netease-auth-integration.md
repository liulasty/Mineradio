# 网易云音乐通用凭证接入方案

> 语言无关 · 端无关 · 跨平台统一授权规范
>
> 本方案定义了一套完整的网易云音乐授权接入体系，覆盖桌面端、移动端、服务端、Web 端。
> 不仅是项目文档，更是一份可被 Java、Go、Python、TypeScript、Kotlin 等多语言复用的**通用 SDK 规范**。

---

## 目录

1. [核心设计原则](#1-核心设计原则)
2. [标准化凭证数据模型](#2-标准化凭证数据模型)
3. [可插拔凭证获取器抽象](#3-可插拔凭证获取器抽象)
4. [跨端会话持久化层](#4-跨端会话持久化层)
5. [独立凭证校验与过期管理器](#5-独立凭证校验与过期管理器)
6. [统一 HTTP 请求拦截器](#6-统一-http-请求拦截器)
7. [风控适配层](#7-风控适配层)
8. [无浏览器兜底方案](#8-无浏览器兜底方案)
9. [七层架构总览](#9-七层架构总览)
10. [完整交互时序](#10-完整交互时序)
11. [多语言实现指南](#11-多语言实现指南)
12. [安全规范](#12-安全规范)
13. [附录](#13-附录)

---

## 1. 核心设计原则

### 1.1 设计目标

| 目标 | 说明 |
|------|------|
| **语言无关** | 核心数据模型用 Protobuf / JSON Schema 定义，任何语言生成绑定代码 |
| **端无关** | 同一套接口在 Electron、Android、iOS、服务端 Python/Go/Java 上行为一致 |
| **可插拔** | 凭证获取、存储、校验均为接口注入，不绑定具体实现 |
| **安全优先** | 凭证必须加密存储，传输必须防泄露，过期自动清理 |
| **降级优先** | 支持多渠道获取（WebView / 二维码 / 手动导入），任一渠道失败自动降级 |

### 1.2 七层架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                     应用层 (Application)                            │
│        使用 SDK 的应用代码 — 不关心底层凭证管理细节                    │
├─────────────────────────────────────────────────────────────────────┤
│                     7 层核心能力                                     │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ ① 凭证    │  │ ② 凭证    │  │ ③ 会话    │  │ ④ 凭证    │           │
│  │ 数据模型   │  │ 获取器    │  │ 持久化    │  │ 校验器    │           │
│  │ (跨语言)   │  │ (可插拔)  │  │ (跨端)    │  │ (独立)    │           │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
│       │              │              │              │                │
│  ┌────┴─────┐  ┌────┴─────┐  ┌────┴─────┐                        │
│  │ ⑤ HTTP    │  │ ⑥ 风控    │  │ ⑦ 无界面   │                        │
│  │ 拦截器     │  │ 适配层    │  │ 兜底方案   │                        │
│  └──────────┘  └──────────┘  └──────────┘                        │
├─────────────────────────────────────────────────────────────────────┤
│                     基础设施层 (Infrastructure)                      │
│        网络 / 存储 / 加密 / 平台 WebView                            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 标准化凭证数据模型

### 2.1 问题

现有方案只传递裸 Cookie 字符串（`MUSIC_U=xxx; __csrf=yyy`），没有结构化定义，导致：

- 跨语言无法反序列化（Java 拿到的是一段字符串而非对象）
- 无法区分核心鉴权字段与设备字段
- 无法附加元信息（过期时间、来源平台）
- 需要序列化时必须各自重写一遍解析逻辑

### 2.2 Protobuf 定义（权威模型）

```protobuf
syntax = "proto3";

package netease.auth;
option go_package = "github.com/example/netease-auth-sdk/proto";
option java_package = "com.netease.auth.sdk";

import "google/protobuf/timestamp.proto";

// ─── 网易云音乐统一凭证 ──────────────────────────────────────────────

message Credential {
  // 核心鉴权字段（必须完整保存才能恢复登录态）
  CoreFields core = 1;

  // 设备标识字段（用于接口风控验证）
  DeviceFields device = 2;

  // 扩展可选字段（非关键，但保留有助于识别会话）
  OptionalFields optional = 3;

  // 凭证过期时间戳（UTC），由校验器更新；0 表示未知
  google.protobuf.Timestamp expire_at = 4;

  // 凭证来源平台
  CredentialSource source = 5;

  // 原始 Cookie Header（保留原始格式以兼容旧系统）
  string raw_cookie_header = 6;

  // 凭证获取时间
  google.protobuf.Timestamp acquired_at = 7;

  // 最后验证时间
  google.protobuf.Timestamp last_verified_at = 8;
}

// ─── 核心鉴权字段 ────────────────────────────────────────────────────

message CoreFields {
  // 登录态核心标识 — 存在即表示登录成功（必需）
  string music_u = 1;          // MUSIC_U

  // CSRF 防护令牌（必需）
  string csrf = 2;             // __csrf

  // 备用鉴权 token（部分接口降级使用）
  string music_a = 3;          // MUSIC_A

  // "记住登录"标记
  string remember_me = 4;      // __remember_me
}

// ─── 设备标识字段 ────────────────────────────────────────────────────

message DeviceFields {
  // 设备/浏览器唯一标识
  string nmtid = 1;            // NMTID

  // 网易通用设备 ID（新）
  string netease_nuid = 2;     // _ntes_nuid

  // 网易通用设备 ID（旧）
  string netease_nnid = 3;     // _ntes_nnid

  // 会话 ID
  string jsessionid_wyyy = 4;  // JSESSIONID-WYYY

  // WebView 环境标识
  string wevnsm = 5;           // WEVNSM

  // 无线设备 ID
  string wnmcid = 6;           // WNMCID
}

// ─── 扩展可选字段 ────────────────────────────────────────────────────

message OptionalFields {
  // 用户昵称（缓存，减少 API 调用）
  string nickname = 1;

  // 用户 ID（缓存）
  int64 user_id = 2;

  // VIP 类型
  int32 vip_type = 3;

  // 头像 URL（缓存）
  string avatar_url = 4;

  // 自定义扩展 KV
  map<string, string> extensions = 16;
}

// ─── 凭证来源 ────────────────────────────────────────────────────────

enum CredentialSource {
  SOURCE_UNSPECIFIED = 0;
  SOURCE_WEB = 1;              // 网页端（music.163.com WebView 抓取）
  SOURCE_ANDROID = 2;          // Android 客户端
  SOURCE_IOS = 3;              // iOS 客户端
  SOURCE_PC_CLIENT = 4;        // PC 客户端
  SOURCE_QR_API = 5;           // 无界面二维码 API
  SOURCE_MANUAL_IMPORT = 6;    // 手动导入
}

// ─── 凭证校验结果 ─────────────────────────────────────────────────────

message CredentialStatus {
  bool valid = 1;
  string nickname = 2;
  int64 user_id = 3;
  int32 vip_type = 4;
  int32 vip_level = 5;         // 0=none, 1=vip, 2=svip
  bool is_vip = 6;
  bool is_svip = 7;
  string error_code = 8;       // 失效时填写：EXPIRED / INVALID / NETWORK_ERROR
  string error_message = 9;
  google.protobuf.Timestamp checked_at = 10;
}
```

### 2.3 JSON Schema（轻量序列化）

对于无需 Protobuf 的场景（如 Web 前端、简单的 Python 脚本），使用此 JSON Schema：

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "NeteaseMusicCredential",
  "type": "object",
  "required": ["core", "source"],
  "properties": {
    "core": {
      "type": "object",
      "description": "核心鉴权字段",
      "required": ["musicU"],
      "properties": {
        "musicU":    { "type": "string", "description": "MUSIC_U — 登录态核心标识" },
        "csrf":      { "type": "string", "description": "__csrf — CSRF 令牌" },
        "musicA":    { "type": "string", "description": "MUSIC_A — 备用鉴权" },
        "rememberMe": { "type": "string", "description": "__remember_me — 记住登录" }
      }
    },
    "device": {
      "type": "object",
      "description": "设备标识字段",
      "properties": {
        "nmtid":      { "type": "string" },
        "neteaseNuid": { "type": "string" },
        "neteaseNnid": { "type": "string" },
        "jsessionidWyyy": { "type": "string" },
        "wevnsm":     { "type": "string" },
        "wnmcid":     { "type": "string" }
      }
    },
    "optional": {
      "type": "object",
      "description": "扩展可选字段",
      "properties": {
        "nickname":  { "type": "string" },
        "userId":    { "type": "integer" },
        "vipType":   { "type": "integer" },
        "avatarUrl": { "type": "string", "format": "uri" }
      }
    },
    "expireAt":       { "type": "integer", "description": "过期 Unix 时间戳 (秒)" },
    "acquiredAt":     { "type": "integer", "description": "获取时间 Unix 时间戳" },
    "lastVerifiedAt": { "type": "integer", "description": "最后验证时间戳" },
    "source": {
      "type": "string",
      "enum": ["web", "android", "ios", "pcclient", "qr_api", "manual_import"]
    },
    "rawCookieHeader": {
      "type": "string",
      "description": "原始 Cookie Header 字符串（兼容旧系统）"
    }
  }
}
```

### 2.4 示例实例

```json
{
  "core": {
    "musicU": "DEB07AB6_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "csrf": "a1b2c3d4e5f6",
    "rememberMe": "true"
  },
  "device": {
    "nmtid": "0x12345678_abcdef1234567890abcdef",
    "neteaseNuid": "a1b2c3d4e5f6g7h8i9j0",
    "wevnsm": "WNMCID_xxxxxxxx"
  },
  "optional": {
    "nickname": "音乐爱好者",
    "userId": 123456789,
    "vipType": 11,
    "avatarUrl": "https://p1.music.126.net/xxx/avatar.jpg"
  },
  "expireAt": 1750000000,
  "acquiredAt": 1749913600,
  "lastVerifiedAt": 1749913600,
  "source": "web",
  "rawCookieHeader": "MUSIC_U=DEB07AB6_...; __csrf=a1b2c3d4e5f6; NMTID=0x12345678_...; __remember_me=true"
}
```

### 2.5 各语言绑定示意

```java
// Java — 从 Protobuf 生成
Credential cred = Credential.parseFrom(bytes);
String musicU = cred.getCore().getMusicU();
CredentialSource source = cred.getSource();
```

```go
// Go — 从 Protobuf 生成
cred := &neteaseauth.Credential{}
proto.Unmarshal(bytes, cred)
musicU := cred.Core.MusicU
```

```python
# Python — JSON 反序列化
cred = NeteaseCredential.from_json(data)
music_u = cred.core.music_u
```

```typescript
// TypeScript — 类型化
const cred: NeteaseCredential = JSON.parse(data);
const musicU = cred.core.musicU;
```

---

## 3. 可插拔凭证获取器抽象

### 3.1 问题

现有方案只有一种获取方式（Electron WebView），且没有抽象接口。不同平台（移动端、服务端、Web）各自需要重新实现完整的获取逻辑。

### 3.2 顶层接口定义（语言无关伪代码）

```
interface CredentialProvider {
  // ── 生命周期 ──────────────────────────────────────────────

  // 唤起登录流程，返回完整凭证
  // 可能涉及 WebView 弹出、二维码扫描、或从本地读取已有凭证
  // Returns: Credential | null（用户取消时返回 null）
  // Throws: LoginCancelledError, LoginTimeoutError, LoginFailedError
  async login(): Credential | null

  // ── 读取 ──────────────────────────────────────────────────

  // 读取当前已有凭证（不从远程获取，仅从本地缓存读取）
  // Returns: Credential | null
  getCurrent(): Credential | null

  // ── 校验 ──────────────────────────────────────────────────

  // 校验当前凭证是否仍然有效
  // 内部通过调用 /user/account 或 login_status 检测
  // Returns: CredentialStatus
  async checkValid(): CredentialStatus

  // ── 清理 ──────────────────────────────────────────────────

  // 清除当前会话（本地 + 远程登出）
  async logout(): void

  // ── 事件 ──────────────────────────────────────────────────

  // 注册凭证变化回调（登录成功 / 过期 / 登出时触发）
  onCredentialChange(callback: (cred: Credential | null) => void): void
}
```

### 3.3 多端实现矩阵

| 实现 | 平台 | 依赖 | 界面 | 适用场景 |
|------|------|------|------|----------|
| `DesktopWebViewProvider` | Electron / CEF / WebView2 | Chromium WebView | 有 | Windows/Mac/Linux 桌面应用 |
| `MobileWebViewProvider` | Android / iOS | 系统 WebView | 有 | 移动 App |
| `QrCodeApiProvider` | 全平台 | HTTP 客户端 | 有（需展示二维码） | 有显示屏但不便使用 WebView 的场景 |
| `HeadlessBrowserProvider` | 服务端 | Playwright / Puppeteer | 无（可由远端设备扫码） | 后端服务、自动化脚本 |
| `TokenImportProvider` | 全平台 | 用户输入 | 有（粘贴框） | 调试、迁移 |
| `FileProvider` | 全平台 | 文件系统 | 无 | 自动化、CI/CD 测试 |

### 3.4 各实现规范

#### DesktopWebViewProvider

```typescript
// 输入参数
interface DesktopWebViewOptions {
  // WebView 窗口配置
  windowTitle?: string;           // 默认: "网易云音乐登录"
  windowWidth?: number;           // 默认: 940
  windowHeight?: number;          // 默认: 760
  minWidth?: number;              // 默认: 780
  minHeight?: number;             // 默认: 580
  backgroundColor?: string;       // 默认: "#111111"
  loginUrl?: string;              // 默认: "https://music.163.com/#/login"
  sessionPartition?: string;      // 默认: "persist:netease-login"

  // Cookie 采集配置
  pollIntervalMs?: number;        // 轮询间隔, 默认: 1200
  cookieDomains?: string[];       // 采集的域名, 默认: ["163.com", "music.163.com"]
  requiredCookies?: string[];     // 成功标志, 默认: ["MUSIC_U"]
  cookiePriority?: string[];      // 排序优先级

  // 超时
  timeoutMs?: number;             // 登录超时, 默认: 300000 (5分钟)

  // 行为控制
  autoClickLogin?: boolean;       // 是否自动点击"登录"按钮, 默认: true
}

// 实现行为
// 1. 创建独立 session 分区的 WebView / BrowserWindow
// 2. 导航到 loginUrl
// 3. 以 pollIntervalMs 间隔轮询 session cookie
// 4. 检测到 requiredCookies 中所有字段存在 → 采集 → 返回 Credential
// 5. 超时或用户关闭窗口 → 返回 null / 抛出异常
```

#### QrCodeApiProvider

```typescript
// 输入参数
interface QrCodeApiOptions {
  // NeteaseCloudMusicApi 端点
  apiBaseUrl: string;             // 必填, 如 "http://localhost:3000"
  pollIntervalMs?: number;        // 轮询间隔, 默认: 2000
  timeoutMs?: number;             // 超时, 默认: 120000 (2分钟)

  // UI 回调（可选，用于在屏幕上显示二维码）
  onQrCodeGenerated?: (base64Image: string, qrUrl: string) => void;
  onStatusChange?: (status: QrStatus) => void;
}

enum QrStatus {
  WAITING = 'waiting',            // 801 - 等待扫码
  SCANNED = 'scanned',            // 802 - 已扫待确认
  CONFIRMED = 'confirmed',        // 803 - 授权成功
  EXPIRED = 'expired',            // 800 - 二维码过期
}

// 实现行为
// 1. GET {apiBaseUrl}/login/qr/key → 获取 unikey
// 2. GET {apiBaseUrl}/login/qr/create?key=xxx → 获取二维码 base64
// 3. 回调 onQrCodeGenerated(base64, qrUrl) → 由调用方展示
// 4. 每 2s 轮询 GET {apiBaseUrl}/login/qr/check?key=xxx
// 5. 803 + 提取 cookie → 结构化 → 返回 Credential
// 6. 801/802 → 更新状态, 800 → 超时/刷新
```

#### HeadlessBrowserProvider

```typescript
// 输入参数
interface HeadlessBrowserOptions {
  // 浏览器控制
  browserType: 'playwright' | 'puppeteer' | 'selenium';
  browserExecutablePath?: string;
  headless?: boolean;             // 默认: true
  proxy?: string;                 // 代理配置

  // 登录策略
  strategy: 'qr_scan' | 'phone_login' | 'email_login';
  
  // 手机号/邮箱登录（仅在 strategy = phone_login | email_login 时需要）
  phone?: string;
  email?: string;
  password?: string;

  // QR 生成回调
  onQrImage?: (base64: string) => void;

  // 超时
  timeoutMs?: number;             // 默认: 180000 (3分钟)
}

// 实现行为
// 1. 启动无头浏览器
// 2. 导航到 https://music.163.com/#/login
// 3. 根据 strategy 执行登录流程
//    - qr_scan: 等待二维码 → 回调 onQrImage → 等待扫码
//    - phone_login: 填入手机号 → 密码/验证码登录
// 4. 从浏览器上下文中提取所有 163.com/music.163.com Cookie
// 5. 结构化 → 返回 Credential
// 6. 关闭浏览器
```

#### TokenImportProvider

```typescript
// 输入参数
interface TokenImportOptions {
  // 用户手动粘贴的原始 Cookie 字符串
  rawCookieHeader: string;

  // 来源标记
  source?: CredentialSource;      // 默认: manual_import
}

// 实现行为
// 1. 解析原始 Cookie Header
// 2. 提取 MUSIC_U → core.musicU
// 3. 提取 __csrf → core.csrf
// 4. 提取其余设备字段
// 5. 校验有效性（可选，调用 /user/account）
// 6. 返回 Credential 或抛出校验失败异常
```

### 3.5 Provider 工厂与自动降级

```typescript
// ── 工厂 ─────────────────────────────────────────────────────

interface ProviderFactoryOptions {
  // 可用 Provider 列表（按优先级排列）
  providers: CredentialProvider[];
}

class CredentialProviderFactory {
  // 按优先级依次尝试，任一成功即返回
  static async tryLogin(providers: CredentialProvider[]): Promise<Credential | null> {
    for (const provider of providers) {
      try {
        const cred = await provider.login();
        if (cred) return cred;
      } catch (e) {
        console.warn(`Provider ${provider.constructor.name} failed:`, e.message);
        continue; // 降级到下一 Provider
      }
    }
    return null; // 全部失败
  }
}

// ── 使用示例 ──────────────────────────────────────────────────

// 桌面端：优先 WebView，降级到二维码
const cred = await CredentialProviderFactory.tryLogin([
  new DesktopWebViewProvider(webViewOptions),
  new QrCodeApiProvider(qrOptions),
  new TokenImportProvider(manualOptions),
]);

// 服务端：优先二维码 API，降级到无头浏览器
const cred = await CredentialProviderFactory.tryLogin([
  new QrCodeApiProvider(qrOptions),
  new HeadlessBrowserProvider(headlessOptions),
]);
```

---

## 4. 跨端会话持久化层

### 4.1 问题

当前方案使用明文 `.cookie` 文件，无加密、无统一存储接口、不可跨端共享。

### 4.2 分层存储接口

```
interface CredentialStore {
  // ── 基础操作 ──────────────────────────────────────────

  // 保存凭证（底层会自动序列化 + 加密）
  async save(credential: Credential): void

  // 读取凭证（自动解密 + 反序列化）
  async load(): Credential | null

  // 删除凭证
  async delete(): void

  // ── 状态查询 ──────────────────────────────────────────

  // 检查是否存在凭证
  async exists(): boolean

  // 获取存储元信息
  async getMetadata(): StoreMetadata
}

interface StoreMetadata {
  storageType: string;       // 'file' | 'keychain' | 'redis' | 'shared_prefs'
  encrypted: boolean;        // 是否加密存储
  lastWriteAt: number;       // 最后写入时间戳
  credentialSource?: string; // 凭证来源
}
```

### 4.3 各端实现

| 实现 | 平台 | 后端 | 加密 | 说明 |
|------|------|------|------|------|
| `EncryptedFileStore` | 桌面端 (Electron/CEF) | 文件系统 | AES-256-GCM | 密钥派生自机器指纹+应用密钥 |
| `SystemKeychainStore` | 桌面端 (macOS/Windows/Linux) | 系统密钥链 | 系统级 | macOS Keychain / Windows Credential Manager / libsecret |
| `SharedPreferencesStore` | Android | 沙盒 SharedPreferences | EncryptedSharedPreferences | AndroidX Security |
| `KeychainStore` | iOS | iOS Keychain | 系统级 | kSecClassGenericPassword |
| `RedisStore` | 服务端 | Redis | AES-256-GCM | 集群共享，可选 TTL |
| `DatabaseStore` | 服务端 | MySQL/PostgreSQL | AES-256-GCM | 持久化 + 关联业务表 |
| `MemoryStore` | 全平台（测试/临时） | 内存 Map | 无 | 应用重启即丢失 |

### 4.4 加密规范

```yaml
加密算法: AES-256-GCM
密钥长度: 256 bits
IV 长度: 96 bits (12 bytes)
认证标签: 128 bits (16 bytes)
密钥派生: HKDF-SHA256 (用于从主密钥派生出存储密钥)
序列化格式: Base64(IV + ciphertext + auth_tag)
```

**加密流程：**

```
明文 Credential (Protobuf/JSON)
        │
        ▼
  ┌─────────────┐
  │ AES-256-GCM  │ ← 密钥: 应用主密钥 (或系统密钥链托管)
  │ Encrypt      │ ← IV: 随机 12 bytes
  └──────┬──────┘
         │
         ▼
  Base64(IV || Ciphertext || AuthTag)
         │
         ▼
  写入存储后端 (文件/Redis/SharedPrefs)
```

**Electron 推荐方案：使用 `safeStorage` API**

```javascript
const { safeStorage } = require('electron');

// 加密
const plaintext = JSON.stringify(credential);
const encrypted = safeStorage.encryptString(plaintext);
// 写入文件
fs.writeFileSync(credentialPath, encrypted);

// 解密
const encrypted = fs.readFileSync(credentialPath);
const plaintext = safeStorage.decryptString(encrypted);
const credential = JSON.parse(plaintext);
```

**服务端推荐方案：环境变量主密钥 + AES-256-GCM**

### 4.5 多 Store 级联策略

```typescript
class CascadeStore implements CredentialStore {
  private stores: CredentialStore[];

  constructor(stores: CredentialStore[]) {
    this.stores = stores;
  }

  async save(credential: Credential): Promise<void> {
    // 写入所有 Store（冗余保存）
    for (const store of this.stores) {
      await store.save(credential).catch(e => 
        console.warn(`Store ${store.constructor.name} save failed:`, e.message));
    }
  }

  async load(): Promise<Credential | null> {
    // 按优先级读取，任一成功即返回
    for (const store of this.stores) {
      try {
        const cred = await store.load();
        if (cred) return cred;
      } catch (e) {
        console.warn(`Store ${store.constructor.name} load failed:`, e.message);
        continue;
      }
    }
    return null;
  }

  async delete(): Promise<void> {
    for (const store of this.stores) {
      await store.delete().catch(() => {});
    }
  }
}

// ── 使用示例 ──────────────────────────────────────────────────

// 桌面端：Keychain 为主，加密文件为备份
const store = new CascadeStore([
  new SystemKeychainStore('com.app.netease'),
  new EncryptedFileStore(path.join(userDataDir, '.credential')),
]);

// 服务端：Redis 为主，数据库为持久化备份
const store = new CascadeStore([
  new RedisStore(redisClient, { keyPrefix: 'netease:cred:' }),
  new DatabaseStore(dbPool, { tableName: 'netease_credentials' }),
]);
```

---

## 5. 独立凭证校验与过期管理器

### 5.1 问题

当前校验逻辑（`getLoginInfo()`）耦合在本地 HTTP 服务里，后端程序无法复用。

### 5.2 校验器接口

```typescript
interface CredentialValidator {
  // 校验凭证是否有效
  // 内部调用 /user/account 或 login_status
  async validate(credential: Credential): CredentialStatus

  // 解析 API 响应中的错误
  parseError(httpStatus: number, responseBody: any): ValidationError | null
}
```

### 5.3 过期管理器

```typescript
interface ExpiryManagerConfig {
  validator: CredentialValidator;
  store: CredentialStore;
  
  // 后台自检间隔（毫秒），0 表示不启用后台检查
  backgroundCheckIntervalMs?: number;   // 默认: 300000 (5分钟)
  
  // 过期前多久触发预刷新（毫秒），0 表示不预刷新
  preemptiveRefreshThresholdMs?: number; // 默认: 3600000 (1小时)
  
  // 校验失败时是否自动清除凭证
  autoClearOnInvalid?: boolean;         // 默认: true
}

class ExpiryManager {
  constructor(config: ExpiryManagerConfig);

  // 启动后台自检定时器
  start(): void;

  // 停止后台自检定时器
  stop(): void;

  // 手动触发一次校验，更新 lastVerifiedAt
  async verifyNow(): CredentialStatus;

  // 获取当前状态（含缓存，不触发网络请求）
  getCachedStatus(): CredentialStatus;

  // 注册过期事件回调
  onExpired(callback: (credential: Credential) => void): void;

  // 注册即将过期回调
  onExpiring(callback: (credential: Credential) => void): void;

  // 注册状态变化回调
  onStatusChange(callback: (status: CredentialStatus) => void): void;
}
```

### 5.4 失效判定标准

```python
def is_credential_invalid(http_status: int, response_body: dict) -> bool:
    """判断 API 响应是否表示凭证失效"""

    # HTTP 401 → 未授权
    if http_status == 401:
        return True

    # 解析 code 字段（可能出现在 body.code / body.data.code）
    code = (response_body.get('code')
            or (response_body.get('data') or {}).get('code')
            or 200)

    # 解析 message 字段
    message = (response_body.get('message')
               or response_body.get('msg')
               or '')

    # 301 / 401 → 未登录
    if code in (301, 401):
        return True

    # 消息包含"未登录/需要登录/请先登录"且 code >= 300
    invalid_keywords = ['未登录', '需要登录', '请先登录', 'login']
    has_keyword = any(kw in message for kw in invalid_keywords)
    if has_keyword and code >= 300:
        return True

    return False
```

### 5.5 后台自检实现

```typescript
class ExpiryManagerImpl implements ExpiryManager {
  private timer: NodeJS.Timeout | null = null;
  private cachedStatus: CredentialStatus | null = null;

  start(): void {
    if (this.config.backgroundCheckIntervalMs <= 0) return;

    this.timer = setInterval(async () => {
      const credential = await this.config.store.load();
      if (!credential) return;

      const status = await this.config.validator.validate(credential);
      this.cachedStatus = status;
      this.onStatusChangeCallbacks.forEach(cb => cb(status));

      if (!status.valid) {
        this.onExpiredCallbacks.forEach(cb => cb(credential));
        if (this.config.autoClearOnInvalid) {
          await this.config.store.delete();
        }
      } else if (credential.expireAt && credential.expireAt > 0) {
        const remaining = credential.expireAt - Date.now() / 1000;
        if (remaining > 0
            && remaining < (this.config.preemptiveRefreshThresholdMs / 1000)) {
          this.onExpiringCallbacks.forEach(cb => cb(credential));
        }
      }
    }, this.config.backgroundCheckIntervalMs);
  }

  stop(): void {
    if (this.timer) {
      clearInterval(this.timer);
      this.timer = null;
    }
  }
}
```

---

## 6. 统一 HTTP 请求拦截器

### 6.1 问题

Cookie 组装逻辑分散在 Electron 主进程中，其他语言/平台需要各自实现一套，且容易遗漏过滤属性、排序等细节。

### 6.2 拦截器接口

```typescript
interface CredentialInterceptor {
  // 为 HTTP 请求附加认证头
  // 从 Credential 对象组装标准 Cookie Header
  async apply(request: HttpRequest): Promise<HttpRequest>

  // 检查响应是否需要清除凭证
  async handleResponse(response: HttpResponse): Promise<void>
}
```

### 6.3 Cookie Header 构建工具

```typescript
class NeteaseCookieBuilder {
  // 凭证优先级顺序
  static readonly PRIORITY_ORDER = [
    'MUSIC_U', '__csrf', 'NMTID', 'MUSIC_A',
    '__remember_me', '_ntes_nuid', '_ntes_nnid',
    'WEVNSM', 'WNMCID', 'JSESSIONID-WYYY',
  ];

  // 需过滤的 HTTP 属性名
  static readonly ATTRIBUTE_FILTER = new Set([
    'path', 'domain', 'expires', 'max-age',
    'maxage', 'samesite', 'secure', 'httponly',
  ]);

  /**
   * 从结构化 Credential 构建标准 Cookie Header
   *
   * 输入: Credential { core: { musicU: 'xxx', csrf: 'yyy' }, device: { nmtid: 'zzz' } }
   * 输出: "MUSIC_U=xxx; __csrf=yyy; NMTID=zzz"
   */
  static build(credential: Credential): string {
    const pairs: [string, string][] = [];

    // 1. 从结构化字段提取
    if (credential.core?.musicU)  pairs.push(['MUSIC_U', credential.core.musicU]);
    if (credential.core?.csrf)    pairs.push(['__csrf', credential.core.csrf]);
    if (credential.core?.musicA)  pairs.push(['MUSIC_A', credential.core.musicA]);
    if (credential.core?.rememberMe) pairs.push(['__remember_me', credential.core.rememberMe]);

    if (credential.device?.nmtid)     pairs.push(['NMTID', credential.device.nmtid]);
    if (credential.device?.neteaseNuid) pairs.push(['_ntes_nuid', credential.device.neteaseNuid]);
    if (credential.device?.neteaseNnid) pairs.push(['_ntes_nnid', credential.device.neteaseNnid]);
    if (credential.device?.wevnsm)    pairs.push(['WEVNSM', credential.device.wevnsm]);
    if (credential.device?.wnmcid)    pairs.push(['WNMCID', credential.device.wnmcid]);
    if (credential.device?.jsessionidWyyy) pairs.push(['JSESSIONID-WYYY', credential.device.jsessionidWyyy]);

    // 2. 按优先级排序
    const priority = NeteaseCookieBuilder.PRIORITY_ORDER;
    pairs.sort((a, b) => {
      const ia = priority.indexOf(a[0]);
      const ib = priority.indexOf(b[0]);
      return (ia === -1 ? 999 : ia) - (ib === -1 ? 999 : ib);
    });

    // 3. 序列化
    return pairs
      .filter(([_, v]) => v != null && v !== '')
      .map(([k, v]) => `${k}=${encodeURIComponent(v)}`)
      .join('; ');
  }

  /**
   * 从原始 Cookie Header 解析为结构化 Credential
   */
  static parse(rawHeader: string): Partial<Credential> {
    const obj: Record<string, string> = {};
    rawHeader.split(';').forEach(part => {
      const idx = part.indexOf('=');
      if (idx > 0) {
        const key = part.slice(0, idx).trim();
        const val = part.slice(idx + 1).trim();
        if (!NeteaseCookieBuilder.ATTRIBUTE_FILTER.has(key.toLowerCase())) {
          obj[key] = decodeURIComponent(val);
        }
      }
    });

    return {
      core: {
        musicU: obj['MUSIC_U'] || '',
        csrf: obj['__csrf'] || '',
        musicA: obj['MUSIC_A'] || '',
        rememberMe: obj['__remember_me'] || '',
      },
      device: {
        nmtid: obj['NMTID'] || '',
        neteaseNuid: obj['_ntes_nuid'] || '',
        neteaseNnid: obj['_ntes_nnid'] || '',
        wevnsm: obj['WEVNSM'] || '',
        wnmcid: obj['WNMCID'] || '',
        jsessionidWyyy: obj['JSESSIONID-WYYY'] || '',
      },
      rawCookieHeader: rawHeader,
    };
  }
}
```

### 6.4 各语言/HTTP 客户端适配

```typescript
// ── TypeScript: Axios 拦截器 ────────────────────────────────

const interceptor = new CredentialInterceptorImpl(store, expiryManager);

axios.interceptors.request.use(async (config) => {
  const credential = await store.load();
  if (credential) {
    config.headers['Cookie'] = NeteaseCookieBuilder.build(credential);
    config.headers['User-Agent'] = getUserAgentForSource(credential.source);
    config.headers['Referer'] = 'https://music.163.com/';
  }
  return config;
});

axios.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response && isCredentialInvalid(error.response.status, error.response.data)) {
      await store.delete();
      expiryManager.notifyExpired();
    }
    return Promise.reject(error);
  }
);
```

```python
# ── Python: requests Session 适配 ──────────────────────────

import requests
from typing import Optional
from netease_auth import Credential, CookieBuilder, is_credential_invalid

class NeteaseAuthSession(requests.Session):
    """自动注入 Cookie 的 requests Session"""

    def __init__(self, credential: Optional[Credential] = None):
        super().__init__()
        self._credential = credential
        self.headers.update({
            'Referer': 'https://music.163.com/',
        })

    def set_credential(self, credential: Credential):
        self._credential = credential

    def request(self, method, url, **kwargs):
        if self._credential:
            kwargs.setdefault('headers', {})
            kwargs['headers']['Cookie'] = CookieBuilder.build(self._credential)
            kwargs['headers']['User-Agent'] = self._resolve_ua()
        return super().request(method, url, **kwargs)

    def _resolve_ua(self) -> str:
        if not self._credential:
            return 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...'
        # 根据来源切换 UA，见 §7 风控适配
        return get_user_agent_for_source(self._credential.source)
```

```go
// ── Go: http.Client 包装 ────────────────────────────────────

package neteaseauth

import "net/http"

type AuthTransport struct {
    inner     http.RoundTripper
    credential *Credential
}

func (t *AuthTransport) RoundTrip(req *http.Request) (*http.Response, error) {
    if t.credential != nil {
        cookie := BuildCookieHeader(t.credential)
        req.Header.Set("Cookie", cookie)
        req.Header.Set("User-Agent", ResolveUserAgent(t.credential.Source))
        req.Header.Set("Referer", "https://music.163.com/")
    }
    return t.inner.RoundTrip(req)
}

func NewAuthClient(cred *Credential) *http.Client {
    return &http.Client{
        Transport: &AuthTransport{
            inner:      http.DefaultTransport,
            credential: cred,
        },
    }
}
```

```java
// ── Java: OkHttp 拦截器 ──────────────────────────────────────

public class NeteaseAuthInterceptor implements Interceptor {
    private final CredentialStore store;

    public NeteaseAuthInterceptor(CredentialStore store) {
        this.store = store;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        Credential cred = store.load();
        Request.Builder builder = chain.request().newBuilder();

        if (cred != null) {
            String cookie = NeteaseCookieBuilder.build(cred);
            builder.header("Cookie", cookie);
            builder.header("User-Agent", resolveUserAgent(cred.getSource()));
            builder.header("Referer", "https://music.163.com/");
        }

        Response response = chain.proceed(builder.build());

        if (isCredentialInvalid(response.code(), response.body())) {
            store.delete();
            // notify expiry manager
        }

        return response;
    }
}
```

---

## 7. 风控适配层

### 7.1 问题

网易云音乐的部分高级接口会校验**设备指纹**和**请求特征**。仅携带 Cookie 是不够的——网页端凭据与 App 端凭据能访问的接口范围不同。纯网页 Cookie 调用某些接口可能触发设备校验风控。

### 7.2 来源与能力矩阵

| 凭证来源 | 对应 UA | 可用接口范围 | 风控级别 | 限制 |
|----------|---------|-------------|----------|------|
| `web` (music.163.com) | `Mozilla/5.0 ... Chrome/...` | 歌单、搜索、评论、歌词 | 低 | 部分高敏接口可能拦截 |
| `android` | `NeteaseMusic/... Android/...` | 全部（含高码率、VIP） | 中 | 需要补齐设备 ID |
| `ios` | `NeteaseMusic/... iOS/...` | 全部（含高码率、VIP） | 中 | 需要补齐设备 ID |
| `pcclient` | `Mozilla/5.0 ... NeteaseMusic/...` | 全部 | 低-中 | 需模拟客户端请求特征 |
| `qr_api` | `Mozilla/5.0 ...` | 同 `web` | 低 | 降级方案，只保证基础能力 |

### 7.3 请求头抹平策略

```typescript
class RequestHeaderNormalizer {
  /**
   * 根据凭证来源，生成适配的请求头集合
   */
  static normalize(credential: Credential): Record<string, string> {
    const headers: Record<string, string> = {
      'Referer': 'https://music.163.com/',
    };

    switch (credential.source) {
      case 'web':
        headers['User-Agent'] = this.WEB_UA;
        break;

      case 'android':
        headers['User-Agent'] = this.ANDROID_UA;
        headers['X-Real-IP'] = this.randomChinaIP();
        // Android App 需额外补齐设备参数
        if (credential.device?.nmtid) {
          headers['X-Device-Id'] = credential.device.nmtid;
        }
        break;

      case 'ios':
        headers['User-Agent'] = this.IOS_UA;
        headers['X-Real-IP'] = this.randomChinaIP();
        break;

      case 'pcclient':
        headers['User-Agent'] = this.PC_CLIENT_UA;
        headers['X-Real-IP'] = this.randomChinaIP();
        break;

      default:
        headers['User-Agent'] = this.WEB_UA;
    }

    return headers;
  }

  // ── 示例 UA（实际使用时替换为最新版本号）──

  // Web 端
  static readonly WEB_UA =
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) '
    + 'AppleWebKit/537.36 (KHTML, like Gecko) '
    + 'Chrome/125.0.0.0 Safari/537.36';

  // Android 客户端
  static readonly ANDROID_UA =
    'NeteaseMusic/9.1.5 (Android 14; Xiaomi 14)';

  // iOS 客户端
  static readonly IOS_UA =
    'NeteaseMusic/9.1.5 (iOS 18.0; iPhone16,2)';

  // PC 客户端
  static readonly PC_CLIENT_UA =
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) '
    + 'AppleWebKit/537.36 (KHTML, like Gecko) '
    + 'NeteaseMusic/2.10.11 Chrome/91.0.4472.124 Safari/537.36';

  // ── 辅助 ──

  private static randomChinaIP(): string {
    // 返回一个随机国内 IP，用于部分对 IP 归属地敏感的接口
    const prefixes = ['1.2.4.', '39.134.', '42.80.', '58.42.', '101.80.', '116.228.', '120.36.', '183.6.'];
    const prefix = prefixes[Math.floor(Math.random() * prefixes.length)];
    return prefix + Math.floor(Math.random() * 255);
  }
}
```

### 7.4 接口访问策略

```typescript
// 不同来源可安全调用的接口范围
const SOURCE_API_MATRIX = {
  'web': [
    '/api/v1/playlist/detail',
    '/api/v1/search',
    '/api/v1/artist/detail',
    '/api/v1/lyric',
    '/api/v1/comment/list',
    '/api/v1/song/detail',
    '/api/v1/user/playlist',
    '/api/playlist/track/all',
  ],
  'android': [
    // 全部 web 接口 +
    '/api/v2/music/cloud',        // 云盘
    '/api/v1/song/pc',            // 高码率（需 VIP）
    '/api/v1/member/level',       // 会员等级
    '/api/v2/user/info',          // 详细用户信息
  ],
  'ios': [
    // 同 android
  ],
  'pcclient': [
    // 全部
  ],
};
```

---

## 8. 无浏览器兜底方案

### 8.1 问题

WebView 方案强依赖 GUI 环境。纯后端服务、Docker 容器、CI/CD 流水线无法弹出浏览器窗口。一套真正通用的方案必须有不依赖界面的凭证获取路径。

### 8.2 全场景获取路径矩阵

```
                    ┌── 有界面 ──┬── WebView 官方登录页
                    │            └── 内嵌 iframe 二维码
获取凭证 ───┬── 人机交互 ──┤
            │            └── 无界面 ──┬── NeteaseCloudMusicApi QR 接口
            │                         └── 粘贴已导出 Cookie 字符串
            │
            └── 自动 ────┬── 从持久化存储读取（已有凭证）
                         └── Headless Browser 自动登录
```

### 8.3 Provider 自动降级链路

```
tryDesktopWebView()          ← Electron 桌面端
    ↓ 失败/不可用
tryMobileWebView()           ← Android/iOS 移动端
    ↓ 失败/不可用
tryHeadlessBrowser()         ← 服务端有 Chromium 环境
    ↓ 失败/不可用
tryQrCodeApiProvider()       ← 全平台（需要二维码 API 服务）
    ↓ 失败/不可用
tryTokenImportProvider()     ← 全平台（用户手动粘贴 Cookie）
    ↓ 全部失败
throw LoginNotAvailableError ← 告知用户没有可用登录方式
```

### 8.4 QR Code API Provider 完整实现

这是最关键的降级方案：不依赖任何 WebView，仅需 HTTP 请求。

```typescript
class QrCodeApiProvider implements CredentialProvider {
  private apiBaseUrl: string;
  private pollIntervalMs: number;
  private timeoutMs: number;
  private onQrCode: ((base64: string, url: string) => void) | null;
  private onStatus: ((status: QrStatus) => void) | null;

  constructor(options: QrCodeApiOptions) {
    this.apiBaseUrl = options.apiBaseUrl;
    this.pollIntervalMs = options.pollIntervalMs || 2000;
    this.timeoutMs = options.timeoutMs || 120000;
    this.onQrCode = options.onQrCodeGenerated || null;
    this.onStatus = options.onStatusChange || null;
  }

  async login(): Promise<Credential | null> {
    // STEP 1: 获取 unikey
    const { key } = await this.get('/login/qr/key');

    // STEP 2: 生成二维码
    const { img, url } = await this.get(`/login/qr/create?key=${key}&qrimg=true`);
    if (this.onQrCode) this.onQrCode(img, url);

    // STEP 3: 轮询扫码状态
    const deadline = Date.now() + this.timeoutMs;
    let lastCode = 0;

    while (Date.now() < deadline) {
      const result = await this.get(`/login/qr/check?key=${key}`);
      const code = result.code || 0;

      // 801 = 等待扫码, 802 = 已扫待确认, 803 = 授权成功, 800 = 过期
      if (code === 803) {
        // ✅ 登录成功 — 提取 Cookie 构建 Credential
        return this.buildCredential(result, key);
      }

      if (code === 800) {
        this.onStatus?.('expired');
        return null;  // 二维码过期
      }

      if (code !== lastCode) {
        this.onStatus?.(
          code === 802 ? 'scanned'
          : code === 801 ? 'waiting'
          : undefined as any
        );
        lastCode = code;
      }

      await sleep(this.pollIntervalMs);
    }

    return null;  // 超时
  }

  private async buildCredential(result: any, key: string): Promise<Credential> {
    // 提取 Cookie
    if (result.cookie || result.hasCookie) {
      const rawCookie = result.cookie || '';
      const cookieHeader = NeteaseCookieBuilder.parse(rawCookie);

      // 回查获取用户信息
      let status: any = {};
      try {
        status = await this.get('/login/status');
      } catch {}

      return {
        core: cookieHeader.core!,
        device: cookieHeader.device!,
        optional: {
          nickname: result.nickname || status.nickname || '',
          userId: result.userId || status.userId || 0,
          vipType: result.vipType ?? status.vipType ?? 0,
          avatarUrl: result.avatar || status.avatar || '',
        },
        expireAt: 0,
        acquiredAt: Math.floor(Date.now() / 1000),
        source: 'qr_api',
        rawCookieHeader: rawCookie,
      };
    }

    throw new Error('QR login succeeded but no cookie received');
  }

  private async get(path: string): Promise<any> {
    const resp = await fetch(`${this.apiBaseUrl}${path}`);
    return resp.json();
  }

  getCurrent(): Credential | null {
    return null;  // QR Provider 不维护本地会话
  }

  async checkValid(): Promise<CredentialStatus> {
    throw new Error('Not implemented — use ExpiryManager with a store');
  }

  async logout(): Promise<void> {}
  onCredentialChange(callback: any): void {}
}
```

### 8.5 Headless Browser Provider 实现

```typescript
class HeadlessBrowserProvider implements CredentialProvider {
  private options: HeadlessBrowserOptions;

  constructor(options: HeadlessBrowserOptions) {
    this.options = options;
  }

  async login(): Promise<Credential | null> {
    const browser = await launchBrowser(this.options);
    try {
      const page = await browser.newPage();
      await page.goto('https://music.163.com/#/login');

      if (this.options.strategy === 'qr_scan') {
        // 等待二维码出现，回调给调用方
        await page.waitForSelector('.qr-code');
        const qrImage = await page.screenshot({ clip: { ... } });
        this.options.onQrImage?.(qrImage.toString('base64'));

        // 等待 MUSIC_U cookie 出现
        await page.waitForFunction(() => {
          return document.cookie.includes('MUSIC_U');
        }, { timeout: this.options.timeoutMs });

      } else if (this.options.strategy === 'phone_login') {
        await page.fill('input[type="tel"]', this.options.phone!);
        await page.fill('input[type="password"]', this.options.password!);
        await page.click('button[type="submit"]');
        await page.waitForNavigation();
      }

      // 提取所有 Cookie
      const cookies = await page.context().cookies();
      const rawHeader = cookies
        .filter(c => c.domain.includes('163.com') || c.domain.includes('music.163.com'))
        .map(c => `${c.name}=${c.value}`)
        .join('; ');

      return NeteaseCookieBuilder.parse(rawHeader) as Credential;

    } finally {
      await browser.close();
    }
  }

  // ...
}
```

---

## 9. 七层架构总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         应用代码 (Application)                           │
│       只依赖：CredentialProvider 工厂 / Credential 对象 / 拦截器           │
└────────────────────────┬────────────────────────────────────────────────┘
                         │
┌────────────────────────┴────────────────────────────────────────────────┐
│                    SDK 核心 (Language Bindings)                          │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ ① 凭证数据模型 Credential (protobuf / JSON Schema)                 ││
│  │   跨语言序列化 | 结构化字段 | 来源标记 | 过期时间                     ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ ② 凭证获取器 CredentialProvider (可插拔)                            ││
│  │   接口: login() / getCurrent() / checkValid() / logout()            ││
│  │   ├─ DesktopWebViewProvider (Electron/CEF/WebView2)                 ││
│  │   ├─ MobileWebViewProvider (Android/iOS WebView)                    ││
│  │   ├─ QrCodeApiProvider (纯接口, 无浏览器)                           ││
│  │   ├─ HeadlessBrowserProvider (Playwright/Puppeteer)                 ││
│  │   └─ TokenImportProvider (手动导入)                                 ││
│  │   降级链: tryLogin([优先, 降级1, 降级2])                            ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ ③ 会话持久化 CredentialStore (跨端)                                 ││
│  │   接口: save() / load() / delete() / exists()                       ││
│  │   ├─ EncryptedFileStore (AES-256-GCM)                               ││
│  │   ├─ SystemKeychainStore (macOS/Windows/Linux)                      ││
│  │   ├─ RedisStore / DatabaseStore (服务端)                            ││
│  │   ├─ SharedPreferencesStore / KeychainStore (移动端)                ││
│  │   └─ CascadeStore (多级联)                                          ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ ④ 凭证校验 & 过期管理 ExpiryManager                                 ││
│  │   主动校验 /user/account + login_status                             ││
│  │   失效判定: code 301/401 / "未登录"消息                              ││
│  │   后台自检: 可配置间隔, 超清自动清理                                 ││
│  │   事件: onExpired / onExpiring / onStatusChange                     ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ ⑤ HTTP 请求拦截器 CredentialInterceptor                             ││
│  │   自动注入 Cookie Header（按优先级排序, 过滤属性）                    ││
│  │   自动注入 UA / Referer / X-Real-IP                                 ││
│  │   自动检测响应中的凭证失效信号                                       ││
│  │   适配: Axios / OkHttp / requests / http.Client / Fetch              ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ ⑥ 风控适配层 RequestHeaderNormalizer                                ││
│  │   根据 source 切换 UA: Web / Android / iOS / PC 客户端              ││
│  │   设备指纹补齐: X-Device-Id / _ntes_nuid                           ││
│  │   IP 伪装: X-Real-IP 国内出口 IP                                  ││
│  │   接口权限矩阵: 不同来源可安全访问的 API 范围                        ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │ ⑦ 无浏览器兜底                                                      ││
│  │   QrCodeApiProvider: 纯 API 二维码登录（无 GUI）                     ││
│  │   HeadlessBrowserProvider: Playwright 全自动登录                    ││
│  │   TokenImportProvider: 手动 Cookie 粘贴                             ││
│  └─────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 10. 完整交互时序

### 10.1 Desktop WebView 登录

```
应用初始化                   凭证获取器                    用户
    │                          │                          │
    │ store.load()             │                          │
    │◄──── Credential ────────│                          │
    │     (有缓存)             │                          │
    │                          │                          │
    │ expiryManager.verifyNow()│                          │
    │────────────────────────►│                          │
    │  (调用 /user/account)    │                          │
    │◄── valid ──────────────│                          │
    │                          │                          │
    │ [使用现有凭证, 跳过登录]   │                          │
    │                          │                          │
    ─── 或 ───                 │                          │
    │                          │                          │
    │ provider.login()         │                          │
    │────────────────────────►│                          │
    │                          │ 创建独立 session WebView │
    │                          │ loadURL(官方登录页) ────►│
    │                          │                          │  ← 用户扫码
    │                          │ 轮询 cookies (1.2s)     │
    │                          │◄── MUSIC_U 出现 ──────│
    │                          │                          │
    │                          │ close WebView            │
    │◄── Credential ──────────│                          │
    │                          │                          │
    │ store.save(credential)  │                          │
    │ expiryManager.start()   │                          │
    │ [继续业务流程]           │                          │
```

### 10.2 服务端 QR API 登录（无界面）

```
服务端进程               QR API Provider           NeteaseCloudMusicApi
    │                          │                          │
    │ provider.login()         │                          │
    │────────────────────────►│                          │
    │                          │ GET /login/qr/key        │
    │                          │─────────────────────────►│
    │                          │◄── { key: "xxx" } ─────│
    │                          │                          │
    │                          │ GET /login/qr/create     │
    │                          │    ?key=xxx&qrimg=true   │
    │                          │─────────────────────────►│
    │                          │◄── { img, url } ───────│
    │                          │                          │
    │ 回调: onQrCode(img, url) │                          │
    │ [外部队列等待扫码]       │                          │
    │                          │                          │
    │            ┌─────────────┴─────────────┐            │
    │            │ 用户用手机网易云 App 扫码    │            │
    │            └───────────────────────────┘            │
    │                          │                          │
    │                          │ GET /login/qr/check      │
    │                          │    ?key=xxx  (每 2s)     │
    │                          │─────────────────────────►│
    │                          │◄── 801 (等待扫码) ──────│
    │                          │◄── 802 (已扫待确认) ───│
    │                          │◄── 803 (授权成功) ─────│
    │                          │    + cookie              │
    │                          │                          │
    │                          │ 构建 Credential          │
    │◄── Credential ──────────│                          │
    │                          │                          │
    │ store.save(credential)   │                          │
    │ [继续业务流程]           │                          │
```

### 10.3 凭证过期自动清理

```
请求拦截器                   ExpiryManager                 网易云 API
    │                          │                              │
    │ 发起 API 请求            │                              │
    │─────────────────────────►                              │
    │  (自动注入 Cookie)       │                              │
    │                          │                              │
    │◄── 301 / "未登录" ──────│                              │
    │                          │                              │
    │ 触发 handler             │                              │
    │─────────────────────────►                              │
    │                          │ 校验确认失效                   │
    │                          │─────────────────────────────►│
    │                          │◄── 未登录 ──────────────────│
    │                          │                              │
    │                          │ store.delete()               │
    │                          │ onExpired 回调               │
    │                          │ 通知上层刷新 UI               │
    │◄── 已清理 ─────────────│                              │
```

---

## 11. 多语言实现指南

### 11.1 TypeScript / JavaScript（Electron + 浏览器）

```
关键包:
  @netease-auth/sdk         — 核心模型 + 接口定义
  @netease-auth/electron    — DesktopWebViewProvider
  @netease-auth/providers   — QrCodeApiProvider, TokenImportProvider
  @netease-auth/fetch-interceptor — Fetch API 拦截器

最小集成:
  import { CredentialProviderFactory, QrCodeApiProvider } from '@netease-auth/sdk';
  const cred = await CredentialProviderFactory.tryLogin([
    new DesktopWebViewProvider(),
    new QrCodeApiProvider({ apiBaseUrl: 'http://localhost:3000' }),
  ]);
  // cred.core.musicU → 直接使用
```

### 11.2 Python（Flask / FastAPI / 脚本）

```
关键包:
  netease-auth               — 核心模型 + CookieBuilder + 校验器
  netease-auth[requests]     — NeteaseAuthSession (requests Session 包装)

最小集成:
  from netease_auth import NeteaseAuthSession, Credential
  session = NeteaseAuthSession()
  session.set_credential(cred)
  resp = session.get('https://music.163.com/api/v1/playlist/detail?id=xxx')
```

### 11.3 Go（服务端 / 工具）

```
关键包:
  github.com/example/netease-auth-sdk — Credential 模型 + 构建器
  github.com/example/netease-auth-sdk/transport — AuthTransport

最小集成:
  import "github.com/example/netease-auth-sdk/transport"
  client := transport.NewAuthClient(cred)
  resp, _ := client.Get("https://music.163.com/api/v1/search?keywords=周杰伦")
```

### 11.4 Java（Spring Boot / Android）

```
关键包:
  com.netease.auth:auth-sdk:1.0.0          — 核心模型
  com.netease.auth:okhttp-interceptor:1.0.0 — OkHttp 拦截器

最小集成:
  OkHttpClient client = new OkHttpClient.Builder()
    .addInterceptor(new NeteaseAuthInterceptor(store))
    .build();
  Request req = new Request.Builder()
    .url("https://music.163.com/api/v1/song/detail?id=xxx")
    .build();
```

### 11.5 Protobuf 代码生成

```bash
# 从 proto 文件生成各语言绑定
protoc --java_out=./java netease_credential.proto
protoc --go_out=./go netease_credential.proto
protoc --python_out=./python netease_credential.proto
protoc --js_out=./typescript netease_credential.proto
protoc --kotlin_out=./kotlin netease_credential.proto
```

---

## 12. 安全规范

### 12.1 凭证存储

| 规则 | 说明 |
|------|------|
| **禁止明文存储** | 任何持久化存储必须使用 AES-256-GCM 或系统密钥链 |
| **禁止硬编码** | 凭证不得出现在配置文件、代码仓库、日志中 |
| **禁止跨设备传输** | 凭证不应通过不安全的信道（HTTP、未加密的 WebSocket）传输 |
| **最小暴露原则** | 尽可能使用 `core.musicU` 而非 `rawCookieHeader` |

### 12.2 传输安全

- 所有 API 调用必须使用 HTTPS
- Cookie Header 不应出现在日志或监控系统的采集范围中
- `Referer` 固定为 `https://music.163.com/`

### 12.3 WebView 安全

```
sandbox: true                // 启用沙箱
contextIsolation: true       // 隔离上下文
nodeIntegration: false       // 禁用 Node.js
setWindowOpenHandler: deny   // 拦截弹窗
```

### 12.4 凭证生命周期

```
获取 ──→ 加密存储 ──→ 定时校验 ──→ 过期自动清理
         │                     │
         ▼                     ▼
     内存缓存              触发 re-login
```

---

## 13. 附录

### 13.1 完整依赖关系图

```
Credential (数据模型 — 语言无关)
  ├── 被 CredentialProvider.login() 返回
  ├── 被 CredentialStore.save()/load() 序列化
  ├── 被 CredentialValidator.validate() 校验
  ├── 被 CredentialInterceptor 组装为 HTTP Header
  └── 被 RequestHeaderNormalizer 用于 UA 决策

CredentialProvider (接口)
  ├── DesktopWebViewProvider  ← 依赖: Electron / CEF
  ├── MobileWebViewProvider   ← 依赖: 系统 WebView
  ├── QrCodeApiProvider       ← 依赖: HTTP 客户端
  ├── HeadlessBrowserProvider ← 依赖: Playwright
  └── TokenImportProvider     ← 依赖: 用户输入

CredentialStore (接口)
  ├── EncryptedFileStore     ← 依赖: Node.js fs / crypto
  ├── SystemKeychainStore    ← 依赖: keytar / 系统 API
  ├── RedisStore             ← 依赖: Redis 客户端
  └── MemoryStore            ← 无依赖

ExpiryManager ─── 依赖: CredentialValidator + CredentialStore

CredentialInterceptor ─── 依赖: CredentialStore (+ ExpiryManager 可选)
```

### 13.2 配置参考汇总

```yaml
# .netease-auth.yaml — 通用 SDK 配置
credential:
  store:
    primary: keychain           # keychain | encrypted_file | redis | memory
    fallback: encrypted_file
    encrypted_file_path: ~/.netease/.credential   # 仅 encrypted_file 用
    redis:
      key_prefix: "netease:cred:"
      ttl_seconds: 604800       # 7 天

  expiry:
    background_check_interval_ms: 300000   # 5 分钟
    preemptive_refresh_threshold_ms: 3600000  # 1 小时
    auto_clear_on_invalid: true

  http:
    default_referer: "https://music.163.com/"
    auto_inject_cookie: true
    auto_inject_ua: true
    auto_detect_invalid: true

  provider:
    preferred: ["desktop_webview", "qr_api", "headless", "manual"]
    webview:
      window_width: 940
      window_height: 760
      login_url: "https://music.163.com/#/login"
    qr_api:
      base_url: "http://localhost:3000"
      poll_interval_ms: 2000
      timeout_ms: 120000
    headless:
      browser_type: playwright
      strategy: qr_scan
      timeout_ms: 180000
```

### 13.3 与 Mineradio 现方案的映射

| 通用层 | Mineradio 现有实现 | 状态 |
|--------|-------------------|------|
| ① 凭证数据模型 | 无结构化模型，裸 Cookie 字符串 | ❌ 缺失 |
| ② DesktopWebViewProvider | `desktop/main.js` `openNeteaseMusicLoginWindow()` | ✅ 已有 |
| ② QrCodeApiProvider | `server.js` QR endpoints + `index.html` 前端轮询 | ⚠️ 代码分散 |
| ② TokenImportProvider | `server.js` `POST /api/login/cookie` | ⚠️ 无前端 UI |
| ③ EncryptedFileStore | `server.js` `.cookie` 明文文件 | ❌ 无加密 |
| ④ ExpiryManager | `server.js` `getLoginInfo()` + `isNeteaseAuthInvalidPayload()` | ⚠️ 耦合在 HTTP 服务中 |
| ⑤ 请求拦截器 | `server.js` 拼接 `cookie: userCookie` | ⚠️ 仅服务端一处 |
| ⑥ 风控适配层 | 无 UA 切换，无设备指纹补齐 | ❌ 缺失 |
| ⑦ 无浏览器兜底 | 二维码降级仅在前端渲染层 | ⚠️ 后端不可调用 |
