# CLAUDE.md - WPSan 项目指南

## 项目概述

WPSan (WordPress Security Analyzer) 是一个商业级分布式 WordPress 安全扫描工具，采用无入侵方式进行安全检测，支持**几十万级**资产批量扫描。

### 设计原则

1. **无入侵检测** - 所有探测均基于被动信息收集，不对目标系统造成任何破坏
2. **渐进式扫描** - 按顺序逐步深入：识别CMS → 检测CDN → 版本探测 → 解析资产 → 漏洞匹配
3. **大规模支持** - 支持 50万+ 资产导入，高效队列处理
4. **POC 可扩展** - 插件式 POC 架构，支持动态加载和热更新
5. **分布式架构** - 支持多节点部署，任务调度和负载均衡
6. **商业友好** - 模块化设计，支持许可证管理和定价策略

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
     │ 检测 CF (IP) │                      │  结束扫描    │
     └──────┬───────┘                      └──────────────┘
            │
            ▼
     ┌──────────────┐
     │ 检测WP版本    │
     └──────┬───────┘
            │
            ▼
     ┌──────────────────────┐
     │ 解析插件/主题 (JSON)  │ ◄─── 从页面/REST API 提取
     └──────┬───────────────┘
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
| 2 | Cloudflare 检测 | 通过 IP 段判断是否使用 CF | `cloudflare: { detected, ip }` |
| 3 | 版本检测 | 检测 WordPress 核心版本 | `version: string` |
| 4 | 资产解析 | 从 JSON/HTML 解析插件和主题 | `plugins[], themes[]` |
| 5 | 插件版本检测 | 检测每个插件的版本 | `plugins: [{ slug, version }]` |
| 6 | 主题版本检测 | 检测每个主题的版本 | `themes: [{ slug, version }]` |
| 7 | 漏洞匹配 | 根据版本匹配漏洞库 | `vulnerabilities: [{ cve, poc }]` |
| 8 | POC 验证 | 用户触发，验证漏洞 | `{ vulnerable, evidence }` |

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
│   ├── targets/                 # 目标资产管理（支持50万+）
│   │   ├── manager.ts           # 资产管理器
│   │   ├── importer.ts          # 批量导入器
│   │   ├── parsers/             # 文件解析器
│   │   │   ├── txt-parser.ts    # TXT 解析 (一行一个URL)
│   │   │   └── csv-parser.ts    # CSV 解析 (支持多列)
│   │   ├── validator.ts         # URL 验证/去重
│   │   └── group.ts             # 资产分组
│   │
│   ├── detectors/               # 检测模块（按扫描顺序）
│   │   ├── 01-wordpress/        # Step 1: WordPress 识别
│   │   │   ├── detector.ts      # 主检测器
│   │   │   └── signatures.ts    # WP 特征签名
│   │   │
│   │   ├── 02-cloudflare/       # Step 2: Cloudflare 检测 (IP段)
│   │   │   ├── detector.ts      # 检测器
│   │   │   └── ip-ranges.ts     # IP 段管理
│   │   │
│   │   ├── 03-version/          # Step 3: WordPress 版本
│   │   │   ├── detector.ts
│   │   │   └── methods/         # 各种版本检测方法
│   │   │
│   │   ├── 04-assets/           # Step 4: 资产解析（插件/主题）
│   │   │   ├── parser.ts        # JSON/HTML 解析器
│   │   │   ├── plugin-parser.ts # 插件解析
│   │   │   └── theme-parser.ts  # 主题解析
│   │   │
│   │   ├── 05-plugin-version/   # Step 5: 插件版本检测
│   │   │   ├── detector.ts
│   │   │   └── extractors/      # 版本提取器
│   │   │
│   │   └── 06-theme-version/    # Step 6: 主题版本检测
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
│   │   ├── queue.ts             # 任务队列（高吞吐）
│   │   ├── batch-scheduler.ts   # 批量任务调度器
│   │   ├── coordinator.ts       # 协调器
│   │   └── protocol.ts          # 通信协议
│   │
│   ├── api/                     # API 服务
│   │   ├── server.ts            # HTTP 服务器
│   │   ├── routes/
│   │   │   ├── targets.ts       # 资产管理 API
│   │   │   ├── import.ts        # 批量导入 API
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
│       ├── ip-utils.ts          # IP 地址工具
│       ├── stream-utils.ts      # 流式处理工具
│       └── config.ts
│
├── data/
│   ├── cloudflare/              # Cloudflare IP 段
│   │   ├── ips-v4.txt           # IPv4 段 (定期从 CF 更新)
│   │   └── ips-v6.txt           # IPv6 段
│   └── vulndb/                  # 漏洞数据库
│
├── uploads/                     # 上传文件临时目录
│   └── imports/                 # 导入文件
│
├── poc-modules/                 # 外部 POC 模块（可扩展）
│   └── custom/                  # 用户自定义 POC
│
├── tests/
├── docs/
├── scripts/
│   ├── update-cf-ips.ts         # 更新 Cloudflare IP 段
│   └── ...
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

