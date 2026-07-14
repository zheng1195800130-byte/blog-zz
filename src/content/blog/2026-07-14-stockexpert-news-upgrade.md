---
title: '股票大师资讯模块重构：23 个数据源 + 频道管理 + 全文预览'
description: '从硬编码种子数据升级为实时聚合 23 个财经资讯源，新增拖拽频道管理、文章全文预览弹窗、7 天时效过滤、东方财富数据源接入。'
pubDate: '2026-07-14'
project: 'stockexpert'
tags: ['news', 'react', 'taro', 'drag-and-drop', 'financial-data', 'nestjs', 'content-extraction']
---

## 概述

资讯模块是股票大师首页的核心功能。这个 Sprint 做了一次深度重构——从最初的硬编码种子数据，升级为从 23 个真实财经数据源实时聚合新闻，并补齐了频道管理、全文预览等用户体验关键功能。

---

## 📰 从硬编码到实时数据源

### 改造前：种子数据

初始版本在 Supabase 中预置了 16 条写死的新闻，服务启动时 seed 进去。缺点是数据不更新、不实时。

### 改造后：StockDataService 实时聚合

删除了全部种子数据逻辑（从 320 行缩减到 136 行），改为从 `StockDataService` 实时拉取：

```typescript
// 改造后的 NewsService
@Injectable()
export class NewsService {
  constructor(private readonly stockDataService: StockDataService) {}

  async getCachedNews(): Promise<NewsApiItem[]> {
    // 5 分钟缓存，避免频繁请求外部 API
    if (this.cache && Date.now() - this.cache.timestamp < CACHE_TTL_MS) {
      return this.cache.data
    }
    const data = await this.stockDataService.fetchAllNews()
    this.cache = { data, timestamp: Date.now() }
    return data
  }
}
```

### 新增东方财富数据源

接入东方财富新闻 API，加上原有 iTick / 必盈 / 同花顺 / Yahoo 等数据源，资讯来源扩展到 **23 个**：

```
东方财富 / 新浪财经 / 证券时报 / 证券日报 / 上海证券报 /
中国基金报 / 华尔街见闻 / 36氪 / 新华社财经 / 中国人民银行 /
证监会 / 国家统计局 / 中汽协 / ... + 7 个金融数据源
```

---

## 🗂️ 频道管理：长按拖拽排序

这是前端体验最大的升级——新增了 `CategoryManager` 组件（394 行），支持：

### 功能特性

| 功能 | 交互方式 |
|---|---|
| 拖拽排序 | 长按 500ms → 震动反馈 → 拖动到目标位置松手 |
| 隐藏频道 | 拖到页面下半区 → 自动移入隐藏区 |
| 恢复频道 | 点击隐藏区标签 → 恢复到显示区 |
| 恢复默认 | 一键重置所有频道设置 |
| 本地持久化 | `Taro.setStorageSync` 保存用户偏好 |

### 长按拖拽实现

```typescript
const handleTouchStart = (index: number, zone: 'displayed' | 'hidden') => (e) => {
  longPressTimer.current = setTimeout(() => {
    setDragIndex(index)
    setDragZone(zone)
    setIsDragging(true)
    Taro.vibrateShort()  // 震动反馈
  }, 500)
}

const handleTouchEnd = (index: number, zone) => (e) => {
  // 根据手指 Y 坐标判断目标区域
  const targetZone = touch.clientY < window.innerHeight * 0.55
    ? 'displayed' : 'hidden'

  if (dragZone === 'displayed' && targetZone === 'hidden') {
    moveToHidden(source)           // 拖到下半屏 = 隐藏
  } else if (dragZone !== targetZone) {
    moveToDisplayed(source)        // 拖到上半屏 = 恢复
  } else if (dragIndex !== index) {
    // 同区域内重排序
  }
}
```

### 安全区适配

底部保存按钮使用 `env(safe-area-inset-bottom)` 适配刘海屏：

```css
padding-bottom: calc(12px + env(safe-area-inset-bottom, 0px));
```

---

## 📄 全文预览弹窗

之前点击新闻直接跳转外部链接，现在改为**应用内预览弹窗**：

### 预览弹窗功能

- **标题 + 来源 + 发布时间**：顶部信息栏
- **全文/摘要展示**：优先展示服务端抓取的全文，降级显示摘要
- **段落拆分渲染**：段间距 + 首行缩进，阅读体验更好
- **查看原文**：按钮跳转外部链接
- **复制链接**：一键复制到剪贴板

### 服务端全文抓取

新增 `/api/news/content` 端点，后端抓取目标页面内容：

```typescript
@Get('content')
async getContent(@Query('url') url: string, @Query('source') source: string) {
  // 新浪手机版页面内容提取
  const content = await this.newsService.getArticleContent(url, source)
  return { code: 0, msg: 'success', data: { content } }
}
```

支持解析新浪财经手机版 HTML，提取正文内容。

---

## 🏷️ 来源色板系统

从固定分类（宏观/股票/基金...）改为按**来源**筛选，为 23 个来源自动分配颜色标签：

```typescript
const SOURCE_COLORS = [
  { bg: 'bg-[#E8F3FF]', text: 'text-[#165DFF]' },   // 蓝
  { bg: 'bg-[#FFECE8]', text: 'text-[#F53F3F]' },   // 红
  { bg: 'bg-[#E8FFEA]', text: 'text-[#00B42A]' },   // 绿
  // ... 10 种颜色循环使用
]

function getSourceColor(source: string) {
  if (!SOURCE_PALETTE[source]) {
    SOURCE_PALETTE[source] = Object.keys(SOURCE_PALETTE).length
  }
  return SOURCE_COLORS[SOURCE_PALETTE[source] % SOURCE_COLORS.length]
}
```

---

## ⏳ 7 天时效过滤

新增时效性过滤，自动排除 7 天前的旧闻，确保首页展示的都是最新财经资讯。

---

## 🐛 修复

- **API 请求 Invalid URL**：添加 `.env.local`，修复 `PROJECT_DOMAIN` 环境变量缺失导致请求 URL 拼接错误

---

## 📊 变更规模

| 维度 | 数据 |
|---|---|
| 变更文件 | 14 个 |
| 新增代码 | ~1300 行 |
| 删除代码 | ~480 行（种子数据） |
| 新增数据源 | 1 个（东方财富）+ 聚合 23 源 |
| 新增组件 | CategoryManager（394 行） |
| 新增 API | `/api/news/content` 全文抓取 |

---

## 下一步

P0 任务：AI 深度研报功能——前端卡片式报告渲染 + 后端深度分析工具链。
