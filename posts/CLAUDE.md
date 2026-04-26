# posts/

Each `.md` file here becomes a blog post at `/posts/<slug>/`.

## Required frontmatter

```yaml
---
title: "Post Title"
description: "One-sentence summary shown on /blog listing."
date: 2026-04-26
tags: post
layout: post.njk
---
```

| Field         | Required | Notes                                              |
|---------------|----------|----------------------------------------------------|
| `title`       | yes      | Shown as page `<h1>` and listing card heading      |
| `description` | yes      | Shown on /blog listing card                        |
| `date`        | yes      | Controls sort order (newest first on listing)      |
| `tags`        | yes      | Must include `post` — adds to `collections.post`   |
| `layout`      | yes      | Always `post.njk`                                  |

## Mermaid diagrams

Use raw HTML in the markdown body (fenced blocks won't render):

```html
<pre class="mermaid">
flowchart LR
  A --> B --> C
</pre>
```

## Slug

Eleventy uses the filename as the slug. `my-post.md` → `/posts/my-post/`.
Use kebab-case filenames.
