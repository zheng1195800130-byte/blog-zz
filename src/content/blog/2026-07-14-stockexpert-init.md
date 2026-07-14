---
title: '股票大师初始版本：AI 金融助手微信小程序正式立项'
description: '完成 Taro + NestJS 全栈项目脚手架，集成豆包 AI Agent、7 个金融数据源、50+ shadcn/ui 组件。'
pubDate: '2026-07-14'
project: 'stockexpert'
tags: ['taro', 'nestjs', 'ai-agent', 'react', 'supabase', 'project-init']
---

## 做了什么

股票大师 (Stock Master) 的初始版本已完成脚手架搭建和核心链路贯通。这是一个 AI 驱动的金融助手微信小程序，定位为"随身携带的专业股票分析师"。

## 技术栈总览

| 层 | 技术 | 说明 |
|---|---|---|
| 跨端框架 | Taro 4.1 + React 18 | 一套代码编译到微信小程序 + H5 |
| UI | TailwindCSS + shadcn/ui | 50+ 适配 Taro 的组件 |
| 状态管理 | Zustand 5 | 轻量级状态管理 |
| 后端 | NestJS 10 + Express | TypeScript 全栈 |
| 数据库 | Supabase (PostgreSQL) + Drizzle ORM | 类型安全的数据库操作 |
| AI 引擎 | 豆包 Seed 2.0 Lite | 自研 Agent 循环 (think → tools → re-think) |
| 数据源 | 7 个源多级回退 | iTick / 必盈 / 同花顺 iFinD / 东方财富妙想 / Yahoo |

## 核心架构决策

### Agent 优先架构

系统的核心是一个自定义 AI Agent 循环：

```
用户提问 → Agent 思考 → 调用工具(8选1) → 拿到数据 → 再思考
→ 最多 5 轮循环 → 生成最终回答 → SSE 流式返回
```

8 个工具映射到 StockDataService 的 6 个查询方法，确保 AI 只基于真实数据回答，杜绝幻觉。

### 多级数据回退

7 个金融数据源按优先级排列，某个源挂了自动切下一个：

1. **iTick** — 全球 17 个市场实时行情（最高优先级）
2. **必盈** — A 股回退 + 技术指标 + 资金流向
3. **同花顺 iFinD** — 财报 / 研报 / ESG
4. **东方财富妙想 x3** — 机构大单 / 公告 / 智能选股
5. **Yahoo Finance** — 终极兜底

## 当前功能范围

| 功能 | 状态 |
|---|---|
| 财经资讯首页（分类/热度/骨架屏） | ✅ |
| AI 对话页（推荐话题/免责声明/SSE 流式） | ✅ |
| 个人中心（用户信息/免责签署/缓存清理） | ✅ |
| 敏感词过滤 | ✅ |
| AI 深度研报 | 🔜 P0 |
| 自选股监控 | 🔜 v1.1 |
| K 线 / 趋势图 | 🔜 待排期 |

## 下一步

P0：AI 深度研报功能 — 前端卡片式报告渲染，后端增加深度分析工具链。
