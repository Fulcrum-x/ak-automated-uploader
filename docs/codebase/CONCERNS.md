# Codebase Concerns

## Core Sections (Required)

### 1) Top Risks (Prioritized)

| Severity | Concern | Evidence | Impact | Suggested action |
|----------|---------|----------|--------|------------------|
| High | No automated tests | [package.json](../../package.json), [CLAUDE.md](../../CLAUDE.md) | Regressions in release parsing, tracker uploads, or metadata mapping go undetected until manual testing | [ASK USER] — intentional decision, but release parsing regex complexity (800+ lines) is a strong candidate for unit tests |
| High | All state is in-memory | [uploads.ts](../../src/lib/server/uploads.ts), [sessions.ts](../../src/lib/server/sessions.ts) | Process restart loses all active uploads and sessions; users must re-authenticate and re-create uploads | Consider persisting active upload state to disk or add a lightweight store |
| Medium | Plaintext API keys on disk | [settings.ts](../../src/lib/server/settings.ts) | Credential exposure if settings.json is accessed by unauthorized party | Acceptable for self-hosted tool; consider filesystem permissions documentation |
| Medium | No HTTP timeouts on fetch() | Various tracker/API client files | Hung connections can block PQueue slots indefinitely | Add AbortSignal with timeout to all external HTTP calls |
| Medium | Large monolithic files | `unit3d-distributors.ts` (50KB), `release.ts` (33.6KB) | Difficult to navigate and maintain | `release.ts` complexity is inherent to parsing; `unit3d-distributors.ts` is data-heavy (may be generated) |
| Low | Windows FFI coupling | [torrent.ts](../../src/lib/server/torrent.ts) | Platform-specific code for process suspension; potential FFI stability issues | Already has Unix fallback via SIGSTOP/SIGCONT; Windows path is necessary for Bun on Windows |

### 2) Technical Debt

| Debt item | Why it exists | Where | Risk if ignored | Suggested fix |
|-----------|---------------|-------|-----------------|---------------|
| No linter or formatter | Not yet configured | Project root (no `.eslintrc`, `.prettierrc`) | Style inconsistencies accumulate as contributors increase | [ASK USER] — intentional omission or planned addition? |
| In-memory sessions | Simplest implementation | [sessions.ts](../../src/lib/server/sessions.ts) | Users must re-login after every restart | Persist sessions to settings.json or a separate file |
| No `.env.example` | Config managed through UI/settings.json | Project root | New developers don't know what external deps/config is needed | Document required external binaries and optional env vars |
| Observer pattern without cleanup | Most `onX()` callbacks have no `offX()` removal | [tracker.ts](../../src/lib/server/tracker.ts), [upload.ts](../../src/lib/server/upload.ts) | Potential memory leaks if objects are long-lived (mitigated by upload lifecycle) | Acceptable given upload objects are short-lived and cleaned up via `close()` |

### 3) Security Concerns

| Risk | OWASP category | Evidence | Current mitigation | Gap |
|------|----------------|----------|--------------------|-----|
| Plaintext credential storage | A02 Cryptographic Failures | [settings.ts](../../src/lib/server/settings.ts) | Self-hosted, single-user app | No encryption at rest for API keys |
| Session fixation on restart | A07 Identification Failures | [sessions.ts](../../src/lib/server/sessions.ts) | Random 32-byte tokens, 7-day rolling expiry, `timingSafeEqual` | Sessions lost on restart (inconvenience, not a vulnerability) |
| No CSRF protection on forms | A01 Broken Access Control | [hooks.server.ts](../../src/hooks.server.ts) | SvelteKit CSP with `mode: 'auto'` | SvelteKit handles CSRF by default via origin checking |
| Subprocess injection | A03 Injection | [torrent.ts](../../src/lib/server/torrent.ts), [screenshots.ts](../../src/lib/server/screenshots.ts) | Arguments passed as arrays to `Bun.spawn()` (not shell interpolation) | Safe — array-based spawn prevents shell injection |
| API key in query params | A04 Insecure Design | [hooks.server.ts](../../src/hooks.server.ts) `?apiKey=` | Also accepts `Authorization: Bearer` header | Query param keys may appear in server logs; prefer header-only |

### 4) Performance and Scaling Concerns

| Concern | Evidence | Current symptom | Scaling risk | Suggested improvement |
|---------|----------|-----------------|-------------|-----------------------|
| Sequential screenshot extraction | [screenshots.ts](../../src/lib/server/screenshots.ts) PQueue concurrency: 1 | Slow for many screenshots | Bottleneck on uploads with many files | Intentional — prevents disk I/O contention with torrent hashing |
| Cooperative hashing suspension | [torrent.ts](../../src/lib/server/torrent.ts) `pauseHashing()` | ffmpeg/mediainfo/file-browser pause all torrent hashing globally | Multiple concurrent uploads contend for hashing time | Acceptable tradeoff for single-user app |
| WASM MediaInfo in-process | [mediainfo.ts](../../src/lib/server/mediainfo.ts) | Blocks event loop during analysis of large files | Could cause request timeouts under load | Acceptable for single-user; would need worker threads for multi-user |
| No request queuing for uploads | [uploads.ts](../../src/lib/server/uploads.ts) | All uploads start immediately | Concurrent uploads compete for CPU, disk, and network | Add a global upload queue with configurable concurrency |

### 5) Fragile/High-Churn Areas

| Area | Why fragile | Churn signal | Safe change strategy |
|------|-------------|-------------|----------------------|
| [trackers/aither.ts](../../src/lib/server/trackers/aither.ts) | 8 changes in 90 days; complex field mappings | Highest churn source file | Test field mapping with sample releases before deploying |
| [types.ts](../../src/lib/types.ts) | 8 changes in 90 days; shared across client/server | Core type definitions | Run `bun run check` after any change; changes cascade |
| [trackers/lst.ts](../../src/lib/server/trackers/lst.ts) | 7 changes in 90 days; reference Unit3D implementation | Template for new trackers | Verify both LST and trackers that copy its pattern |
| [tracker.ts](../../src/lib/server/tracker.ts) | 6 changes in 90 days; abstract base class | Changes affect all trackers | Verify all tracker implementations after base class changes |
| [release.ts](../../src/lib/server/release.ts) | Regex patterns match backwards from end of filename; 800+ lines | Complex parsing logic | Any regex change can break unrelated parsing; test with diverse filenames |

### 6) `[ASK USER]` Questions

1. [ASK USER] Is the lack of automated tests an intentional long-term decision, or is a test framework planned? Release parsing (`release.ts`) has high complexity that would benefit from regression tests.
2. [ASK USER] Is the absence of a linter/formatter intentional, or would adding one (e.g., Biome, ESLint + Prettier) be welcome?
3. [ASK USER] Should the `?apiKey=` query parameter authentication be removed in favor of header-only auth to prevent key leakage in logs?
4. [ASK USER] Is there a plan to add persistence for active uploads (surviving process restarts)?

### 7) Evidence

- Scan output: [.codebase-scan.txt](.codebase-scan.txt)
- [src/lib/server/settings.ts](../../src/lib/server/settings.ts)
- [src/lib/server/sessions.ts](../../src/lib/server/sessions.ts)
- [src/lib/server/torrent.ts](../../src/lib/server/torrent.ts)
- [src/lib/server/release.ts](../../src/lib/server/release.ts)
- [src/lib/server/uploads.ts](../../src/lib/server/uploads.ts)
- [src/hooks.server.ts](../../src/hooks.server.ts)
- [CLAUDE.md](../../CLAUDE.md)
