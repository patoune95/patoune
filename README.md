# Patoune — Technical Blog

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
git clone https://github.com/heka95/patoune.git
cd patoune
```

Start the development server:

```bash
hugo server -D
```

The site is available at [http://localhost:1313/patoune/](http://localhost:1313/patoune/)

> The `-D` flag includes draft posts. Remove it to preview only published content.

## Write a new post

```bash
hugo new posts/YYYY-MM-DD-my-post-title.md
```

Edit the generated file in `content/posts/`, then set `draft: false` when ready to publish.

## Deploy

Deployment is automatic: push to `main` and GitHub Actions builds and publishes the site to GitHub Pages.
