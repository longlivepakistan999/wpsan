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

## 项目初始化

### 创建项目目录结构

```bash
# 创建主目录
mkdir -p wpsan/{public/assets/{css,js},src/{Core,Scanner,Import,Poc,Queue/Jobs,Api,Model,Utils},pocs/{wordpress,plugins,themes},probe,config,data/cloudflare,storage/{imports,logs,cache},views/{targets,scans,vulns,pocs,probes,logs},bin,sql}

# 创建必要文件
touch wpsan/public/{index.php,api.php}
touch wpsan/src/Core/{App.php,Router.php,Database.php,Redis.php,Config.php,Logger.php}
touch wpsan/src/Scanner/{ScannerService.php,WordPressDetector.php,CloudflareDetector.php,VersionDetector.php,AssetParser.php,PluginVersionDetector.php,ThemeVersionDetector.php}
touch wpsan/src/Import/{ImportService.php,TxtParser.php,CsvParser.php}
touch wpsan/src/Poc/{PocRunner.php,PocRegistry.php,BasePoc.php}
touch wpsan/src/Queue/{QueueManager.php,Worker.php}
touch wpsan/src/Api/{TargetApi.php,ScanApi.php,PocApi.php,VulnApi.php,ProbeApi.php,ExportApi.php}
touch wpsan/src/Model/{Project.php,Target.php,ScanPlugin.php,ScanTheme.php,Vulnerability.php,PocExecution.php}
touch wpsan/src/Utils/{HttpClient.php,IpUtils.php,Validator.php}
touch wpsan/config/{app.php,database.php,scan.php}
touch wpsan/bin/{worker.php,import.php,scan.php,update-cf-ips.php}
touch wpsan/sql/schema.sql
touch wpsan/{composer.json,.env.example}
touch wpsan/probe/{probe.php,ProbeWorker.php,config.php}
```

### 初始化 Composer

```json
// composer.json
{
    "name": "wpsan/wpsan",
    "description": "WordPress Security Analyzer",
    "type": "project",
    "require": {
        "php": ">=8.2",
        "predis/predis": "^2.2",
        "guzzlehttp/guzzle": "^7.8",
        "workerman/workerman": "^4.1"
    },
    "autoload": {
        "psr-4": {
            "WPSan\\": "src/"
        }
    }
}
```

```bash
composer install
```

---

## 开发任务列表

按模块组织的开发任务，用于跟踪项目进度。

### 1. 核心框架 (Core)

| 任务 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 应用入口 | `src/Core/App.php` | ⬜ 待开发 | 应用初始化、配置加载 |
| 路由器 | `src/Core/Router.php` | ⬜ 待开发 | API 路由分发 |
| 数据库连接 | `src/Core/Database.php` | ⬜ 待开发 | PDO 封装、连接池 |
| Redis 连接 | `src/Core/Redis.php` | ⬜ 待开发 | Predis 封装 |
| 配置管理 | `src/Core/Config.php` | ⬜ 待开发 | 环境变量、配置文件读取 |
| 日志系统 | `src/Core/Logger.php` | ⬜ 待开发 | 操作日志、错误日志 |

### 2. 扫描模块 (Scanner)

| 任务 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 扫描主服务 | `src/Scanner/ScannerService.php` | ⬜ 待开发 | 扫描流程编排 |
| WordPress 识别 | `src/Scanner/WordPressDetector.php` | ⬜ 待开发 | WP 特征检测 |
| Cloudflare 检测 | `src/Scanner/CloudflareDetector.php` | ⬜ 待开发 | IP 段判断 |
| 版本检测 | `src/Scanner/VersionDetector.php` | ⬜ 待开发 | WP 核心版本 |
| 资产解析 | `src/Scanner/AssetParser.php` | ⬜ 待开发 | 从 JSON/HTML 提取插件主题 |
| 插件版本检测 | `src/Scanner/PluginVersionDetector.php` | ⬜ 待开发 | readme.txt 等方式 |
| 主题版本检测 | `src/Scanner/ThemeVersionDetector.php` | ⬜ 待开发 | style.css 等方式 |

### 3. 导入模块 (Import)

| 任务 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 导入服务 | `src/Import/ImportService.php` | ⬜ 待开发 | 批量导入主逻辑 |
| TXT 解析器 | `src/Import/TxtParser.php` | ⬜ 待开发 | 逐行解析 URL |
| CSV 解析器 | `src/Import/CsvParser.php` | ⬜ 待开发 | CSV 列识别 |

### 4. POC 模块 (Poc)

| 任务 | 文件 | 状态 | 说明 |
|------|------|------|------|
| POC 基类 | `src/Poc/BasePoc.php` | ⬜ 待开发 | 抽象类定义 |
| POC 注册表 | `src/Poc/PocRegistry.php` | ⬜ 待开发 | 动态加载、版本匹配 |
| POC 执行器 | `src/Poc/PocRunner.php` | ⬜ 待开发 | 批量执行、结果收集 |

### 5. 队列模块 (Queue)

| 任务 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 队列管理器 | `src/Queue/QueueManager.php` | ⬜ 待开发 | Redis 队列操作 |
| 队列消费者 | `src/Queue/Worker.php` | ⬜ 待开发 | 任务消费、并发控制 |
| 扫描任务 | `src/Queue/Jobs/ScanJob.php` | ⬜ 待开发 | 扫描任务封装 |
| POC 任务 | `src/Queue/Jobs/PocJob.php` | ⬜ 待开发 | POC 执行任务 |
| 导入任务 | `src/Queue/Jobs/ImportJob.php` | ⬜ 待开发 | 文件导入任务 |

