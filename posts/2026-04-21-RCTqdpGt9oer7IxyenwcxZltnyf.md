---
title: "给博客评论加个飞书通知"
date: 2026-04-21T17:38:10.606Z
tags: ["飞书，代码"]
category: "折腾"
description: "我的博客评论系统之前只有邮件通知。说实话邮件太慢了，而且我经常不看。因为我写文章一直用的飞书，[从飞书到博客：基于 Cloudflare Workers 的全自动私有文章同步系统](https%3A%2F%2Feaste.cc%2Fposts%2F2026-04-20-feishu-sync-sys"
image: "https://img.easte.cc/posts/images/H1pXbrnGho23tXxC2pVc9CgBnGe.jpg"
draft: false
---
## 起因

我的博客评论系统之前只有邮件通知。说实话邮件太慢了，而且我经常不看。因为我写文章一直用的飞书，[从飞书到博客：基于 Cloudflare Workers 的全自动私有文章同步系统](https%3A%2F%2Feaste.cc%2Fposts%2F2026-04-20-feishu-sync-system) 。所以想加个飞书消息，评论一进来就能收到。

## 思路

飞书开放平台有发送消息的 API，拿到 access_token 就能发。主要工作在后端，改动不大

## 核心逻辑

### 通知入口

评论提交后，`sendNotification` 会并行调用的通知函数：

await Promise.all([
  _notifyAdmin(comment, config),      // 邮件-管理员
  _notifyReply(comment, config, getParent),  // 邮件-回复
  _notifyByFeishu(comment, config, getParent)  // 飞书
])

### 飞书通知函数

async function _notifyByFeishu (comment, config, getParent) {
  // 1. 检查开关
  if (config.FEISHU_NOTIFY !== 'true') return

  // 2. 检查配置完整性
  if (!config.FEISHU_APP_ID || !config.FEISHU_APP_SECRET || !config.BLOGGER_OPEN_ID) return

  // 3. 博主本人不发
  if (config.BLOGGER_EMAIL && comment.mail === config.BLOGGER_EMAIL) return

  // 4. 获取 access_token
  const tokenRes = await fetch('https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal', {
    method: 'POST',
    body: JSON.stringify({ app_id, app_secret })
  })
  const tokenData = await tokenRes.json()
  if (!tokenData.tenant_access_token) return

  // 5. 判断是评论还是回复
  const isReply = !!comment.pid

  // 6. 如果是回复，获取父评论信息
  let parentInfo = null
  if (isReply && getParent) {
    try {
      const parent = await getParent(comment)
      if (parent) parentInfo = parent
    } catch (e) { /* 忽略，不阻断发送 */ }
  }

  // 7. 构建消息
  const title = isReply
    ? `💬 ${siteName} - 回复 @${parentInfo?.nick || '某位用户'} 的通知`
    : `💬 ${siteName} - 新评论通知`

  // 8. 内容分段构建
  const contentParts = [
    // 评论者
    [
      { tag: 'text', text: '👤 ' },
      { tag: 'text', text: comment.nick, bold: true },
      { tag: 'text', text: '  ' + (comment.mail || '无邮箱'), color: 'gray' }
    ]
  ]

  // 9. 回复的话加一行「回复 @xxx」
  if (isReply && parentInfo) {
    contentParts.push(
      [{ tag: 'hr' }],
      [
        { tag: 'text', text: '📝 回复 @' },
        { tag: 'text', text: parentInfo.nick, bold: true }
      ]
    )
  }

  // 10. 评论内容 + 链接
  contentParts.push(
    [{ tag: 'hr' }],
    [{ tag: 'text', text: displayText }],
    [{ tag: 'hr' }],
    [{ tag: 'a', text: '🔗 查看详情', href: postUrl }]
  )

  // 11. 发请求
  await fetch('https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=open_id', {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${tokenData.tenant_access_token}` },
    body: JSON.stringify({
      receive_id: BLOGGER_OPEN_ID,
      msg_type: 'post',
      content: JSON.stringify({ zh_cn: { title, content: contentParts } })
    })
  })
}

### 判断逻辑

关键在 `comment.pid`：

---

---

拿到 `pid` 后调用 `getParent(comment)` 获取父评论，里面有 `nick` 字段，就是「回复 @xxx」里要显示的名字。

### 配置存在数据库

飞书的配置项：

---

---

---

---

---

存在 `config` 表，`fetchAdminConfig` 读取时 `FEISHU_APP_SECRET` 返回掩码 `******`，`writeConfig` 写入时跳过掩码保持原值。

PS：[获取OPEN_ID](https%3A%2F%2Fopen.feishu.cn%2Fapi-explorer%2F)点这里，不然找半天你也找不到！

## 邮件通知的逻辑

开启飞书后邮件就多余了：

async function _notifyAdmin (comment, config) {
  if (config.FEISHU_NOTIFY === 'true') return  // 跳过
  // ...邮件逻辑
}

async function _notifyReply (comment, config, getParent) {
  if (config.FEISHU_NOTIFY === 'true') return  // 跳过
  // ...邮件逻辑
}

## 还没做的

---

---

### 结果

![](https://img.easte.cc/posts/images/H1pXbrnGho23tXxC2pVc9CgBnGe.jpg)

整个功能实现不复杂，就是飞书 API 的格式坑比较多了。消息发出去之后如何处理推送通知，那就是另一回事了。

