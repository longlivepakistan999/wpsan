# CLAUDE.md - WPSan 项目指南

## 项目概述

WPSan (WordPress Security Analyzer) 是一个商业级分布式 WordPress 安全扫描工具，采用无入侵方式进行安全检测。

### 设计原则

1. **无入侵检测** - 所有探测均基于被动信息收集，不对目标系统造成任何破坏
2. **渐进式扫描** - 按顺序逐步深入：识别CMS → 检测WAF → 版本探测 → 资产枚举 → 漏洞匹配
3. **POC 可扩展** - 插件式 POC 架构，支持动态加载和热更新
4. **分布式架构** - 支持多节点部署，任务调度和负载均衡
5. **商业友好** - 模块化设计，支持许可证管理和定价策略

---

## 扫描流程设计

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            WPSan 扫描流程                                    │
└─────────────────────────────────────────────────────────────────────────────┘

     ┌──────────────┐
     │   输入 URL    │
     └──────┬───────┘
            ▼
     ┌──────────────┐      否
     │ 是否是 WP？   │─────────────────────────────┐
     └──────┬───────┘                             │
            │ 是                                   ▼
            ▼                              ┌──────────────┐
     ┌──────────────┐                      │  非WP站点    │
     │ 检测 CF/WAF  │                      │  结束扫描    │
     └──────┬───────┘                      └──────────────┘
            │
            ▼
     ┌──────────────┐
     │ 检测WP版本    │
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │ 枚举所有插件  │
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │ 枚举所有主题  │
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │ 检测插件版本  │
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │ 检测主题版本  │
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │ 匹配漏洞库    │──────────► 显示关联的 POC 列表
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │ 用户点击 POC  │──────────► 执行 POC 验证
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │ 返回验证结果  │
     └──────────────┘
