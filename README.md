# 我的本地博客

基于 [Minrock](https://github.com/rnt-rez/minrock) 模板，使用 npm 当前稳定版 **Astro 7.3.5**，运行环境为 **Node.js 24.21.0**。保留模板的排版、四种配色、文章目录、站内搜索、标签、图片放大和 RSS。

## 启动与停止

在终端运行：

```bash
cd /Users/gexiaoyu/code/astro
nvm use
npm run dev
```

浏览器打开 <http://localhost:4321>。保存内容后页面自动更新。普通终端以前台运行时，按 `Ctrl+C` 停止；本次由 Codex 启动的服务在后台运行，用 `npm run dev -- stop` 停止，用 `npm run dev -- status` 查看状态。

如果提示 `nvm: command not found`，先执行 `source ~/.nvm/nvm.sh`。如果换了电脑，先安装 nvm，然后运行 `nvm install` 和 `npm ci`。

## 写一篇文章

新建 `src/content/blog/my-first-post/index.md`。文件夹名决定文章地址 `/blog/my-first-post/`，建议使用英文、数字与短横线。

```markdown
---
title: "我的第一篇文章"
description: "用一句话介绍这篇文章。"
pubDate: 2026-10-03
tags: ["笔记", "生活"]
draft: false
---

## 今天学到了什么

正文从这里开始。支持普通 Markdown、代码块、图片和列表。
```

- `title`、`description`、`pubDate` 必填，日期格式为 `YYYY-MM-DD`。
- `tags` 是标签列表，相关标签页自动生成。
- `draft: true` 隐藏文章，包括本地开发预览、搜索、RSS 和标签。需要预览时暂时改成 `false`；保存本地文件不会自动公开到互联网。
- `updatedDate` 可选，用于记录更新日期。
- `image: "/images/cover.jpg"` 可选，用于显示文章封面。
- `pubDate` 用于排序，未来日期不会自动延迟发布。

现成例子：`src/content/blog/welcome/index.md`、`src/content/blog/writing-guide/index.md`。可以修改或删除这两篇示例文章。`draft-example/index.md` 展示隐藏草稿。

图片可以放在 `public/images/`，在正文使用 `![说明](/images/photo.jpg)`；也可以放在文章文件夹旁，用 `![说明](./photo.jpg)` 引用。路径必须指向实际存在的图片。

## 修改博客信息

编辑 `src/config/site.ts`：

| 设置 | 用途 |
| --- | --- |
| `title` | 博客名称，当前为“我的博客” |
| `author` | 作者名称，当前为“博主” |
| `tagline`、`description` | 首页介绍及网站描述 |
| `socialLinks` | 填入自己的 `github`、`linkedin`、`email`，未配置的链接不显示 |
| `defaultTheme` | 默认配色：`white`、`cream`、`slate`、`midnight` |
| `navLinks` | 顶部导航 |
| `features` | 开关搜索、目录、标签等功能 |

浏览器右上角可切换配色；浏览器保存的配色优先于默认设置。按 `⌘K`（Mac）或 `Ctrl+K`（Windows/Linux）打开站内搜索，搜索标题、简介与标签。

中文字体使用 [寒蝉端黑宋 v1.2](https://github.com/Warren2060/ChillDuanHeiSong/releases/tag/v1.200) 标准版，正文为 Regular、标题为 Bold。字体以 WOFF2 格式保存在 `public/fonts/`，保留完整字符集，无需系统安装；汉字及中文标点使用此字体，英文和代码中的西文保持原来的系统字体/等宽字体。样式在 `src/styles/global.css`，字体许可保存在 `public/fonts/OFL-ChillDuanHeiSong.txt`。首次打开需等待本地字体加载，期间显示后备字体。

“关于我”正文：`src/pages/about.astro`。首页正文：`src/pages/index.astro`。全局样式：`src/styles/global.css`。

项目内容放在 `src/content/projects/<项目名>/index.md`，参考 `my-blog/index.md`；项目的日期字段叫 `date`。原模板的示例文章和项目保存在 `examples/minrock/`，不参与网站构建；原版文档在该目录的 `README.md`。

演示评论和语音朗读已关闭。以后需要真实评论时，应使用自己的 GitHub Discussions 与 ScatterLeaf 配置；原版文档包含接入方式。

## 使用 Obsidian（可选）

把 `src/content/` 作为仓库打开，直接编辑 Markdown。推荐关闭“使用 Wiki 链接”，使用标准 Markdown 链接；新附件存放位置设为“当前文件所在文件夹”。本项目没有安装模板附带的第三方 Obsidian 插件，写作不依赖插件。

## 检查与构建

```bash
npm run check    # Astro / TypeScript 检查
npm run build    # 生成静态站点到 dist/
npm run qa       # 检查 + 构建
npm run preview # 预览构建结果，保存文章后需要重新构建
```

目前仅在本地使用。以后部署时，复制 `.env.example` 为 `.env`，把 `SITE_URL` 改成自己的正式域名，再执行 `npm run build`。该配置会同时用于 canonical、RSS 和 sitemap。线上 `public/robots.txt` 可添加自己域名的 `Sitemap` 地址。

更新 Astro 可运行 `npm install --save-exact astro@latest`，然后执行 `npm run qa`。依赖版本由 `package-lock.json` 锁定，其他机器用 `npm ci` 复现。

## 本次验证记录

- Astro 稳定版版本号通过 `npm view astro version` 实时确认。
- 已安装 Node.js 24.21.0，避免当前依赖 `undici` 对较旧 Node.js 的兼容警告。
- `npm run qa` 已通过：0 错误、0 警告；静态输出的文章、草稿过滤、RSS、sitemap、站内链接和图片路径均已检查。
- 已执行兼容依赖修复。`npm audit` 仍报告 Astro 的上游 `http-cache-semantics` 问题（2 条关联高危告警）；当前未提供保持 Astro 7 的修复方案。不要执行建议的 `npm audit fix --force`，它会降级到 Astro 2。后续关注上游补丁。

Minrock 的 MIT 许可保留在 `LICENSE`，页脚保留模板署名。
