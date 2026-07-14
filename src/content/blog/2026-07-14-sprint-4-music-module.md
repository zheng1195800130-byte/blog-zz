---
title: 'Sprint 4：音乐聚合搜索/播放/下载全链路完成 + 实时进度推送'
description: '40 个文件 4300 行变更：4 平台 Provider 架构（QQ/酷狗/酷我/咪咕）、跨平台降级播放、链接解析下载、Redis Pub/Sub → WebSocket 实时进度条。'
pubDate: '2026-07-14'
project: 'dowmore'
tags: ['music', 'provider-pattern', 'websocket', 'redis-pubsub', 'reverse-engineering', 'kugou', 'kuwo']
---

## 概述

音乐模块是 DowMore Phase 1 第二阶段的核心功能，分两个提交完成：

| 提交 | 内容 |
|---|---|
| `b0c4a65` | 音乐搜索模块（Provider 架构 + QQ/咪咕 + 聚合搜索） |
| `6ec2930` | 全链路补齐（酷狗/酷我 + 降级播放 + 链接下载 + 实时进度） |

最终实现了**搜歌 → 播放 → 下载**完整闭环，覆盖 6 个音乐平台，支持分享链接直接解析下载。

---

## 🏗️ Provider 模式音乐平台架构

这是后端设计上最大的亮点——为每个音乐平台编写独立的 Provider，统一接口、各管各的：

```
server/app/services/music_providers/
├── __init__.py          # Provider 注册表
├── base.py              # 抽象基类 BaseProvider + SongResult
├── qq.py                # QQ音乐 Provider
├── migu.py              # 咪咕音乐 Provider
├── kugou.py             # 酷狗音乐 Provider
└── kuwo.py              # 酷我音乐 Provider
```

### Provider 接口

```python
class BaseProvider(ABC):
    @abstractmethod
    def name(self) -> str: ...

    @abstractmethod
    def search(self, keyword: str, limit: int = 20) -> list[SongResult]: ...

    @abstractmethod
    def get_stream_url(self, song_id: str, extra: dict | None = None) -> Optional[str]: ...
```

每个 Provider 只需实现 `name` / `search` / `get_stream_url` 三个方法，注册表自动发现和管理：

```python
# __init__.py — 自动注册
_providers: dict[str, BaseProvider] = {}

def register_provider(provider: BaseProvider):
    _providers[provider.name()] = provider

def get_provider(name: str) -> Optional[BaseProvider]:
    return _providers.get(name)

def get_all_providers() -> list[BaseProvider]:
    return list(_providers.values())
```

新增平台只需实现 Provider 基类并在 `__init__.py` 注册，无需改动聚合搜索和 API 层代码。**对扩展开放，对修改关闭**。

---

## 🔍 四平台聚合搜索

使用 `asyncio.gather` 并发请求四个平台，汇总结果按匹配度排序：

```python
async def search_across_platforms(keyword: str, platform: str | None = None, limit: int = 20):
    if platform:
        provider = get_provider(platform)
        return provider.search(keyword, limit) if provider else []

    # 并发搜索所有平台
    providers = get_all_providers()
    tasks = [asyncio.to_thread(p.search, keyword, limit) for p in providers]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    songs = []
    for r in results:
        if isinstance(r, list):
            songs.extend(r)
    return sorted(songs, key=lambda s: _match_score(s, keyword), reverse=True)
```

### 各平台搜索方案

| 平台 | 搜索接口 | 播放链接接口 | 特点 |
|---|---|---|---|
| QQ音乐 | `smartbox` API | `vkey` 播放链接 | 需 vkey 鉴权 |
| 咪咕 | 搜索 API + SPA 适配 | 自动补全备用 | RESTful 风格 |
| 酷狗 | `mobilecdn` 移动端 API | `getSongInfo` + CDN tracker | MD5 hash 鉴权 |
| 酷我 | `search.kuwo.cn` + Web API 双通道 | `antiserver` 接口 | JS 字面量解析 |

### 酷我 JS 字面量解析

酷我搜索接口返回的是 **JS 对象字面量**（单引号），不是标准 JSON。处理方式：

```python
try:
    data = json.loads(text)
except json.JSONDecodeError:
    import ast
    data = ast.literal_eval(text)  # Python 字面量解析
```

### 酷狗获取播放链接 — 双通道回退

```python
def get_stream_url(self, song_id: str, extra: dict | None = None) -> Optional[str]:
    # 方法1: m.kugou.com 的 getSongInfo 接口
    url = self._get_stream_v1(song_id)
    if url:
        return url
    # 方法2: CDN tracker 接口（需要 album_id + MD5 签名）
    if extra and extra.get("album_id"):
        url = self._get_stream_cdn(song_id, extra["album_id"])
        if url:
            return url
    return None
```

---

## 🔀 跨平台降级播放

QQ/酷狗/咪咕的播放链接通常有时效或鉴权限制。系统实现了**自动跨平台降级**：

