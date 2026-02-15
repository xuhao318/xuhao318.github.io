# Hao's Blog

Personal Hugo blog deployed to GitHub Pages at https://xuhao318.github.io.

## Commands

```bash
hugo server          # Local dev server at http://localhost:1313/
hugo                 # Build static site to ./public/
hugo new posts/my-new-post.md  # Create new post from archetype
```

## Architecture

```
content/posts/        # Blog posts (Markdown)
themes/hugo-geekblog/ # Theme (git submodule — do not edit directly)
archetypes/default.md # Template for new posts
hugo.toml             # Site configuration
.github/workflows/    # CI/CD (auto-deploys on push to main)
```

## Post Front Matter

```yaml
---
title: "Post Title"
date: 2025-01-17
tags: ["tag1", "tag2"]
authors: ["Hao"]
---
```

## Deployment

Pushing to `main` triggers GitHub Actions (`.github/workflows/hugo.yaml`) which builds with Hugo 0.140.2 extended and deploys to GitHub Pages automatically.

## Gotchas

- Theme is a **git submodule** — checkout with `git clone --recurse-submodules`
- `disablePathToLower = true` in `hugo.toml` — post filenames preserve case
- `unsafe = true` in markdown renderer — required for mermaid diagram shortcodes
