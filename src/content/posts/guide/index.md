---
title: Simple Guides for Mizuki
published: 2025-07-26
description: "How to use this blog template."
image: "./cover.jpeg"
tags: ["Mizuki", "Blogging"]
category: Guides
draft: false
---


This blog template is built with [Astro](https://astro.build/). For the things that are not mentioned in this guide, you may find the answers in the [Astro Docs](https://docs.astro.build/).

## Front-matter of Posts

```yaml
---
title: My First Blog Post
published: 2023-09-09
description: This is the first post of my new Astro blog.
image: ./cover.jpg
tags: [Foo, Bar]
category: Front-end
---
```




| Attribute         | Description                                                                                                                                                                                                 |
|-------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `title`           | 文章标题                                                                                                                                                                                     |
| `published`       | 发布日期                                                                                                                                                                           |
| `pinned`          | 是否粘贴在顶部                                                                                                                                                   |
| `description`     | 文章的简短描述，列在文章开头                                                                                                                                                   |
| `image`           | 文章封面图片url.<br/>1. Start with `http://` or `https://`: Use web image<br/>2. Start with `/`: For image in `public` dir<br/>3. With none of the prefixes: Relative to the markdown file |
| `tags`            | 文章标签                                                                                                                                                                                      |
| `category`        | 文章种类                                                                                                                                                                                   |
| `licenseName`     | The license name for the post content.                                                                                                                                                                      |
| `author`          | 作者                                                                                                                                                                      |
| `sourceLink`      | The source link or reference for the post content.                                                                                                                                                          |
| `draft`           | If this post is still a draft, which won't be displayed.                                                                                                                                                    |


## Where to Place the Post Files


你的文章应该存放在 `src/content/posts/` . 你也可以创建自己的子目录来更好的管理文章和资产。

```
src/content/posts/
├── post-1.md
└── post-2/
    ├── cover.png
    └── index.md
```