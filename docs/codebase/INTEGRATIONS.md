# External Integrations

## Core Sections (Required)

### 1) Integration Inventory

| System | Type | Purpose | Auth model | Criticality | Evidence |
|--------|------|---------|------------|-------------|----------|
| TMDB API | REST API | Movie/TV metadata lookup (title, year, genre, cast, external IDs) | API key (Bearer token) | High — required for metadata | [tmdb.ts](../../src/lib/server/tmdb.ts) |
| Jikan (MAL) API | REST API | Anime ID lookup for MyAnimeList cross-referencing | None (public API) | Medium — anime uploads only | [jikan.ts](../../src/lib/server/jikan.ts) |
| Private torrent tracker APIs | REST API (per-tracker) | Upload torrent + metadata, search existing releases, download torrents | API key (per tracker) | High — core purpose of app | [trackers/](../../src/lib/server/trackers/) |
| Image hosting services (7) | REST API (per-host) | Upload screenshots for torrent descriptions | API key (varies by host) | High — screenshots required for most uploads | [image-hosts/](../../src/lib/server/image-hosts/) |
| qBittorrent | REST API | Send completed torrents to torrent client for seeding | Session cookie (login) | Medium — optional auto-seeding | [torrent-clients/qbittorrent.ts](../../src/lib/server/torrent-clients/qbittorrent.ts) |
| ffmpeg | CLI subprocess | Extract screenshots from video files | N/A (local binary) | High — screenshot generation | [screenshots.ts](../../src/lib/server/screenshots.ts) |
| ffprobe | CLI subprocess | Detect video duration for screenshot timing | N/A (local binary) | High — used by screenshots | [screenshots.ts](../../src/lib/server/screenshots.ts) |
| mkbrr | CLI subprocess | Create and edit .torrent files | N/A (local binary) | High — torrent creation | [torrent.ts](../../src/lib/server/torrent.ts) |
| MediaInfo (WASM) | In-process library | Analyze video/audio codec, resolution, HDR, channels | N/A (mediainfo.js) | High — codec detection | [mediainfo.ts](../../src/lib/server/mediainfo.ts) |

### 2) Data Stores

| Store | Role | Access layer | Key risk | Evidence |
|-------|------|--------------|----------|----------|
| `settings.json` (filesystem) | Persists all user configuration (tracker keys, image host settings, general prefs) | `Settings` singleton via `Bun.file()` | Corruption on concurrent writes; plaintext API keys on disk | [settings.ts](../../src/lib/server/settings.ts) |
| In-memory Maps | Active uploads, sessions, caches (TMDB, screenshots, tracker data) | Direct Map access in singletons | All state lost on restart | [uploads.ts](../../src/lib/server/uploads.ts), [sessions.ts](../../src/lib/server/sessions.ts) |
| Temporary files (filesystem) | Screenshot PNGs, torrent files during creation | `Bun.file()` + cleanup in `close()` | Temp files may leak if process crashes | [screenshots.ts](../../src/lib/server/screenshots.ts), [torrent.ts](../../src/lib/server/torrent.ts) |

### 3) Secrets and Credentials Handling

- **Credential sources**: All stored in `settings.json` file on disk (API keys for trackers, image hosts, TMDB; torrent client passwords)
- **Hardcoding checks**: No hardcoded credentials found in source code
- **Auth token**: App generates a random `authToken` on first boot via `randomBytes(32).toString('base64url')`, stored in settings
- **Timing-safe comparison**: `timingSafeEqual()` used for auth token and API key verification in [hooks.server.ts](../../src/hooks.server.ts)
- **Rotation/lifecycle**: No credential rotation mechanism. Tokens persist until manually changed.
- **Risk**: API keys stored as plaintext in settings.json — no encryption at rest

### 4) Reliability and Failure Behavior

