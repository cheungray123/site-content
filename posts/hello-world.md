---
title: Hello World - 欢迎来到我的博客
date: 2026-1-1 12:00:00
tags: [SvelteKit, Svelte 5, 博客]
category: 随笔
draft: false
description: 这是我的第一篇博客文章，介绍一下这个基于 SvelteKit 5 构建的博客系统。
---

## 你好，世界！

第一篇文章，总要从 Hello World 落笔，这篇也不例外。早已记不清这是我搭建的第几个博客了，但这份始于 Hello World 的仪式感，始终不能丢。

## 这个博客是怎么来的？

我的上一个博客，是用 **Svelte 4** 搭建的，一晃两年过去，仓库里还静静躺着几十篇随性随笔。新年本就想开启新征程，恰逢 **Svelte 5** 发布，博客升级的念头也愈发强烈，干脆彻底重写一版。
其实这个想法从今年3月就萌生了，只是平日里琐事缠身，难得的闲暇时光总想放空放松，便一拖再拖，直到年底才腾出几天时间动工。虽说好饭不怕晚，但总算赶在年末落地了。

目前用 Svelte 5 重写的版本已经可以正常使用，尽管还有不少细节有待优化，但先上线再说。毕竟博客本就是写给自己的自留地，不用苛求完美，舒心自在才最重要，后续再慢慢打磨完善就好。

### 技术栈

| 技术 | 版本 | 用途 |
|------|------|------|
| SvelteKit | 2.50.2 | 全栈框架 |
| Svelte | 5.51.0 | UI 框架（Runes API） |
| TypeScript | 5.9.3 | 类型安全 |
| Tailwind | 4.2.2 | 原子化 CSS |
| MDSvex | 0.12.6 | Markdown 支持 |
| Shiki | 4.0.2 | 代码高亮 |

### 核心功能

- **明暗主题切换** - 跟随系统或手动切换
- **Markdown 写作** - 支持 GFM、数学公式、Emoji
- **代码高亮** - Shiki 双主题，支持 diff、focus 标注
- **图片优化** - 自动转换为 WebP/AVIF 格式
- **RSS 订阅** - 支持分类和标签订阅
- **PWA 支持** - 可离线访问

## 图片优化

博客的相册功能使用 `@sveltejs/enhanced-img`，自动将图片转换为 WebP 和 AVIF 格式：

| 格式 | 压缩率 | 浏览器支持 |
|------|--------|-----------|
| AVIF | 最高 | 现代浏览器 |
| WebP | 高 | 广泛支持 |
| JPEG/PNG | 原始 | 兜底支持 |

## 评论系统

评论系统我使用的是Twikoo Cloudflare版本 这个版本不需要自己搭建服务器，只需要一个Cloudflare账号就可以使用，而且评论数据是存储在Cloudflare的KV中，非常安全。而且冷启动非常快。

对比官方的库我添加了缓存，优化了sql语句，改了点bug，目前使用非常稳定，欢迎大家使用。[仓库地址](https://github.com/cheungray123/twikoo-cloudflare)

## 音乐api 
音乐api使用的是[NeteaseCloudMusicApi](https://github.com/Binaryify/NeteaseCloudMusicApi) 这个库是开源的，但是需要自己搭建服务器，我使用的是Cloudflare Workers 部署的。 这个库本身并不支持Cloudflare Workers，但是可以通过一些修改来支持。我把ui全部干掉，只保留了核心api部分。只需要后台添加token就可以获取首页的每日推荐歌单，目前使用非常稳定。

## 部署

既然前面已经使用了Cloudflare，那么部署自然也是使用Cloudflare了，Cloudflare 全家桶成就以达成！无限流量，全球CDN，免费使用，而且速度飞快，部署也方便。对比vercel和netlify每个月只有100g的流量，Cloudflare简直是大善人！当然Cloudflare 也有很多技术限制。

## 写在最后

以前的博客数据这里就不导入了，和以前的自己来一场断舍离！往后这里会记录各类技术笔记、随性生活随笔，还有随手拍下的摄影作品，把日常的思考、感悟与美好瞬间悉数珍藏。没有固定的更新节奏，也不刻意迎合谁，只做最真实的表达。欢迎大家常来逛逛，一起在这片小天地里，慢慢记录、慢慢成长。

---