### 6. API 模块 (Api)

| 任务 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 资产 API | `src/Api/TargetApi.php` | ⬜ 待开发 | CRUD、导入、分组 |
| 扫描 API | `src/Api/ScanApi.php` | ⬜ 待开发 | 启动扫描、状态查询 |
| POC API | `src/Api/PocApi.php` | ⬜ 待开发 | POC 列表、批量执行 |
| 漏洞 API | `src/Api/VulnApi.php` | ⬜ 待开发 | 漏洞库管理 |
| 探针 API | `src/Api/ProbeApi.php` | ⬜ 待开发 | 探针状态、任务分配 |
| 导出 API | `src/Api/ExportApi.php` | ⬜ 待开发 | CSV/JSON 导出 |

### 7. 数据模型 (Model)

| 任务 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 项目模型 | `src/Model/Project.php` | ⬜ 待开发 | 项目管理（资产归属于项目） |
| 目标模型 | `src/Model/Target.php` | ⬜ 待开发 | 资产 CRUD |
| 插件模型 | `src/Model/ScanPlugin.php` | ⬜ 待开发 | 扫描发现的插件 |
| 主题模型 | `src/Model/ScanTheme.php` | ⬜ 待开发 | 扫描发现的主题 |
| 漏洞模型 | `src/Model/Vulnerability.php` | ⬜ 待开发 | 漏洞库 |
| POC 执行记录 | `src/Model/PocExecution.php` | ⬜ 待开发 | POC 结果 |

### 8. 工具类 (Utils)

| 任务 | 文件 | 状态 | 说明 |
|------|------|------|------|
| HTTP 客户端 | `src/Utils/HttpClient.php` | ⬜ 待开发 | cURL/Guzzle 封装 |
| IP 工具 | `src/Utils/IpUtils.php` | ⬜ 待开发 | CIDR 匹配 |
| 验证器 | `src/Utils/Validator.php` | ⬜ 待开发 | URL 验证等 |

### 9. 前端视图 (Views)

| 任务 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 公共布局 | `views/layout.php` | ⬜ 待开发 | 侧边栏、头部 |
| 仪表盘 | `views/dashboard.php` | ✅ HTML完成 | 统计概览、项目概览 |
| 项目列表 | `views/projects/index.php` | ✅ HTML完成 | 项目管理 |
| 项目详情 | `views/projects/detail.php` | ✅ HTML完成 | 项目内资产、漏洞、扫描记录 |
| 扫描详情 | `views/scans/detail.php` | ✅ HTML完成 | 单个资产扫描结果 |
| 漏洞库 | `views/vulns/index.php` | ✅ HTML完成 | 漏洞列表（全局） |
| POC 管理 | `views/pocs/index.php` | ✅ HTML完成 | POC 列表和执行（全局） |
| 探针状态 | `views/probes/index.php` | ✅ HTML完成 | 探针监控 |
| 操作日志 | `views/logs/index.php` | ✅ HTML完成 | 审计日志 |

### 10. 探针程序 (Probe)

| 任务 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 探针入口 | `probe/probe.php` | ⬜ 待开发 | 启动脚本 |
| 探针工作器 | `probe/ProbeWorker.php` | ⬜ 待开发 | 任务拉取、执行、上报 |
| 探针配置 | `probe/config.php` | ⬜ 待开发 | 主服务器连接信息 |

### 11. CLI 脚本 (Bin)

| 任务 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 队列 Worker | `bin/worker.php` | ⬜ 待开发 | 启动队列处理 |
| 导入脚本 | `bin/import.php` | ⬜ 待开发 | 命令行导入 |
| 扫描脚本 | `bin/scan.php` | ⬜ 待开发 | 命令行扫描 |
| CF IP 更新 | `bin/update-cf-ips.php` | ⬜ 待开发 | 更新 Cloudflare IP |

### 12. 数据库 (SQL)

| 任务 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 数据库结构 | `sql/schema.sql` | ⬜ 待开发 | 建表语句 |

### 13. WebSocket (实时通知)

| 任务 | 文件 | 状态 | 说明 |
|------|------|------|------|
| WS 服务器 | `src/WebSocket/Server.php` | ⬜ 待开发 | Workerman 服务 |
| 通知推送 | `src/WebSocket/Notifier.php` | ⬜ 待开发 | 频道广播 |

---

## 开发优先级

建议按以下顺序开发：

