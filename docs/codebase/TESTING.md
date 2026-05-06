# Testing Patterns

## Core Sections (Required)

### 1) Test Stack and Commands

- **Primary test framework**: None — no test framework is configured
- **Assertion/mocking tools**: None
- **Commands**:

```bash
bun run check          # Type checking only (svelte-kit sync + svelte-check)
bun run check:watch    # Type checking in watch mode
```

There is no `test` script in [package.json](../../package.json).

### 2) Test Layout

- **No test files exist** anywhere in the codebase
- **No test directories** (`__tests__/`, `tests/`, `test/`, `*.test.ts`, `*.spec.ts`)
- **No test configuration files** (`jest.config.*`, `vitest.config.*`, etc.)

### 3) Test Scope Matrix

| Scope | Covered? | Typical target | Notes |
|-------|----------|----------------|-------|
| Unit | No | — | No test framework installed |
| Integration | No | — | No test framework installed |
| E2E | No | — | No browser testing framework |

### 4) Mocking and Isolation Strategy

- **No mocking infrastructure**
- **Verification method**: Type checking (`bun run check`) and manual testing only
- **Per CLAUDE.md**: "There are no automated tests in this codebase. Type checking and manual testing are the current verification methods. Don't add a test framework without discussing it first."

### 5) Coverage and Quality Signals

- **Coverage tool**: None
- **Current coverage**: 0% (no tests)
- **Quality signals**: TypeScript strict mode with additional strictness flags provides compile-time safety. No runtime test coverage.
- **Known gaps**: All runtime behavior, API integration correctness, release parsing edge cases, tracker upload flows

### 6) Evidence

- [package.json](../../package.json) — no test script or test dependencies
- [CLAUDE.md](../../CLAUDE.md) — explicitly documents no-test policy
- [tsconfig.json](../../tsconfig.json) — strict mode as primary quality gate
