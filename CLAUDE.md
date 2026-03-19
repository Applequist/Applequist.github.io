# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev        # Start local development server
npm run build      # Type-check + build for production (astro check && astro build)
npm run preview    # Preview the production build locally
```

## Architecture

This is an **Astro** static site (personal blog at `https://Applequist.github.io`).

**Content collections** are the core abstraction. Blog posts live in `src/content/blog/` as `.md` or `.mdx` files with this frontmatter schema (defined in `src/content/config.ts`):

```yaml
title: string         # required
description: string   # required
pubDate: date         # required
updatedDate: date     # optional
hidden: boolean       # optional, default false — hides from blog listing
```

**Routing:** `src/pages/blog/[...slug].astro` dynamically generates routes from all content collection entries. The blog index at `src/pages/blog/index.astro` lists all non-hidden posts sorted by `pubDate` descending.

**Layouts:** Two layouts exist — `BlogPost.astro` (wraps individual posts) and `Page.astro` (base layout with header/footer, Google Analytics via Partytown). Components in `src/components/` are used across both.

**Site constants** (title, description, social links) are centralized in `src/consts.ts`.

**Deployment** happens automatically via GitHub Actions (`.github/workflows/deploy.yml`) on push to `main`.
