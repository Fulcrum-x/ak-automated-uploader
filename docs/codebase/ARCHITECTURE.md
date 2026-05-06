# Architecture

## Core Sections (Required)

### 1) Architectural Style

- **Primary style**: Layered with plugin architecture — SvelteKit routes delegate to server-side orchestrators, which coordinate domain modules and extensible plugin registries (trackers, image hosts, torrent clients)
- **Why this classification**: Clear separation between routes (presentation), orchestration (`Upload`, `Trackers`), and plugins (tracker/image-host/client implementations). Plugin registries use a common pattern: abstract base class + concrete implementations + index registry.
- **Primary constraints**:
  1. Bun-only runtime (uses Bun FFI, file APIs, spawn — not portable to Node.js)
  2. External binary dependencies (`ffmpeg`, `ffprobe`, `mkbrr`) must be on PATH
  3. Single-process, in-memory state — no database, no external message queue

### 2) System Flow

```text
User/API Request
  → SvelteKit Route (+server.ts / +page.server.ts)
    → uploads.create(path) → Upload orchestrator
      → Release (parse filename)
      → TMDB/Jikan (metadata lookup)
      → Files (directory scan)
      → Screenshots (ffprobe + ffmpeg)
      → Torrent (mkbrr create)
      → MediaInfo (WASM analysis)
      → Trackers (configure all enabled trackers)
        → Tracker.submit()
          → transformTags() (LiquidJS templates + screenshot upload)
          → upload() (HTTP POST to tracker API)
          → sendTorrent() (push to torrent client)
    → SSE stream updates to UI
```

1. **Request arrives** at a SvelteKit route handler or API endpoint
2. **Upload created** — `uploads.create(path)` instantiates an `Upload` which immediately begins async initialization
3. **Metadata gathered** — Release parses filename; TMDB/Jikan look up external IDs; MediaInfo analyzes codec/resolution; Screenshots extracts frames via ffmpeg
4. **Trackers configured** — Each enabled tracker gets `applyRelease()` and `applyMetadata()` called with gathered data
5. **Submission** — `Tracker.submit()` runs the template engine (LiquidJS) to build descriptions, uploads screenshots to image hosts, posts to tracker API, downloads returned torrent, sends to torrent client
6. **Status streaming** — Every state change emits via observer callbacks, serialized to SSE for real-time UI updates

### 3) Layer/Module Responsibilities

| Layer or module | Owns | Must not own | Evidence |
|-----------------|------|--------------|----------|
| `Upload` (orchestrator) | Coordinating all subsystems for a single upload, lifecycle management, abort handling | Individual tracker upload logic, image host details | [upload.ts](../../src/lib/server/upload.ts) |
| `Tracker` (abstract base) | Template method for submit flow, field validation, LiquidJS tag processing, status transitions | Tracker-specific API calls, field definitions | [tracker.ts](../../src/lib/server/tracker.ts) |
| `trackers/*.ts` (concrete) | Tracker-specific fields, layout, API communication, metadata mapping, search | Cross-tracker orchestration | [trackers/lst.ts](../../src/lib/server/trackers/lst.ts) |
| `Trackers` (collection) | Managing multiple tracker instances, propagating shared state (release, metadata, screenshots) | Individual tracker submit logic | [trackers.ts](../../src/lib/server/trackers.ts) |
| `Release` | Filename parsing into structured metadata, regex pattern matching | File I/O, external API calls | [release.ts](../../src/lib/server/release.ts) |
| `Settings` (singleton) | Persisting and validating all user configuration | Business logic, upload flow | [settings.ts](../../src/lib/server/settings.ts) |
| `Screenshots` | FFmpeg/FFprobe interaction, screenshot lifecycle | Image hosting, tracker descriptions | [screenshots.ts](../../src/lib/server/screenshots.ts) |
| `Torrent` | mkbrr process management, torrent creation/editing | Tracker uploads, screenshot logic | [torrent.ts](../../src/lib/server/torrent.ts) |
| `ImageHost` (abstract) | Single image upload contract | Host selection strategy, caching | [image-host.ts](../../src/lib/server/image-host.ts) |
| `uploadScreenshots` | Host selection strategy, fallback logic, result caching | Individual host upload mechanics | [upload-screenshots.ts](../../src/lib/server/upload-screenshots.ts) |
| Routes | Request handling, SSE streaming, page data loading | Core business logic | [src/routes/](../../src/routes/) |

### 4) Reused Patterns

| Pattern | Where found | Why it exists |
|---------|-------------|---------------|
| **Singleton** | `settings`, `uploads`, `tmdb`, each image host, each torrent client | In-memory state management without a database; single-process app |
| **Template Method** | `Tracker.submit()` calls abstract `upload()` | Standard submit flow with tracker-specific upload implementation |
| **Strategy** | `uploadScreenshots` tries image hosts in priority order | Fallback across multiple image hosting services |
| **Observer (callbacks)** | Every major class (`Upload`, `Tracker`, `Trackers`, `Settings`, `Files`, `Screenshots`) | Real-time SSE streaming to UI without EventEmitter dependency |
| **Registry** | `trackers/index.ts`, `image-hosts/index.ts`, `torrent-clients/index.ts` | Plugin discovery — add new implementations by registering in index |
| **Abstract Base Class** | `Tracker`, `ImageHost`, `TorrentClient` | Enforce interface contracts for plugins |
| **Cooperative suspension** | `pauseHashing()`/`resumeHashing()` with refcount | Prevent I/O contention between mkbrr hashing and ffmpeg/mediainfo/file-browser |
| **PQueue rate limiting** | TMDB (concurrency: 1), Jikan (1 req/sec), torrent ops (concurrency: 1), screenshots (concurrency: 1) | Respect API rate limits and prevent resource contention |
| **AbortSignal threading** | `Upload` → `Tracker` → all async operations | Clean cancellation of long-running upload workflows |

### 5) Known Architectural Risks

- **In-memory state only** — All uploads, sessions, and caches are lost on process restart. No persistence layer for active uploads.
- **Single-process constraint** — Cannot horizontally scale; singletons and in-memory Maps assume a single instance.
- **Windows FFI coupling** — `torrent.ts` uses `bun:ffi` with `kernel32.dll` and `ntdll.dll` for process suspension on Windows, creating tight platform coupling.
- **Large files** — `unit3d-distributors.ts` (50KB) and `release.ts` (33.6KB) are complex single-file modules that could become maintenance burdens.
- **No request queuing** — Multiple simultaneous uploads compete for CPU/disk with no backpressure beyond PQueue on individual operations.

### 6) Evidence

- [src/lib/server/upload.ts](../../src/lib/server/upload.ts)
- [src/lib/server/tracker.ts](../../src/lib/server/tracker.ts)
- [src/lib/server/trackers.ts](../../src/lib/server/trackers.ts)
- [src/lib/server/torrent.ts](../../src/lib/server/torrent.ts)
- [src/lib/server/upload-screenshots.ts](../../src/lib/server/upload-screenshots.ts)
- [src/lib/server/settings.ts](../../src/lib/server/settings.ts)
- [src/hooks.server.ts](../../src/hooks.server.ts)
