# Coding Conventions

## Core Sections (Required)

### 1) Naming Rules

| Item | Rule | Example | Evidence |
|------|------|---------|----------|
| Files | kebab-case | `error-string.ts`, `upload-screenshots.ts`, `torrent-client.ts` | All files in `src/lib/server/` |
| Svelte components | PascalCase | `Tracker.svelte`, `Screenshots.svelte`, `FormControl.svelte` | `src/routes/uploads/[id]/` |
| Functions/methods | camelCase | `errorString()`, `getMalId()`, `sendTorrent()`, `applyMetadata()` | Throughout codebase |
| Classes | PascalCase | `Tracker`, `Upload`, `Release`, `Screenshots` | `src/lib/server/*.ts` |
| Types/interfaces | PascalCase | `TrackerField`, `ReleaseState`, `FieldsToType<T>` | [types.ts](../../src/lib/types.ts) |
| Private fields | `_` prefix | `_title`, `_contentPath`, `_videoCodec` | [release.ts](../../src/lib/server/release.ts) |
| Constants | camelCase or UPPER_SNAKE for true constants | `cacheTTL`, `RETAKE_THRESHOLD`, `UPLOAD_URL` | Various server files |
| No abbreviations | Spelled out except standard terms | `const response = await fetch(...)` not `const res = ...`; `cacheTTL`, `toJSON` acceptable | CLAUDE.md |

### 2) Formatting and Linting

- **Formatter**: None configured (no Prettier, no Biome)
- **Linter**: None configured (no ESLint)
- **TypeScript strictness** ([tsconfig.json](../../tsconfig.json)):
  - `strict: true`
  - `noUncheckedIndexedAccess: true`
  - `noImplicitOverride: true`
  - `noImplicitReturns: true`
  - `noFallthroughCasesInSwitch: true`
  - `verbatimModuleSyntax: true` (enforces `import type`)
  - `allowImportingTsExtensions: true`
  - `rewriteRelativeImportExtensions: true`
- **Verification command**: `bun run check` (svelte-kit sync + svelte-check)

### 3) Import and Module Conventions

- **`import type`**: Required for type-only imports (enforced by `verbatimModuleSyntax`). Svelte 5 requirement.
- **Path alias**: `$lib/` maps to `src/lib/` (SvelteKit standard). Used for cross-module imports.
- **Relative imports**: Used within the same module/directory (e.g., `./util/log`, `../tracker`)
- **Barrel exports**: Registry files (`trackers/index.ts`, `image-hosts/index.ts`, `torrent-clients/index.ts`) export a single record mapping names to implementations
- **Default exports**: Classes use default export. Singletons use default export of instance. Tracker files additionally export named `settings` and `fields`.
- **Import grouping**: No enforced order (no linter rule). General pattern: external deps first, then `$lib/` imports, then relative imports.

### 4) Error and Logging Conventions

- **Error strategy**: `errorString(description, error)` wraps all errors with context, chaining messages with ` ⮞ ` separator. Pattern: `catch (error) { throw Error(errorString("Couldn't X", error)); }`
  - `ValiError` → auto-flattened via `v.flatten()`
  - `Error` → recursively follows `.cause` chain
  - `string` → appended directly
  - No custom error classes — all plain `Error` with descriptive messages
- **Logging**: `log(message, color?)` from [util/log.ts](../../src/lib/server/util/log.ts)
  - `'tomato'` → fatal error (stderr)
  - `'khaki'` → warning (stderr)
  - `'aquamarine'` → success (stdout)
  - `'lightgrey'` → informational (stdout, default)
  - Format: `[HH:MM:SS] [ak-automated-uploader] message` with ANSI 16M colors via Bun's `color()`
  - No log levels, no structured logging, no file output
- **Sensitive-data redaction**: API keys stored in settings singleton, not logged. Auth tokens compared with `timingSafeEqual()`.

### 5) Testing Conventions

- **No test framework** — explicitly stated in CLAUDE.md
- **Verification**: Type checking (`bun run check`) and manual testing only
- **No test files, no test directories, no coverage tooling**

### 6) Evidence

- [tsconfig.json](../../tsconfig.json)
- [src/lib/server/util/log.ts](../../src/lib/server/util/log.ts)
- [src/lib/server/util/error-string.ts](../../src/lib/server/util/error-string.ts)
- [CLAUDE.md](../../CLAUDE.md)
- [package.json](../../package.json)

## Extended Sections (Optional)

### Comments Policy

- **No comments by default** — only add a comment if the *why* is genuinely non-obvious. Do not narrate what the code does. (Per CLAUDE.md)

### Validation Convention

- **Valibot** (`import * as v from 'valibot'`) for all external input validation
- Schemas defined near the code that uses them
- `v.parse()` for runtime validation, `v.InferOutput<>` for deriving types from schemas
- `buildSchemaFromFields()` utility dynamically constructs Valibot schemas from tracker/host field definitions

### Concurrency Convention

- **AbortSignal** threaded through all async operations for cancellation
- **PQueue** for concurrency control (never unbounded concurrency)
- **Cooperative hashing suspension** via `pauseHashing()`/`resumeHashing()` refcount pattern

### State Serialization Convention

- Every major class implements `toJSON()` or `getXState()` methods
- Used for SSE streaming and SvelteKit page load data
- Partial serialization supported via key filter parameter