## 资产管理（支持 50万+ 目标）

### 数据模型

```typescript
// 目标资产
interface ITarget {
  id: string;
  url: string;                    // 目标 URL
  domain: string;                 // 域名（自动提取）
  groupId?: string;               // 分组 ID
  tags?: string[];                // 标签
  status: 'pending' | 'scanning' | 'completed' | 'failed';

  // 最近扫描结果摘要
  lastScan?: {
    scanId: string;
    isWordPress: boolean;
    wpVersion?: string;
    cloudflare: boolean;
    vulnCount: number;
    scannedAt: Date;
  };

  createdAt: Date;
  updatedAt: Date;
}

// 资产分组
interface ITargetGroup {
  id: string;
  name: string;
  description?: string;
  targetCount: number;            // 资产数量
  createdAt: Date;
}

// 导入任务
interface IImportJob {
  id: string;
  filename: string;
  fileType: 'txt' | 'csv';
  fileSize: number;

  status: 'pending' | 'processing' | 'completed' | 'failed';

  // 进度
  progress: {
    total: number;                // 总行数
    processed: number;            // 已处理
    valid: number;                // 有效 URL
    duplicates: number;           // 重复
    invalid: number;              // 无效
  };

  groupId?: string;               // 导入到的分组
  error?: string;

  createdAt: Date;
  completedAt?: Date;
}
```

### 批量导入设计

#### 支持的文件格式

**TXT 格式** - 一行一个 URL（必须带协议）：
```
https://example1.com
https://example2.com
http://example3.com/blog
```

> **注意**：URL 必须以 `http://` 或 `https://` 开头，不带协议的会被标记为无效。

**CSV 格式** - 支持多列，自动识别 URL 列：
```csv
url,name,tags
https://example1.com,站点1,"tag1,tag2"
https://example2.com,站点2,"tag3"
```

或简单格式：
```csv
https://example1.com
https://example2.com
```

#### 导入流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           批量导入流程                                       │
└─────────────────────────────────────────────────────────────────────────────┘

     ┌──────────────┐
     │  上传文件     │  TXT / CSV (支持 500MB+)
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │  创建导入任务 │  返回 jobId，异步处理
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │  流式解析     │  逐行读取，低内存占用
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │  URL 验证     │  格式校验、规范化
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │  批量去重     │  数据库级别去重
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │  批量写入     │  每 1000 条批量 INSERT
     └──────┬───────┘
            │
            ▼
     ┌──────────────┐
     │  实时进度     │  WebSocket 推送
     └──────────────┘
```

#### 核心代码

```typescript
// src/targets/importer.ts

import { createReadStream } from 'fs';
import { createInterface } from 'readline';
import { parse as csvParse } from 'csv-parse';

class TargetImporter {
  private batchSize = 1000;        // 批量写入大小
  private progressInterval = 100;  // 进度上报间隔

  // 导入 TXT 文件
  async importTxt(
    filePath: string,
    jobId: string,
    groupId?: string
  ): Promise<void> {
    const stream = createReadStream(filePath, { encoding: 'utf-8' });
    const rl = createInterface({ input: stream });

    let batch: string[] = [];
    let processed = 0;
    let valid = 0;
    let duplicates = 0;
    let invalid = 0;

    for await (const line of rl) {
      processed++;
      const url = this.validateUrl(line.trim());

      if (!url) {
        invalid++;
        continue;
      }

      batch.push(url);

      // 批量写入
      if (batch.length >= this.batchSize) {
        const result = await this.batchInsert(batch, groupId);
        valid += result.inserted;
        duplicates += result.duplicates;
        batch = [];

        // 上报进度
        if (processed % this.progressInterval === 0) {
          await this.reportProgress(jobId, { processed, valid, duplicates, invalid });
        }
      }
    }

    // 处理剩余
    if (batch.length > 0) {
      const result = await this.batchInsert(batch, groupId);
      valid += result.inserted;
      duplicates += result.duplicates;
    }

    await this.completeJob(jobId, { processed, valid, duplicates, invalid });
  }