```
Phase 1: 基础框架
├── Core (App, Config, Database, Redis, Logger)
├── Utils (HttpClient, IpUtils, Validator)
└── SQL Schema

Phase 2: 扫描核心
├── Scanner (全部检测器)
├── Model (Target, ScanPlugin, ScanTheme)
└── Queue (QueueManager, Worker, ScanJob)

Phase 3: 资产管理
├── Import (ImportService, TxtParser, CsvParser)
├── Api (TargetApi, ScanApi)
└── Views (dashboard, targets)

Phase 4: POC 系统
├── Poc (BasePoc, PocRegistry, PocRunner)
├── Model (Vulnerability, PocExecution)
├── Api (PocApi, VulnApi)
└── Views (pocs, vulns)

Phase 5: 分布式
├── Probe (ProbeWorker)
├── Api (ProbeApi)
└── Views (probes)

Phase 6: 完善功能
├── Export (ExportApi)
├── WebSocket (实时通知)
├── Logs (操作日志)
└── 前端交互完善
```

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
├── public/                      # Web 入口
│   ├── index.php                # 主入口
│   ├── api.php                  # API 入口
│   └── assets/                  # 静态资源
│       ├── css/
│       └── js/
│
├── src/                         # 核心源码
│   ├── Core/                    # 核心类
│   │   ├── App.php              # 应用主类
│   │   ├── Router.php           # 路由器
│   │   ├── Database.php         # 数据库连接 (PDO)
│   │   ├── Redis.php            # Redis 连接
│   │   ├── Config.php           # 配置管理
│   │   └── Logger.php           # 日志
│   │
│   ├── Scanner/                 # 扫描模块
│   │   ├── ScannerService.php   # 扫描主服务
│   │   ├── WordPressDetector.php
│   │   ├── CloudflareDetector.php
│   │   ├── VersionDetector.php
│   │   ├── AssetParser.php
│   │   ├── PluginVersionDetector.php
│   │   └── ThemeVersionDetector.php
│   │
│   ├── Import/                  # 导入模块
│   │   ├── ImportService.php
│   │   ├── TxtParser.php
│   │   └── CsvParser.php
│   │
│   ├── Poc/                     # POC 模块
│   │   ├── PocRunner.php
│   │   ├── PocRegistry.php
│   │   └── BasePoc.php
│   │
│   ├── Queue/                   # 队列模块
│   │   ├── QueueManager.php     # 队列管理器
│   │   ├── Worker.php           # 队列消费者
│   │   └── Jobs/
│   │       ├── ScanJob.php
│   │       ├── PocJob.php
│   │       └── ImportJob.php
│   │
│   ├── Api/                     # API 控制器
│   │   ├── TargetApi.php
│   │   ├── ScanApi.php
│   │   ├── PocApi.php
│   │   ├── VulnApi.php
│   │   └── ProbeApi.php
│   │
│   ├── Model/                   # 数据模型
│   │   ├── Project.php          # 项目（资产归属于项目）
│   │   ├── Target.php
│   │   ├── ScanPlugin.php
│   │   ├── ScanTheme.php
│   │   ├── Vulnerability.php
│   │   └── PocExecution.php
│   │
│   └── Utils/                   # 工具类
│       ├── HttpClient.php       # cURL 封装
│       ├── IpUtils.php          # IP 地址工具
│       └── Validator.php        # 验证器
│
├── pocs/                        # POC 模块目录
│   ├── wordpress/               # WP 核心漏洞
│   ├── plugins/                 # 插件漏洞
│   │   ├── elementor/
│   │   ├── woocommerce/
│   │   └── ...
│   └── themes/                  # 主题漏洞
│
├── probe/                       # 探针程序
│   ├── probe.php                # 探针入口
│   ├── ProbeWorker.php
│   └── config.php
│
├── config/                      # 配置文件
│   ├── app.php                  # 应用配置
│   ├── database.php             # 数据库配置
│   └── scan.php                 # 扫描配置
│
├── data/                        # 数据文件
│   └── cloudflare/              # CF IP 段
│       ├── ips-v4.txt
│       └── ips-v6.txt
│
├── storage/                     # 存储目录
│   ├── imports/                 # 导入文件
│   ├── logs/                    # 日志
│   └── cache/                   # 缓存
│
├── views/                       # 视图模板
│   ├── layout.php
│   ├── dashboard.php
│   ├── projects/                # 项目管理
│   ├── scans/
│   └── vulns/
│
├── bin/                         # CLI 脚本
│   ├── worker.php               # 队列 Worker
│   ├── import.php               # 导入命令
│   ├── scan.php                 # 扫描命令
│   └── update-cf-ips.php        # 更新 CF IP
│
├── sql/                         # SQL 文件
│   └── schema.sql               # 数据库结构
│
├── vendor/                      # Composer 依赖
├── composer.json
├── .env                         # 环境变量
└── .env.example
```

---

## 技术栈

### 核心技术
- **语言**: PHP 8.2+
- **框架**: 原生 PHP（无框架）
- **HTTP 客户端**: cURL / Guzzle
- **数据库**: MySQL 8.0+ (PDO)
- **缓存/队列**: Redis (Predis)
- **消息队列**: 自定义 Redis 队列 + Supervisor
- **WebSocket**: Swoole 或 Workerman

### 分布式
- **部署方式**: 探针模式（无 Docker）
- **进程管理**: Supervisor

### 前端 (Web UI)
- **模板**: 原生 PHP 模板
- **样式**: Bootstrap 5 或 Tailwind CSS
- **交互**: 原生 JavaScript / jQuery

### 认证
- **无认证**: 单用户私有部署，不需要 API 认证
- **安全建议**: 通过防火墙/Nginx 限制访问 IP，或使用 VPN

---

## 项目化资产管理（支持 50万+ 目标）

### 架构说明

采用**项目（Project）**作为顶层组织单位，资产必须归属于某个项目：

```
项目 (Project)
└── 资产 (Target)

POC / 漏洞库 → 全局共享（不分项目）
```

### 数据模型

```typescript
// 项目
interface IProject {
  id: string;
  name: string;                   // 项目名称
  description?: string;           // 项目描述

  // 统计摘要（定期更新）
  stats: {
    targetCount: number;          // 资产总数
    wpCount: number;              // WordPress 站点数
    vulnCount: number;            // 发现漏洞数
    scannedCount: number;         // 已扫描数
    scanProgress: number;         // 扫描进度 0-100
  };

  lastScanAt?: Date;              // 最后扫描时间
  createdAt: Date;
  updatedAt: Date;
}

