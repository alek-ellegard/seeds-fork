---
last_mapped: 2026-03-01T00:00:00Z
total_files: 64
total_tokens: 68361
---

# Codebase Map

## System Overview

```
┌─────────────────────────────────────────────────────┐
│                    CLI (sd)                          │
│              src/index.ts                            │
│   Commander router · typo suggest · --timing         │
└──────────────┬──────────────────────────────────────┘
               │ lazy import
┌──────────────▼──────────────────────────────────────┐
│              Commands (src/commands/)                │
│  init · create · show · list · ready · update       │
│  close · dep · sync · blocked · stats · tpl         │
│  migrate · doctor · prime · onboard · upgrade       │
│  completions                                        │
└──────┬────────┬────────┬────────┬───────────────────┘
       │        │        │        │
┌──────▼──┐ ┌──▼────┐ ┌─▼────┐ ┌─▼──────┐
│ store.ts│ │config │ │id.ts │ │output  │
│ JSONL   │ │.ts    │ │ ID   │ │.ts     │
│ r/w/lock│ │find   │ │ gen  │ │ chalk  │
└────┬────┘ │seeds  │ └──────┘ │ format │
     │      │dir    │          └────────┘
     │      └───┬───┘
     │          │
┌────▼──────────▼─────────────────────────────────────┐
│              .seeds/ (on disk)                       │
│  config.yaml · issues.jsonl · templates.jsonl       │
│  Advisory locks · Atomic writes · merge=union       │
└─────────────────────────────────────────────────────┘
```

**Supporting modules:** `yaml.ts` (flat YAML parser), `markers.ts` (HTML comment section helpers), `types.ts` (interfaces + constants)

## Directory Structure

```
seeds-fork/
├── src/
│   ├── index.ts              # CLI entry, Commander router, VERSION, typo suggest
│   ├── types.ts              # Issue/Template/Config interfaces, constants
│   ├── store.ts              # JSONL read/write, advisory locks, atomic writes
│   ├── id.ts                 # Collision-resistant ID generation ({prefix}-{4hex})
│   ├── config.ts             # Config I/O, .seeds/ discovery, worktree awareness
│   ├── output.ts             # Chalk formatting, --json/--quiet output helpers
│   ├── yaml.ts               # Minimal flat key-value YAML parser (~50 LOC)
│   ├── markers.ts            # <!-- seeds:start/end --> section helpers
│   ├── *.test.ts             # Colocated unit tests (store, id, config, yaml, markers)
│   └── commands/
│       ├── init.ts           # Bootstrap .seeds/ directory
│       ├── create.ts         # Create issue (append under lock)
│       ├── show.ts           # Display single issue
│       ├── list.ts           # Filter/list issues (default: open/in_progress)
│       ├── ready.ts          # Open issues with no unresolved blockers
│       ├── update.ts         # Patch issue fields (rewrite under lock)
│       ├── close.ts          # Close issues (batch, single transaction)
│       ├── dep.ts            # Bidirectional dependency management
│       ├── sync.ts           # Git stage+commit .seeds/, worktree guard
│       ├── blocked.ts        # Issues with unresolved blockers
│       ├── stats.ts          # Aggregate project statistics
│       ├── tpl.ts            # Template/molecule system (create, pour, convoy)
│       ├── migrate.ts        # Import from .beads/ format
│       ├── doctor.ts         # 9 health checks with --fix
│       ├── prime.ts          # AI agent context output
│       ├── onboard.ts        # Inject seeds section into CLAUDE.md/AGENTS.md
│       ├── upgrade.ts        # npm self-upgrade
│       ├── completions.ts    # Shell completion scripts (bash/zsh/fish)
│       └── *.test.ts         # Colocated integration tests (CLI subprocess)
├── scripts/
│   └── version-bump.ts       # Sync version across package.json + src/index.ts
├── .seeds/                   # Issue data (JSONL, self-hosted)
├── .overstory/               # Multi-agent orchestration config
├── .mulch/                   # Expertise management config
├── .github/workflows/
│   ├── ci.yml                # lint + typecheck + test
│   └── publish.yml           # Version-gated npm publish + git tag + GH release
├── biome.json                # Tabs, 100 char width, double quotes
├── tsconfig.json             # Strict, noUncheckedIndexedAccess, noEmit
└── package.json              # @os-eco/seeds-cli, bin: sd
```

## Module Guide

### Core: src/index.ts

- **Purpose**: CLI entry point and Commander router
- **Key exports**: `VERSION` ("0.2.4")
- **External deps**: chalk, commander
- **Behavior**: Lazy-loads all commands via `Promise.all(import(...))`. Each command module exports `register(program)`. Handles `--version --json` before Commander parses. Implements Levenshtein distance for typo suggestions (threshold ≤ 2). `--timing` via Commander preAction/postAction hooks.

### Core: src/store.ts

