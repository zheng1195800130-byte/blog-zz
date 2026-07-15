---
title: 'Sprint 5：图片批量保存模块 + 小红书独立解析引擎'
description: '19 个文件 2500 行变更：gallery-dl 90+ 图站封装、小红书 HTML 逆向解析（window.__INITIAL_STATE__）、图片网格选择器、Celery 异步打包下载。'
pubDate: '2026-07-15'
project: 'dowmore'
tags: ['image', 'gallery-dl', 'xiaohongshu', 'reverse-engineering', 'celery', 'wechat-miniapp']
---

## 概述

图片保存是 DowMore 继视频、音乐之后的第三个内容模块。本次 Sprint 分三个提交完成：

| 提交 | 内容 |
|---|---|
| `4c5bfa0` | 图片保存模块完整实现 + 小红书解析支持（核心） |
| `a341dbc` | 修复图片/音乐任务重新解析跳转错误 |
| `482785f` | 清理冗余文件和无用代码 |

核心成果：**粘贴链接 → 预览图库 → 选择图片 → 一键打包下载**，覆盖 90+ 图站。

---

## 🏗️ 架构概览

```
小程序前端                        FastAPI 后端
    │                                  │
    ├─ 粘贴图片链接                    ├─ /image/parse  ←─ gallery_service.py
    │  (支持分享文案自动提取URL)       │      ├─ gallery-dl (90+ 图站)
    │                                  │      └─ xhs_service.py (小红书独立)
    ├─ 图片网格 + 全选/单选           │
    │                                  ├─ /image/download  → Celery Worker
    └─ 下载进度 + 结果通知             │      ├─ gallery-dl 下载
                                       │      ├─ httpx 流式下载（小红书）
                                       │      └─ ZIP 打包 → COS 上传
```

---

## 📕 小红书独立解析引擎（447 行）

这是本次最大的技术难点。**gallery-dl 不支持小红书**，需要从零实现一个完整的解析器。

### 为什么难

小红书是强反爬站点，所有数据通过 JS 渲染，接口需要签名。传统方案需要 APP 抓包获取 Cookie。

### 我们的方案：纯 HTML 逆向

不做接口签名、不维护 Cookie 池、不需要手机抓包。直接在服务器端抓取页面 HTML，提取服务端渲染的 `window.__INITIAL_STATE__`：

```python
def parse_note(url: str) -> dict:
    # Step 1: 解析短链接 → 跟随重定向获取真实 URL 和 Cookie
    with httpx.Client(follow_redirects=True) as client:
        resp = client.get(url)        # xhslink.com/xxx → xiaohongshu.com/explore/xxx
        html = resp.text

    # Step 2: 如果没有数据，尝试 /discovery/item/ 和 /explore/ 路径
    if "noteDetailMap" not in html:
        resp = client.get(f"https://www.xiaohongshu.com/discovery/item/{note_id}")
        html = resp.text

    # Step 3: 正则提取 __INITIAL_STATE__ JSON
    init_data = _extract_initial_state(html, note_id)

    # Step 4: 遍历 noteDetailMap → imageList → infoList
    note = init_data["note"]
    for img in note["imageList"]:
        for info in img["infoList"]:
            # 从 CDN URL 后缀判断画质
            # !nd_dft = 默认画质, !nd_prv = 预览画质
            if "!nd_dft" in info["url"]:
                best_url = info["url"]   # 优先取默认画质
```

### 短链接解析

小红书分享链接多为 `xhslink.com` 短链，需要跟随 HTTP 重定向：

```python
def resolve_short_link(url: str) -> str:
    resp = httpx.get(url, follow_redirects=True)
    return str(resp.url)  # 含 xsec_token 等鉴权参数
```

### CDN 画质选择

小红书图片 URL 通过 `!nd_xxx` 后缀区分画质等级：

| 后缀 | 含义 | 优先级 |
|---|---|---|
| `!nd_dft` | 默认画质（原始） | 🥇 最高 |
| `!nd_prv` | 预览画质（压缩） | 🥈 次选 |

解析时自动选择最佳可用画质。

### 支持的链接格式

- `xhslink.com/xxx` — 短链接（自动解析）
- `xiaohongshu.com/discovery/item/{note_id}` — 标准链接
- `xiaohongshu.com/explore/{note_id}` — 备用路径
- `xiaohongshu.com/user/profile/{user_id}` — 作者主页（提示用户用单篇链接）

---

## 🖼️ gallery-dl 封装（458 行）

对于小红书之外的 90+ 图站（Instagram / Pixiv / Twitter / DeviantArt 等），使用 gallery-dl 命令行工具：

```python
def parse_gallery(url: str, range_start=None, range_end=None) -> dict:
    # 小红书委托给 xhs_service
    if is_xhs_url(url):
        return parse_note(url)

    # Step 1: --get-urls 获取图片直链列表
    url_proc = subprocess.run(
        ["gallery-dl", "--get-urls", "--no-download", url],
        capture_output=True, text=True, timeout=60
    )

    # Step 2: --dump-json 获取元数据（宽高/大小/作者/标题）
    meta_proc = subprocess.run(
        ["gallery-dl", "--dump-json", "--no-download", url],
        capture_output=True, text=True, timeout=60
    )

    # Step 3: 合并 URL + 元数据 → 统一 ImageInfo 列表
```