  // 导入 CSV 文件
  async importCsv(
    filePath: string,
    jobId: string,
    groupId?: string
  ): Promise<void> {
    const stream = createReadStream(filePath);
    const parser = csvParse({
      columns: true,              // 第一行作为列名
      skip_empty_lines: true,
      relaxColumnCount: true,
    });

    let batch: { url: string; name?: string; tags?: string[] }[] = [];
    let processed = 0;

    stream.pipe(parser);

    for await (const row of parser) {
      processed++;

      // 自动检测 URL 列
      const url = this.extractUrlFromRow(row);
      if (!url) continue;

      const validUrl = this.validateUrl(url);
      if (!validUrl) continue;  // 跳过无协议的 URL

      batch.push({
        url: validUrl,
        name: row.name || row.title || row.域名,
        tags: this.parseTags(row.tags || row.标签),
      });

      if (batch.length >= this.batchSize) {
        await this.batchInsertWithMeta(batch, groupId);
        batch = [];
        await this.reportProgress(jobId, { processed });
      }
    }

    if (batch.length > 0) {
      await this.batchInsertWithMeta(batch, groupId);
    }
  }

  // URL 验证（必须带协议）
  private validateUrl(url: string): string | null {
    if (!url) return null;

    // 必须以 http:// 或 https:// 开头，不自动补充
    if (!url.startsWith('http://') && !url.startsWith('https://')) {
      return null;  // 无效，缺少协议
    }

    try {
      const parsed = new URL(url);
      // 返回规范化的 URL（去除末尾斜杠等）
      return `${parsed.protocol}//${parsed.host}${parsed.pathname}`.replace(/\/$/, '');
    } catch {
      return null;
    }
  }

  // 批量插入（带去重）
  private async batchInsert(
    urls: string[],
    groupId?: string
  ): Promise<{ inserted: number; duplicates: number }> {
    // 使用 ON CONFLICT 实现数据库级别去重
    const result = await db.query(`
      INSERT INTO targets (id, url, domain, group_id, status, created_at)
      SELECT
        gen_random_uuid(),
        unnest($1::text[]),
        unnest($2::text[]),
        $3,
        'pending',
        NOW()
      ON CONFLICT (url) DO NOTHING
      RETURNING id
    `, [urls, urls.map(u => new URL(u).host), groupId]);

    return {
      inserted: result.rowCount,
      duplicates: urls.length - result.rowCount,
    };
  }

  // 从 CSV 行中提取 URL
  private extractUrlFromRow(row: Record<string, string>): string | null {
    // 尝试常见列名
    const urlColumns = ['url', 'URL', 'domain', 'site', 'website', '网址', '域名'];
    for (const col of urlColumns) {
      if (row[col]) return row[col];
    }
    // 如果只有一列，使用第一列
    const keys = Object.keys(row);
    if (keys.length === 1) return row[keys[0]];
    return null;
  }
}
```

### API 设计

```typescript
// 资产管理 API
POST   /api/v1/targets/import           // 上传并导入文件
GET    /api/v1/targets/import/:jobId    // 获取导入进度
GET    /api/v1/targets                  // 获取资产列表（分页）
GET    /api/v1/targets/:id              // 获取单个资产详情
DELETE /api/v1/targets/:id              // 删除资产
DELETE /api/v1/targets/batch            // 批量删除

// 分组管理
POST   /api/v1/targets/groups           // 创建分组
GET    /api/v1/targets/groups           // 获取分组列表
PUT    /api/v1/targets/groups/:id       // 更新分组
DELETE /api/v1/targets/groups/:id       // 删除分组

// 批量扫描
POST   /api/v1/targets/scan             // 扫描选中的资产
POST   /api/v1/targets/groups/:id/scan  // 扫描整个分组
```

### 导入 API 示例

```typescript
// POST /api/v1/targets/import
// Content-Type: multipart/form-data

