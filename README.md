# blog.ogenki.io

Source for **[blog.ogenki.io](https://blog.ogenki.io)** — a personal blog covering Cloud, security, DevOps, Kubernetes, and platform engineering. Posts are bilingual (English and French).

Built with [Hugo](https://gohugo.io) and the [hugo-clarity](https://github.com/chipzoller/hugo-clarity) theme, deployed to GitHub Pages via GitHub Actions.

## Local development

Hugo **extended** is required — the exact version is pinned in `.mise.toml`.

The theme is a git submodule, so clone with it (a plain `git clone` leaves
`themes/hugo-clarity/` empty and the build fails):

```bash
git clone --recurse-submodules https://github.com/Smana/smana.github.io.git
# already cloned? git submodule update --init
```

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

## Images

Posts use page bundles: images live next to the `index.md` that references them.
Insert them with the `img` shortcode rather than raw Markdown, so they get
resized and served as WebP:

```
{{< img src="diagram.png" alt="Architecture overview" width="900" >}}
```

Resizing and WebP encoding happen at build time (`layouts/shortcodes/img.html`
for in-content images, `layouts/partials/figure.html` for thumbnails). The
generated derivatives go to `resources/_gen`, which is gitignored and cached in
CI — commit only the original image.

## Deployment

Every push to `main` builds the site and publishes it to GitHub Pages
(`.github/workflows/gh-pages.yaml`). No manual step.

## Author

Smaine Kahlouch — [@Smana](https://github.com/Smana)