遵循项目约定（与 yt-dlp 调用方式一致）：`subprocess.run` 调用，设置超时，不引入 Python API。

### 平台自动检测

```python
_PLATFORM_DOMAINS = {
    "instagram": ["instagram.com"],
    "pixiv": ["pixiv.net", "fanbox.cc"],
    "twitter": ["twitter.com", "x.com"],
    "xiaohongshu": ["xiaohongshu.com", "xhslink.com"],
    "deviantart": ["deviantart.com"],
    "weibo": ["weibo.com", "weibo.cn"],
    "tumblr": ["tumblr.com"],
    "flickr": ["flickr.com"],
    "imgur": ["imgur.com"],
}
```

通过 URL 域名快速识别平台，前端据此显示不同颜色的平台标签。

---

## 🎨 前端体验

### 完整交互流程

```
粘贴链接 → 解析按钮 → 加载动画 → 图片网格
    ├─ 平台标签（渐变色） + 作者 + 数量/大小
    ├─ 全选/取消全选
    ├─ 点击单张切换选中（蓝色遮罩 + ✓）
    └─ 下载 N 张 按钮 → 创建任务 → 跳转任务页看进度
```

### 智能链接提取

用户可以直接粘贴整段分享文案，前端自动提取 URL：

```typescript
function extractUrl(input: string): string {
  // 1. 提取 http/https URL（支持粘贴整段分享文案）
  const urlMatch = input.match(/https?:\/\/[^\s，,。；;]+/i)
  if (urlMatch) return urlMatch[0]

  // 2. 缺协议自动补全
  const bareMatch = input.match(/^(xhslink\.com|instagram\.com|...)/i)
  if (bareMatch) return "https://" + bareMatch[0]

  return input
}
```

### 平台渐变色标签

| 平台 | 渐变色 |
|---|---|
| Instagram | 橙→粉→紫 `#F58529 → #DD2A7B → #8134AF` |
| Pixiv | 蓝 `#0096FA → #005BFF` |
| Twitter/X | 天蓝→黑 `#1DA1F2 → #14171A` |
| 小红书 | 红 `#FE2C55 → #FF6B6B` |
| DeviantArt | 绿 `#05CC47 → #008A3D` |

### vConsole 调试日志

图片解析流程复杂（短链解析→HTML抓取→JSON提取→CDN选择），前端集成了全链路 vConsole 日志：

```typescript
console.log("[图片保存] ========== 开始解析 ==========")
console.log("[图片保存] 原始输入:", rawInput)
console.log("[图片保存] 提取后链接:", url)
console.log("[图片保存] ✅ 解析成功, 平台:", result.platform)
console.log("[图片保存] 图片URL:", result.images.map(img => ({...})))
```

### 任务重新解析修复

`a341dbc` 修复了一个重要 bug：任务历史中点击「重新解析」，图片/音乐任务错误跳转到视频页面。

```typescript
// 修复前：全部跳转视频页
// 修复后：pageMap 按 type 路由
const pageMap: Record<string, string> = {
  video: "/pages/video/index",
  music: "/pages/music/index",
  image: "/pages/image/index",
}
```

同时 image 和 music 页面新增 `onShow` 钩子，自动读取 `reparse_url` 参数并触发解析。

---

## 📦 Celery 异步下载

```python
# server/app/tasks/image_task.py (248 行)
@app.task(bind=True, max_retries=2)
def download_image_task(self, url, title, platform, selected_indices,
                        range_start, range_end, quality):
    # 1. gallery-dl 下载到临时目录
    # 2. ZIP 打包
    # 3. 上传 COS
    # 4. 通过 Redis Pub/Sub 推送进度
    # 5. 清理临时文件
```

---

## 🧹 代码清理

- 删除空目录 `components/`、`static/tab/`
- 删除未使用文件 `logo.png`、`research.md`、`cos.py`
- 修复拼写错误 `shime-uni.d.ts` → `shims-uni.d.ts`
- `.gitignore` 新增 `server/downloads/` 忽略规则

---

## 📊 变更规模

| 维度 | 数据 |
|---|---|
| 变更文件 | 19 个 |
| 新增代码 | ~2500 行 |
| 删除代码 | ~125 行（清理） |
| 新增后端文件 | xhs_service.py (447行) / gallery_service.py (458行) / image_task.py (248行) / image.py API (211行) / image schemas (51行) |
| 新增前端文件 | image API (72行) / image 页面重写 (686行) |
| 支持图站 | gallery-dl 90+ + 小红书独立 |

---

## 验证结果

| 测试项 | 结果 |
|---|---|
| 小红书笔记解析（短链接→图片列表） | ✅ |
| 小红书 CDN 画质选择（nd_dft 优先） | ✅ |
| Instagram/Twitter/Pixiv gallery-dl 解析 | ✅ |
| 分享文案中自动提取 URL | ✅ |
| 缺协议链接自动补全 | ✅ |
| 图片网格全选/单选/反选 | ✅ |
| Celery 异步下载 → COS 上传 | ✅ |
| 任务重新解析正确跳转 | ✅ |
| 前端 H5 + 小程序双端编译 | ✅ |

---

## 下一步

Sprint 6：前端边界体验打磨（错误状态/空状态/网络检查）+ 真机测试 + 微信审核准备。
