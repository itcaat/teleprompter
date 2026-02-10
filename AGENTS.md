# readmode.ru  — Agent context

## What this app is

**readmode.ru ** is a minimal, single-page text editor that runs entirely in the browser. All content is stored in the **URL hash** (and optionally in **localStorage**). There is no backend; sharing = sharing the URL. This project is **based on** [antonmedv/textarea](https://github.com/antonmedv/textarea), extended with a teleprompter and other features.

- **Site**: https://readmode.ru 
- **Stack**: Vanilla HTML/CSS/JS, no frameworks. PWA (manifest + service worker).

## Core behavior

1. **Storage**
   - On load: content comes from `location.hash` if present, else from `localStorage.getItem('hash')`.
   - On input/blur: content is compressed, written to `location.hash`, and saved to `localStorage`.
   - Hash format (shorter = better for QR): **`#b`** + base64url(brotli) when Brotli is supported; **`#z`** + base86(deflate) otherwise; legacy **`#`** + base64url(deflate) for old links. Payload is content + optional `\x00` + style.

2. **Compression**
   - `compress(string)` → try Brotli → `b`+base64url; else deflate-raw → base86 (URL-safe) → `z`+encoded.
   - `decompress(payload)` → if `b` prefix: brotli; if `z`: base86 decode → inflate; else base64url → inflate (legacy).
   - Uses `CompressionStream`/`DecompressionStream` (deflate-raw or brotli), and base64url or custom base86 for encoding.

3. **Editor**
   - One `<article contenteditable="plaintext-only">` as the main editor.
   - **Editor** class: undo/redo (history), selection save/restore, debounced markdown highlighting.
   - **Markdown** is parsed for display only (headings, code, bold, italic, strike, URLs); stored data is plain text. Parser is in `parseMarkdown()` with regex matchers and `md-*` classes.

4. **Title**
   - If the first line matches `# Title`, `document.title` is set to that title (see `updateTitle()`).

5. **Export / share**
   - **Share**: copy current URL to clipboard.
   - **Save as HTML**: full page clone, scripts and `.noprint` removed, article no longer contenteditable.
   - **Save as TXT**: plain text of article.
   - **QR**: `/qr` page shows a QR code for the current URL (hash preserved); uses a bundled qrcode library and hash from `location.hash`.

6. **Teleprompter (Телетекст)**
   - **UI**: One green **Play** button (left) and one **Menu** button (right) at bottom. Menu contains: New document, QR, Share, Save as HTML/TXT, **Телетекст** section, GitHub.
   - **Телетекст section** (inside menu): one row with **A−** / **A+** (font size 14–72 px), **⇄** / **⇅** (mirror H/V), **color picker** (background); then **speed row**: snail icon, **speed slider** (20–500 px/s), lightning icon.
   - **Live preview**: Font size, background color, and mirror (H/V) apply immediately to the article in edit mode (CSS variables `--prompter-font-size`, `--prompter-bg`; classes `prompter-mirror-h`, `prompter-mirror-v` on `article`).
   - **Play mode**: Click Play → body gets `prompter-mode`, article becomes non-editable, smooth auto-scroll down at current speed. Play and Menu buttons become semi-transparent (opacity 0.35); Play shows **pause** icon and toggles to stop on click. **Arrow Up/Down**: smooth fast scroll up/down (~140 px, ~180 ms ease).
   - **State**: Teleprompter settings (fontSize, bgColor, mirrorH, mirrorV, speed) are stored in **localStorage** under key `teleprompter` (JSON). Not in URL hash.
   - **Init**: `initTeleprompter()` runs after `initUI()`; reads state, binds toolbar/menu controls, applies font/bg/mirror on load, and sets up play/pause and scroll loop (`requestAnimationFrame`).

## File layout

| File / folder | Purpose |
|---------------|--------|
| `index.html` | Single-file app: HTML, CSS, and all JS (load/save, compress/decompress, Editor, parseMarkdown, UI, menu, teleprompter, download helpers). |
| `qr.html` | QR code page; reads `location.hash`, builds full URL, renders QR SVG. |
| `sw.js` | Service worker: caches `/` and `/qr`, cache-first fetch. |
| `manifest.json` | PWA manifest (name, display, icons, theme). |
| `404.html` | Custom 404 if used by host. |
| `assets/` | Teleprompter speed icons: `snail-slow.png`, `lightning-fast.png` (referenced from menu). |

## Tech details

- **No build step**: everything is static. No npm/node required for the app itself.
- **Debounce**: 500 ms for save on input; 30 ms for markdown highlight; 300 ms for history recording.
- **History**: undo/redo keeps last 10k states (HTML + selection position).
- **Light/dark**: CSS uses `prefers-color-scheme` and variables (`--text`, `--elevated`, `--link`, etc.).
- **Print**: `.noprint` hides FAB and menu; article stays visible.

## Conventions for changes

- Keep the app **single-file** for `index.html` (no JS/CSS split) unless there's a strong reason.
- Preserve **hash-based storage**: URL must remain the source of truth for shared state; localStorage is a local fallback.
- Don't break **base64url** and **deflate-raw** format; existing shared links must keep working.
- **qr.html** must continue to use `location.hash` and the same origin/domain (e.g. `https://readmode.ru /`) when building the URL for the QR.
- When touching the Editor or selection, be careful with **contenteditable** and the way **parseMarkdown** replaces content (selection save/restore must work after re-renders).
- Service worker **CACHE_NAME** in `sw.js` uses placeholder `DEPLOY_SHA`; the deploy workflow replaces it with the git short hash so each deploy gets a new cache and users get fresh content.
- **Teleprompter**: Settings stay in `localStorage` only (key `teleprompter`); do not put them in the URL hash. Keep live preview for font size, background, and mirror. Play button must toggle to pause icon when in `prompter-mode` and stay in the same place; menu button stays visible and semi-transparent in prompter mode. Speed slider range: min 20, max 500 (constants `SPEED_MIN`, `SPEED_MAX` and input `min`/`max` must match). Assets for speed icons live in `assets/` and are referenced as `assets/snail-slow.png`, `assets/lightning-fast.png`.

## Quick reference

- **Get current payload**: `article.textContent` (+ optional `article.getAttribute('style')` with `\x00` separator).
- **Load from hash**: `decompress(hash.slice(1))` → split by `\x00` → content + optional style.
- **Save**: `compress(content + (\x00 + style))` → `#` + base64url → `history.replaceState` and `localStorage.setItem('hash', …)`.
- **Teleprompter state**: `localStorage.getItem('teleprompter')` → JSON `{ fontSize, bgColor, mirrorH, mirrorV, speed }`. Apply to UI via `applyFontSize()`, `applyBgColor()`, `updateArticleMirror()`; scroll in play mode: `state.speed` px/s, `requestAnimationFrame`.