// Request:
// - file: 上传的文件 (TXT/CSV)
// - groupId: 可选，导入到指定分组

// Response:
{
  "jobId": "import_abc123",
  "status": "processing",
  "filename": "targets.csv",
  "fileSize": 52428800  // 50MB
}

// 通过 WebSocket 获取实时进度
ws.subscribe('import:import_abc123');

// 进度事件
{
  "event": "import:progress",
  "data": {
    "jobId": "import_abc123",
    "progress": {
      "total": 500000,
      "processed": 125000,
      "valid": 124500,
      "duplicates": 300,
      "invalid": 200,
      "percent": 25
    }
  }
}

// 完成事件
{
  "event": "import:complete",
  "data": {
    "jobId": "import_abc123",
    "summary": {
      "total": 500000,
      "imported": 498000,
      "duplicates": 1500,
      "invalid": 500,
      "duration": 120  // 秒
    }
  }
}
```

### PostgreSQL Schema（大规模优化）

```sql
-- 目标资产表（优化大规模数据）
CREATE TABLE targets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  url VARCHAR(2048) NOT NULL,
  domain VARCHAR(255) NOT NULL,
  group_id UUID REFERENCES target_groups(id) ON DELETE SET NULL,
  tags TEXT[],
  status VARCHAR(20) NOT NULL DEFAULT 'pending',

  -- 最近扫描摘要（避免 JOIN）
  last_scan_id UUID,
  last_scan_at TIMESTAMP,
  is_wordpress BOOLEAN,
  wp_version VARCHAR(20),
  cloudflare BOOLEAN,
  vuln_count INTEGER DEFAULT 0,

  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),

  -- 唯一约束用于去重
  CONSTRAINT targets_url_unique UNIQUE (url)
);

-- 索引优化
CREATE INDEX idx_targets_domain ON targets (domain);
CREATE INDEX idx_targets_group ON targets (group_id);
CREATE INDEX idx_targets_status ON targets (status);
CREATE INDEX idx_targets_created ON targets (created_at DESC);
CREATE INDEX idx_targets_is_wp ON targets (is_wordpress) WHERE is_wordpress = true;
CREATE INDEX idx_targets_vuln ON targets (vuln_count DESC) WHERE vuln_count > 0;

-- 分区表（按创建时间，可选）
-- CREATE TABLE targets_2024 PARTITION OF targets
--   FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- 分组表
CREATE TABLE target_groups (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  description TEXT,
  target_count INTEGER DEFAULT 0,  -- 缓存计数
  created_at TIMESTAMP DEFAULT NOW()
);

-- 导入任务表
CREATE TABLE import_jobs (
  id VARCHAR(50) PRIMARY KEY,
  filename VARCHAR(255) NOT NULL,
  file_type VARCHAR(10) NOT NULL,
  file_size BIGINT NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending',

  -- 进度
  total_rows INTEGER DEFAULT 0,
  processed_rows INTEGER DEFAULT 0,
  valid_count INTEGER DEFAULT 0,
  duplicate_count INTEGER DEFAULT 0,
  invalid_count INTEGER DEFAULT 0,

  group_id UUID REFERENCES target_groups(id),
  error TEXT,

  created_at TIMESTAMP DEFAULT NOW(),
  completed_at TIMESTAMP
);
```

### 性能优化策略

```typescript
// 50万资产处理策略

const PERFORMANCE_CONFIG = {
  // 导入优化
  import: {
    batchSize: 1000,              // 每批插入 1000 条
    streamChunkSize: 64 * 1024,   // 64KB 流式读取
    maxConcurrentInserts: 4,      // 并发写入数
  },

  // 扫描优化
  scan: {
    batchSize: 10000,             // 每次从 DB 取 10000 个目标
    queueBatchSize: 100,          // 每次入队 100 个任务
    maxConcurrentScans: 1000,     // 最大并发扫描数
  },

  // 查询优化
  query: {
    defaultPageSize: 50,          // 默认分页大小
    maxPageSize: 1000,            // 最大分页大小
    useCountEstimate: true,       // 大表使用估算计数
  },
};

// 大表计数优化（避免 COUNT(*)）
async function getEstimatedCount(table: string): Promise<number> {
  const result = await db.query(`
    SELECT reltuples::bigint AS estimate
    FROM pg_class
    WHERE relname = $1
  `, [table]);
  return result.rows[0]?.estimate || 0;
}

