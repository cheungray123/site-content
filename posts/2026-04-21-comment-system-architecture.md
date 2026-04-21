---
title: "评论系统架构解析：从设计到实现"
date: 2026-04-21T00:00:00.000Z
tags: ["架构", "Cloudflare Workers", "评论系统", "技术分享"]
category: "技术"
description: "基于 Cloudflare Workers + D1 + R2 的无服务器评论系统架构设计，包含事件驱动通信、评论树构建、垃圾检测和图片上传等核心模块的详细解析。"
draft: false
---

评论系统这东西，说简单也简单，说复杂也复杂。无非就是存数据、查数据。但真要做到好用，还是有些东西值得聊聊。

我的评论系统跑了很久了，没出过什么毛病。最近整理了一下代码，顺手写了这篇。

## 设计思路

选 Cloudflare Workers 的原因很简单：便宜、速度快、不用管服务器。Workers + D1 + R2，三个服务搞定一切。

数据库用的是 D1，实际上就是个 SQLite，但有全球加速。R2 存图片，比 GitHub 稳定且免费额度够用。

## 通信协议

这套系统没用 REST，而是用事件驱动。

前端发请求带一个 `event` 字段，Worker 里 `switch(event)` 分发到不同处理器。返回格式统一是 `{ code, data, message }`。`code = 0` 表示成功，其他都是错误码。

这样设计的好处是前端和后端完全解耦，后端改不影响前端调用。

```javascript
// 前端发的是这样
{ event: 'COMMENT_SUBMIT', url: '/posts/xxx', comment: '...', ... }

// 后端返回是这样
{ code: 0, data: { id: 'xxx' }, message: '' }
```

## 数据库设计

D1 是 SQLite 结构，评论表字段不少，但用起来还好。

```sql
CREATE TABLE comment (
  _id TEXT PRIMARY KEY,       -- UUID
  uid TEXT NOT NULL,          -- 用户ID
  nick, mail, link,          -- 用户信息
  url, href,                 -- 页面路径
  comment TEXT,              -- HTML 内容
  pid, rid,                  -- 回复关系
  isSpam, master, top,       -- 状态标记
  like TEXT DEFAULT '[]',    -- 点赞用户列表
  created, updated,          -- 时间戳
  ipRegion TEXT              -- IP 地区
);
```

索引按查询场景建的：按页面查、按时间查、按 IP 统计。够用就行，没过度优化。

## 评论树构建

评论展示的核心是树形结构，但 D1 不支持递归查询，所以得在代码里构造。

逻辑是 O(n) 的：先扫描一遍按 `rid` 分组，再遍历构建树。代码里用了 Map 做预处理：

```javascript
// O(n) 预处理
const replyMap = new Map()
for (const comment of comments) {
  if (comment.rid) {
    if (!replyMap.has(comment.rid)) replyMap.set(comment.rid, [])
    replyMap.get(comment.rid).push(comment)
  }
}

// O(n) 构建树
for (const comment of comments) {
  if (!comment.rid) {
    const replies = replyMap.get(comment._id) || []
    tree.push(formatComment(comment, replies))
  }
}
```

置顶评论单独处理，在查询阶段就优先取出。

## 垃圾检测

检测分两层：预检测和异步检测。

提交时先过预检测，违禁词直接拒掉，长度超限直接拒掉。没问题的进入待审状态，然后异步跑 Akismet。

如果配置了 `AKISMET_KEY = MANUAL_REVIEW`，就进人工审核模式——有历史正常评论的用户直接放行，其他人全部待审。

腾讯云内容安全本来也接了，但 Workers 里实现 HMAC-SHA1 签名太麻烦，暂跳过了。

## 限流

限流用滑动窗口，存 D1 里。每 10 分钟算一个时间窗口，查当前 IP 和全局的评论数。

```javascript
const since = Date.now() - 600000
const countByIp = await db.stmt.countByIpSince.bind(since, ip).first('count')
const totalCount = await db.stmt.countSince.bind(since).first('count')
```

本地开发环境跳过限流，避免调试时卡自己。

## 图片上传

评论里的图片先 base64 传上来，Worker 解码后存 R2，返回公开 URL。

删除评论时会清理 R2 里的图片，通过正则匹配 `<img src>` 然后逐个删。有点糙，但够用。

## 管理员功能

管理面板用的 Cookie 认证，密码 SHA-256 存 D1 里。没有注册功能，初始化时在控制台设一次密码。

管理员可以查看所有评论（包括垃圾），删除、置顶、标记垃圾。导出导入支持 Disqus 和 WordPress 格式。

## 部署

Workers 部署很简单：


`wrangler.toml` 里绑定 D1、R2、KV：

```toml
[[d1_databases]]
binding = "DB"
database_name = "comment"
database_id = "xxx"

[[r2_buckets]]
binding = "R2"
bucket_name = "comment"

[[kv_namespaces]]
binding = "KV"
id = "xxx"
```

环境变量里设 `R2_PUBLIC_URL` 和 `REDIRECT_URL`。

## 踩过的坑

D1 的 prepared statement 在 Workers 重启后会失效，所以加了 binding 变化检测，重用已有数据库实例。

评论树构建时 `rid` 为空的是主评论，有值的是回复。查询时用 `rid = ''` 找主评论。

垃圾检测的异步任务用 `Promise.race` 限制 5 秒，超时就跳过，避免拖慢主流程。

---

这套系统跑了大半年，稳得很。最早想用传统的 REST API，后来发现事件驱动更好扩展，代码也更清晰。