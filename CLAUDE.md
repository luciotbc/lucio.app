# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Personal blog at https://lucio.app — a Jekyll site built on the **Chirpy** theme (`jekyll-theme-chirpy ~> 7.5` gem, from the chirpy-starter template). Content is technical notes (Ruby/Rails, Sidekiq/Resque, Postgres, Arch Linux, macOS). Site language is `pt-BR`, timezone `America/Sao_Paulo`.

Ruby 3.4.5 (`.ruby-version` / `.tool-versions`, managed by mise — the default `ruby` may be 4.x, which Chirpy rejects; in non-interactive shells run `eval "$(mise env -s bash)"` first). htmlproofer needs a UTF-8 locale (`LANG=en_US.UTF-8`) or it fails on accented pages.

## Commands

```bash
bundle install
bundle exec jekyll serve          # local dev server (or: bash tools/run.sh [-H host] [-p for production mode])
bash tools/test.sh                # production build into _site + htmlproofer (internal links only) — same checks as CI
```

There is no unit test suite; `tools/test.sh` (build + `htmlproofer --disable-external`) is the only validation. Run it after changing posts, since broken `{% post_url %}` references fail the build.

## Architecture

- **Theme lives in the gem.** Layouts, includes, sass, and most JS come from `jekyll-theme-chirpy`; this repo only holds overrides: `_config.yml`, `_tabs/` (about/archives/categories/tags pages), `_data/` (locales, contact, share), `_plugins/`, `index.html`. To change theme markup, copy the file from the gem into the matching local path rather than editing the gem.
- `assets/lib` is a git submodule (`chirpy-static-assets`), only used if `assets.self_host` is enabled (currently not).
- `_plugins/posts-lastmod-hook.rb` sets `last_modified_at` from git history — builds need full git history (CI uses `fetch-depth: 0`).
- `_config.yml` defaults: posts get `layout: post`, `toc: true`, `comments: true`, and `permalink: /posts/:title/` (slug from filename). Tabs get `/:title/`. Archives for tags/categories via `jekyll-archives`.
- `tools/`, `README.md`, etc. are excluded from the build; `_site/` and `.jekyll-cache/` are generated.

## Posts

- Filename: `_posts/YYYY-MM-DD-slug.md`, written in Portuguese. Front matter used in this repo: `title`, `date` and `updated` (format `2026-04-07 14:00:00 -0300`), `tags: [...]`, `excerpt`.
- Multi-part series (Sidekiq 1–4, Resque 1–3) open with a blockquote index linking siblings via `{% post_url YYYY-MM-DD-slug %}`; keep all parts' indexes in sync when adding/renaming a part.
- `published: false` posts are drafts kept in the repo (e.g. the Raspberry Pi post); keep the flag in sync across translations.

## i18n (jekyll-polyglot)

- `languages: ["pt-BR", "en", "es"]` in `_config.yml`. Polyglot builds the whole site once per language: pt-BR at `/`, others at `/en/` and `/es/`. Each pass only sees that language's posts, so home, pagination, tags, archives, feed and search are already filtered.
- **A translation is a file with the same filename under `_posts/en/` or `_posts/es/`** (`lang_from_path: true`; `lang:`/`ref:` front matter is also present). Same filename → same `/posts/:title/` permalink → Polyglot links them. Keep filenames identical when translating.
- In translated posts, don't use `{% post_url %}` (subdirectory posts trigger mismatch warnings); link as `/posts/<slug>/` — Polyglot rewrites absolute internal links to the active language prefix. Links to files in `exclude_from_localization` (e.g. `/assets/downloads/...`) stay unprefixed; don't use `{% link %}` for them, it fails in non-default passes.
- To emit a link that must NOT be rewritten (e.g. to another language), wrap it: `{% static_href %}href="..."{% endstatic_href %}`.
- Theme overrides for i18n (copied from the gem, keep in sync on theme upgrades):
  - `_includes/lang.html` — UI locale from `site.active_lang` via `_data/i18n.yml` (`es` → `es-ES.yml`).
  - `_includes/sidebar.html` — per-language tagline and globe dropdown language switcher (stores choice in `localStorage.lang`).
  - `_includes/metadata-hook.html` — `hreflang` alternates (x-default → en) and the browser-language redirect script, which only runs on pt-BR (root) pages: stored choice wins, else `pt*` stays, `es*` → `/es/`, anything else → `/en/`; bots are skipped.
  - `_includes/topbar.html` — links to the other languages (native names, active language hidden) left of the search box; styled in `assets/css/jekyll-theme-chirpy.scss` (the theme's purged CSS bundle lacks most Bootstrap responsive utilities like `d-lg-flex`, so add custom CSS there instead).
  - `_includes/search-loader.html` + `assets/js/data/search.json` — per-language search index.
- Per-language strings not in `_data/locales` (tagline, language names) live in `_data/i18n.yml`. Tabs translate the same way (`_tabs/en/about.md`, `_tabs/es/about.md`); untranslated tabs fall back to pt-BR.

## Deploy

- `.github/workflows/pages-deploy.yml`: on push to `main`, builds with `JEKYLL_ENV=production`, runs htmlproofer, deploys to GitHub Pages.
- README also references Netlify (https://app.netlify.com/projects/lucio/deploys) as the deploy target for lucio.app.
- `assets/downloads/lucio-charallo-cv.pdf` is the published CV (commits like "update cv" just replace this file).