// 游标分页（大数据集）
async function* iterateTargets(
  groupId?: string,
  batchSize = 10000
): AsyncGenerator<ITarget[]> {
  let cursor: string | null = null;

  while (true) {
    const result = await db.query(`
      SELECT * FROM targets
      WHERE ($1::uuid IS NULL OR group_id = $1)
        AND ($2::uuid IS NULL OR id > $2)
      ORDER BY id
      LIMIT $3
    `, [groupId, cursor, batchSize]);

    if (result.rows.length === 0) break;

    yield result.rows;
    cursor = result.rows[result.rows.length - 1].id;
  }
}
```

### CLI 命令

```bash
# 导入资产
wpsan import targets.txt                    # 导入 TXT
wpsan import targets.csv --group=group1    # 导入到指定分组
wpsan import targets.csv --tags=prod,cn    # 添加标签

# 资产管理
wpsan targets list                          # 列出资产
wpsan targets list --group=group1           # 按分组筛选
wpsan targets list --wp-only                # 只显示 WordPress 站点
wpsan targets count                         # 资产总数
wpsan targets export --format=csv           # 导出资产

# 批量扫描
wpsan scan --all                            # 扫描所有资产
wpsan scan --group=group1                   # 扫描指定分组
wpsan scan --limit=10000                    # 限制扫描数量
wpsan scan --status=pending                 # 只扫描待扫描的
```

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

### 2. Cloudflare 检测 (IP 段判断)

```typescript
// src/detectors/02-cloudflare/detector.ts

import { isIpInCidr } from '../utils/ip-utils.js';

interface ICloudflareDetectionResult {
  detected: boolean;
  ip: string;                // 目标站点解析的 IP
  matchedRange?: string;     // 匹配的 CIDR 段
}

// Cloudflare IP 段数据源
const CF_IP_SOURCES = {
  ipv4: 'https://www.cloudflare.com/ips-v4',
  ipv6: 'https://www.cloudflare.com/ips-v6',
};

// 本地缓存的 IP 段 (定期更新)
// data/cloudflare/ips-v4.txt
// data/cloudflare/ips-v6.txt

class CloudflareDetector {
  private ipv4Ranges: string[] = [];
  private ipv6Ranges: string[] = [];

  // 加载 IP 段
  async loadIpRanges(): Promise<void> {
    this.ipv4Ranges = await this.readIpFile('data/cloudflare/ips-v4.txt');
    this.ipv6Ranges = await this.readIpFile('data/cloudflare/ips-v6.txt');
  }

  // 从远程更新 IP 段
  async updateFromRemote(): Promise<void> {
    const [ipv4Response, ipv6Response] = await Promise.all([
      fetch(CF_IP_SOURCES.ipv4),
      fetch(CF_IP_SOURCES.ipv6),
    ]);

    const ipv4 = await ipv4Response.text();
    const ipv6 = await ipv6Response.text();

    await fs.writeFile('data/cloudflare/ips-v4.txt', ipv4.trim());
    await fs.writeFile('data/cloudflare/ips-v6.txt', ipv6.trim());

    this.ipv4Ranges = ipv4.trim().split('\n');
    this.ipv6Ranges = ipv6.trim().split('\n');
  }

  // 检测目标是否使用 Cloudflare
  async detect(target: string): Promise<ICloudflareDetectionResult> {
    // 1. 解析目标域名的 IP
    const hostname = new URL(target).hostname;
    const ip = await dns.resolve4(hostname).catch(() => null)
              || await dns.resolve6(hostname).catch(() => null);

    if (!ip || ip.length === 0) {
      return { detected: false, ip: 'unknown' };
    }

    const targetIp = ip[0];

    // 2. 判断 IP 是否在 Cloudflare 段内
    const ranges = targetIp.includes(':') ? this.ipv6Ranges : this.ipv4Ranges;

    for (const cidr of ranges) {
      if (isIpInCidr(targetIp, cidr)) {
        return {
          detected: true,
          ip: targetIp,
          matchedRange: cidr,
        };
      }
    }

    return { detected: false, ip: targetIp };
  }
}

// src/utils/ip-utils.ts

import { isInSubnet } from 'is-in-subnet';

