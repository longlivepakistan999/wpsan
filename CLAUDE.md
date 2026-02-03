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
├── app/
│   ├── Console/
│   │   └── Commands/            # CLI 命令
│   │       ├── ImportTargets.php
│   │       ├── ScanTargets.php
│   │       ├── RunPoc.php
│   │       └── UpdateCfIps.php
│   │
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── TargetController.php
│   │   │   ├── ScanController.php
│   │   │   ├── PocController.php
│   │   │   └── VulnController.php
│   │   └── Middleware/
│   │       └── ApiKeyAuth.php   # API Key 认证
│   │
│   ├── Models/                  # 数据模型
│   │   ├── Target.php
│   │   ├── TargetGroup.php
│   │   ├── ScanResult.php
│   │   ├── ScanPlugin.php
│   │   ├── ScanTheme.php
│   │   ├── Vulnerability.php
│   │   ├── Poc.php
│   │   └── ImportJob.php
│   │
│   ├── Services/                # 业务逻辑
│   │   ├── Scanner/
│   │   │   ├── ScannerService.php       # 扫描主服务
│   │   │   ├── WordPressDetector.php    # WP 识别
│   │   │   ├── CloudflareDetector.php   # CF 检测 (IP段)
│   │   │   ├── VersionDetector.php      # 版本检测
│   │   │   ├── AssetParser.php          # 资产解析
│   │   │   ├── PluginVersionDetector.php
│   │   │   └── ThemeVersionDetector.php
│   │   │
│   │   ├── Import/
│   │   │   ├── ImportService.php
│   │   │   ├── TxtParser.php
│   │   │   └── CsvParser.php
│   │   │
│   │   ├── Poc/
│   │   │   ├── PocRunner.php
│   │   │   ├── PocRegistry.php
│   │   │   └── BasePoc.php
│   │   │
│   │   └── HttpClient.php       # HTTP 客户端封装
│   │
│   ├── Jobs/                    # 队列任务
│   │   ├── ScanTargetJob.php
│   │   ├── RunPocJob.php
│   │   └── ImportFileJob.php
│   │
│   └── Events/                  # 事件
│       ├── ScanCompleted.php
│       ├── VulnerabilityFound.php
│       └── PocResultReady.php
│
├── config/
│   ├── wpsan.php                # 扫描配置（并发数等）
│   └── cloudflare.php           # CF IP 段配置
│
├── database/
│   └── migrations/              # 数据库迁移
│
├── pocs/                        # POC 模块目录
│   ├── wordpress/               # WP 核心漏洞 POC
│   ├── plugins/                 # 插件漏洞 POC
│   │   ├── Elementor/
│   │   ├── WooCommerce/
│   │   └── ...
│   └── themes/                  # 主题漏洞 POC
│
├── storage/
│   ├── app/
│   │   ├── imports/             # 导入文件
│   │   └── cloudflare/          # CF IP 段
│   │       ├── ips-v4.txt
│   │       └── ips-v6.txt
│   └── logs/
│
├── routes/
│   ├── api.php                  # API 路由
│   └── web.php
│
├── resources/
│   └── views/                   # 前端视图
│
├── tests/
├── docker-compose.yml
├── composer.json
└── .env
```

---

## 技术栈

### 核心技术
- **语言**: PHP 8.2+
- **框架**: Laravel 10+ 或原生 PHP
- **HTTP 客户端**: Guzzle
- **数据库**: MySQL 8.0+
- **缓存/队列**: Redis
- **消息队列**: Laravel Queue (Redis 驱动) 或 Supervisor + 自定义队列
- **WebSocket**: Laravel Reverb 或 Swoole

### 分布式
- **容器化**: Docker + Docker Compose
- **进程管理**: Supervisor
- **编排**: Kubernetes (可选)

### 前端 (Web UI)
- **框架**: Vue 3 + Inertia.js 或纯 Blade 模板
- **UI 组件**: Element Plus 或 Tailwind CSS

### 认证
- **无认证**: 单用户私有部署，不需要 API 认证
- **安全建议**: 通过防火墙/Nginx 限制访问 IP，或使用 VPN

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

```php
<?php
// app/Services/Import/ImportService.php

namespace App\Services\Import;

use App\Models\Target;
use App\Models\ImportJob;
use Illuminate\Support\Facades\DB;

class ImportService
{
    private int $batchSize = 1000;

