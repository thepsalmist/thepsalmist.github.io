# xodhiambo.com — terminal edition

Jekyll site for GitHub Pages. Terminal aesthetic, amber phosphor accent,
blog at `/blog/`.

## Structure

```
_config.yml                  site config, permalinks, plugins
Gemfile                      pinned to github-pages gem
index.html                   homepage (uses default layout)
blog/index.html              writing.log — post listing + planned titles
_posts/                      blog posts (Markdown, YYYY-MM-DD-slug.md)
_layouts/default.html        status bar + footer chrome
_layouts/post.html           article layout (mono chrome, sans body)
_includes/head.html          meta, ogTitle support, fonts, SEO tags
assets/css/terminal.css      the whole theme
```

## Local preview

```bash
gem install bundler
bundle install
bundle exec jekyll serve   # http://localhost:4000
```

## Publish the MikroTik article

1. Open `_posts/2026-06-15-nordvpn-mikrotik-ikev2-setup.md`
2. Fix the date prefix in the filename to the real publish date
3. Paste the article body below the frontmatter (replace the placeholder)
4. Optionally set `ogTitle` for a different social-card title
5. Commit and push — GitHub Pages builds automatically

The post URL is controlled by the filename slug:
`_posts/2026-06-20-nordvpn-mikrotik-ikev2-setup.md`
→ `https://xodhiambo.com/blog/nordvpn-mikrotik-ikev2-setup/`

## Adding a new post

Drop a new `YYYY-MM-DD-slug.md` in `_posts/` with:

```yaml
---
layout: post
title: "Post Title"
description: "Meta description for search."
tags: [tag1, tag2]
---
```

Then remove the matching `[PLANNED]` line from `blog/index.html`
and `index.html`.

## Deploy checklist (GitHub Pages)

- Repo settings → Pages → build from branch (or Actions workflow)
- Custom domain: xodhiambo.com with CNAME file (add `CNAME` containing
  `xodhiambo.com` at repo root if not configured in settings)
- `jekyll-feed` serves RSS at /blog/feed.xml
- `jekyll-sitemap` serves /sitemap.xml — submit to Google Search Console

## Still TODO (content)

- Confirm `/assets/resume.pdf` (currently Xavier_General_Resume_20260629)
  is the resume to ship
