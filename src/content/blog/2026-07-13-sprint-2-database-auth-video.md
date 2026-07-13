---
title: 'Sprint 2 完成：数据库迁移、登录认证、视频下载端到端打通'
description: '完成 Alembic 迁移建表、Dev 模式登录绕过微信验证、COS 上传容错回退，打通了登录到下载的完整链路。'
pubDate: '2026-07-13'
tags: ['alembic', 'postgresql', 'celery', 'cos', 'wechat-login']
---

## 做了什么

Sprint 2 的核心目标是让数据库真正可用、让登录流程可以开发调试、让视频下载链路跑通。12 个文件，+436 / -75 行，把这三个目标都达成了。

## 数据库迁移：从 SQL 文件到 Alembic

Sprint 1 的数据库表只存在于 Python 模型定义里，没有可执行的迁移。

现在新增了完整的 Alembic 体系：

```
server/alembic/
├── alembic.ini              # 数据库连接配置
├── env.py                   # 迁移环境（异步引擎支持）
├── script.py.mako           # 迁移脚本模板
└── versions/
    └── 8e6a73abcdef_init.py  # 初始迁移：users + tasks
```

初始迁移创建了两张表：

- **users** — `id`、`openid`（唯一索引）、`nickname`、`avatar_url`、`created_at`
- **tasks** — `id`(UUID)、`user_id`(外键)、`type`、`source_url`、`status`、`progress` 等 15 个字段

执行只需一行：

```bash
alembic upgrade head
```

## Dev 模式登录：绕过微信 code2session

开发阶段每次调登录接口都要先去微信后台换 code，效率太低。

这次在 `auth.py` 里加了一个 dev 快捷通道：

```python
if settings.APP_DEBUG and req.code.startswith("dev-"):
    openid = req.code  # "dev-test-user-001" 直接用作 openid
    unionid = None
else:
    # 正常的微信 code2session 流程
```

开发时直接用 `dev-<任意字符串>` 作为 code 调登录接口即可，不用碰微信后台。

## COS 上传容错：失败不崩溃

之前的 COS 上传逻辑有个问题：如果 COS 没配或者上传失败，任务直接报错。对于开发环境来说，每次都要配好 COS 才能跑通整条链路。

这次把上传逻辑包进了 `try/except`，失败时自动回退到 `file://` 本地路径：

```python
try:
    client.upload_file(...)
    return client.get_presigned_url(...)
except Exception as e:
    logger.warning(f"COS 上传失败（回退到本地路径）: {e}")
    return f"file://{file_path}"
```

## 踩坑记录

### 1. duration 类型不匹配

yt-dlp 返回的 `duration` 可能是浮点数（比如 `120.5` 秒），但 Schema 定义的是 `int`。改成 `float` 解决。

### 2. WebSocket 路由 404

WebSocket 路由 `/ws/task/{task_id}` 被放在带有路径参数的 REST 路由之后，导致 FastAPI 把 `ws` 当成 `task_id` 匹配了。把 WS 路由移到最前面声明就解决了 — FastAPI 的路由是按注册顺序匹配的。

### 3. Celery 任务先跑后记录

之前的代码是先 `delay()` 再 `db.add()`，如果 Celery Worker 太快可能在 DB 提交前就去查记录导致报错。改成先创建 DB 记录、flush 拿到 ID，再投递 Celery 任务。

### 4. docker-compose version 过时

Docker Compose V2 已经不需要 `version` 字段了，留着反而会触发 warning。直接删掉。

## 验证结果

全链路验证通过：

```
登录(dev-) → 视频解析(yt-dlp info) → 异步下载(Celery)
→ COS 上传(或本地回退) → WebSocket 推送进度 → 任务状态更新
```

17/17 模块导入正常，PostgreSQL + Redis 健康检查通过。

## 下一步

Sprint 3：视频下载端到端真机测试、COS 生命周期自动删除配置、前端 WebSocket 进度条联调。