```

### 扫描阶段详解

| 阶段 | 名称 | 描述 | 输出 |
|------|------|------|------|
| 1 | WordPress 识别 | 判断目标是否为 WordPress | `isWordPress: boolean` |
| 2 | WAF/CDN 检测 | 检测 Cloudflare、Sucuri 等 | `waf: { type, detected }` |
| 3 | 版本检测 | 检测 WordPress 核心版本 | `version: string` |
| 4 | 插件枚举 | 发现所有已安装插件 | `plugins: [{ slug, name }]` |
| 5 | 主题枚举 | 发现所有已安装主题 | `themes: [{ slug, name }]` |
| 6 | 插件版本检测 | 检测每个插件的版本 | `plugins: [{ slug, version }]` |
| 7 | 主题版本检测 | 检测每个主题的版本 | `themes: [{ slug, version }]` |
| 8 | 漏洞匹配 | 根据版本匹配漏洞库 | `vulnerabilities: [{ cve, poc }]` |
| 9 | POC 验证 | 用户触发，验证漏洞 | `{ vulnerable, evidence }` |

---

## 项目结构

```
wpsan/
├── src/
│   ├── core/                    # 核心引擎
│   │   ├── scanner.ts           # 扫描器主类（流程编排）
│   │   ├── pipeline.ts          # 扫描管道（阶段控制）
│   │   ├── http-client.ts       # HTTP 客户端封装
│   │   └── rate-limiter.ts      # 请求限速器
│   │
│   ├── detectors/               # 检测模块（按扫描顺序）
│   │   ├── 01-wordpress/        # Step 1: WordPress 识别
│   │   │   ├── detector.ts      # 主检测器
│   │   │   └── signatures.ts    # WP 特征签名
│   │   │
│   │   ├── 02-waf/              # Step 2: WAF/CDN 检测
│   │   │   ├── detector.ts
│   │   │   ├── cloudflare.ts    # Cloudflare 检测
│   │   │   ├── sucuri.ts        # Sucuri 检测
│   │   │   ├── wordfence.ts     # Wordfence 检测
│   │   │   └── generic.ts       # 通用 WAF 检测
│   │   │
│   │   ├── 03-version/          # Step 3: WordPress 版本
│   │   │   ├── detector.ts
│   │   │   └── methods/         # 各种版本检测方法
│   │   │
│   │   ├── 04-plugins/          # Step 4: 插件枚举
│   │   │   ├── enumerator.ts    # 插件枚举器
│   │   │   ├── passive.ts       # 被动枚举（源码分析）
│   │   │   ├── aggressive.ts    # 主动枚举（路径探测）
│   │   │   └── wordlist.ts      # 字典管理
│   │   │
│   │   ├── 05-themes/           # Step 5: 主题枚举
│   │   │   ├── enumerator.ts
│   │   │   ├── passive.ts
│   │   │   └── aggressive.ts
│   │   │
│   │   ├── 06-plugin-version/   # Step 6: 插件版本检测
│   │   │   ├── detector.ts
│   │   │   └── extractors/      # 版本提取器
│   │   │
│   │   └── 07-theme-version/    # Step 7: 主题版本检测
│   │       ├── detector.ts
│   │       └── extractors/
│   │
│   ├── vulndb/                  # 漏洞数据库
│   │   ├── database.ts          # 数据库接口
│   │   ├── matcher.ts           # 漏洞匹配引擎
│   │   ├── models/              # 数据模型
│   │   └── sync/                # 数据同步
│   │
│   ├── poc/                     # POC 模块（可扩展）
│   │   ├── loader.ts            # POC 加载器（支持热加载）
│   │   ├── runner.ts            # POC 执行器
│   │   ├── registry.ts          # POC 注册表
│   │   ├── base.ts              # POC 基类
│   │   └── modules/             # POC 模块目录
│   │       ├── wordpress/       # WP 核心漏洞 POC
│   │       ├── plugins/         # 插件漏洞 POC
│   │       │   ├── elementor/
│   │       │   ├── woocommerce/
│   │       │   ├── contact-form-7/
│   │       │   └── ...
│   │       └── themes/          # 主题漏洞 POC
│   │
│   ├── distributed/             # 分布式模块
│   │   ├── master.ts            # 主节点（任务调度）
│   │   ├── worker.ts            # 工作节点（执行扫描）
│   │   ├── queue.ts             # 任务队列
│   │   ├── coordinator.ts       # 协调器
│   │   └── protocol.ts          # 通信协议
│   │
│   ├── api/                     # API 服务
│   │   ├── server.ts            # HTTP 服务器
│   │   ├── routes/
│   │   │   ├── scan.ts          # 扫描相关 API
│   │   │   ├── poc.ts           # POC 相关 API
│   │   │   └── vulndb.ts        # 漏洞库 API
│   │   └── websocket.ts         # WebSocket 实时推送
│   │
│   ├── cli/                     # 命令行接口
│   │   ├── index.ts
│   │   └── commands/
│   │
│   └── utils/
│       ├── logger.ts
│       ├── cache.ts
│       └── config.ts
│
├── data/
│   ├── wordlists/               # 枚举字典
│   │   ├── plugins-popular.txt  # 热门插件 (~1000)
│   │   ├── plugins-full.txt     # 完整插件 (~100000)
│   │   ├── themes-popular.txt
│   │   └── themes-full.txt
│   └── vulndb/                  # 漏洞数据库
│
├── poc-modules/                 # 外部 POC 模块（可扩展）
│   └── custom/                  # 用户自定义 POC
│
├── tests/
├── docs/
├── scripts/
├── package.json
├── tsconfig.json
└── docker-compose.yml           # 分布式部署配置
```

---

## 技术栈

### 核心技术
- **语言**: TypeScript (Node.js 20+)
- **HTTP 客户端**: undici (高性能 HTTP/1.1 & HTTP/2)
- **数据库**: PostgreSQL (主数据库) + Redis (缓存/队列)
- **消息队列**: BullMQ (基于 Redis)
- **API 框架**: Fastify
- **WebSocket**: ws 或 Socket.io

### 分布式
- **容器化**: Docker + Docker Compose
- **编排**: Kubernetes (可选)
- **服务发现**: Consul 或 etcd (可选)

### 前端 (Web UI)
- **框架**: Vue 3 或 React
- **状态管理**: Pinia 或 Zustand
- **UI 组件**: Element Plus 或 Ant Design

---

## 核心功能设计

### 1. WordPress 识别

```typescript
// src/detectors/01-wordpress/detector.ts

interface IWordPressDetectionResult {
  isWordPress: boolean;
  confidence: number;        // 0-100
  indicators: string[];      // 检测到的特征
}

