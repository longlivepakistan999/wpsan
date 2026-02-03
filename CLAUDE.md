# CLAUDE.md - WPSan 项目指南

## 项目概述

WPSan (WordPress Security Analyzer) 是一个商业级 WordPress 安全扫描工具，采用无入侵方式进行安全检测。

### 设计原则

1. **无入侵检测** - 所有探测均基于被动信息收集，不对目标系统造成任何破坏
2. **合法合规** - 仅提供安全评估功能，POC 仅验证无条件可利用漏洞
3. **准确可靠** - 减少误报，提供可验证的检测结果
4. **商业友好** - 模块化设计，支持许可证管理和定价策略

---

## 项目结构

```
wpsan/
├── src/
│   ├── core/                    # 核心引擎
│   │   ├── scanner.ts           # 扫描器主类
│   │   ├── fingerprint.ts       # 指纹识别引擎
│   │   ├── http-client.ts       # HTTP 客户端封装
│   │   └── rate-limiter.ts      # 请求限速器
│   │
│   ├── detectors/               # 检测模块
│   │   ├── wordpress/           # WordPress 核心检测
│   │   │   ├── version.ts       # 版本检测
│   │   │   ├── config.ts        # 配置检测
│   │   │   └── users.ts         # 用户枚举
│   │   ├── plugins/             # 插件检测
│   │   │   ├── enumerator.ts    # 插件枚举
│   │   │   └── fingerprints/    # 插件指纹库
│   │   ├── themes/              # 主题检测
│   │   │   ├── enumerator.ts    # 主题枚举
│   │   │   └── fingerprints/    # 主题指纹库
│   │   └── security/            # 安全配置检测
│   │       ├── headers.ts       # HTTP 头检测
│   │       ├── waf.ts           # WAF 检测
│   │       └── exposures.ts     # 敏感文件暴露
│   │
│   ├── vulndb/                  # 漏洞数据库
│   │   ├── database.ts          # 数据库接口
│   │   ├── models/              # 数据模型
│   │   ├── sources/             # 漏洞源同步
│   │   └── data/                # 漏洞数据文件
│   │
│   ├── poc/                     # POC 验证模块
│   │   ├── runner.ts            # POC 执行器
│   │   ├── validators/          # 验证器
│   │   └── modules/             # POC 模块
│   │       ├── info-disclosure/ # 信息泄露
│   │       ├── unauth-access/   # 未授权访问
│   │       └── xmlrpc/          # XMLRPC 相关
│   │
│   ├── reporter/                # 报告生成
│   │   ├── generator.ts         # 报告生成器
│   │   ├── templates/           # 报告模板
│   │   └── formatters/          # 格式化器 (PDF/HTML/JSON)
│   │
│   ├── api/                     # API 服务 (商业版)
│   │   ├── server.ts            # API 服务器
│   │   ├── routes/              # API 路由
│   │   └── middleware/          # 中间件
│   │
│   ├── cli/                     # 命令行接口
│   │   ├── index.ts             # CLI 入口
│   │   ├── commands/            # 子命令
│   │   └── output.ts            # 输出格式化
│   │
│   └── utils/                   # 工具函数
│       ├── logger.ts            # 日志工具
│       ├── cache.ts             # 缓存管理
│       └── config.ts            # 配置管理
│
├── data/                        # 静态数据
│   ├── fingerprints/            # 指纹数据库
│   ├── wordlists/               # 枚举字典
│   └── vulndb/                  # 漏洞数据库快照
│
├── tests/                       # 测试
│   ├── unit/                    # 单元测试
│   ├── integration/             # 集成测试
│   └── fixtures/                # 测试数据
│
├── docs/                        # 文档
│   ├── api/                     # API 文档
│   ├── guides/                  # 使用指南
│   └── development/             # 开发文档
│
├── scripts/                     # 构建脚本
│   ├── build.ts                 # 构建脚本
│   ├── update-vulndb.ts         # 漏洞库更新
│   └── generate-fingerprints.ts # 指纹生成
│
├── package.json
├── tsconfig.json
├── .env.example
└── README.md
```

---

## 技术栈

### 核心技术
- **语言**: TypeScript (Node.js 18+)
- **HTTP 客户端**: undici (高性能) 或 got
- **CLI 框架**: commander 或 yargs
- **数据库**: SQLite (本地) / PostgreSQL (服务端)
- **缓存**: Redis (可选)

### 测试
- **框架**: Vitest
- **覆盖率**: c8

### 构建与发布
- **打包**: tsup 或 esbuild
- **发布**: npm / Docker

---

## 开发规范

### 代码风格

```typescript
// 使用 ESM 模块
import { Scanner } from './core/scanner.js';

// 接口命名：I 前缀
interface IScanResult {
  target: string;
  findings: IFinding[];
  timestamp: Date;
}

// 类型命名：大驼峰
type ScanOptions = {
  timeout: number;
  userAgent: string;
  throttle: number;
};

// 常量：全大写下划线
const DEFAULT_TIMEOUT = 30000;
const MAX_CONCURRENT_REQUESTS = 10;

// 异步函数统一使用 async/await
async function detectVersion(url: string): Promise<string | null> {
  // ...
}
```

