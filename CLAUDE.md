# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Static marketing site for **PayXO** (payxo.ru), served via GitHub Pages (`CNAME` → `payxo.ru`, repo `likeahustla/payxo-site`, **public**). The site drives traffic to the Telegram bot `@payxo_bot`, which sells gift cards and digital codes (ChatGPT, Claude, Suno, iCloud+, Steam, Telegram Premium, etc.) — typically via Apple ID balance top-ups so buyers pay in rubles by SBP and activate in the vendor's own app, no card/FX fees. Legal entity: ИП Султанова Фатима Салмановна.

## Commands

There is no build system — no `package.json`, no bundler, no test runner. Pages are plain HTML/CSS/JS files served as-is.

- **Preview locally**: open any `index.html` directly in a browser, or serve the directory with any static file server.
- **Deploy**: commit and push to `main` — GitHub Pages rebuilds automatically. Check build status with `gh api repos/likeahustla/payxo-site/pages/builds/latest`.

## Architecture

**One self-contained HTML file per page, no shared assets, no templating.** Every page under `guides/<slug>/index.html` and `legal/<slug>/index.html` (directory-per-page, for clean URLs on GitHub Pages) inlines its own `<style>` block and its own `<script>` blocks. There is no shared CSS/JS file and no include/template mechanism — `icons/` (SVGs/PNGs) and Bunny Fonts (`fonts.bunny.net`, Unbounded + Onest) are the only assets pages actually share.

**Practical consequence: sitewide changes must be hand-applied to every HTML file.** There are currently 16 HTML pages: `index.html`, 8 under `guides/` (`guides/index.html` + 7 articles — ChatGPT, Claude, Suno, iCloud+, the "как оплатить зарубежные сервисы" pillar guide, App Store region change, Apple balance write-off), and 7 under `legal/` (`legal/index.html` + 6 legal docs). Anything "global" — the Yandex Metrika snippet, CTA click-tracking script, JSON-LD boilerplate, OG/Twitter meta, design tokens — is duplicated verbatim in each file's `<head>`/`<body>`. When editing one of these, grep across all 16 files rather than assuming a single source of truth. Small scripted edits (Python/sed over the file list) are more reliable here than manual per-file editing.

Each page's `<style>` redefines the same CSS custom-property palette (`--ink`, `--lime`, `--card`, `--card-border`, `--dim`, etc.) — keep names/values consistent with the existing pages when adding a new one.

**Internal links never include `index.html`.** Every logo/breadcrumb/back-button link points at a directory (`../../`, `../`, `./`), not the file (`../../index.html`) — GitHub Pages serves the `index.html` inside a directory automatically, so the extra filename only showed up in the address bar after a click for no reason. Keep new internal links directory-style; don't reintroduce `index.html` in an `href`.

## Analytics

- **Yandex Metrika** counter `111606678` (Webvisor + clickmap + ecommerce dataLayer enabled) is embedded in every page's `<head>`, but **gated on cookie consent**: the snippet only defines `window.payxoInitMetrika()` and calls it immediately if `localStorage.getItem('payxo_cookie_consent') === 'accepted'` from a prior visit. It does not fire unconditionally on page load.
- **Cookie banner** (`#cookie-banner`, near `</body>` on every page) has two buttons — `#cookie-accept` and `#cookie-decline`. Accept sets `payxo_cookie_consent = 'accepted'` and calls `window.payxoInitMetrika()`; decline sets it to `'declined'` and never loads Metrika. The banner stays hidden once either choice is stored, so Metrika only ever runs after explicit opt-in.
- **CTA click tracking**: every "Open bot" link (`t.me/payxo_bot`) carries a `data-analytics="<location>"` attribute; a shared inline script (appended before `</body>` on each page) fires `ym(111606678, 'reachGoal', 'open_bot_click', {location: ...})` on click — this only does anything once `ym` actually exists, i.e. after consent. Goal `open_bot_click` (goal id `597461960`) is registered in the Metrika counter via the Management API.
- **`.mcp.json`** registers a local `yandex-metrica` MCP server for querying/managing this counter (read + write scope token). **It is gitignored and must never be committed** — it holds a live OAuth token.
- Yandex Metrika Management API note: the documented condition `type` value for JS-event ("action") goals is `"action"` per the official docs, but the live API actually requires `"exact"` — the docs are wrong on this point.

## Workflow

- After a commit that changes architecture, analytics behavior, or any other fact this file documents, update the relevant section of `CLAUDE.md` in the same commit (or an immediate follow-up commit) — don't let it drift out of sync with what the code actually does.
- After committing, push to `origin main` right away unless the user says to hold off — don't leave commits sitting local-only. GitHub Pages only rebuilds from what's actually pushed.

## SEO / AI discoverability

- Every page carries JSON-LD (`Organization` + a page-specific type: `WebSite` on the homepage, `CollectionPage` on the guides/legal index pages, `HowTo`+`FAQPage` on guide articles, `WebPage` on legal docs), full canonical/robots/OG/Twitter meta.
- The `WebSite`/`CollectionPage`/`HowTo` JSON-LD nodes also carry a `keywords` field (comma-separated string of relevant phrases) — this is a legitimate schema.org signal for AI/search context, not a ranking hack. Don't confuse it with the `<meta name="keywords">` tag, which Google and Yandex both ignore — never add that tag.
- `guides/` mixes two article shapes: service-specific how-tos (ChatGPT, Claude, Suno, iCloud+, region change, balance write-off) and a broader pillar/hub page (`kak-oplatit-zarubezhnye-servisy`) that explains the general method once and links out to the specific guides — use the same pattern for future broad-intent guides rather than duplicating the general explanation into every article.
- `llms.txt` at the root follows the llmstxt.org convention for AI-agent discovery — update it when guides are added/removed.
- `sitemap.xml` has hand-maintained `lastmod`/`changefreq`/`priority` per URL — bump `lastmod` when a page's content changes.
