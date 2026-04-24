# CLAUDE.md — abovethertl-notes repo

This is the publishing source for [notes.abovethertl.com](https://notes.abovethertl.com), the technical reference site that accompanies [Above the RTL](https://www.abovethertl.com) on Substack. Any Claude Code session working in this directory should read this file first.

For the fuller editorial context — voice, tone, series arc, key technical positions, full post history — see the authoritative project context at `/mnt/x/Cloud/GDrive/projects/abovethertl/CLAUDE.md`. That file covers *what* Above the RTL is. This file covers *how to operate the repo*.

---

## Purpose of this repo

This is an Astro static site that builds to `notes.abovethertl.com` via Cloudflare Pages. It contains:

- Technical reference notes that accompany Above the RTL posts (long-form companions with physics, MTBF arithmetic, archetype catalogs, and other content that would bog down a narrative post)
- The site's identity: hero image, typography, navigation, branding

It does *not* contain:

- Narrative posts (those live on Substack at abovethertl.com)
- Drafts in progress (those live in `/mnt/x/Cloud/GDrive/projects/abovethertl/drafts/`)
- The pitch material, research notes unrelated to published content, or anything Real Intent-specific (that lives in `/mnt/x/Cloud/GDrive/projects/ai-for-cdc/`)

---

## Folder structure

```
abovethertl-notes/
├── src/
│   ├── content/blog/        ← reference notes live here (.md files)
│   ├── content.config.ts    ← frontmatter schema
│   ├── pages/               ← index, about, blog index, RSS
│   ├── layouts/             ← BlogPost.astro (the template that renders each note)
│   ├── components/          ← Header, Footer, BaseHead, etc.
│   ├── styles/global.css    ← site-wide typography and layout
│   ├── consts.ts            ← SITE_TITLE and SITE_DESCRIPTION
│   └── assets/              ← images that get Astro-optimized at build time
├── public/                  ← static assets served as-is (hero image, favicon)
├── docs-internal/           ← scaffolding reference (markdown style guide etc.) — not deployed
├── astro.config.mjs
└── package.json
```

---

## Frontmatter schema (src/content.config.ts)

Every markdown file in `src/content/blog/` must start with:

```yaml
---
title: "..."                      # required
description: "..."                # required
pubDate: 2026-04-23               # required
updatedDate: 2026-05-01           # optional
heroImage: '../../assets/foo.png' # optional, path relative to markdown file
---
```

Astro validates this at build time. Missing required fields or wrong types will fail the build.

---

## Publishing workflow

### Moving a draft from Drive into the repo

Drafts live in Drive at `/mnt/x/Cloud/GDrive/projects/abovethertl/drafts/`. When a draft is ready to publish as a reference note:

1. Copy the file into `src/content/blog/` with a hyphenated slug as the filename:
   ```bash
   cp '/mnt/x/Cloud/GDrive/projects/abovethertl/drafts/post_06_design_by_contract.md' \
      src/content/blog/design-by-contract.md
   ```
2. Add frontmatter at the top (title, description, pubDate).
3. Delete any top-level `# title` h1 from the body if it duplicates the frontmatter title (the BlogPost layout renders the frontmatter title as h1 automatically).
4. Test locally: `npm run dev`, then open `http://localhost:4321/blog/<slug>/`.
5. Commit and push:
   ```bash
   git add -A && git commit -m "Add <slug> reference note" && git push
   ```
6. Cloudflare auto-deploys on push. Site updates within ~90 seconds.

### Updating a published note

Edit the file in `src/content/blog/` directly. Commit and push. Do not touch the Drive snapshot — it's archival.

### URL conventions

- Filenames use hyphens, not underscores: `cdc-synchronizer-analysis.md`, not `cdc_synchronizer_analysis.md`.
- The URL becomes `/blog/<filename-without-extension>/`.

---

## Relationship to Drive

Drive (`/mnt/x/Cloud/GDrive/projects/abovethertl/`) is the drafting and metadata layer:

- `CLAUDE.md` — authoritative editorial context (voice, tone, positions, post history)
- `abovethertl_project_instructions.md` — short-form version for the claude.ai Project
- `drafts/` — in-flight posts and notes not yet published
- `published/` — frozen snapshots of published Substack posts
- `_archive/` — superseded files (e.g. pre-repo versions of notes that now live in this repo)

Content flows Drive → repo, never the other direction. Once a reference note is in the repo, the repo is authoritative. The Drive snapshot freezes.

Never edit a Drive file and the corresponding repo file in the same session — the repo is the source of truth for anything that publishes to notes.abovethertl.com.

---

## Local development

```bash
npm install          # once, after cloning
npm run dev          # start dev server at http://localhost:4321
npm run build        # produce dist/ for deploy verification
npm run preview      # preview the built site locally
```

Node 22 LTS. Older versions (20 and below) will warn or fail.

---

## Deployment

Cloudflare Pages watches the `main` branch of `github.com/abovethertl/notes`. Every push triggers an automatic build and deploy. There is nothing to configure manually — `git push` is the entire deployment flow.

Custom domain: `notes.abovethertl.com`. SSL provisioned automatically via Cloudflare.

If Cloudflare builds fail, check:
1. Node version (set `NODE_VERSION=22` env var in Cloudflare project settings if needed)
2. Frontmatter validation errors (missing required field on a new post)
3. Image path errors in frontmatter `heroImage` (path must be relative from the markdown file)

---

## Content conventions

- Prefer SVG over PNG for diagrams when possible (smaller, scales cleanly, no pixelation on high-DPI displays).
- Hero images live in `src/assets/` if they should be Astro-optimized per post, or `public/` for site-level images like the homepage banner.
- Do not reproduce content from other tracks (Real Intent pitch material, Meta-specific methodology writeups) — see the Drive CLAUDE.md for the boundaries on source material.
- Code blocks in markdown use triple-backtick fences with language tags (`verilog`, `systemverilog`, `bash`) for syntax highlighting.
- Tables render via standard markdown pipe syntax; Astro's default styling handles them at the current 960px content width.

---

## Things this repo doesn't need

- A CMS. Markdown is the authoring format.
- A backend. Everything is static.
- Accounts or auth. The site is public.
- Comments. Discussion happens on Substack or LinkedIn, not here.
- Analytics scripts. Cloudflare provides basic traffic stats for free; no third-party analytics are added.

If a future requirement pushes against any of these, reconsider whether this repo is the right place for it, or whether it belongs on Substack.
