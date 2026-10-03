---
title: "如何在这个博客里写文章"
description: "添加 Markdown、设置草稿、插入图片，以及预览你的新文章。"
pubDate: 2026-10-02
tags: ["Astro", "写作"]
draft: false
---

## 新建一篇文章

在 `src/content/blog/` 下新建一个文件夹，例如 `my-first-post`，在里面创建 `index.md`：

```markdown
---
title: "我的第一篇文章"
description: "用一句话介绍文章内容。"
pubDate: 2026-10-03
tags: ["笔记"]
draft: false
---

## 今天学到了什么

从这里开始写正文。
```

文章地址由文件夹名决定，例如 `/blog/my-first-post/`。文件夹名建议使用英文、数字和短横线。

## 预览与草稿

运行 `npm run dev` 后，保存文件即可在浏览器里看到更新。

`draft: true` 会隐藏文章，包括开发预览、搜索、标签和 RSS。想预览时暂时设为 `false`，本地保存不会自动发布到互联网。

## 插入图片

把图片放到 `public/images/`，然后在文章中写：

```markdown
![图片说明](/images/my-photo.jpg)
```

请把 `my-photo.jpg` 换成实际存在的文件名。也可以把图片放在文章旁边，使用标准 Markdown 相对路径引用，例如 `![图片说明](./photo.jpg)`。

## 个性化设置

在 `src/config/site.ts` 中修改博客名、作者、介绍、社交链接和默认配色。关于我的正文位于 `src/pages/about.astro`。

## 构建静态网站

```bash
npm run qa
npm run preview
```

构建结果在 `dist/`。当前博客仅在本地运行。以后部署时，把 `SITE_URL` 改成自己的域名，再重新构建。
