# Technology Stack

## Core Sections (Required)

### 1) Runtime Summary

| Area | Value | Evidence |
|------|-------|----------|
| Primary language | TypeScript (strict mode) | [tsconfig.json](../../tsconfig.json) |
| Runtime + version | Bun 1.x | [Dockerfile](../../Dockerfile), [package.json](../../package.json) |
| Package manager | Bun (`bun install`, `bun.lock`) | [bun.lock](../../bun.lock), [.npmrc](../../.npmrc) |
| Module/build system | Vite 7 + SvelteKit 2 | [vite.config.ts](../../vite.config.ts), [svelte.config.js](../../svelte.config.js) |

### 2) Production Frameworks and Dependencies

| Dependency | Version | Role in system | Evidence |
|------------|---------|----------------|----------|
| `@sveltejs/kit` | ^2.49.1 | Full-stack web framework (SSR + API routes) | [package.json](../../package.json) |
| `svelte` | ^5.45.6 | UI component framework (v5 runes) | [package.json](../../package.json) |
| `svelte-adapter-bun` | ^1.0.1 | Production adapter — runs SvelteKit on Bun | [svelte.config.js](../../svelte.config.js) |
| `valibot` | ^1.3.1 | Schema validation for all external input | [package.json](../../package.json) |
| `liquidjs` | ^10.24.0 | Template engine for torrent descriptions | [package.json](../../package.json) |
| `p-queue` | ^9.1.0 | Concurrency control / rate limiting | [package.json](../../package.json) |
| `sharp` | ^0.34.5 | Image processing (screenshot thumbnails) | [package.json](../../package.json) |
| `mediainfo.js` | ^0.3.6 | WASM-based MediaInfo analysis (in-process) | [package.json](../../package.json) |
| `cheerio` | ^1.2.0 | HTML parsing (tracker response scraping) | [package.json](../../package.json) |
| `@js-temporal/polyfill` | ^0.5.1 | Temporal API for dates/times | [package.json](../../package.json) |
| `@isaacs/ttlcache` | ^2.1.4 | TTL-based caching (tracker data) | [package.json](../../package.json) |
| `p-memoize` | ^8.0.0 | Function memoization | [package.json](../../package.json) |
| `iso-639-2` | ^3.0.2 | Language code lookup (release parsing) | [package.json](../../package.json) |
| `expiry-map` | ^2.0.0 | Expiring key-value store | [package.json](../../package.json) |
| `svelte-dnd-action` | ^0.9.69 | Drag-and-drop UI interactions | [package.json](../../package.json) |

### 3) Development Toolchain

| Tool | Purpose | Evidence |
|------|---------|----------|
| TypeScript | Type checking (`strict: true` + additional strictness flags) | [tsconfig.json](../../tsconfig.json) |
| `svelte-check` | Svelte + TS diagnostic checking | [package.json](../../package.json) `scripts.check` |
| Vite | Dev server with HMR, production bundler | [vite.config.ts](../../vite.config.ts) |
| `@sveltejs/vite-plugin-svelte` | Svelte integration for Vite | [package.json](../../package.json) |

No linter (ESLint) or formatter (Prettier) is configured.

### 4) Key Commands

```bash
bun install                  # Install dependencies
bun run dev                  # Start dev server (uses bunx --bun vite dev)
bun run build                # Production build (bun --bun vite build)
bun run check                # Type check (svelte-kit sync && svelte-check)
bun run check:watch          # Type check in watch mode
```

Production run after build:
```bash
ORIGIN=http://localhost:51901 PORT=51901 bun build/index.js
```

### 5) Environment and Config

- **Config source**: `$APPDATA/ak-automated-uploader/settings.json` (persisted singleton)
  - Windows: `%APPDATA%/ak-automated-uploader/settings.json`
  - macOS: `~/Library/Preferences/ak-automated-uploader/settings.json`
  - Linux: `~/.local/share/ak-automated-uploader/settings.json`
- **No `.env` file** — all configuration managed through the settings UI or API
- **Required external binaries on PATH**: `ffmpeg`, `ffprobe`, `mkbrr`
- **Required env vars for production**: `ORIGIN`, `PORT` (defaults in Dockerfile: `51901`)
- **Runtime constraint**: Bun only — uses `Bun.file()`, `Bun.spawn()`, `bun:ffi`, `Bun.color()` APIs directly

### 6) Evidence

- [package.json](../../package.json)
- [tsconfig.json](../../tsconfig.json)
- [svelte.config.js](../../svelte.config.js)
- [vite.config.ts](../../vite.config.ts)
- [Dockerfile](../../Dockerfile)
- [docker-compose.yml](../../docker-compose.yml)
- [src/lib/server/settings.ts](../../src/lib/server/settings.ts)