    /**
     * 导入 TXT 文件
     */
    public function importTxt(string $filePath, string $jobId, ?string $groupId = null): void
    {
        $handle = fopen($filePath, 'r');
        $batch = [];
        $processed = 0;
        $valid = 0;
        $duplicates = 0;
        $invalid = 0;

        while (($line = fgets($handle)) !== false) {
            $processed++;
            $url = $this->validateUrl(trim($line));

            if (!$url) {
                $invalid++;
                continue;
            }

            $batch[] = $url;

            if (count($batch) >= $this->batchSize) {
                $result = $this->batchInsert($batch, $groupId);
                $valid += $result['inserted'];
                $duplicates += $result['duplicates'];
                $batch = [];

                $this->reportProgress($jobId, compact('processed', 'valid', 'duplicates', 'invalid'));
            }
        }

        fclose($handle);

        // 处理剩余
        if (!empty($batch)) {
            $result = $this->batchInsert($batch, $groupId);
            $valid += $result['inserted'];
            $duplicates += $result['duplicates'];
        }

        $this->completeJob($jobId, compact('processed', 'valid', 'duplicates', 'invalid'));
    }

    /**
     * 导入 CSV 文件
     */
    public function importCsv(string $filePath, string $jobId, ?string $groupId = null): void
    {
        $handle = fopen($filePath, 'r');
        $headers = fgetcsv($handle);
        $batch = [];
        $processed = 0;

        while (($row = fgetcsv($handle)) !== false) {
            $processed++;
            $data = array_combine($headers, $row);

            $url = $this->extractUrlFromRow($data);
            if (!$url) continue;

            $validUrl = $this->validateUrl($url);
            if (!$validUrl) continue;

            $batch[] = [
                'url' => $validUrl,
                'name' => $data['name'] ?? $data['title'] ?? null,
                'tags' => $this->parseTags($data['tags'] ?? null),
            ];

            if (count($batch) >= $this->batchSize) {
                $this->batchInsertWithMeta($batch, $groupId);
                $batch = [];
                $this->reportProgress($jobId, ['processed' => $processed]);
            }
        }

        fclose($handle);

        if (!empty($batch)) {
            $this->batchInsertWithMeta($batch, $groupId);
        }
    }

    /**
     * URL 验证（必须带协议）
     */
    private function validateUrl(string $url): ?string
    {
        if (empty($url)) return null;

        // 必须以 http:// 或 https:// 开头
        if (!str_starts_with($url, 'http://') && !str_starts_with($url, 'https://')) {
            return null;
        }

        $parsed = parse_url($url);
        if (!$parsed || !isset($parsed['host'])) {
            return null;
        }

        // 规范化 URL
        $normalized = $parsed['scheme'] . '://' . $parsed['host'];
        if (isset($parsed['path']) && $parsed['path'] !== '/') {
            $normalized .= rtrim($parsed['path'], '/');
        }

        return $normalized;
    }

    /**
     * 批量插入（带去重）
     */
    private function batchInsert(array $urls, ?string $groupId): array
    {
        $inserted = 0;

        foreach ($urls as $url) {
            $domain = parse_url($url, PHP_URL_HOST);

            // INSERT IGNORE 实现去重
            $result = DB::insert(
                'INSERT IGNORE INTO targets (url, domain, group_id, status, created_at)
                 VALUES (?, ?, ?, "pending", NOW())',
                [$url, $domain, $groupId]
            );

            if ($result) $inserted++;
        }

        return [
            'inserted' => $inserted,
            'duplicates' => count($urls) - $inserted,
        ];
    }

