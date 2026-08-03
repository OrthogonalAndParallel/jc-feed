# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指导。

## 项目概述

FeedMe（禅游阅读）是一个 AI 驱动的 RSS 阅读器，使用 LLM API 生成多语言摘要。作为静态 React 应用构建，可零后端部署到 GitHub Pages 或 Docker。

## 技术栈

- **框架**: React 19 + TypeScript
- **构建工具**: Vite 6
- **样式**: Tailwind CSS + shadcn/ui 组件
- **包管理器**: pnpm 8.4.0+
- **Node.js**: 24 LTS（通过 `.nvmrc` 强制）

## 关键命令

```bash
# 开发
pnpm dev              # 启动开发服务器，访问 http://localhost:3000

# 构建与类型检查
pnpm typecheck        # 运行 TypeScript 类型检查
pnpm build            # 构建静态站点到 ./out

# RSS 数据管理
pnpm update-feeds     # 获取所有 RSS 源并生成 AI 摘要
pnpm full-build       # 更新源 + 构建一步完成
```

## 架构

### 配置系统

**YAML 驱动配置** (`src/config/feedme.config.yaml`):
- 定义 RSS 源、分类和摘要设置
- 构建时和运行时的唯一真实来源
- 支持源和分类的本地化名称（`zh`、`en`）
- 摘要配置：提示词模板、temperature、max tokens、降级文案

**配置加载**:
- 构建时：`vite.config.ts` 读取 YAML 并通过 `__FEEDME_CLIENT_CONFIG__` 注入客户端安全子集
- 运行时：`feedme-config-loader.ts` 解析并验证，使用严格的类型守卫
- 脚本：`scripts/update-feeds.ts` 加载完整配置（包括摘要设置）

### 数据流

1. **RSS 获取** (`scripts/update-feeds.ts`):
   - 使用 `rss-parser` 解析 RSS 源
   - 合并新条目与已有数据（保留摘要）
   - 通过 OpenAI 兼容 API 生成多语言摘要
   - 将每个源的 JSON 文件存储到 `public/data/{hash(url)}.json`

2. **客户端加载** (`src/lib/data-store.ts`):
   - 从相对路径 `./data/{hash}.json` 获取 JSON 文件
   - 使用 `window.location.pathname` 处理 GitHub Pages 子路径
   - 无缓存层（依赖浏览器 HTTP 缓存）

### 国际化

**i18n 结构** (`src/config/i18n-config.ts`):
- 支持的语言：`zh`（默认）、`en`
- 每个语言定义：label、htmlLang、dateLocale、summaryLanguage
- 摘要语言通过 `SUMMARY_LOCALES` 环境变量控制（如 `zh,en`）

**本地化模式**:
- 配置使用 `LocalizedName` 类型：`Record<string, string>`
- 运行时使用 `getLocalizedValue(values, locale)` 辅助函数
- 添加语言需要：在 `i18n-config.ts` 添加元数据 + 在 YAML 中补充翻译

### 组件架构

**核心组件**:
- `App.tsx`：根布局、源选择状态、搜索协调
- `rss-feed.tsx`：显示 feed 条目，支持摘要展开
- `source-switcher.tsx`：按分类分组的源选择下拉菜单
- `global-search.tsx`：跨源搜索，支持键盘导航

**状态管理**:
- URL 状态：`?source=` 参数用于可分享链接
- Local storage：跨会话持久化选中的源
- React 状态：搜索高亮和条目选择

## 环境变量

`pnpm update-feeds` 所需：

```bash
LLM_API_KEY=your_api_key
LLM_API_BASE=https://api.example.com/v1
LLM_NAME=model-name
SUMMARY_LOCALES=zh,en  # 可选，默认 zh,en
```

## 开发模式

### 添加新的 RSS 源

编辑 `src/config/feedme.config.yaml`：

```yaml
sources:
  - id: unique-id
    name:
      zh: 中文名称
      en: English Name
    url: https://example.com/feed.xml
    category: existing-category-id
```

### 路径别名

TypeScript 和 Vite 使用 `@/*` 别名表示 `./src/*`：
```typescript
import { Something } from '@/lib/types'
```

### Feed 数据结构

```typescript
interface FeedData {
  sourceUrl: string
  title: string
  description: string
  link: string
  items: FeedItem[]
  lastUpdated: string
}

interface FeedItem {
  title: string
  link?: string
  pubDate?: string
  content?: string
  summary?: string           // 默认语言的摘要
  summaries?: Record<string, string>  // 多语言摘要
}
```

### 构建输出

- 生产构建输出到 `./out`（而非 `dist`）
- 使用相对路径（`base: './'`）以兼容 GitHub Pages
- 静态 JSON 文件在 `out/data/`，由客户端加载

## 部署工作流

**GitHub Pages**（主要方式）：
- `.github/workflows/update-deploy.yml` 每小时运行
- 单工作流：获取 RSS → 生成摘要 → 构建 → 部署
- 推送产物到 `deploy` 分支供 Vercel/ESA 集成

**Docker**：
- 容器内 Cron 运行 `pnpm update-feeds && pnpm build`
- Entrypoint 处理 cron 调度和静态文件服务
- 配置：`src/config/crontab-docker`

## 重要说明

### Feed 更新安全性

`update-feeds.ts` 在以下情况下保留已有摘要：
- RSS 源返回为空（保留旧数据）
- 条目在该语言已有摘要
- 仅为新条目生成摘要

### 数据文件命名

文件名使用源 URL 的 SHA-256 哈希（前 16 位）：
```typescript
// src/lib/source-data-path.ts
export function getSourceDataFilename(sourceUrl: string): string {
  return createHash('sha256').update(sourceUrl).digest('hex').slice(0, 16) + '.json'
}
```

### 类型安全

- 严格的 TypeScript 配置，启用 `noUnusedLocals`、`noUnusedParameters`
- 配置加载器使用详尽验证，提供描述性错误信息
- 禁止 `any` 类型；运行时验证使用类型守卫