// 目标资产
interface ITarget {
  id: string;
  projectId: string;              // 所属项目 ID（必须）
  url: string;                    // 目标 URL
  domain: string;                 // 域名（自动提取）
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

// 导入任务
interface IImportJob {
  id: string;
  projectId: string;              // 导入到的项目（必须）
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
// 项目管理 API
POST   /api/v1/projects                         // 创建项目
GET    /api/v1/projects                         // 获取项目列表
GET    /api/v1/projects/:id                     // 获取项目详情
PUT    /api/v1/projects/:id                     // 更新项目
DELETE /api/v1/projects/:id                     // 删除项目

// 项目内资产管理 API
POST   /api/v1/projects/:projectId/targets/import    // 导入资产到项目
GET    /api/v1/projects/:projectId/targets/import/:jobId  // 获取导入进度
GET    /api/v1/projects/:projectId/targets           // 获取项目内资产列表
POST   /api/v1/projects/:projectId/targets           // 添加单个资产
DELETE /api/v1/projects/:projectId/targets/:id       // 删除资产
DELETE /api/v1/projects/:projectId/targets/batch     // 批量删除

// 项目扫描
POST   /api/v1/projects/:projectId/scan              // 扫描整个项目
POST   /api/v1/projects/:projectId/targets/scan      // 扫描选中的资产

// 项目漏洞
GET    /api/v1/projects/:projectId/vulnerabilities   // 获取项目内发现的漏洞
```

### 导入 API 示例

```typescript
// POST /api/v1/projects/:projectId/targets/import
// Content-Type: multipart/form-data

// Request:
// - file: 上传的文件 (TXT/CSV)

// Response:
{
  "jobId": "import_abc123",
  "projectId": "project_xyz",
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
-- 项目表
CREATE TABLE projects (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  description TEXT,

  -- 统计摘要（定期更新）
  target_count INT UNSIGNED DEFAULT 0,
  wp_count INT UNSIGNED DEFAULT 0,
  vuln_count INT UNSIGNED DEFAULT 0,
  scanned_count INT UNSIGNED DEFAULT 0,

  last_scan_at DATETIME NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

  KEY idx_projects_created (created_at DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 目标资产表（优化大规模数据）
CREATE TABLE targets (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  project_id BIGINT UNSIGNED NOT NULL,   -- 必须属于某个项目
  url VARCHAR(2048) NOT NULL,
  domain VARCHAR(255) NOT NULL,
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

  -- 唯一约束：同一项目内 URL 不重复
  UNIQUE KEY uk_targets_project_url (project_id, url(500)),

  -- 索引优化
  KEY idx_targets_project (project_id),
  KEY idx_targets_domain (domain),
  KEY idx_targets_status (status),
  KEY idx_targets_created (created_at DESC),
  KEY idx_targets_is_wp (is_wordpress),
  KEY idx_targets_vuln (vuln_count DESC),

  FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 导入任务表
CREATE TABLE import_jobs (
  id VARCHAR(50) PRIMARY KEY,
  project_id BIGINT UNSIGNED NOT NULL,   -- 导入到的项目
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

  error TEXT,

  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  completed_at DATETIME NULL,

  KEY idx_import_project (project_id),
  FOREIGN KEY (project_id) REFERENCES projects(id) ON DELETE CASCADE
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
  projectId: string,
  batchSize = 10000
): AsyncGenerator<ITarget[]> {
  let cursor: string | null = null;

  while (true) {
    const result = await db.query(`
      SELECT * FROM targets
      WHERE project_id = $1
        AND ($2::uuid IS NULL OR id > $2)
      ORDER BY id
      LIMIT $3
    `, [projectId, cursor, batchSize]);

    if (result.rows.length === 0) break;

    yield result.rows;
    cursor = result.rows[result.rows.length - 1].id;
  }
}
```

### CLI 命令

```bash
# 项目管理
wpsan project create "项目名称"              # 创建项目
wpsan project list                          # 列出所有项目
wpsan project delete <project_id>           # 删除项目

# 导入资产到项目
wpsan import targets.txt --project=<id>     # 导入 TXT 到指定项目
wpsan import targets.csv --project=<id>     # 导入 CSV 到指定项目

# 资产管理（项目内）
wpsan targets list --project=<id>           # 列出项目内资产
wpsan targets list --project=<id> --wp-only # 只显示 WordPress 站点
wpsan targets count --project=<id>          # 项目内资产总数
wpsan targets export --project=<id> --format=csv  # 导出项目资产

# 批量扫描
wpsan scan --project=<id>                   # 扫描整个项目
wpsan scan --project=<id> --limit=10000     # 限制扫描数量
wpsan scan --project=<id> --status=pending  # 只扫描待扫描的
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

### 6. POC 可扩展架构（PHP 类）

```php
<?php
// src/Poc/BasePoc.php - POC 基类

namespace WPSan\Poc;

abstract class BasePoc
{
    // POC 元信息（子类必须定义）
    public const ID = '';                    // 唯一标识
    public const NAME = '';                  // POC 名称
    public const DESCRIPTION = '';           // 描述
    public const CVE = '';                   // CVE 编号（可选）
    public const SEVERITY = 'medium';        // critical, high, medium, low, info

    // 适用条件
    public const COMPONENT_TYPE = 'plugin';  // core, plugin, theme
    public const COMPONENT_SLUG = '';        // 插件/主题 slug
    public const AFFECTED_VERSIONS = '';     // 受影响版本，如 "< 3.5.0"

    protected HttpClient $http;

    public function __construct(HttpClient $http)
    {
        $this->http = $http;
    }

    /**
     * 验证漏洞是否存在（子类必须实现）
     *
     * @param string $target 目标 URL
     * @param array $context 上下文信息（插件版本等）
     * @return PocResult
     */
    abstract public function verify(string $target, array $context = []): PocResult;

    /**
     * 获取 POC 信息
     */
    public static function getInfo(): array
    {
        return [
            'id' => static::ID,
            'name' => static::NAME,
            'description' => static::DESCRIPTION,
            'cve' => static::CVE,
            'severity' => static::SEVERITY,
            'component_type' => static::COMPONENT_TYPE,
            'component_slug' => static::COMPONENT_SLUG,
            'affected_versions' => static::AFFECTED_VERSIONS,
        ];
    }
}

// src/Poc/PocResult.php - POC 执行结果

class PocResult
{
    public bool $vulnerable;
    public ?string $evidence;
    public array $details;

    public function __construct(bool $vulnerable, ?string $evidence = null, array $details = [])
    {
        $this->vulnerable = $vulnerable;
        $this->evidence = $evidence;
        $this->details = $details;
    }

    public function toArray(): array
    {
        return [
            'vulnerable' => $this->vulnerable,
            'evidence' => $this->evidence,
            'details' => $this->details,
        ];
    }
}
```

**POC 注册表：**

```php
<?php
// src/Poc/PocRegistry.php

namespace WPSan\Poc;

class PocRegistry
{
    private array $pocs = [];

    /**
     * 从目录加载所有 POC
     */
    public function loadFromDirectory(string $dir): void
    {
        $files = glob($dir . '/**/*.php', GLOB_BRACE);

        foreach ($files as $file) {
            require_once $file;

            // 获取文件中定义的类
            $className = $this->getClassNameFromFile($file);
            if ($className && is_subclass_of($className, BasePoc::class)) {
                $this->register($className);
            }
        }
    }

    /**
     * 注册 POC
     */
    public function register(string $pocClass): void
    {
        $id = $pocClass::ID;
        $this->pocs[$id] = $pocClass;
    }

    /**
     * 根据插件/主题查找关联的 POC
     */
    public function findByComponent(string $type, string $slug, ?string $version = null): array
    {
        $matched = [];

        foreach ($this->pocs as $pocClass) {
            if ($pocClass::COMPONENT_TYPE !== $type) continue;
            if ($pocClass::COMPONENT_SLUG !== $slug) continue;

            // 版本匹配
            if ($version && $pocClass::AFFECTED_VERSIONS) {
                if (!$this->versionMatches($version, $pocClass::AFFECTED_VERSIONS)) {
                    continue;
                }
            }

            $matched[] = $pocClass::getInfo();
        }

        return $matched;
    }

    /**
     * 获取 POC 实例
     */
    public function get(string $pocId): ?BasePoc
    {
        if (!isset($this->pocs[$pocId])) {
            return null;
        }

        $http = new HttpClient();
        return new $this->pocs[$pocId]($http);
    }

    /**
     * 版本范围匹配
     */
    private function versionMatches(string $version, string $range): bool
    {
        // 支持格式: "< 3.5.0", "<= 2.0", "> 1.0", "1.0 - 2.0"
        if (preg_match('/^<\s*(.+)$/', $range, $m)) {
            return version_compare($version, trim($m[1]), '<');
        }
        if (preg_match('/^<=\s*(.+)$/', $range, $m)) {
            return version_compare($version, trim($m[1]), '<=');
        }
        if (preg_match('/^>\s*(.+)$/', $range, $m)) {
            return version_compare($version, trim($m[1]), '>');
        }
        if (preg_match('/^(.+)\s*-\s*(.+)$/', $range, $m)) {
            return version_compare($version, trim($m[1]), '>=')
                && version_compare($version, trim($m[2]), '<=');
        }
        return $version === $range;
    }
}
```

**POC 示例：**

```php
<?php
// pocs/plugins/elementor/ElementorRcePoc.php

namespace WPSan\Poc\Plugins\Elementor;

use WPSan\Poc\BasePoc;
use WPSan\Poc\PocResult;

class ElementorRcePoc extends BasePoc
{
    public const ID = 'elementor-rce-2024-xxxx';
    public const NAME = 'Elementor Remote Code Execution';
    public const DESCRIPTION = 'Elementor plugin allows unauthenticated RCE via template import';
    public const CVE = 'CVE-2024-XXXX';
    public const SEVERITY = 'critical';

    public const COMPONENT_TYPE = 'plugin';
    public const COMPONENT_SLUG = 'elementor';
    public const AFFECTED_VERSIONS = '< 3.5.0';

    public function verify(string $target, array $context = []): PocResult
    {
        // 构造验证请求（仅验证，不利用）
        $response = $this->http->post(
            $target . '/wp-admin/admin-ajax.php',
            [
                'action' => 'elementor_ajax',
                'actions' => json_encode([
                    [
                        'action' => 'test_vulnerable_action',
                    ]
                ]),
            ]
        );

        // 判断漏洞是否存在
        $vulnerable = strpos($response['body'], 'specific_vulnerable_indicator') !== false;

        return new PocResult(
            $vulnerable,
            $vulnerable ? substr($response['body'], 0, 500) : null,
            ['status_code' => $response['status']]
        );
    }
}
```

**POC 命名规范：**

```
pocs/
├── wordpress/                    # WP 核心漏洞
│   └── WpCoreXxePoc.php
├── plugins/                      # 插件漏洞
│   ├── elementor/
│   │   ├── ElementorRcePoc.php
│   │   └── ElementorSsrfPoc.php
│   ├── woocommerce/
│   │   └── WooSqlInjectionPoc.php
│   └── contact-form-7/
│       └── Cf7FileUploadPoc.php
└── themes/                       # 主题漏洞
    └── flavor/
        └── FlavorLfiPoc.php
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

# 4. 导入数据库结构
mysql -u root -p wpsan < sql/schema.sql

# 5. 设置目录权限
chmod -R 755 storage/
chown -R www-data:www-data storage/

# 6. 更新 Cloudflare IP 段
php bin/update-cf-ips.php

# 7. 启动队列 Worker
php bin/worker.php &

# 8. 配置 Nginx
sudo cp nginx.conf /etc/nginx/sites-available/wpsan
sudo ln -s /etc/nginx/sites-available/wpsan /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 环境变量配置

```bash
# .env

APP_NAME=WPSan
APP_DEBUG=false
APP_URL=https://your-domain.com

# 数据库
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=wpsan
DB_USERNAME=wpsan
DB_PASSWORD=your-db-password

# Redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=
REDIS_PORT=6379

# 扫描配置
SCAN_CONCURRENCY=10
SCAN_SKIP_SCANNED=true
```

### Supervisor 配置（队列处理器）

```ini
; /etc/supervisor/conf.d/wpsan-queue.conf

[program:wpsan-queue]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/wpsan/bin/worker.php
directory=/var/www/wpsan
autostart=true
autorestart=true
user=www-data
numprocs=4
redirect_stderr=true
stdout_logfile=/var/log/wpsan-queue.log
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

    location /api/ {
        try_files $uri $uri/ /api.php?$query_string;
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
# 队列 Worker
php bin/worker.php                     # 启动队列处理器
php bin/worker.php --concurrency=20    # 指定并发数

# 资产导入
php bin/import.php targets.txt                    # 导入 TXT
php bin/import.php targets.csv --group=group1    # 导入到指定分组

# 扫描
php bin/scan.php --all                            # 扫描所有资产
php bin/scan.php --group=group1                   # 扫描指定分组
php bin/scan.php --limit=10000                    # 限制扫描数量

# POC 执行
php bin/poc.php --poc=elementor-rce-2024 --plugin=elementor

# Cloudflare IP 更新
php bin/update-cf-ips.php              # 从 cloudflare.com 更新 IP 段

# 数据库
mysql -u root -p wpsan < sql/schema.sql           # 初始化数据库
mysql -u root -p wpsan < sql/seed.sql             # 填充测试数据

# 探针 (在探针服务器上运行)
php probe/probe.php                    # 启动探针

# Supervisor 管理
sudo supervisorctl status              # 查看状态
sudo supervisorctl restart wpsan-queue # 重启队列
sudo supervisorctl restart wpsan-probe # 重启探针
```

---

## 前端界面设计

### 页面结构

```
views/
├── layout.php                   # 公共布局
├── dashboard.php                # 仪表盘首页（项目概览）
├── projects/
│   ├── index.php                # 项目列表
│   └── detail.php               # 项目详情（资产、漏洞、扫描记录）
├── scans/
│   └── detail.php               # 单个资产扫描详情
├── vulns/
│   ├── index.php                # 漏洞库列表（全局）
│   ├── create.php               # 添加漏洞
│   └── edit.php                 # 编辑漏洞
├── pocs/
│   ├── index.php                # POC 列表（全局）
│   ├── run.php                  # POC 批量执行
│   └── results.php              # POC 执行结果
├── probes/
│   └── index.php                # 探针状态
└── logs/
    └── index.php                # 操作日志
```

### 1. 仪表盘 (Dashboard)

显示系统概览、项目概览和关键统计数据：

```php
<?php
// 仪表盘数据结构
$dashboard = [
    // 项目统计
    'projects' => [
        'total' => 8,                // 项目总数
        'total_targets' => 500000,   // 总资产数
    ],

    // 资产统计
    'targets' => [
        'wordpress' => 320000,       // WordPress 站点数
        'non_wordpress' => 50000,    // 非 WP 站点
        'pending' => 130000,         // 待扫描
    ],

    // 漏洞统计
    'vulnerabilities' => [
        'total_found' => 12500,      // 发现的漏洞总数
        'critical' => 500,           // 严重
        'high' => 2000,              // 高危
        'medium' => 5000,            // 中危
        'low' => 5000,               // 低危
    ],

    // 探针状态
    'probes' => [
        'online' => 5,               // 在线探针
        'offline' => 1,              // 离线探针
    ],

    // 项目概览（前3个项目）
    'recent_projects' => [
        ['name' => '电商客户 A', 'targets' => 125430, 'vulns' => 2345, 'progress' => 78],
        ['name' => '政府网站检测', 'targets' => 50000, 'vulns' => 892, 'progress' => 100],
        ['name' => '企业官网批量', 'targets' => 320000, 'vulns' => 15678, 'progress' => 45],
    ],

    // 最近发现的漏洞（实时滚动）
    'recent_vulns' => [
        ['target' => 'https://example1.com', 'vuln' => 'Elementor RCE', 'severity' => 'critical', 'time' => '2分钟前'],
        ['target' => 'https://example2.com', 'vuln' => 'WooCommerce SQLi', 'severity' => 'high', 'time' => '5分钟前'],
        // ...
    ],
];
?>
```

### 2. 项目列表

显示所有项目卡片，每个项目显示：
- 项目名称和描述
- 资产总数、WordPress 站点数
- 发现的漏洞数
- 扫描进度
- 最后扫描时间

### 3. 项目详情

项目详情页包含三个 Tab：
- **资产列表**：显示项目内所有资产，支持筛选、批量操作
- **漏洞列表**：显示项目内发现的漏洞，可执行 POC 验证
- **扫描记录**：显示项目的扫描历史

### 4. 资产列表（项目内）

| 功能 | 说明 |
|------|------|
| 列表展示 | 分页显示资产，支持搜索/筛选 |
| 筛选条件 | 按分组、状态、是否WP、有无漏洞筛选 |
| 批量操作 | 批量扫描、批量删除 |
| 导入入口 | 上传 TXT/CSV 文件导入 |
| 导出 | 导出为 CSV/JSON |

### 3. 扫描详情

展示单个目标的扫描结果：
- 基本信息：URL、域名、是否 WP、WP 版本、CF 状态
- 插件列表：slug、版本、关联漏洞数
- 主题列表：slug、版本
- 漏洞列表：漏洞名称、严重性、关联 POC（可点击执行）

### 4. POC 执行界面

```
┌─────────────────────────────────────────────────────────────┐
│  POC 批量执行                                                │
├─────────────────────────────────────────────────────────────┤
│  选择 POC:  [Elementor RCE CVE-2024-XXXX    ▼]              │
│  目标范围:  ○ 所有匹配资产 (1,523 个)                        │
│             ○ 指定分组: [选择分组 ▼]                         │
│             ○ 手动选择                                       │
│                                                             │
│  预计目标: 1,523 个资产包含 Elementor < 3.5.0               │
│                                                             │
│  [开始执行]                                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 实时通知机制

使用 WebSocket 实现实时推送：

### WebSocket 服务器

```php
<?php
// src/WebSocket/Server.php

namespace WPSan\WebSocket;

use Workerman\Worker;

class WsServer
{
    private Worker $ws;
    private array $clients = [];

    public function __construct(string $host = '0.0.0.0', int $port = 8080)
    {
        $this->ws = new Worker("websocket://{$host}:{$port}");

        $this->ws->onConnect = function ($connection) {
            $this->clients[$connection->id] = $connection;
        };

        $this->ws->onClose = function ($connection) {
            unset($this->clients[$connection->id]);
        };

        $this->ws->onMessage = function ($connection, $data) {
            $msg = json_decode($data, true);
            if ($msg['action'] === 'subscribe') {
                $connection->channels = $msg['channels'] ?? [];
            }
        };
    }

    /**
     * 广播消息到指定频道
     */
    public function broadcast(string $channel, array $data): void
    {
        $message = json_encode([
            'channel' => $channel,
            'data' => $data,
            'time' => date('Y-m-d H:i:s'),
        ]);

        foreach ($this->clients as $client) {
            if (in_array($channel, $client->channels ?? [])) {
                $client->send($message);
            }
        }
    }

    public function run(): void
    {
        Worker::runAll();
    }
}
```

### 通知频道

| 频道 | 用途 | 推送内容 |
|------|------|----------|
| `scan:progress` | 扫描进度 | 当前扫描阶段、已完成数、队列数 |
| `scan:complete` | 扫描完成 | 单个目标扫描完成 |
| `vuln:found` | 发现漏洞 | 目标URL、漏洞信息、严重性 |
| `poc:result` | POC 结果 | POC 执行结果、是否存在漏洞 |
| `import:progress` | 导入进度 | 已处理、有效、重复、无效数量 |

### 前端订阅

```javascript
// public/assets/js/websocket.js

const ws = new WebSocket('ws://localhost:8080');

ws.onopen = () => {
    // 订阅感兴趣的频道
    ws.send(JSON.stringify({
        action: 'subscribe',
        channels: ['scan:progress', 'vuln:found', 'poc:result']
    }));
};

ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);

    switch (msg.channel) {
        case 'vuln:found':
            // 显示漏洞发现通知
            showNotification('发现漏洞', `${msg.data.target} - ${msg.data.vuln_name}`, 'danger');
            // 更新仪表盘统计
            updateDashboardStats();
            break;

        case 'scan:progress':
            // 更新扫描进度条
            updateScanProgress(msg.data);
            break;

        case 'poc:result':
            // 更新 POC 执行结果表格
            appendPocResult(msg.data);
            break;
    }
};
```

---

## 数据导出功能

### 导出格式

支持 **CSV** 和 **JSON** 两种格式。

### 导出内容

| 导出类型 | 包含字段 |
|----------|----------|
| 资产列表 | URL, 域名, 分组, 是否WP, WP版本, 漏洞数, 最后扫描时间 |
| 扫描结果 | URL, 插件列表, 主题列表, 漏洞列表 |
| 漏洞报告 | URL, 漏洞名称, CVE, 严重性, 插件/主题, 版本, 发现时间 |
| POC 结果 | URL, POC名称, 是否存在, 证据, 执行时间 |

### 导出 API

```php
<?php
// src/Api/ExportApi.php

namespace WPSan\Api;

class ExportApi
{
    /**
     * 导出资产列表
     * GET /api/export/targets?format=csv&group_id=1&is_wordpress=1
     */
    public function exportTargets(array $params): void
    {
        $format = $params['format'] ?? 'csv';
        $filters = [
            'group_id' => $params['group_id'] ?? null,
            'is_wordpress' => $params['is_wordpress'] ?? null,
            'has_vuln' => $params['has_vuln'] ?? null,
        ];

        $targets = $this->getFilteredTargets($filters);

        if ($format === 'csv') {
            $this->outputCsv($targets, 'targets_export.csv');
        } else {
            $this->outputJson($targets, 'targets_export.json');
        }
    }

    /**
     * 导出漏洞报告
     * GET /api/export/vulnerabilities?format=csv&severity=critical,high
     */
    public function exportVulnerabilities(array $params): void
    {
        $format = $params['format'] ?? 'csv';
        $severities = isset($params['severity']) ? explode(',', $params['severity']) : null;

        $vulns = $this->getVulnerabilityReport($severities);

        $filename = 'vuln_report_' . date('Ymd_His');
        if ($format === 'csv') {
            $this->outputCsv($vulns, $filename . '.csv');
        } else {
            $this->outputJson($vulns, $filename . '.json');
        }
    }

    private function outputCsv(array $data, string $filename): void
    {
        header('Content-Type: text/csv; charset=utf-8');
        header("Content-Disposition: attachment; filename=\"{$filename}\"");

        $output = fopen('php://output', 'w');

        // BOM for Excel UTF-8 support
        fwrite($output, "\xEF\xBB\xBF");

        // Header row
        if (!empty($data)) {
            fputcsv($output, array_keys($data[0]));
        }

        // Data rows
        foreach ($data as $row) {
            fputcsv($output, $row);
        }

        fclose($output);
        exit;
    }

    private function outputJson(array $data, string $filename): void
    {
        header('Content-Type: application/json; charset=utf-8');
        header("Content-Disposition: attachment; filename=\"{$filename}\"");

        echo json_encode($data, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE);
        exit;
    }
}
```

---

## 操作日志

记录所有重要操作，便于审计追踪。

### 日志表结构

```sql
CREATE TABLE operation_logs (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  action VARCHAR(50) NOT NULL,          -- 操作类型
  target_type VARCHAR(50),              -- 操作对象类型
  target_id VARCHAR(100),               -- 操作对象 ID
  detail JSON,                          -- 详细信息
  ip VARCHAR(45),                       -- 操作者 IP
  user_agent VARCHAR(500),              -- 浏览器信息
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,

  KEY idx_logs_action (action),
  KEY idx_logs_created (created_at DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 记录的操作类型

| 操作类型 | 说明 |
|----------|------|
| `target.import` | 导入资产 |
| `target.delete` | 删除资产 |
| `target.batch_delete` | 批量删除资产 |
| `scan.start` | 开始扫描 |
| `scan.batch_start` | 批量扫描 |
| `poc.execute` | 执行 POC |
| `poc.batch_execute` | 批量执行 POC |
| `vuln.create` | 添加漏洞 |
| `vuln.update` | 更新漏洞 |
| `vuln.delete` | 删除漏洞 |
| `export.targets` | 导出资产 |
| `export.vulns` | 导出漏洞报告 |

### 日志服务

```php
<?php
// src/Core/Logger.php

namespace WPSan\Core;

class OperationLogger
{
    private \PDO $db;

    public function log(
        string $action,
        ?string $targetType = null,
        ?string $targetId = null,
        array $detail = []
    ): void {
        $stmt = $this->db->prepare('
            INSERT INTO operation_logs (action, target_type, target_id, detail, ip, user_agent, created_at)
            VALUES (?, ?, ?, ?, ?, ?, NOW())
        ');

        $stmt->execute([
            $action,
            $targetType,
            $targetId,
            json_encode($detail, JSON_UNESCAPED_UNICODE),
            $_SERVER['REMOTE_ADDR'] ?? 'cli',
            $_SERVER['HTTP_USER_AGENT'] ?? 'cli',
        ]);
    }

    /**
     * 查询日志
     */
    public function query(array $filters = [], int $page = 1, int $perPage = 50): array
    {
        $where = ['1=1'];
        $params = [];

        if (!empty($filters['action'])) {
            $where[] = 'action = ?';
            $params[] = $filters['action'];
        }

        if (!empty($filters['start_date'])) {
            $where[] = 'created_at >= ?';
            $params[] = $filters['start_date'];
        }

        if (!empty($filters['end_date'])) {
            $where[] = 'created_at <= ?';
            $params[] = $filters['end_date'];
        }

        $offset = ($page - 1) * $perPage;
        $sql = 'SELECT * FROM operation_logs WHERE ' . implode(' AND ', $where)
             . ' ORDER BY created_at DESC LIMIT ' . $perPage . ' OFFSET ' . $offset;

        $stmt = $this->db->prepare($sql);
        $stmt->execute($params);

        return $stmt->fetchAll(\PDO::FETCH_ASSOC);
    }
}
```

### 日志页面

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  操作日志                                                   [导出] [刷新]   │
├─────────────────────────────────────────────────────────────────────────────┤
│  筛选: [操作类型 ▼] [开始日期] [结束日期] [搜索]                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  时间                 │ 操作类型      │ 对象         │ 详情          │ IP    │
├─────────────────────────────────────────────────────────────────────────────┤
│  2024-01-15 10:30:00 │ poc.execute   │ POC: xxx     │ 执行1523个目标 │ 192.. │
│  2024-01-15 10:25:00 │ scan.start    │ Group: test  │ 开始扫描5000个 │ 192.. │
│  2024-01-15 10:20:00 │ target.import │ Job: abc123  │ 导入50000条    │ 192.. │
│  ...                                                                        │
└─────────────────────────────────────────────────────────────────────────────┘
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
