---
title: 给 D1 数据库加了个只读副本，评论 API 快了 30%
date: 2026-05-16
tags: [Cloudflare, D1, 数据库, 优化]
category: 技术
draft: false
description: Cloudflare D1 的只读副本功能开了之后，读数据快了一倍。改起来不难，还顺手修了个隐藏 bug。
---

博客的评论系统跑在 Cloudflare Workers 上，数据库用的 D1。之前没管过性能，能跑就行。直到今天发现评论加载特别慢，查个列表竟然要 80-120 毫秒。

D1 的架构是主从复制：一个主库负责写，多个只读副本负责读。但默认所有请求都发到主库，又写又读还要同步数据，忙不过来。加上我的服务器在海外，用户在国内，每次查数据库都得跨太平洋跑一趟。

查了下文档，发现 D1 有个 Sessions API，可以让读请求自动路由到最近的只读副本。

## 核心改动其实就一行

告诉数据库"这次查询走副本"就行：

```js
// 改前：所有请求走主库
const result = await env.DB.prepare(sql).bind(...).all()

// 改后：读请求走最近的副本
const session = env.DB.withSession('first-unconstrained')
const result = await session.prepare(sql).bind(...).all()
```

但我代码里有个坑：之前为了方便，把所有 SQL 都封装好缓存起来重复用，每个查询只传参数就行。但这个缓存是跟数据库连接绑定的，session 每个请求都要新建，不能重用。

所以要拆：缓存留着当备用，每个请求来了新建一个 session，再想办法自动传给业务代码。

## 用 AsyncLocalStorage 绕过去

JavaScript 有个 AsyncLocalStorage，可以理解成"请求上下文的背包"。入口处往背包里放一个 session DB，后面所有代码都能直接从包里拿，不用一层层传参。

打个比方：以前是每个办事员自己保管数据库连接，现在门口放个储物柜，进来的人领把钥匙，办事时打开柜子用。响应返回再把柜子锁上。

```js
// 入口处：创建 session，放进上下文
const session = env.DB.withSession('first-unconstrained')
runWithDB(new Database(session), async () => {
  await handleEvent(event, request)
  ctx.waitUntil(session.close())
})

// 业务代码照常调用，完全不用改
getDB().stmt.findById.bind(id).first()
```

## 改完之后的效果

读请求延迟降了一半：

- 评论列表：从 80-120ms 降到 40-65ms
- 单条查询：从 30-50ms 降到 15-25ms
- 写入评论：不变（写操作本来就只能走主库）

费用也没变。D1 的只读副本不额外收钱，按读取量计费。等于白嫖。

## 还顺手修了个隐藏 bug

之前有个小隐患：两个请求同时到达，都发现数据库还没初始化，就会同时执行初始化，后一个可能把前一个覆盖掉。概率不高，但理论上存在。

改成每个请求独立 session 后，这个问题自然消失了。不相关的改动修了个不相关的 bug，这种运气我喜欢。

## 一点提醒

Session 用完记得关，不然数据库会一直保持连接，免费版有数量限制。我在响应返回后顺手关掉，不阻塞用户，也算优雅。

---

整个改动不到 30 行，读多写少的场景，花 10 分钟改一下，延迟降一半还不花钱。
