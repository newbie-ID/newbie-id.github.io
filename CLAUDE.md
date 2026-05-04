# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Hugo static site blog for **Chewy_iD's Blog**, deployed to GitHub Pages at `https://newbie-id.github.io/`. Uses the **Blowfish** theme (git submodule) with Chinese (zh-cn) as the primary language.

## Commands

```bash
# Local dev server with live reload
hugo server

# Production build (output to public/)
hugo --minify

# Create a new blog post
hugo new posts/my-post-name.md
```

Hugo extended version is required (for Sass/SCSS support).

## Architecture

### Configuration

All config lives in `config/_default/` as separate TOML files (not a single hugo.toml):
- **hugo.toml** — base URL, taxonomies (tags/categories/authors/series), pagination, outputs
- **params.toml** — theme behavior: color scheme, homepage layout (hero), article features (TOC, breadcrumbs, reading time), footer/header settings
- **languages.zh-cn.toml** — site title, author profile, date format, logo
- **menus.zh-cn.toml** — main navigation (Blog, Tags)
- **markup.toml** — Goldmark renderer (unsafe enabled), KaTeX math delimiters, code highlighting, TOC levels 2-4
- **module.toml** — Hugo module imports (Blowfish theme)

### Content

- `content/posts/` — blog articles (front matter: title, date, tags, categories, summary, draft)
- `content/pages/` — static pages (about page)
- `content/_index.md` — homepage

### Theme

Blowfish theme is at `themes/blowfish/` as a **git submodule**. Do not modify files inside `themes/blowfish/`. Override theme templates by creating files in `layouts/`, and add custom CSS/JS in `assets/`.

### Static Assets

`static/` holds images (`img/`), favicons, and PWA icons. Referenced directly by URL path (e.g., `/img/background.svg`).

### i18n

Custom translation overrides go in `i18n/zh-CN.yaml`.

### Deployment

GitHub Actions workflow at `.github/workflows/gh-pages.yml`: on push to `main`, builds with `hugo --minify` and deploys `public/` via `peaceiris/actions-gh-pages` to the `gh-pages` branch.