export function isIpInCidr(ip: string, cidr: string): boolean {
  return isInSubnet(ip, cidr);
}
```

**Cloudflare IP 段更新脚本：**

```typescript
// scripts/update-cf-ips.ts

async function updateCloudflareIps() {
  console.log('Fetching Cloudflare IP ranges...');

  const [ipv4, ipv6] = await Promise.all([
    fetch('https://www.cloudflare.com/ips-v4').then(r => r.text()),
    fetch('https://www.cloudflare.com/ips-v6').then(r => r.text()),
  ]);

  await fs.writeFile('data/cloudflare/ips-v4.txt', ipv4.trim());
  await fs.writeFile('data/cloudflare/ips-v6.txt', ipv6.trim());

  console.log(`Updated: ${ipv4.trim().split('\n').length} IPv4 ranges`);
  console.log(`Updated: ${ipv6.trim().split('\n').length} IPv6 ranges`);
}

updateCloudflareIps();
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
];
```

### 4. 资产解析（插件/主题从 JSON 获取）

```typescript
// src/detectors/04-assets/parser.ts

interface IAsset {
  type: 'plugin' | 'theme';
  slug: string;
  name?: string;
  version?: string;          // 可能在这一步就能获取到
  source: string;            // 数据来源
}

interface IAssetsResult {
  plugins: IAsset[];
  themes: IAsset[];
}

class AssetParser {
  // 从多个来源解析资产
  async parse(target: string, html: string): Promise<IAssetsResult> {
    const results: IAssetsResult = { plugins: [], themes: [] };

    // 方法 1: 从 REST API 获取 (/wp-json/)
    const apiAssets = await this.parseFromRestApi(target);
    this.mergeAssets(results, apiAssets);

    // 方法 2: 从 HTML 源码解析
    const htmlAssets = this.parseFromHtml(html);
    this.mergeAssets(results, htmlAssets);

    return results;
  }

  // 从 REST API 解析
  private async parseFromRestApi(target: string): Promise<IAssetsResult> {
    const results: IAssetsResult = { plugins: [], themes: [] };

    try {
      // 尝试获取 /wp-json/ 根端点
      const response = await httpClient.get(`${target}/wp-json/`);
      const json = JSON.parse(response.body);

      // 解析 namespaces 中的插件信息
      // 例如: "wc/v3" => WooCommerce, "jetpack/v4" => Jetpack
      if (json.namespaces) {
        for (const ns of json.namespaces) {
          const plugin = this.namespaceToPlugin(ns);
          if (plugin) {
            results.plugins.push({
              type: 'plugin',
              slug: plugin.slug,
              name: plugin.name,
              source: 'rest_api_namespace',
            });
          }
        }
      }

      // 解析 authentication 中的插件信息
      if (json.authentication) {
        // ...
      }
    } catch (e) {
      // REST API 不可用，忽略
    }

    return results;
  }

