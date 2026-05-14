# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a multilingual Hugo blog (blog.ogenki.io) focused on Cloud, security, DevOps, Kubernetes, and infrastructure topics. Content is available in English and French.

## Build Commands

```bash
# Local development server
hugo server

# Production build (used in CI)
hugo --minify

# Create new post
hugo new post/my-post-name/index.md
```

**Requirements:** Hugo 0.156.0 extended version (pinned via `.mise.toml`)

## Content Structure

- `content/en/post/` - English blog posts
- `content/fr/post/` - French blog posts (translations)
- Posts use page bundles: `post/my-post/index.md` with images in the same directory
- Series are organized under `post/series/` (e.g., observability, workshop_helm)

## Front Matter

Posts use TOML front matter (`+++` delimiters, not YAML `---`). Key fields:
- `draft` - Draft posts aren't published
- `featured` - Featured posts appear on homepage sidebar
- `summary` - Used for post cards and SEO description
- `toc` - Enable table of contents
- `usePageBundles` - Group assets (images) in the same folder as the post
- `categories`, `tags`, `series` - Taxonomies; series values are quoted strings e.g. `series = ["Agentic AI"]`
- `aliases` - URL redirects e.g. `aliases = ["/fr/post/old-slug/"]`
- `codeMaxLines` - Override max lines before code block collapses (global default: 7)

## Shortcodes

Custom shortcodes in `layouts/shortcodes/`:
- `{{< img src="file.png" alt="desc" width="800" caption="optional" >}}` - Image with lazy loading; when `usePageBundles: true`, `src` is relative to the post folder

Theme shortcodes (from `themes/hugo-clarity/`):
- `{{% notice info "Title" %}} ... {{% /notice %}}` - Callout box; types: `info`, `warning`, `tip`, `note`

## Architecture

- **Theme:** hugo-clarity (git submodule in `themes/hugo-clarity`)
- **Config:** Split config in `config/_default/` — `config.toml` (base URL, taxonomies, outputs), `params.toml` (theme behavior), `languages.toml` (EN default/weight 1, FR weight 2), `markup.toml`
- **Menus:** Per-language in `config/_default/menus/`
- **Deployment:** GitHub Actions (`.github/workflows/gh-pages.yaml`) auto-deploys `main` to GitHub Pages using Hugo 0.139.0 extended

## Series

Series group related posts under `post/series/<series-name>/`. Each series has its own `_index.md`. Example series: `observability`, `agentic_ai`, `workshop_helm`, `workshop_kubernetes`. French series live under `content/fr/post/series/`, English under `content/en/post/series/`.

## Writing Blog Posts

When assisting with blog content:
- Analyze and preserve the existing writing tone across posts
- Verify technical claims and accuracy
- Improve logical flow and structure
- Make content engaging and dynamic
- Correct errors in meaning and phrasing
