---
title: 飞书同步机器人 — 部署教程
date: 2026-04-26
tags: [Cloudflare Workers, 飞书, 自动化, 部署]
category: 技术
draft: false
description: 在飞书写完文档，丢一句"同步"就自动推到博客。顺便聊聊这个机器人的架构。
---

飞书文档写完了想同步到博客。以前靠手复制粘贴，写一次，改一次，重新部署一次。后面写了个机器人，飞书里丢一句命令就完事。

项目地址: [github.com/cheungray123/easte-feishu-server](https://github.com/cheungray123/easte-feishu-server)

支持文章、说说、相册三种类型。图片自动传到 R2 或 GitHub。改过的文档只增量更新，不重复推送。

## 架构

一个 Cloudflare Worker，单文件，Hono 框架，大概 900 行。

流程很简单:

```
飞书发消息 → Worker 收 webhook → 调飞书 API 拉文档
→ 转 Markdown → 处理图片 → 推 GitHub → 触发博客构建
```

几个核心模块:

- **命令解析** — 收到飞书消息，判断是同步还是删除。中文英文都认。
- **文档解析** — 递归遍历飞书文档块树，按类型转 Markdown。标题、列表、表格、代码块都处理。
- **图片管线** — 从飞书下载原图，上传到 R2 或 GitHub，替换 Markdown 里的图片链接。
- **增量逻辑** — 比对飞书修改时间和 GitHub 文件的上次提交时间，旧的跳过。
- **GitHub 推送** — 写到 site-content 仓库，再触发博客的构建 workflow。
- **删除确认** — 删东西先存 KV，等用户回复 Y 才执行。

## 部署

需要准备的东西:

1. 一个 Cloudflare 账号
2. 飞书开放平台的应用
3. GitHub Personal Access Token（repo + workflow 权限）

把仓库 clone 下来，用 `wrangler deploy` 部署。

## 配置 secrets

在 Cloudflare Workers 控制台 → Settings → Variables → Secrets 添加:

| 变量名 | 说明 |
|--------|------|
| `FEISHU_APP_ID` | 飞书应用 ID |
| `FEISHU_APP_SECRET` | 飞书应用 Secret |
| `GITHUB_PAT` | GitHub Token，需要 repo 权限 |
| `GITHUB_OWNER` | GitHub 用户名 |
| `GITHUB_CONTENT_REPO` | 放文档 Markdown 的仓库 |
| `GITHUB_CLIENT_REPO` | 博客前端仓库 |
| `GITHUB_WORKFLOW_ID` | 触发构建的 workflow 文件名 |
| `FEISHU_FOLDER_ID` | 飞书文件夹 ID，文章放这里 |
| `FEISHU_MOMENTS_FOLDER_ID` | 说说文件夹 ID（可选） |
| `FEISHU_SIGNING_SECRET` | 事件订阅签名密钥（可选） |
| `IMG_BUCKET` | R2 Bucket 绑定（可选） |
| `R2_PUBLIC_URL` | R2 公开访问地址（可选） |
| `PENDING_KV` | KV 命名空间，存待确认删除 |

wrangler.toml 里配好 R2 和 KV 的 binding。

## 飞书应用配置

在[飞书开放平台](https://open.feishu.cn/)创建应用，权限勾这几个:

- `docx:document:readonly` — 读文档
- `drive:file:readonly` — 读文件夹
- `im:message:send_as_bot` — 发消息

事件订阅配请求地址: `https://你的worker/webhook/feishu`，开 `im.message.receive_v1`。

发布应用。

## 命令

部署好了在飞书给机器人发:

| 命令 | 功能 |
|------|------|
| `同步` | 全量同步所有文章 |
| `同步 <标题>` | 单篇同步，支持中文模糊匹配 |
| `删除 <标题>` | 删文章（要确认） |
| `同步说说 <ID>` | 同步单条说说 |
| `删除说说 <标题>` | 删说说（要确认） |
| `删除相册 <标题>` | 删整个相册 |

删除要回 Y 确认，防止手滑。

## 增量同步

每次同步拿飞书文档的修改时间和 GitHub 的上次提交时间比。GitHub 上已经是新的就不动。不会每次都重写。

## 踩过的一些坑

表格渲染花了不少时间。飞书的 table block 没有行列嵌套结构，只有列数和一维子节点列表，得自己算行列索引。

消息去重也需要注意。飞书 webhook 会重试，不加去重每条消息重复回复。

图片 Base64 在 Workers 里用 btoa 有大小限制，后来换了 Buffer。

时间戳单位也不一样。飞书返回秒，GitHub 返回毫秒，不归一化永远对不上。

封面 URL 里有特殊字符，直接塞 frontmatter 会炸，需要 encode。

---

就这样。在飞书写完 -> 发条消息 -> 自动到博客。不用再手动粘贴了。
