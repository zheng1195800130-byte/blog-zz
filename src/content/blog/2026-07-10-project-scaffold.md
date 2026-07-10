---
title: '项目脚手架搭建完成：DowMore 从零开始'
description: '完成 uni-app + FastAPI 项目脚手架搭建，打通视频下载核心链路，建立完整的前后端工程体系。'
pubDate: '2026-07-10'
tags: ['uni-app', 'fastapi', 'celery', 'vue3', 'project-init']
---

## 做了什么

DowMore 项目的 Phase 1 Sprint 1 正式完成。这个 Sprint 的核心目标是把项目骨架搭好 — 前端能跑、后端能启动、数据库能连、视频下载链路能走通。目前全部达成。

## 技术栈总览

| 层 | 技术 | 说明 |
|---|---|---|
| 前端 | uni-app (Vue 3 + Composition API) + Vite | 一套代码编译到微信小程序 |
| UI 组件 | uview-plus 3.x | uni-app 生态组件库 |
| 后端 | FastAPI (Python 3.11+) | 异步 Web 框架 |
| 异步任务 | Celery + Redis | 下载/转码均异步执行 |
| 数据库 | PostgreSQL 15 | 用户数据、任务记录 |
| 对象存储 | 腾讯云 COS | 文件临时中转 (≤ 1 小时) |

## 关键架构决策

### 文件只做临时中转

所有下载文件在 COS 保留不超过 1 小时，用户保存到相册后立即标记可删。这么设计有两个原因：

1. **版权合规** — 项目定位是工具，不是网盘
2. **存储成本** — COS 按量计费，长期存储不划算

### 默认最高画质

yt-dlp 调用参数默认 `bestvideo*+bestaudio/best`，输出 MP4。不做任何质量降级，用户想省流量可以手动选低质量。

### 前后端分离 + WebSocket 推送

长耗时任务走 Celery 异步队列，前端通过 WebSocket 实时获取进度。不会让用户盯着 loading 转圈。

## 第一版功能范围

当前只开放**视频下载**功能，音乐搜索、图片批量、去水印的页面已经建好（显示"即将上线"），但后端还未实现。

## 下一步

Phase 1 Sprint 2：数据库表设计 + Alembic 迁移 + 微信登录接口联调。
