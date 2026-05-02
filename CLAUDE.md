# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Helixium is a fork of [Vimium](https://vimium.github.io/), a keyboard-driven browser extension for Chrome and Firefox. It uses Manifest V3 and is built/tested with **Deno** (no Node.js / npm, no bundler, no transpilation).

## Commands

All tasks run through `./make.js` (a [Drake](https://github.com/nicholasgasior/deno-drake)-based task runner):

```sh
./make.js test              # Run all tests (unit + DOM)
./make.js test-unit         # Unit tests only (Deno, no browser needed)
./make.js test-dom          # DOM tests only (headless Chrome via Puppeteer)
./make.js package           # Build zip files for Chrome and Firefox stores → dist/
deno fmt                    # Format code (100-char line width)
```

**One-time Puppeteer setup** (required before `test-dom`):
```sh
deno run -A npm:puppeteer browsers install chrome
```

**Dev installation:**
- Chrome: load the repo directory as an unpacked extension at `chrome://extensions`
- Firefox: run `./make.js write-firefox-manifest` first, then load via `about:debugging`

**Other useful tasks:**
```sh
./make.js write-firefox-manifest   # Generate Firefox-compatible manifest.json (overwrites)
./make.js fetch-tlds               # Update resources/tlds.txt from IANA
```

## Architecture

### Extension Structure

The extension uses a standard MV3 split between a background service worker and content scripts injected into every page.

**Background (`background_scripts/`)** — service worker entry point is `main.js`. Handles tab management, omnibar completions (bookmarks, history, domains, tabs, search engines in `completion/`), marks, exclusions, zoom, and command routing.

**Content scripts** — loaded in order from `manifest.json` at `document_start` in all frames. They share global state via `globalThis` assignments (no module bundler, deliberate design):
- `lib/` — shared utilities (`utils.js`, `keyboard_utils.js`, `dom_utils.js`, `handler_stack.js`, `settings.js`, etc.)
- `content_scripts/` — UI and mode logic (`mode.js`, `mode_normal.js`, `mode_insert.js`, `mode_find.js`, `mode_visual.js`, `link_hints.js`, `vomnibar.js`, `hud.js`, etc.)

**UI pages (`pages/`)** — standalone extension pages for the omnibar (`vomnibar_page`), HUD (`hud_page`), options, help dialog, and toolbar popup.

### Mode System

All keyboard interaction flows through a `HandlerStack` (defined in `lib/handler_stack.js`). Each mode (`Mode` base class in `content_scripts/mode.js`) pushes handlers onto the stack and pops them on exit. Handler return values control event propagation: `continueBubbling`, `suppressEvent`, `passEventToPage`, `suppressPropagation`, `restartBubbling`.

### Message Passing

Background ↔ content script communication uses `chrome.runtime.sendMessage` / `onMessage`. Each message has a `handler` property for routing. Use `Utils.addChromeRuntimeOnMessageListener` (not the raw Chrome API) to handle the async/sync quirk in MV3.

### Testing

- **Unit tests** (`tests/unit_tests/`) run in Deno using [`shoulda.js`](tests/vendor/shoulda.js). `test_chrome_stubs.js` provides a full fake `chrome.*` API. Tests use `context()/should()` BDD style.
- **DOM tests** (`tests/dom_tests/`) run in real headless Chrome via Puppeteer — for behavior that requires an actual DOM.

## Key Conventions

- **No bundler, no TypeScript** — plain ES2018 JS loaded directly by the browser.
- **Global namespace is intentional** — objects like `Utils`, `Settings`, `handlerStack` live on `globalThis` so all content scripts share state.
- **`manifest.json` uses JSON5** (JS comments allowed) — `make.js` strips comments before packaging.
- **`Utils.debug` in `lib/utils.js` must be `false`** before building for release; the `package` task enforces this.
- **Line width:** 100 characters (enforced by `deno fmt`).
- Firefox support is a build variant — `make.js write-firefox-manifest` generates the Firefox-specific manifest.