```python
async def get_stream_with_fallback(platform, song_id, song_name, artist, extra):
    # 1. 先尝试原始平台的播放链接
    url = await get_song_stream_url(platform, song_id)
    if url:
        return url, platform

    # 2. 失败 → 用酷我搜同款歌曲（酷我免费歌曲多、链接稳定）
    kuwo = get_provider("kuwo")
    if kuwo:
        search_keyword = f"{song_name} {artist}" if artist else song_name
        results = kuwo.search(search_keyword, limit=3)
        if results:
            url = kuwo.get_stream_url(results[0].id, results[0].raw)
            if url:
                return url, "kuwo"

    return None, platform
```

这意味着用户搜到一首 QQ 音乐的 VIP 歌曲，系统会自动在酷我找到同款免费版并提供播放。

---

## 🔗 链接解析下载

支持直接粘贴音乐分享链接，覆盖 6 个平台 + yt-dlp 兜底：

```python
LINK_PATTERNS = [
    (r"y\.qq\.com/n/ryqq/songDetail/(\w+)", "qq"),          # QQ音乐
    (r"kugou\.com/song/#hash=(\w+)", "kugou"),               # 酷狗
    (r"kuwo\.cn/play_detail/(\d+)", "kuwo"),                 # 酷我
    (r"music\.163\.com/song\?id=(\d+)", "netease"),          # 网易云
    (r"music\.migu\.cn/.+?/(\d+)", "migu"),                  # 咪咕
    (r"bilibili\.com/audio/au(\d+)", "bilibili"),            # B站
]
```

解析流程：正则匹配 → 平台专用 Provider 提取 → yt-dlp 兜底。yt-dlp 对于有官方 API 受限的平台（如网易云）能直接提取可用流。

---

## 📡 Redis Pub/Sub → WebSocket 实时进度推送

这是 Srint 4 架构层面的一个重要升级——下载进度从轮询改为推送：

```
Celery Worker 下载中
  → redis.publish(f"task:{task_id}", json.dumps({"progress": 45, "status": "downloading"}))
  → FastAPI WebSocket 订阅 redis 频道
  → 实时推送给小程序前端
```

```python
# server/app/utils/redis_pubsub.py
import redis.asyncio as aioredis
from app.core.config import settings

async def publish_task_progress(task_id: str, progress: int, status: str, message: str = ""):
    r = aioredis.from_url(settings.REDIS_URL)
    await r.publish(
        f"task:{task_id}",
        json.dumps({"progress": progress, "status": status, "message": message}),
    )
    await r.close()
```

### 下载失败自动重试

音乐流地址有严格的时效性（部分平台几小时就过期），系统实现了**自动刷新重试**：

```python
MAX_RETRIES = 3
for attempt in range(MAX_RETRIES):
    try:
        download_file(url, output_path)
        break
    except DownloadError:
        if attempt < MAX_RETRIES - 1:
            url = await refresh_stream_url(platform, song_id, song_name, artist)
```

---

## 🎨 前端——音乐页面

从「即将开放」占位页变成了完整双模式功能页：

### 搜索模式

- 搜索栏 + 搜索按钮
- 输入歌名或歌手，回车触发搜索
- 搜索结果列表（歌曲名 / 歌手 / 平台标签 / 封面）

### 链接下载模式

- 粘贴框（QQ/酷狗/酷我/网易云/咪咕/B站 分享链接）
- 自动识别平台 → 解析歌曲信息 → 显示详情 → 确认下载

### 迷你播放器

- 内嵌播放控制器（播放 / 暂停）
- 从酷我获取的稳定流地址直接播放

### 音乐页面移入主包

之前音乐页面在小程序分包中，游客模式无法访问。这次移到了主包，确保所有用户都能使用。

---

## 🐛 修的坑

- **localhost → 127.0.0.1**：Windows IPv6 导致 `localhost` 解析失败，全部改为 `127.0.0.1`
- **showLoading/hideLoading 配对**：之前 loading 显示后某些异常路径没调用 hide，导致假死
- **任务历史「重新解析」**：根据 `type` 字段自动跳转对应模块（video/music/image）
- **链接下载流程重写**：先 yt-dlp 提取信息，失败再降级酷我搜索

---

## 📊 变更规模

| 维度 | 数据 |
|---|---|
| 变更文件 | 25 + 17 = 42 文件 |
| 新增代码行 | ~4300 行 |
| 新增 Provider | QQ / 咪咕 / 酷狗 / 酷我 |
| 新增 API 端点 | 5 个（search / stream / download / parse / link-download） |
| 支持搜索平台 | 4 个 |
| 支持链接下载平台 | 6 个 + yt-dlp 1800+ 站点 |

---

## 验证结果

| 测试项 | 结果 |
|---|---|
| 四平台聚合搜索 | ✅ 并发返回 |
| 酷我跨平台降级播放 | ✅ QQ VIP → 酷我免费版 |
| 链接解析（QQ/酷狗/酷我/网易云/咪咕/B站） | ✅ 正则匹配准确 |
| 下载 + 实时进度推送 | ✅ WebSocket 正常 |
| 下载失败自动刷新重试 | ✅ 3 次后放弃 |
| 前端 H5 + 小程序双端编译 | ✅ 通过 |

---

## 下一步

Sprint 5：图片批量下载模块（gallery-dl 集成）+ 前端错误状态/空状态/网络检查等边界体验打磨。