### 错误处理

```typescript
// 自定义错误类
class WPSanError extends Error {
  constructor(
    message: string,
    public code: string,
    public details?: Record<string, unknown>
  ) {
    super(message);
    this.name = 'WPSanError';
  }
}

// 使用 Result 模式处理可预期的失败
type Result<T, E = WPSanError> =
  | { ok: true; value: T }
  | { ok: false; error: E };
```

### 日志规范

```typescript
import { logger } from './utils/logger.js';

// 使用结构化日志
logger.info('Scanning target', { url, options });
logger.debug('HTTP request', { method, path, status });
logger.warn('Rate limited', { retryAfter });
logger.error('Scan failed', { error, target });
```

---

## 核心功能设计

### 1. WordPress 版本检测

检测方法（按优先级）：

```typescript
const VERSION_DETECTION_METHODS = [
  // 1. Meta generator 标签
  { method: 'meta_generator', path: '/', pattern: /<meta name="generator" content="WordPress ([0-9.]+)"/ },

  // 2. Feed 中的版本
  { method: 'feed', path: '/feed/', pattern: /<generator>.*WordPress\/([0-9.]+)<\/generator>/ },

  // 3. readme.html (常被删除)
  { method: 'readme', path: '/readme.html', pattern: /Version ([0-9.]+)/ },

  // 4. CSS/JS 版本参数
  { method: 'asset_version', path: '/wp-includes/css/dist/block-library/style.min.css', pattern: /ver=([0-9.]+)/ },

  // 5. OPML 链接
  { method: 'opml', path: '/wp-links-opml.php', pattern: /generator="WordPress\/([0-9.]+)"/ },

  // 6. REST API
  { method: 'rest_api', path: '/wp-json/', field: 'description' },

  // 7. 文件哈希比对 (高级)
  { method: 'hash_compare', files: [...], database: 'version_hashes.json' }
];
```

### 2. 插件枚举

```typescript
// 枚举策略
enum EnumerationMode {
  PASSIVE = 'passive',     // 仅从页面源码提取
  AGGRESSIVE = 'aggressive' // 主动探测常见路径
}

// 插件检测路径
const PLUGIN_DETECTION_PATHS = [
  '/wp-content/plugins/{slug}/readme.txt',
  '/wp-content/plugins/{slug}/style.css',
  '/wp-content/plugins/{slug}/package.json',
  '/wp-content/plugins/{slug}/{slug}.php'
];

// 版本提取模式
const VERSION_PATTERNS = [
  /Stable tag:\s*([0-9.]+)/i,
  /Version:\s*([0-9.]+)/i,
  /"version":\s*"([0-9.]+)"/
];
```

### 3. 用户枚举

```typescript
// 无入侵用户枚举方法
const USER_ENUM_METHODS = [
  // 1. 作者归档页面
  { method: 'author_archives', path: '/?author={id}' },

  // 2. REST API (如未禁用)
  { method: 'rest_api', path: '/wp-json/wp/v2/users' },

  // 3. oEmbed
  { method: 'oembed', path: '/wp-json/oembed/1.0/embed?url={post_url}' },

  // 4. RSS Feed 作者信息
  { method: 'feed', path: '/feed/', extract: 'dc:creator' }
];
```

### 4. POC 验证框架

```typescript
// POC 模块接口
interface IPocModule {
  id: string;
  name: string;
  description: string;
  severity: 'critical' | 'high' | 'medium' | 'low' | 'info';

  // 适用条件
  conditions: {
    wordpress?: string;      // 版本范围，如 "< 5.8.0"
    plugin?: { name: string; version: string };
    theme?: { name: string; version: string };
  };

  // 验证函数
  verify(target: string, context: IScanContext): Promise<IPocResult>;
}

// POC 结果
interface IPocResult {
  vulnerable: boolean;
  evidence?: string;        // 漏洞证据
  details?: Record<string, unknown>;
}
```

**POC 编写原则：**
1. 仅验证，不利用（不执行任何破坏性操作）
2. 无条件触发（不需要认证或特殊配置）
3. 提供明确的漏洞证据
4. 包含误报检测逻辑

---

## 漏洞数据库设计

### 数据模型

```typescript
interface IVulnerability {
  id: string;                // 内部 ID
  cve?: string;              // CVE 编号
  title: string;
  description: string;

  affected: {
    type: 'core' | 'plugin' | 'theme';
    slug?: string;           // 插件/主题 slug
    versions: string;        // 受影响版本范围
  };

  severity: {
    level: 'critical' | 'high' | 'medium' | 'low';
    cvss?: number;
  };

  references: string[];      // 参考链接
  poc?: string;              // POC 模块 ID

  published: Date;
  updated: Date;
}
```

### 数据源

1. **WPVulnDB API** - 官方漏洞数据库
2. **NVD (National Vulnerability Database)** - CVE 数据
3. **Exploit-DB** - 公开漏洞
4. **自建收集** - 安全公告、博客等