- **Purpose**: JSONL data layer with concurrency safety
- **Key exports**: `withLock`, `readIssues`, `writeIssues`, `appendIssue`, `readTemplates`, `writeTemplates`, `appendTemplate`
- **External deps**: node:crypto, node:fs, node:path, Bun built-ins
- **Patterns**:
  - Advisory locking: `O_CREAT | O_EXCL` on `.lock` sidecar, 30s stale, jittered retry
  - Atomic writes: temp file + `renameSync`
  - Dedup-on-read: `Map` where last occurrence wins (handles `merge=union` conflicts)
  - Two write strategies: `appendIssue` (create) vs `writeIssues` (full rewrite for mutations)

### Core: src/config.ts

- **Purpose**: Config I/O and `.seeds/` directory discovery
- **Key exports**: `readConfig`, `writeConfig`, `findSeedsDir`, `isInsideWorktree`
- **Behavior**: Walks up directory tree to find `.seeds/config.yaml`. Detects git worktrees via `git rev-parse --git-common-dir` and redirects to main worktree's `.seeds/` so all worktrees share one issue store.

### Core: src/types.ts

- **Purpose**: All shared TypeScript interfaces and domain constants
- **Key exports**: `Issue`, `Template`, `Config`, `SEEDS_DIR_NAME`, `VALID_TYPES`, `VALID_STATUSES`, `PRIORITY_LABELS`, lock timing constants
- **External deps**: none (pure types + constants)

### Core: src/id.ts

- **Purpose**: Collision-resistant ID generation
- **Key exports**: `generateId(prefix, existingIds)`
- **Format**: `{prefix}-{4hex}`, falls back to 8-hex after 100 collisions

### Core: src/output.ts

- **Purpose**: Terminal output formatting
- **Key exports**: `brand`, `accent`, `muted` (chalk colors), `setQuiet`, `outputJson`, `printSuccess`, `printError`, `printIssueFull`
- **Palette**: brand=#7CB342, accent=#FFB74D, muted=#78786E
- **Note**: Module-level `_quiet` boolean — shared mutable state

### Core: src/yaml.ts

- **Purpose**: Minimal flat key-value YAML parser (no nesting, no arrays)
- **Key exports**: `parseYaml`, `stringifyYaml`

### Core: src/markers.ts

- **Purpose**: HTML comment marker section helpers for `sd onboard`
- **Key exports**: `hasMarkerSection`, `replaceMarkerSection`, `wrapInMarkers`
- **Markers**: `<!-- seeds:start -->` / `<!-- seeds:end -->`

### Commands Pattern

All commands follow a consistent contract:
- Export `run(args: string[], seedsDir?: string)` for programmatic/test invocation
- Export `register(program: Command)` for Commander wiring
- Exception: `completions.ts` exports only `register` (no standalone `run`)

**Write commands** (create, update, close, dep, tpl pour): acquire lock via `withLock`
**Read commands** (show, list, ready, blocked, stats): no lock needed

### Commands: Notable Behaviors

| Command | Key behavior |
|---------|-------------|
| `create` | `appendIssue` (append-only under lock) |
| `update` | `writeIssues` (full rewrite under lock) |
| `close` | Batch close in single transaction, all-or-nothing |
| `dep` | Bidirectional links (blockedBy + blocks) updated atomically |
| `tpl pour` | Creates multiple issues in single lock, wires sequential blockedBy chain |
| `sync` | Shells out to git, skips commit in worktrees |
| `doctor` | 9 checks, `--fix` writes via `writeFileSync` bypassing lock |
| `onboard` | Version detection via `<!-- seeds-onboard-v:N -->`, idempotent updates |

## Data Flow

### Issue Lifecycle

```
sd create --title "..."
  → findSeedsDir() → walk up dirs, resolve worktree
  → withLock(issues.jsonl.lock)
    → readIssues() → parse JSONL, dedup by ID
    → generateId(prefix, existingIds)
    → appendIssue() → read + concat + atomic write
  → release lock
  → outputJson / printSuccess

sd update <id> --status in_progress
  → withLock(issues.jsonl.lock)
    → readIssues() → find by ID → patch fields
    → writeIssues() → atomic rewrite
  → release lock

sd close <id>
  → withLock(issues.jsonl.lock)
    → readIssues() → find all IDs → set closed + timestamps
    → writeIssues() → atomic rewrite
  → release lock
```

### Template Pour

```
sd tpl pour <tpl-id> --prefix "Auth"
  → withLock(issues.jsonl.lock)
    → readTemplates() → find template + steps
    → For each step:
      → generateId()
      → Create issue with {prefix} interpolation
      → Wire blockedBy: step[i+1].blockedBy = [step[i].id]
    → appendIssue() for each
  → release lock
```

### Sync Flow

```
sd sync
  → findSeedsDir()
  → isInsideWorktree() → if yes, warn and skip
  → git add .seeds/
  → git diff --cached --quiet
  → git commit -m "seeds: sync YYYY-MM-DD"
```

## Conventions