const WP_INDICATORS = [
  // HTML 特征
  { type: 'html', pattern: /wp-content\//, weight: 30 },
  { type: 'html', pattern: /wp-includes\//, weight: 30 },
  { type: 'html', pattern: /<meta name="generator" content="WordPress/, weight: 50 },

  // HTTP 头特征
  { type: 'header', name: 'X-Powered-By', pattern: /WordPress/, weight: 40 },
  { type: 'header', name: 'Link', pattern: /wp-json/, weight: 35 },

  // 路径探测
  { type: 'path', path: '/wp-login.php', status: 200, weight: 50 },
  { type: 'path', path: '/wp-admin/', status: [200, 301, 302], weight: 40 },
  { type: 'path', path: '/xmlrpc.php', status: 200, weight: 30 },
  { type: 'path', path: '/wp-json/', status: 200, weight: 45 },

  // Cookie 特征
  { type: 'cookie', pattern: /wordpress_/, weight: 35 },
];

// 置信度阈值
const CONFIDENCE_THRESHOLD = 60;
```

### 2. WAF/CDN 检测

```typescript
// src/detectors/02-waf/detector.ts

interface IWafDetectionResult {
  detected: boolean;
  type: 'cloudflare' | 'sucuri' | 'wordfence' | 'akamai' | 'incapsula' | 'generic' | null;
  details: {
    bypassPossible: boolean;
    realIp?: string;         // 如果能获取真实 IP
    notes: string[];
  };
}

// Cloudflare 检测
const CLOUDFLARE_INDICATORS = {
  headers: ['cf-ray', 'cf-cache-status', 'cf-request-id'],
  cookies: ['__cfduid', '__cf_bm'],
  serverHeader: /cloudflare/i,
  errorPage: /cloudflare|cf-error/i,
  ipRanges: ['173.245.48.0/20', '103.21.244.0/22', ...],  // CF IP 段
};

// Sucuri 检测
const SUCURI_INDICATORS = {
  headers: ['x-sucuri-id', 'x-sucuri-cache'],
  serverHeader: /Sucuri/i,
  cookies: ['sucuri_'],
};

// Wordfence 检测
const WORDFENCE_INDICATORS = {
  cookies: ['wfvt_', 'wordfence_'],
  blockPage: /wordfence|wf-blocked/i,
};
```

### 3. WordPress 版本检测

```typescript
// src/detectors/03-version/detector.ts

interface IVersionDetectionResult {
  version: string | null;
  method: string;            // 使用的检测方法
  confidence: number;
}

// 检测方法（按可靠性排序）
const VERSION_METHODS = [
  {
    name: 'meta_generator',
    path: '/',
    extract: (html: string) => {
      const match = html.match(/<meta name="generator" content="WordPress ([0-9.]+)"/);
      return match?.[1] || null;
    },
    confidence: 95,
  },
  {
    name: 'feed_generator',
    path: '/feed/',
    extract: (xml: string) => {
      const match = xml.match(/<generator>.*WordPress\/([0-9.]+)<\/generator>/);
      return match?.[1] || null;
    },
    confidence: 95,
  },
  {
    name: 'opml',
    path: '/wp-links-opml.php',
    extract: (xml: string) => {
      const match = xml.match(/generator="WordPress\/([0-9.]+)"/);
      return match?.[1] || null;
    },
    confidence: 90,
  },
  {
    name: 'readme',
    path: '/readme.html',
    extract: (html: string) => {
      const match = html.match(/Version ([0-9.]+)/);
      return match?.[1] || null;
    },
    confidence: 85,
  },
  {
    name: 'css_version',
    path: '/',
    extract: (html: string) => {
      const match = html.match(/wp-includes\/css\/.*\?ver=([0-9.]+)/);
      return match?.[1] || null;
    },
    confidence: 70,
  },
  {
    name: 'js_version',
    path: '/',
    extract: (html: string) => {
      const match = html.match(/wp-includes\/js\/.*\?ver=([0-9.]+)/);
      return match?.[1] || null;
    },
    confidence: 70,
  },
  {
    name: 'hash_compare',
    files: [
      '/wp-includes/version.php',
      '/wp-includes/css/admin-bar.min.css',
    ],
    database: 'version_hashes.json',
    confidence: 99,
  },
];
```

### 4. 插件枚举

```typescript
// src/detectors/04-plugins/enumerator.ts

interface IPlugin {
  slug: string;
  name?: string;
  detected: boolean;
  method: 'passive' | 'aggressive';
}

// 被动枚举：从页面源码提取
async function passiveEnumeration(html: string): Promise<IPlugin[]> {
  const plugins: IPlugin[] = [];

  // 从 wp-content/plugins/ 路径提取
  const pluginPaths = html.matchAll(/wp-content\/plugins\/([^\/'"]+)/g);
  for (const match of pluginPaths) {
    plugins.push({ slug: match[1], detected: true, method: 'passive' });
  }

  return [...new Set(plugins)];
}

// 主动枚举：字典探测
async function aggressiveEnumeration(
  target: string,
  wordlist: string[],
  concurrency: number = 10
): Promise<IPlugin[]> {
  const plugins: IPlugin[] = [];

  // 探测路径
  const paths = [
    '/wp-content/plugins/{slug}/readme.txt',
    '/wp-content/plugins/{slug}/',
  ];

  // 并发探测
  for (const slug of wordlist) {
    for (const pathTemplate of paths) {
      const path = pathTemplate.replace('{slug}', slug);
      const response = await httpClient.head(target + path);

      if (response.status === 200) {
        plugins.push({ slug, detected: true, method: 'aggressive' });
        break;  // 找到就跳过该插件的其他路径
      }
    }
  }

  return plugins;
}
```

### 5. 插件版本检测

```typescript
// src/detectors/06-plugin-version/detector.ts

interface IPluginWithVersion extends IPlugin {
  version: string | null;
  versionSource: string;     // 版本来源
}

// 版本提取位置
const VERSION_SOURCES = [
  {
    name: 'readme.txt',
    path: '/wp-content/plugins/{slug}/readme.txt',
    patterns: [
      /Stable tag:\s*([0-9][0-9.]*)/i,
      /Version:\s*([0-9][0-9.]*)/i,
    ],
  },
  {
    name: 'main_plugin_file',
    path: '/wp-content/plugins/{slug}/{slug}.php',
    patterns: [
      /Version:\s*([0-9][0-9.]*)/i,
      /\* @version\s+([0-9][0-9.]*)/i,
    ],
  },
  {
    name: 'package.json',
    path: '/wp-content/plugins/{slug}/package.json',
    extract: (json: string) => JSON.parse(json).version,
  },
  {
    name: 'changelog',
    path: '/wp-content/plugins/{slug}/changelog.txt',
    patterns: [
      /= ([0-9][0-9.]*) =/,
      /Version ([0-9][0-9.]*)/i,
    ],
  },
];
```

### 6. POC 可扩展架构

```typescript
// src/poc/base.ts - POC 基类

export abstract class BasePoc {
  abstract readonly id: string;           // 唯一标识
  abstract readonly name: string;         // POC 名称
  abstract readonly description: string;  // 描述
  abstract readonly cve?: string;         // CVE 编号
  abstract readonly severity: 'critical' | 'high' | 'medium' | 'low' | 'info';

  // 适用条件
  abstract readonly conditions: {
    type: 'core' | 'plugin' | 'theme';
    slug?: string;                        // 插件/主题 slug
    versionRange: string;                 // 受影响版本，如 "< 3.5.0"
  };

  // 验证方法（子类实现）
  abstract verify(context: IPocContext): Promise<IPocResult>;
}

// src/poc/registry.ts - POC 注册表

class PocRegistry {
  private pocs: Map<string, BasePoc> = new Map();

  // 注册 POC
  register(poc: BasePoc): void {
    this.pocs.set(poc.id, poc);
  }

  // 根据插件/主题查找关联的 POC
  findByTarget(type: string, slug: string, version: string): BasePoc[] {
    return Array.from(this.pocs.values()).filter(poc => {
      if (poc.conditions.type !== type) return false;
      if (poc.conditions.slug && poc.conditions.slug !== slug) return false;
      return semver.satisfies(version, poc.conditions.versionRange);
    });
  }

  // 热加载 POC 模块
  async loadFromDirectory(dir: string): Promise<void> {
    const files = await glob(`${dir}/**/*.poc.ts`);
    for (const file of files) {
      const module = await import(file);
      if (module.default instanceof BasePoc) {
        this.register(module.default);
      }
    }
  }
}

// src/poc/modules/plugins/elementor/CVE-2024-XXXX.poc.ts - POC 示例

export default class ElementorRcePoc extends BasePoc {
  readonly id = 'elementor-rce-2024-xxxx';
  readonly name = 'Elementor Remote Code Execution';
  readonly description = 'Elementor plugin allows unauthenticated RCE via...';
  readonly cve = 'CVE-2024-XXXX';
  readonly severity = 'critical';

  readonly conditions = {
    type: 'plugin' as const,
    slug: 'elementor',
    versionRange: '< 3.5.0',
  };

  async verify(context: IPocContext): Promise<IPocResult> {
    // 构造验证请求
    const payload = '...';
    const response = await context.http.post(
      `${context.target}/wp-admin/admin-ajax.php`,
      { action: 'elementor_...' , data: payload }
    );

    // 判断漏洞是否存在
    const vulnerable = response.body.includes('specific_indicator');

    return {
      vulnerable,
      evidence: vulnerable ? response.body.substring(0, 500) : undefined,
      details: { statusCode: response.status },
    };
  }
}
```

### 7. POC 关联与触发

```typescript
// 扫描结果数据结构

interface IScanResult {
  target: string;
  timestamp: Date;

  // 各阶段结果
  wordpress: IWordPressDetectionResult;
  waf: IWafDetectionResult;
  version: IVersionDetectionResult;
  plugins: IPluginWithVersion[];
  themes: IThemeWithVersion[];

  // 漏洞匹配结果（包含关联的 POC）
  vulnerabilities: IVulnerabilityMatch[];
}

interface IVulnerabilityMatch {
  vulnerability: IVulnerability;      // 漏洞信息
  affectedComponent: {
    type: 'core' | 'plugin' | 'theme';
    slug?: string;
    version: string;
  };
  pocs: IPocInfo[];                   // 关联的 POC 列表
}

interface IPocInfo {
  id: string;
  name: string;
  severity: string;
  canRun: boolean;                    // 是否可执行
}

// API: 获取扫描结果
// GET /api/v1/scans/:scanId
// Response: IScanResult

// API: 执行 POC
// POST /api/v1/scans/:scanId/poc/:pocId/run
// Response: IPocResult
```

---

## 分布式架构设计

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           WPSan 分布式架构                                   │
└─────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────┐
                              │   Web UI    │
                              │  (Vue/React)│
                              └──────┬──────┘
                                     │
                                     ▼
                              ┌─────────────┐
                              │  API Server │
                              │  (Fastify)  │
                              └──────┬──────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
       ┌─────────────┐        ┌─────────────┐        ┌─────────────┐
       │   Master    │◄──────►│    Redis    │◄──────►│  PostgreSQL │
       │  (调度器)   │        │ (队列/缓存)  │        │  (持久化)   │
       └──────┬──────┘        └─────────────┘        └─────────────┘
              │
              │ 任务分发
    ┌─────────┼─────────┬─────────────┐
    │         │         │             │
    ▼         ▼         ▼             ▼
┌────────┐┌────────┐┌────────┐   ┌────────┐
│Worker 1││Worker 2││Worker 3│...│Worker N│
│  扫描   ││  扫描   ││  扫描   │   │  扫描   │
└────────┘└────────┘└────────┘   └────────┘
```

### 组件职责

| 组件 | 职责 |
|------|------|
| **Web UI** | 用户交互界面，提交扫描任务，查看结果，点击运行 POC |
| **API Server** | RESTful API，处理请求，返回结果 |
| **Master** | 任务调度，负载均衡，监控 Worker 状态 |
| **Worker** | 执行实际扫描任务，运行 POC |
| **Redis** | 任务队列（BullMQ），结果缓存，实时状态 |
| **PostgreSQL** | 持久化存储（扫描结果、漏洞库、用户数据） |

### 任务队列设计

```typescript
// src/distributed/queue.ts

import { Queue, Worker, Job } from 'bullmq';

// 扫描任务队列
const scanQueue = new Queue('scan-tasks', { connection: redis });

// 任务类型
interface IScanJob {
  type: 'full_scan' | 'plugin_enum' | 'poc_verify';
  target: string;
  options: {
    stages?: string[];        // 指定执行的阶段
    pocId?: string;           // POC 验证时的 POC ID
    pluginSlug?: string;      // 特定插件扫描
  };
  priority: number;           // 优先级
  userId: string;
}

// Worker 处理任务
const worker = new Worker('scan-tasks', async (job: Job<IScanJob>) => {
  const { type, target, options } = job.data;

  switch (type) {
    case 'full_scan':
      return await runFullScan(target, options);
    case 'plugin_enum':
      return await runPluginEnumeration(target, options);
    case 'poc_verify':
      return await runPocVerification(target, options);
  }
}, { connection: redis, concurrency: 5 });

// 任务进度上报
worker.on('progress', (job, progress) => {
  // 通过 WebSocket 推送进度
  wsServer.broadcast(`scan:${job.id}`, { progress });
});
```

### Worker 扩展

```typescript
// src/distributed/worker.ts

class ScanWorker {
  private id: string;
  private status: 'idle' | 'busy' | 'offline';
  private currentJob: Job | null;

  constructor() {
    this.id = generateWorkerId();
    this.registerWithMaster();
  }

  // 向 Master 注册
  async registerWithMaster(): Promise<void> {
    await redis.hset('workers', this.id, JSON.stringify({
      status: 'idle',
      registeredAt: Date.now(),
      capabilities: ['scan', 'poc'],
    }));
  }

  // 心跳
  async heartbeat(): Promise<void> {
    await redis.hset('workers', this.id, JSON.stringify({
      status: this.status,
      lastHeartbeat: Date.now(),
      currentJob: this.currentJob?.id,
    }));
  }
}
```

---

## API 设计

### RESTful 端点

```typescript
// 扫描相关
POST   /api/v1/scans                    // 创建扫描任务
GET    /api/v1/scans                    // 获取扫描列表
GET    /api/v1/scans/:id                // 获取扫描详情
DELETE /api/v1/scans/:id                // 删除扫描记录

// POC 相关
GET    /api/v1/scans/:id/pocs           // 获取扫描关联的 POC 列表
POST   /api/v1/scans/:id/pocs/:pocId/run // 执行指定 POC
GET    /api/v1/pocs                     // 获取所有 POC 列表
GET    /api/v1/pocs/:id                 // 获取 POC 详情

// 漏洞库
GET    /api/v1/vulns                    // 搜索漏洞库
GET    /api/v1/vulns/:id                // 获取漏洞详情
POST   /api/v1/vulns/sync               // 同步漏洞库

// 系统状态
GET    /api/v1/system/workers           // 获取 Worker 状态
GET    /api/v1/system/stats             // 获取系统统计
```

### WebSocket 事件

```typescript
// 客户端连接后订阅扫描进度
ws.subscribe(`scan:${scanId}`);

// 服务端推送事件
ws.emit('scan:stage', { stage: 'plugins', status: 'running' });
ws.emit('scan:plugin_found', { slug: 'woocommerce', version: '8.0.0' });
ws.emit('scan:vulnerability', { cve: 'CVE-2024-xxx', severity: 'high' });
ws.emit('scan:complete', { summary: {...} });

// POC 执行进度
ws.emit('poc:started', { pocId: '...' });
ws.emit('poc:result', { pocId: '...', vulnerable: true, evidence: '...' });
```

---

## 数据模型

### 扫描结果

```typescript
// PostgreSQL Schema

interface Scan {
  id: string;
  target: string;
  status: 'pending' | 'running' | 'completed' | 'failed';

  // 扫描结果
  isWordPress: boolean;
  wpVersion: string | null;
  wafDetected: boolean;
  wafType: string | null;

  // 时间戳
  createdAt: Date;
  startedAt: Date | null;
  completedAt: Date | null;

  // 关联
  userId: string;
}

interface ScanPlugin {
  id: string;
  scanId: string;
  slug: string;
  name: string | null;
  version: string | null;
  detectionMethod: string;
}

interface ScanTheme {
  id: string;
  scanId: string;
  slug: string;
  name: string | null;
  version: string | null;
}

interface ScanVulnerability {
  id: string;
  scanId: string;
  vulnId: string;           // 关联漏洞库
  componentType: string;
  componentSlug: string;
  componentVersion: string;
}

interface PocExecution {
  id: string;
  scanId: string;
  pocId: string;
  status: 'pending' | 'running' | 'success' | 'failed';
  vulnerable: boolean | null;
  evidence: string | null;
  executedAt: Date;
}
```

### 漏洞库

```typescript
interface Vulnerability {
  id: string;
  cve: string | null;
  title: string;
  description: string;

  // 影响范围
  componentType: 'core' | 'plugin' | 'theme';
  componentSlug: string | null;
  affectedVersions: string;   // semver range
  fixedVersion: string | null;

  // 严重性
  severity: 'critical' | 'high' | 'medium' | 'low';
  cvssScore: number | null;

  // POC 关联
  pocIds: string[];

  // 参考
  references: string[];

  // 时间
  publishedAt: Date;
  updatedAt: Date;
}
```

---

## Docker 部署

```yaml
# docker-compose.yml

version: '3.8'

services:
  # API 服务
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/wpsan
      - REDIS_URL=redis://redis:6379
      - NODE_ENV=production
    depends_on:
      - db
      - redis

  # Master 调度器
  master:
    build: .
    command: npm run start:master
    environment:
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis

  # Worker (可扩展)
  worker:
    build: .
    command: npm run start:worker
    environment:
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
    deploy:
      replicas: 3              # 默认 3 个 Worker

  # PostgreSQL
  db:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=wpsan
      - POSTGRES_PASSWORD=password

  # Redis
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### 扩展 Worker

```bash
# 手动扩展 Worker 数量
docker-compose up -d --scale worker=10
```

---

## 开发规范

### POC 开发规范

1. **命名规则**: `{component}-{vuln-type}-{cve/date}.poc.ts`
2. **继承基类**: 必须继承 `BasePoc`
3. **仅验证**: 禁止执行破坏性操作
4. **无条件触发**: 不依赖认证或特殊配置
5. **提供证据**: 返回明确的漏洞证据
6. **错误处理**: 捕获异常，不影响整体流程

### 目录结构约定

- 检测器按执行顺序编号: `01-wordpress`, `02-waf`...
- POC 按组件类型分目录: `wordpress/`, `plugins/`, `themes/`
- 插件 POC 再按插件 slug 分目录

### Git 提交规范

```
feat(detector): add Cloudflare detection
fix(poc): fix false positive in elementor poc
chore(deps): update dependencies
docs: update API documentation
```

---

## 常用命令

```bash
# 开发
npm run dev                    # 开发模式 (API)
npm run dev:worker             # 开发模式 (Worker)
npm run build                  # 构建

# 测试
npm run test                   # 运行测试
npm run test:poc               # 测试 POC 模块

# 数据库
npm run db:migrate             # 运行迁移
npm run db:seed                # 填充测试数据
npm run vulndb:sync            # 同步漏洞库

# Docker
docker-compose up -d           # 启动所有服务
docker-compose up -d --scale worker=5  # 扩展 Worker
docker-compose logs -f worker  # 查看 Worker 日志

# POC 开发
npm run poc:create             # 创建 POC 模板
npm run poc:validate           # 验证 POC 格式
```

---

## 路线图

### Phase 1: MVP
- [ ] 核心扫描流程（7 个阶段）
- [ ] 基础漏洞匹配
- [ ] CLI 工具
- [ ] 单节点运行

### Phase 2: POC 框架
- [ ] POC 可扩展架构
- [ ] POC 热加载
- [ ] 常见插件 POC (Top 50)
- [ ] POC 执行 API

### Phase 3: 分布式
- [ ] Master/Worker 架构
- [ ] 任务队列
- [ ] Worker 自动扩展
- [ ] 负载均衡

### Phase 4: Web UI
- [ ] 扫描管理界面
- [ ] 实时进度展示
- [ ] POC 一键执行
- [ ] 报告导出

### Phase 5: 商业化
- [ ] 用户管理
- [ ] 许可证系统
- [ ] API 配额
- [ ] 高级报告
