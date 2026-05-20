# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Git Remotes

- `origin` → `https://github.com/hughengwu/playwright-crx.git` (our fork)
- `upstream` → `https://github.com/ruifigueira/playwright-crx` (original author)

Sync upstream: `git fetch upstream && git merge upstream/main && git push origin main`

## What This Repo Is

`playwright-crx` is a library that makes Playwright work inside Chrome extensions. Instead of connecting via `--remote-debugging-port`, it uses Chrome's `chrome.debugger` API as a CDP transport. This lets extensions control browser tabs with the full Playwright API.

The repo also contains `examples/recorder-crx` — a ready-to-use Chrome extension that embeds the Playwright recorder (`playwright codegen`) so users can record scripts in their real browser.

The `playwright/` directory is a **git subtree** of the upstream Playwright repo (not a submodule). Do not edit files inside it directly.

## Build Commands

All commands run from the repo root unless noted.

```bash
# Full build (lint + playwright bundles + library + examples + tests)
npm ci
npm run build   # requires NODE_OPTIONS="--max_old_space_size=4096"

# Incremental build (faster, skips lint and tests)
npm run ci:pw:bundles        # build Playwright's internal bundles (required first)
npm run build:crx            # build the playwright-crx library → lib/
npm run build:examples:recorder  # build the recorder extension → examples/recorder-crx/dist/

# Lint
npm run lint
```

After building, load `examples/recorder-crx/dist/` as an unpacked extension in Chrome (`chrome://extensions/` → Developer mode → Load unpacked).

## Testing

Tests live in `tests/` and use Playwright to test the extension itself in a real Chrome browser.

```bash
# Install browsers (once)
npm run test:install

# Run all tests (must have built first)
npm run test

# Run a single spec file
cd tests && npx playwright test crx/recorder.spec.ts

# Run with UI
cd tests && npx playwright test --ui --timeout 0

# On Linux CI (headless display required)
xvfb-run --auto-servernum --server-args="-screen 0 1280x960x24" -- npm run test
```

### How Tests Work

The `context` fixture in `tests/crx/crxTest.ts` launches Chrome with `--load-extension` pointing to `tests/test-extension/dist/`. Tests use two mechanisms:

- **`runCrxTest(fn)`** — serializes `fn` as a string and evaluates it inside the extension's service worker via `extensionServiceWorker.evaluate()`. Use this to test `playwright-crx` API behavior.
- **Direct Playwright** — controls the Chrome UI (clicks the extension toolbar button, reads the recorder panel) to test the recorder UX.

The `Chrome DevTools` project (non-CI only) opens the service worker DevTools and pauses execution at a `debugger` statement before each test — useful for setting breakpoints.

## Architecture

```
playwright-crx/
├── lib/                    # compiled library output (generated)
├── index.d.ts              # public API types entry point
├── src/                    # library source (crxTransport.ts is the key file)
│   └── types/types.ts      # CrxApplication, Crx, CrxFs types
├── playwright/             # git subtree of microsoft/playwright
├── examples/
│   └── recorder-crx/       # the Chrome extension
│       ├── public/manifest.json   # MV3: permissions=[debugger,tabs,sidePanel,storage,contextMenus]
│       └── src/
│           ├── background.ts      # service worker — all extension logic lives here
│           └── settings.ts       # persisted user preferences (language, sidepanel, testIdAttr)
└── tests/
    ├── crx/
    │   ├── crxTest.ts       # base test fixture (loads extension, provides runCrxTest)
    │   ├── crxRecorderTest.ts  # recorder-specific fixture
    │   └── *.spec.ts        # test files
    └── test-extension/      # minimal extension used by tests (separate from recorder-crx)
```

### Core Concept: `crxTransport.ts`

The library implements Playwright's `ConnectionTransport` interface using `chrome.debugger` instead of a WebSocket. This means Playwright's client-server protocol flows through the Chrome extension API.

### `background.ts` (recorder-crx service worker)

- Lazy-initializes a single `CrxApplication` via `crx.start()`
- `attach(tab)` → opens recorder (side panel or popup) → `crxApp.attach(tabId)`
- Triggered by: toolbar button click, context menu, keyboard shortcuts (`Shift+Alt+R` = record, `Shift+Alt+C` = inspect)
- Badge shows `REC` (dark red) while recording, `INS` (blue) while inspecting

### Recorder UI

Built with React + Vite. Three HTML entry points:
- `index.html` → recorder panel (displayed in side panel or popup)
- `preferences.html` → options page
- `public/empty.html` → side panel placeholder when recorder is closed

### Player

The recorder has a built-in player that replays recorded steps. It uses an internal JSONL format rather than executing Python/Java/C# — it maps JSONL actions to the currently selected language's display while executing the underlying steps via `playwright-crx`.

## CI / Release

- **`build.yml`**: triggers on push to `main`/`release-*` and PRs. Runs full build + tests + uploads `examples/recorder-crx/dist/` as artifact.
- **`release.yml`**: triggers on GitHub Release publish or `workflow_dispatch` (requires `tag` input). Builds and uploads `recorder-crx.zip` to the GitHub Release assets via `gh release upload`.

To publish a new release:
1. Create a GitHub Release with a version tag (e.g. `v0.15.2`)
2. `release.yml` triggers automatically and attaches the built zip

## Updating Playwright Subtree

```bash
git subtree pull --prefix=playwright git@github.com:microsoft/playwright.git v1.XX.0 --squash
```
