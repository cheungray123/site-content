---
title: "从飞书到博客：基于 Cloudflare Workers 的全自动私有文章同步系统"
date: 2026-04-20T02:30:00.000Z
tags: ["自动化", "飞书", "Cloudflare", "技术分享"]
category: "折腾"
description: "利用 Cloudflare Workers + R2 图床，打造一个免运维、极速、低成本的飞书文章自动同步解决方案。支持文章和相册双模式同步，从富文本解析到图片管线，手把手教你搭建完整的自动化发布流程。"
draft: false
---

写博客这事，谁写谁知道。最烦的不是写，是发布。

飞书写完一篇，复制到 Markdown 编辑器，格式十有八九要乱。图片更是噩梦——得一张张保存、上传图床、替换链接。改一次文章，这流程就得重来一遍。

后来我写了这么个东西：一个跑在 Cloudflare Workers 上的同步服务。现在流程变成了这样：

> 飞书写完 -> 给机器人发句"同步" -> 喝口水 -> 博客已经更新了。

## 功能

这是个能用的自动化系统，不是玩具：

- **双模式同步**：支持文章（posts）和相册（photos）两种文档类型
- **指令驱动**：在飞书私聊里发指令，机器人会回复同步结果
- **富文本解析**：支持多级标题、加粗斜体删除线、代码、链接、@人、引用、待办，还有嵌套表格
- **增量更新**：同步时会对比飞书文档修改时间和 GitHub 提交时间，没改动的文章直接跳过
- **图片处理**：自动下载图片、缩放、上传 R2、替换链接（GitHub 为备选）
- **元数据**：在飞书文档开头写两行字，就能自动生成 Frontmatter，支持分类、标签、标题、地点
- **删除同步**：删了飞书文档，发个指令就能删除 GitHub 里的对应文章

## 设计

用 Cloudflare Workers 的原因很简单：免服务器、速度快、成本低。Workers 和 R2 在同一个内网，图片上传不需要转 Base64，几十张图几秒钟就传完了。

### 架构

```
飞书文档 -> 飞书机器人 -> Webhook -> Cloudflare Workers
                                                        |
                                                        |- 鉴权
                                                        |- 拉取 Block 数据
                                                        |- 解析成 Markdown
                                                        |- 下载图片
                                                        |- 上传 R2 / GitHub
                                                        |- 拼接 Frontmatter
                                                        |- 推送 GitHub
                                                        |- 触发 Actions
                                                        v
                                              博客更新
```

### Block 解析

飞书文档是树状结构，段落里可能嵌套加粗、链接，列表里可能嵌套子列表。代码里用的是递归下降解析器，不是正则匹配。

处理表格时，读取 `column_size` 按列数遍历子节点，提取内容拼成 Markdown，然后直接 return 阻断递归，避免重复渲染。列表也是类似的思路——增加缩进级别，递归处理子节点。

### 增量更新

全量同步时，判断是否更新很简单：

- 取飞书文档的 `updated_at`
- 取 GitHub 上该文件的最后一次 commit 时间
- 如果 GitHub 时间 >= 飞书时间，跳过

这样做还有个好处：如果直接在 GitHub 上改了个错别字，再触发飞书同步，代码会发现 GitHub 更新了，自动跳过那篇，不会覆盖你的修改。

### 相册模式

相册文档不生成 Markdown 内容，而是提取文档中以图片 URL 开头的内容行作为图片列表。图片下载后直接上传到 R2 的 `photos/{slug}/` 目录，生成的文件结构：

```
photos/{slug}/
  ├── index.md   # 元数据和图片列表
  └── 0.jpg     # 实际图片文件
```

## 部署

### 准备工作

1. 飞书企业管理员权限
2. GitHub 账号（需要两个仓库：一个放文章 Markdown，一个放前端代码）
3. Cloudflare 账号

### 配置飞书

1. 在[飞书开放平台](https://open.feishu.cn/app)创建企业自建应用，记下 App ID 和 App Secret
2. 添加"机器人"能力
3. 开通权限：`docx:document:readonly`、`drive:drive`、`wiki:wiki:readonly`、`im:message`、`im:message:send_as_bot`（需要管理员审批）
4. 配置事件订阅，添加 `im.message.receive_v1`，URL 先填占位符
5. 发布应用

### 配置 GitHub

生成一个 Personal Access Token (Classic)，勾上 `repo` 和 `workflow` 权限。

### 部署 Workers

1. 安装 Wrangler CLI
2. 写 `wrangler.toml`：

```toml
name = "feishu-sync-bot"
main = "src/index.js"
compatibility_date = "2024-02-20"

[[r2_buckets]]
binding = "IMG_BUCKET"
bucket_name = "你的R2桶名字"

[[kv_namespaces]]
binding = "PENDING_KV"
id = "你的KV命名空间ID"
```

3. 配置环境变量（用 `wrangler secret put`）：
   - `FEISHU_APP_ID`
   - `FEISHU_APP_SECRET`
   - `GITHUB_PAT`
   - `FEISHU_WIKI_SPACE_ID` 或 `FEISHU_FOLDER_ID`（文章仓库，二选一）
   - `FEISHU_PHOTOS_FOLDER_ID`（相册仓库，没有则不配置）
   - `R2_PUBLIC_URL`
   - `FEISHU_SIGNING_SECRET`（建议配置）
   - `GITHUB_CLIENT_BRANCH`（可选，客户端分支，默认 main）
4. `wrangler deploy`
5. 把飞书后台事件订阅的 URL 改成 `https://<你的Worker域名>/webhook/feishu`

## 使用

### 指令

在飞书找到机器人，发送：

- `同步` 或 `sync`：全量同步文章和相册
- `同步 <关键词>`：搜索并同步单篇文章
- `同步相册 <关键词>`：搜索并同步单个相册
- `删除 <文章标题>`：删除文章（需要回复 Y 确认）
- `删除 <document_id>`：直接删除

### 元数据

在文档最开头写：

```
分类: 随笔
标签: 生活, 技术分享
草稿: true
# 标题
正文...
```

支持的元数据字段：
- `分类`：文章分类
- `标签`：逗号分隔的标签列表
- `草稿：`设为 `true` 则生成 `draft: true`
- `title`：手动指定文章标题
- `地点`：用于相册的地点信息

标题可以不写，程序会自动用正文里第一个 `#` 后面的内容。标签用英文逗号分隔。

## FAQ

**发"同步"没反应？**
检查飞书权限是否审批通过、应用是否发布、Webhook URL 是否配置正确。

**一直显示"跳过"？**
说明 GitHub 上的提交时间晚于飞书文档的修改时间。如果确定改了没同步，用单篇强制同步。

**R2 配置失败？**
`wrangler.toml` 里 `bucket_name` 只填 R2 的纯名字；`R2_PUBLIC_URL` 才填访问域名。图片会优先上传 R2，如果 R2 不可用则自动 fallback 到 GitHub。

**相册和文章有什么区别？**
文章同步到 `posts/` 目录，会解析富文本内容生成完整 Markdown。相册同步到 `photos/` 目录，仅提取图片 URL，不生成正文内容。

---

这套东西让我从繁琐的发布流程里脱了身。现在只管写，脏活累活都交给代码。
