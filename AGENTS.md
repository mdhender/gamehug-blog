# AGENTS.md

## Project Overview

This is the Hugo **blog** site for "EC" (Epimethean Challenge), a play-by-mail game.
The repository is [`mdhender/gamehub-blog`](https://github.com/mdhender/gamehub-blog).
It uses the [Hextra](https://github.com/imfing/hextra) theme via Hugo modules and is published to https://blog.epimethean.dev/.

> **Note:** The documentation site is a separate repository (`gamehub-docs`). This repo contains only the blog.

## Tech Stack

- **Hugo** static site generator (Go-based, uses Hugo modules)
- **Hextra theme** — imported as a Hugo module (`github.com/imfing/hextra`)
- **Content format**: Markdown with YAML front matter
- **Configuration**: `hugo.yaml`

## Content Structure

All content lives directly under `content/` as flat blog posts:

```
content/
├── _index.md                        # Blog landing page
├── about-epimethean-challenge.md    # About page
├── sprint-1-api-server-v0.md        # Sprint / dev-log posts
├── switching-to-laravel.md          # ...
└── ...
```

## Conventions

- All content pages use YAML front matter delimited by `---`.
- The blog landing page is `content/_index.md`.
- Blog post files use kebab-case (e.g., `sprint-6-ff-v4-migration.md`).
- Use Hextra shortcodes where appropriate: `{{< callout >}}`, `{{< cards >}}`, `{{< card >}}`, `{{< tabs >}}`, `{{< tab >}}`.

## Build & Preview

```sh
tools/server.sh   # Local dev server
hugo              # Build to public/
tools/deploy.sh   # Deploy to production
```

## Do Not Modify

- `go.mod` / `go.sum` — managed by Hugo modules; do not edit manually.
- `public/` — generated output; gitignored.
- `themes/` — empty; theme is loaded via Hugo modules.
