---
title: Hello World - 欢迎来到我的博客
date: 2025-12-31 12:00:00
tags: [SvelteKit, Svelte 5, 博客]
category: 随笔
draft: false
description: 这是我的第一篇博客文章，介绍一下这个基于 SvelteKit 5 构建的博客系统。
---

## 你好，世界！

第一篇总要从 Hello World 落笔。记不清第几个博客了，但这套仪式感还是得有。

## 怎么来的

上一个用 Svelte 4 搭的，两年了，仓库里躺着几十篇旧文。新年想换新，Svelte 5 发布了，干脆重写。

三月就有这念头，一直拖着没动。年底终于腾出几天搞定。好饭不怕晚，总算落地了。

目前能正常用，还有细节要调，先上线再说。博客是写给自己的，舒服就行。

## 技术栈

SvelteKit 2 + Svelte 5（Runes 模式），TypeScript 做类型检查。MDSvex 处理 Markdown，支持 GFM、数学公式和 Emoji。代码高亮用 Prism.js，可以切主题。RSS 通过 `+server.ts` 实现。部署在 EdgeOne Pages。

样式没用 Tailwind，纯 CSS 加 CSS 变量，够直观。

## 功能

明暗主题跟随系统，也可以手动切。图片在构建时处理，直接复制到 static 目录。搜索用的 Fuse.js。支持 RSS 分类订阅。PWA 配置了 manifest，离线能看。

评论系统是自己写的，基于 Cloudflare Workers + D1，支持回复、点赞、表情、头像。管理面板可以审核、配置项。代码高亮用 Prism.js，多主题可切换。

## 评论

评论后端跑在 Cloudflare Workers 上，数据存 D1。全是手写的，没用 Twikoo 或别的库。接了 Akismet 垃圾检测，也支持人工审核模式。

前端在 `src/lib/components/comment/`，评论列表用树形结构，回复嵌套显示。有管理面板，可以看所有评论、置顶、删评论。表情、头像、UA 所在地都能显示。

具体实现改天再写。

## 音乐

接口用 [NeteaseCloudMusicApi](https://github.com/Binaryify/NeteaseCloudMusicApi)，官方不支持 Cloudflare Workers，改了改才跑起来。只留了核心 API，后台配 token 拿每日推荐。

Workers 存 KV 里做简单缓存。

## 部署

前端走 EdgeOne Pages，GitHub Actions 构建后自动部署。后端跑 Cloudflare Workers，Wrangler 部署。Cloudflare 全家桶，无限流量、全球 CDN。

## 最后

旧博客数据不导入了，算是和以前做个了断。往后写技术笔记、随笔，还有拍的照片。没有固定更新节奏，也不打算讨好谁，就这样。