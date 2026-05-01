# Patoune-IT — Technical Blog

Personal technical blog built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme, hosted on GitHub Pages.

## Prerequisites

Install Hugo Extended (required for SCSS support):

```bash
# Ubuntu/Debian
sudo apt install hugo

# Or download the latest extended binary from GitHub releases:
# https://github.com/gohugoio/hugo/releases
# Pick the hugo_extended_*_linux-amd64.deb package
```

Verify your installation:

```bash
hugo version
# Should display: hugo v0.x.x+extended ...
```

## Run locally

Clone the repo:

```bash
git clone https://github.com/patoune95/patoune.git
cd patoune
```

Start the development server:

```bash
hugo server -D --baseURL http://localhost:1313/
```

The site is available at [http://localhost:1313/](http://localhost:1313/)

> Use `--baseURL http://localhost:1313/` to override the production base URL locally. The `-D` flag includes draft posts.

## Write a new post

The blog is bilingual (French / English). Posts use **Hugo Page Bundles**: each post is a **single folder** in `content/posts/` containing the content files per language and all shared assets.

```
content/posts/
└── YYYY-MM-DD-mon-titre/
    ├── index.fr.md       ← French version
    ├── index.en.md       ← English version (optional)
    ├── cover.jpg         ← cover image (always named "cover")
    └── screenshot.png    ← any other image used in the article
```

Hugo automatically links `index.fr.md` and `index.en.md` as translations of the same post — no extra configuration needed. A post without a translation is simply not shown in the other language.

### 1. Create the post folder and write the French version

```bash
mkdir content/posts/YYYY-MM-DD-mon-titre
# then create index.fr.md inside it
```

### 2. Frontmatter template

```yaml
---
title: "Titre de l'article"
date: YYYY-MM-DDTHH:MM:SS+02:00
draft: false
tags:
  - tag1
  - tag2
cover:
  image: cover.jpg      ← filename relative to the post folder
  alt: "Description de l'image"
---
```

Set `draft: true` while writing, then switch to `draft: false` when ready to publish.

### 3. Reference images in the article

Since images are in the same folder as `index.fr.md`, reference them by filename only:

```markdown
![Description](screenshot.png)
![Schema](diagram.svg)
```

No path prefix needed — Hugo resolves them automatically as page resources.

### 4. Add a translation

To add an English version, create `index.en.md` in the same folder with the translated content:

```
content/posts/2026-05-01-mon-titre/
  ├── index.fr.md    ← French
  ├── index.en.md    ← English (same images, no duplication)
  └── cover.jpg
```

Hugo detects the translations automatically. A language toggle appears in the post header.

### Site-wide assets

Images shared across the entire site (logo, avatar) go in `static/assets/images/` and are served at `/assets/images/`. Post-specific images always go inside the post's own folder.

## Deploy

Deployment is automatic: push to `main` and GitHub Actions builds and publishes the site to GitHub Pages.
