---
title: 'Sprint 3：UI 大升级 + 全平台解析 + 抖音零 Cookie 方案'
description: '33 个文件 3000 行变更：全局靛蓝主题、4 个平台专用解析引擎（抖音/快手/B站/微博）、任务管理中心、vConsole 调试面板。'
pubDate: '2026-07-14'
tags: ['ui-design', 'douyin', 'reverse-engineering', 'wechat-miniapp', 'svg', 'vconsole']
---

## 概述

Sprint 3 是 Phase 1 里最大的一次迭代，分两个提交完成：

| 提交 | 内容 |
|---|---|
| `9c9001e` Sprint 3 | UI 大改 + 全平台解析 + 任务管理 + 抖音零 Cookie |
| `c6769a8` Fix | 快手/微博预览修复 + vConsole 调试面板 + 3 个新解析引擎 |

---

## 🎨 UI 全面翻新

之前的界面是「能跑就行」阶段，这次做了系统性重构：

### 设计系统建立

- **主色调**：Indigo `#4F46E5`，替代了默认的灰色系
- **卡片式布局**：白色卡片 + 浅灰背景，微信小程序风格统一
- **SVG 矢量图标**：建了 `icons.ts` 图标库，10+ 个场景图标全部用 SVG base64 内嵌，不再用 emoji 凑合

```typescript
// icons.ts — 所有图标统一 24×24，Indigo 描边风格
export const ICON_VIDEO = svgToDataUri(`<svg ...>...</svg>`)
export const ICON_MUSIC = svgToDataUri(`<svg ...>...</svg>`)
export const ICON_DOWNLOAD = svgToDataUri(`<svg ...>...</svg>`)
// ... 10+ 个业务图标
```

### 视频页重构

视频下载页改成了**三页卡片布局**：视频预览 > 封面 > 标题，输入框和按钮比例重新调整，视觉层次清晰了很多。

### 任务页重写

任务列表从依赖 uview-plus 组件的简单列表，重写成了纯自定义组件：
- 自定义 Tab 切换（进行中 / 已完成）
- 进度条实时更新
- 状态标签（排队中 / 下载中 / 上传中 / 已完成 / 失败）
- 时间友好展示（刚刚 / 5 分钟前 / 昨天）
- 去除了对 uview-plus 的硬依赖

---

## 📹 多平台解析引擎

之前只有 yt-dlp 一个通用引擎，Sprint 3 新增了 4 个**平台专用解析器**：

```
server/app/services/
├── ytdlp_service.py      # 通用引擎（1800+ 站点）
├── douyin_svc.py          # 抖音专用 ★ 新增
├── kuaishou_svc.py        # 快手专用 ★ 新增
├── bilibili_svc.py        # B站专用 ★ 新增
└── weibo_svc.py           # 微博专用 ★ 新增
```

### 抖音零 Cookie 方案 🏆

这是本次最大的技术亮点。抖音的反爬很严格，传统方案需要 Cookie 或手机抓包。

我们的方案是直接解析抖音分享页的 HTML 源码，从中提取 `ROUTER_DATA` JSON：

```python
def parse_douyin_video(url: str) -> dict:
    """无需 Cookie、无需登录，纯服务器端方案"""
    # 1. 请求分享页 HTML
    resp = client.get(url)
    # 2. 正则提取 ROUTER_DATA 中的 JSON
    match = re.search(r'ROUTER_DATA\s*=\s*(\{.+?\});', resp.text)
    data = json.loads(match.group(1))
    # 3. 从 JSON 中解析出无水印视频地址、标题、封面
    video_info = data['loaderData']['video_(id)']['videoInfoRes']
    return {
        "title": ...,
        "video_url": ...,  # 无水印直链
        "platform": "Douyin",
    }
```

实测解析速度：**5.6 秒**，包含 HTTP 请求 + HTML 解析，不需要任何鉴权。

### 快手解析

类似思路，从分享页提取真实 `.mp4` 直链，过滤掉 HLS `.ts`/`.m3u8` 分片 URL（那些无法直接播放）。

### B站 + 微博

B站和微博走的是专用解析 + yt-dlp 回退的双通道策略，优先用专用引擎（更快），失败自动 fallback 到 yt-dlp。

---

## 📋 任务管理中心

任务管理从一个简陋列表进化成了完整的管理系统：

| 功能 | 说明 |
|---|---|
| ⏸️ 取消 | 进行中的任务可取消 |
| 🗑️ 删除 | 清除已完成/失败的任务 |
| 🔄 重试 | 失败任务一键重试 |
| 🔍 重新解析 | 已完成任务换格式重新下载 |
| 📋 复制链接 | 复制下载地址到剪贴板 |
| 🔄 自动轮询 | 进行中任务每 3 秒刷新进度 |

---

## 🛠️ 开发体验优化

### vConsole 调试面板

在 H5 模式下自动启用 vConsole，手机浏览器里也能看 console.log/网络请求/DOM 结构，真机调试再也不用盲猜。

```typescript
// main.ts
if (process.env.NODE_ENV === 'development') {
  import('vconsole').then(({ default: VConsole }) => {
    new VConsole()
  })
}
```

### 缩略图 base64 内嵌

之前的缩略图直接引用第三方 CDN URL，部分平台有防盗链导致图片加载失败。现在改为 base64 编码后通过 API 返回，前端直接渲染 data URI，完全绕过防盗链。

### 短链自动展开

支持抖音/快手的短链接，自动跟随重定向到完整 URL 后再解析。

---

## 🐛 修的坑

- **format_id 字段太短**：32→255 字符，yt-dlp 某些站点的 format_id 超过 200 字符
- **WebSocket 路由被 REST 拦截**：调整路由注册顺序，WS 路由放在最前面
- **POST/DELETE 路由顺序**：`/tasks/{id}` 的 GET 会拦截 POST/DELETE，调整注册顺序后 405 消失
- **快手 HLS 分片误判**：过滤 `.ts`/`.m3u8` URL，只保留 `.mp4` 直链

---

## 验证结果

| 测试项 | 结果 |
|---|---|
| B站视频解析+下载全链路 | ✅ 1.1s 解析 |
| 抖音零 Cookie 解析 | ✅ 5.6s |
| WebSocket 心跳 | ✅ 连通 |
| 前端 H5 + 小程序双端编译 | ✅ 通过 |
| 快手/微博预览播放 | ✅ 修复后正常 |

---

## 下一步

Sprint 4：前端错误状态/空状态/网络检查等边界体验打磨 + 真机测试 + 微信审核准备。
