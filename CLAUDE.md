# folio

Personal portfolio site for Gowtham S. Static site built with Eleventy (11ty).

## Stack

- **Eleventy 3.x** — static site generator, input: `.`, output: `_site/`
- **Nunjucks** — template engine for layouts (`_includes/`)
- **Vanilla HTML/CSS/JS** — no framework, no bundler
- **Mermaid.js** — diagram rendering via CDN in `_includes/base.njk`
- **GitHub Actions** — builds and deploys to GitHub Pages on push to `main`

## Structure

```
index.html          ← portfolio homepage (plain HTML, not a template)
style.css           ← global styles + blog styles
script.js           ← nav toggle, scroll behavior
_includes/
  base.njk          ← shared shell: nav, head, Mermaid script
  post.njk          ← individual post layout (extends base.njk)
blog/
  index.njk         ← /blog listing, queries collections.post
posts/
  *.md              ← blog posts (see posts/CLAUDE.md for schema)
asset/
  resume.pdf
.github/workflows/
  deploy.yml        ← CI: npm ci → eleventy → deploy-pages
```

## Commands

```bash
npm install       # first time
npm start         # dev server at localhost:8080 with live reload
npm run build     # build to _site/
```

## Eleventy config (`.eleventy.js`)

- Passthrough copy: `style.css`, `script.js`, `asset/`, `CNAME`
- Custom filter: `readableDate` — formats JS Date to "Month DD, YYYY"
- `markdownTemplateEngine: "njk"` — Nunjucks runs inside `.md` files

## Deploy

Push to `main` → GitHub Actions builds `_site/` → deploys via `actions/deploy-pages`.
GitHub Pages source must be set to **GitHub Actions** (not branch) in repo settings.

## Adding content

- **New blog post**: add `.md` to `posts/` — see `posts/CLAUDE.md`
- **New portfolio section**: edit `index.html` directly
- **Nav links**: edit both `index.html` and `_includes/base.njk`