    /**
     * 从 CSV 行提取 URL
     */
    private function extractUrlFromRow(array $row): ?string
    {
        $urlColumns = ['url', 'URL', 'domain', 'site', 'website', '网址', '域名'];

        foreach ($urlColumns as $col) {
            if (!empty($row[$col])) {
                return $row[$col];
            }
        }

        // 只有一列则使用第一列
        if (count($row) === 1) {
            return array_values($row)[0];
        }

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

### MySQL Schema（大规模优化）

```sql
-- 目标资产表（优化大规模数据）
CREATE TABLE targets (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  url VARCHAR(2048) NOT NULL,
  domain VARCHAR(255) NOT NULL,
  group_id BIGINT UNSIGNED NULL,
  tags JSON,
  status VARCHAR(20) NOT NULL DEFAULT 'pending',

  -- 最近扫描摘要（避免 JOIN）
  last_scan_at DATETIME NULL,
  is_wordpress TINYINT(1) DEFAULT NULL,
  wp_version VARCHAR(20) DEFAULT NULL,
  cloudflare TINYINT(1) DEFAULT NULL,
  cloudflare_ip VARCHAR(45) DEFAULT NULL,
  vuln_count INT UNSIGNED DEFAULT 0,

  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

  -- 唯一约束用于去重
  UNIQUE KEY uk_targets_url (url(500)),

  -- 索引优化
  KEY idx_targets_domain (domain),
  KEY idx_targets_group (group_id),
  KEY idx_targets_status (status),
  KEY idx_targets_created (created_at DESC),
  KEY idx_targets_is_wp (is_wordpress),
  KEY idx_targets_vuln (vuln_count DESC),

  FOREIGN KEY (group_id) REFERENCES target_groups(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 分组表
CREATE TABLE target_groups (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  description TEXT,
  target_count INT UNSIGNED DEFAULT 0,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 导入任务表
CREATE TABLE import_jobs (
  id VARCHAR(50) PRIMARY KEY,
  filename VARCHAR(255) NOT NULL,
  file_type VARCHAR(10) NOT NULL,
  file_size BIGINT UNSIGNED NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending',

  -- 进度
  total_rows INT UNSIGNED DEFAULT 0,
  processed_rows INT UNSIGNED DEFAULT 0,
  valid_count INT UNSIGNED DEFAULT 0,
  duplicate_count INT UNSIGNED DEFAULT 0,
  invalid_count INT UNSIGNED DEFAULT 0,

  group_id BIGINT UNSIGNED NULL,
  error TEXT,

  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  completed_at DATETIME NULL,

  FOREIGN KEY (group_id) REFERENCES target_groups(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 扫描发现的插件
CREATE TABLE scan_plugins (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  target_id BIGINT UNSIGNED NOT NULL,
  slug VARCHAR(100) NOT NULL,
  name VARCHAR(200),
  version VARCHAR(20),
  version_source VARCHAR(50),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

  KEY idx_scan_plugins_target (target_id),
  KEY idx_scan_plugins_slug (slug),
  FOREIGN KEY (target_id) REFERENCES targets(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 扫描发现的主题
CREATE TABLE scan_themes (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  target_id BIGINT UNSIGNED NOT NULL,
  slug VARCHAR(100) NOT NULL,
  name VARCHAR(200),
  version VARCHAR(20),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

  KEY idx_scan_themes_target (target_id),
  FOREIGN KEY (target_id) REFERENCES targets(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 漏洞库（自维护）
CREATE TABLE vulnerabilities (
  id VARCHAR(50) PRIMARY KEY,
  cve VARCHAR(20),
  title VARCHAR(500) NOT NULL,
  description TEXT,

  component_type VARCHAR(20) NOT NULL,  -- core, plugin, theme
  component_slug VARCHAR(100),
  affected_versions VARCHAR(100),        -- semver range, 如 "< 3.5.0"
  fixed_version VARCHAR(20),

  severity VARCHAR(20) NOT NULL,         -- critical, high, medium, low
  cvss_score DECIMAL(3, 1),

  poc_id VARCHAR(100),                   -- 关联的 POC ID
  reference_urls JSON,                   -- 参考链接

  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

  KEY idx_vulns_component (component_type, component_slug),
  KEY idx_vulns_severity (severity)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- POC 执行记录
CREATE TABLE poc_executions (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  target_id BIGINT UNSIGNED NOT NULL,
  poc_id VARCHAR(100) NOT NULL,
  job_id VARCHAR(50),                    -- 批量任务 ID
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  vulnerable TINYINT(1),
  evidence TEXT,
  executed_at DATETIME DEFAULT CURRENT_TIMESTAMP,

  KEY idx_poc_target (target_id),
  KEY idx_poc_job (job_id),
  FOREIGN KEY (target_id) REFERENCES targets(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 探针状态表
CREATE TABLE probes (
  id VARCHAR(100) PRIMARY KEY,
  ip VARCHAR(45),
  concurrency INT UNSIGNED DEFAULT 10,
  status VARCHAR(20) DEFAULT 'offline',
  last_heartbeat DATETIME,
  tasks_completed INT UNSIGNED DEFAULT 0,
  registered_at DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 性能优化策略

```typescript
// 扫描配置

const SCAN_CONFIG = {
  // 并发控制（队列模式，非同时扫描）
  concurrency: {
    default: 10,                  // 默认并发数
    min: 1,
    max: 100,
    configurable: true,           // 用户可配置
  },

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
    skipScanned: true,            // 跳过已扫描的（避免重复）
    keepOnlyLastResult: true,     // 只保留最后一次扫描结果
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

### 7. POC 执行策略

**重要设计决策：**
- POC **不自动执行**，必须用户点击触发
- POC 按**插件维度**执行：选择某个插件漏洞 → 批量扫描所有包含该插件的资产
- 漏洞库**自己维护**，不依赖外部数据源

```typescript
// POC 批量执行接口

interface IPocBatchRunRequest {
  pocId: string;                      // 要执行的 POC
  targetIds?: string[];               // 指定目标（可选）
  groupId?: string;                   // 按分组执行（可选）
  pluginSlug: string;                 // 插件 slug
  pluginVersion?: string;             // 版本范围过滤（可选）
}

// 示例：对所有安装了 elementor 插件的资产执行某个 POC
// POST /api/v1/poc/batch-run
{
  "pocId": "elementor-rce-2024-xxxx",
  "pluginSlug": "elementor"
  // 不指定 targetIds 则扫描所有匹配的资产
}

// 执行流程
class PocBatchRunner {
  async run(request: IPocBatchRunRequest): Promise<string> {
    // 1. 查找所有包含该插件的资产
    const targets = await this.findTargetsWithPlugin(
      request.pluginSlug,
      request.pluginVersion,
      request.targetIds,
      request.groupId
    );

    // 2. 创建批量任务
    const jobId = generateJobId();

    // 3. 入队（按配置的并发数执行）
    for (const target of targets) {
      await pocQueue.add({
        jobId,
        pocId: request.pocId,
        targetId: target.id,
        targetUrl: target.url,
      });
    }

    return jobId;
  }

  // 查找包含指定插件的资产
  private async findTargetsWithPlugin(
    pluginSlug: string,
    version?: string,
    targetIds?: string[],
    groupId?: string
  ): Promise<ITarget[]> {
    let query = `
      SELECT DISTINCT t.* FROM targets t
      JOIN scan_plugins sp ON t.last_scan_id = sp.scan_id
      WHERE sp.slug = $1
    `;
    const params: any[] = [pluginSlug];

    if (version) {
      // 版本范围过滤（如 < 3.5.0）
      query += ` AND sp.version IS NOT NULL`;
    }
    if (targetIds?.length) {
      query += ` AND t.id = ANY($${params.length + 1})`;
      params.push(targetIds);
    }
    if (groupId) {
      query += ` AND t.group_id = $${params.length + 1}`;
      params.push(groupId);
    }

    return db.query(query, params);
  }
}

// POC 执行结果
interface IPocExecutionResult {
  targetId: string;
  targetUrl: string;
  pocId: string;
  vulnerable: boolean;
  evidence?: string;
  executedAt: Date;
}

// 发现漏洞时提示（WebSocket）
ws.emit('poc:vulnerable', {
  targetUrl: 'https://example.com',
  pocId: 'elementor-rce-2024-xxxx',
  pluginSlug: 'elementor',
  severity: 'critical',
  evidence: '...'
});
```

### 8. 漏洞库（自维护）

漏洞数据由用户自己添加和维护，不依赖外部 API。

```typescript
// 漏洞数据模型
interface IVulnerability {
  id: string;
  title: string;
  description: string;

  // 影响组件
  componentType: 'core' | 'plugin' | 'theme';
  componentSlug: string;
  affectedVersions: string;           // semver range, 如 "< 3.5.0"
  fixedVersion?: string;

  // 严重性
  severity: 'critical' | 'high' | 'medium' | 'low';

  // 关联 POC
  pocId?: string;

  // 参考信息
  cve?: string;
  references?: string[];

  createdAt: Date;
  updatedAt: Date;
}

// API: 漏洞管理（自维护）
POST   /api/v1/vulns                  // 添加漏洞
GET    /api/v1/vulns                  // 列表
GET    /api/v1/vulns/:id              // 详情
PUT    /api/v1/vulns/:id              // 更新
DELETE /api/v1/vulns/:id              // 删除

// CLI: 漏洞管理
wpsan vuln add --title="..." --plugin=elementor --versions="< 3.5.0"
wpsan vuln list
wpsan vuln import vulns.json          // 批量导入
```

---

## 分布式架构设计（探针模式）

采用**探针模式**部署，无需 Docker，每台服务器独立部署程序，数据统一上报到主服务器。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        WPSan 探针模式分布式架构                               │
└─────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────┐
                              │   Web UI    │
                              │ (主服务器)   │
                              └──────┬──────┘
                                     │
                                     ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                              主服务器 (Master)                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                    │
│  │  API Server │    │    Redis    │    │    MySQL    │                    │
│  │   (PHP)     │    │  (队列/缓存) │    │  (主数据库)  │                    │
│  └─────────────┘    └─────────────┘    └─────────────┘                    │
└────────────────────────────────────────────────────────────────────────────┘
          ▲                     ▲                     ▲
          │                     │                     │
          │     HTTP/WebSocket  │   Redis 连接         │   MySQL 连接
          │                     │                     │
    ┌─────┴─────┬───────────────┴───────────────┬─────┴─────┐
    │           │                               │           │
    ▼           ▼                               ▼           ▼
┌────────┐  ┌────────┐                     ┌────────┐  ┌────────┐
│ 探针 1  │  │ 探针 2  │        ...          │ 探针 N  │  │ 探针 N+1│
│ 北京    │  │ 上海    │                     │ 香港    │  │ 美国    │
└────────┘  └────────┘                     └────────┘  └────────┘
   独立部署    独立部署                        独立部署    独立部署
```

### 探针模式特点

| 特点 | 说明 |
|------|------|
| **独立部署** | 每台服务器运行独立的探针程序，无需 Docker |
| **数据集中** | 所有扫描结果直接写入主服务器 MySQL |
| **队列共享** | 探针从主服务器 Redis 拉取任务 |
| **易于扩展** | 新增节点只需部署探针程序并配置连接信息 |
| **地理分布** | 可部署在不同地区，提高扫描覆盖 |

### 组件职责

| 组件 | 部署位置 | 职责 |
|------|----------|------|
| **Web UI** | 主服务器 | 用户界面，任务管理，结果展示 |
| **API Server** | 主服务器 | RESTful API，WebSocket 推送 |
| **MySQL** | 主服务器 | 所有数据持久化（资产、扫描结果、漏洞库） |
| **Redis** | 主服务器 | 任务队列、缓存、实时状态 |
| **探针 (Probe)** | 分布式部署 | 拉取任务、执行扫描、上报结果 |

### 探针程序设计

```php
<?php
// probe/ProbeWorker.php

namespace WPSan\Probe;

use Predis\Client as Redis;
use PDO;

class ProbeWorker
{
    private Redis $redis;
    private PDO $db;
    private string $probeId;
    private int $concurrency;

    public function __construct()
    {
        // 从配置文件读取主服务器连接信息
        $config = require __DIR__ . '/config.php';

        $this->probeId = $config['probe_id'] ?? gethostname();
        $this->concurrency = $config['concurrency'] ?? 10;

        // 连接主服务器 Redis
        $this->redis = new Redis([
            'scheme' => 'tcp',
            'host'   => $config['redis_host'],
            'port'   => $config['redis_port'],
            'password' => $config['redis_password'] ?? null,
        ]);

        // 连接主服务器 MySQL
        $this->db = new PDO(
            "mysql:host={$config['mysql_host']};dbname={$config['mysql_database']};charset=utf8mb4",
            $config['mysql_user'],
            $config['mysql_password'],
            [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
        );
    }

    /**
     * 启动探针
     */
    public function run(): void
    {
        echo "[Probe:{$this->probeId}] Started with concurrency: {$this->concurrency}\n";

        // 注册探针
        $this->registerProbe();

        // 心跳进程
        $this->startHeartbeat();

        // 工作循环
        while (true) {
            // 从队列拉取任务
            $task = $this->redis->brpop('wpsan:scan:queue', 5);

            if ($task) {
                $this->processTask(json_decode($task[1], true));
            }

            // 检查 POC 任务队列
            $pocTask = $this->redis->brpop('wpsan:poc:queue', 1);
            if ($pocTask) {
                $this->processPocTask(json_decode($pocTask[1], true));
            }
        }
    }

    /**
     * 注册探针到主服务器
     */
    private function registerProbe(): void
    {
        $this->redis->hset('wpsan:probes', $this->probeId, json_encode([
            'id' => $this->probeId,
            'ip' => $this->getLocalIp(),
            'concurrency' => $this->concurrency,
            'status' => 'online',
            'registered_at' => date('Y-m-d H:i:s'),
        ]));
    }

    /**
     * 心跳上报
     */
    private function startHeartbeat(): void
    {
        // 使用 pcntl_fork 或独立进程
        $this->redis->hset('wpsan:probes:heartbeat', $this->probeId, time());
    }

    /**
     * 处理扫描任务
     */
    private function processTask(array $task): void
    {
        $targetId = $task['target_id'];
        $targetUrl = $task['target_url'];

        echo "[Probe:{$this->probeId}] Scanning: {$targetUrl}\n";

        try {
            // 更新状态为扫描中
            $this->updateTargetStatus($targetId, 'scanning');

            // 执行扫描流程
            $scanner = new ScannerService();
            $result = $scanner->scan($targetUrl);

            // 直接写入主服务器数据库
            $this->saveScanResult($targetId, $result);

            // 更新状态为完成
            $this->updateTargetStatus($targetId, 'completed');

            // 发现漏洞时通知
            if (!empty($result['vulnerabilities'])) {
                $this->notifyVulnerabilities($targetId, $targetUrl, $result['vulnerabilities']);
            }

        } catch (\Exception $e) {
            $this->updateTargetStatus($targetId, 'failed');
            $this->logError($targetId, $e->getMessage());
        }
    }

    /**
     * 保存扫描结果到主数据库
     */
    private function saveScanResult(int $targetId, array $result): void
    {
        // 删除旧的扫描结果（只保留最后一次）
        $this->db->prepare('DELETE FROM scan_plugins WHERE target_id = ?')->execute([$targetId]);
        $this->db->prepare('DELETE FROM scan_themes WHERE target_id = ?')->execute([$targetId]);

        // 更新目标摘要
        $stmt = $this->db->prepare('
            UPDATE targets SET
                is_wordpress = ?,
                wp_version = ?,
                cloudflare = ?,
                cloudflare_ip = ?,
                vuln_count = ?,
                last_scan_at = NOW(),
                updated_at = NOW()
            WHERE id = ?
        ');
        $stmt->execute([
            $result['is_wordpress'] ? 1 : 0,
            $result['wp_version'],
            $result['cloudflare']['detected'] ? 1 : 0,
            $result['cloudflare']['ip'] ?? null,
            count($result['vulnerabilities'] ?? []),
            $targetId,
        ]);

        // 插入插件信息
        foreach ($result['plugins'] ?? [] as $plugin) {
            $stmt = $this->db->prepare('
                INSERT INTO scan_plugins (target_id, slug, name, version, version_source, created_at)
                VALUES (?, ?, ?, ?, ?, NOW())
            ');
            $stmt->execute([
                $targetId,
                $plugin['slug'],
                $plugin['name'] ?? null,
                $plugin['version'] ?? null,
                $plugin['version_source'] ?? null,
            ]);
        }

        // 插入主题信息
        foreach ($result['themes'] ?? [] as $theme) {
            $stmt = $this->db->prepare('
                INSERT INTO scan_themes (target_id, slug, name, version, created_at)
                VALUES (?, ?, ?, ?, NOW())
            ');
            $stmt->execute([
                $targetId,
                $theme['slug'],
                $theme['name'] ?? null,
                $theme['version'] ?? null,
            ]);
        }
    }

    /**
     * 通知发现的漏洞
     */
    private function notifyVulnerabilities(int $targetId, string $url, array $vulns): void
    {
        foreach ($vulns as $vuln) {
            $this->redis->publish('wpsan:vulnerabilities', json_encode([
                'target_id' => $targetId,
                'target_url' => $url,
                'probe_id' => $this->probeId,
                'vulnerability' => $vuln,
                'found_at' => date('Y-m-d H:i:s'),
            ]));
        }
    }

    private function getLocalIp(): string
    {
        return gethostbyname(gethostname());
    }
}

// 启动探针
$worker = new ProbeWorker();
$worker->run();
```

### 探针配置文件

```php
<?php
// probe/config.php

return [
    // 探针标识（唯一）
    'probe_id' => env('PROBE_ID', 'probe-' . gethostname()),

    // 并发数
    'concurrency' => (int) env('PROBE_CONCURRENCY', 10),

    // 主服务器 Redis
    'redis_host' => env('MASTER_REDIS_HOST', '主服务器IP'),
    'redis_port' => (int) env('MASTER_REDIS_PORT', 6379),
    'redis_password' => env('MASTER_REDIS_PASSWORD', null),

    // 主服务器 MySQL
    'mysql_host' => env('MASTER_MYSQL_HOST', '主服务器IP'),
    'mysql_port' => (int) env('MASTER_MYSQL_PORT', 3306),
    'mysql_database' => env('MASTER_MYSQL_DATABASE', 'wpsan'),
    'mysql_user' => env('MASTER_MYSQL_USER', 'wpsan'),
    'mysql_password' => env('MASTER_MYSQL_PASSWORD', ''),
];
```

### 探针部署步骤

```bash
# 1. 在探针服务器上克隆代码
git clone https://github.com/your-repo/wpsan.git
cd wpsan/probe

# 2. 安装依赖
composer install

# 3. 配置连接信息
cp .env.example .env
vim .env
# 设置 MASTER_REDIS_HOST, MASTER_MYSQL_HOST 等

# 4. 启动探针
php probe.php

# 5. 使用 Supervisor 守护进程（推荐）
sudo cp wpsan-probe.conf /etc/supervisor/conf.d/
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start wpsan-probe
```

### Supervisor 配置

```ini
; /etc/supervisor/conf.d/wpsan-probe.conf

[program:wpsan-probe]
command=php /opt/wpsan/probe/probe.php
directory=/opt/wpsan/probe
user=www-data
autostart=true
autorestart=true
stderr_logfile=/var/log/wpsan-probe.err.log
stdout_logfile=/var/log/wpsan-probe.out.log
environment=PROBE_ID="probe-%(host_node_name)s",PROBE_CONCURRENCY="10"
```

### 探针管理 API

```php
// 主服务器 API

// 获取所有探针状态
GET /api/v1/probes
// Response:
{
    "probes": [
        {
            "id": "probe-beijing-01",
            "ip": "192.168.1.100",
            "status": "online",
            "concurrency": 10,
            "last_heartbeat": "2024-01-15 10:30:00",
            "tasks_completed": 1523
        },
        {
            "id": "probe-shanghai-01",
            "ip": "192.168.2.100",
            "status": "online",
            "concurrency": 20,
            "last_heartbeat": "2024-01-15 10:30:05",
            "tasks_completed": 2341
        }
    ]
}

// 向指定探针分配任务（可选，默认自动分配）
POST /api/v1/probes/{probeId}/tasks
{
    "target_ids": [1, 2, 3]
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

## 主服务器部署

### 系统要求

- PHP 8.2+
- MySQL 8.0+
- Redis 6.0+
- Composer 2.x
- Nginx/Apache

### 部署步骤

```bash
# 1. 克隆代码
git clone https://github.com/your-repo/wpsan.git
cd wpsan

# 2. 安装 PHP 依赖
composer install --optimize-autoloader --no-dev

# 3. 配置环境变量
cp .env.example .env
vim .env

# 4. 数据库迁移
php artisan migrate

# 5. 生成应用密钥
php artisan key:generate

# 6. 更新 Cloudflare IP 段
php artisan wpsan:update-cf-ips

# 7. 启动队列处理器
php artisan queue:work redis --queue=scan,poc --tries=3

# 8. 配置 Nginx
sudo cp nginx.conf /etc/nginx/sites-available/wpsan
sudo ln -s /etc/nginx/sites-available/wpsan /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 环境变量配置

```bash
# .env

APP_NAME=WPSan
APP_ENV=production
APP_DEBUG=false
APP_URL=https://your-domain.com

# 数据库
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=wpsan
DB_USERNAME=wpsan
DB_PASSWORD=your-db-password

# Redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

# 队列
QUEUE_CONNECTION=redis

# 扫描配置
SCAN_CONCURRENCY=10
SCAN_SKIP_SCANNED=true
```

### Supervisor 配置（队列处理器）

```ini
; /etc/supervisor/conf.d/wpsan-queue.conf

[program:wpsan-queue]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/wpsan/artisan queue:work redis --queue=scan,poc --sleep=3 --tries=3
autostart=true
autorestart=true
user=www-data
numprocs=4
redirect_stderr=true
stdout_logfile=/var/log/wpsan-queue.log

[program:wpsan-scheduler]
command=php /var/www/wpsan/artisan schedule:work
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/var/log/wpsan-scheduler.log
```

### Nginx 配置

```nginx
server {
    listen 80;
    server_name your-domain.com;
    root /var/www/wpsan/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}

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