### Code Patterns

- **Tab indentation**, 100 char line width (Biome enforced)
- **No `any`** — use `unknown` and narrow, or define types in `types.ts`
- **`noUncheckedIndexedAccess`** — all index access returns `T | undefined`, use `!` assertion as escape hatch (Biome allows it)
- **Bun-native APIs** — `Bun.file()`, `Bun.write()`, `Bun.spawnSync()`
- **No build step** — Bun runs TypeScript directly, `package.json` bin points to `.ts`

### Testing Patterns

- **Real I/O, no mocks** — temp dirs via `mkdtemp`, real filesystem operations
- **Integration tests via CLI subprocess** — `Bun.spawn(["bun", "run", "src/index.ts", ...args])`
- **`runJson<T>()` helper** — appends `--json`, parses stdout
- **Colocated tests** — `{module}.test.ts` next to source

### Dependency Rules

- Runtime: only chalk + commander
- All shared types in `src/types.ts`
- Each command in its own file under `src/commands/`
- Core modules at `src/` root

## Gotchas

1. **Dedup-on-read, not dedup-on-write** — `appendIssue` doesn't check for duplicates. After `merge=union` git merges, duplicates are resolved silently on next read (last occurrence wins).

2. **`writeConfig` is NOT atomic** — Unlike `writeIssues`/`writeTemplates`, config writes use `Bun.write` directly with no temp-file-rename and no lock. Crash mid-write can corrupt `config.yaml`.

3. **Worktree redirect is transparent** — `findSeedsDir` silently redirects to the main worktree's `.seeds/`. All worktrees share one issue store without configuration.

4. **`ready` excludes `in_progress`** — Only `open` status issues appear. Started work (`in_progress`) disappears from the ready list.

5. **`list` default hides closed** — Without `--all`, closed issues are omitted. Use `--status closed` or `--all`.

6. **`dep remove` sets arrays to `undefined`** — Not empty `[]`. All consumers must handle absent keys.

7. **`doctor --fix` bypasses the lock** — Writes via `writeFileSync` directly. Unsafe if another process is writing concurrently.

8. **Quiet flag is global mutable state** — `setQuiet(true)` mutates module-level variable in `output.ts`. Tests sharing this module must be aware of state leakage.

9. **Version in two places** — `package.json` and `src/index.ts` must stay in sync. CI verifies. Bump script uses brittle string replacement.

10. **`tpl pour` creates sequential chains only** — No parallel template instantiation. Step N+1 is always blockedBy step N.

## Test Coverage

| Area | Status | Notes |
|------|--------|-------|
| CRUD (create/show/list/update/close) | Strong | Full integration via CLI subprocess |
| Dependencies (dep add/remove/list) | Strong | Bidirectional consistency verified |
| Templates (tpl create/step/pour/status) | Strong | Convoy chain tested |
| Doctor checks (9 categories) | Strong | Pass/fail/fix paths for each |
| Store (JSONL r/w, locks, dedup) | Strong | Concurrency test via Promise.all |
| Config (findSeedsDir, worktree) | Strong | Real git worktree setup in tests |
| CLI flags (--json, --timing, --quiet) | Good | Channel separation verified |
| Onboard (idempotency, version upgrade) | Good | Version marker tested |
| Shell completions | Good | All three shells tested |
| migrate-from-beads | None | No test file |
| upgrade | None | No test file |
| --quiet / --verbose flags | None | Not directly tested |

## Ecosystem Integration

### Overstory (Multi-Agent Orchestration)

`.overstory/` configures agent roles (scout, builder, reviewer, lead, merger, coordinator, supervisor, monitor) with model assignments and capability grants. Hooks inject context at session start, check agent mail on prompt submit, block `git push` from agents, and capture mulch expertise after commits.

### Mulch (Expertise Management)

`.mulch/` stores accumulated project knowledge in JSONL expertise files. Overstory hooks trigger `mulch diff` after commits and `mulch learn` at session end. Domains: cicd, docs, orchestration.

### Self-Hosted

Seeds tracks its own issues via `.seeds/`. The `sd onboard` command injected the seeds section into `CLAUDE.md`. The worktree guard in `sd sync` was designed specifically for overstory's agent-in-worktree pattern.

## Navigation Guide

**To add a new command**: Create `src/commands/{name}.ts` with `run()` + `register()` exports, add lazy import in `src/index.ts` `registerAll()`, create `src/commands/{name}.test.ts`

**To modify the data model**: Update interfaces in `src/types.ts`, adjust `store.ts` read/write logic, update `doctor.ts` schema validation check

**To add a health check**: Add to the `checks` array in `src/commands/doctor.ts`, update the test count assertion in `doctor.test.ts`

**To change output formatting**: Edit `src/output.ts` helpers, respect `_quiet` flag and `--json` path

**To debug locking issues**: Check `withLock` in `src/store.ts`, stale threshold is 30s, retry with jitter at 50ms intervals
