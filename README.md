# blog.ogenki.io

Source for **[blog.ogenki.io](https://blog.ogenki.io)** — a personal blog covering Cloud, security, DevOps, Kubernetes, and platform engineering. Posts are bilingual (English and French).

Built with [Hugo](https://gohugo.io) and the [hugo-clarity](https://github.com/chipzoller/hugo-clarity) theme, deployed to GitHub Pages via GitHub Actions.

## Local development

Hugo extended is required — the exact version is pinned in `.mise.toml`.

```bash
# Start a local dev server with live reload
hugo server

# Build the production site
hugo --minify

# Create a new post
hugo new post/my-post-name/index.md
```

## Layout

- `content/en/post/` — English posts
- `content/fr/post/` — French posts (translations)
- `content/*/post/series/` — Multi-part series (e.g. `agentic_ai/`, `observability/`)
- `themes/hugo-clarity/` — Theme (git submodule)
- `config/_default/` — Hugo configuration split across `config.toml`, `params.toml`, `languages.toml`, `markup.toml`
- `layouts/` — Site-specific overrides on top of the theme

See [`CLAUDE.md`](./CLAUDE.md) for the full content and front-matter conventions.

## Author

Smaine Kahlouch — [@Smana](https://github.com/Smana)
