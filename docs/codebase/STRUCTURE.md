# Codebase Structure

## Core Sections (Required)

### 1) Top-Level Map

| Path | Purpose | Evidence |
|------|---------|----------|
| `src/lib/server/` | Core backend logic: trackers, upload orchestration, external tool wrappers, settings | All server-side `.ts` files |
| `src/lib/server/trackers/` | One file per tracker implementation + registry | [trackers/index.ts](../../src/lib/server/trackers/index.ts) |
| `src/lib/server/image-hosts/` | One file per image hosting service + registry | [image-hosts/index.ts](../../src/lib/server/image-hosts/index.ts) |
| `src/lib/server/torrent-clients/` | One file per torrent client + registry | [torrent-clients/index.ts](../../src/lib/server/torrent-clients/index.ts) |
| `src/lib/server/util/` | Shared utilities: logging, error formatting, normalization | [util/log.ts](../../src/lib/server/util/log.ts) |
| `src/lib/` | Shared types, client-side utilities, assets | [types.ts](../../src/lib/types.ts) |
| `src/lib/util/` | Client-side utilities (get-why, insert-wbr, tracker-name-to-id) | [src/lib/util/](../../src/lib/util/) |
| `src/routes/` | SvelteKit routes — pages and API endpoints | [src/routes/](../../src/routes/) |
| `src/routes/api/` | REST API: `POST /api/upload`, `POST /api/preview` | [api/upload/+server.ts](../../src/routes/api/upload/+server.ts) |
| `src/routes/uploads/` | Main upload management UI (list + detail pages) | [uploads/+page.svelte](../../src/routes/uploads/+page.svelte) |
| `src/routes/settings/` | Settings UI (trackers, image hosts, torrent clients, general) | [settings/+page.svelte](../../src/routes/settings/+page.svelte) |
| `src/routes/login/` | Authentication page | [login/+page.svelte](../../src/routes/login/+page.svelte) |
| `src/routes/open/` | File browser for selecting content to upload | [open/+page.svelte](../../src/routes/open/+page.svelte) |
| `static/` | Static assets (logo, robots.txt) | [static/](../../static/) |
| `assets/` | Design assets (logo.ai) | [assets/](../../assets/) |
| `.github/workflows/` | CI/CD — GitHub Actions release workflow | [release.yml](../../.github/workflows/release.yml) |
| `.claude/skills/` | Claude AI skill definitions for codebase tasks | [.claude/skills/](../../.claude/skills/) |

### 2) Entry Points

- **Main runtime entry**: SvelteKit app — `src/hooks.server.ts` is the server entry point (startup + request handling)
- **Production entry**: `bun build/index.js` (output of `bun run build`)
- **Dev entry**: `bunx --bun vite dev` (via `bun run dev`)
- **API entry**: `POST /api/upload` at [src/routes/api/upload/+server.ts](../../src/routes/api/upload/+server.ts)
- **No CLI entry point** — web-only application
- **How entry is selected**: SvelteKit routing via filesystem convention, adapter-bun for production

### 3) Module Boundaries

| Boundary | What belongs here | What must not be here |
|----------|-------------------|------------------------|
| `src/lib/server/` | All backend logic, external API calls, file I/O, process spawning | Svelte components, client-side rendering logic |
| `src/lib/server/trackers/` | Individual tracker implementations (one class per file) | Upload orchestration, shared business logic |
| `src/lib/server/image-hosts/` | Image host upload implementations (one class per file) | Tracker logic, UI code |
| `src/lib/server/torrent-clients/` | Torrent client send implementations | Tracker or upload logic |
| `src/lib/server/util/` | Pure utility functions (logging, error formatting, normalization) | Business logic, stateful singletons |
| `src/lib/types.ts` | Shared types and Valibot schemas used across client and server | Implementation code, server-only imports |
| `src/routes/` | Page components, server load functions, API route handlers | Core business logic (should delegate to `lib/server/`) |
| `src/routes/uploads/[id]/` | Per-upload UI components and sub-routes | Cross-upload logic |

### 4) Naming and Organization Rules

- **File naming**: kebab-case for all files (`error-string.ts`, `upload-screenshots.ts`, `torrent-client.ts`)
- **Directory organization**: Hybrid layer + feature — `server/` separates backend, then sub-directories group by domain (trackers, image-hosts, torrent-clients)
- **Import aliasing**: `$lib/` maps to `src/lib/` (SvelteKit standard)
- **Route naming**: SvelteKit conventions — `+page.svelte`, `+page.server.ts`, `+server.ts`, `+layout.svelte`
- **Component files**: PascalCase Svelte components within route directories (`Tracker.svelte`, `Screenshots.svelte`)
- **Registry pattern**: Each plugin directory has an `index.ts` that exports a `Record<string, ...>` mapping names to implementations

### 5) Evidence

- [src/hooks.server.ts](../../src/hooks.server.ts)
- [src/lib/server/trackers/index.ts](../../src/lib/server/trackers/index.ts)
- [src/lib/server/image-hosts/index.ts](../../src/lib/server/image-hosts/index.ts)
- [src/lib/server/torrent-clients/index.ts](../../src/lib/server/torrent-clients/index.ts)
- [src/lib/types.ts](../../src/lib/types.ts)
- [src/routes/api/upload/+server.ts](../../src/routes/api/upload/+server.ts)