- **Retry/backoff**: Image host uploads have fallback strategy (tries hosts in priority order, skips failed ones) but no retry on the same host. No general retry mechanism.
- **Rate limiting**:
  - TMDB: PQueue concurrency 1 ([tmdb.ts](../../src/lib/server/tmdb.ts))
  - Jikan: PQueue concurrency 1, 1 request/second interval ([jikan.ts](../../src/lib/server/jikan.ts))
  - Torrent operations: PQueue concurrency 1 for both create and edit ([torrent.ts](../../src/lib/server/torrent.ts))
  - Screenshots: PQueue concurrency 1 ([screenshots.ts](../../src/lib/server/screenshots.ts))
- **Timeout policy**: No explicit HTTP timeouts configured on `fetch()` calls. AbortSignal used for user-initiated cancellation only.
- **Circuit-breaker**: None. Image host fallback is the closest pattern.

### 5) Observability for Integrations

- **Logging around external calls**: Yes — `log()` used around TMDB/Jikan/tracker/image-host/torrent-client calls with color-coded severity
- **Metrics/tracing coverage**: None — no APM, no Prometheus, no OpenTelemetry
- **Missing visibility gaps**:
  - No HTTP request/response timing metrics
  - No error rate tracking per integration
  - No health checks for external dependencies
  - No structured logging for aggregation

### 6) Evidence

- [src/lib/server/tmdb.ts](../../src/lib/server/tmdb.ts)
- [src/lib/server/jikan.ts](../../src/lib/server/jikan.ts)
- [src/lib/server/trackers/lst.ts](../../src/lib/server/trackers/lst.ts)
- [src/lib/server/image-hosts/](../../src/lib/server/image-hosts/)
- [src/lib/server/torrent-clients/qbittorrent.ts](../../src/lib/server/torrent-clients/qbittorrent.ts)
- [src/lib/server/screenshots.ts](../../src/lib/server/screenshots.ts)
- [src/lib/server/torrent.ts](../../src/lib/server/torrent.ts)
- [src/lib/server/mediainfo.ts](../../src/lib/server/mediainfo.ts)
- [src/lib/server/settings.ts](../../src/lib/server/settings.ts)
- [src/hooks.server.ts](../../src/hooks.server.ts)

## Extended Sections (Optional)

### Tracker Integrations Detail

| Tracker | Type | API Pattern | Evidence |
|---------|------|-------------|----------|
| Aither | Unit3D-based | REST API with JSON body + multipart torrent | [trackers/aither.ts](../../src/lib/server/trackers/aither.ts) |
| BeyondHD | Custom | REST API | [trackers/beyondhd.ts](../../src/lib/server/trackers/beyondhd.ts) |
| LST | Unit3D-based | REST API with JSON body + multipart torrent | [trackers/lst.ts](../../src/lib/server/trackers/lst.ts) |
| MidnightScene | Unit3D-based | REST API | [trackers/midnightscene.ts](../../src/lib/server/trackers/midnightscene.ts) |
| Seedpool | Unit3D-based | REST API | [trackers/seedpool.ts](../../src/lib/server/trackers/seedpool.ts) |

### Image Host Integrations Detail

| Host | Has API Key | Max Size | Evidence |
|------|-------------|----------|----------|
| Catbox | No | Varies | [image-hosts/catbox.ts](../../src/lib/server/image-hosts/catbox.ts) |
| Freeimage.host | Yes | Varies | [image-hosts/freeimage-host.ts](../../src/lib/server/image-hosts/freeimage-host.ts) |
| ImgBB | Yes | Varies | [image-hosts/imgbb.ts](../../src/lib/server/image-hosts/imgbb.ts) |
| imgbox | No | Varies | [image-hosts/imgbox.ts](../../src/lib/server/image-hosts/imgbox.ts) |
| PiXhost | No | Varies | [image-hosts/pixhost.ts](../../src/lib/server/image-hosts/pixhost.ts) |
| ptpimg | Yes | Varies | [image-hosts/ptpimg.ts](../../src/lib/server/image-hosts/ptpimg.ts) |
| Zipline | Yes (+ server URL) | Varies | [image-hosts/zipline.ts](../../src/lib/server/image-hosts/zipline.ts) |
