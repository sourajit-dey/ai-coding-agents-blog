# AI Coding Agents Blog — Astro site

Free Astro site, built to publish on GitHub Pages at no cost.

## Status
- Built and verified locally on 2026-10-24 (see `dist/` after `npm run build`).
- NOT live yet. It becomes live only after you create a GitHub repo and publish it (steps below).
- No domain purchase needed. GitHub Pages gives a free `username.github.io` address. A custom domain is optional later.

## Local preview
```bash
cd blog-site
npm install
npm run dev
```

## Build
```bash
npm run build
```
Output goes to `dist/`.

## Publish on GitHub Pages (free)
Live URL for this blog: https://sourajit-dey.github.io/ai-coding-agents-blog/
Repo: https://github.com/sourajit-dey/ai-coding-agents-blog
1. Upload the contents of this `blog-site/` folder to that repo (not the folder itself — `package.json`, `astro.config.mjs`, `src/`, `public/`, `.github/` at the repo root).
2. In the repo: Settings → Pages → Build and deployment → Source: GitHub Actions. The included `.github/workflows/deploy.yml` builds on every push to `main` and deploys `dist/`.
3. Wait for the Actions run to finish, then open https://sourajit-dey.github.io/ai-coding-agents-blog/
4. After it is live, add the site to Google Search Console and submit `https://sourajit-dey.github.io/ai-coding-agents-blog/sitemap.xml`. That is when ranking and tracking can start.

## Content
- Article 1: `src/content/blog/claude-code-mcp-setup-fix.md` — Claude Code MCP setup and fix, with the MCP config generator widget.
- Cluster plan: `../blog-seo/cluster-ai-coding-agents.md` (in the workspace, not published as a page).
- New posts go in `src/content/blog/` as Markdown with title, description, date, and slug frontmatter. The homepage lists them automatically.
