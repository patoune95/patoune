# Patoune Blog

Hugo blog hosted on GitHub Pages at `https://heka95.github.io/patoune/`.

## Stack

- Hugo Extended + custom theme based on PaperMod (embedded directly at `themes/PaperMod`, no submodule)
- GitHub Actions build/deploy (`.github/workflows/hugo.yml`)
- Config: `hugo.toml`

## Key paths

- Posts: `content/posts/YYYY-MM-DD-title.md`
- Images: `static/assets/images/` → served at `/assets/images/`
- Favicon: `static/favicon.ico`

## Local dev

```bash
hugo server -D
```

## Pending

- Theme is a copy of PaperMod to be customised together — it is NOT a submodule.
- GitHub Pages source must be set to **GitHub Actions** in repo Settings → Pages.
