# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal blog built with Hexo (v7.3.0), a Node.js static site generator. The blog focuses on web development, DevOps, startups, machine learning, artificial intelligence, and engineering management topics.

## Architecture

- **Framework**: Hexo static site generator (v7.3.0)
- **Theme**: Custom "readme" theme located in `themes/readme/`
- **Content**: Blog posts in Markdown format in `source/_posts/`
- **Template Engine**: EJS for theme templating
- **Content Structure**:
  - Posts support front matter with title, date, categories, tags
  - Posts can be hidden with `hidden: true` front matter
  - Images stored in `source/images/`
  - Static pages in `source/` (e.g., about page, 404)

## Common Commands

### Development
```bash
npm install            # Install dependencies (yarn not used)
npm run start          # Start development server (hexo serve)
npm run hexo           # Run hexo CLI directly
```

### Content Creation
```bash
npx hexo new post "Post Title"     # Create new blog post
npx hexo new page "page-name"      # Create new page
npx hexo generate                  # Generate static files
npx hexo clean                     # Clean generated files
```

### Deployment
```bash
npx hexo generate      # Build static site to public/ directory
```

## Theme Structure

The custom "readme" theme follows standard Hexo theme conventions:
- `layout/` - EJS templates for different page types
- `layout/_partial/` - Reusable template components
- `layout/_widget/` - Sidebar widgets
- `_config.yml` - Theme-specific configuration

## Configuration

- Main site config: `_config.yml` (root)
- Theme config: `themes/readme/_config.yml`
- Site generates RSS feed, sitemap, and robots.txt
- Google Analytics integration (UA-111580819-1)
- Posts display 5 per page on index

## Content Guidelines

- Posts use categories like "devops", "book-review"
- Common tags include kubernetes, management, engineering, AI
- Technical posts often focus on practical implementations
- Posts can include custom CSS classes and HTML elements