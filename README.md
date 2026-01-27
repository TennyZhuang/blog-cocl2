# COCl2's Blog

Static site for https://blog.cocl2.com built with Hugo, the Paper theme, Plausible analytics, and Giscus-powered comments.

## Prerequisites
- Hugo **extended** v0.154.5+ (install via `go install github.com/gohugoio/hugo@latest` or your package manager).
- Go 1.20+ available on your PATH so Hugo can download theme modules.

## Run Locally
1. Install modules (first run only): `hugo mod tidy`
2. Start the dev server with drafts: `hugo server -D`
3. Open http://localhost:1313 to preview live reloads.

## Write Posts
- Create a draft: `hugo new posts/my-title/index.md` (uses `archetypes/default.md`, sets `draft = true`).
- Keep images next to the post’s `index.md` to take advantage of page bundles.
- Math rendering is enabled site-wide; code blocks use the `catppuccin-latte` highlight style.

## Build for Production
- Generate the site: `hugo --minify` (outputs to `public/`).
- `baseURL` is set to `https://blog.cocl2.com` in `hugo.toml`; set `HUGO_ENV=production` if your deploy target relies on it.
- Plausible tracking (`params.plausible`) and social/meta tags are wired in `layouts/partials/head.html`.

## Update Dependencies
- Refresh themes/modules: `hugo mod get -u`
- Clean unused entries: `hugo mod tidy`

## Repo Tour
- `hugo.toml` – site config, theme list, analytics, and Giscus settings.
- `content/posts/` – markdown posts and page bundles.
- `archetypes/default.md` – front matter template for new content.
- `layouts/partials/head.html` – custom head tags, analytics, SEO, and feed links.