  // 从 HTML 源码解析
  private parseFromHtml(html: string): IAssetsResult {
    const results: IAssetsResult = { plugins: [], themes: [] };

    // 提取插件路径: /wp-content/plugins/{slug}/
    const pluginMatches = html.matchAll(/wp-content\/plugins\/([a-z0-9_-]+)\//gi);
    for (const match of pluginMatches) {
      const slug = match[1];
      if (!results.plugins.find(p => p.slug === slug)) {
        results.plugins.push({
          type: 'plugin',
          slug,
          source: 'html_path',
        });
      }
    }

    // 提取主题路径: /wp-content/themes/{slug}/
    const themeMatches = html.matchAll(/wp-content\/themes\/([a-z0-9_-]+)\//gi);
    for (const match of themeMatches) {
      const slug = match[1];
      if (!results.themes.find(t => t.slug === slug)) {
        results.themes.push({
          type: 'theme',
          slug,
          source: 'html_path',
        });
      }
    }

    return results;
  }

  // 命名空间到插件的映射
  private namespaceToPlugin(namespace: string): { slug: string; name: string } | null {
    const mapping: Record<string, { slug: string; name: string }> = {
      'wc': { slug: 'woocommerce', name: 'WooCommerce' },
      'jetpack': { slug: 'jetpack', name: 'Jetpack' },
      'yoast': { slug: 'wordpress-seo', name: 'Yoast SEO' },
      'contact-form-7': { slug: 'contact-form-7', name: 'Contact Form 7' },
      'elementor': { slug: 'elementor', name: 'Elementor' },
      'wpforms': { slug: 'wpforms-lite', name: 'WPForms' },
      'acf': { slug: 'advanced-custom-fields', name: 'Advanced Custom Fields' },
      // 更多映射...
    };

    const prefix = namespace.split('/')[0];
    return mapping[prefix] || null;
  }
}
```

### 5. 插件版本检测

```typescript
// src/detectors/05-plugin-version/detector.ts

interface IPluginWithVersion {
  slug: string;
  name?: string;
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
    extract: (json: string) => {
      try {
        return JSON.parse(json).version;
      } catch {
        return null;
      }
    },
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

class PluginVersionDetector {
  async detect(target: string, plugins: IAsset[]): Promise<IPluginWithVersion[]> {
    const results: IPluginWithVersion[] = [];

    for (const plugin of plugins) {
      const version = await this.detectPluginVersion(target, plugin.slug);
      results.push({
        slug: plugin.slug,
        name: plugin.name,
        version: version?.version || null,
        versionSource: version?.source || 'unknown',
      });
    }

    return results;
  }

  private async detectPluginVersion(
    target: string,
    slug: string
  ): Promise<{ version: string; source: string } | null> {
    for (const source of VERSION_SOURCES) {
      const path = source.path.replace('{slug}', slug);
      const url = `${target}${path}`;

      try {
        const response = await httpClient.get(url);
        if (response.status !== 200) continue;

        // 使用自定义提取函数
        if (source.extract) {
          const version = source.extract(response.body);
          if (version) {
            return { version, source: source.name };
          }
        }

        // 使用正则模式匹配
        if (source.patterns) {
          for (const pattern of source.patterns) {
            const match = response.body.match(pattern);
            if (match?.[1]) {
              return { version: match[1], source: source.name };
            }
          }
        }
      } catch {
        // 忽略请求错误
      }
    }

    return null;
  }
}
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

// POC 上下文
interface IPocContext {
  target: string;
  http: HttpClient;
  scanResult: IScanResult;    // 当前扫描结果
}

// POC 结果
interface IPocResult {
  vulnerable: boolean;
  evidence?: string;          // 漏洞证据
  details?: Record<string, unknown>;
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
```

**POC 示例：**

```typescript
// src/poc/modules/plugins/elementor/CVE-2024-XXXX.poc.ts

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
    // 构造验证请求（仅验证，不利用）
    const response = await context.http.post(
      `${context.target}/wp-admin/admin-ajax.php`,
      {
        body: new URLSearchParams({
          action: 'elementor_test_action',
        }),
      }
    );

    // 判断漏洞是否存在
    const vulnerable = response.body.includes('specific_vulnerable_indicator');

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
  cloudflare: ICloudflareDetectionResult;
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
  canRun: boolean;                    // 是否可执行（无条件）
}

// API: 执行 POC
// POST /api/v1/scans/:scanId/pocs/:pocId/run
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
  type: 'full_scan' | 'poc_verify';
  target: string;
  options: {
    pocId?: string;           // POC 验证时的 POC ID
  };
  priority: number;           // 优先级
  userId: string;
}

// Worker 处理任务
const worker = new Worker('scan-tasks', async (job: Job<IScanJob>) => {
  const { type, target, options } = job.data;

  switch (type) {
    case 'full_scan':
      return await runFullScan(target, options, job);
    case 'poc_verify':
      return await runPocVerification(target, options);
  }
}, { connection: redis, concurrency: 5 });

// 任务进度上报
async function runFullScan(target: string, options: any, job: Job) {
  await job.updateProgress({ stage: 'wordpress', status: 'running' });
  const wpResult = await wordpressDetector.detect(target);

  await job.updateProgress({ stage: 'cloudflare', status: 'running' });
  const cfResult = await cloudflareDetector.detect(target);

  // ... 继续执行其他阶段
}
```

### Worker 扩展

```bash
# 手动扩展 Worker 数量
docker-compose up -d --scale worker=10
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

// 系统
GET    /api/v1/system/workers           // 获取 Worker 状态
GET    /api/v1/system/stats             // 获取系统统计
POST   /api/v1/system/cf-ips/update     // 更新 Cloudflare IP 段
```

### WebSocket 事件

```typescript
// 客户端连接后订阅扫描进度
ws.subscribe(`scan:${scanId}`);

// 服务端推送事件
ws.emit('scan:stage', { stage: 'cloudflare', status: 'running' });
ws.emit('scan:cloudflare', { detected: true, ip: '104.21.xx.xx' });
ws.emit('scan:plugin_found', { slug: 'woocommerce', version: '8.0.0' });
ws.emit('scan:vulnerability', { cve: 'CVE-2024-xxx', severity: 'high', pocs: [...] });
ws.emit('scan:complete', { summary: {...} });

// POC 执行
ws.emit('poc:started', { pocId: '...' });
ws.emit('poc:result', { pocId: '...', vulnerable: true, evidence: '...' });
```

---

## 数据模型

### PostgreSQL Schema

```sql
-- 扫描记录
CREATE TABLE scans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  target VARCHAR(500) NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, running, completed, failed

  -- 检测结果
  is_wordpress BOOLEAN,
  wp_version VARCHAR(20),
  cloudflare_detected BOOLEAN,
  cloudflare_ip VARCHAR(45),

  -- 时间戳
  created_at TIMESTAMP DEFAULT NOW(),
  started_at TIMESTAMP,
  completed_at TIMESTAMP,

  -- 关联
  user_id UUID REFERENCES users(id)
);

-- 扫描发现的插件
CREATE TABLE scan_plugins (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scan_id UUID REFERENCES scans(id) ON DELETE CASCADE,
  slug VARCHAR(100) NOT NULL,
  name VARCHAR(200),
  version VARCHAR(20),
  version_source VARCHAR(50)
);

-- 扫描发现的主题
CREATE TABLE scan_themes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scan_id UUID REFERENCES scans(id) ON DELETE CASCADE,
  slug VARCHAR(100) NOT NULL,
  name VARCHAR(200),
  version VARCHAR(20)
);

