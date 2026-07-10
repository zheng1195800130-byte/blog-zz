---
name: blog-generate
description: 从 DowMore 项目的最新 commits 生成博客文章并发布。检查 d:\小程序\dowmore 的最近提交，分析代码变更，生成中文技术博客，提交到 blog-zz 项目。
---

# 博客生成 Skill

## 触发方式

用户输入以下任一即可触发：
- `/blog-generate`
- "帮我生成一篇博客"
- "写篇开发日志"
- "更新博客"

## 执行流程

### 第一步：同步项目代码

```bash
cd d:/小程序/dowmore
git pull origin master
```

### 第二步：确定检查范围

读取 `d:/blog-zz/.claude/skills/last-check.json` 获取上次检查的时间戳。
如果文件不存在，只分析最近 **1 个** commit（首次运行不要太猛）。

格式：
```json
{
  "lastCheck": "2026-07-09T00:00:00+08:00",
  "lastCommitHash": "ed2aea5"
}
```

### 第三步：获取新 commits

```bash
cd d:/小程序/dowmore
git log --since="<lastCheck>" --format="%H|%ai|%s" --reverse
```

如果 `lastCommitHash` 存在，加上 `git log <lastCommitHash>..HEAD` 确保不遗漏。

### 第四步：分析每个 commit 的变更

对每个新的 commit：
```bash
git show <hash> --stat        # 看改了哪些文件
git show <hash> --format="%B" # 看完整 commit message
```

对于重要的文件变更，用 `git diff <hash>^ <hash> -- <文件>` 看具体 diff。

### 第五步：生成博客文章

根据分析结果，生成一篇 Markdown 博客文章，放到 `d:/blog-zz/src/content/blog/<date>-<slug>.md`。

#### 文章格式要求

```markdown
---
title: '文章标题（中文，简洁有力）'
description: '一段话概述今天做了什么，50-100字'
pubDate: '2026-07-10'
tags: ['标签1', '标签2']
---

正文内容...
```

#### 写作风格要求

- **中文**：所有内容用中文撰写
- **技术向**：重点讲技术决策、架构变化、踩坑经验，不要流水账
- **有代码**：适当贴关键代码片段，用 ``` 代码块
- **有思考**：不只是"做了什么"，还要写"为什么这样做"
- **适度长度**：300-800 字，视变更量而定
  - 如果只有 1 个小 commit，文章可以短（200-300 字）
  - 如果 commit 很多，挑重点写，控制在 800 字以内
- **标签规范**：用 kebab-case，如 `fastapi`、`wechat-miniapp`、`vue3`、`docker`

#### 文章模板参考

```markdown
---
title: '项目脚手架搭建完成：从零搭建 DowMore 前后端'
description: '完成 uni-app + FastAPI 项目脚手架，打通视频下载核心链路'
pubDate: '2026-07-10'
tags: ['uni-app', 'fastapi', 'celery', 'project-init']
---

## 做了什么

今天完成了 DowMore 项目的脚手架搭建，前后端的基本骨架已经就位。

## 技术选型

- **前端**：uni-app (Vue 3 + Composition API) + uview-plus
- **后端**：FastAPI + Celery + PostgreSQL

## 关键决策

### 文件只做临时中转
所有下载文件在 COS 保留不超过 1 小时……

### 默认最高画质
yt-dlp 调用参数默认 `bestvideo*+bestaudio/best`……

## 踩坑记录

中间遇到一个问题……（如果有的话）

## 下一步

明天开始数据库表设计和 Alembic 迁移。
```

### 第六步：提交到博客仓库

```bash
cd d:/blog-zz
git add src/content/blog/<新文章>.md
git commit -m "blog: <文章标题>"
git push origin main
```

### 第七步：更新检查记录

更新 `d:/blog-zz/.claude/skills/last-check.json`：
```json
{
  "lastCheck": "<当前时间的 ISO 格式>",
  "lastCommitHash": "<最新的 commit hash>"
}
```

## 注意事项

1. **去重**：如果某个 commit 已经生成过文章（通过 slug 判断），跳过
2. **无变更处理**：如果没有新的 commits，告诉用户"项目最近没有新的提交，无需生成文章"，不生成空文章
3. **失败处理**：如果 git pull 失败（比如没有网络），告知用户并退出
4. **文章 slug**：格式 `<YYYY-MM-DD>-<英文关键词>`，如 `2026-07-10-project-scaffold`
5. **原创性**：每篇文章根据实际 diff 内容生成，不要套话模板，确保每篇都有独特的技术干货
