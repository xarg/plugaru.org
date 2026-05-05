# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal blog built with Hexo (v7.3.0), a Node.js static site generator. Topics: web development, DevOps, startups, ML/AI, engineering management.

## Commands

```bash
npm install                     # Install dependencies (npm, not yarn)
npm run start                   # hexo serve — local dev server with live reload
npx hexo new post "Title"       # Scaffold a post in source/_posts/
npx hexo generate               # Build static site into public/
npx hexo clean                  # Remove public/ and db.json (run before generate if templates/config change)
```

Node version is pinned to **22.22.2** (`.nvmrc`); CI/Netlify use the same.

## Architecture

- **Hexo** consumes `_config.yml` (site) + `themes/readme/_config.yml` (theme) and renders Markdown in `source/_posts/` through EJS templates in `themes/readme/layout/`.
- **Custom theme** lives at `themes/readme/` — it is the only theme and is edited in-tree (not an npm dep). `layout/_partial/` holds reusable fragments (header, footer, article, etc.); `layout/_widget/` holds sidebar widgets enabled via `widgets:` in the theme config.
- **Permalinks** are `:year/:month/:day/:title/` — changing a post's date or filename will break inbound links.
- **Hidden posts**: front matter `hidden: true` (via `hexo-hide-posts`) excludes a post from the index but still renders it; by default tag/category/archive/sitemap/feed generators also skip hidden posts (see `hide_posts.public_generators` in `_config.yml`).
- **Output**: `public/` is the build artifact (gitignored). `db.json` is Hexo's cache and is also gitignored despite being present locally.

## Deployment

Deployed via **Netlify** (`netlify.toml`): build runs `npm install && hexo generate`, publishes `public/`. There is no `hexo deploy` target configured — pushing to the default branch is the deploy mechanism.

## Content Conventions

- Front matter: `title`, `date`, `categories`, `tags`. Existing categories include `devops` and `book-review`; common tags: kubernetes, management, engineering, AI.
- Posts may embed raw HTML and custom CSS classes — Markdown rendering is via `hexo-renderer-marked` with `breaks: false`, so single newlines do **not** become `<br>`.
- Images live in `source/images/` and are referenced by absolute path (`/images/foo.png`) since `post_asset_folder` is disabled.