-- 扫描匹配的漏洞
CREATE TABLE scan_vulnerabilities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scan_id UUID REFERENCES scans(id) ON DELETE CASCADE,
  vuln_id VARCHAR(50) NOT NULL,  -- 关联漏洞库
  component_type VARCHAR(20) NOT NULL,  -- core, plugin, theme
  component_slug VARCHAR(100),
  component_version VARCHAR(20)
);

-- POC 执行记录
CREATE TABLE poc_executions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scan_id UUID REFERENCES scans(id) ON DELETE CASCADE,
  poc_id VARCHAR(100) NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  vulnerable BOOLEAN,
  evidence TEXT,
  executed_at TIMESTAMP DEFAULT NOW()
);

-- 漏洞库
CREATE TABLE vulnerabilities (
  id VARCHAR(50) PRIMARY KEY,
  cve VARCHAR(20),
  title VARCHAR(500) NOT NULL,
  description TEXT,

  component_type VARCHAR(20) NOT NULL,
  component_slug VARCHAR(100),
  affected_versions VARCHAR(100),  -- semver range
  fixed_version VARCHAR(20),

  severity VARCHAR(20) NOT NULL,
  cvss_score DECIMAL(3, 1),

  poc_ids TEXT[],  -- 关联的 POC ID 列表
  references TEXT[],

  published_at TIMESTAMP,
  updated_at TIMESTAMP DEFAULT NOW()
);

-- 索引
CREATE INDEX idx_scans_user ON scans(user_id);
CREATE INDEX idx_scans_status ON scans(status);
CREATE INDEX idx_scan_plugins_scan ON scan_plugins(scan_id);
CREATE INDEX idx_vulnerabilities_component ON vulnerabilities(component_type, component_slug);
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
    volumes:
      - ./data:/app/data  # Cloudflare IP 段等数据

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

- 检测器按执行顺序编号: `01-wordpress`, `02-cloudflare`...
- POC 按组件类型分目录: `wordpress/`, `plugins/`, `themes/`
- 插件 POC 再按插件 slug 分目录

### Git 提交规范

```
feat(detector): add Cloudflare IP detection
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

# Cloudflare IP 更新
npm run cf:update              # 从 cloudflare.com 更新 IP 段

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
- [ ] 核心扫描流程（6 个阶段）
- [ ] Cloudflare IP 段检测
- [ ] 资产解析（从 JSON/HTML）
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