---

## CLI 命令设计

```bash
# 基础扫描
wpsan scan https://example.com

# 指定扫描模块
wpsan scan https://example.com --modules=version,plugins,themes,users

# 仅枚举
wpsan enum plugins https://example.com --wordlist=popular.txt

# 漏洞检测
wpsan vuln https://example.com --severity=high,critical

# POC 验证
wpsan poc https://example.com --poc-id=CVE-2023-xxxx

# 生成报告
wpsan scan https://example.com --output=report.pdf --format=pdf

# 批量扫描
wpsan scan --targets=urls.txt --output-dir=./reports

# 更新数据库
wpsan update vulndb
wpsan update fingerprints
```

---

## API 设计 (商业版)

### RESTful 端点

```
POST /api/v1/scans           # 创建扫描任务
GET  /api/v1/scans/:id       # 获取扫描结果
GET  /api/v1/scans           # 列出扫描历史

GET  /api/v1/vulndb/search   # 搜索漏洞库
GET  /api/v1/vulndb/:id      # 获取漏洞详情

POST /api/v1/reports         # 生成报告
GET  /api/v1/reports/:id     # 下载报告
```

### WebSocket 实时更新

```typescript
// 扫描进度实时推送
ws.on('scan:progress', { scanId, progress, currentModule });
ws.on('scan:finding', { scanId, finding });
ws.on('scan:complete', { scanId, summary });
```

---

## 安全考虑

### 合规要求

1. **授权扫描** - 用户必须确认有权扫描目标
2. **速率限制** - 默认限制请求频率，避免对目标造成压力
3. **日志记录** - 记录所有扫描行为用于审计
4. **数据保护** - 扫描结果加密存储

### 请求伪装

```typescript
// User-Agent 轮换
const USER_AGENTS = [
  'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36...',
  'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36...',
  // ...
];

// 请求间隔随机化
const DELAY_RANGE = { min: 100, max: 500 }; // ms
```

---

## 测试策略

### 单元测试

```typescript
// tests/unit/detectors/version.test.ts
import { describe, it, expect } from 'vitest';
import { detectVersion } from '../../../src/detectors/wordpress/version.js';

describe('WordPress Version Detection', () => {
  it('should detect version from meta generator', async () => {
    const html = '<meta name="generator" content="WordPress 6.4.2">';
    const result = await detectVersion(mockTarget(html));
    expect(result).toBe('6.4.2');
  });
});
```

### 集成测试

使用本地 WordPress Docker 环境进行集成测试。

```yaml
# docker-compose.test.yml
services:
  wordpress:
    image: wordpress:6.4
    ports:
      - "8080:80"
```

---

## 发布与部署

### NPM 包发布

```bash
npm version patch|minor|major
npm publish
```

### Docker 镜像

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY dist ./dist
ENTRYPOINT ["node", "dist/cli/index.js"]
```

### 许可证管理 (商业版)

```typescript
interface ILicense {
  key: string;
  type: 'trial' | 'pro' | 'enterprise';
  features: string[];
  expiresAt: Date;
  maxTargets?: number;  // 并发扫描目标数
  maxScansPerDay?: number;
}
```

---

## AI 助手开发指南

### 开发新检测模块

1. 在 `src/detectors/` 下创建模块文件
2. 实现 `IDetector` 接口
3. 注册到 `scanner.ts` 的检测器列表
4. 编写单元测试
5. 更新 CLI 命令支持

### 添加新 POC

1. 在 `src/poc/modules/` 下创建 POC 文件
2. 实现 `IPocModule` 接口
3. **确保仅验证不利用**
4. 添加到漏洞数据库关联
5. 编写测试用例

### 更新漏洞库

1. 运行 `scripts/update-vulndb.ts`
2. 验证数据格式
3. 更新版本号
4. 提交数据文件

---

## 常用命令

```bash
# 开发
npm run dev           # 开发模式
npm run build         # 构建
npm run test          # 运行测试
npm run lint          # 代码检查

# 数据库
npm run vulndb:update # 更新漏洞库
npm run vulndb:stats  # 漏洞统计

# 发布
npm run release       # 发布新版本
```

---

## 路线图

### Phase 1: MVP (核心功能)
- [ ] 项目初始化与基础架构
- [ ] WordPress 版本检测
- [ ] 插件/主题枚举
- [ ] 基础漏洞匹配
- [ ] CLI 基础命令
- [ ] JSON 报告输出

### Phase 2: 增强功能
- [ ] 用户枚举
- [ ] 安全配置检测
- [ ] POC 验证框架
- [ ] PDF/HTML 报告
- [ ] 漏洞库自动更新

### Phase 3: 商业化
- [ ] API 服务
- [ ] Web 界面
- [ ] 许可证管理
- [ ] 多租户支持
- [ ] 定时扫描

### Phase 4: 高级功能
- [ ] 分布式扫描
- [ ] 自定义规则引擎
- [ ] 集成 CI/CD
- [ ] SIEM 集成
