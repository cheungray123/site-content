---
title: 写了一个 RSS 博客订阅自动监控工具
date: 2026-04-28
tags: [RSS, 飞书, 邮件, 自动化]
category: 技术
draft: false
description: 零依赖的 RSS 监控工具，发现新文章自动推送到飞书和邮箱。支持 RSS/Atom/JSON Feed，不用盯着博客看了。
---

现在去看别人博客都是从评论区进去的。这样有点太麻烦了！干脆写了个脚本，每 30 分钟跑一次，关注的博主有新文章就推飞书和邮箱。

[项目地址](https://github.com/cheungray123/rss-robot)

零依赖、纯 Node.js、单文件 900 行。GitHub Actions 跑，省得自己 VPS。

## 支持的格式

RSS 2.0、Atom、JSON Feed 都认。碰到网站首页而不是 Feed 地址，会自动找 `<link rel="alternate">`，找不到就试常见路径（`/feed/`、`/rss/`、`/atom.xml` 等）。

还支持自定义 JSON API。配个 path 和 mapping 字段就行，比如:

```yaml
feeds:
  - url: https://api.example.com/posts
    format: json
    path: data.list
    feedTitle: 某某博客
    mapping:
      title: title
      link: url
      pubDate: publishedAt
```

## 订阅源配置

两种方式。

**远程 JSON（推荐）**：设 `FEEDS_URL` 环境变量指向一个 JSON 文件，修改订阅源不用推代码。可以放 GitHub Gist、自己的服务器、或者 COS/OSS 公开链接。

```json
{
  "feeds": [
    { "url": "https://www.ruanyifeng.com/blog/atom.xml" },
    { "url": "https://api.example.com/posts", "format": "json", "path": "data.list" }
  ]
}
```

**本地 feeds.yml**：当 `FEEDS_URL` 不可用时回退使用。

```yaml
feeds:
  - https://www.ruanyifeng.com/blog/atom.xml
  - https://www.zhangxinxu.com/wordpress/
```

两种格式都支持简单写法（只写 URL）和完整写法（带 format、path、mapping）。

## 通知方式

**飞书**：在群里建自定义机器人，复制 Webhook 地址。环境变量设 `FEISHU_WEBHOOK_URL`。

**邮件**：三种方式。

SMTP（国内邮箱，QQ/163/126/阿里都行）：

```bash
EMAIL_PROVIDER=smtp
SMTP_HOST=smtp.qq.com
SMTP_PORT=465
SMTP_USER=xxx@qq.com
SMTP_PASS=xxxxxxxxxxxxxxxx
```

Resend（海外，需要域名验证）：

```bash
EMAIL_PROVIDER=resend
EMAIL_API_KEY=re_xxxx
```

自定义 HTTP API：

```bash
EMAIL_PROVIDER=custom
EMAIL_API_URL=https://your-api.com/send
EMAIL_API_KEY=your-key
```

邮件会按博客分组，HTML 格式，标题、时间、作者都带。

## 部署

### 方式一：GitHub Actions

项目里有现成的工作流，`.github/workflows/check-feeds.yml`。每 30 分钟自动跑一次。

在仓库 Settings → Secrets and variables → Actions 添加这些 secrets：

| Secret | 说明 |
|--------|------|
| `FEISHU_WEBHOOK_URL` | 飞书 Webhook 地址 |
| `FEEDS_URL` | 远程 JSON 订阅源地址 |
| `EMAIL_PROVIDER` | `smtp` / `resend` / `custom` |
| `EMAIL_API_KEY` | Resend 或自定义 API 密钥 |
| `SMTP_HOST` | SMTP 服务器 |
| `SMTP_PORT` | 端口，默认 465 |
| `SMTP_USER` | 邮箱账号 |
| `SMTP_PASS` | 授权码 |
| `EMAIL_FROM` | 发件人 |
| `EMAIL_TO` | 收件人，多个用逗号分隔 |

配好之后不用管了。

### 方式二：本地跑

```bash
node check-feeds.js
```

Windows CMD:

```cmd
set FEISHU_WEBHOOK_URL=https://open.feishu.cn/open-apis/bot/v2/hook/xxx
set EMAIL_PROVIDER=smtp
set SMTP_HOST=smtp.qq.com
set SMTP_PORT=465
set SMTP_USER=xxx@qq.com
set SMTP_PASS=xxxxxxxxxxxxxxxx
set EMAIL_FROM=博客订阅 ^<xxx@qq.com^>
set EMAIL_TO=target@163.com
node check-feeds.js
```

PowerShell 用 `$env:VAR="value"` 语法。

## 工作原理

流程很简单：

1. 加载订阅源（优先远程 JSON，回退本地 feeds.yml）
2. 逐个请求，用正则识别 RSS/Atom，自动发现 JSON Feed
3. 和本地已读记录对比，过滤已推送过的
4. 只推送 24 小时内发布的新文章（首次运行不推送，避免批量轰炸）
5. 更新 `data/seen-articles.json`，超过 500 条自动清理旧记录
6. 邮件支持 SMTP（零依赖自己实现）、Resend API、自定义 HTTP 三种方式

已读记录会 commit 回仓库，其他机器跑的时候也能用。

## 踩过的一些坑

**SMTP 认证**。国内邮箱用授权码不是登录密码。QQ 邮箱在设置 → 账户 → POP3/SMTP 那里开，生成 16 位授权码。

**时间单位**。文章发布时间存的是 ISO 字符串，比较的时候统一转时间戳。

**首次运行**。脚本会抓所有历史文章但不发送通知。第二次运行才开始推送，避免一上来收一堆旧文章。

**SMTP 零依赖**。没用 nodemailer，自己实现了个简单 SMTP 客户端。SSL 用 465 端口，STARTTLS 用 587。

---

不用再一个个博客刷新了。配好之后坐着等通知就行。
