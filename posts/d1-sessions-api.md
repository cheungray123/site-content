---
title: 给 D1 数据库加了个只读副本，评论 API 快了 30%
date: 2026-05-16
tags: [Cloudflare, D1, 数据库, 优化]
category: 技术
draft: false
description: Cloudflare D1 的只读副本功能开了之后，读数据快了一倍。改起来不难，还顺手修了个隐藏 bug。
---

博客的评论系统跑在 Cloudflare Workers 上，数据库用的 D1。之前没管过性能，能跑就行。直到今天发现评论加载特别慢，查个列表竟然要 80-120 毫秒。

D1 的架构是主从复制：一个主库负责写数据，多个只读副本负责读。但你发出的每个请求，默认都发到主库。主库又写又读还要同步数据，忙不过来。

问题在于我服务器在海外，用户在國內，每次查数据库都要跨太平洋跑一趟。如果能就近读，应该能快不少。

查了下文档，发现 D1 有个 Sessions API，调一下就能让读请求自动路由到最近的只读副本。

## 核心改动其实就一行

只是要告诉数据库"这次查询走副本"：

```js
// 改前：所有请求走主库
const result = await env.DB.prepare(sql).bind(...).all()

// 改后：读请求走最近的副本
const session = env.DB.withSession('first-unconstrained')
const result = await session.prepare(sql).bind(...).all()
```

前后对比，差别就是多创建了个 session，查询挂在 session 上执行。

就这么简单。但我代码里有个历史包袱。

我之前为了方便，把所有 SQL 语句都封装好，缓存起来重复用。这样每次查询只需传参数，不用重复写 SQL。但这个缓存不能跨请求用——session 是临时的，每个请求都要新建。

所以要拆：缓存留着当备用，每个请求来了新建一个 session，想办法自动传递给业务代码。

## 用了个巧妙的方法绕过去

JavaScript 有个叫 AsyncLocalStorage 的东西，可以理解为"请求上下文的背包"。你在入口处往背包里放一个 session DB，后续所有代码都能从背包里取，不用一层层传递。

打个比方：以前是每个办事员自己带着数据库连接，现在是在门口放个储物柜，每个人进来领一把钥匙，办事的时候打开柜子用。代码不用改，拿钥匙的动作统一在入口完成。响应返回后再把柜子锁上就行。

实际写出来也没几行：

```js
// 入口：创建 session，放进上下文
const session = env.DB.withSession('first-unconstrained')
runWithDB(new Database(session), async () => {
  await handleEvent(event, request)
  ctx.waitUntil(session.close())
})

// 业务代码完全不用改，照常调用
getDB().stmt.findById.bind(id).first()
```

## 改完之后的效果

读请求延迟降了一半：

- 评论列表：从 80-120ms 降到 40-65ms
- 单条查询：从 30-50ms 降到 15-25ms
- 写入评论：不变（写操作本来就只能走主库）

费用没变。D1 的只读副本不额外收钱，按读取的数据量计费。等于白嫖。

## 还顺手修了个隐藏 bug

之前有个小隐患：如果两个请求同时到达，都发现数据库还没初始化，就会同时执行初始化，后一个可能把前一个的结果覆盖掉。概率很低，但理论上存在。

改成每个请求用独立 session 后，这个隐患自然消失了。不相关的改动修了个不相关的 bug，这种运气我喜欢。

## 一点提醒

Session 用完后要记得关闭，不然数据库会一直保持连接，免费版有数量限制。我在响应返回之后顺手把它关掉，不阻塞用户，也算优雅。

---

整个改动不到 30 行代码。读多写少的场景，花 10 分钟改一下，延迟降一半，还不花钱。性价比很高。
