# Pauls Education — portfolio & articles site

Built with [Astro](https://astro.build), designed for **Azure Static Web Apps** (free tier).
Portfolio + multi-author technical blog with VS Code-quality syntax highlighting,
a web CMS at `/admin`, and a comments slot powered by GitHub Discussions.

## Run it locally

```bash
npm install
npm run dev        # http://localhost:4321
npm run build      # production build into ./dist
```

## Write an article (the simple way)

Add a markdown file to `src/content/blog/`:

```markdown
---
title: "My article title"
description: "One sentence that appears on cards and LinkedIn previews."
pubDate: 2026-07-10
author: "Paul"
tags: ["copilot", "agents"]
---

Your article here. Code blocks get automatic syntax highlighting:

​```typescript
const hello = "world";
​```
```

Push to `main` → the site rebuilds and deploys automatically.

## Deploy to Azure Static Web Apps

1. Push this folder to a new GitHub repository.
2. In the Azure portal: **Create resource → Static Web App** (Free plan).
3. Connect your GitHub repo. Build settings:
   - **App location:** `/`
   - **Output location:** `dist`
   - Build preset: Astro (or Custom with `npm run build`)
4. Azure creates a GitHub Actions workflow — every push to `main` deploys.
5. Add your custom domain under **Custom domains** (free SSL included).

Also update `site` in `astro.config.mjs` to your real domain, and the
LinkedIn/GitHub/email links in `src/layouts/Base.astro`.

## Enable comments & reactions (Giscus)

Follow the steps at the top of `src/components/Comments.astro` (about 5 minutes).
Readers comment and react with their GitHub accounts — no database, no spam
management, free forever.

## Enable the CMS (`/admin`) — the "SharePoint News" experience

The editor is pre-configured in `public/admin/config.yml`. It needs two things:

1. Your repo name in `config.yml` (`repo: YOUR_GITHUB_USER/YOUR_REPO`).
2. A small OAuth handler so contributors can log in with GitHub.
   Free options:
   - An **Azure Function** OAuth proxy (fits your subscription — search
     "decap cms github oauth azure function" for maintained templates), or
   - A Cloudflare Worker (e.g. the `decap-proxy` template).
   Set its URL as `base_url` in `config.yml`.

Then invite friends as **collaborators** on the GitHub repo. They visit
`yourdomain.com/admin`, log in with GitHub, and get a clean writing
interface — title, description, tags, body, publish button. No code.

> Until you set up the CMS, you (and technical friends) can simply add
> markdown files — the CMS is a convenience layer, not a requirement.

## Where things live

```
src/
  content/blog/       ← articles (markdown)
  pages/              ← home, blog, portfolio, about
  layouts/            ← Base (site chrome) and Post (article template)
  components/         ← PostCard, Comments
  styles/global.css   ← the whole design system
public/
  admin/              ← Decap CMS
staticwebapp.config.json  ← Azure SWA settings
```

## The LinkedIn workflow

1. Publish the article here first (canonical version on your domain).
2. On LinkedIn, write a short native post: hook + 3 takeaways + link to the article.
3. The article's `description` frontmatter becomes the link preview text.
