# AGENTS.md


## Project

Single-file Tampermonkey userscript (`vinted_downloader.user.js`) that adds image-download buttons to Vinted item listings. No build system, no package manager, no tests — the `.user.js` file *is* the deliverable. Editing it and reinstalling in a userscript manager is the entire dev loop.

## Commands

No automated tests. To verify a change:
1. Open the `.user.js` in Tampermonkey (or reload it if the `@version` bumped — Tampermonkey re-fetches on version change).
2. Load a real Vinted item page (e.g. `https://www.vinted.com/items/...`) and exercise the "Download All Images" button + the per-image "Save JPG" button in the fullscreen lightbox.
3. Bump `@version` in the header on any user-facing change so installed copies auto-update.

The install URL in README points at `main` branch raw file — pushing to `main` publishes the new version.

## Architecture (single IIFE, top → bottom)

The script runs against Vinted's React SPA, so **there is no stable page-load event** — everything hinges on a `MutationObserver` on `document.body` that re-runs `injectUI()` + `injectSingleSaveButtons()` on every DOM change (throttled via `requestAnimationFrame`). Idempotency is enforced by class markers (`.vinted-dl-all-btn` existence check, `.vinted-dl-processed` per image) — any new injection code MUST add a similar guard or it will duplicate on every mutation.

Key pieces and how they connect:
- **`getImageUrls()`** — scrapes `img` src from `.item-thumbnail`/`.item-main-image`, dedupes via a `Set`, and rewrites the resolution segment `/f{N}/` → `/f800/` (Vinted's max standard size). The same `/f[0-9]+/` → `/f800/` rewrite is duplicated in the single-save click handler — keep both in sync.
- **`fetchAndConvertImage(url)`** — the core webp→jpg conversion. Fetches via `GM_xmlhttpRequest` (bypasses CORS), draws onto a `<canvas>` with a white background fill (handles transparency), exports `image/jpeg` at 0.95. Canvas conversion is the whole reason for the script — it sidesteps Vinted's CDN/CSP restrictions.
- **`getItemMetadata()`** — reads `script[type="application/ld+json"]` Product schema for brand/name, falls back to `<h1>`. Feeds filename generation. Names are always run through `sanitizeFilename()` (strips diacritics via NFD normalization, replaces invalid FS chars with `_`).
- **`downloadAllAsZip()`** — loops URLs, converts each, builds a ZIP in-thread with `fflate.zipSync` at `level: 0` (STORE — JPGs are already compressed, and STORE avoids worker/CSP issues). Files nested under a `{Brand} - {Name}/` folder.
- **`injectSingleSaveButtons()`** — only injects "Save JPG" onto images detected as being inside a fullscreen modal/lightbox (checks known overlay classes first, then falls back to walking ≤4 parents looking for fixed/absolute + high z-index). Intentional: avoids buttons on thumbnails.

Dependencies are `@require`d from CDNs (fflate for zipping, FileSaver.js for `saveAs`) — not vendored.

## Conventions

- `@match` list (script header) and README "Supported Websites" list are parallel registries — adding a Vinted regional domain means updating both.
- The README `## Todo` list is the roadmap; check it before proposing features.

## Decisions
- 2026-02-28: ZIP uses fflate `level: 0` (STORE, no compression) — JPGs don't compress further and STORE is fast + avoids CSP/worker restrictions.
- 2026-02-28: webp→jpg conversion done client-side via `<canvas>` rather than fetching a jpg from the CDN — bypasses Vinted CDN format limits.

## Repository policy

Public repository (NightMean/Vinted_Downloader).
