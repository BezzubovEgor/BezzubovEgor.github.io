# yahor.dev

Personal blog built with [Astro](https://astro.build), deployed to GitHub Pages.

## Local development

```bash
npm install
npm run dev       # starts dev server at localhost:4321
npm run build     # builds to ./dist
npm run preview   # preview production build
```

## Writing posts

Add Markdown files to `src/content/blog/`:

```markdown
---
title: "Your Post Title"
description: "A short description for the post list and SEO."
pubDate: 2026-04-13
tags: ["typescript", "ai-agents"]
---

Your content here...
```

## Deploy

Pushes to `main` auto-deploy via GitHub Actions → GitHub Pages.

## Stack

- Astro 4.x
- Markdown/MDX
- GitHub Pages
- Zero JavaScript shipped to client
